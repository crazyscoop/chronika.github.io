---
title: Breaking Down Our Server Architecture; A Deep Dive into Secure Traffic Flow
date: 2025-02-14 07:00:00 +0530
categories: [Architecture, Chronika]
tags: [kubernetes, networking, ingress, waf, security]
description: "In this series, we'll break down our multi-layered server architecture that efficiently handles client requests, secures data with TLS, and simplifies certificate management. We will then walk through the step-by-step implementation of this architecture."
mermaid: true
---

## Introduction
This series explores how we architect secure, scalable traffic flow from the public internet to internal Kubernetes services. Our setup uses NGINX at the edge, Ingress NGINX inside the cluster, Cert-Manager for automated TLS, ModSecurity for WAF protection, and DigitalOcean DNS for internal domain delegation.

Each part walks through a specific layer, from TLS termination at the edge to encrypted communication within your infrastructure.

### Series Overview

#### [Part 1: Setting Up The Edge with NGINX and Certbot](../Breaking-Down-Our-Server-Architecture-Part-1)
We begin by setting up edge NGINX to handle TLS termination using Certbot and Let’s Encrypt. It enforces HTTPS and handles certificate renewal, ensuring all traffic entering your system is encrypted.

#### [Part 2: Setting up Ingress with Ingress Nginx](../Breaking-Down-Our-Server-Architecture-Part-2)
Next, we deploy the Ingress NGINX controller in Kubernetes, automate internal TLS with Cert-Manager and DNS-01 challenges, and use ModSecurity as a WAF to inspect and block threats at the ingress level.

#### [Part 3: Securely Bridging Edge and Internal with TLS](../Breaking-Down-Our-Server-Architecture-Part-3)
Finally, we secure the link between the edge and internal ingress using HTTPS. We validate backend certificates, preserve client IPs, and run ModSecurity in prevention mode to complete end-to-end encryption across the stack.

