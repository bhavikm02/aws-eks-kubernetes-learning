---
title: AWS Load Balancer Controller - Ingress Target Type IP
description: Learn AWS Load Balancer Controller - Ingress Target Type IP
---

## Target Type IP vs Instance Diagram

```mermaid
graph TB
    USER[Client Request] -->|HTTP/HTTPS| ALB[Application Load Balancer]
    
    ALB -->|Target Type?| CHOICE{Target Type<br/>Annotation}
    
    CHOICE -->|instance| INSTANCE_MODE[Instance Mode<br/>Default]
    CHOICE -->|ip| IP_MODE[IP Mode]
    
    subgraph "Instance Mode Flow"
        INSTANCE_MODE -->|Forward to| TG_INST[Target Group<br/>Target Type: instance]
        TG_INST -->|NodePort: 31XXX| NODE1[Worker Node 1]
        TG_INST -->|NodePort: 31XXX| NODE2[Worker Node 2]
        TG_INST -->|NodePort: 31XXX| NODE3[Worker Node 3]
        
        NODE1 -->|kube-proxy iptables| POD1[Pod A on Node 1]
        NODE1 -.->|Load Balance| POD2[Pod B on Node 2]
        NODE2 -->|kube-proxy iptables| POD2
        NODE3 -->|kube-proxy iptables| POD3[Pod C on Node 3]
        
        INST_NOTE[Extra Hop: ALB → Node → Pod<br/>Source IP lost<br/>Works with any CNI]
    end
    
    subgraph "IP Mode Flow"
        IP_MODE -->|Forward to| TG_IP[Target Group<br/>Target Type: ip]
        TG_IP -->|Direct to Pod IP| POD_IP1[Pod IP: 192.168.1.10]
        TG_IP -->|Direct to Pod IP| POD_IP2[Pod IP: 192.168.1.11]
        TG_IP -->|Direct to Pod IP| POD_IP3[Pod IP: 192.168.1.12]
        
        POD_IP1 --> APP1[App Pod 1]
        POD_IP2 --> APP2[App Pod 2]
        POD_IP3 --> APP3[App Pod 3]
        
        IP_NOTE[Direct Routing: ALB → Pod<br/>Source IP preserved<br/>Requires CNI support]
    end
    
    subgraph "Ingress Annotation"
        ANNO[alb.ingress.kubernetes.io/target-type]
        ANNO --> INST_VAL[instance - Default]
        ANNO --> IP_VAL[ip - Required for sticky sessions]
    end
    
    subgraph "Instance Mode Characteristics"
        INST1[Targets: EC2 Worker Nodes]
        INST2[Port: NodePort 30000-32767]
        INST3[kube-proxy routes to pods]
        INST4[Source IP: ALB IP address]
        INST5[Health check: Node health]
        INST6[Extra network hop]
    end
    
    subgraph "IP Mode Characteristics"
        IP1[Targets: Pod IP addresses]
        IP2[Port: Container Port directly]
        IP3[No kube-proxy overhead]
        IP4[Source IP: Client IP preserved]
        IP5[Health check: Pod health]
        IP6[Direct routing]
        IP7[Required for sticky sessions]
        IP8[Better performance]
    end
    
    subgraph "Comparison"
        COMP[Instance Mode vs IP Mode]
        COMP --> PERF[IP: Better Performance]
        COMP --> STICK[IP: Enables Sticky Sessions]
        COMP --> SOURCE[IP: Preserves Source IP]
        COMP --> COMPAT[Instance: Works with all CNI]
        COMP --> SIMPLE[Instance: Simpler setup]
    end
    
    subgraph "CNI Requirements"
        CNI[IP Mode Requirement]
        CNI --> VPC_CNI[AWS VPC CNI<br/>Native support]
        CNI --> CALICO[Calico CNI<br/>Supported]
        CNI --> OTHER[Other CNIs<br/>Check compatibility]
    end
    
    subgraph "Sticky Session Support"
        STICKY[Session Affinity]
        STICKY --> COOKIE[ALB Cookie-based]
        STICKY --> REQUIRES[Requires target-type: ip]
        STICKY --> DURATION[Session duration: 1s - 7 days]
    end
    
    style ALB fill:#FF9900
    style IP_MODE fill:#2E8B57
    style INSTANCE_MODE fill:#4A90E2
    style TG_IP fill:#FFD700
    style TG_INST fill:#9B59B6
```

### Diagram Explanation

- **Target Type Instance**: ALB routes to **EC2 worker nodes** on NodePort, kube-proxy forwards to pod IPs (extra hop)
- **Target Type IP**: ALB routes **directly to pod IPs**, bypassing NodePort, more efficient networking
- **NodePort Overhead**: Instance mode requires **kube-proxy iptables** rules to route from NodePort to pod, adds latency
- **Direct Pod Routing**: IP mode eliminates NodePort layer, traffic goes **straight from ALB to pod**, lower latency
- **Source IP Preservation**: IP mode maintains **original client IP**, useful for logging, security, and geolocation
- **Sticky Sessions**: Only **IP mode supports** ALB cookie-based sticky sessions for stateful applications
- **Health Checks**: Instance mode checks **node health**, IP mode checks **individual pod health**, more granular
- **CNI Compatibility**: IP mode requires **AWS VPC CNI** or compatible CNI that exposes pod IPs to AWS APIs
- **Performance**: IP mode offers **better performance** due to fewer network hops and reduced iptables overhead
- **Use Cases**: Use **IP mode** for sticky sessions, better performance, and source IP; use **instance mode** for simplicity

## Step-01: Introduction
- `alb.ingress.kubernetes.io/target-type` specifies how to route traffic to pods. 
- You can choose between `instance` and `ip`
- **Instance Mode:** `instance mode` will route traffic to all ec2 instances within cluster on NodePort opened for your service.
- **IP Mode:** `ip mode` is required for sticky sessions to work with Application Load Balancers.


## Step-02: Ingress Manifest - Add target-type
- **File Name:** 04-ALB-Ingress-target-type-ip.yml
```yaml
    # Target Type: IP
    alb.ingress.kubernetes.io/target-type: ip   
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
- **PRIMARILY VERIFY - TARGET GROUPS which contain thePOD IPs instead of WORKER NODE IP with NODE PORTS**
```t
# List Pods and their IPs
kubectl get pods -o wide
```

### Verify External DNS Log
```t
# Verify External DNS logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```
### Verify Route53
- Go to Services -> Route53
- You should see **Record Sets** added for 
  - target-type-ip-501.stacksimplify.com 


## Step-04: Access Application using newly registered DNS Name
### Perform nslookup tests before accessing Application
- Test if our new DNS entries registered and resolving to an IP Address
```t
# nslookup commands
nslookup target-type-ip-501.stacksimplify.com 
```
### Access Application using DNS domain
```t
# Access App1
http://target-type-ip-501.stacksimplify.com /app1/index.html

# Access App2
http://target-type-ip-501.stacksimplify.com /app2/index.html

# Access Default App (App3)
http://target-type-ip-501.stacksimplify.com 
```

## Step-05: Clean Up
```t
# Delete Manifests
kubectl delete -f kube-manifests/

## Verify Route53 Record Set to ensure our DNS records got deleted
- Go to Route53 -> Hosted Zones -> Records 
- The below records should be deleted automatically
  - target-type-ip-501.stacksimplify.com 
```