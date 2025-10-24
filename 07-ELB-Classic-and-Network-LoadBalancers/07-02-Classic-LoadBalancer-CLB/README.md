# AWS - Classic Load Balancer - CLB

## Classic Load Balancer Architecture Diagram

```mermaid
graph TB
    USER[Client Request] -->|HTTP/HTTPS| CLB[Classic Load Balancer<br/>Legacy ELB]
    
    CLB -->|NodePort| NODE1[Worker Node 1<br/>NodePort: 31XXX]
    CLB -->|NodePort| NODE2[Worker Node 2<br/>NodePort: 31XXX]
    CLB -->|NodePort| NODE3[Worker Node 3<br/>NodePort: 31XXX]
    
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
        
        SVC[Service Type: LoadBalancer<br/>No annotation required]
        SVC -.->|Selects| POD1
        SVC -.->|Selects| POD2
        SVC -.->|Selects| POD3
    end
    
    subgraph "Service Manifest"
        YAML[type: LoadBalancer]
        YAML --> PORT[port: 80]
        YAML --> TARGET[targetPort: 8095]
        YAML --> NO_ANNO[No annotations needed<br/>Default: CLB]
    end
    
    SVC --> YAML
    
    subgraph "CLB Characteristics"
        C1[Layer 4 TCP and Layer 7 HTTP/HTTPS]
        C2[Targets: Worker Nodes]
        C3[Port: NodePort 30000-32767]
        C4[Legacy load balancer type]
        C5[Basic health checks]
        C6[No advanced features]
        C7[Being phased out by AWS]
    end
    
    subgraph "Automatic Creation"
        AUTO1[K8s cloud provider creates CLB]
        AUTO2[Registers all worker nodes]
        AUTO3[Configures health checks]
        AUTO4[Sets up listeners]
    end
    
    subgraph "CLB vs NLB vs ALB"
        CLB_FEATURE[CLB:<br/>Legacy<br/>Both L4 & L7<br/>Basic features<br/>Deprecated]
        NLB_FEATURE[NLB:<br/>Modern<br/>Layer 4 only<br/>High performance<br/>Static IP<br/>Recommended]
        ALB_FEATURE[ALB:<br/>Modern<br/>Layer 7 only<br/>Advanced routing<br/>Host/Path rules<br/>Recommended]
    end
    
    subgraph "Migration Path"
        OLD[CLB Legacy]
        OLD --> NLB_NEW[Migrate to NLB<br/>For Layer 4]
        OLD --> ALB_NEW[Migrate to ALB<br/>For Layer 7]
        NLB_NEW --> LBC[Use AWS Load Balancer Controller]
        ALB_NEW --> LBC
    end
    
    subgraph "Limitations"
        L1[No WebSocket support]
        L2[No HTTP/2 support]
        L3[No SNI support]
        L4[No path-based routing]
        L5[No host-based routing]
        L6[Higher cost vs NLB]
    end
    
    subgraph "Health Check"
        HC[Target Group Health]
        HC --> HC_PROTO[Protocol: HTTP/TCP]
        HC --> HC_PORT[Port: NodePort]
        HC --> HC_PATH[Path: /]
    end
    
    style CLB fill:#FFD700
    style SVC fill:#2E8B57
    style POD1 fill:#90EE90
    style CLB_FEATURE fill:#FF6B6B
    style NLB_FEATURE fill:#2E8B57
    style ALB_FEATURE fill:#4A90E2
```

### Diagram Explanation

- **Classic Load Balancer**: **Legacy AWS load balancer** automatically created by Kubernetes cloud provider when Service type is LoadBalancer
- **No Annotations Needed**: CLB is **default behavior** when no `service.beta.kubernetes.io/aws-load-balancer-type` annotation specified
- **NodePort Targets**: CLB routes traffic to **worker node IPs** on NodePort, kube-proxy forwards to pods
- **Deprecated**: AWS recommends **migrating to NLB or ALB**, CLB lacks modern features and is being phased out
- **Basic Features Only**: Supports **basic Layer 4/7** load balancing, no advanced routing, WebSocket, HTTP/2, or SNI
- **Hybrid Layer Support**: Can handle **both TCP (L4) and HTTP/HTTPS (L7)** but without advanced Layer 7 features
- **Health Checks**: Performs **simple health checks** on NodePort, less sophisticated than NLB/ALB health checks
- **Migration to NLB**: For **Layer 4 traffic** (TCP/UDP), migrate to NLB with annotation `service.beta.kubernetes.io/aws-load-balancer-type: nlb`
- **Migration to ALB**: For **Layer 7 traffic** (HTTP/HTTPS), migrate to ALB using Ingress with AWS Load Balancer Controller
- **Cost**: CLB typically **more expensive** than NLB for same traffic volume, another reason to migrate

## Step-01: Create AWS Classic Load Balancer Kubernetes Manifest & Deploy
- **04-ClassicLoadBalancer.yml**
```yml
apiVersion: v1
kind: Service
metadata:
  name: clb-usermgmt-restapp
  labels:
    app: usermgmt-restapp
spec:
  type: LoadBalancer  # Regular k8s Service manifest with type as LoadBalancer
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

# List Services (Verify newly created CLB Service)
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
http://<CLB-DNS-NAME>/usermgmt/health-status
```    

## Step-03: Clean Up 
```
# Delete all Objects created
kubectl delete -f kube-manifests/

# Verify current Kubernetes Objects
kubectl get all
```