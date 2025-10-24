---
title: AWS Load Balancer Controller - Ingress Host Header Routing
description: Learn AWS Load Balancer Controller - Ingress Host Header Routing
---

## Host Header Based Routing Diagram

```mermaid
graph TB
    USER1[Client: app101.example.com] -->|Host Header| ALB[Application Load Balancer<br/>Single ALB]
    USER2[Client: app201.example.com] -->|Host Header| ALB
    USER3[Client: default101.example.com] -->|Host Header| ALB
    
    ALB -->|Check Host Header| RULES{Routing Rules}
    
    RULES -->|host: app101.example.com| TG1[Target Group 1<br/>App1 Pods]
    RULES -->|host: app201.example.com| TG2[Target Group 2<br/>App2 Pods]
    RULES -->|default or no match| TG3[Target Group 3<br/>App3 Pods<br/>Default Backend]
    
    subgraph "EKS Cluster"
        subgraph "App1 Service"
            SVC1[NodePort Service<br/>app1-nginx-nodeport-service]
            POD1[App1 Pods]
            SVC1 -.->|Selects| POD1
        end
        
        subgraph "App2 Service"
            SVC2[NodePort Service<br/>app2-nginx-nodeport-service]
            POD2[App2 Pods]
            SVC2 -.->|Selects| POD2
        end
        
        subgraph "App3 Service"
            SVC3[NodePort Service<br/>app3-nginx-nodeport-service]
            POD3[App3 Pods]
            SVC3 -.->|Selects| POD3
        end
        
        ING[Ingress: ingress-namedbasedvhost-demo]
        ING -->|rules[0].host| SVC1
        ING -->|rules[1].host| SVC2
        ING -->|defaultBackend| SVC3
    end
    
    TG1 -->|NodePort| SVC1
    TG2 -->|NodePort| SVC2
    TG3 -->|NodePort| SVC3
    
    subgraph "Ingress Rules Configuration"
        RULE1[rules[0]:<br/>host: app101.example.com<br/>backend: app1-service]
        RULE2[rules[1]:<br/>host: app201.example.com<br/>backend: app2-service]
        DEFAULT[defaultBackend:<br/>app3-service<br/>Catch-all]
    end
    
    ING --> RULE1
    ING --> RULE2
    ING --> DEFAULT
    
    subgraph "External DNS Integration"
        EXTDNS[External DNS Pod]
        EXTDNS -->|Read annotations| ING
        EXTDNS -->|Create DNS Records| R53[Route53 Hosted Zone]
        
        R53 --> DNS1[A Record: default101.example.com → ALB]
        R53 --> DNS2[A Record: app101.example.com → ALB]
        R53 --> DNS3[A Record: app201.example.com → ALB]
    end
    
    subgraph "Host Header Matching"
        HTTP_REQ[HTTP Request Headers]
        HTTP_REQ --> HOST_HDR[Host: app101.example.com]
        HTTP_REQ --> METHOD[GET /index.html]
        HTTP_REQ --> OTHER[Other headers...]
        
        HOST_HDR -->|ALB extracts| MATCH[Match against rules]
    end
    
    subgraph "SSL Configuration"
        SSL_LISTEN[Listen Ports: HTTPS:443, HTTP:80]
        SSL_CERT[Certificate ARN from ACM]
        SSL_REDIR[SSL Redirect: 443<br/>HTTP → HTTPS]
    end
    
    ING --> SSL_LISTEN
    ING --> SSL_CERT
    ING --> SSL_REDIR
    
    subgraph "Benefits of Name-Based Routing"
        B1[Multiple apps on single ALB]
        B2[Cost-effective vs separate LBs]
        B3[Domain-based isolation]
        B4[Easy to add new apps]
        B5[Simplified DNS management]
    end
    
    style ALB fill:#FF9900
    style ING fill:#2E8B57
    style TG1 fill:#4A90E2
    style TG2 fill:#4A90E2
    style TG3 fill:#FFD700
    style R53 fill:#9B59B6
```

### Diagram Explanation

