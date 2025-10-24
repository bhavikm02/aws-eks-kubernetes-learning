# Kubernetes - Secrets

## Secrets Workflow Diagram

```mermaid
graph TB
    START([Sensitive Data]) --> ENCODE[Base64 Encode]
    
    ENCODE --> B64[echo -n 'dbpassword11' | base64<br/>Result: ZGJwYXNzd29yZDEx]
    
    B64 --> CREATE[Create Secret Object]
    
    CREATE --> SECRET[Secret: mysql-db-password<br/>Type: Opaque]
    SECRET --> KEY[Key: db-password<br/>Value: ZGJwYXNzd29yZDEx]
    
    SECRET --> USAGE{How to Use?}
    
    USAGE -->|Option 1| ENV[Environment Variable]
    USAGE -->|Option 2| VOL[Volume Mount]
    
    ENV --> MYSQL[MySQL Deployment]
    ENV --> UMS[User Management Deployment]
    
    MYSQL --> MYSQLENV[MYSQL_ROOT_PASSWORD]
    MYSQLENV --> DECODE1[Auto-decoded at runtime]
    
    UMS --> UMSENV[DB_PASSWORD]
    UMSENV --> DECODE2[Auto-decoded at runtime]
    
    VOL --> MOUNT[Mount as File in Container]
    MOUNT --> FILE[/etc/secrets/db-password]
    
    DECODE1 --> APP1[Application Access:<br/>Plain text password]
    DECODE2 --> APP2[Application Access:<br/>Plain text password]
    FILE --> APP3[Read from file:<br/>Plain text password]
    
    subgraph "Security Features"
        SF1[Not stored in image]
        SF2[Not in deployment YAML]
        SF3[Encrypted at rest etcd]
        SF4[RBAC controlled access]
        SF5[Namespace scoped]
    end
    
    SECRET -.->|Benefits| SF1
    SECRET -.->|Benefits| SF2
    SECRET -.->|Benefits| SF3
    SECRET -.->|Benefits| SF4
    SECRET -.->|Benefits| SF5
    
    subgraph "Best Practices"
        BP1[Use external secret managers<br/>AWS Secrets Manager, Vault]
        BP2[Never commit secrets to Git]
        BP3[Rotate secrets regularly]
        BP4[Use sealed secrets for GitOps]
        BP5[Minimize secret scope]
    end
    
    style SECRET fill:#FF6B6B
    style DECODE1 fill:#90EE90
    style DECODE2 fill:#90EE90
    style SF1 fill:#4A90E2
    style SF2 fill:#4A90E2
    style SF3 fill:#4A90E2
```

### Diagram Explanation

- **Base64 Encoding**: Secrets are **base64-encoded** (not encrypted) to handle binary data, easily reversible so **not secure by itself**
- **Opaque Secret Type**: Default secret type for **arbitrary key-value pairs**, other types include **TLS**, **dockerconfigjson**, and **service-account-token**
- **Environment Variable Injection**: Kubernetes **automatically decodes** base64 and injects plain text value into container environment variables
- **secretKeyRef**: References specific **key** from named **secret**, allows selective exposure of secret data to containers
- **Volume Mount Method**: Mounts secrets as **files** in container filesystem, useful for **multi-line secrets** like certificates or config files
- **etcd Encryption**: Enable **encryption at rest** in etcd to protect secrets, requires configuration of **encryption provider** in API server
- **RBAC Access Control**: Use **Role-Based Access Control** to restrict which **service accounts** and **users** can read secrets
- **Namespace Isolation**: Secrets are **namespace-scoped**, pods can only access secrets in **same namespace**, provides isolation between teams
- **External Secret Managers**: For production, use **AWS Secrets Manager**, **HashiCorp Vault**, or **External Secrets Operator** for better security
- **Sealed Secrets**: Use **Bitnami Sealed Secrets** for GitOps workflows, encrypts secrets so they can be **safely stored in Git**

## Step-01: Introduction
- Kubernetes Secrets let you store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys. 
- Storing confidential information in a Secret is safer and more flexible than putting it directly in a Pod definition or in a container image. 

## Step-02: Create Secret for MySQL DB Password
### 
```
# Mac
echo -n 'dbpassword11' | base64

# URL: https://www.base64encode.org
```
### Create Kubernetes Secrets manifest
```yml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-db-password
#type: Opaque means that from kubernetes's point of view the contents of this Secret is unstructured.
#It can contain arbitrary key-value pairs. 
type: Opaque
data:
  # Output of echo -n 'dbpassword11' | base64
  db-password: ZGJwYXNzd29yZDEx
```
## Step-03: Update secret in MySQL Deployment for DB Password
```yml
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-db-password
                  key: db-password
```

## Step-04: Update secret in UMS Deployment
- UMS means User Management Microservice
```yml
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-db-password
                  key: db-password
```

## Step-05: Create & Test
```
# Create All Objects
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Access Application Health Status Page
http://<WorkerNode-Public-IP>:31231/usermgmt/health-status
```

## Step-06: Clean-Up
- Delete all k8s objects created as part of this section
```
# Delete All
kubectl delete -f kube-manifests/

# List Pods
kubectl get pods

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```