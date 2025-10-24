---
title: AWS Load Balancer Ingress SSL
description: Learn AWS Load Balancer Controller - Ingress SSL
---

## ALB Ingress SSL Diagram

```mermaid
graph TB
    USER[Client Browser] -->|HTTPS Request<br/>Port 443| ALB[Application Load Balancer]
    USER -.->|HTTP Request<br/>Port 80| ALB
    
    ALB -->|TLS Handshake| CERT[ACM Certificate<br/>*.yourdomain.com]
    
    subgraph "ALB Listeners"
        HTTPS_LIST[Listener: Port 443<br/>Protocol: HTTPS]
        HTTP_LIST[Listener: Port 80<br/>Protocol: HTTP]
        
        HTTPS_LIST -->|TLS Termination| DECRYPT[Decrypt Traffic]
        HTTP_LIST -->|Plain HTTP| FORWARD[Forward to Targets]
        
        DECRYPT --> FORWARD
    end
    
    ALB --> HTTPS_LIST
    ALB --> HTTP_LIST
    
    FORWARD -->|NodePort| TG[Target Group<br/>Worker Nodes]
    
    subgraph "EKS Cluster"
        TG -->|Route to Pods| PODS[Application Pods]
        
        ING[Ingress Resource]
        SVC[NodePort Service]
        
        ING -->|Backend| SVC
        SVC -.->|Selects| PODS
    end
    
    subgraph "SSL Annotations"
        SSL1[alb.ingress.kubernetes.io/listen-ports<br/>[HTTPS:443, HTTP:80]]
        SSL2[alb.ingress.kubernetes.io/certificate-arn<br/>ACM Certificate ARN]
        SSL3[alb.ingress.kubernetes.io/ssl-policy<br/>ELBSecurityPolicy-TLS-1-1-2017-01]
    end
    
    ING --> SSL1
    ING --> SSL2
    
    subgraph "AWS Certificate Manager"
        ACM[ACM Service]
        ACM --> REQ[Request Public Certificate]
        REQ --> WILDCARD[Domain: *.yourdomain.com]
        WILDCARD --> DNS_VAL[DNS Validation]
        DNS_VAL --> R53[Create Record in Route53]
        R53 --> ISSUED[Certificate Issued]
        ISSUED --> AUTO_RENEW[Auto-renewal enabled]
    end
    
    CERT -.->|Managed by| ACM
    
    subgraph "Certificate Validation Flow"
        V1[1. Request certificate in ACM]
        V2[2. Add domain names]
        V3[3. Choose DNS validation]
        V4[4. Create CNAME record in Route53]
        V5[5. Wait 5-10 minutes for validation]
        V6[6. Certificate status: Issued]
        
        V1 --> V2 --> V3 --> V4 --> V5 --> V6
    end
    
    subgraph "Security Features"
        TLS_VER[TLS 1.2 / 1.3 support]
        CIPHER[Strong cipher suites]
        SNI[SNI support for multi-domain]
        HSTS[HSTS headers optional]
    end
    
    subgraph "Traffic Flow"
        T1[Client → HTTPS:443 → ALB]
        T2[ALB → Decrypt with ACM Cert]
        T3[ALB → HTTP:80 → Pods]
        T4[End-to-end: HTTPS to HTTP]
    end
    
    style ALB fill:#FF9900
    style CERT fill:#2E8B57
    style ING fill:#4A90E2
    style ACM fill:#FFD700
    style HTTPS_LIST fill:#9B59B6
```

### Diagram Explanation

- **ACM Certificate**: **AWS Certificate Manager** provides free SSL/TLS certificates with automatic renewal, supports wildcard domains
- **TLS Termination**: ALB **decrypts HTTPS traffic** using ACM certificate, forwards plain HTTP to backend pods
- **Dual Listeners**: ALB configured with **both HTTP (80) and HTTPS (443)** listeners for flexibility
- **DNS Validation**: Certificate validation via **Route53 CNAME record**, automated by clicking "Create record in Route53"
- **Wildcard Certificate**: `*.yourdomain.com` covers **all subdomains** (app1.yourdomain.com, app2.yourdomain.com, etc.)
- **SSL Policy**: Controls **TLS versions and cipher suites**, default is ELBSecurityPolicy-2016-08, optional to override
- **Certificate ARN**: Unique **Amazon Resource Name** for certificate, referenced in Ingress annotation
- **Automatic Renewal**: ACM automatically **renews certificates** 60 days before expiration, no manual intervention
- **SNI Support**: ALB supports **multiple certificates** via Server Name Indication for different hostnames
- **Zero-Cost SSL**: ACM certificates are **free for ALB/NLB**, only pay for load balancer usage

## Step-01: Introduction
- We are going to register a new DNS in AWS Route53
- We are going to create a SSL certificate 
- Add Annotations related to SSL Certificate in Ingress manifest
- Deploy the manifests and test
- Clean-Up

## Step-02: Pre-requisite - Register a Domain in Route53 (if not exists)
- Goto Services -> Route53 -> Registered Domains
- Click on **Register Domain**
- Provide **desired domain: somedomain.com** and click on **check** (In my case its going to be `stacksimplify.com`)
- Click on **Add to cart** and click on **Continue**
- Provide your **Contact Details** and click on **Continue**
- Enable Automatic Renewal
- Accept **Terms and Conditions**
- Click on **Complete Order**

## Step-03: Create a SSL Certificate in Certificate Manager
- Pre-requisite: You should have a registered domain in Route53 
- Go to Services -> Certificate Manager -> Create a Certificate
- Click on **Request a Certificate**
  - Choose the type of certificate for ACM to provide: Request a public certificate
  - Add domain names: *.yourdomain.com (in my case it is going to be `*.stacksimplify.com`)
  - Select a Validation Method: **DNS Validation**
  - Click on **Confirm & Request**    
- **Validation**
  - Click on **Create record in Route 53**  
- Wait for 5 to 10 minutes and check the **Validation Status**  

## Step-04: Add annotations related to SSL
- **04-ALB-Ingress-SSL.yml**
```yaml
    ## SSL Settings
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}, {"HTTP":80}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:180789647333:certificate/632a3ff6-3f6d-464c-9121-b9d97481a76b
    #alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-1-2017-01 #Optional (Picks default if not used)    
```
## Step-05: Deploy all manifests and test
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

## Step-06: Add DNS in Route53   
- Go to **Services -> Route 53**
- Go to **Hosted Zones**
  - Click on **yourdomain.com** (in my case stacksimplify.com)
- Create a **Record Set**
  - **Name:** ssldemo101.stacksimplify.com
  - **Alias:** yes
  - **Alias Target:** Copy our ALB DNS Name here (Sample: ssl-ingress-551932098.us-east-1.elb.amazonaws.com)
  - Click on **Create**
  
## Step-07: Access Application using newly registered DNS Name
- **Access Application**
- **Important Note:** Instead of `stacksimplify.com` you need to replace with your registered Route53 domain (Refer pre-requisite Step-02)
```t
# HTTP URLs
http://ssldemo101.stacksimplify.com/app1/index.html
http://ssldemo101.stacksimplify.com/app2/index.html
http://ssldemo101.stacksimplify.com/

# HTTPS URLs
https://ssldemo101.stacksimplify.com/app1/index.html
https://ssldemo101.stacksimplify.com/app2/index.html
https://ssldemo101.stacksimplify.com/
```

## Annotation Reference
- [AWS Load Balancer Controller Annotation Reference](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)