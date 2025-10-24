# EKS - Create Cluster

## Architecture Diagram

```mermaid
graph TB
    START([Start EKS Setup]) --> CLI[Install CLI Tools]
    
    CLI --> AWS[AWS CLI]
    CLI --> KUBECTL[kubectl]
    CLI --> EKSCTL[eksctl]
    
    AWS --> CONFIG[Configure AWS Credentials]
    KUBECTL --> CONFIG
    EKSCTL --> CONFIG
    
    CONFIG --> CREATE[Create EKS Cluster]
    
    CREATE --> CP[EKS Control Plane<br/>Managed by AWS]
    CREATE --> VPC[VPC & Subnets<br/>Networking Setup]
    
    CP --> NG[Create Node Groups]
    
    NG --> CHOICE{Node Type?}
    CHOICE -->|Managed| MNG[Managed Node Group<br/>EC2 Instances]
    CHOICE -->|Fargate| FP[Fargate Profile<br/>Serverless]
    
    MNG --> PRICING[Understand Pricing]
    FP --> PRICING
    
    PRICING --> CP_COST[Control Plane: $0.10/hour]
    PRICING --> WN_COST[Worker Nodes: EC2 pricing]
    PRICING --> FG_COST[Fargate: vCPU + Memory pricing]
    
    CP_COST --> DEPLOY[Deploy Applications]
    WN_COST --> DEPLOY
    FG_COST --> DEPLOY
    
    DEPLOY --> DELETE{Cleanup?}
    DELETE -->|Yes| DEL1[Delete Node Groups]
    DELETE -->|No| END1([Cluster Running])
    
    DEL1 --> DEL2[Delete EKS Cluster]
    DEL2 --> DEL3[Verify Resources Deleted]
    DEL3 --> END2([Cleanup Complete])
    
    style START fill:#90EE90
    style CP fill:#FF9900
    style MNG fill:#4A90E2
    style FP fill:#9B59B6
    style PRICING fill:#FFD700
    style END1 fill:#90EE90
    style END2 fill:#90EE90
```

### Diagram Explanation

- **CLI Installation**: Three essential tools required - **AWS CLI** for AWS operations, **kubectl** for Kubernetes management, and **eksctl** for EKS cluster automation
- **AWS Configuration**: Set up **IAM credentials** and **default region** to authenticate and authorize all AWS API calls
- **EKS Control Plane**: Fully managed by AWS, provides **high availability** across multiple AZs, handles **API server** and **etcd** database
- **VPC Networking**: Automatically creates **VPC**, **subnets**, **security groups**, and **routing tables** for cluster communication
- **Node Group Options**: Choose between **Managed Node Groups** (EC2 instances with auto-scaling) or **Fargate Profiles** (serverless, no node management)
- **Control Plane Pricing**: Fixed cost of **$0.10 per hour** regardless of cluster size or workload volume
- **Worker Node Costs**: **EC2-based pricing** for managed node groups based on instance type, plus **EBS storage** costs
- **Fargate Pricing**: Pay only for **vCPU and memory** resources used by pods, billed per second with 1-minute minimum
- **Resource Cleanup**: Important to delete **node groups first**, then **cluster**, then verify **VPC and security groups** are removed
- **Cost Optimization**: Proper cleanup prevents unnecessary charges; use **eksctl delete cluster** command for complete removal

## List of Topics 
- Install CLIs
  - AWS CLI
  - kubectl
  - eksctl
- Create EKS Cluster
- Create EKS Node Groups
- Understand EKS Cluster Pricing
  - EKS Control Plane
  - EKS Worker Nodes
  - EKS Fargate Profile
- Delete EKS Clusters 

