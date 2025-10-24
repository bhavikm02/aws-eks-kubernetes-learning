# Amazon EKS Pod Identity Agent - Demo

## Sequence Diagram

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant IAM as IAM Console
    participant EKS as EKS Console
    participant API as EKS API Server
    participant WEBHOOK as Pod Identity Webhook
    participant POD as AWS CLI Pod
    participant PIA as Pod Identity Agent<br/>(DaemonSet)
    participant AUTH as EKS Auth API
    participant S3 as Amazon S3

    DEV->>IAM: 1. Create IAM Role
    IAM->>IAM: Trust: pods.eks.amazonaws.com
    IAM->>IAM: Attach AmazonS3ReadOnlyAccess

    DEV->>EKS: 2. Create Pod Identity Association
    EKS->>EKS: Map: namespace/SA → IAM Role
    
    DEV->>API: 3. Deploy Pod (serviceAccount: aws-cli-sa)
    API->>WEBHOOK: Pod Creation Event
    WEBHOOK->>WEBHOOK: Mutate Pod Spec
    WEBHOOK->>POD: Inject AWS_CONTAINER_CREDENTIALS_FULL_URI
    WEBHOOK->>POD: Mount projected SA token
    
    POD->>POD: 4. Pod starts, AWS SDK initializes
    POD->>PIA: 5. Request credentials (via env vars)
    PIA->>AUTH: 6. AssumeRoleForPodIdentity API
    AUTH->>AUTH: 7. Validate Association
    AUTH->>PIA: 8. Return temporary credentials
    PIA->>POD: 9. Provide IAM credentials
    
    POD->>S3: 10. aws s3 ls (with credentials)
    S3->>POD: 11. List of S3 buckets ✓
    
    Note over POD,PIA: Credentials auto-refresh every 15 minutes
    Note over AUTH: Validates: Cluster + Namespace + ServiceAccount + IAM Role
```

### Diagram Explanation

- **Pod Identity Agent**: **DaemonSet** running on every node that acts as a **credential proxy** between pods and **AWS STS**
- **IAM Role Trust Policy**: Must trust **pods.eks.amazonaws.com** service principal, enabling EKS to assume roles on behalf of pods
- **Pod Identity Association**: Maps **Kubernetes ServiceAccount** in a **namespace** to an **IAM role** via EKS control plane
- **Webhook Mutation**: EKS **admission webhook** automatically injects **environment variables** and **token mounts** into pods using associated service accounts
- **Credential Injection**: Pod receives **AWS_CONTAINER_CREDENTIALS_FULL_URI** pointing to PIA endpoint and **token file path** for authentication
- **AssumeRoleForPodIdentity**: EKS-specific **STS API** that validates pod identity and returns **temporary IAM credentials** (15-min lifetime)
- **No Code Changes**: Applications using **AWS SDK** automatically discover credentials via **default credential chain** - zero code modification needed
- **Security Isolation**: Each pod gets **unique credentials** scoped to its service account, preventing **lateral movement** between workloads
- **Auto-Refresh**: Pod Identity Agent **automatically refreshes** credentials before expiration, ensuring **uninterrupted access** to AWS services
- **Simplified RBAC**: Replaces complex **IRSA annotations** with centralized **Pod Identity Associations** managed via EKS console or API

## Amazon EKS Pod Identity – High-Level Flow
Amazon EKS Pod Identity enables pods in your cluster to securely assume IAM roles without managing static credentials or using IRSA annotations. The high-level flow is shown below:

![EKS Pod Identity Components](images/04-00-PIA.png)

![EKS Pod Identity Flow](images/Pod-Identity-Worklow.jpg)

1. **Create IAM Role**  
   An IAM administrator creates a role that can be assumed by the new EKS service principal:  
   `pods.eks.amazonaws.com`.  
   - Trust policy can be restricted by cluster ARN or AWS account.  
   - Attach required IAM policies (e.g., AmazonS3ReadOnlyAccess).  

2. **Create Pod Identity Association**  
   The EKS administrator associates the IAM Role with a Kubernetes Service Account + Namespace.  
   - Done via the EKS Console or `CreatePodIdentityAssociation` API.  

3. **Webhook Mutation**  
   When a pod using that Service Account is created, the **EKS Pod Identity Webhook** (running in the control plane) mutates the pod spec:  
   - Injects environment variables such as `AWS_CONTAINER_CREDENTIALS_FULL_URI`.  
   - Mounts a projected service account token for use by the Pod Identity Agent.  

**Verify Mutation Example:**    
```bash
kubectl exec -it aws-cli -- env | grep AWS_CONTAINER
AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token
```


4. **Pod Requests Credentials**
   Inside the pod, the AWS SDK/CLI uses the default credential provider chain.

   * It discovers the injected environment variables and calls the **EKS Pod Identity Agent** running as a DaemonSet on the worker node.

   **4a. PIA Agent Calls EKS Auth API**

   * The Pod Identity Agent exchanges the projected token with the **EKS Auth API** using `AssumeRoleForPodIdentity`.

   **4b. EKS Auth API Validates Association**

   * The API checks the Pod Identity Association (Namespace + ServiceAccount → IAM Role).
   * If valid, it returns temporary IAM credentials back to the Pod Identity Agent.

5. **Pod Accesses AWS Resources**
   The AWS SDK/CLI inside the pod now has valid, short-lived credentials and can call AWS services (e.g., list S3 buckets).
   

---

### Key Notes
* Pods receive **temporary IAM credentials** automatically — no `aws configure` needed.
* Leverages the **standard AWS SDK credential chain** (no code changes required).
* Requires the **EKS Pod Identity Agent Add-on** running on worker nodes.
* Supported only with newer versions of AWS SDKs and CLI.

### Additional References
* [Amazon EKS Pod Identity Blog Post](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/)

---

## Step-00: What we’ll do
In this demo, we’ll understand and implement the **Amazon EKS Pod Identity Agent (PIA)**.

1. Install the **EKS Pod Identity Agent** add-on  
2. Create a **Kubernetes AWS CLI Pod** in the EKS Cluster and attempt to list S3 buckets (this will fail initially)  
3. Create an **IAM Role** with trust policy for Pod Identity → allow Pods to access **Amazon S3**  
4. Create a **Pod Identity Association** between the Kubernetes Service Account and IAM Role  
5. Re-test from the AWS CLI Pod, successfully list S3 buckets  
6. Through this flow, we will clearly understand how **Pod Identity Agent** works in EKS  

---

## Step-01: Install EKS Pod Identity Agent
1. Open **EKS Console** → **Clusters** → select your cluster (`eksdemo1`)  
2. Go to **Add-ons** → **Get more add-ons**  
3. Search for **EKS Pod Identity Agent**  
4. Click **Next** → **Create**  

This installs a **DaemonSet** (`eks-pod-identity-agent`) that enables Pod Identity associations.

```yaml
# List k8s PIA Resources
kubectl get daemonset -n kube-system

