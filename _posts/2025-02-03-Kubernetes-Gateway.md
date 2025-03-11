---
title: Kubernetes Gateway
date: 2025-02-04 07:00:00 +0530
categories: [Kubernetes, Ingress]
tags: [kubernetes, networking, ingress]
description: "The Kubernetes Gateway API simplifies traffic management in your cluster by offering improved routing and clear roles for platform engineers and developers."
---

## What is Kubernetes Gateway?
- The Kubernetes Gateway API (or "Gateway API") is a standard specification for configuring and managing application traffic routing within a Kubernetes cluster.
- It is designed to be the successor to the Ingress API, providing a more flexible and robust way to handle traffic routing.

<br>

## What is Ingress API?
- The Ingress API is a built-in Kubernetes feature that helps route external HTTP and HTTPS traffic to services inside a cluster.
- It is widely used and supported by many vendors, each offering their own implementations.

<br>

## Why Switch to the Gateway API?
- **Limited Features in Ingress**: Ingress primarily supports TLS termination and basic, content-based HTTP routing. Advanced routing features require non-standard, vendor-specific annotations.
- **Advanced Routing Needs**: No standard configuration for advanced routing features, which can only be achieved through non-portable annotations. Every implementation has its own supported extensions that may not translate to any other implementation. For example, achieving URL redirection with the NGINX Ingress Controller uses the `nginx.ingress.kubernetes.io/rewrite-target` annotation, which isn’t compatible with other proxies like Envoy. 
- **Clearer Responsibilities**: With Ingress, tasks such as setting up and managing gateways (which are more relevant to platform engineering) are often handled by application developers. The Gateway API helps delineate these responsibilities.

<br>

## Gateway API Resource Model

<img src="/assets/img/posts/gateway-resource-model.png" alt="Gateway Resource Model" width="450" height="450" style="border: 4px solid grey;">

### Gateway Class
- Defines a group of Gateways that share the same configuration and behavior.
- Each GatewayClass is managed by a single controller, though a controller may handle multiple GatewayClasses.
- Since GatewayClass is a cluster-wide resource, at least one must be defined for any Gateway to function.
- This is similar to [IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class) for Ingress and [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) for PersistentVolumes.

### Gateway
- A Gateway defines how traffic is translated to Services within the cluster. 
- In simple terms, it sets up a mechanism to take traffic from a source that doesn't understand Kubernetes—like a cloud load balancer, an in-cluster proxy, or even external hardware—and direct it to a Kubernetes Service. Although many scenarios involve traffic coming from outside the cluster, that isn’t a strict requirement.
- The Gateway essentially requests a specific load balancer configuration that follows the rules and behavior defined by its GatewayClass. The gateway resource can be created either directly by an operator or automatically by a controller that manages the GatewayClass.
- Additionally, a Gateway can be linked to one or more Routes, which determine how specific portions of the traffic should be directed to particular services.

### Route Resources
- Define the rules for routing incoming traffic from a Gateway to the appropriate Kubernetes Services.
- You can specify rules based on protocols (HTTP, HTTPS, TCP), URL paths, host names, and other attributes.
- Support functionalities like traffic splitting, retries, and header modifications, offering more control compared to the traditional Ingress resource.





