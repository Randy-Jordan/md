One of the most useful, but often misunderstood and misconfigured, features of NGINX is rate limiting. It allows you to limit the amount of HTTP requests a user can make in a given period of time. A request can be as simple as a `GET` request for the homepage of a website or a `POST` request on a log‑in form.

Rate limiting can be used for security purposes, for example to slow down brute‑force password‑guessing attacks. It can help [protect against DDoS attacks](https://www.nginx.com/blog/mitigating-ddos-attacks-with-nginx-and-nginx-plus/) by limiting the incoming request rate to a value typical for real users, and (with logging) identify the targeted URLs. More generally, it is used to protect upstream application servers from being overwhelmed by too many user requests at the same time.

In this blog we will cover the basics of rate limiting with NGINX as well as more advanced configurations. Rate limiting works the same way in NGINX Plus.

To learn more about rate limiting with NGINX, watch our [on-demand webinar](https://www.nginx.com/resources/webinars/rate-limiting-nginx/).

## How NGINX Rate Limiting Works

NGINX rate limiting uses the “leaky bucket algorithm”, which is widely used in telecommunications and packet‑switched computer networks to deal with burstiness when bandwidth is limited. The analogy is with a bucket where water is poured in at the top and leaks from the bottom; if the rate at which water is poured in exceeds the rate at which it leaks, the bucket overflows. In terms of request processing, the water represents requests from clients, and the bucket represents a queue where requests wait to be processed according to a first‑in‑first‑out (FIFO) scheduling algorithm. The leaking water represents requests exiting the buffer for processing by the server, and the overflow represents requests that are discarded and never serviced.

## Configuring Basic Rate Limiting

Rate limiting is configured with two main directives, `limit_req_zone` and `limit_req`, as in this example:

```
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
 
server {
    location /login/ {
        limit_req zone=mylimit;
        
        proxy_pass http://my_upstream;
    }
}
```

The [`limit_req_zone`](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req_zone) directive defines the parameters for rate limiting while [`limit_req`](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req) enables rate limiting within the context where it appears (in the example, for all requests to **/login/**).

The `limit_req_zone` directive is typically defined in the `http` block, making it available for use in multiple contexts. It takes the following three parameters:

-   **Key** – Defines the request characteristic against which the limit is applied. In the example it is the NGINX variable `$binary_remote_addr`, which holds a binary representation of a client’s IP address. This means we are limiting each unique IP address to the request rate defined by the third parameter. (We’re using this variable because it takes up less space than the string representation of a client IP address, `$remote_addr`).  
    
-   **Zone** – Defines the shared memory zone used to store the state of each IP address and how often it has accessed a request‑limited URL. Keeping the information in shared memory means it can be shared among the NGINX worker processes. The definition has two parts: the zone name identified by the `zone=` keyword, and the size following the colon. State information for about 16,000 IP addresses takes 1 ;megabyte, so our zone can store about 160,000 addresses.
    -   If storage is exhausted when NGINX needs to add a new entry, it removes the oldest entry. If the space freed is still not enough to accommodate the new record, NGINX returns status code `503 (Service Temporarily Unavailable)`. Additionally, to prevent memory from being exhausted, every time NGINX creates a new entry it removes up to two entries that have not been used in the previous 60 seconds.  
        
-   **Rate** – Sets the maximum request rate. In the example, the rate cannot exceed 10 requests per second. NGINX actually tracks requests at millisecond granularity, so this limit corresponds to 1 request every 100 milliseconds (ms). Because we are not allowing for bursts (see the [next section](https://blog.nginx.org/blog/rate-limiting-nginx#bursts)), this means that a request is rejected if it arrives less than 100ms after the previous permitted one.

The `limit_req_zone` directive sets the parameters for rate limiting and the shared memory zone, but it does not actually limit the request rate. For that you need to apply the limit to a specific `location` or `server` block by including a `limit_req` directive there. In the example, we are rate limiting requests to **/login/**.

So now each unique IP address is limited to 10 requests per second for **/login/** – or more precisely, cannot make a request for that URL within 100ms of its previous one.

## Handling Bursts

What if we get 2 requests within 100ms of each other? For the second request NGINX returns status code `503` to the client. This is probably not what we want, because applications tend to be bursty in nature. Instead we want to buffer any excess requests and service them in a timely manner. This is where we use the `burst` parameter to `limit_req`, as in this updated configuration:

```
location /login/ {
    limit_req zone=mylimit <strong>burst=20</strong>;
 
    proxy_pass http://my_upstream;
}
```

The `burst` parameter defines how many requests a client can make in excess of the rate specified by the zone (with our sample **mylimit** zone, the rate limit is 10 requests per second, or 1 every 100ms). A request that arrives sooner than 100ms after the previous one is put in a queue, and here we are setting the queue size to 20.

That means if 21 requests arrive from a given IP address simultaneously, NGINX forwards the first one to the upstream server group immediately and puts the remaining 20 in the queue. It then forwards a queued request every 100ms, and returns `503` to the client only if an incoming request makes the number of queued requests go over 20.

## Queueing with No Delay

A configuration with `burst` results in a smooth flow of traffic, but is not very practical because it can make your site appear slow. In our example, the 20th packet in the queue waits 2 seconds to be forwarded, at which point a response to it might no longer be useful to the client. To address this situation, add the `nodelay` parameter along with the `burst` parameter:

```
location /login/ {
    limit_req zone=mylimit <strong>burst=20 nodelay</strong>;
 
    proxy_pass http://my_upstream;
}
```

With the `nodelay` parameter, NGINX still allocates slots in the queue according to the `burst` parameter and imposes the configured rate limit, but not by spacing out the forwarding of queued requests. Instead, when a request arrives “too soon”, NGINX forwards it immediately as long as there is a slot available for it in the queue. It marks that slot as “taken” and does not free it for use by another request until the appropriate time has passed (in our example, after 100ms).

Suppose, as before, that the 20‑slot queue is empty and 21 requests arrive simultaneously from a given IP address. NGINX forwards all 21 requests immediately and marks the 20 slots in the queue as taken, then frees 1 slot every 100ms. (If there were 25 requests instead, NGINX would immediately forward 21 of them, mark 20 slots as taken, and reject 4 requests with status `503`.)

Now suppose that 101ms after the first set of requests was forwarded another 20 requests arrive simultaneously. Only 1 slot in the queue has been freed, so NGINX forwards 1 request and rejects the other 19 with status `503`. If instead 501ms have passed before the 20 new requests arrive, 5 slots are free so NGINX forwards 5 requests immediately and rejects 15.

The effect is equivalent to a rate limit of 10 requests per second. The `nodelay` option is useful if you want to impose a rate limit without constraining the allowed spacing between requests.

**Note:** For most deployments, we recommend including the `burst` and `nodelay` parameters to the `limit_req` directive.

## Two-Stage Rate Limiting

With NGINX Open Source 1.15.7, you can configure NGINX to allow a burst of requests to accommodate the typical web browser request pattern, and then throttle additional excessive requests up to a point, beyond which additional excessive requests are rejected. Two-stage rate limiting is enabled with the [`delay`](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req_delay) parameter to the `limit_req` directive.

To illustrate two‑stage rate limiting, here we configure NGINX to protect a website by imposing a rate limit of 5 requests per second (r/s). The website typically has 4–6 resources per page, and never more than 12 resources. The configuration allows bursts of up to 12 requests, the first 8 of which are processed without delay. A delay is added after 8 excessive requests to enforce the 5 r/s limit. After 12 excessive requests, any further requests are rejected.

```
limit_req_zone $binary_remote_addr zone=ip:10m rate=5r/s;

server {
    listen 80;
    location / {
        limit_req zone=ip burst=12 delay=8;
        proxy_pass http://website;
    }
}
```

The `delay` parameter defines the point at which, within the burst size, excessive requests are throttled (delayed) to comply with the defined rate limit. With this configuration in place, a client that makes a continuous stream of requests at 8 r/s experiences the following behavior.

![Illustration of rate‑limiting behavior with rate=5r/s burst=12 delay=8](https://nginxblog-8de1046ff5a84f2c-endpoint.azureedge.net/blobnginxbloga72cde487e/wp-content/uploads/2024/06/two-stage-rate-limiting-example.png)

Illustration of rate‑limiting behavior with `rate=5r/s` `burst=12` `delay=8`

The first 8 requests (the value of `delay`) are proxied by NGINX Plus without delay. The next 4 requests (`burst` `-` `delay`) are delayed so that the defined rate of 5 r/s is not exceeded. The next 3 requests are rejected because the total burst size has been exceeded. Subsequent requests are delayed.

## Advanced Configuration Examples

By combining basic rate limiting with other NGINX features, you can implement more nuanced traffic limiting.

### Allowlisting

This example shows how to impose a rate limit on requests from anyone who is not on an “allowlist”.

```
geo $limit {
    default 1;
    10.0.0.0/8 0;
    192.168.0.0/24 0;
}
 
map $limit $limit_key {
    0 "";
    1 $binary_remote_addr;
}
 
limit_req_zone $limit_key zone=req_zone:10m rate=5r/s;
 
server {
    location / {
        limit_req zone=req_zone burst=10 nodelay;
 
        # ...
    }
}
```

This example makes use of both the [`geo`](http://nginx.org/en/docs/http/ngx_http_geo_module.html#geo) and [`map`](http://nginx.org/en/docs/http/ngx_http_map_module.html#map) directives. The `geo` block assigns a value of `0` to `$limit` for IP addresses in the allowlist and `1` for all others. We then use a map to translate those values into a key, such that:

-   If `$limit` is `0`, `$limit_key` is set to the empty string
-   If `$limit` is `1`, `$limit_key` is set to the client’s IP address in binary format

Putting the two together, `$limit_key` is set to an empty string for allowlisted IP addresses, and to the client’s IP address otherwise. When the first parameter to the `limit_req_zone` directory (the key) is an empty string, the limit is not applied, so allowlisted IP addresses (in the 10.0.0.0/8 and 192.168.0.0/24 subnets) are not limited. All other IP addresses are limited to 5 requests per second.

The `limit_req` directive applies the limit to the **/** location and allows bursts of up to 10 packets over the configured limit with no delay on forwarding

### Including Multiple `limit_req` Directives in a Location

You can include multiple `limit_req` directives in a single location. All limits that match a given request are applied, meaning the most restrictive one is used. For example, if more than one directive imposes a delay, the longest delay is used. Similarly, requests are rejected if that is the effect of any directive, even if other directives allow them through.

Extending the previous example we can apply a rate limit to IP addresses on the allowlist:

```
http {
    # ...
 
    limit_req_zone $limit_key zone=req_zone:10m rate=5r/s;
    limit_req_zone $binary_remote_addr zone=req_zone_wl:10m rate=15r/s;
 
    server {
        # ...
        location / {
            limit_req zone=req_zone burst=10 nodelay;
            limit_req zone=req_zone_wl burst=20 nodelay;
            # ...
        }
    }
}
```

IP addresses on the allowlist do not match the first rate limit (**req\_zone**) but do match the second (**req\_zone\_wl**) and so are limited to 15 requests per second. IP addresses not on the allowlist match both rate limits so the more restrictive one applies: 5 requests per second.

## Configuring Related Features

### Logging

By default, NGINX logs requests that are delayed or dropped due to rate limiting, as in this example:

```
2015/06/13 04:20:00 [error] 120315#0: *32086 limiting requests, excess: 1.000 by zone "mylimit", client: 192.168.1.2, server: nginx.com, request: "GET / HTTP/1.0", host: "nginx.com"
```

Fields in the log entry include:

-   **`2015/06/13` `04:20:00`** – Date and time the log entry was written
-   `**[error]**` – Severity level
-   `**120315#0**` – Process ID and thread ID of the NGINX worker, separated by the `#` sign
-   `***32086**` – ID for the proxied connection that was rate‑limited
-   **`limiting` `requests`** – Indicator that the log entry records a rate limit
-   `**excess**` – Number of requests per millisecond over the configured rate that this request represents
-   `**zone**` – Zone that defines the imposed rate limit
-   `**client**` – IP address of the client making the request
-   `**server**` – IP address or hostname of the server
-   `**request**` – Actual HTTP request made by the client
-   `**host**` – Value of the `Host` HTTP header

By default, NGINX logs refused requests at the `error` level, as shown by `[error]` in the example above. (It logs delayed requests at one level lower, so `warn` by default.) To change the logging level, use the [`limit_req_log_level`](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req_log_level) directive. Here we set refused requests to log at the `warn` level:

```
location /login/ {
    limit_req zone=mylimit burst=20 nodelay;
    <strong>limit_req_log_level warn;</strong>
 
    proxy_pass http://my_upstream;
}
```

### Error Code Sent to Client

By default NGINX responds with status code `503 (Service Temporarily Unavailable)` when a client exceeds its rate limit. Use the [`limit_req_status`](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req_status) directive to set a different status code (`444` in this example):

```
location /login/ {
    limit_req zone=mylimit <strong>burst=20 nodelay;
    limit_req_status 444</strong>;
}
```

### Denying All Requests to a Specific Location

If you want to deny all requests for a specific URL, rather than just limiting them, configure a [`location`](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) block for it and include the [`deny`](http://nginx.org/en/docs/http/ngx_http_access_module.html#deny) `all` directive:

```
location /foo.php {
    deny all;
}
```

## Conclusion

We have covered many features of rate limiting that NGINX offers, including setting up request rates for different locations on HTTP requests, and configuring additional features to rate limiting such as the `burst` and `nodelay` parameters. We have also covered advanced configuration for applying different limits for allowlisted and denylisted client IP addresses, and explained how to log rejected and delayed requests.