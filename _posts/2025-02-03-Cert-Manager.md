---
title: Cert Manager
date: 2025-02-03 07:00:00 +0530
categories: [Kubernetes, Tools]
tags: [kubernetes]
description: "Cert-manager is a powerful Kubernetes add-on that automates the issuance and renewal of TLS certificates. In this hands-on guide, you'll learn how to install cert-manager, configure it to work with a certificate authority like Let's Encrypt, and integrate it with your Kubernetes workloads."
---

## Why do we need Cert-Manager?
- Cert-manager automates the process of issuing, renewing, and managing TLS certificates, saving you from manual certificate handling.
- It works natively within Kubernetes, automatically storing certificates as secrets and integrating with Ingress resources for secure HTTPS endpoints.
- Certificates have expiration dates. Cert-manager ensures certificates are renewed in time, reducing the risk of service interruptions due to expired certificates.
- With cert-manager, you can easily use various certificate authorities (like Let's Encrypt), ensuring your applications maintain a strong security posture with minimal effort.

<br>

## Pre-requisite
### Ingress Controller
1. **Deploy Ingress-NGINX Controller**
- Run the following command to install or upgrade the Ingress-NGINX controller using Helm:
```sh
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

2. **Retrieve the External IP of the Load Balancer**
- Once the controller is deployed, check the external IP assigned to the ingress-nginx-controller service.
- Look for the EXTERNAL-IP under the LoadBalancer type service. It may take a few moments to be assigned.
```sh
kubectl get svc -n ingress-nginx
```

3. **Map the External IP to a DNS Name**: 
- Use your preferred DNS provider (e.g., Cloudflare, AWS Route 53, Google Domains) to create an A record pointing to the retrieved external IP.
- For exact instructions, refer to your DNS provider’s documentation.

4. **Verification**:
- Once updated, verify that the domain is correctly resolving to the external IP by running
```sh
nslookup app.example.com
# OR
dig app.example.com
```

### Deploy a Example Service
1. Create a dedicated namespace for the example service:.

    ```yaml
    kubectl create ns example
    ```

2. Create a YAML file for the deployment based on [http-echo](https://github.com/hashicorp/http-echo).

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: http-echo
      namespace: example
    spec:
      selector:
        matchLabels:
          app: http-echo
      replicas: 1
      template:
        metadata:
          labels:
            app: http-echo
        spec:
          containers:
          - image: hashicorp/http-echo:1.0
            imagePullPolicy: Always
            name: http-echo
            ports:
            - containerPort: 5678
    ```

    - Apply the deployment:

    ```sh
    kubectl apply -f http-echo-deployment.yaml
    ```

3. Create a YAML file (e.g., http-echo-service.yaml) for the service that exposes the deployment

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: http-echo
      namespace: example
    spec:
      ports:
      - port: 80
        targetPort: 5678
        protocol: TCP
      selector:
        app: echo
    ```

    - Apply the deployment:

    ```sh
    kubectl apply -f http-echo-service.yaml
    ```

<br>

## Installation
1. Add the helm repository.
```
helm repo add jetstack https://charts.jetstack.io --force-update
```

2. Install cert-manager. Other installation options are mentioned [here](https://cert-manager.io/docs/installation/helm/#installation-options).
```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true
```

3. Verify Installation
```
helm ls -n cert-manager
```

<br>

## Configuring Issuers
- Issuers and ClusterIssuers are custom resources provided by cert-manager that define how TLS certificates are issued and managed. 
- They configure the details needed to obtain and sign certificates, such as which certificate authority (CA) or signing method to use.
- **Issuer** is a namespaced resource. An Issuer can only be used to issue certificates for resources within its own namespace.
- **ClusterIssuer** is a cluster-wide resource and is not bound to any single namespace.

#### ACME
- ACME (Automatic Certificate Management Environment) is a protocol used to automate the process of obtaining, renewing, and managing TLS certificates.
- When you configure an Issuer or ClusterIssuer to use ACME, cert-manager communicates with an ACME server to prove domain ownership and then automatically obtain the corresponding certificate.

- Create this definition locally and update the email address to your own. This email is required by Let's Encrypt and used to notify you of certificate expiration and updates.

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: letsencrypt-staging
    spec:
      acme:
        # The ACME server URL
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        # Email address used for ACME registration
        email: user@example.com
        # Name of a secret used to store the ACME account private key
        privateKeySecretRef:
          name: letsencrypt-staging
        # Enable the HTTP-01 challenge provider
        solvers:
          - http01:
              ingress:
                ingressClassName: nginx
    ```

- Apply the custom resource:

    ```yaml
    kubectl apply -f staging-cluster-issuer.yaml
    ```

- Verify the status of the issuer after creating

    ```yaml
    kubectl describe clusterissuer letsencrypt-staging -n cert-manager
    ```

## Deploy TLS-Enabled Ingress Resource
- Create an Ingress resource with TLS enabled.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: kuard
      annotations:
        cert-manager.io/issuer: "letsencrypt-staging"
    spec:
      ingressClassName: nginx
      tls:
      - hosts:
        - example.example.com
        secretName: quickstart-example-tls
      rules:
      - host: example.example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kuard
                port:
                  number: 80
    ```

- Apply the ingress resource

    ```yaml
    kubectl apply -f ingress.yaml
    ```