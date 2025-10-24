# Monitoring EKS using CloudWatch Container Insigths

## CloudWatch Container Insights Architecture Diagram

```mermaid
graph TB
    subgraph "EKS Cluster"
        subgraph "Worker Node 1"
            POD1[Application Pods]
            KUBELET1[kubelet<br/>cAdvisor]
            CW_AGENT1[CloudWatch Agent<br/>DaemonSet]
            FLUENTD1[Fluentd<br/>DaemonSet]
        end
        
        subgraph "Worker Node 2"
            POD2[Application Pods]
            KUBELET2[kubelet<br/>cAdvisor]
            CW_AGENT2[CloudWatch Agent<br/>DaemonSet]
            FLUENTD2[Fluentd<br/>DaemonSet]
        end
        
        POD1 -->|Metrics| KUBELET1
        POD1 -->|Stdout/Stderr| DOCKER_LOG1[Container Logs]
        
        KUBELET1 -->|Scrape| CW_AGENT1
        DOCKER_LOG1 -->|Read| FLUENTD1
        
        POD2 -->|Metrics| KUBELET2
        POD2 -->|Stdout/Stderr| DOCKER_LOG2[Container Logs]
        
        KUBELET2 -->|Scrape| CW_AGENT2
        DOCKER_LOG2 -->|Read| FLUENTD2
    end
    
    subgraph "Amazon CloudWatch"
        CW_AGENT1 -->|Send Metrics| CW_METRICS[CloudWatch Metrics]
        CW_AGENT2 -->|Send Metrics| CW_METRICS
        
        FLUENTD1 -->|Send Logs| CW_LOGS[CloudWatch Logs]
        FLUENTD2 -->|Send Logs| CW_LOGS
        
        CW_METRICS --> METRICS_TYPES[Metric Types]
        METRICS_TYPES --> M1[CPU Utilization]
        METRICS_TYPES --> M2[Memory Utilization]
        METRICS_TYPES --> M3[Network I/O]
        METRICS_TYPES --> M4[Disk I/O]
        METRICS_TYPES --> M5[Pod/Container Count]
        
        CW_LOGS --> LOG_GROUPS[Log Groups]
        LOG_GROUPS --> LG1[/aws/containerinsights/cluster/application]
        LOG_GROUPS --> LG2[/aws/containerinsights/cluster/dataplane]
        LOG_GROUPS --> LG3[/aws/containerinsights/cluster/host]
        LOG_GROUPS --> LG4[/aws/containerinsights/cluster/performance]
    end
    
    subgraph "CloudWatch Features"
        DASHBOARD[Container Insights Dashboard]
        INSIGHTS_Q[CloudWatch Logs Insights<br/>Query Language]
        ALARMS[CloudWatch Alarms]
        SNS[SNS Notifications]
    end
    
    CW_METRICS --> DASHBOARD
    CW_LOGS --> DASHBOARD
    CW_LOGS --> INSIGHTS_Q
    CW_METRICS --> ALARMS
    ALARMS --> SNS
    
    subgraph "IAM Permissions"
        NODE_ROLE[Worker Node IAM Role]
        NODE_ROLE --> CW_POLICY[CloudWatchAgentServerPolicy]
        CW_POLICY --> PERMS[PutMetricData<br/>CreateLogGroup<br/>PutLogEvents]
    end
    
    CW_AGENT1 -.->|Uses| NODE_ROLE
    FLUENTD1 -.->|Uses| NODE_ROLE
    
    subgraph "Sample Queries"
        Q1[Average Node CPU]
        Q2[Container Restarts]
        Q3[Pod Failures]
        Q4[Error Log Count]
    end
    
    INSIGHTS_Q --> Q1
    INSIGHTS_Q --> Q2
    
    style CW_AGENT1 fill:#FF9900
    style FLUENTD1 fill:#4A90E2
    style CW_METRICS fill:#FF6B6B
    style CW_LOGS fill:#2E8B57
    style DASHBOARD fill:#9B59B6
```

### Diagram Explanation

- **CloudWatch Agent DaemonSet**: Runs on **every node**, scrapes metrics from **kubelet cAdvisor** and sends to CloudWatch Metrics
- **Fluentd DaemonSet**: Collects **container logs** from /var/log/containers, enriches with Kubernetes metadata, sends to CloudWatch Logs
- **Container Insights Dashboard**: Pre-built **visual dashboard** showing cluster, node, pod, and container-level metrics automatically
- **Performance Metrics**: Tracks **CPU**, **memory**, **network**, **disk I/O** at cluster, node, namespace, pod, and container levels
- **Log Groups Structure**: Organizes logs into **application** (stdout/stderr), **dataplane** (kubelet/kube-proxy), **host** (system), **performance** (metrics)
- **CloudWatch Logs Insights**: Query language for **searching and analyzing** logs with aggregations, filters, and visualizations
- **Metric Filters**: Extract **custom metrics** from log patterns, create alarms on application-specific events
- **IAM Role Required**: Worker nodes need **CloudWatchAgentServerPolicy** to write metrics and logs to CloudWatch
- **Cost Optimization**: Consider **log retention policies** and **metric resolution** to manage costs, use sampling for high-volume applications
- **Alerting Integration**: Create **CloudWatch Alarms** on metrics, send notifications to **SNS** for email/SMS, integrate with PagerDuty/Slack

## Step-01: Introduction
- What is CloudWatch?
- What are CloudWatch Container Insights?
- What is CloudWatch Agent and Fluentd?

## Step-02: Associate CloudWatch Policy to our EKS Worker Nodes Role
- Go to Services -> EC2 -> Worker Node EC2 Instance -> IAM Role -> Click on that role
```
# Sample Role ARN
arn:aws:iam::180789647333:role/eksctl-eksdemo1-nodegroup-eksdemo-NodeInstanceRole-1FVWZ2H3TMQ2M

# Policy to be associated
Associate Policy: CloudWatchAgentServerPolicy
```

