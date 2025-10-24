---
title: AWS Load Balancer Controller - NLB External DNS
description: Learn to use AWS Network Load Balancer & External DNS with AWS Load Balancer Controller
---

## NLB with External DNS Diagram

```mermaid
graph TB
    USER[End User] -->|DNS Query<br/>nlbdns101.example.com| DNS_RESOLVER[DNS Resolver]
    
    DNS_RESOLVER -->|Query| R53[AWS Route53<br/>Hosted Zone]
    
    R53 -->|Returns| NLB_DNS[NLB DNS Name<br/>*.elb.amazonaws.com]
    
    DNS_RESOLVER -->|Resolved| NLB[Network Load Balancer]
    
    NLB -->|Target: IP Mode| POD1[Pod IP: 192.168.1.10]
    NLB -->|Target: IP Mode| POD2[Pod IP: 192.168.1.11]
    
    subgraph "EKS Cluster"
        subgraph "Application Pods"
            POD1[Nginx Pod 1]
            POD2[Nginx Pod 2]
        end
        
        SVC[LoadBalancer Service<br/>Type: LoadBalancer]
        SVC -.->|Selects| POD1
        SVC -.->|Selects| POD2
        
        EXTDNS[External DNS Pod]
        EXTDNS -->|Watch Services| API[Kubernetes API]
        API -->|List Services| EXTDNS
    end
    
    subgraph "External DNS Flow"
        EXTDNS -->|1. Detect Service| SVC
        SVC -->|2. Read Annotation| ANNO[external-dns.alpha.kubernetes.io/hostname<br/>nlbdns101.example.com]
        ANNO -->|3. Get LB DNS| LB_INFO[NLB DNS from Service Status]
        LB_INFO -->|4. Create/Update| R53
    end
    
    subgraph "Route53 Record"
        R53 --> ALIAS[ALIAS Record]
        ALIAS --> NAME[Name: nlbdns101.example.com]
        ALIAS --> TYPE[Type: A Record]
        ALIAS --> TARGET[Target: NLB DNS]
        ALIAS --> TTL[TTL: 300 seconds]
    end
    
    subgraph "Service Annotations"
        SA1[service.beta.kubernetes.io/aws-load-balancer-type: external]
        SA2[service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip]
        SA3[external-dns.alpha.kubernetes.io/hostname: nlbdns101.example.com]
        SA4[external-dns.alpha.kubernetes.io/ttl: "60"]
    end
    
    SVC --> SA1
    SVC --> SA2
    SVC --> SA3
    
    subgraph "AWS Load Balancer Controller"
        LBC[LB Controller Pod]
        LBC -->|Watch Service| SVC
        LBC -->|Create NLB| NLB
        LBC -->|Register Targets| TG[Target Group<br/>Pod IPs]
    end
    
    NLB --> TG
    TG --> POD1
    TG --> POD2
    
    subgraph "Automatic DNS Management"
        AUTO1[Service created → ExternalDNS detects]
        AUTO2[ExternalDNS creates Route53 record]
        AUTO3[Service deleted → ExternalDNS removes record]
        AUTO4[NLB DNS changes → ExternalDNS updates record]
        
        AUTO1 --> AUTO2 --> AUTO3
        AUTO2 -.->|Update| AUTO4
    end
    
    subgraph "Benefits"
        B1[Friendly DNS names vs LB DNS]
        B2[Automatic record management]
        B3[No manual Route53 changes]
        B4[Consistent with Ingress pattern]
        B5[Multi-service support]
    end
    
    subgraph "External DNS IAM Permissions"
        ROLE[IAM Role via IRSA]
        ROLE --> P1[route53:ChangeResourceRecordSets]
        ROLE --> P2[route53:ListHostedZones]
        ROLE --> P3[route53:ListResourceRecordSets]
    end
    
    EXTDNS -.->|Uses| ROLE
    
    style NLB fill:#FF9900
    style R53 fill:#9B59B6
    style EXTDNS fill:#4A90E2
    style SVC fill:#2E8B57
    style LBC fill:#FFD700
```

### Diagram Explanation

- **External DNS Service Integration**: External DNS watches **LoadBalancer Services** in addition to Ingress resources
- **Hostname Annotation**: `external-dns.alpha.kubernetes.io/hostname` on Service triggers **automatic DNS record creation**
- **ALIAS Record**: Route53 creates **ALIAS record** pointing to NLB DNS name, better than CNAME for zone apex
- **Automatic Lifecycle**: DNS records **created on service creation**, **deleted on service deletion**, fully automated
- **NLB DNS Resolution**: Route53 ALIAS resolves to **NLB's static IPs** in each AZ, maintains high availability
- **TTL Control**: Optional **TTL annotation** controls DNS caching duration, useful for fast failover scenarios
- **Multiple Hostnames**: Can specify **comma-separated hostnames** for single service, creates multiple records
- **TXT Record Ownership**: External DNS creates **TXT records** to track ownership, prevents conflicts
- **IRSA Authentication**: External DNS uses **IAM Role for Service Accounts** to securely access Route53 API
- **User-Friendly Access**: Provides **memorable domain names** instead of long AWS-generated NLB DNS names

## Step-01: Introduction
- Implement External DNS Annotation in NLB Kubernetes Service Manifest


## Step-02: Review External DNS Annotations
- **File Name:** `kube-manifests\02-LBC-NLB-LoadBalancer-Service.yml`
```yaml
    # External DNS - For creating a Record Set in Route53
    external-dns.alpha.kubernetes.io/hostname: nlbdns101.stacksimplify.com
```

## Step-03: Deploy all kube-manifests
```t
# Verify if External DNS Pod exists and Running
kubectl get pods
Observation: 
external-dns pod should be running

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

# Verify External DNS logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')

# Perform nslookup Test
nslookup nlbdns101.stacksimplify.com

# Access Application
# Test HTTP URL
http://nlbdns101.stacksimplify.com

# Test HTTPS URL
https://nlbdns101.stacksimplify.com
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

