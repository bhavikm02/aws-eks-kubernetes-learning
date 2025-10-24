# Microservices Deployment on EKS

## Microservices Architecture Diagram

```mermaid
graph TB
    USER[End Users] -->|HTTPS| ALB[Application Load Balancer<br/>SSL Termination]
    ALB -->|Route 53 DNS| DNS[External DNS<br/>services.kubeoncloud.com]
    
    ALB -->|Forward to| INGRESS[Ingress Controller]
    INGRESS -->|Route to| UMSSVC[User Management Service<br/>NodePort 8095]
    
    UMSSVC -->|Load Balance| UMS1[UMS Pod 1]
    UMSSVC -->|Load Balance| UMS2[UMS Pod 2]
    
    UMS1 -->|DB Operations| MYSQL_SVC[MySQL ExternalName Service]
    UMS2 -->|DB Operations| MYSQL_SVC
    
    MYSQL_SVC -->|DNS Lookup| RDS[AWS RDS MySQL<br/>usermgmtdb.rds.amazonaws.com]
    
    UMS1 -->|Send Notification API| NOTIF_SVC[Notification ClusterIP Service<br/>Port 8096]
    UMS2 -->|Send Notification API| NOTIF_SVC
    
    NOTIF_SVC -->|Load Balance| NOTIF1[Notification Pod V1]
    NOTIF_SVC -->|Load Balance| NOTIF2[Notification Pod V2]
    
    NOTIF1 -->|SMTP| SMTP_SVC[SMTP ExternalName Service]
    NOTIF2 -->|SMTP| SMTP_SVC
    
    SMTP_SVC -->|DNS Lookup| SES[AWS SES<br/>email-smtp.us-east-1.amazonaws.com]
    
    SES -->|Email| RECIPIENT[User Email]
    
    subgraph "Microservices Communication"
        SYNC[Synchronous REST API]
        SERVICE_DISC[Service Discovery via DNS]
        CLUSTERIP[ClusterIP for internal]
        EXTERNALNAME[ExternalName for AWS services]
    end
    
    subgraph "API Endpoints"
        API1[POST /usermgmt/user<br/>Create User]
        API2[GET /usermgmt/users<br/>List Users]
        API3[POST /notification/send<br/>Send Email]
        API4[GET /usermgmt/health-status]
    end
    
    UMS1 -.->|Implements| API1
    UMS1 -.->|Implements| API2
    NOTIF1 -.->|Implements| API3
    
    subgraph "Deployment Strategies"
        ROLLING[Rolling Update]
        CANARY[Canary Deployment]
        BLUE_GREEN[Blue-Green]
        KUBECTL[kubectl set image]
    end
    
    style UMS1 fill:#4A90E2
    style NOTIF1 fill:#9B59B6
    style RDS fill:#527FFF
    style SES fill:#FF9900
    style ALB fill:#FF6B6B
```

### Diagram Explanation

- **Service-to-Service Communication**: User Management calls Notification Service via **ClusterIP**, using **service name** (notification-clusterip-service) for **DNS resolution**
- **ExternalName Service**: Maps Kubernetes service name to **external DNS** (RDS endpoint, SES SMTP), enables seamless **AWS service integration**
- **Synchronous REST API**: UMS makes **HTTP POST** to Notification Service on user creation, waits for response, **tight coupling** for immediate feedback
- **RDS Integration**: MySQL **ExternalName service** resolves to **RDS endpoint**, eliminates pod-based database, provides **managed backup** and **high availability**
- **SES SMTP**: Notification Service connects to **AWS SES** via SMTP protocol, requires **SMTP credentials** and **verified email addresses**
- **ClusterIP for Internal**: Notification Service uses **ClusterIP**, not exposed externally, accessible only from **within cluster**
- **ALB Ingress**: Exposes only User Management Service externally via **ALB**, Notification Service remains **internal-only microservice**
- **Service Discovery**: Kubernetes **CoreDNS** resolves service names to **ClusterIP addresses**, enables location-independent service calls
- **Rolling Update**: Deploy new versions using **kubectl set image** or **kubectl apply**, gradual replacement of old pods with new ones
- **Health Endpoints**: Both services expose **/health-status** for **readiness/liveness probes** and **external monitoring**

## Step-00: What are Microservices?
- Understand what are microservices on  a very high level

## Step-01: What are we going to learn in this section?
- We are going to deploy two microservices.
    - User Management Service
    - Notification Service

### Usecase Description
- User Management **Create User API**  will call Notification service **Send Notification API** to send an email to user when we create a user. 


