# Deploy UserManagement Service with MySQL Database

## Application Architecture Diagram

```mermaid
graph TB
    USER[External User<br/>Postman/Browser] -->|NodePort: 31231| WN[Worker Node]
    
    WN -->|Forward to Service| NPSVC[NodePort Service<br/>usermgmt-nodeport-service]
    
    NPSVC -->|Load Balance| POD1[User Management Pod 1]
    NPSVC -->|Load Balance| POD2[User Management Pod 2]
    
    subgraph "User Management Pods"
        POD1 -->|Read Env Vars| ENV1[Environment Variables]
        POD2 -->|Read Env Vars| ENV2[Environment Variables]
        
        ENV1 --> DB_HOST1[DB_HOSTNAME: mysql]
        ENV1 --> DB_PORT1[DB_PORT: 3306]
        ENV1 --> DB_NAME1[DB_NAME: usermgmt]
        ENV1 --> DB_USER1[DB_USERNAME: root]
        ENV1 --> DB_PASS1[DB_PASSWORD: dbpassword11]
        
        ENV2 --> DB_HOST2[DB_HOSTNAME: mysql]
    end
    
    POD1 -->|Connect via Service| MYSQL_SVC[ClusterIP Service<br/>mysql:3306]
    POD2 -->|Connect via Service| MYSQL_SVC
    
    MYSQL_SVC -->|Route to Pod| MYSQL[MySQL Pod<br/>with EBS Volume]
    
    MYSQL -->|Persistent Storage| EBS[EBS Volume<br/>4Gi gp3]
    
    MYSQL -->|Schema| DB[usermgmt Database<br/>users table]
    
    subgraph "API Endpoints"
        API1[POST /usermgmt/user<br/>Create User]
        API2[GET /usermgmt/users<br/>List Users]
        API3[PUT /usermgmt/user<br/>Update User]
        API4[DELETE /usermgmt/user/:id<br/>Delete User]
        API5[GET /usermgmt/health-status<br/>Health Check]
    end
    
    USER -.->|Test APIs| API1
    USER -.->|Test APIs| API2
    USER -.->|Test APIs| API3
    USER -.->|Test APIs| API4
    USER -.->|Test APIs| API5
    
    POD1 -.->|Process Requests| API1
    
    subgraph "Database Operations"
        CRUD[CRUD Operations]
        CRUD --> CREATE[INSERT INTO users]
        CRUD --> READ[SELECT FROM users]
        CRUD --> UPDATE[UPDATE users]
        CRUD --> DELETE[DELETE FROM users]
    end
    
    DB --> CRUD
    
    style POD1 fill:#4A90E2
    style POD2 fill:#4A90E2
    style MYSQL fill:#FF6B6B
    style EBS fill:#FF9900
    style NPSVC fill:#9B59B6
    style USER fill:#2E8B57
```

### Diagram Explanation

- **User Management Microservice**: **Spring Boot** application providing RESTful APIs for **user CRUD operations** with MySQL backend
- **NodePort Service**: Exposes application on **port 31231** on all worker nodes, allowing **external access** without load balancer
- **Environment Variables**: Pod configuration includes **DB connection parameters** (hostname, port, database name, credentials) injected at runtime
- **Service Discovery**: Application connects to MySQL using **service name** (mysql) which resolves to **ClusterIP**, no hardcoded IPs
- **Connection Pooling**: Spring Boot manages **database connection pool** for efficient **concurrent request handling** and **resource management**
- **Pod Replication**: Multiple pods provide **high availability** and **horizontal scaling**, service distributes traffic across all healthy pods
- **RESTful API Design**: Follows **REST principles** with proper **HTTP methods** (GET, POST, PUT, DELETE) and **JSON payloads**
- **Health Status Endpoint**: Provides **liveness** and **readiness** checks for Kubernetes health monitoring and **automatic pod recovery**
- **Data Persistence**: All user data stored in **MySQL database** backed by **EBS volume**, survives pod restarts and redeployments
- **Testing with Postman**: Import collection to test all APIs with **environment variables** for easy endpoint management and **request templates**

## Step-01: Introduction
- We are going to deploy a **User Management Microservice** which will connect to MySQL Database schema **usermgmt** during startup.
- Then we can test the following APIs
  - Create Users
  - List Users
  - Delete User
  - Health Status 

| Kubernetes Object  | YAML File |
| ------------- | ------------- |
| Deployment, Environment Variables  | 06-UserManagementMicroservice-Deployment.yml  |
| NodePort Service  | 07-UserManagement-Service.yml  |

## Step-02: Create following Kubernetes manifests

### Create User Management Microservice Deployment manifest
- **Environment Variables**

| Key Name  | Value |
| ------------- | ------------- |
| DB_HOSTNAME  | mysql |
| DB_PORT  | 3306  |
| DB_NAME  | usermgmt  |
| DB_USERNAME  | root  |
| DB_PASSWORD | dbpassword11  |  

### Create User Management Microservice NodePort Service manifest
- NodePort Service

## Step-03: Create UserManagement Service Deployment & Service 
```
# Create Deployment & NodePort Service
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Verify logs of Usermgmt Microservice pod
kubectl logs -f <Pod-Name>

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```
- **Problem Observation:** 
  - If we deploy all manifests at a time, by the time mysql is ready our `User Management Microservice` pod will be restarting multiple times due to unavailability of Database. 
  - To avoid such situations, we can apply `initContainers` concept to our User management Microservice `Deployment manifest`.
  - We will see that in our next section but for now lets continue to test the application
- **Access Application**
```
# List Services
kubectl get svc

# Get Public IP
kubectl get nodes -o wide

# Access Health Status API for User Management Service
http://<EKS-WorkerNode-Public-IP>:31231/usermgmt/health-status
```

## Step-04: Test User Management Microservice using Postman
### Download Postman client 
- https://www.postman.com/downloads/ 
### Import Project to Postman
- Import the postman project `AWS-EKS-Masterclass-Microservices.postman_collection.json` present in folder `04-03-UserManagement-MicroService-with-MySQLDB`
### Create Environment in postman
- Go to Settings -> Click on Add
- **Environment Name:** UMS-NodePort
  - **Variable:** url
  - **Initial Value:** http://WorkerNode-Public-IP:31231
  - **Current Value:** http://WorkerNode-Public-IP:31231
  - Click on **Add**
### Test User Management Services
- Select the environment before calling any API
- **Health Status API**
  - URL: `{{url}}/usermgmt/health-status`
- **Create User Service**
  - URL: `{{url}}/usermgmt/user`
  - `url` variable will replaced from environment we selected
```json
    {
        "username": "admin1",
        "email": "dkalyanreddy@gmail.com",
        "role": "ROLE_ADMIN",
        "enabled": true,
        "firstname": "fname1",
        "lastname": "lname1",
        "password": "Pass@123"
    }
```
- **List User Service**
  - URL: `{{url}}/usermgmt/users`

- **Update User Service**
  - URL: `{{url}}/usermgmt/user`
```json
    {
        "username": "admin1",
        "email": "dkalyanreddy@gmail.com",
        "role": "ROLE_ADMIN",
        "enabled": true,
        "firstname": "fname2",
        "lastname": "lname2",
        "password": "Pass@123"
    }
```  
- **Delete User Service**
  - URL: `{{url}}/usermgmt/user/admin1`

## Step-05: Verify Users in MySQL Database
```
# Connect to MYSQL Database
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -u root -pdbpassword11

# Verify usermgmt schema got created which we provided in ConfigMap
mysql> show schemas;
mysql> use usermgmt;
mysql> show tables;
mysql> select * from users;
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


