---
title: 'Part 1: Setting Up The Edge with NGINX and Certbot'
date: 2025-02-14 07:00:00 +0530
categories: [Architecture, Chronika]
tags: [nginx, networking, tls, certificate]
description: "In this post, we focus on setting up the edge of our secure architecture. We’ll walk you through installing NGINX and configuring Certbot to manage TLS certificates. This setup is essential for terminating client TLS connections, automating certificate renewals, and securely forwarding traffic to the internal network."
mermaid: true
---

## Introduction
To secure client interactions, our architecture begins at the edge with an NGINX that terminates TLS connections. By leveraging `Certbot`, we automate the process of obtaining and renewing certificates from `Let’s Encrypt`. This post covers:

- Installing `NGINX` and `Certbot` on your server.
- Configuring `NGINX` to handle HTTP redirection and HTTPS requests.
- Integrating `Certbot` for automated certificate management.

## Step-by-Step Implementation

### Step 1: Install NGINX
Nginx is available in Ubuntu’s default repositories, it is possible to install it from these repositories using the apt packaging system.

```sh
sudo apt update
sudo apt install nginx
```

### Step 2: Adjusting the Firewall
The firewall software needs to be adjusted to allow access to the service. Nginx registers itself as a service with ufw upon installation, making it straightforward to allow Nginx access.

Enable the firewall:
```shell
ufw enable
```

List the application that `ufw` knows:
```shell
ufw app list
```

The output lists available profiles:
```sh
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  Nginx QUIC
  OpenSSH
```

Allow HTTP and HTTPS traffic:
```shell
ufw allow 'Nginx HTTP'
ufw allow 'Nginx HTTPS'
```

Verify the change:
```shell
ufw status
```

You should see:
```shell
Status: active

To                         Action      From
--                         ------      ----
Nginx HTTP                 ALLOW       Anywhere                  
Nginx HTTPS                ALLOW       Anywhere                  
Nginx HTTP (v6)            ALLOW       Anywhere (v6)             
Nginx HTTPS (v6)           ALLOW       Anywhere (v6) 
```

### Step 3: Check the Web Server
Verify that the NGINX service is running:
```sh
systemctl status nginx
```

Confirm the server is accessible by finding your server's public IP address:
```sh
curl -4 icanhazip.com
```

Enter the server’s IP in your browser `http://your_server_ip`
You should see the default NGINX landing page.

### Step 4: Setting Up Server Block
- Nginx has one server block enabled by default that is configured to serve documents out of a directory at `/var/www/html`.
- Instead of modifying `/var/www/html`, create a directory structure within `/var/www` for our `your_domain` site, leaving `/var/www/html` in place as the default directory to be served if a client request doesn’t match any other sites.

Create the directory for your domain:
```sh
sudo mkdir -p /var/www/your_domain/html
```

Create a sample `index.html`:
```sh
vi /var/www/your_domain/html/index.html
```

Add the following HTML:
```html
<html>
    <head>
        <title>Welcome to your_domain!</title>
    </head>
    <body>
        <h1>Success! The your_domain server block is working!</h1>
    </body>
</html>
```

In order for Nginx to serve this content, it’s necessary to create a server block with the correct directives. Instead of modifying the default configuration file(/etc/nginx/nginx.conf) directly, let’s make a new one at `/etc/nginx/sites-available/your_domain`
```sh
vi /etc/nginx/sites-available/your_domain
```

Insert the following configuration:
```sh
server {
        listen 80;
        listen [::]:80;

        root /var/www/your_domain/html;
        index index.html index.htm index.nginx-debian.html;

        server_name your_domain www.your_domain;

        location / {
                try_files $uri $uri/ =404;
        }
}
```
- `root /var/www/your_domain/html;` Sets the root directory for serving files. For example, if a request is made for /about.html, Nginx will look for /var/www/your_domain/html/about.html.
- `index index.html index.htm index.nginx-debian.html;` Sets the default files to serve when a directory is requested (e.g., if user visits http://your_domain/, Nginx will try to serve index.html, then index.htm, and so on).


Enable the file by creating a link from it to the sites-enabled directory, which Nginx reads from during startup
```sh
ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
``` 

Test the configuration:
```sh
nginx -t
```

Restart NGINX:
```sh
systemctl restart nginx
```

Verify by navigating to `http://your_domain`.

### Step 5: Install Certbot
Certbot recommends using their snap package for installation. Snap packages work on nearly all Linux distributions. Make sure the `snapd` core is up to date:
```sh
snap install core; snap refresh core
```

Install `certbot` package:
```sh
snap install --classic certbot
```

Create a symbolic link for `Certbot`:
```sh
ln -s /snap/bin/certbot /usr/bin/certbot
```

### Step 6: Obtaining an SSL Certificate
Certbot provides a variety of ways to obtain SSL certificates through plugins. The Nginx plugin will take care of reconfiguring Nginx and reloading the config whenever necessary.
```sh
certbot --nginx -d example.com
```

Replace `example.com` with your actual domain name. Certbot will:
-  Run certbot with the `--nginx` plugin, using `-d` to specify the domain names.
- Configure SSL settings in your NGINX configuration.
- Obtain certificates from Let's Encrypt.
- Set up auto-renewal.


### Step 7: Verify Secure Connection

- Once `Certbot` completes, your site should automatically redirect to `https://` and display a secure padlock in the browser. 
- Test server using the [SSL Labs Server Test](https://www.ssllabs.com/ssltest/), it should get an A grade.


### Step 8: Verifying Certbot Auto-Renewal
Let’s Encrypt’s certificates are valid for ninety days. This is to encourage users to automate their certificate renewal process. The `certbot` package takes care of this for us by adding a `systemd` timer that will run twice a day and automatically renew any certificate that’s within `thirty days` of expiration.

Check the timer status:
```sh
systemctl status snap.certbot.renew.service
```

```sh
Output
○ snap.certbot.renew.service - Service for snap application certbot.renew
     Loaded: loaded (/etc/systemd/system/snap.certbot.renew.service; static)
     Active: inactive (dead)
TriggeredBy: ● snap.certbot.renew.timer
```

## Next
With the edge layer secured using NGINX and Certbot, the next step is to handle internal routing within our Kubernetes cluster. In [Part 2: Setting up Ingress with Ingress NGINX](../Breaking-Down-Our-Server-Architecture-Part-2), we walk through:

- Deploying an example internal service,
- Installing the NGINX Ingress Controller with ModSecurity for enhanced security,
- Using Cert-Manager to automate TLS certificates for internal load balancers,
- And delegating DNS to manage internal domains via DigitalOcean's DNS and DNS-01 challenges.

## References
- [How to install nginx on ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04)
- [How to secure nginx with Let's Encrypt on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-22-04)
- [SSL Labs Server Test](https://www.ssllabs.com/ssltest/)