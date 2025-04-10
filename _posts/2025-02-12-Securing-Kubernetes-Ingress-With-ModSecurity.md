---
title: Securing Kubernetes Ingress with ModSecurity
date: 2025-02-12 07:00:00 +0530
categories: [Kubernetes, Ingress]
tags: [kubernetes, networking, ingress, waf, security]
description: "This post explains how to set up ModSecurity with Kubernetes Ingress to protect web applications from common threats. It covers enabling a Web Application Firewall (WAF) using ModSecurity, configuring security rules, and testing the setup with practical examples."
mermaid: true
---

## What is a WAF?
- A WAF or **Web Application Firewall** helps protect web applications by filtering and monitoring HTTP traffic between a web application and the Internet.
- Operating at the **application layer (Layer 7 of the OSI model)**, a WAF primarily addresses threats like SQL injection, XSS, and other application-layer attacks but is not intended for network-level defense.
- A WAF operates through a set of rules often called policies. These policies aim to protect against vulnerabilities in the application by filtering out malicious traffic.

## What is ModSecurity and OWASP?
- [OWASP](https://github.com/owasp) or **Open Worldwide Application Security Project** is a nonprofit organization focused on improving software security.
    - Best known for the OWASP Top 10, a list of the most critical web application security risks.
    - The OWASP ModSecurity [Core Rule Set](https://github.com/coreruleset/coreruleset) is the de facto set of free and open source WAF rules used around the world. 
- [ModSecurity](https://github.com/owasp-modsecurity/ModSecurity) is an open source, cross-platform web application firewall (WAF) module.
    - Compatible with popular web servers like Apache, NGINX, and IIS.
    - Operated under the custodianship of OWASP, it is frequently paired with the OWASP CRS for enhanced protection.

## Ingress NGINX with ModSecurity
When installing Ingress NGINX using Helm charts, you can pass the ModSecurity configuration values along with your deployment. Follow these steps to set up and customize your WAF rules.

#### Create modsecurity-values.yaml
Create a YAML file containing the ModSecurity directives. This snippet enables prevention mode, logs via stdout, and sets up content-type processing for XML, JSON, and GRPC formats.

```yaml
controller:
  service:
    type: LoadBalancer
  config:
    enable-modsecurity: "true" 
    enable-owasp-modsecurity-crs: "false"
    use-forwarded-headers: "true"
    # Update ModSecurity config and rules
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

> When setting both **enable-owasp-modsecurity-crs: true** and **enable-modsecurity: true**, the former overrides the recommended set of [ModSecurity settings](https://github.com/owasp-modsecurity/ModSecurity/blob/v3/master/modsecurity.conf-recommended).
{: .prompt-info }

> Currently there is no provision for ModSecurity to [parse through GRPC request](https://github.com/owasp-modsecurity/ModSecurity/issues/2645) body as it can do with JSON and XML. Hence, in the above configuration requests are just let in without processing. 
{: .prompt-warning }

#### Deploy Ingress NGINX with the ModSecurity Values
Upgrade the Ingress NGINX deployment using Helm, passing the values file:

```sh
helm upgrade --install ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx  \
    --namespace ingress-nginx \
    --set controller.config.enable-modsecurity=true \
    --set controller.config.enable-owasp-modsecurity-crs=true \
    -f modsecurity-values.yaml 
```


### Externalizing ModSecurity Configuration with a ConfigMap
The modsecurity-snippet field used to configure and tweak ModSecurity and the CRS defaults has a limit of 4096 characters. To avoid this we can extract the configuration to a separate ConfigMap.

#### Create a ConfigMap (`modsecurity-config.yaml`)
Create a new ConfigMap containing the ModSecurity directives as a single file property.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "modsecurity-config"
  namespace: ingress-nginx
data:
  custom-modsecurity.conf: |    
    # Enable prevention mode. Options: DetectionOnly, On, Off (default is DetectionOnly)
    SecRuleEngine On

    # Enable scanning of the request body
    SecRequestBodyAccess On

    # Include additional custom configurations as necessary...
```

#### Mount the ConfigMap as a volume to the ingress nginx controller.
- Use the **modsecurity-snippet** to include the ConfigMap data mounted as a volume.
- The exact path depends on the volume mount and the name of the property inside the ConfigMap.
```yaml
controller:
  service:
    type: LoadBalancer
  config:
    enable-modsecurity: "true" 
    enable-owasp-modsecurity-crs: "true"
    modsecurity-snippet: |  
      # Increment this to force nginx to reload the rules when you change the configmap: 1.0.0
      Include /etc/nginx/owasp-modsecurity-crs/custom/custom-modsecurity.conf

      # Enable prevention mode. Options: DetectionOnly,On,Off (default is DetectionOnly)
      SecRuleEngine On

      # Reject if larger (we could also let it pass with ProcessPartial)
      SecRequestBodyLimitAction Reject

      # Send ModSecurity audit logs to the stdout (only for rejected requests)
      SecAuditLog /dev/stdout

      # Format the logs in JSON
      SecAuditLogFormat JSON

      # Could be On/Off/RelevantOnly
      SecAuditEngine RelevantOnly
    
  extraVolumeMounts:
    - name: modsecurity-config
      mountPath: /etc/nginx/owasp-modsecurity-crs/custom/

  extraVolumes:
    - name: modsecurity-config
      configMap:
        name: modsecurity-config
```

#### Upgrade Ingress NGINX with the Updated Configuration
Apply the updated Helm values with the mounted ConfigMap:
```sh
helm upgrade --install ingress-nginx ingress-nginx \
    --repo https://kubernetes.github.io/ingress-nginx  \
    --namespace ingress-nginx \
    --set controller.config.enable-modsecurity=true \
    --set controller.config.enable-owasp-modsecurity-crs=true \
    -f modsecurity-values.yaml 
```

### Future
The Ingress NGINX project has [announced plans](https://github.com/kubernetes/ingress-nginx/issues/11668) to deprecate and eventually remove ModSecurity support in future releases. This is due to challenges maintaining and integrating ModSecurity reliably within the project.

## References
- [Kubernetes NGINX Ingress WAF with ModSecurity](https://systemweakness.com/nginx-ingress-waf-with-modsecurity-from-zero-to-hero-fa284cb6f54a)
- [Ingress Nginx Documentation](https://kubernetes.github.io/ingress-nginx/user-guide/third-party-addons/modsecurity/)
- [Misleading Documentation Issue](https://github.com/kubernetes/ingress-nginx/issues/8388)

