---
title: Practical Guide to Setting Up ingress-nginx on Kubernetes
date: 2025-02-02 07:00:00 +0530
categories: [Kubernetes, Ingress]
tags: [kubernetes, networking, ingress]
description: "This post walks you through setting up the ingress-nginx controller on your Kubernetes cluster, deploying a minimal backend application, and creating an Ingress resource to route external traffic to your backend."
mermaid: true
---

## Pre-Requisite

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

### Install Cert-Manager
Follow installation instructions from [Cert-Manager post](../Cert-Manager/#installation/) to install cert-manager and issuer.

