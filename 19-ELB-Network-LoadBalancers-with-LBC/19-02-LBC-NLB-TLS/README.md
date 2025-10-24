---
title: AWS Load Balancer Controller - NLB TLS
description: Learn to use AWS Network Load Balancer TLS with AWS Load Balancer Controller
---

## NLB TLS Termination Diagram

```mermaid
graph TB
    USER[Client] -->|HTTPS Request<br/>Port 443| NLB[Network Load Balancer<br/>TLS Listener]
    
    NLB -->|TLS Handshake| CERT[ACM Certificate<br/>SSL/TLS Certificate]
    CERT -->|Validate| NLB
    
    NLB -->|Decrypt TLS| BACKEND_PROTO{Backend Protocol?}
    
    BACKEND_PROTO -->|tcp| TCP_BACKEND[Plain TCP to Pods<br/>Port 80]
    BACKEND_PROTO -->|tls| TLS_BACKEND[Re-encrypt TLS to Pods<br/>Port 443]
    
    TCP_BACKEND -->|Target: Pod IP| POD1[Nginx Pod 1<br/>Port 80]
    TCP_BACKEND -->|Target: Pod IP| POD2[Nginx Pod 2<br/>Port 80]
    
    subgraph "TLS Configuration Annotations"
        SSL_CERT[service.beta.kubernetes.io/aws-load-balancer-ssl-cert<br/>ACM Certificate ARN]
        SSL_PORTS[service.beta.kubernetes.io/aws-load-balancer-ssl-ports<br/>443]
        SSL_POLICY[service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy<br/>ELBSecurityPolicy-TLS13-1-2-2021-06]
        BACKEND_PROTO_ANNO[service.beta.kubernetes.io/aws-load-balancer-backend-protocol<br/>tcp or tls]
    end
    
    subgraph "NLB Listener Configuration"
        LISTENER443[Listener: Port 443<br/>Protocol: TLS]
        LISTENER80[Listener: Port 80<br/>Protocol: TCP<br/>Optional]
        
        LISTENER443 --> TLS_TERMINATE[TLS Termination at NLB]
        LISTENER80 --> NO_TLS[No TLS]
    end
    
    NLB --> LISTENER443
    NLB -.->|Optional| LISTENER80
    
    subgraph "TLS Security Policies"
        POL1[ELBSecurityPolicy-TLS13-1-2-2021-06<br/>Latest TLS 1.3]
        POL2[ELBSecurityPolicy-TLS-1-2-2017-01<br/>TLS 1.2 minimum]
        POL3[ELBSecurityPolicy-2016-08<br/>Older TLS versions]
    end
    
    SSL_POLICY -.->|Configures| POL1
    
    subgraph "Certificate Management"
        ACM[AWS Certificate Manager]
        ACM --> CERT_REQ[Request Certificate]
        ACM --> CERT_VALID[DNS or Email Validation]
        ACM --> CERT_AUTO[Auto-renewal]
        ACM --> CERT_ARN[Certificate ARN]
    end
    
    CERT -.->|Managed by| ACM
    
    subgraph "Traffic Flow Scenarios"
        S1[Client --HTTPS--> NLB --HTTP--> Pods<br/>TLS termination at NLB]
        S2[Client --HTTPS--> NLB --HTTPS--> Pods<br/>End-to-end encryption]
        S3[Client --HTTPS:443--> NLB<br/>Client --HTTP:80--> NLB<br/>Mixed listeners]
    end
    
    style NLB fill:#FF9900
    style CERT fill:#2E8B57
    style POD1 fill:#90EE90
    style LISTENER443 fill:#4A90E2
    style SSL_POLICY fill:#FFD700
```

### Diagram Explanation

- **TLS Termination at NLB**: NLB **decrypts** incoming HTTPS traffic using ACM certificate, sends plain TCP to pods
- **ACM Certificate**: Certificate from **AWS Certificate Manager** referenced by ARN in annotation, automatically renewed
- **SSL Ports Annotation**: Specifies which listener ports use **TLS** (e.g., 443), allows mixed TLS and non-TLS listeners
- **Backend Protocol**: Set to **tcp** for TLS termination or **tls** for end-to-end encryption (pass-through)
- **TLS Security Policy**: Defines **allowed TLS versions** and cipher suites, latest policy supports TLS 1.3
- **Mixed Listeners**: Can have both **port 443 (TLS)** and **port 80 (plain TCP)** on same NLB
- **Target Type IP**: NLB forwards decrypted traffic **directly to pod IPs** on configured target port
- **Certificate Validation**: ACM validates **domain ownership** via DNS or email before issuing certificate
- **Cipher Suites**: Security policy controls **encryption algorithms** used during TLS handshake
- **Performance**: NLB TLS offloading reduces **CPU load on pods**, centralizes certificate management

## Step-01: Introduction
- Understand about the 4 TLS Annotations for Network Load Balancers
- aws-load-balancer-ssl-cert
- aws-load-balancer-ssl-ports
- aws-load-balancer-ssl-negotiation-policy
- aws-load-balancer-ssl-negotiation-policy

## Step-02: Review TLS Annotations
- **File Name:** `kube-manifests\02-LBC-NLB-LoadBalancer-Service.yml`
- **Security Policies:** https://docs.aws.amazon.com/elasticloadbalancing/latest/network/create-tls-listener.html#describe-ssl-policies
```yaml
    # TLS
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:180789647333:certificate/d86de939-8ffd-410f-adce-0ce1f5be6e0d
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: 443, # Specify this annotation if you need both TLS and non-TLS listeners on the same load balancer
    service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp 
```


## Step-03: Deploy all kube-manifests
```t
# Deploy kube-manifests
kubectl apply -f kube-manifests/

# Verify Pods
kubectl get pods

# Verify Services
kubectl get svc
Observation: 
1. Verify the network lb DNS name

# Verify AWS Load Balancer Controller pod logs
kubectl -n kube-system get pods
kubectl -n kube-system logs -f <aws-load-balancer-controller-POD-NAME>

# Verify using AWS Mgmt Console
Go to Services -> EC2 -> Load Balancing -> Load Balancers
1. Verify Description Tab - DNS Name matching output of "kubectl get svc" External IP
2. Verify Listeners Tab
Observation:  Should see two listeners Port 80 and 443

Go to Services -> EC2 -> Load Balancing -> Target Groups
1. Verify Registered targets
2. Verify Health Check path
Observation: Should see two target groups. 1 Target group for 1 listener

# Access Application
# Test HTTP URL
http://<NLB-DNS-NAME>
http://lbc-network-lb-tls-demo-a956479ba85953f8.elb.us-east-1.amazonaws.com

# Test HTTPS URL
https://<NLB-DNS-NAME>
https://lbc-network-lb-tls-demo-a956479ba85953f8.elb.us-east-1.amazonaws.com
```

## Step-04: Clean-Up
```t
# Delete or Undeploy kube-manifests
kubectl delete -f kube-manifests/

# Verify if NLB deleted 
In AWS Mgmt Console, 
Go to Services -> EC2 -> Load Balancing -> Load Balancers
```

## References
- [Network Load Balancer](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html)
- [NLB Service](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/service/nlb/)
- [NLB Service Annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/service/annotations/)

