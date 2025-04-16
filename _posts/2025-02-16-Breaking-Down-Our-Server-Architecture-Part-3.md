---
title: 'Part 3: Securely Bridging Edge and Internal with TLS'
date: 2025-02-16 07:00:00 +0530
categories: [Architecture, Chronika]
tags: [kubernetes, nginx, networking, tls, certificate]
description: "In this post, we continue from the edge where TLS was terminated and explore how to forward traffic from the public-facing NGINX to an internal load balancer securely. We establish a new TLS connection between the edge NGINX and the internal ingress, ensuring end-to-end encryption within your infrastructure."
mermaid: true
---

## Introduction
Terminating TLS at the edge is common, but it's not always sufficient, especially in zero-trust or multi-tenant environments. To truly secure internal communication, we need to extend TLS encryption to backend services as well.

In this part, we’ll configure the edge NGINX server to proxy HTTPS requests to the internal ingress, ensuring that encrypted traffic flows across your infrastructure—not just up to the edge.

This post covers:
- Configuring the edge NGINX as a reverse proxy over TLS.
- Setting up an internal ingress to accept HTTPS.
- Validating certificate-based authentication and securing internal communication.


## Step-by-Step Implementation

### Step 1: Prerequisites
Ensure the following setup is already complete:
- A working internal load balancer managed by the Ingress NGINX controller, as set up in [Part 2](../Breaking-Down-Our-Server-Architecture-Part-2).
- Valid TLS certificates configured on the internal ingress.
- An edge NGINX server running with Certbot handling TLS termination and certificate management, as described in [Part 1](../Breaking-Down-Our-Server-Architecture-Part-1).


### Step 2: Update Edge NGINX to Proxy via HTTPS
Before updating your NGINX configuration, verify that the edge machine can connect to the internal load balancer over HTTPS:
```sh
curl -X GET "https://lb.internal.example.com:443/" -v
```

If everything is set up correctly, this should return a response from your backend (e.g., a sample app from Part 2).

Now, update the edge NGINX configuration to proxy over HTTPS.

```conf
server {

    listen 443 ssl;
    listen [::]:443 ssl;

    server_name your_domain.com www.your_domain.fun;

    # SSL Settings (managed by Certbot)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass https://lb.internal.example.com:443/;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Upstream TLS settings for HTTPS proxying
        proxy_ssl_verify on;
        proxy_ssl_server_name on;
        proxy_ssl_name lb.internal.example.com;
        proxy_ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name example.com;

    # Redirect to HTTPS
    if ($host = example.com) {
        return 301 https://$host$request_uri;
    }

    # Fallback (should never hit this)
    return 404;
}
```
- `proxy_pass https://...`: Sends requests to the internal load balancer over HTTPS.
- `proxy_ssl_verify on;`: Validates the TLS cert from the internal LB.
- `proxy_ssl_name and proxy_ssl_server_name on;`: Ensures proper SNI and hostname match.
- `proxy_ssl_trusted_certificate`: Specifies trusted CAs for backend cert validation.
- `proxy_set_header lines`: Forward original client info to your backend.

> Do not use `proxy_set_header Host $host`. This will preserve the original host (e.g., `your_domain.com`), which may not match the internal ingress host rule (e.g., `lb.internal.example.com`). The internal ingress requires the correct Host header to route properly.
{: .prompt-danger }


After editing the configuration, validate and reload NGINX:
```sh
nginx -t
systemctl restart nginx
```

### Step 3: Update Ingress Controller to Show Real Client IP
Update your Helm values for the ingress controller:
```yaml
controller:
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/do-loadbalancer-tls-passthrough: "true"
      service.beta.kubernetes.io/do-loadbalancer-network: "INTERNAL"
      service.beta.kubernetes.io/do-loadbalancer-protocol: "tcp"
      service.beta.kubernetes.io/do-loadbalancer-hostname: lb.internal.example.com
      service.beta.kubernetes.io/do-loadbalancer-name: lb-internal
  config:
    enable-modsecurity: "true"
    enable-owasp-modsecurity-crs: "true"
    use-forwarded-headers: "true"
    ssl-redirect: "true"
    modsecurity-snippet: |
      # Enable prevention mode. Options: DetectionOnly, On, Off (default is DetectionOnly)
      SecRuleEngine On

      # Enable scanning of the request body
      SecRequestBodyAccess On

      # Reject if larger (we could also let it pass with ProcessPartial)
      SecRequestBodyLimitAction Reject

      # Send ModSecurity audit logs to the stdout (only for rejected requests)
      SecAuditLog /dev/stdout

      # Format the logs in JSON
      SecAuditLogFormat JSON

      # Could be On/Off/RelevantOnly
      SecAuditEngine RelevantOnly

      # Enable XML parsing.
      SecRule REQUEST_HEADERS:Content-Type "(?:text|application(?:/soap\+|/)|application/xml)/" \
        "id:2000000,phase:1,t:none,t:lowercase,pass,nolog,ctl:requestBodyProcessor=XML"

      # Enable JSON parsing.
      SecRule REQUEST_HEADERS:Content-Type "application/json" \
        "id:2000001,phase:1,t:none,t:lowercase,pass,nolog,ctl:requestBodyProcessor=JSON"      

      # Enable requests with GRPC body.
      SecRule REQUEST_HEADERS:Content-Type "application/grpc" \
        "id:2000002,phase:1,t:none,t:lowercase,pass,nolog,setvar:tx.allowed_request_content_type=application/xml|application/grpc|application/json"      

      SecRule REQUEST_HEADERS:Content-Type "application/grpc" \
        "id:2000003,phase:1,pass,nolog,setvar:tx.allowed_request_content_type=application/xml|application/grpc|application/json"

```

Upgrade ingress controller using helm
```sh
helm upgrade --install ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx  \
    --namespace ingress-nginx \
    --set controller.config.enable-modsecurity=true \
    --set controller.config.enable-owasp-modsecurity-crs=true \
    -f values.yaml
```

Enabling `use-forwarded-headers` ensures the ingres controller logs and forwards the actual client IP from the edge proxy.


### Step 4: Verify the TLS Setup and Connectivity
After updating the configurations, verify that TLS is correctly established from end to end. Use the following command from outside your network:

The output
```sh
curl -vk https://example.com/status
```

What to Look For:
- Successful SSL Handshake: Ensure that the SSL handshake completes without errors.
- Proper Certificate Verification: The proxy_ssl_verify setting on the NGINX configuration ensures that the backend certificate is valid and trusted.
- Expected Response: `hello-world` with 200.

## Wrapping Up
With this setup, you've now extended secure HTTPS communication beyond the edge and into your internal infrastructure. TLS is no longer just a boundary-layer concern, it’s part of every connection from the edge NGINX to your internal services, ensuring end-to-end encryption and integrity.