### List of Docker Images used in this section
| Application Name                 | Docker Image Name                          |
| ------------------------------- | --------------------------------------------- |
| User Management Microservice | stacksimplify/kube-usermanagement-microservice:1.0.0 |
| Notifications Microservice V1 | stacksimplify/kube-notifications-microservice:1.0.0 |
| Notifications Microservice V2 | stacksimplify/kube-notifications-microservice:2.0.0 |

## Step-02: Pre-requisite -1: AWS RDS Database, ALB Ingress Controller & External DNS

### AWS RDS Database
- We have created AWS RDS Database as part of section [06-EKS-Storage-with-RDS-Database](/06-EKS-Storage-with-RDS-Database/README.md)
- We even created a `externalName service: 01-MySQL-externalName-Service.yml` in our Kubernetes manifests to point to that RDS Database. 

### ALB Ingress Controller & External DNS
- We are going to deploy a application which will also have a `ALB Ingress Service` and also will register its DNS name in Route53 using `External DNS`
- Which means we should have both related pods running in our EKS cluster. 
- We have installed **ALB Ingress Controller** as part of section [08-01-ALB-Ingress-Install](/08-ELB-Application-LoadBalancers/08-01-ALB-Ingress-Install/README.md)
- We have installed **External DNS** as part of section [08-06-01-Deploy-ExternalDNS-on-EKS](/08-ELB-Application-LoadBalancers/08-06-ALB-Ingress-ExternalDNS/08-06-01-Deploy-ExternalDNS-on-EKS/README.md)
```
# Verify alb-ingress-controller pod running in namespace kube-system
kubectl get pods -n kube-system

# Verify external-dns pod running in default namespace
kubectl get pods
```


## Step-03: Pre-requisite-2: Create Simple Email Service - SES SMTP Credentials
### SMTP Credentials
- Go to Services -> Simple Email Service
- SMTP Settings --> Create My SMTP Credentials
- **IAM User Name:** append the default generated name with microservice or something so we have a reference of this IAM user created for our ECS Microservice deployment
- Download the credentials and update the same for below environment variables which you are going to provide in kubernetes manifest `04-NotificationMicroservice-Deployment.yml`
```
AWS_MAIL_SERVER_HOST=email-smtp.us-east-1.amazonaws.com
AWS_MAIL_SERVER_USERNAME=****
AWS_MAIL_SERVER_PASSWORD=***
AWS_MAIL_SERVER_FROM_ADDRESS= use-a-valid-email@gmail.com 
```
- **Important Note:** Environment variable AWS_MAIL_SERVER_FROM_ADDRESS value should be a **valid** email address and also verified in SES. 

### Verfiy Email Addresses to which notifications we need to send.
- We need two email addresses for testing Notification Service.  
-  **Email Addresses**
    - Verify a New Email Address
    - Email Address Verification Request will be sent to that address, click on link to verify your email. 
    - **From Address:** stacksimplify@gmail.com (replace with your ids during verification)
    - **To Address:** dkalyanreddy@gmail.com (replace with your ids during verification)
- **Important Note:** We need to ensure all the emails (FromAddress email) and (ToAddress emails) to be verified here. 
    - Reference Link: https://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-email-addresses.html    
- Environment Variables
    - AWS_MAIL_SERVER_HOST=email-smtp.us-east-1.amazonaws.com
    - AWS_MAIL_SERVER_USERNAME=*****
    - AWS_MAIL_SERVER_PASSWORD=*****
    - AWS_MAIL_SERVER_FROM_ADDRESS=stacksimplify@gmail.com


## Step-04: Create Notification Microservice Deployment Manifest
- Update environment Variables for Notification Microservice
- **Notification Microservice Deployment**
```yml
          - name: AWS_MAIL_SERVER_HOST
            value: "smtp-service"
          - name: AWS_MAIL_SERVER_USERNAME
            value: "AKIABCDEDFASUBKLDOAX"
          - name: AWS_MAIL_SERVER_PASSWORD
            value: "Bdsdsadsd32qcsads65B4oLo7kMgmKZqhJtEipuE5unLx"
          - name: AWS_MAIL_SERVER_FROM_ADDRESS
            value: "stacksimplify@gmail.com"
```

## Step-05: Create Notification Microservice SMTP ExternalName Service
```yml
apiVersion: v1
kind: Service
metadata:
  name: smtp-service
spec:
  type: ExternalName
  externalName: email-smtp.us-east-1.amazonaws.com
```

