# Kubernetes - Init Containers

## Init Container Flow Diagram

```mermaid
graph TB
    START([Pod Creation]) --> INIT1[Init Container 1:<br/>init-db]
    
    INIT1 --> CHECK[Check MySQL Availability]
    CHECK --> NC[nc -z mysql 3306]
    
    NC --> READY{MySQL Ready?}
    READY -->|No| WAIT[Sleep 1 second<br/>Print "-"]
    WAIT --> NC
    READY -->|Yes| SUCCESS1[Init Container 1: Completed ✓]
    
    SUCCESS1 --> INIT2{More Init Containers?}
    INIT2 -->|Yes| NEXT[Start Init Container 2]
    NEXT --> NEXTSUCCESS[Init Container 2: Completed ✓]
    
    INIT2 -->|No| APPSTART[Start Main Container]
    NEXTSUCCESS --> APPSTART
    
    APPSTART --> APP[User Management App Running]
    
    subgraph "Init Container Features"
        F1[Runs to completion before app]
        F2[Sequential execution]
        F3[Contains utilities not in app image]
        F4[Setup & dependency checks]
        F5[Failure causes pod restart]
    end
    
    subgraph "Use Cases"
        UC1[Wait for external services]
        UC2[Database migrations]
        UC3[Configuration generation]
        UC4[Security setup]
        UC5[Clone git repositories]
        UC6[Warm up caches]
    end
    
    subgraph "Failure Handling"
        FAIL[Init Container Fails]
        POLICY{restartPolicy?}
        FAIL --> POLICY
        POLICY -->|Always| RESTART[Restart Pod]
        POLICY -->|Never| FAILED[Pod Failed State]
        RESTART -.->|Retry| INIT1
    end
    
    style INIT1 fill:#4A90E2
    style SUCCESS1 fill:#90EE90
    style APP fill:#2E8B57
    style FAIL fill:#FF6B6B
    style RESTART fill:#FFD700
```

### Diagram Explanation

- **Sequential Execution**: Init containers run **one at a time** in the order defined, each must **complete successfully** before the next starts
- **Runs to Completion**: Unlike app containers, init containers must **finish their task** and exit with status code 0 before pod proceeds
- **Separate Images**: Init containers use **different images** with utilities like **busybox**, **curl**, **git** not needed in main application
- **Dependency Checking**: Common use case is waiting for **external services** to be available (databases, APIs) before starting main app
- **Database Initialization**: Can run **schema migrations**, **seed data**, or create **database users** before application starts
- **Restart on Failure**: If init container fails, Kubernetes **restarts the entire pod** (unless restartPolicy is Never), retrying the initialization
- **Resource Isolation**: Init containers have **separate resource requests/limits**, don't compete with app containers for resources
- **Volume Sharing**: Init containers can **write to shared volumes** that app containers read, useful for **configuration generation**
- **Security Benefits**: Keeps **sensitive setup tools** and **credentials** out of main application image, improving **security posture**
- **Status Visibility**: `kubectl describe pod` shows init container **status**, **logs**, and **exit codes** for troubleshooting

## Step-01: Introduction
- Init Containers run **before** App containers
- Init containers can contain **utilities or setup scripts** not present in an app image.
- We can have and run **multiple Init Containers** before App Container. 
- Init containers are exactly like regular containers, **except:**
  - Init containers always **run to completion.**
  - Each init container must **complete successfully** before the next one starts.
- If a Pod's init container fails, Kubernetes repeatedly restarts the Pod until the init container succeeds.
- However, if the Pod has a `restartPolicy of Never`, Kubernetes does not restart the Pod.


## Step-02: Implement Init Containers
- Update `initContainers` section under Pod Template Spec which is `spec.template.spec` in a Deployment
```yml
  template:
    metadata:
      labels:
        app: usermgmt-restapp
    spec:
      initContainers:
        - name: init-db
          image: busybox:1.31
          command: ['sh', '-c', 'echo -e "Checking for the availability of MySQL Server deployment"; while ! nc -z mysql 3306; do sleep 1; printf "-"; done; echo -e "  >> MySQL DB Server has started";']
```


## Step-03: Create & Test
```
# Create All Objects
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Watch List Pods screen
kubectl get pods -w

# Describe Pod & Discuss about init container
kubectl describe pod <usermgmt-microservice-xxxxxx>

# Access Application Health Status Page
http://<WorkerNode-Public-IP>:31231/usermgmt/health-status
```

## Step-04: Clean-Up
- Delete all k8s objects created as part of this section
```
# Delete All
kubectl delete -f kube-manifests/

# List Pods
kubectl get pods

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```

## References:
- https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
