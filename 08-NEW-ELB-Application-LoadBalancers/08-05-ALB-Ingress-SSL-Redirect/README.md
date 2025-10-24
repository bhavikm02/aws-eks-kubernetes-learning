---
title: AWS Load Balancer - Ingress SSL HTTP to HTTPS Redirect
description: Learn AWS Load Balancer - Ingress SSL HTTP to HTTPS Redirect
---

## SSL Redirect Architecture Diagram

```mermaid
graph TB
    USER[Client Browser] -->|HTTP Request<br/>Port 80| ALB[Application Load Balancer]
    USER -->|HTTPS Request<br/>Port 443| ALB
    
    ALB -->|HTTP Listener| HTTP_RULE{Listener Rule}
    ALB -->|HTTPS Listener| HTTPS_RULE{Listener Rule}
    
    HTTP_RULE -->|301/302 Redirect| REDIRECT[Redirect Action<br/>Location: https://#{host}:443/#{path}?#{query}]
    
    REDIRECT -->|Browser Follows| USER
    USER -->|New HTTPS Request| ALB
    
    HTTPS_RULE -->|TLS Termination| DECRYPT[Decrypt with ACM Cert]
    DECRYPT -->|Forward to Backend| TG[Target Group<br/>Worker Nodes]
    
    subgraph "EKS Cluster"
        TG -->|NodePort| PODS[Application Pods]
        
        ING[Ingress Resource]
        SVC[NodePort Service]
        
        ING -->|Backend| SVC
        SVC -.->|Selects| PODS
    end
    
    subgraph "Ingress Annotations"
        ANNO1[alb.ingress.kubernetes.io/listen-ports<br/>[HTTPS:443, HTTP:80]]
        ANNO2[alb.ingress.kubernetes.io/certificate-arn<br/>ACM Certificate]
        ANNO3[alb.ingress.kubernetes.io/ssl-redirect: '443'<br/>Enable HTTP → HTTPS]
    end
    
    ING --> ANNO1
    ING --> ANNO2
    ING --> ANNO3
    
    subgraph "Redirect Flow"
        REQ1[1. Client sends HTTP request<br/>http://example.com/app1]
        REQ2[2. ALB HTTP listener receives]
        REQ3[3. ALB returns 301/302 redirect<br/>Location: https://example.com/app1]
        REQ4[4. Browser follows redirect]
        REQ5[5. Client sends HTTPS request]
        REQ6[6. ALB HTTPS listener receives]
        REQ7[7. ALB decrypts & forwards]
        REQ8[8. Pod receives plain HTTP]
        
        REQ1 --> REQ2 --> REQ3 --> REQ4 --> REQ5 --> REQ6 --> REQ7 --> REQ8
    end
    
    subgraph "Redirect Types"
        PERM[301 Permanent Redirect<br/>Browser caches]
        TEMP[302 Temporary Redirect<br/>No browser cache]
        DEFAULT[Default: 301 for SSL redirect]
    end
    
    subgraph "Benefits"
        B1[Enforce HTTPS for all traffic]
        B2[Automatic redirect handling]
        B3[No app code changes needed]
        B4[SEO friendly 301 redirects]
        B5[Browser security indicators]
        B6[Data encryption in transit]
    end
    
    subgraph "ALB Listener Rules"
        RULE80[Port 80 Listener<br/>Action: Redirect to HTTPS]
        RULE443[Port 443 Listener<br/>Action: Forward to Target Group]
    end
    
    HTTP_RULE -.->|Configured as| RULE80
    HTTPS_RULE -.->|Configured as| RULE443
    
    subgraph "Security Headers"
        HSTS[Strict-Transport-Security<br/>HSTS enforcement]
        SECURE[Secure cookie flags]
        CSP[Content-Security-Policy]
    end
    
    style ALB fill:#FF9900
    style REDIRECT fill:#FF6B6B
    style ING fill:#2E8B57
    style DECRYPT fill:#4A90E2
    style ANNO3 fill:#FFD700
```

### Diagram Explanation

- **SSL Redirect Annotation**: `alb.ingress.kubernetes.io/ssl-redirect: '443'` configures **automatic HTTP to HTTPS redirect** on ALB
- **301 Permanent Redirect**: ALB returns **301 status code** by default, browsers cache the redirect for future requests
- **Transparent to Application**: Redirect happens at **ALB layer**, application code doesn't need modification
- **Both Listeners Required**: Must configure **both HTTP (80) and HTTPS (443)** listeners for redirect to work
- **Redirect Preserves Path**: Original **URL path and query parameters** maintained in redirected HTTPS request
- **SEO Benefits**: **301 redirects** properly signal to search engines that HTTPS is the canonical version
- **Browser Behavior**: After redirect, browser **security indicators** (padlock icon) show connection is secure
- **HSTS Consideration**: Can add **Strict-Transport-Security** header for additional security, forces HTTPS on subsequent visits
- **Certificate Required**: SSL redirect requires **valid ACM certificate** configured for HTTPS listener
- **Cost Neutral**: Redirect action is **free**, no additional ALB charges for redirect traffic

## Step-01: Add annotations related to SSL Redirect
- **File Name:** 04-ALB-Ingress-SSL-Redirect.yml
- Redirect from HTTP to HTTPS
```yaml
    # SSL Redirect Setting
    alb.ingress.kubernetes.io/ssl-redirect: '443'   
```

## Step-02: Deploy all manifests and test

### Deploy and Verify
```t
# Deploy kube-manifests
kubectl apply -f kube-manifests/

# Verify Ingress Resource
kubectl get ingress

# Verify Apps
kubectl get deploy
kubectl get pods

# Verify NodePort Services
kubectl get svc
```
### Verify Load Balancer & Target Groups
- Load Balancer -  Listeneres (Verify both 80 & 443) 
- Load Balancer - Rules (Verify both 80 & 443 listeners) 
- Target Groups - Group Details (Verify Health check path)
- Target Groups - Targets (Verify all 3 targets are healthy)
 
## Step-03: Access Application using newly registered DNS Name
- **Access Application**
```t
# HTTP URLs (Should Redirect to HTTPS)
http://ssldemo101.stacksimplify.com/app1/index.html
http://ssldemo101.stacksimplify.com/app2/index.html
http://ssldemo101.stacksimplify.com/

# HTTPS URLs
https://ssldemo101.stacksimplify.com/app1/index.html
https://ssldemo101.stacksimplify.com/app2/index.html
https://ssldemo101.stacksimplify.com/
```

## Step-04: Clean Up
```t
# Delete Manifests
kubectl delete -f kube-manifests/

## Delete Route53 Record Set
- Delete Route53 Record we created (ssldemo101.stacksimplify.com)
```

## Annotation Reference
- [AWS Load Balancer Controller Annotation Reference](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)