## Step-06: Create Notification Microservice NodePort Service
```yml
apiVersion: v1
kind: Service
metadata:
  name: notification-clusterip-service
  labels:
    app: notification-restapp
spec:
  type: ClusterIP
  selector:
    app: notification-restapp
  ports:
  - port: 8096
    targetPort: 8096
```
## Step-07: Update User Management Microservice Deployment Manifest with Notification Service Environment Variables. 
- User Management Service new environment varibales related to Notification Microservice in addition to already which were configured related to MySQL
- Update in `02-UserManagementMicroservice-Deployment.yml`
```yml
          - name: NOTIFICATION_SERVICE_HOST
            value: "notification-clusterip-service"
          - name: NOTIFICATION_SERVICE_PORT
            value: "8096"    
```
## Step-08: Update ALB Ingress Service Kubernetes Manifest
- Update Ingress Service to ensure only target it is going to have is User Management Service
- Remove /app1, /app2 contexts
```yml
    # External DNS - For creating a Record Set in Route53
    external-dns.alpha.kubernetes.io/hostname: services.kubeoncloud.com, ums.kubeoncloud.com
spec:
  rules:
    - http:
        paths:
          - path: /* # SSL Redirect Setting
            backend:
              serviceName: ssl-redirect
              servicePort: use-annotation                   
          - path: /*
            backend:
              serviceName: usermgmt-restapp-nodeport-service
              servicePort: 8095              
```

## Step-09: Deploy Microservices manifests
```
# Deploy Microservices manifests
kubectl apply -f V1-Microservices/
```

## Step-10: Verify the Deployment using kubectl
```
# List Pods
kubectl get pods

# User Management Microservice Logs
kubectl logs -f $(kubectl get po | egrep -o 'usermgmt-microservice-[A-Za-z0-9-]+')

# Notification Microservice Logs
kubectl logs -f $(kubectl get po | egrep -o 'notification-microservice-[A-Za-z0-9-]+')

# External DNS Logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns-[A-Za-z0-9-]+')

# List Ingress
kubectl get ingress
```

## Step-11: Verify Microservices health-status via browser
```
# User Management Service Health-Status
https://services.kubeoncloud.com/usermgmt/health-status

# Notification Microservice Health-Status via User Management
https://services.kubeoncloud.com/usermgmt/notification-health-status
https://services.kubeoncloud.com/usermgmt/notification-service-info
```

## Step-12: Import postman project to Postman client on our desktop. 
- Import postman project
- Add environment url 
    - https://services.kubeoncloud.com (**Replace with your ALB DNS registered url on your environment**)

## Step-13: Test both Microservices using Postman
### User Management Service
- **Create User**
    - Verify the email id to confirm account creation email received.
- **List User**   
    - Verify if newly created user got listed. 
    


## Step-14: Rollout New Deployment - Set Image Option
```
# Rollout New Deployment using Set Image
kubectl set image deployment/notification-microservice notification-service=stacksimplify/kube-notifications-microservice:2.0.0 --record=true

# Verify Rollout Status
kubectl rollout status deployment/notification-microservice

# Verify ReplicaSets
kubectl get rs

# Verify Rollout History
kubectl rollout history deployment/notification-microservice

# Access Application (Should see V2)
https://services.kubeoncloud.com/usermgmt/notification-health-status

# Roll back to Previous Version
kubectl rollout undo deployment/notification-microservice

# Access Application (Should see V1)
https://services.kubeoncloud.com/usermgmt/notification-health-status
```    

## Step-15: Rollout New Deployment - kubectl Edit
```
# Rollout New Deployment using kubectl edit, change image version to 2.0.0
kubectl edit deployment/notification-microservice

# Verify Rollout Status
kubectl rollout status deployment/notification-microservice

# Verify ReplicaSets
kubectl get rs

# Verify Rollout History
kubectl rollout history deployment/notification-microservice

# Access Application (Should see V2)
https://services.kubeoncloud.com/usermgmt/notification-health-status

# Roll back to Previous Version
kubectl rollout undo deployment/notification-microservice

# Access Application (Should see V1)
https://services.kubeoncloud.com/usermgmt/notification-health-status
```

## Step-16: Rollout New Deployment - Update manifest & kubectl apply
```
# Rollout New Deployment by updating yaml manifest 2.0.0
kubectl apply -f kube-manifests/

# Verify Rollout Status
kubectl rollout status deployment/notification-microservice

# Verify ReplicaSets
kubectl get rs

# Verify Rollout History
kubectl rollout history deployment/notification-microservice

# Access Application (Should see V2)
https://services.kubeoncloud.com/usermgmt/notification-health-status

# Roll back to Previous Version
kubectl rollout undo deployment/notification-microservice

# Access Application (Should see V1)
https://services.kubeoncloud.com/usermgmt/notification-health-status
```

## Step-17: Clean-up
```
kubectl delete -f kube-manifests/    
```
