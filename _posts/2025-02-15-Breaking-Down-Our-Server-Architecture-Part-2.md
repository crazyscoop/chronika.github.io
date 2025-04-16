---
title: 'Part 2: Setting up Ingress with Ingress Nginx'
date: 2025-02-14 07:00:00 +0530
categories: [Architecture, Chronika]
tags: [nginx, networking, tls, certificates]
description: "In this post, we continue building our secure server architecture by configuring the internal ingress layer. We’ll set up an example service, install the NGINX Ingress Controller with ModSecurity, and integrate Cert-Manager to automate TLS certificate management. Additionally, we’ll walk through delegating subdomain DNS to DigitalOcean for internal load balancers and securing them with DNS-01 challenges."
mermaid: true
---

## Introduction
In the [previous post](../Breaking-Down-Our-Server-Architecture-Part-1/) we configured NGINX at the edge to terminate TLS connection.  In this post we continue by adding an ingress to our Kubernetes cluster and managing the load balancer that is provisioned. We will cover:

- Setting up an example service. 
- Installing the `Ingress Nginx` ingress controller with `ModSecurity`.
- Integrating `CertManager` to automate certificate management for the load balancer.

## Step-by-Step Implementation

### Step 1: Setting up an Example Service
- Follow the steps mentioned in [this post](../Example-Service/) to setup your example service for testing.
- Confirm that the service is reachable using port-forwarding.


### Step 2: Install Ingress NGINX controller with ModSecurity
Follow the guidelines in [this post](../Securing-Kubernetes-Ingress-With-ModSecurity/) to deploy the kubernetes ingress controller with ModSecurity.

> If you are using DigitalOcean, ensure that the provisioned load balancer is private and not accessible externally. This can be done by adding the following annotation in your Helm values file: `service.beta.kubernetes.io/do-loadbalancer-network: "INTERNAL"`.
{: .prompt-danger }


### Step 3: Install Cert-Manager
Add the helm repository for Cert-Manager:
```sh
helm repo add jetstack https://charts.jetstack.io --force-update
```

Install Cert-manager with CRDs enabled.
```sh
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true
```


### Step 4: (For DigitalOcean) Delegate Subdomain DNS management to DigitalOcean DNS
Since the load balancer created by ingress in private, Cert-Manager's certificate issuer must verify domain ownership using the ACME `DNS-01` challenge. For DigitalOcean, follow these steps:


1. Create a [Personal Access Token](https://docs.digitalocean.com/reference/api/create-personal-access-token/) with read and write access for DigitalOcean.

2. Base64 Encode your token:
```sh
echo -n 'access-token' | base64
```

3. Create a kubernetes secret: Save the encoded token in a secret, for example `lets-encrypt-do-dns.yaml`:
```sh
apiVersion: v1
kind: Secret
metadata:
  namespace: cert-manager
  name: lets-encrypt-do-dns
data:
  access-token: base64-access-token
```

4. Apply the secret to your cluster
```sh
kubectl apply -f lets-encrypt-do-dns.yaml
```

#### Internal DNS Management
To point a subdomain (e.g., `sub2.sub1.domain.com`) to an internal load balancer using DigitalOcean-managed DNS and to delegate full DNS management for a subdomain (e.g., internal.example.com):
1. Set Up NS Records: At your DNS registrar for the main domain (e.g., example.com), create NS (Name Server) records to delegate the subdomain:
  - Type: `NS`
  - Host: `internal` (or your chosen subdomain)
  - Value: `ns1.digitalocean.com, ns2.digitalocean.com, ns3.digitalocean.com` (DigitalOcean's name servers)
2. Add the Domain to DigitalOcean: Go to DigitalOcean → Networking → Domains
  - Click “Add Domain”
  - Enter: `internal.example.com`
  - You’ll now be managing `*.internal.example.com` records within DigitalOcean.
3. Create an A record in DigitalOcean
  - Type: `A`
  - Name: `lb` (to create `lb.internal.example.com`)
  - Value: Internal IP of the load balancer.
  - Ensure this IP is reachable within your private network (e.g., VPC or private droplets).


### Step 5: Update Ingress Controller with Load Balancer Domain Name
Configure your load balancer with a custom hostname and name by adding the following annotations to your Helm `values.yaml` file under `controller.service.annotations`:
```yaml
controller:
  service:
    annotations:
      service.beta.kubernetes.io/do-loadbalancer-hostname: lb.internal.chronika.fun
      service.beta.kubernetes.io/do-loadbalancer-name: lb-internal
```
> These annotations instruct DigitalOcean to assign the specified hostname and internal resource name to the created Load Balancer.
{: .prompt-info }

Then install or upgrade the Ingress Controller with the updated values:
```sh
helm upgrade --install ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx  \
    --namespace ingress-nginx \
    -f values.yaml
```


### Step 6: Creating a ClusterIssuer with DNS-01 Solver
Create a `ClusterIssuer` that uses the DigitalOcean DNS challenge. Note that ClusterIssuers are cluster-scoped so the namespace field is omitted:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-internal-lb
spec:
  acme:
    email: your_email@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-internal-lb-secret
    solvers:
      - dns01:
          digitalocean:
            tokenSecretRef:
              name: lets-encrypt-do-dns
              key: access-token
```
- `privateKeySecretRef` is name of the secret that Cert-Manager will use to store the ACME account private key used for issuing certificates.
- `dns01.digitalocean.tokenSecretRef.name` is the secret created in last step and `key` is the key in the secret which holds base64 encoded personal access token.

Apply the ClusterIssuer:
```sh
kubectl apply -f letsencrypt-cluster-issuer.yaml -n cert-manager
```

### Step 7: Create Ingress for Example Service
Now tie together the components by creating an Ingress resource that directs traffic to your example service. Make sure the hostname is consistent in both the TLS configuration and the rules.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress-http
  namespace: example
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-internal-lb"
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - lb.internal.example.com
    secretName: example-service-cert
  rules:
    - host: lb.internal.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: http-echo
                port:
                  number: 80
  ingressClassName: nginx
```
- `cert-manager.io/cluster-issuer` is set to `letsencrypt-internal-lb` to tell cert-manager the issuer we'd like to use when issuing secure certificates for this Ingress.
- The `secretName` is the Kubernetes Secret where cert-manager will put the issued secure certificate, and the Secret `ingress-nginx` will use to load the issued certificate.

Deploy the Ingress resource:
```sh
kubectl apply -f example-ingress-http.yaml -n example
```

Verify that the certificate is created and the certificate’s status is marked as `Ready`:
```sh
kubectl get certificate -n example
``` 

## Next
With internal ingress and certificate automation in place, your services are now securely reachable within your infrastructure. But TLS is still terminated at the edge.

In [Part 3](../Breaking-Down-Our-Server-Architecture-Part-3), we’ll take the next step:
- Proxy traffic from the edge NGINX to the internal ingress over HTTPS.
- Validate backend certificates for end-to-end TLS.
- Preserve client IPs using forwarded headers.
- This closes the loop on a fully encrypted, secure internal architecture.

## References
1. [How to Create a Personal Access Token.](https://docs.digitalocean.com/reference/api/create-personal-access-token/)
2. [How to Secure Your Site in K8s with cert-manager.](https://www.digitalocean.com/community/tutorials/how-to-secure-your-site-in-kubernetes-with-cert-manager-traefik-and-let-s-encrypt)
3. [Cert-Manager DNS-01 Challenge Docs](https://cert-manager.io/docs/configuration/acme/dns01/)
