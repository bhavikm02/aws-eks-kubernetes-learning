# EKS - Create EKS Node Group in Private Subnets

## Private Node Group Architecture Diagram

```mermaid
graph TB
    INET[Internet] -->|HTTPS| IGW[Internet Gateway]
    IGW -->|Public Subnet| ALB[Application Load Balancer<br/>Public Subnets]
    
    ALB -->|Private IP| PRIVATE[Private Subnet Worker Nodes]
    
    subgraph "VPC"
        subgraph "Public Subnets"
            ALB
            NAT1[NAT Gateway AZ-1]
            NAT2[NAT Gateway AZ-2]
        end
        
        subgraph "Private Subnets AZ-1"
            NODE1[Worker Node 1<br/>No External IP]
            POD1[Application Pods]
            NODE1 --> POD1
        end
        
        subgraph "Private Subnets AZ-2"
            NODE2[Worker Node 2<br/>No External IP]
            POD2[Application Pods]
            NODE2 --> POD2
        end
        
        NODE1 -->|Outbound Traffic| NAT1
        NODE2 -->|Outbound Traffic| NAT2
        
        NAT1 -->|Internet Access| IGW
        NAT2 -->|Internet Access| IGW
    end
    
    ALB -->|NodePort| NODE1
    ALB -->|NodePort| NODE2
    
    subgraph "Node Group Configuration"
        NG[Private Node Group<br/>eksdemo1-ng-private1]
        NG --> FLAG[--node-private-networking]
        NG --> SUBNET[Subnet Selection:<br/>Private Subnets Only]
        NG --> SIZE[Instance Type: t3.medium]
        NG --> ASG[Auto Scaling: min 2, max 4]
    end
    
    subgraph "Private Node Benefits"
        B1[Enhanced Security]
        B2[No direct internet exposure]
        B3[Controlled egress via NAT]
        B4[Better compliance posture]
        B5[Internal-only workloads]
    end
    
    subgraph "Networking Flow"
        INBOUND[Inbound: Internet → IGW → ALB → Nodes]
        OUTBOUND[Outbound: Nodes → NAT → IGW → Internet]
    end
    
    subgraph "Key Differences: Private vs Public"
        PUB[Public Node Group:<br/>External IP on nodes<br/>Direct internet access]
        PRIV[Private Node Group:<br/>No external IP<br/>NAT Gateway for outbound]
    end
    
    subgraph "NAT Gateway Components"
        NATGW[NAT Gateway]
        NATGW --> EIP[Elastic IP Address]
        NATGW --> AVAILABILITY[High Availability<br/>One per AZ]
        NATGW --> COST[Cost: Per hour + Data transfer]
    end
    
    subgraph "Security Considerations"
        SG[Security Groups<br/>Control traffic]
        NACL[Network ACLs<br/>Subnet-level rules]
        PRIVATE_ONLY[No SSH from internet]
        BASTION[Bastion host for access<br/>Optional]
    end
    
    subgraph "IAM Permissions"
        NODE_ROLE[Node IAM Role]
        NODE_ROLE --> ECR[ECR pull images]
        NODE_ROLE --> CW[CloudWatch logs]
        NODE_ROLE --> ASG_PERM[Auto Scaling access]
        NODE_ROLE --> ALB_PERM[ALB Ingress access]
        NODE_ROLE --> DNS[External DNS access]
    end
    
    NODE1 -.->|Uses| NODE_ROLE
    NODE2 -.->|Uses| NODE_ROLE
    
    style ALB fill:#FF9900
    style NODE1 fill:#4A90E2
    style NODE2 fill:#4A90E2
    style NAT1 fill:#2E8B57
    style NAT2 fill:#2E8B57
    style IGW fill:#FFD700
```

### Diagram Explanation

- **Private Node Group**: Worker nodes deployed in **private subnets** with no external IP addresses, enhanced security posture
- **--node-private-networking Flag**: eksctl option that places nodes in **private subnets** and configures NAT Gateway routing
- **NAT Gateway Outbound**: Nodes use **NAT Gateway** in public subnet for outbound internet access (pulling images, API calls)
- **Load Balancer Placement**: ALB/NLB created in **public subnets**, routes traffic to private nodes via NodePort or TargetGroup
- **High Availability**: NAT Gateway in **each AZ** ensures nodes in that AZ maintain internet access if other AZ fails
- **No Direct Internet Access**: Nodes cannot receive **inbound connections** from internet, only via load balancer
- **Security Groups**: Control **inbound traffic** to nodes, typically only from load balancer and internal services
- **Cost Considerations**: **NAT Gateway charges** per hour plus data transfer, but provides security and isolation
- **Bastion Host**: Optional **jump server** in public subnet for SSH access to private nodes for troubleshooting
- **Compliance Ready**: Private networking meets **compliance requirements** for sensitive workloads, reduces attack surface

## Step-01: Introduction
- We are going to create a node group in VPC Private Subnets
- We are going to deploy workloads on the private node group wherein workloads will be running private subnets and load balancer gets created in public subnet and accessible via internet.

## Step-02: Delete existing Public Node Group in EKS Cluster
```
# Get NodeGroups in a EKS Cluster
eksctl get nodegroup --cluster=<Cluster-Name>
eksctl get nodegroup --cluster=eksdemo1

# Delete Node Group - Replace nodegroup name and cluster name
eksctl delete nodegroup <NodeGroup-Name> --cluster <Cluster-Name>
eksctl delete nodegroup eksdemo1-ng-public1 --cluster eksdemo1
```

## Step-03: Create EKS Node Group in Private Subnets
- Create Private Node Group in a Cluster
- Key option for the command is `--node-private-networking`

```
eksctl create nodegroup --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-private1 \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking                       
```

## Step-04: Verify if Node Group created in Private Subnets

### Verify External IP Address for Worker Nodes
- External IP Address should be none if our Worker Nodes created in Private Subnets
```
kubectl get nodes -o wide
```
### Subnet Route Table Verification - Outbound Traffic goes via NAT Gateway
- Verify the node group subnet routes to ensure it created in private subnets
  - Go to Services -> EKS -> eksdemo -> eksdemo1-ng1-private
  - Click on Associated subnet in **Details** tab
  - Click on **Route Table** Tab.
  - We should see that internet route via NAT Gateway (0.0.0.0/0 -> nat-xxxxxxxx)
