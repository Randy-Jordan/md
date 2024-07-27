# Nginx Server

Walkthrough of installing Nginx Webserver

## Table of Contents

- [Step One - Installing Nginx](#step-one---installing-nginx)
- [Step Two - Configuring our Server](#step-2---configuring-our-server)
- [Step Three - Troubleshooting your Server](#step-3---troubleshooting-your-server)
- [Step Four - OPTIONAL - Setting up HTTPS with Certbot](#step-4---setting-up-https-with-certbot)
- [Resources](#resoureces)

### Step One - Installing Nginx
`sudo apt update`<br>
`sudo apt install nginx`<br>
If you have UFW on you should allow HTTP with:<br>
`sudo ufw allow 'Nginx Full'`
Next we need to install Nginx.<br>


Nginx should now be fully operational, to check.<br>
`sudo systemctl enable nginx` - To set up nginx to start up on reboot.<br>
`sudo systemctl start nginx`<br>
`sudo systemctl status nginx` - Should give you the status of the server.<br>

Go to your ip address to see the nginx default page. Next, we'll configure our server.

### Step 2 - Configuring Our Server

Now that nginx is install and running, we should now configure our server. Here are the files we normally edit.<br>
`/var/www/html` Typically you'll remove the default and create a folder like `/var/www/mysite.com`<br>
`/etc/nginx`: The Nginx configuration directory. All of the Nginx configuration files reside here.<br>
`/etc/nginx/nginx.conf`:The main Nginx configuration file. This can be modified to make changes to the Nginx global configuration.<br>
`/etc/nginx/sites-available/mysite.com`:The directory where per-site server blocks can be stored. Nginx will not use the configuration files found in this directory unless they are linked to the **sites-enabled** directory. Typically, all server block configuration is done in this directory, and then enabled by linking to the other directory.<br>
`/etc/nginx/sites-enabled/mysite.com`: he directory where enabled per-site server blocks are stored. Typically, these are created by linking to configuration files found in the **sites-available** directory.<br>

You can link those by `sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/`<br>

`/etc/nginx/snippets`: This directory contains configuration fragments that can be included elsewhere in the Nginx configuration. Potentially repeatable configuration segments are good candidates for refactoring into snippets.<br>
<br>
Here are some samples of files you may encounter, and settings you may set.<br>
[/etc/nginx/nginx.conf](./defaults/nginx.conf)<br>
[/etc/nginx/sites-available](./defaults/nginx.conf)<br>
[Digital Ocean - Server Blocks](./resources/DigitalOcean_ServerBlocks.md)<br>
[Rate Limiting With Nginx](./resources/Rate_Limiting_With_Nginx)<br>


### Step 3 - Troubleshooting Your Server

`/var/log/nginx/access.log`: Every request to your web server is recorded in this log file unless Nginx is configured to do otherwise.<br>
`/var/log/nginx/error.log`: 
Any Nginx errors will be recorded in this log.<br>

### Step 4 - Setting up HTTPS with Certbot
Now with Nginx configured and running we should set up HTTPS with Certbot.<br>
`sudo apt update`<br>
`sudo apt install snapd`<br>
`sudo snap install core`<br>
`sudo snap refresh core`<br>
`sudo snap install --classic certbot`<br>
`sudo ln -s /snap/bin/certbot /usr/bin/certbot`<br>


Fill out the questions about domain name email etc.<br>

`sudo nginx -t`<br>
`sudo systemctl reload nginx`<br>

Now you should have a fully functional HTTPS server.

### Resources 
I found these resources helpful.

[Digital Ocean - Nginx ](./resources/DigitalOcean_Nginx.md)<br>
[Digital Ocean - Let's Encrypt](./resources/DigitalOcean_LetsEncrypt.md)<br>
[Digital Ocean - Server Blocks](./resources/DigitalOcean_ServerBlocks.md)<br>
[Rate Limiting With Nginx](./resources/Rate_Limiting_With_Nginx)<br>
[![Youtube](https://i.ytimg.com/vi/HaY8QB5kkGw/hqdefault.jpg)](https://youtu.be/HaY8QB5kkGw?si=9k44i9hon35KsaYp)<br>
[![YouTube](https://i.ytimg.com/vi/-lrSPJTeGhQ/hqdefault.jpg)](https://www.youtube.com/watch?v=-lrSPJTeGhQ)

