# Delete EKS Cluster & Node Groups

## Deletion Workflow Diagram

```mermaid
graph TB
    START([Start Deletion Process]) --> IMPORTANT[Important: Rollback Changes First]
    
    IMPORTANT --> CHECK1{Modified Security Groups?}
    CHECK1 -->|Yes| ROLLBACK1[Remove custom inbound rules<br/>Restore to port 22 only]
    CHECK1 -->|No| CHECK2
    ROLLBACK1 --> CHECK2
    
    CHECK2{Modified IAM Policies?}
    CHECK2 -->|Yes| ROLLBACK2[Remove custom policies<br/>e.g., EBS CSI policies]
    CHECK2 -->|No| LISTNG
    ROLLBACK2 --> LISTNG
    
    LISTNG[List Resources] --> CMD1[eksctl get clusters]
    LISTNG --> CMD2[eksctl get nodegroup]
    
    CMD1 --> DELNG[Step 1: Delete Node Groups]
    CMD2 --> DELNG
    
    DELNG --> NGCMD[eksctl delete nodegroup<br/>--cluster=clusterName<br/>--name=nodegroupName]
    
    NGCMD --> NGCF[CloudFormation Deletes:]
    NGCF --> NG1[Terminate EC2 Instances]
    NGCF --> NG2[Delete Auto Scaling Group]
    NGCF --> NG3[Delete Launch Template]
    NGCF --> NG4[Remove IAM Instance Profile]
    NGCF --> NG5[Delete IAM Role & Policies]
    
    NG1 --> WAIT1[Wait for completion<br/>5-10 minutes]
    NG2 --> WAIT1
    NG3 --> WAIT1
    NG4 --> WAIT1
    NG5 --> WAIT1
    
    WAIT1 --> VERIFY1{Node Group Deleted?}
    VERIFY1 -->|No| ERROR1[Check CloudFormation Events<br/>Manual cleanup may be needed]
    VERIFY1 -->|Yes| DELCLUSTER
    
    ERROR1 --> MANUAL1[Manually delete stuck resources]
    MANUAL1 --> DELCLUSTER
    
    DELCLUSTER[Step 2: Delete Cluster] --> CLUSTERCMD[eksctl delete cluster<br/>clusterName]
    
    CLUSTERCMD --> CLUSTERCF[CloudFormation Deletes:]
    CLUSTERCF --> CL1[Delete Control Plane]
    CLUSTERCF --> CL2[Remove OIDC Provider]
    CLUSTERCF --> CL3[Delete VPC Resources]
    CLUSTERCF --> CL4[Delete Subnets]
    CLUSTERCF --> CL5[Delete Internet Gateway]
    CLUSTERCF --> CL6[Delete NAT Gateway]
    CLUSTERCF --> CL7[Delete Route Tables]
    CLUSTERCF --> CL8[Delete Security Groups]
    CLUSTERCF --> CL9[Delete Elastic IPs]
    
    CL1 --> WAIT2[Wait for completion<br/>10-15 minutes]
    CL2 --> WAIT2
    CL3 --> WAIT2
    CL4 --> WAIT2
    CL5 --> WAIT2
    CL6 --> WAIT2
    CL7 --> WAIT2
    CL8 --> WAIT2
    CL9 --> WAIT2
    
    WAIT2 --> VERIFY2{Cluster Deleted?}
    VERIFY2 -->|No| ERROR2[Check CloudFormation Events<br/>Dependencies may exist]
    VERIFY2 -->|Yes| FINALCHECK
    
    ERROR2 --> MANUAL2[Check for:<br/>- Lingering Load Balancers<br/>- Orphaned ENIs<br/>- Security Group Dependencies]
    MANUAL2 --> FINALCHECK
    
    FINALCHECK[Final Verification] --> V1[Verify no CloudFormation Stacks]
    FINALCHECK --> V2[Verify no EKS Clusters]
    FINALCHECK --> V3[Check for orphaned VPCs]
    FINALCHECK --> V4[Check AWS Cost Explorer]
    
    V1 --> SUCCESS([Deletion Complete])
    V2 --> SUCCESS
    V3 --> SUCCESS
    V4 --> SUCCESS
    
    style START fill:#90EE90
    style IMPORTANT fill:#FF6B6B
    style DELNG fill:#FFD700
    style DELCLUSTER fill:#FF9900
    style SUCCESS fill:#90EE90
    style ERROR1 fill:#FF6B6B
    style ERROR2 fill:#FF6B6B
```

### Diagram Explanation

- **Pre-Deletion Rollback**: Critical to **revert security group changes** and **IAM policy modifications** before deletion to avoid **stuck resources**
- **Order Matters**: Always delete **node groups first**, then **cluster** - reversing this order causes **CloudFormation errors** and **orphaned resources**
- **CloudFormation Orchestration**: eksctl uses **CloudFormation stacks** to track resources - deletion happens in **reverse dependency order**
- **Node Group Deletion**: Terminates **EC2 instances** gracefully, deletes **Auto Scaling Group**, **Launch Template**, and associated **IAM roles**
- **Deletion Timing**: Node groups take **5-10 minutes**, cluster deletion takes **10-15 minutes** - be patient and monitor **CloudFormation events**
- **Security Group Dependencies**: Custom rules added for **NodePort services** or **load balancers** can block deletion - **must be removed first**
- **IAM Policy Cleanup**: Custom policies like **EBS CSI Driver** or **ALB Controller** policies attached to node role **prevent clean deletion**
- **VPC Resource Cleanup**: Cluster deletion removes **subnets**, **route tables**, **NAT gateways**, **internet gateways**, and **security groups** automatically
- **Common Failure Causes**: Orphaned **ENIs** (Elastic Network Interfaces), **load balancers**, or **security group dependencies** can block VPC deletion
- **Cost Verification**: After deletion, check **AWS Cost Explorer** and **CloudFormation console** to ensure **no lingering resources** causing charges

## Step-01: Delete Node Group
- We can delete a nodegroup separately using below `eksctl delete nodegroup`
```
# List EKS Clusters
eksctl get clusters

# Capture Node Group name
eksctl get nodegroup --cluster=<clusterName>
eksctl get nodegroup --cluster=eksdemo1

# Delete Node Group
eksctl delete nodegroup --cluster=<clusterName> --name=<nodegroupName>
eksctl delete nodegroup --cluster=eksdemo1 --name=eksdemo1-ng-public1
```

## Step-02: Delete Cluster  
- We can delete cluster using `eksctl delete cluster`
```
# Delete Cluster
eksctl delete cluster <clusterName>
eksctl delete cluster eksdemo1
```

## Important Notes

### Note-1: Rollback any Security Group Changes
- When we create a EKS cluster using `eksctl` it creates the worker node security group with only port 22 access.
- When we progress through the course, we will be creating many **NodePort Services** to access and test our applications via browser. 
- During this process, we need to add an additional rule to this automatically created security group, allowing access to our applications we have deployed. 
- So the point we need to understand here is when we are deleting the cluster using `eksctl`, its core components should be in same state which means roll back the change we have done to security group before deleting the cluster.
- In this way, cluster will get deleted without any issues, else we might have issues and we need to refer cloudformation events and manually delete few things. In short, we need to go to many places for deletions. 

### Note-2: Rollback any EC2 Worker Node Instance Role - Policy changes
- When we are doing `EBS Storage Section with EBS CSI Driver` we will add a custom policy to worker node IAM role.
- When you are deleting the cluster, first roll back that change and delete it. 
- This way we don't face any issues during cluster deletion.
