# EKS Cluster Pricing

## Pricing Diagram

```mermaid
graph TB
    EKS[EKS Cluster Costs] --> CP[Control Plane]
    EKS --> WN[Worker Nodes]
    EKS --> FG[Fargate]
    
    CP --> CPCOST[Fixed: $0.10/hour]
    CPCOST --> CPDAY[$2.40/day]
    CPDAY --> CPMONTH[$72/month]
    
    WN --> WNTYPE{Instance Type}
    WNTYPE --> T3M[t3.medium]
    WNTYPE --> T3L[t3.large]
    WNTYPE --> M5[m5.xlarge]
    
    T3M --> T3MCOST[$0.0416/hour]
    T3MCOST --> T3MDAY[$1/day]
    T3MDAY --> T3MMONTH[$30/month]
    
    T3L --> T3LCOST[$0.0832/hour]
    T3LCOST --> T3LDAY[$2/day]
    T3LDAY --> T3LMONTH[$60/month]
    
    M5 --> M5COST[$0.192/hour]
    M5COST --> M5DAY[$4.61/day]
    M5DAY --> M5MONTH[$138/month]
    
    WN --> STORAGE[EBS Storage]
    STORAGE --> GP3[gp3: $0.08/GB/month]
    STORAGE --> GP2[gp2: $0.10/GB/month]
    
    FG --> FGCPU[vCPU Pricing]
    FG --> FGMEM[Memory Pricing]
    
    FGCPU --> FGCPUCOST[$0.04048/vCPU/hour]
    FGMEM --> FGMEMCOST[$0.004445/GB/hour]
    
    CPMONTH --> EXAMPLE1[Example 1: 5-Day Course]
    T3MMONTH --> EXAMPLE1
    EXAMPLE1 --> EX1CALC[1 Cluster + 2 t3.medium nodes<br/>5 days x 24 hours]
    EX1CALC --> EX1COST[$12 CP + $10 Nodes = $22-25]
    
    CPMONTH --> EXAMPLE2[Example 2: 30-Day Production]
    T3MMONTH --> EXAMPLE2
    EXAMPLE2 --> EX2CALC[1 Cluster + 1 t3.medium node<br/>30 days continuous]
    EX2CALC --> EX2COST[$72 CP + $30 Node = $102-110]
    
    EX1COST --> TIPS[Cost Optimization Tips]
    EX2COST --> TIPS
    
    TIPS --> TIP1[Delete clusters when not in use]
    TIPS --> TIP2[Cannot stop EKS worker nodes]
    TIPS --> TIP3[Delete node groups separately]
    TIPS --> TIP4[Use Fargate for variable workloads]
    TIPS --> TIP5[Monitor with Cost Explorer]
    
    style CPCOST fill:#FF6B6B
    style T3MCOST fill:#4ECDC4
    style FGCPUCOST fill:#95E1D3
    style TIPS fill:#FFD700
```

### Diagram Explanation

- **Control Plane Cost**: Fixed **$0.10 per hour** ($72/month) regardless of cluster size, workload, or traffic volume - **no free tier available**
- **Worker Node Pricing**: Based on **EC2 instance type** and **running hours** - a single **t3.medium** costs approximately **$30/month** if running continuously
- **Multiple Node Cost**: With 2 t3.medium nodes, monthly cost is **$60 for compute** plus **$72 for control plane** = **$132 total minimum**
- **EBS Storage Costs**: Each node requires **root volume** (typically 20GB), charged at **$0.08-0.10/GB/month** for gp3/gp2 volumes
- **Fargate Pricing Model**: Pay for **actual vCPU and memory** consumed per second with **1-minute minimum**, ideal for **variable workloads**
- **Course Duration Cost**: For a **5-day learning period** with 2 worker nodes, expect approximately **$22-25** in total AWS charges
- **Cannot Stop Nodes**: Unlike regular EC2 instances, you **cannot stop** EKS worker nodes - must **delete the node group** to avoid charges
- **Cost Optimization Strategy**: **Delete entire cluster** when not actively learning, **recreate as needed** using eksctl for hands-on practice
- **Regional Pricing Variance**: Prices shown are for **us-east-1** (N. Virginia) - other regions may have **5-15% higher** instance costs
- **Hidden Costs**: Don't forget **data transfer**, **load balancer hours**, **EBS snapshots**, and **NAT gateway charges** in total calculation

## Steo-01: Very Important EKS Pricing Note
- EKS is not free (Unlike other AWS Services)
- In short, no free-tier for EKS.
### EKS Cluster Pricing
    - We pay $0.10 per hour for each Amazon EKS cluster
    - Per Day: $2.4
    - For 30 days: $72
### EKS Worker Nodes Pricing - EC2
    - You pay for AWS resources (e.g. EC2 instances or EBS volumes) 
    - T3 Medium Server in N.Virginia
        - $0.0416 per Hour
        - Per Day: $0.9984 - Approximately $1
        - Per Month: $30 per 1 t3.medium server
    - Reference: https://aws.amazon.com/ec2/pricing/on-demand/
    - In short, if we run 1 EKS Cluster and 1 t3.medium worker node **continuously** for 1 month, our bill is going to be around $102 to $110
    - If we take 5 days to complete this course, and if we run 1 EKS Cluster and 2 t3.medium Worker nodes continuosly for 5 days it will cost us approximately around $25. 
### EKS Fargate Profile
    - AWS Fargate pricing is calculated based on the **vCPU and memory** resources used from the time you start to download your container image until the EKS Pod terminates.
    - **Reference:** https://aws.amazon.com/fargate/pricing/    
    - Amazon EKS support for AWS Fargate is available in us-east-1, us-east-2, eu-west-1, and ap-northeast-1.

### Important Notes    
- **Important Note-1:** If you are using your personal AWS Account, then ensure you delete and recreate cluster and worker nodes as and when needed. 
- **Important Note-2:** We cant stop our EC2 Instances which are in Kubernetes cluster unlike regular EC2 Instances. So we need to delete the worker nodes (Node Group) if we are not using it during our learning process.
 