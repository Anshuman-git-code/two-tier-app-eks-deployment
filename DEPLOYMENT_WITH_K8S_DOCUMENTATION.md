# Two-Tier Application Deployment with Kubernetes Documentation

## Table of Contents
1. [Project Overview](#project-overview)
2. [Prerequisites](#prerequisites)
3. [Kubernetes Manifest Files Creation](#kubernetes-manifest-files-creation)
4. [Deployment Process](#deployment-process)
5. [Challenges Faced and Solutions](#challenges-faced-and-solutions)
6. [Final Deployment Status](#final-deployment-status)
7. [Testing and Verification](#testing-and-verification)
8. [Lessons Learned](#lessons-learned)

## Project Overview

This documentation covers the complete journey of deploying a two-tier Flask application with MySQL database on Kubernetes using Kind (Kubernetes in Docker). The application consists of:

- **Frontend**: Flask web application (Python)
- **Backend**: MySQL database
- **Container Image**: `anshuman0506/flaskapp:latest`
- **Deployment Platform**: Kind cluster on EC2 instance
- **External Access**: NodePort service on port 30004

**Project Structure:**
```
/home/ubuntu/two-tier-app-deployment/
├── k8s/
│   ├── mysql-pv.yml
│   ├── mysql-pvc.yml
│   ├── mysql-deployment.yml
│   ├── mysql-svc.yml
│   ├── two-tier-app-pod.yml
│   ├── two-tier-app-deployment.yml
│   ├── two-tier-app-svc.yml
│   └── kind-config.yaml
├── app.py
├── Dockerfile
├── requirements.txt
└── templates/
```

## Prerequisites

**Environment Setup:**
- EC2 Instance: Ubuntu Linux
- Docker: Installed and running
- Kind: Kubernetes in Docker
- kubectl: Kubernetes CLI tool
- Public IP: 13.126.0.103

**Initial Setup Commands:**
```bash
# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

## Kubernetes Manifest Files Creation

### 1. MySQL Persistent Volume (mysql-pv.yml)
**Created:** 2025-08-17 09:19:08

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 256Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/ubuntu/two-tier-flask-app/mysqldata
```

### 2. MySQL Persistent Volume Claim (mysql-pvc.yml)
**Created:** 2025-08-17 09:20:43

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
```

### 3. MySQL Deployment (mysql-deployment.yml)
**Created:** 2025-08-17 09:33:32

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:latest
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "admin"
            - name: MYSQL_DATABASE
              value: "mydb"
            - name: MYSQL_USER
              value: "admin"
            - name: MYSQL_PASSWORD
              value: "admin"
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysqldata
              mountPath: /var/lib/mysql     
      volumes:
        - name: mysqldata
          persistentVolumeClaim:
            claimName: mysql-pvc
```

### 4. MySQL Service (mysql-svc.yml)
**Created:** 2025-08-17 09:37:41

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306       # service port
      targetPort: 3306 # container port
  type: ClusterIP      # or NodePort if you want external access
```

### 5. Two-Tier App Pod (two-tier-app-pod.yml)
**Created:** 2025-08-17 09:05:56

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-tier-app-pod
spec:
  containers:
  - name: two-tier-app
    image: anshuman0506/flaskapp:latest
    env:
      - name: MYSQL_HOST
        value: "mysql"
      - name: MYSQL_PASSWORD
        value: "admin"
      - name: MYSQL_USER
        value: "root"
      - name: MYSQL_DB
        value: "mydb"
    ports:
      - containerPort: 5000
    imagePullPolicy: Always
```

### 6. Two-Tier App Deployment (two-tier-app-deployment.yml)
**Created:** 2025-08-17 09:34:32, **Updated:** 2025-08-17 09:45:16

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: two-tier-app-deployment
  labels:
    app: two-tier-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: two-tier-app
  template:
    metadata:
      labels:
        app: two-tier-app
    spec:
      containers:
        - name: two-tier-app
          image: anshuman0506/flaskapp:latest
          env:
            - name: MYSQL_HOST
              value: "10.96.216.127"  # Updated to actual service IP
            - name: MYSQL_PASSWORD
              value: "admin"
            - name: MYSQL_USER
              value: "root"
            - name: MYSQL_DB
              value: "mydb"
          ports:
            - containerPort: 5000
          imagePullPolicy: Always
```

### 7. Two-Tier App Service (two-tier-app-svc.yml)
**Created:** 2025-08-17 09:09:35

```yaml
apiVersion: v1
kind: Service
metadata:
  name: two-tier-app-service
spec:
  selector:
    app: two-tier-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
      nodePort: 30004
  type: NodePort
```

### 8. Kind Cluster Configuration (kind-config.yaml)
**Created:** 2025-08-17 09:41:49

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30004
    hostPort: 30004
    protocol: TCP
```

## Deployment Process

### Phase 1: Initial Cluster Creation
```bash
# Create initial Kind cluster (without port mapping)
kind create cluster --name two-tier-cluster
```

### Phase 2: Deploy MySQL Components
```bash
# Deploy MySQL Persistent Volume
kubectl apply -f mysql-pv.yml

# Deploy MySQL Persistent Volume Claim
kubectl apply -f mysql-pvc.yml

# Deploy MySQL Database
kubectl apply -f mysql-deployment.yml

# Deploy MySQL Service
kubectl apply -f mysql-svc.yml

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app=mysql --timeout=120s
```

### Phase 3: Deploy Application Components
```bash
# Deploy Flask Application
kubectl apply -f two-tier-app-deployment.yml

# Deploy Application Service
kubectl apply -f two-tier-app-svc.yml

# Wait for application to be ready
kubectl wait --for=condition=ready pod -l app=two-tier-app --timeout=120s
```

### Phase 4: Cluster Recreation with Port Mapping
```bash
# Delete existing cluster
kind delete cluster --name two-tier-cluster

# Create new cluster with port mapping
kind create cluster --name two-tier-cluster --config kind-config.yaml

# Redeploy all components
kubectl apply -f mysql-pv.yml
kubectl apply -f mysql-pvc.yml
kubectl apply -f mysql-deployment.yml
kubectl apply -f mysql-svc.yml
kubectl apply -f two-tier-app-deployment.yml
kubectl apply -f two-tier-app-svc.yml
```

## Challenges Faced and Solutions

### Challenge 1: External Access Issue
**Problem:** Application was not accessible via http://13.126.0.103:30004/

**Root Cause:** Kind cluster was running without proper port mapping. The NodePort 30004 was only accessible within the Kind network, not from the EC2 public IP.

**Investigation Steps:**
```bash
# Checked cluster type
kubectl get nodes -o wide
# Result: Identified Kind cluster (two-tier-cluster-control-plane)

# Checked Docker containers
docker ps | grep kind
# Result: No port mapping for 30004

# Checked service configuration
kubectl get services -o wide
# Result: Service was correctly configured as NodePort
```

**Solution:** 
1. Created Kind configuration file with port mapping
2. Recreated the cluster with proper port mapping
3. Redeployed all components

**Commands Used:**
```bash
# Temporary solution - Port forwarding
kubectl port-forward service/two-tier-app-service 8080:80 --address=0.0.0.0

# Permanent solution - Cluster recreation
kind delete cluster --name two-tier-cluster
kind create cluster --name two-tier-cluster --config kind-config.yaml
```

### Challenge 2: MySQL Service IP Configuration
**Problem:** Application couldn't connect to MySQL database after cluster recreation.

**Root Cause:** MySQL service got a new ClusterIP after cluster recreation, but the application deployment still had the old IP hardcoded.

**Investigation Steps:**
```bash
# Get new MySQL service IP
kubectl get service mysql -o jsonpath='{.spec.clusterIP}'
# Result: 10.96.216.127 (different from previous 10.96.167.4)
```

**Solution:** Updated the MYSQL_HOST environment variable in the application deployment.

**Before:**
```yaml
- name: MYSQL_HOST
  value: "10.96.167.4"
```

**After:**
```yaml
- name: MYSQL_HOST
  value: "10.96.216.127"
```

### Challenge 3: DNS Resolution Testing
**Problem:** Needed to verify internal cluster DNS was working properly.

**Investigation:**
```bash
# Test DNS resolution
kubectl exec deployment/two-tier-app-deployment -- getent hosts mysql
# Result: 10.96.216.127   mysql.default.svc.cluster.local

# Test MySQL connectivity
kubectl exec deployment/two-tier-app-deployment -- python3 -c "
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.settimeout(5)
result = sock.connect_ex(('mysql', 3306))
print('MySQL connection: SUCCESS' if result == 0 else 'MySQL connection: FAILED')
"
# Result: MySQL connection: SUCCESS
```

## Final Deployment Status

### Current Cluster State
```bash
kubectl get all -o wide
```

**Pods:**
- `mysql-7f8dbdd66-czn67`: Running (IP: 10.244.0.6)
- `two-tier-app-deployment-7cdd446664-gn4lt`: Running (IP: 10.244.0.7)

**Services:**
- `mysql`: ClusterIP 10.96.216.127:3306
- `two-tier-app-service`: NodePort 10.96.84.235:80 → 30004

**Deployments:**
- `mysql`: 1/1 ready
- `two-tier-app-deployment`: 1/1 ready

### Port Mapping Verification
```bash
docker port two-tier-cluster-control-plane
```
**Result:**
- `6443/tcp -> 127.0.0.1:43695` (Kubernetes API)
- `30004/tcp -> 0.0.0.0:30004` (Application access)

## Testing and Verification

### Application Accessibility Test
```bash
curl -I http://localhost:30004
```
**Result:**
```
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.9.23
Content-Type: text/html; charset=utf-8
Content-Length: 5361
```

### Performance Metrics
```bash
curl -s -o /dev/null -w "HTTP Status: %{http_code}\nResponse Time: %{time_total}s\nContent Length: %{size_download} bytes\n" http://localhost:30004
```
**Result:**
- HTTP Status: 200
- Response Time: 0.004846s
- Content Length: 5361 bytes

### Database Verification
```bash
kubectl exec deployment/mysql -- mysql -u root -padmin -e "SHOW DATABASES;"
```
**Result:**
```
Database
information_schema
mydb
mysql
performance_schema
sys
```

### Application Logs
```bash
kubectl logs deployment/two-tier-app-deployment --tail=5
```
**Result:**
```
* Running on all addresses (0.0.0.0)
* Running on http://127.0.0.1:5000
* Running on http://10.244.0.7:5000
10.244.0.1 - - [17/Aug/2025 09:45:42] "HEAD / HTTP/1.1" 200 -
10.244.0.1 - - [17/Aug/2025 09:46:10] "GET / HTTP/1.1" 200 -
```

## Lessons Learned

### 1. Kind Cluster Networking
- Kind clusters require explicit port mapping configuration for external access
- NodePort services work differently in Kind compared to regular Kubernetes clusters
- Always verify port mappings with `docker port <container-name>`

### 2. Service Discovery
- Use service names for internal communication instead of hardcoded IPs
- ClusterIP addresses change when services are recreated
- DNS resolution works reliably within the cluster

### 3. Deployment Best Practices
- Always wait for pods to be ready before proceeding: `kubectl wait --for=condition=ready`
- Use labels consistently across deployments and services
- Implement proper health checks and readiness probes

### 4. Troubleshooting Approach
- Check cluster type first (Kind vs regular K8s)
- Verify network connectivity at each layer
- Use `kubectl logs` and `kubectl describe` for debugging
- Test internal connectivity before external access

### 5. Configuration Management
- Keep environment-specific configurations separate
- Use ConfigMaps and Secrets for sensitive data in production
- Document all configuration changes with timestamps

## Final Application Access

**Application URL:** http://13.126.0.103:30004/

**Architecture Summary:**
```
Internet → EC2 (13.126.0.103:30004) → Kind Container (30004) → 
NodePort Service (30004) → Flask Pod (5000) → MySQL Service (3306) → MySQL Pod
```

**Deployment Timeline:**
- 09:05 - Initial pod configuration
- 09:09 - Service configuration
- 09:19 - Storage configuration
- 09:33 - MySQL deployment
- 09:37 - Service updates
- 09:41 - Kind configuration with port mapping
- 09:43 - Cluster recreation and final deployment
- 09:45 - Application successfully accessible

**Status:** ✅ **DEPLOYMENT SUCCESSFUL** - All components running and accessible.
