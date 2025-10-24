---
title: AWS Load Balancer Controller - External DNS & Service
description: Learn AWS Load Balancer Controller - External DNS & Kubernetes Service
---

## External DNS with LoadBalancer Service Diagram

```mermaid
graph TB
    USER[End User] -->|DNS Query| R53[Route53 Hosted Zone]
    R53 -->|Returns ALB DNS| USER
    USER -->|HTTP Request| ALB[Application Load Balancer]
    
    ALB -->|Forward Traffic| PODS[Application Pods]
    
    subgraph "EKS Cluster"
        SVC[Service<br/>type: LoadBalancer]
        SVC -->|Creates| ALB
        SVC -.->|Selects| PODS
        
        EXTDNS[External DNS Pod]
        EXTDNS -->|Watch Services| API[Kubernetes API]
        API -->|List Services| EXTDNS
    end
    
    subgraph "External DNS Flow"
        EXTDNS -->|1. Detect Service| SVC
        SVC -->|2. Read Annotation| ANNO[external-dns.alpha.kubernetes.io/hostname]
        ANNO -->|3. Get LB DNS| LB_INFO[ALB DNS from Service Status]
        LB_INFO -->|4. Create DNS Record| R53
    end
    
    subgraph "Service Manifest"
        TYPE[type: LoadBalancer]
        SELECTOR[selector: app=app1-nginx]
        PORT[port: 80]
        HOSTNAME[annotation:<br/>external-dns.alpha.kubernetes.io/hostname<br/>externaldns-k8s-service-demo101.example.com]
    end
    
    SVC --> TYPE
    SVC --> SELECTOR
    SVC --> PORT
    SVC --> HOSTNAME
    
    subgraph "Route53 Record"
        RECORD[A Record (ALIAS)]
        RECORD --> NAME[Name: externaldns-k8s-service-demo101.example.com]
        RECORD --> TARGET[Target: ALB DNS Name]
        RECORD --> TTL[TTL: 300 seconds]
    end
    
    R53 --> RECORD
    
    subgraph "Comparison: Ingress vs Service"
        ING_DNS[Ingress + External DNS:<br/>- Host-based routing<br/>- Path-based routing<br/>- SSL termination<br/>- Multiple services]
        SVC_DNS[Service + External DNS:<br/>- Simple L4 load balancing<br/>- Single service<br/>- Direct access<br/>- No routing rules]
    end
    
    subgraph "Cloud Provider Load Balancer"
        CLOUD[Kubernetes Cloud Provider]
        CLOUD -->|Creates| CLB[Classic Load Balancer<br/>Default]
        CLOUD -->|OR Creates| ALB_OPT[ALB if annotated]
        CLOUD -->|OR Creates| NLB_OPT[NLB if annotated]
    end
    
    SVC -.->|Managed by| CLOUD
    
    subgraph "DNS Automation Benefits"
        B1[No manual Route53 updates]
        B2[Automatic record creation]
        B3[Automatic record deletion]
        B4[Sync with LB lifecycle]
        B5[Friendly DNS names]
        B6[Multiple hostname support]
    end
    
    subgraph "External DNS IAM"
        ROLE[IAM Role via IRSA]
        ROLE --> P1[route53:ChangeResourceRecordSets]
        ROLE --> P2[route53:ListHostedZones]
        ROLE --> P3[route53:ListResourceRecordSets]
    end
    
    EXTDNS -.->|Uses| ROLE
    
    subgraph "Lifecycle Management"
        CREATE[Service created → DNS record created]
        UPDATE[LB DNS changes → DNS record updated]
        DELETE[Service deleted → DNS record deleted]
        
        CREATE --> UPDATE --> DELETE
    end
    
    style ALB fill:#FF9900
    style R53 fill:#9B59B6
    style EXTDNS fill:#4A90E2
    style SVC fill:#2E8B57
    style HOSTNAME fill:#FFD700
```

### Diagram Explanation

- **LoadBalancer Service**: Service **type: LoadBalancer** automatically creates AWS load balancer (CLB/ALB/NLB) via cloud provider
- **External DNS Annotation**: Adding `external-dns.alpha.kubernetes.io/hostname` triggers **automatic DNS record creation** in Route53
- **Service vs Ingress**: Service approach is **simpler** but less feature-rich, good for exposing single app without advanced routing
- **ALIAS Record**: Route53 creates **ALIAS record** pointing to load balancer DNS, better performance than CNAME
- **Automatic Lifecycle**: DNS records are **created, updated, and deleted** automatically following Service lifecycle
- **Multiple Hostnames**: Can specify **comma-separated hostnames** to create multiple DNS records for same service
- **Cloud Provider Integration**: Service uses **built-in Kubernetes cloud provider** to create load balancer, simpler than Load Balancer Controller
- **No Routing Rules**: Unlike Ingress, Service-based approach provides **direct access** without path/host routing capabilities
- **IRSA Authentication**: External DNS uses **IAM Roles for Service Accounts** to securely access Route53 API
- **Cost Consideration**: Each LoadBalancer Service creates **separate load balancer**, more expensive than Ingress sharing single ALB

## Step-01: Introduction
- We will create a Kubernetes Service of `type: LoadBalancer`
- We will annotate that Service with external DNS hostname `external-dns.alpha.kubernetes.io/hostname: externaldns-k8s-service-demo101.stacksimplify.com` which will register the DNS in Route53 for that respective load balancer

## Step-02: 02-Nginx-App1-LoadBalancer-Service.yml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1-nginx-loadbalancer-service
  labels:
    app: app1-nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: externaldns-k8s-service-demo101.stacksimplify.com
spec:
  type: LoadBalancer
  selector:
    app: app1-nginx
  ports:
    - port: 80
      targetPort: 80  
```
## Step-03: Deploy & Verify

### Deploy & Verify
```t
# Deploy kube-manifests
kubectl apply -f kube-manifests/

# Verify Apps
kubectl get deploy
kubectl get pods

# Verify Service
kubectl get svc
```
### Verify Load Balancer 
- Go to EC2 -> Load Balancers -> Verify Load Balancer Settings

### Verify External DNS Log
```t
# Verify External DNS logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```
### Verify Route53
- Go to Services -> Route53
- You should see **Record Sets** added for `externaldns-k8s-service-demo101.stacksimplify.com`


## Step-04: Access Application using newly registered DNS Name
### Perform nslookup tests before accessing Application
- Test if our new DNS entries registered and resolving to an IP Address
```t
# nslookup commands
nslookup externaldns-k8s-service-demo101.stacksimplify.com
```
### Access Application using DNS domain
```t
# HTTP URL
http://externaldns-k8s-service-demo101.stacksimplify.com/app1/index.html
```

## Step-05: Clean Up
```t
# Delete Manifests
kubectl delete -f kube-manifests/

## Verify Route53 Record Set to ensure our DNS records got deleted
- Go to Route53 -> Hosted Zones -> Records 
- The below records should be deleted automatically
  - externaldns-k8s-service-demo101.stacksimplify.com
```


## References
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/alb-ingress.md
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md
