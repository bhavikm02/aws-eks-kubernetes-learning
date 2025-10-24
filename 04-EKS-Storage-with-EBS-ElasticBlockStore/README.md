# AWS EKS Storage

## Architecture Diagram

```mermaid
graph TB
    subgraph "EKS Cluster"
        subgraph "Control Plane"
            API[API Server]
        end
        
        subgraph "Worker Nodes"
            subgraph "EBS CSI Driver Components"
                CTRL[EBS CSI Controller<br/>Deployment]
                NODE[EBS CSI Node<br/>DaemonSet]
            end
            
            subgraph "Application Pods"
                MYSQL[MySQL Pod]
                APP[User Management<br/>Microservice]
            end
        end
        
        subgraph "Kubernetes Resources"
            SC[Storage Class<br/>ebs-sc]
            PVC[Persistent Volume Claim<br/>ebs-mysql-pv-claim]
            PV[Persistent Volume<br/>Dynamically Created]
            CM[ConfigMap<br/>usermgmt-dbcreation-script]
        end
    end
    
    subgraph "AWS"
        EBS[EBS Volume<br/>gp3 - 4GB]
        IAM[IAM Role<br/>EBS CSI Driver Policy]
        PIA[Pod Identity Agent]
    end
    
    USER[Developer] -->|1. Create SC & PVC| API
    API --> SC
    API --> PVC
    
    PVC -->|2. Request Storage| CTRL
    CTRL -->|3. Authenticate| PIA
    PIA -->|4. Verify Permissions| IAM
    
    CTRL -->|5. Create EBS Volume| EBS
    EBS -->|6. Volume Ready| PV
    PV -->|7. Bind| PVC
    
    API -->|8. Deploy MySQL| MYSQL
    MYSQL -->|9. Mount Volume| NODE
    NODE -->|10. Attach EBS| EBS
    
    CM -->|Initialize DB Schema| MYSQL
    
    MYSQL -->|11. ClusterIP Service| APP
    APP -->|12. Connect to MySQL| MYSQL
    
    EBS -.->|Persistent Data| MYSQL
    
    style CTRL fill:#FF9900
    style NODE fill:#FF9900
    style EBS fill:#FF6B6B
    style MYSQL fill:#2E8B57
    style APP fill:#4A90E2
    style PIA fill:#9B59B6
```

### Diagram Explanation

- **EBS CSI Driver**: **Kubernetes controller** that enables dynamic provisioning of **AWS EBS volumes** as **PersistentVolumes** for pod storage
- **CSI Controller**: **Deployment** (runs on few nodes) handles **volume creation**, **attachment**, **deletion**, and communicates with **AWS EBS API**
- **CSI Node DaemonSet**: Runs on **every worker node**, responsible for **mounting volumes** into pods and managing **block device operations**
- **Storage Class**: Defines **EBS volume type** (gp3/gp2), **IOPS**, **throughput**, and **reclaim policy** for dynamic provisioning
- **Persistent Volume Claim (PVC)**: Kubernetes **storage request** by pods - specifies **size**, **access mode**, and references **StorageClass**
- **Dynamic Provisioning**: When PVC is created, CSI Controller **automatically creates** EBS volume and binds it as a **PersistentVolume**
- **Pod Identity Agent**: Provides **IAM credentials** to CSI driver pods using **EKS Pod Identity**, eliminating need for **node-level IAM policies**
- **Volume Attachment**: EBS CSI Node mounts the **attached EBS volume** to the node, then **bind-mounts** it into the pod's filesystem
- **Data Persistence**: MySQL data survives **pod restarts** and **rescheduling** as EBS volume persists independently of pod lifecycle
- **ConfigMap Integration**: Pre-loads MySQL with **database schema** (usermgmt) for application initialization during first startup

## AWS EBS CSI Driver
- We are going to use EBS CSI Driver and use EBS Volumes for persistence storage to MySQL Database

## Topics
1. Install EBS CSI Driver
2. Create MySQL Database Deployment & ClusterIP Service
3. Create User Management Microservice Deployment & NodePort Service

## Concepts
| Kubernetes Object  | YAML File |
| ------------- | ------------- |
| Storage Class  | 01-storage-class.yml |
| Persistent Volume Claim | 02-persistent-volume-claim.yml   |
| Config Map  | 03-UserManagement-ConfigMap.yml  |
| Deployment, Environment Variables, Volumes, VolumeMounts  | 04-mysql-deployment.yml  |
| ClusterIP Service  | 05-mysql-clusterip-service.yml  |
| Deployment, Environment Variables  | 06-UserManagementMicroservice-Deployment.yml  |
| NodePort Service  | 07-UserManagement-Service.yml  |



## References:
- **Dynamic Volume Provisioning:** https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/
- https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/deploy/kubernetes/overlays/stable
- https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html
- https://github.com/kubernetes-sigs/aws-ebs-csi-driver
- https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes/dynamic-provisioning
- https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/deploy/kubernetes/overlays/stable
- https://github.com/kubernetes-sigs/aws-ebs-csi-driver
- **Legacy: Will be deprecated** 
  - https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-ebs
  - https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html