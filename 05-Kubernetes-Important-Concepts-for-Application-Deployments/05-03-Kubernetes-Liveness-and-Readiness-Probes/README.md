# Kubernetes - Liveness & Readiness Probes

## Probes Architecture Diagram

```mermaid
graph TB
    POD[Application Pod] --> PROBES{Health Checks}
    
    PROBES --> LIVE[Liveness Probe]
    PROBES --> READY[Readiness Probe]
    PROBES --> STARTUP[Startup Probe]
    
    subgraph "Liveness Probe"
        LIVE --> LIVETYPE{Check Type}
        LIVETYPE -->|Exec| CMD[Command: nc -z localhost 8095]
        LIVETYPE -->|HTTP| HTTP1[HTTP GET: /healthz]
        LIVETYPE -->|TCP| TCP1[TCP Socket: port 8095]
        
        CMD --> LIVERESULT{Success?}
        HTTP1 --> LIVERESULT
        TCP1 --> LIVERESULT
        
        LIVERESULT -->|Yes| ALIVE[Container Alive]
        LIVERESULT -->|No| RESTART[Restart Container]
        
        RESTART -.->|After restart| POD
    end
    
    subgraph "Readiness Probe"
        READY --> READYTYPE{Check Type}
        READYTYPE -->|HTTP GET| HTTPGET[GET /usermgmt/health-status]
        READYTYPE -->|Exec| READYCMD[Command]
        READYTYPE -->|TCP| READYTCP[TCP Socket]
        
        HTTPGET --> READYRESULT{Success?}
        READYCMD --> READYRESULT
        READYTCP --> READYRESULT
        
        READYRESULT -->|Yes| ADDSVC[Add to Service Endpoints]
        READYRESULT -->|No| REMOVESVC[Remove from Service Endpoints]
    end
    
    subgraph "Startup Probe"
        STARTUP --> STARTCHECK[Check if app started]
        STARTCHECK --> STARTED{Started?}
        STARTED -->|Yes| ENABLELIVE[Enable Liveness Probe]
        STARTED -->|No| WAITSTART[Keep checking]
        WAITSTART -.->|Retry| STARTCHECK
    end
    
    subgraph "Probe Configuration"
        INIT[initialDelaySeconds: 60]
        PERIOD[periodSeconds: 10]
        TIMEOUT[timeoutSeconds: 5]
        SUCCESS[successThreshold: 1]
        FAILURE[failureThreshold: 3]
    end
    
    subgraph "Common Issues"
        ISS1[Too short initialDelay → Premature restarts]
        ISS2[Long periodSeconds → Slow failure detection]
        ISS3[Liveness == Readiness → Cascading failures]
        ISS4[Heavy probes → Resource overhead]
    end
    
    LIVE -.->|Uses| INIT
    LIVE -.->|Uses| PERIOD
    READY -.->|Uses| INIT
    READY -.->|Uses| PERIOD
    
    style ALIVE fill:#90EE90
    style RESTART fill:#FF6B6B
    style ADDSVC fill:#90EE90
    style REMOVESVC fill:#FFD700
    style ENABLELIVE fill:#4A90E2
```

### Diagram Explanation

- **Liveness Probe**: Detects **deadlocked** or **unresponsive** containers, kubelet **restarts** container when probe fails repeatedly
- **Readiness Probe**: Determines if container can **serve traffic**, removes pod from **service endpoints** when not ready, preventing failed requests
- **Startup Probe**: For **slow-starting** applications, disables liveness checks until app is **fully initialized**, prevents premature kills
- **HTTP GET Probe**: Sends HTTP request to specified **path and port**, success if response code is **200-399**
- **Exec Command Probe**: Executes command inside container, success if exit code is **0** (e.g., nc -z checks port availability)
- **TCP Socket Probe**: Attempts TCP connection to **specified port**, success if connection established, useful for non-HTTP services
- **initialDelaySeconds**: Wait time before **first probe**, must be longer than application **startup time** to avoid false failures
- **periodSeconds**: How often to perform the probe, balance between **quick detection** (low value) and **resource usage** (high value)
- **failureThreshold**: Number of **consecutive failures** before taking action (restart for liveness, remove from service for readiness)
- **Best Practice**: Readiness should be **lighter weight** than liveness, use **different endpoints** to avoid cascading failures during high load

## Step-01: Introduction
- Refer `Probes` slide for additional details

## Step-02: Create Liveness Probe with Command
```yml
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - nc -z localhost 8095
            initialDelaySeconds: 60
            periodSeconds: 10
```

## Step-03: Create Readiness Probe with HTTP GET
```yml
          readinessProbe:
            httpGet:
              path: /usermgmt/health-status
              port: 8095
            initialDelaySeconds: 60
            periodSeconds: 10     
```


## Step-04: Create k8s objects & Test
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
- **Observation:** User Management Microservice pod witll not be in READY state to accept traffic until it completes the `initialDelaySeconds=60seconds`. 

## Step-05: Clean-Up
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
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
