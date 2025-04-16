---
title: Cert Manager
date: 2025-02-03 07:00:00 +0530
categories: [Kubernetes, Tools]
tags: [kubernetes, certificate]
description: "Cert-manager is a powerful Kubernetes add-on that automates the issuance and renewal of TLS certificates. In this hands-on guide, you'll learn how to install cert-manager, configure it to work with a certificate authority like Let's Encrypt, and integrate it with your Kubernetes workloads."
---

## Why do we need Cert-Manager?
- Cert-manager automates the process of issuing, renewing, and managing TLS certificates, saving you from manual certificate handling.
- It works natively within Kubernetes, automatically storing certificates as secrets and integrating with Ingress resources for secure HTTPS endpoints.
- Certificates have expiration dates. Cert-manager ensures certificates are renewed in time, reducing the risk of service interruptions due to expired certificates.
- With cert-manager, you can easily use various certificate authorities (like Let's Encrypt), ensuring your applications maintain a strong security posture with minimal effort.

<br>

## Installation
1. Add the helm repository.
```sh
helm repo add jetstack https://charts.jetstack.io --force-update
```

2. Install cert-manager. 
  - Other installation options are mentioned [here](https://cert-manager.io/docs/installation/helm/#installation-options).
  - This command will create a new namespace `cert-manager` and install cert-manager in it.
```sh
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true
```

3. Verify Installation
```sh
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
kind: ClusterIssuer
metadata:
  name: letsencrypt-cluster-issuer
  namespace: cert-manager
spec:
  acme:
    # Email - replace this with your email id.
    email: example@gmail.com 
    # Server - the URL used to access the ACME server’s directory endpoint.
    # Prod server - https://acme-v02.api.letsencrypt.org/directory.
    # Stage server - https://acme-staging-v02.api.letsencrypt.org/directory
    # check https://letsencrypt.org/docs/staging-environment/ for more info.
    server: https://acme-v02.api.letsencrypt.org/directory 
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-production
    # Enable the HTTP-01 challenge provider
    solvers:
      # Use the HTTP-01 challenge provider
      - http01:
          ingress:
            class: nginx

```

- Apply the custom resource.
```sh
kubectl apply -f cluster-issuer.yaml
```

- Verify the status of the issuer after creating
```sh
kubectl describe clusterissuer letsencrypt-cluster-issuer -n cert-manager
```

## References
1. [Cert-Manager documentation](https://cert-manager.io/docs/).
2. [Practical Guide to Setting Up ingress-nginx on Kubernetes](../Setting-Ingress).