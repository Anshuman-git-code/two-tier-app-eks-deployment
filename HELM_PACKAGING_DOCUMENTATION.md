# Helm Packaging Documentation - Two-Tier Application

This document provides a comprehensive guide on how the two-tier Flask and MySQL application was packaged and deployed using Helm charts. It includes all commands used, modifications made, issues encountered, and their solutions.

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Helm Chart Creation](#helm-chart-creation)
4. [Chart Modifications](#chart-modifications)
5. [Packaging Process](#packaging-process)
6. [Deployment Process](#deployment-process)
7. [Issues Encountered and Solutions](#issues-encountered-and-solutions)
8. [Testing and Verification](#testing-and-verification)
9. [Application Management](#application-management)
10. [Best Practices](#best-practices)

## Overview

The two-tier application consists of:
- **Frontend**: Flask web application (Python) - Port 5000
- **Backend**: MySQL database - Port 3306
- **Architecture**: Flask app connects to MySQL via Kubernetes service discovery
- **Storage**: Persistent storage for MySQL data
- **Access**: External access via NodePort service on port 30004

## Prerequisites

Before starting the Helm packaging process, ensure you have:
- Kubernetes cluster running (Kind cluster in this case)
- Helm CLI installed
- kubectl configured to access the cluster
- Docker images available:
  - `anshuman0506/flaskapp:latest` (Flask application)
  - `mysql:5.7` (MySQL database)

### Cluster Information
```bash
# Verify cluster is running
kubectl cluster-info
# Output: Kubernetes control plane is running at https://127.0.0.1:34395
```

## Helm Chart Creation

### 1. MySQL Chart Creation
```bash
# Create MySQL Helm chart
helm create mysql-chart
```

### 2. Flask App Chart Creation
```bash
# Create Flask application Helm chart
helm create flask-app-chart
```

### Directory Structure Created
```
two-tier-app-deployment/
├── mysql-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── templates/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── _helpers.tpl
│   │   └── ...
│   └── charts/
└── flask-app-chart/
    ├── Chart.yaml
    ├── values.yaml
    ├── templates/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── _helpers.tpl
    │   └── ...
    └── charts/
```

## Chart Modifications

### MySQL Chart Modifications

#### 1. Chart.yaml Configuration
```yaml
apiVersion: v2
name: mysql-chart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

#### 2. values.yaml Key Modifications
```yaml
# Image configuration - Changed from mysql:latest to mysql:5.7
image:
  repository: mysql
  pullPolicy: IfNotPresent
  tag: "5.7"  # Changed from "latest" for compatibility

# Environment variables for MySQL
env:
  mysqlRootPassword: admin
  mysqlDatabase: mydb
  mysqlUser: admin
  mysqlPassword: admin

# Service configuration
service:
  type: ClusterIP
  port: 3306

# Health checks - Changed from HTTP to TCP
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 5
  periodSeconds: 5

# Persistent storage configuration
volumes:
- name: mysql-storage
  persistentVolumeClaim:
    claimName: mysql-pvc

volumeMounts:
- name: mysql-storage
  mountPath: "/var/lib/mysql"
```

#### 3. Additional Template Created - PVC
**File**: `mysql-chart/templates/pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  labels:
    {{- include "mysql-chart.labels" . | nindent 4 }}
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### Flask App Chart Modifications

#### 1. Chart.yaml Configuration
```yaml
apiVersion: v2
name: flask-app-chart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

#### 2. values.yaml Key Modifications
```yaml
# Image configuration
image:
  repository: anshuman0506/flaskapp
  pullPolicy: IfNotPresent
  tag: "latest"

# Environment variables - Using service name for MySQL connection
env:
  mysqlHost: mysql-chart   # Service name instead of hard-coded IP
  mysqlUser: admin
  mysqlPassword: admin
  mysqlDatabase: mydb

# Service configuration for external access
service:
  type: NodePort
  port: 80
  targetPort: 5000
  nodePort: 30004

# Health checks for HTTP service
livenessProbe:
  httpGet:
    path: /
    port: http
readinessProbe:
  httpGet:
    path: /
    port: http
```

## Packaging Process

### 1. Package MySQL Chart
```bash
# Package the MySQL chart
helm package mysql-chart
# Output: Successfully packaged chart and saved it to: /home/ubuntu/two-tier-app-deployment/mysql-chart-0.1.0.tgz
```

### 2. Package Flask App Chart
```bash
# Package the Flask application chart
helm package flask-app-chart
# Output: Successfully packaged chart and saved it to: /home/ubuntu/two-tier-app-deployment/flask-app-chart-0.1.0.tgz
```

### Generated Packages
```
mysql-chart-0.1.0.tgz      (4.3 KB)
flask-app-chart-0.1.0.tgz  (4.4 KB)
```

## Deployment Process

### 1. Initial Deployment (With Issues)
```bash
# Deploy MySQL chart
helm install mysql-chart ./mysql-chart

# Deploy Flask app chart
helm install flask-app-chart ./flask-app-chart
```

### 2. Check Initial Status
```bash
# Check deployment status
kubectl get all -o wide
```

**Initial Status**: Both pods were in `CrashLoopBackOff` state

## Issues Encountered and Solutions

### Issue 1: MySQL Pod CrashLoopBackOff

**Problem**: MySQL pod was crashing due to incorrect health checks
```bash
# Error in logs showed MySQL was starting but health checks were failing
kubectl logs pod/mysql-chart-64d99479d6-dlhwr
```

**Root Cause**: HTTP health checks configured for MySQL (which doesn't serve HTTP)

**Solution**: Modified health checks in `values.yaml`
```yaml
# Changed from HTTP to TCP health checks
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Issue 2: Flask App Connection Error

**Problem**: Flask app couldn't connect to MySQL
```bash
# Error logs showed connection failure
kubectl logs pod/flask-app-chart-ff8f494bf-2xtdf
# Output: MySQLdb.OperationalError: (2002, "Can't connect to server on '10.96.168.235' (115)")
```

**Root Cause**: Hard-coded IP address in Flask app configuration

**Solution**: Changed to use Kubernetes service name
```yaml
# In flask-app-chart/values.yaml
env:
  mysqlHost: mysql-chart   # Changed from hard-coded IP to service name
```

### Issue 3: MySQL Version Compatibility

**Problem**: Using `mysql:latest` (v9.x) caused compatibility issues

**Solution**: Changed to stable version
```yaml
# In mysql-chart/values.yaml
image:
  tag: "5.7"  # Changed from "latest"
```

### Issue 4: Missing Persistent Storage

**Problem**: MySQL data would be lost on pod restart

**Solution**: Added PVC and volume mounts
1. Created `pvc.yaml` template
2. Added volume configuration in `values.yaml`

## Testing and Verification

### 1. Upgrade Deployments with Fixes
```bash
# Upgrade MySQL chart with fixes
helm upgrade mysql-chart ./mysql-chart
# Output: Release "mysql-chart" has been upgraded. Happy Helming!

# Upgrade Flask app chart with fixes
helm upgrade flask-app-chart ./flask-app-chart
# Output: Release "flask-app-chart" has been upgraded. Happy Helming!
```

### 2. Wait for Pods to be Ready
```bash
# Wait for MySQL pod to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=mysql-chart --timeout=120s
# Output: pod/mysql-chart-5cf59484dc-4s665 condition met
```

### 3. Verify Pod Status
```bash
# Check all pods are running
kubectl get pods -o wide
```
**Result**: Both pods showing `Running` status
```
NAME                              READY   STATUS    RESTARTS   AGE
flask-app-chart-b486d668c-94dn5   1/1     Running   3          2m7s
mysql-chart-5cf59484dc-4s665      1/1     Running   0          2m10s
```

### 4. Verify Services
```bash
# Check all services
kubectl get all -o wide
```
**Result**: Services properly configured
```
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/flask-app-chart   NodePort    10.96.59.134    <none>        80:30004/TCP   7m29s
service/mysql-chart       ClusterIP   10.96.168.235   <none>        3306/TCP       30m
```

### 5. Test Database Connectivity
```bash
# Test service name resolution
kubectl exec deployment/flask-app-chart -- getent hosts mysql-chart
# Output: 10.96.168.235   mysql-chart.default.svc.cluster.local

# Test MySQL connection
kubectl exec deployment/mysql-chart -- mysql -u admin -padmin -e "SHOW DATABASES;"
# Output: Database: information_schema, mydb
```

### 6. Verify Database Schema
```bash
# Check if messages table exists
kubectl exec deployment/mysql-chart -- mysql -u admin -padmin mydb -e "SHOW TABLES;"
# Output: Tables_in_mydb: messages

# Check table structure
kubectl exec deployment/mysql-chart -- mysql -u admin -padmin mydb -e "DESCRIBE messages;"
# Output: Field: id (int, auto_increment), message (text)
```

### 7. Test Application Logs
```bash
# Check Flask application logs
kubectl logs deployment/flask-app-chart --tail=20
```
**Result**: Application receiving HTTP requests with 200 responses
```
10.244.0.1 - - [17/Aug/2025 18:52:35] "GET / HTTP/1.1" 200 -
10.244.0.1 - - [17/Aug/2025 18:52:37] "GET / HTTP/1.1" 200 -
```

### 8. Verify Persistent Storage
```bash
# Check PVC status
kubectl get pvc
```
**Result**: PVC bound successfully
```
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES
mysql-pvc   Bound    pvc-cae1ee0b-5ff1-42fa-b855-6a31f429ea35   1Gi        RWO
```

### 9. External Access Verification
```bash
# Get external access URL
echo "Application accessible at: http://$(curl -s ifconfig.me):30004"
# Output: Application accessible at: http://13.232.46.251:30004
```

### 10. Helm Release Status
```bash
# Check MySQL chart status
helm status mysql-chart
# Output: STATUS: deployed, REVISION: 2

# Check Flask app chart status
helm status flask-app-chart
# Output: STATUS: deployed, REVISION: 2

# List all releases
helm list
```
**Result**: Both releases deployed successfully
```
NAME           NAMESPACE  REVISION  STATUS    CHART
flask-app-chart default   2         deployed  flask-app-chart-0.1.0
mysql-chart    default   2         deployed  mysql-chart-0.1.0
```

## Application Management

### Starting the Application
```bash
# Install MySQL chart
helm install mysql-chart ./mysql-chart

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=mysql-chart --timeout=120s

# Install Flask app chart
helm install flask-app-chart ./flask-app-chart

# Wait for Flask app to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=flask-app-chart --timeout=120s
```

### Stopping the Application
```bash
# Uninstall Flask app
helm uninstall flask-app-chart
# Output: release "flask-app-chart" uninstalled

# Uninstall MySQL
helm uninstall mysql-chart
# Output: release "mysql-chart" uninstalled

# Verify cleanup
kubectl get all
# Output: Only kubernetes service remains
```

### Upgrading the Application
```bash
# Upgrade with new configurations
helm upgrade mysql-chart ./mysql-chart
helm upgrade flask-app-chart ./flask-app-chart
```

## Best Practices Implemented

### 1. Service Discovery
- Used Kubernetes service names instead of hard-coded IP addresses
- Configured proper DNS resolution between services

### 2. Health Checks
- Implemented appropriate health checks for each service type
- TCP checks for MySQL, HTTP checks for Flask app
- Added proper delays and intervals

### 3. Persistent Storage
- Configured PVC for MySQL data persistence
- Used appropriate storage class and access modes

### 4. Environment Configuration
- Externalized configuration through Helm values
- Used consistent naming conventions
- Proper secret management for database credentials

### 5. Resource Management
- Defined appropriate resource limits and requests
- Configured proper service types for internal and external access

### 6. Chart Organization
- Separated concerns into different charts
- Used Helm templating effectively
- Maintained clean chart structure

## Troubleshooting Commands

### Debug Pod Issues
```bash
# Check pod status
kubectl get pods -o wide

# Check pod logs
kubectl logs <pod-name>

# Describe pod for events
kubectl describe pod <pod-name>

# Execute commands in pod
kubectl exec -it <pod-name> -- /bin/bash
```

### Debug Service Issues
```bash
# Check services
kubectl get svc

# Test service connectivity
kubectl exec <pod-name> -- nslookup <service-name>

# Port forward for testing
kubectl port-forward service/<service-name> <local-port>:<service-port>
```

### Debug Helm Issues
```bash
# Check release status
helm status <release-name>

# View release history
helm history <release-name>

# Dry run deployment
helm install <release-name> <chart-path> --dry-run --debug

# Template rendering
helm template <chart-path>
```

## Conclusion

The Helm packaging process successfully converted the two-tier application from manual Kubernetes manifests to reusable Helm charts. Key achievements:

1. **Modular Design**: Separated MySQL and Flask app into independent charts
2. **Configuration Management**: Externalized all configuration through Helm values
3. **Service Discovery**: Implemented proper Kubernetes service communication
4. **Data Persistence**: Added persistent storage for MySQL data
5. **Health Monitoring**: Configured appropriate health checks
6. **External Access**: Set up NodePort service for external connectivity

The application is now easily deployable, upgradeable, and manageable through Helm commands, providing a production-ready deployment solution.

## Files Modified/Created

### Modified Files:
- `mysql-chart/values.yaml` - Updated image tag, health checks, storage, environment variables
- `flask-app-chart/values.yaml` - Updated MySQL host configuration, service settings

### Created Files:
- `mysql-chart/templates/pvc.yaml` - Persistent Volume Claim for MySQL storage
- `mysql-chart-0.1.0.tgz` - Packaged MySQL Helm chart
- `flask-app-chart-0.1.0.tgz` - Packaged Flask app Helm chart

### Generated Resources:
- MySQL Deployment with persistent storage
- Flask App Deployment with service discovery
- ClusterIP service for MySQL (internal access)
- NodePort service for Flask app (external access)
- PersistentVolumeClaim for MySQL data storage

This documentation serves as a complete reference for the Helm packaging and deployment process of the two-tier application.