# List k8s Pods
kubectl get pods -n kube-system
```

---

## Step-02: Deploy AWS CLI Pod (without Pod Identity Association)
### Step-02-01: Create Service Account 
- ![01_k8s_service_account.yaml](kube-manifests/01_k8s_service_account.yaml)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-cli-sa
  namespace: default
```

### Step-02-02: Create a simple Kubernetes Pod with AWS CLI image:
- ![02_k8s_aws_cli_pod.yaml](kube-manifests/02_k8s_aws_cli_pod.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  namespace: default
spec:
  serviceAccountName: aws-cli-sa
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: ["sleep", "infinity"]
```

### Step-02-03: Deploy CLI Pod
```bash
kubectl apply -f kube-manifests/
kubectl get pods
```

### Step-02-04: Exec into the pod and try to list S3 buckets:
```bash
kubectl exec -it aws-cli -- aws s3 ls
```

**Observation:** ❌ This should **fail** because no IAM permissions are associated with the Pod.  

#### Error Message
```
kalyan-mini2:04-00-EKS-Pod-Identity-Agent kalyan$ kubectl exec -it aws-cli -- aws s3 ls

An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:sts::180789647333:assumed-role/eksctl-eksdemo1-nodegroup-eksdemo1-NodeInstanceRole-kxEPBrWVcOzO/i-0d65729d6d09d2540 is not authorized to perform: s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action
command terminated with exit code 254
```

**Important Note:**  Notice how the error references the **EC2 NodeInstanceRole** — this proves the Pod had no direct IAM mapping.

---

## Step-03: Create IAM Role for Pod Identity
1. Go to **IAM Console** → **Roles** → **Create Role**  
2. Select **Trusted entity type** → **Custom trust policy**  
3. Add trust policy for Pod Identity, for example:  

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
```

4. Attach **AmazonS3ReadOnlyAccess** policy  
5. Create role → example name: `EKS-PodIdentity-S3-ReadOnly-Role-101`  

---

## Step-04: Create Pod Identity Association
* Go to **EKS Console** → Cluster → **Access** → **Pod Identity Associations**  
* Create new association:  

  * Namespace: `default`  
  * Service Account: `aws-cli-sa`  
  * IAM Role: `EKS-PodIdentity-S3-ReadOnly-Role-101`  
  * Click on **create**  

---

## Step-05: Test Again

> Pods don’t automatically refresh credentials after a new Pod Identity Association; they must be restarted.

1. Restart Pod
```bash
# Delete Pod
kubectl delete pod aws-cli -n default

# Create Pod
kubectl apply -f kube-manifests/02_k8s_aws_cli_pod.yaml

# List Pods
kubectl get pods
```

2. Exec into the Pod:
```bash
# List S3 buckets
kubectl exec -it aws-cli -- aws s3 ls
```

**Observation:** This time it should **succeed**, listing all S3 buckets.

---

## Step-06: Clean Up

```bash
# 1. Delete the aws-cli pod
kubectl delete -f kube-manifests/
```

- **Remove Pod Identity Association** → via **EKS Console → Access → Pod Identity Associations**  
- **Remove IAM role** → via **IAM Console → Roles** `EKS-PodIdentity-S3-ReadOnly-Role-101`

---

## Step-07: Concept Recap

* **Without Pod Identity Association:** Pod has no IAM permissions → AWS API calls fail  
* **With Pod Identity Association:** Pod Identity Agent maps Pod’s Service Account → IAM Role → AWS Permissions → API calls succeed  

---



