### [Introduction](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#introduction)

When using the Nginx web server, **server blocks** (similar to virtual hosts in Apache) can be used to encapsulate configuration details and host more than one domain on a single server.

In this guide, we’ll discuss how to configure server blocks in Nginx on an Ubuntu 16.04 server.

Deploy your applications from GitHub using [DigitalOcean App Platform](https://www.digitalocean.com/products/app-platform). Let DigitalOcean focus on scaling your app.

## [Prerequisites](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#prerequisites)

We’re going to be using a non-root user with `sudo` privileges throughout this tutorial. If you do not have a user like this configured, you can create one by following our [Ubuntu 16.04 initial server setup](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04) guide.

You will also need to have Nginx installed on your server. The following guides cover this procedure:

-   [How To Install Nginx on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-16-04): Use this guide to set up Nginx on its own.
-   [How To Install Linux, Nginx, MySQL, PHP (LEMP stack) in Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-in-ubuntu-16-04): Use this guide if you will be using Nginx in conjunction with MySQL and PHP.

When you have fulfilled these requirements, you can continue on with this guide.

## [Example Configuration](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#example-configuration)

For demonstration purposes, we’re going to set up two domains with our Nginx server. The domain names we’ll use in this guide are **[example.com](http://example.com/)** and **[test.com](http://test.com/)**.

If you do not have two spare domain names to play with, use placeholder names for now and we’ll show you later how to configure your local computer to test your configuration.

## [Step 1 — Setting Up New Document Root Directories](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#step-1-setting-up-new-document-root-directories)

By default, Nginx on Ubuntu 16.04 has one server block enabled. It is configured to serve documents out of a directory at `/var/www/html`.

While this works well for a single site, we need additional directories if we’re going to serve multiple sites. We can consider the `/var/www/html` directory the default directory that will be served if the client request doesn’t match any of our other sites.

We will create a directory structure within `/var/www` for each of our sites. The actual web content will be placed in an `html` directory within these site-specific directories. This gives us some additional flexibility to create other directories associated with our sites as siblings to the `html` directory if necessary.

We need to create these directories for each of our sites. The `-p` flag tells `mkdir` to create any necessary parent directories along the way:

Now that we have our directories, we will reassign ownership of the web directories to our normal user account. This will let us write to them without `sudo`.

**Note:** Depending on your needs, you might need to adjust the permissions or ownership of the folders again to allow certain access to the `www-data` user. For instance, dynamic sites will often need this. The specific permissions and ownership requirements entirely depend on your configuration. Follow the recommendations for the specific technology you’re using.

We can use the `$USER` environmental variable to assign ownership to the account that we are currently signed in on (make sure you’re not logged in as **root**). This will allow us to easily create or edit the content in this directory:

The permissions of our web roots should be correct already if you have not modified your `umask` value, but we can make sure by typing:

Our directory structure is now configured and we can move on.

## [Step 2 — Creating Sample Pages for Each Site](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#step-2-creating-sample-pages-for-each-site)

Now that we have our directory structure set up, let’s create a default page for each of our sites so that we will have something to display.

Create an `index.html` file in your first domain:

Inside the file, we’ll create a really basic file that indicates what site we are currently accessing. It will look like this:

/var/www/example.com/html/index.html

```
&lt;html&gt;
    &lt;head&gt;
        &lt;title&gt;Welcome to <mark>Example.com</mark>!&lt;/title&gt;
    &lt;/head&gt;
    &lt;body&gt;
        &lt;h1&gt;Success! The <mark>example.com</mark> server block is working!&lt;/h1&gt;
    &lt;/body&gt;
&lt;/html&gt;
```

Save and close the file when you are finished. To do this in `nano`, press `CTRL+o` to write the file out, then `CTRL+x` to exit.

Since the file for our second site is basically going to be the same, we can copy it over to our second document root like this:

Now, we can open the new file in our editor:

Modify it so that it refers to our second domain:

/var/www/test.com/html/index.html

```
&lt;html&gt;
    &lt;head&gt;
        &lt;title&gt;Welcome to <mark>Test.com</mark>!&lt;/title&gt;
    &lt;/head&gt;
    &lt;body&gt;
        &lt;h1&gt;Success!  The <mark>test.com</mark> server block is working!&lt;/h1&gt;
    &lt;/body&gt;
&lt;/html&gt;
```

Save and close this file when you are finished. We now have some pages to display to visitors of our two domains.

## [Step 3 — Creating Server Block Files for Each Domain](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#step-3-creating-server-block-files-for-each-domain)

Now that we have the content we wish to serve, we need to create the server blocks that will tell Nginx how to do this.

By default, Nginx contains one server block called `default` which we can use as a template for our own configurations. We will begin by designing our first domain’s server block, which we will then copy over for our second domain and make the necessary modifications.

### [Creating the First Server Block File](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#creating-the-first-server-block-file)

As mentioned above, we will create our first server block config file by copying over the default file:

Now, open the new file you created in your text editor with `sudo` privileges:

Ignoring the commented lines, the file will look similar to this:

/etc/nginx/sites-available/example.com

```
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

First, we need to look at the listen directives. **Only one of our server blocks on the server can have the `default_server` option enabled.** This specifies which block should serve a request if the `server_name` requested does not match any of the available server blocks. This shouldn’t happen very frequently in real world scenarios since visitors will be accessing your site through your domain name.

You can choose to designate one of your sites as the “default” by including the `default_server` option in the `listen` directive, or you can leave the default server block enabled, which will serve the content of the `/var/www/html` directory if the requested host cannot be found.

In this guide, we’ll leave the default server block in place to serve non-matching requests, so we’ll remove the `default_server` from this and the next server block. You can choose to add the option to whichever of your server blocks makes sense to you.

/etc/nginx/sites-available/example.com

```
server {
        listen 80;
        listen [::]:80;

        . . .
}
```

**Note:** You can check that the `default_server` option is only enabled in a single active file by typing:

If matches are found uncommented in more than on file (shown in the leftmost column), Nginx will complain about an invalid configuration.

The next thing we’re going to have to adjust is the document root, specified by the `root` directive. Point it to the site’s document root that you created:

/etc/nginx/sites-available/example.com

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/<mark>example.com</mark>/html;

}
```

Next, we need to modify the `server_name` to match requests for our first domain. We can additionally add any aliases that we want to match. We will add a `www.example.com` alias to demonstrate.

When you are finished, your file will look something like this:

/etc/nginx/sites-available/example.com

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/<mark>example.com</mark>/html;
        index index.html index.htm index.nginx-debian.html;

        server_name <mark>example.com</mark> www.<mark>example.com</mark>;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

That is all we need for a basic configuration. Save and close the file to exit.

### [Creating the Second Server Block File](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#creating-the-second-server-block-file)

Now that we have our initial server block configuration, we can use that as a basis for our second file. Copy it over to create a new file:

Open the new file with `sudo` privileges in your editor:

Again, make sure that you do not use the `default_server` option for the `listen` directive in this file if you’ve already used it elsewhere. Adjust the `root` directive to point to your second domain’s document root and adjust the `server_name` to match your second site’s domain name (make sure to include any aliases).

When you are finished, your file will likely look something like this:

/etc/nginx/sites-available/test.com

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/<mark>test.com</mark>/html;
        index index.html index.htm index.nginx-debian.html;

        server_name <mark>test.com</mark> www.<mark>test.com</mark>;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

When you are finished, save and close the file.

## [Step 4 — Enabling your Server Blocks and Restart Nginx](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#step-4-enabling-your-server-blocks-and-restart-nginx)

Now that we have our server block files, we need to enable them. We can do this by creating symbolic links from these files to the `sites-enabled` directory, which Nginx reads from during startup.

We can create these links by typing:

These files are now linked into the enabled directory. We now have three server blocks enabled, which are configured to respond based on their `listen` directive and the `server_name` (you can read more about how Nginx processes these directives [here](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms)):

-   `example.com`: Will respond to requests for `example.com` and `www.example.com`
-   `test.com`: Will respond to requests for `test.com` and `www.test.com`
-   `default`: Will respond to any requests on port 80 that do not match the other two blocks.

In order to avoid a possible hash bucket memory problem that can arise from adding additional server names, we will also adjust a single value within our `/etc/nginx/nginx.conf` file. Open the file now:

Within the file, find the `server_names_hash_bucket_size` directive. Remove the `#` symbol to uncomment the line:

/etc/nginx/nginx.conf

```
http {
    . . .

    server_names_hash_bucket_size 64;

    . . .
}
```

Save and close the file when you are finished.

Next, test to make sure that there are no syntax errors in any of your Nginx files:

If no problems were found, restart Nginx to enable your changes:

Nginx should now be serving both of your domain names.

## [Step 5 — Modifying Your Local Hosts File for Testing (Optional)](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#step-5-modifying-your-local-hosts-file-for-testing-optional)

If you have not been using domain names that you own and instead have been using placeholder values, you can modify your local computer’s configuration to let you to temporarily test your Nginx server block configuration.

This will not allow other visitors to view your site correctly, but it will give you the ability to reach each site independently and test your configuration. This works by intercepting requests that would usually go to DNS to resolve domain names. Instead, we can set the IP addresses we want our local computer to go to when we request the domain names.

**Note:** Make sure you are operating on your local computer during these steps and not a remote server. You will need to have root access, be a member of the administrative group, or otherwise be able to edit system files to do this.

If you are on a Mac or Linux computer at home, you can edit the file needed by typing:

If you are on Windows, you can [find instructions for altering your hosts file](https://www.thewindowsclub.com/hosts-file-in-windows) here.

You need to know your server’s public IP address and the domains you want to route to the server. Assuming that my server’s public IP address is `203.0.113.5`, the lines I would add to my file would look something like this:

/etc/hosts

```
127.0.0.1   localhost
. . .

<mark>203.0.113.5 example.com www.example.com</mark>
<mark>203.0.113.5 test.com www.test.com</mark>
```

This will intercept any requests for `example.com` and `test.com` and send them to your server, which is what we want if we don’t actually own the domains that we are using.

Save and close the file when you are finished.

## [Step 6 — Testing Your Results](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#step-6-testing-your-results)

Now that you are all set up, you should test that your server blocks are functioning correctly. You can do that by visiting the domains in your web browser:

```
http://<mark>example.com</mark>
```

You should see a page that looks like this:

![Nginx first server block](https://assets.digitalocean.com/articles/nginx_server_block_1404/first_block.png)

If you visit your second domain name, you should see a slightly different site:

```
http://<mark>test.com</mark>
```

![Nginx second server block](https://assets.digitalocean.com/articles/nginx_server_block_1404/second_block.png)

If both of these sites work, you have successfully configured two independent server blocks with Nginx.

At this point, if you adjusted your `hosts` file on your local computer in order to test, you’ll probably want to remove the lines you added.

If you need domain name access to your server for a public-facing site, you will probably want to purchase a domain name for each of your sites.

## [Conclusion](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04#conclusion)

You should now have the ability to create server blocks for each domain you wish to host from the same server. There aren’t any real limits on the number of server blocks you can create, so long as your hardware can handle the traffic.