- **Host Header Routing**: ALB routes requests to different **backend services** based on HTTP Host header value
- **Name-Based Virtual Hosts**: Similar to Apache/Nginx **vhosts**, multiple domains share single ALB, routed by hostname
- **Ingress Rules**: Each rule specifies **host field** matching domain name, directs to specific backend service
- **Default Backend**: Catches **unmatched hosts** or requests without Host header, serves fallback application
- **External DNS Integration**: Automatically creates **Route53 A records** for each hostname pointing to ALB DNS
- **SSL/TLS per Host**: Single **ACM certificate** (wildcard) covers all subdomains, ALB handles TLS termination
- **Cost Optimization**: **One ALB** serves multiple domains, cheaper than creating separate load balancers per app
- **SNI Support**: ALB uses **Server Name Indication** to present correct certificate for each hostname
- **Priority Matching**: Rules evaluated in **order defined**, more specific hosts before catch-all default backend
- **Multi-Tenant Architecture**: Different applications/customers accessible via **different domains** on same infrastructure

## Step-01: Introduction
- Implement Host Header routing using Ingress
- We can also call it has name based virtual host routing

## Step-02: Review Ingress Manifests for Host Header Routing
- **File Name:** 04-ALB-Ingress-HostHeader-Routing.yml
```yaml
# Annotations Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-namedbasedvhost-demo
  annotations:
    # Load Balancer Name
    alb.ingress.kubernetes.io/load-balancer-name: namedbasedvhost-ingress
    # Ingress Core Settings
    #kubernetes.io/ingress.class: "alb" (OLD INGRESS CLASS NOTATION - STILL WORKS BUT RECOMMENDED TO USE IngressClass Resource)
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Health Check Settings
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP 
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    #Important Note:  Need to add health check path annotations in service level if we are planning to use multiple targets in a load balancer    
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/success-codes: '200'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'   
    ## SSL Settings
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:180789647333:certificate/632a3ff6-3f6d-464c-9121-b9d97481a76b
    #alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-1-2017-01 #Optional (Picks default if not used)    
    # SSL Redirect Setting
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    # External DNS - For creating a Record Set in Route53
    external-dns.alpha.kubernetes.io/hostname: default101.stacksimplify.com 
spec:
  ingressClassName: my-aws-ingress-class   # Ingress Class                  
  defaultBackend:
    service:
      name: app3-nginx-nodeport-service
      port:
        number: 80     
  rules:
    - host: app101.stacksimplify.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-nodeport-service
                port: 
                  number: 80
    - host: app201.stacksimplify.com
      http:
        paths:                  
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2-nginx-nodeport-service
                port: 
                  number: 80

# Important Note-1: In path based routing order is very important, if we are going to use  "/*", try to use it at the end of all rules.                                        
                        
# 1. If  "spec.ingressClassName: my-aws-ingress-class" not specified, will reference default ingress class on this kubernetes cluster
# 2. Default Ingress class is nothing but for which ingress class we have the annotation `ingressclass.kubernetes.io/is-default-class: "true"`     
                         
```

## Step-03: Deploy all Application Kubernetes Manifests and Verify
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

### Verify External DNS Log
```t
# Verify External DNS logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```
### Verify Route53
- Go to Services -> Route53
- You should see **Record Sets** added for 
  - default101.stacksimplify.com
  - app101.stacksimplify.com
  - app201.stacksimplify.com

## Step-04: Access Application using newly registered DNS Name
### Perform nslookup tests before accessing Application
- Test if our new DNS entries registered and resolving to an IP Address
```t
# nslookup commands
nslookup default101.stacksimplify.com
nslookup app101.stacksimplify.com
nslookup app201.stacksimplify.com
```
### Positive Case: Access Application using DNS domain
```t
# Access App1
http://app101.stacksimplify.com/app1/index.html

# Access App2
http://app201.stacksimplify.com/app2/index.html

# Access Default App (App3)
http://default101.stacksimplify.com
```

### Negative Case: Access Application using DNS domain
```t
# Access App2 using App1 DNS Domain
http://app101.stacksimplify.com/app2/index.html  -- SHOULD FAIL

# Access App1 using App2 DNS Domain
http://app201.stacksimplify.com/app1/index.html  -- SHOULD FAIL

# Access App1 and App2 using Default Domain
http://default101.stacksimplify.com/app1/index.html -- SHOULD FAIL
http://default101.stacksimplify.com/app2/index.html -- SHOULD FAIL
```

## Step-05: Clean Up
```t
# Delete Manifests
kubectl delete -f kube-manifests/

## Verify Route53 Record Set to ensure our DNS records got deleted
- Go to Route53 -> Hosted Zones -> Records 
- The below records should be deleted automatically
  - default101.stacksimplify.com
  - app101.stacksimplify.com
  - app201.stacksimplify.com
```

