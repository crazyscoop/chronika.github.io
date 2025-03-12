---
title: Setting Up an Example Service
date: 2025-02-04 07:00:00 +0530
categories: [Kubernetes, Basics]
tags: [kubernetes]
description: "In this guide, we'll walk through setting up an example service in Kubernetes, covering key concepts like deployments, services, and pods."
---

## Prerequisites
Before you begin, ensure you have the following installed:
- kubectl: The Kubernetes command-line tool.
- k8s cluster: Kubernetes cluster such as Minikube, Google Kubernetes Engine (GKE), Amazon Elastic Kubernetes Service (EKS), or Azure Kubernetes Service (AKS).


## Namespace
- Create a dedicated namespace for the example service.

```sh
kubectl create ns example
```


## Deployment
- Create a YAML file for the deployment based on [http-echo](https://github.com/hashicorp/http-echo).

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

## Service
- Create a YAML file (e.g., http-echo-service.yaml) for the service that exposes the deployment

```yaml
apiVersion: v1
kind: Service
metadata:
  name: http-echo
  namespace: example
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 5678
  selector:
    app: http-echo
```

- Apply the deployment:

```sh
kubectl apply -f http-echo-service.yaml
```

## Access the Service
#### Using Port Forwarding
- Run the following command to forward a local port to your service. This allows you to access the service locally without exposing it externally:
  ```sh
  kubectl port-forward svc/http-echo -n example 8080:80
  ```
	- This command forwards port 8080 on your local machine to port 80 on the service in the example namespace.

-  Curl to access the service:
  ```sh
  curl http://localhost:8080 -v
  ```
	- You should get a `hello-world` response from the server.

## References
- Docker image for [hashicorp/http-echo](https://hub.docker.com/r/hashicorp/http-echo/tags)
- Github repository for [http-echo](https://hub.docker.com/r/hashicorp/http-echo/tags)
