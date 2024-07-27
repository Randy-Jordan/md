### [Introduction](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04#introduction)

Let’s Encrypt is a Certificate Authority (CA) that provides an accessible way to obtain and install free [TLS/SSL certificates](https://www.digitalocean.com/community/tutorials/openssl-essentials-working-with-ssl-certificates-private-keys-and-csrs), thereby enabling encrypted HTTPS on web servers. It simplifies the process by providing a software client, Certbot, that attempts to automate most (if not all) of the required steps. Currently, the entire process of obtaining and installing a certificate is fully automated on both Apache and Nginx.

In this tutorial, you will use Certbot to obtain a free SSL certificate for Nginx on Ubuntu 22.04 and set up your certificate to renew automatically.

This tutorial will use a separate Nginx server configuration file instead of the default file. [We recommend](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-22-04#step-5-%E2%80%93-setting-up-server-blocks-(recommended)) creating new Nginx server block files for each domain because it helps to avoid common mistakes and maintains the default files as a fallback configuration.

## [Prerequisites](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04#prerequisites)

To follow this tutorial, you will need:

-   One Ubuntu 22.04 server set up by following this [initial server setup for Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-22-04) tutorial, including a sudo-enabled non-**root** user and a firewall.
    
-   A registered domain name. This tutorial will use `example.com` throughout. You can purchase a domain name from [Namecheap](https://namecheap.com/), get one for free with [Freenom](https://www.freenom.com/), or use the domain registrar of your choice.
    
-   Both of the following DNS records set up for your server. If you are using DigitalOcean, please see our [DNS documentation](https://www.digitalocean.com/docs/networking/dns/) for details on how to add them.
    
    -   An A record with `example.com` pointing to your server’s public IP address.
    -   An A record with `www.example.com` pointing to your server’s public IP address.
-   Nginx installed by following [How To Install Nginx on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-22-04). Be sure that you have a [server block](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-22-04#step-5-%E2%80%93-setting-up-server-blocks-(recommended)) for your domain. This tutorial will use `/etc/nginx/sites-available/example.com` as an example.
    

## [Step 1 — Installing Certbot](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04#step-1-installing-certbot)

Certbot recommends using their _snap_ package for installation. Snap packages work on nearly all Linux distributions, but they require that you’ve installed snapd first in order to manage snap packages. Ubuntu 22.04 comes with support for snaps out of the box, so you can start by making sure your snapd core is up to date:

If you’re working on a server that previously had an older version of certbot installed, you should remove it before going any further:

After that, you can install the `certbot` package:

Finally, you can link the `certbot` command from the snap install directory to your path, so you’ll be able to run it by just typing `certbot`. This isn’t necessary with all packages, but snaps tend to be less intrusive by default, so they don’t conflict with any other system packages by accident:

Now that we have Certbot installed, let’s run it to get our certificate.

## [Step 2 — Confirming Nginx’s Configuration](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04#step-2-confirming-nginx-s-configuration)

Certbot needs to be able to find the correct `server` block in your Nginx configuration for it to be able to automatically configure SSL. Specifically, it does this by looking for a `server_name` directive that matches the domain you request a certificate for.

If you followed the [server block set up step in the Nginx installation tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-22-04#step-5-%E2%80%93-setting-up-server-blocks-(recommended)), you should have a server block for your domain at `/etc/nginx/sites-available/example.com` with the `server_name` directive already set appropriately.

To check, open the configuration file for your domain using `nano` or your favorite text editor:

Find the existing `server_name` line. It should look like this:

/etc/nginx/sites-available/example.com

If it does, exit your editor and move on to the next step.

If it doesn’t, update it to match. Then save the file, quit your editor, and verify the syntax of your configuration edits:

If you get an error, reopen the server block file and check for any typos or missing characters. Once your configuration file’s syntax is correct, reload Nginx to load the new configuration:

Certbot can now find the correct `server` block and update it automatically.

Next, let’s update the firewall to allow HTTPS traffic.

## [Step 3 — Allowing HTTPS Through the Firewall](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04#step-3-allowing-https-through-the-firewall)

If you have the `ufw` firewall enabled, as recommended by the prerequisite guides, you’ll need to adjust the settings to allow for HTTPS traffic. Luckily, Nginx registers a few profiles with `ufw` upon installation.

You can see the current setting by typing:

It will probably look like this, meaning that only HTTP traffic is allowed to the web server:

```
<p>Output</p>Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Nginx HTTP                 ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx HTTP (v6)            ALLOW       Anywhere (v6)
```

To additionally let in HTTPS traffic, allow the Nginx Full profile and delete the redundant Nginx HTTP profile allowance:

Your status should now look like this:

```
<p>Output</p>Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
Nginx Full (v6)            ALLOW       Anywhere (v6)
```

Next, let’s run Certbot and fetch our certificates.

## [Step 4 — Obtaining an SSL Certificate](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04#step-4-obtaining-an-ssl-certificate)

Certbot provides a variety of ways to obtain SSL certificates through plugins. The Nginx plugin will take care of reconfiguring Nginx and reloading the config whenever necessary. To use this plugin, type the following:

This runs `certbot` with the `--nginx` plugin, using `-d` to specify the domain names we’d like the certificate to be valid for.

When running the command, you will be prompted to enter an email address and agree to the terms of service. After doing so, you should see a message telling you the process was successful and where your certificates are stored:

```
<p>Output</p>IMPORTANT NOTES:
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/<mark>your_domain</mark>/fullchain.pem
Key is saved at: /etc/letsencrypt/live/<mark>your_domain</mark>/privkey.pem
This certificate expires on 2022-06-01.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
* Donating to ISRG / Let's Encrypt: https://letsencrypt.org/donate
* Donating to EFF: https://eff.org/donate-le
```

Your certificates are downloaded, installed, and loaded, and your Nginx configuration will now automatically redirect all web requests to `https://`. Try reloading your website and notice your browser’s security indicator. It should indicate that the site is properly secured, usually with a lock icon. If you test your server using the [SSL Labs Server Test](https://www.ssllabs.com/ssltest/), it will get an **A** grade.

Let’s finish by testing the renewal process.

## [Step 5 — Verifying Certbot Auto-Renewal](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04#step-5-verifying-certbot-auto-renewal)

Let’s Encrypt’s certificates are only valid for ninety days. This is to encourage users to automate their certificate renewal process. The `certbot` package we installed takes care of this for us by adding a systemd timer that will run twice a day and automatically renew any certificate that’s within thirty days of expiration.

You can query the status of the timer with `systemctl`:

```
<p>Output</p>○ snap.certbot.renew.service - Service for snap application certbot.renew
     Loaded: loaded (/etc/systemd/system/snap.certbot.renew.service; static)
     Active: inactive (dead)
TriggeredBy: ● snap.certbot.renew.timer
```

To test the renewal process, you can do a dry run with `certbot`:

If you see no errors, you’re all set. When necessary, Certbot will renew your certificates and reload Nginx to pick up the changes. If the automated renewal process ever fails, Let’s Encrypt will send a message to the email you specified, warning you when your certificate is about to expire.

## [Conclusion](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04#conclusion)

In this tutorial, you installed the Let’s Encrypt client `certbot`, downloaded SSL certificates for your domain, configured Nginx to use these certificates, and set up automatic certificate renewal. If you have further questions about using Certbot, [the official documentation](https://certbot.eff.org/docs/) is a good place to start.