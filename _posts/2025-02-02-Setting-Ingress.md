---
title: Practical Guide to Setting Up ingress-nginx on Kubernetes
date: 2025-02-02 07:00:00 +0530
categories: [Kubernetes, Ingress]
tags: [kubernetes, networking, ingress]
description: "This post walks you through setting up the ingress-nginx controller on your Kubernetes cluster, deploying a minimal backend application, and creating an Ingress resource to route external traffic to your backend."
mermaid: true
---

## Prerequisites

#### Deploy Example Service
Follow the instructions from [Setting Up an Example Service](../Example-Service/) to set up an example kubernetes service.

> Deploying an example service helps verify that the Ingress controller correctly routes traffic to a live service.
{: .prompt-info }


#### Install Cert-Manager
Follow installation instructions from [Cert-Manager post](../Cert-Manager/) to install cert-manager.

> cert-manager is crucial for securing your Ingress traffic with TLS. (The ClusterIssuer will be created in a later step.)
{: .prompt-info }


## Ingress Controller Setup
Follow these steps to deploy the Ingress-NGINX controller and configure DNS routing.

1. **Deploy Ingress-NGINX Controller**
- Run the following command to install or upgrade the Ingress-NGINX controller using Helm.
```sh
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
- This command creates a new namespace (ingress-nginx) and deploys the Ingress-NGINX controller within it.

2. **Retrieve the External IP of the Load Balancer**
- Once the controller is deployed, check the external IP assigned to the ingress-nginx-controller service.
- Look for the `EXTERNAL-IP` under the LoadBalancer type service. It may take a few moments to be assigned.
```sh
kubectl get svc -n ingress-nginx
```

3. **Map the External IP to a DNS Name**: 
- Use your preferred DNS provider (e.g., Cloudflare, AWS Route 53, Google Domains) to create an `A record` pointing to the retrieved `External IP`.
- For exact instructions, refer to your DNS provider’s documentation.

4. **Verification**:
- Once updated, verify that the domain is correctly resolving to the External IP by running
```sh
nslookup app.example.com # replace this with your dns.
# OR
dig app.example.com # replace this with your dns
```

## Configuring Cluster Issuer
Configure your ClusterIssuer by following the instructions in the [Cert-Manager post](../Cert-Manager/). This issuer will be used later to generate TLS certificates for your Ingress.

## Deploy TLS-Enabled Ingress Resource
Create an Ingress resource to route traffic to your service and enable TLS.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: http-echo-ingress
  namespace: example
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-cluster-issuer" # name of the cluster issuer.
    kubernetes.io/ingress.class: "nginx"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com # replace with your domain name.
    secretName: http-echo-staging-tls
  rules:
  - host: app.example.com # replace with your domain name.
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: http-echo
            port:
              number: 80
```

- Apply the ingress resource.

```sh
  kubectl apply -f ingress.yaml
```

## Verification
Test the Ingress configuration by accessing your service:

```sh
curl https://app.example.com -v # replace with your domain name.
```

If everything is set up correctly, you should receive a `hello-world` response from the server.

## References
1. [Ingress-Nginx documentation](https://kubernetes.github.io/ingress-nginx/).
2. [Cert-Manager documentation](https://cert-manager.io/docs/).