## Step-03: Install Container Insights

### Deploy CloudWatch Agent and Fluentd as DaemonSets
- This command will 
  - Creates the Namespace amazon-cloudwatch.
  - Creates all the necessary security objects for both DaemonSet:
    - SecurityAccount
    - ClusterRole
    - ClusterRoleBinding
  - Deploys `Cloudwatch-Agent` (responsible for sending the metrics to CloudWatch) as a DaemonSet.
  - Deploys fluentd (responsible for sending the logs to Cloudwatch) as a DaemonSet.
  -  Deploys ConfigMap configurations for both DaemonSets.
```
# Template
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/<REPLACE_CLUSTER_NAME>/;s/{{region_name}}/<REPLACE-AWS_REGION>/" | kubectl apply -f -

# Replaced Cluster Name and Region
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/eksdemo1/;s/{{region_name}}/us-east-1/" | kubectl apply -f -
```

## Verify
```
# List Daemonsets
kubectl -n amazon-cloudwatch get daemonsets
```


## Step-04: Deploy Sample Nginx Application
```
# Deploy
kubectl apply -f kube-manifests
```

## Step-05: Generate load on our Sample Nginx Application
```
# Generate Load
kubectl run --generator=run-pod/v1 apache-bench -i --tty --rm --image=httpd -- ab -n 500000 -c 1000 http://sample-nginx-service.default.svc.cluster.local/ 
```

## Step-06: Access CloudWatch Dashboard 
- Access CloudWatch Container Insigths Dashboard


## Step-07: CloudWatch Log Insights
- View Container logs
- View Container Performance Logs

## Step-08: Container Insights  - Log Insights in depth
- Log Groups
- Log Insights
- Create Dashboard

### Create Graph for Avg Node CPU Utlization
- DashBoard Name: EKS-Performance
- Widget Type: Bar
- Log Group: /aws/containerinsights/eksdemo1/performance
```
STATS avg(node_cpu_utilization) as avg_node_cpu_utilization by NodeName
| SORT avg_node_cpu_utilization DESC 
```

### Container Restarts
- DashBoard Name: EKS-Performance
- Widget Type: Table
- Log Group: /aws/containerinsights/eksdemo1/performance
```
STATS avg(number_of_container_restarts) as avg_number_of_container_restarts by PodName
| SORT avg_number_of_container_restarts DESC
```

### Cluster Node Failures
- DashBoard Name: EKS-Performance
- Widget Type: Table
- Log Group: /aws/containerinsights/eksdemo1/performance
```
stats avg(cluster_failed_node_count) as CountOfNodeFailures 
| filter Type="Cluster" 
| sort @timestamp desc
```
### CPU Usage By Container
- DashBoard Name: EKS-Performance
- Widget Type: Bar
- Log Group: /aws/containerinsights/eksdemo1/performance
```
stats pct(container_cpu_usage_total, 50) as CPUPercMedian by kubernetes.container_name 
| filter Type="Container"
```

### Pods Requested vs Pods Running
- DashBoard Name: EKS-Performance
- Widget Type: Bar
- Log Group: /aws/containerinsights/eksdemo1/performance
```
fields @timestamp, @message 
| sort @timestamp desc 
| filter Type="Pod" 
| stats min(pod_number_of_containers) as requested, min(pod_number_of_running_containers) as running, ceil(avg(pod_number_of_containers-pod_number_of_running_containers)) as pods_missing by kubernetes.pod_name 
| sort pods_missing desc
```

### Application log errors by container name
- DashBoard Name: EKS-Performance
- Widget Type: Bar
- Log Group: /aws/containerinsights/eksdemo1/application
```
stats count() as countoferrors by kubernetes.container_name 
| filter stream="stderr" 
| sort countoferrors desc
```

- **Reference**: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-view-metrics.html


## Step-09: Container Insights - CloudWatch Alarms
### Create Alarms - Node CPU Usage
- **Specify metric and conditions**
  - **Select Metric:** Container Insights -> ClusterName -> node_cpu_utilization
  - **Metric Name:** eksdemo1_node_cpu_utilization
  - **Threshold Value:** 4 
  - **Important Note:** Anything above 4% of CPU it will send a notification email, ideally it should 80% or 90% CPU but we are giving 4% CPU just for load simulation testing 
- **Configure Actions**
  - **Create New Topic:** eks-alerts
  - **Email:** dkalyanreddy@gmail.com
  - Click on **Create Topic**
  - **Important Note:**** Complete Email subscription sent to your email id.
- **Add name and description**
  - **Name:** EKS-Nodes-CPU-Alert
  - **Descritption:** EKS Nodes CPU alert notification  
  - Click Next
- **Preview**
  - Preview and Create Alarm
- **Add Alarm to our custom Dashboard**
- Generate Load & Verify Alarm
```
# Generate Load
kubectl run --generator=run-pod/v1 apache-bench -i --tty --rm --image=httpd -- ab -n 500000 -c 1000 http://sample-nginx-service.default.svc.cluster.local/ 
```

## Step-10: Clean-Up Container Insights
```
# Template
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/cluster-name/;s/{{region_name}}/cluster-region/" | kubectl delete -f -

# Replace Cluster Name & Region Name
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/eksdemo1/;s/{{region_name}}/us-east-1/" | kubectl delete -f -
```

## Step-11: Clean-Up Application
```
# Delete Apps
kubectl delete -f  kube-manifests/
```

## References
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights-Prometheus-Setup.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-reference-performance-entries-EKS.html

