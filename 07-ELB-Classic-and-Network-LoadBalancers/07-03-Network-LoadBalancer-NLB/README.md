 # AWS - Network Load Balancer - NLB

## Network Load Balancer (Legacy) Diagram

```mermaid
graph TB
    USER[Client Request] -->|TCP/UDP| NLB[Network Load Balancer<br/>Created by Cloud Provider]
    
    NLB -->|NodePort| NODE1[Worker Node 1<br/>NodePort: 31XXX]
    NLB -->|NodePort| NODE2[Worker Node 2<br/>NodePort: 31XXX]
    NLB -->|NodePort| NODE3[Worker Node 3<br/>NodePort: 31XXX]
    
    subgraph "EKS Cluster"
        subgraph "Worker Node 1"
            NODE1 -->|kube-proxy| POD1[App Pod]
        end
        
        subgraph "Worker Node 2"
            NODE2 -->|kube-proxy| POD2[App Pod]
        end
        
        subgraph "Worker Node 3"
            NODE3 -->|kube-proxy| POD3[App Pod]
        end
        
        SVC[Service Type: LoadBalancer<br/>with NLB annotation]
        SVC -.->|Selects| POD1
        SVC -.->|Selects| POD2
        SVC -.->|Selects| POD3
    end
    
    subgraph "Service Manifest"
        YAML[type: LoadBalancer]
        YAML --> PORT[port: 80]
        YAML --> TARGET[targetPort: 8095]
        YAML --> ANNO[annotation:<br/>aws-load-balancer-type: nlb]
    end
    
    SVC --> YAML
    
    subgraph "Legacy Cloud Provider vs AWS LB Controller"
        LEGACY[Legacy Cloud Provider<br/>In-tree driver]
        LEGACY --> L_ANNO[annotation: nlb]
        LEGACY --> L_TARGET[Target Type: instance only]
        LEGACY --> L_FEATURES[Limited features]
        
        MODERN[AWS Load Balancer Controller<br/>Out-of-tree CSI]
        MODERN --> M_ANNO[annotation: external]
        MODERN --> M_TARGET[Target Type: ip or instance]
        MODERN --> M_FEATURES[Full feature set]
        MODERN --> M_INGRESS[Also supports Ingress]
    end
    
    subgraph "NLB Features"
        F1[Layer 4 TCP/UDP/TLS]
        F2[High performance]
        F3[Ultra-low latency]
        F4[Millions requests/sec]
        F5[Static IP per AZ]
        F6[Preserve source IP]
        F7[Cross-zone load balancing]
    end
    
    subgraph "Legacy vs Modern Annotation"
        OLD_ANNO[service.beta.kubernetes.io/aws-load-balancer-type: nlb<br/>Legacy Cloud Provider]
        NEW_ANNO[service.beta.kubernetes.io/aws-load-balancer-type: external<br/>service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance/ip<br/>AWS Load Balancer Controller]
    end
    
    subgraph "Migration Recommendation"
        CURRENT[Current: Legacy NLB<br/>Cloud Provider]
        CURRENT --> MIGRATE[Migrate to AWS LB Controller]
        MIGRATE --> BENEFITS[Benefits:<br/>- Target type IP<br/>- More annotations<br/>- Better features<br/>- Active development]
    end
    
    subgraph "NLB Target Types"
        INST[Instance Mode<br/>Targets: Worker Nodes<br/>Port: NodePort<br/>Legacy default]
        IP[IP Mode<br/>Targets: Pod IPs<br/>Port: Container Port<br/>Requires LB Controller]
    end
    
    subgraph "Use Cases"
        UC1[TCP/UDP applications]
        UC2[Extreme performance needs]
        UC3[Static IP requirements]
        UC4[Non-HTTP protocols]
        UC5[Gaming servers]
        UC6[IoT applications]
    end
    
    style NLB fill:#FF9900
    style SVC fill:#2E8B57
    style POD1 fill:#90EE90
    style MODERN fill:#2E8B57
    style LEGACY fill:#FFD700
```

### Diagram Explanation

- **Legacy Cloud Provider NLB**: Created by Kubernetes **in-tree cloud provider** with annotation `service.beta.kubernetes.io/aws-load-balancer-type: nlb`
- **Target Type Instance Only**: Legacy NLB supports **instance mode** only, routes to worker node IPs on NodePort
- **Modern Alternative**: **AWS Load Balancer Controller** (separate pod) provides more features including IP target type
- **NLB Annotation**: Single annotation `aws-load-balancer-type: nlb` creates NLB instead of default CLB
- **Layer 4 Only**: NLB operates at **TCP/UDP layer**, no HTTP-specific features, ideal for non-HTTP protocols
- **High Performance**: Handles **millions of requests per second** with ultra-low latency, better than CLB/ALB
- **Static IP per AZ**: Each AZ gets **static IP address**, useful for IP whitelisting and firewall rules
- **Source IP Preservation**: Maintains **client source IP** when routing to backends, useful for logging and security
- **Migration Path**: Recommend **migrating to AWS Load Balancer Controller** for target-type IP and advanced features
- **Limited Features**: Legacy cloud provider NLB has **fewer configuration options** compared to Load Balancer Controller version

## Step-01: Create AWS Network Load Balancer Kubernetes Manifest & Deploy
- **04-NetworkLoadBalancer.yml**
```yml
apiVersion: v1
kind: Service
metadata:
  name: nlb-usermgmt-restapp
  labels:
    app: usermgmt-restapp
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb    # To create Network Load Balancer
spec:
  type: LoadBalancer # Regular k8s Service manifest with type as LoadBalancer
  selector:
    app: usermgmt-restapp     
  ports:
  - port: 80
    targetPort: 8095
```
- **Deploy all Manifest**
```
# Deploy all manifests
kubectl apply -f kube-manifests/

# List Services (Verify newly created NLB Service)
kubectl get svc

# Verify Pods
kubectl get pods
```

## Step-02: Verify the deployment
- Verify if new CLB got created 
  - Go to  Services -> EC2 -> Load Balancing -> Load Balancers 
    - CLB should be created
    - Copy DNS Name (Example: a85ae6e4030aa4513bd200f08f1eb9cc-7f13b3acc1bcaaa2.elb.us-east-1.amazonaws.com)
  - Go to  Services -> EC2 -> Load Balancing -> Target Groups
    - Verify the health status, we should see active. 
- **Access Application** 
```
# Access Application
http://<NLB-DNS-NAME>/usermgmt/health-status
```    

## Step-03: Clean Up 
```
# Delete all Objects created
kubectl delete -f kube-manifests/

# Verify current Kubernetes Objects
kubectl get all
```


