# Two-Tier Application Deployment on Amazon EKS

## Overview

This document provides a comprehensive guide for deploying the two-tier Flask and MySQL application on Amazon Elastic Kubernetes Service (EKS). The deployment was successfully completed on August 18, 2025, demonstrating the migration from Kind (local Kubernetes) to a production-ready EKS cluster.

## Architecture

**EKS Deployment Architecture:**
```
Internet → AWS Load Balancer → EKS Cluster → 
LoadBalancer Service → Flask Pod (5000) → MySQL Service (3306) → MySQL Pod
```

**Key Components:**
- **EKS Cluster**: `Anshuman-cluster` (Kubernetes v1.32)
- **Node Group**: `standard-workers` (2 x t3.small instances)
- **Region**: ap-south-1 (Asia Pacific - Mumbai)
- **Networking**: AWS VPC with public/private subnets
- **Load Balancer**: AWS Application Load Balancer for external access

## Prerequisites Setup

### 1. Tools Installation

The following tools were required and installed:

```bash
# eksctl - EKS cluster management tool
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# kubectl - Kubernetes CLI
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Installed Versions:**
- eksctl: 0.212.0
- kubectl: v1.19.6-eks-49a6c0
- aws-cli: 2.28.11

### 2. AWS Configuration

```bash
# Configure AWS credentials and region
aws configure
# Access Key ID: ****************TBW4
# Secret Access Key: ****************ZaLz
# Default region: ap-south-1
# Default output format: json
```

## EKS Cluster Creation

### 1. Create EKS Cluster

```bash
# Create EKS cluster with managed node group
eksctl create cluster \
  --name Anshuman-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3 \
  --managed
```

**Cluster Creation Details:**
- **Cluster Name**: Anshuman-cluster
- **Creation Time**: 2025-08-18T07:00:25.224Z
- **Duration**: ~10 minutes
- **Kubernetes Version**: 1.32
- **Platform Version**: eks.17

**Node Group Configuration:**
- **Name**: standard-workers
- **Instance Type**: t3.small
- **Capacity Type**: ON_DEMAND
- **Min Size**: 2
- **Max Size**: 3
- **Desired Size**: 2
- **AMI Type**: AL2023_x86_64_STANDARD

### 2. Verify Cluster Creation

```bash
# Verify cluster status
aws eks list-clusters --region ap-south-1
aws eks describe-cluster --name Anshuman-cluster --region ap-south-1

# Verify node group
aws eks list-nodegroups --cluster-name Anshuman-cluster --region ap-south-1
aws eks describe-nodegroup --cluster-name Anshuman-cluster --nodegroup-name standard-workers --region ap-south-1

# Verify kubectl connectivity
kubectl get nodes -o wide
```

## Application Modifications for EKS

### 1. Created EKS-Specific Manifests Directory

```bash
mkdir eks-manifests
```

### 2. Key Modifications Made

#### A. MySQL Secret Management
**New File**: `eks-manifests/mysql-secrets.yml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: YWRtaW4=  # base64 encoded "admin"
```

#### B. MySQL Database Initialization
**New File**: `eks-manifests/mysql-configmap.yml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-initdb-config
data:
  init.sql: |
    CREATE DATABASE IF NOT EXISTS mydb;
    USE mydb;
    CREATE TABLE messages (id INT AUTO_INCREMENT PRIMARY KEY, message TEXT);
```

#### C. MySQL Deployment Changes
**Modified**: `eks-manifests/mysql-deployment.yml`

**Key Changes from Original:**
1. **Removed Persistent Volume**: Eliminated PV/PVC dependency for simplicity
2. **Added Secret Reference**: Used Kubernetes Secret for root password
3. **Added ConfigMap Mount**: Mounted initialization script via ConfigMap
4. **Simplified Storage**: Used ephemeral storage instead of persistent volumes

```yaml
# Original (k8s/mysql-deployment.yml)
- name: MYSQL_ROOT_PASSWORD
  value: "admin"
volumeMounts:
  - name: mysqldata
    mountPath: /var/lib/mysql
volumes:
  - name: mysqldata
    persistentVolumeClaim:
      claimName: mysql-pvc

# Modified (eks-manifests/mysql-deployment.yml)
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secret
      key: MYSQL_ROOT_PASSWORD
volumeMounts:
  - name: mysql-initdb
    mountPath: docker-entrypoint-initdb.d
volumes:
  - name: mysql-initdb
    configMap:
      name: mysql-initdb-config
```

#### D. Service Type Changes
**Modified**: `eks-manifests/two-tier-app-svc.yml`

**Key Changes:**
1. **Service Type**: Changed from NodePort to LoadBalancer
2. **Port Configuration**: Simplified port mapping
3. **External Access**: Leveraged AWS Load Balancer for external connectivity

```yaml
# Original (k8s/two-tier-app-svc.yml)
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 5000
      nodePort: 30004

# Modified (eks-manifests/two-tier-app-svc.yml)
spec:
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

#### E. Application Configuration
**Modified**: `eks-manifests/two-tier-app-deployment.yml`

**Key Changes:**
1. **MySQL Host**: Changed from hardcoded IP to service name
2. **Service Discovery**: Leveraged Kubernetes DNS for service resolution

```yaml
# Original (k8s/two-tier-app-deployment.yml)
- name: MYSQL_HOST
  value: "10.96.216.127"  # Hardcoded ClusterIP

# Modified (eks-manifests/two-tier-app-deployment.yml)
- name: MYSQL_HOST
  value: "mysql"  # Service name for DNS resolution
```

## Deployment Process

### 1. Deploy MySQL Components

```bash
# Deploy MySQL secret
kubectl apply -f eks-manifests/mysql-secrets.yml

# Deploy MySQL ConfigMap
kubectl apply -f eks-manifests/mysql-configmap.yml

# Deploy MySQL deployment
kubectl apply -f eks-manifests/mysql-deployment.yml

# Deploy MySQL service
kubectl apply -f eks-manifests/mysql-svc.yml

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app=mysql --timeout=120s
```

### 2. Deploy Flask Application

```bash
# Deploy Flask application
kubectl apply -f eks-manifests/two-tier-app-deployment.yml

# Deploy Flask service
kubectl apply -f eks-manifests/two-tier-app-svc.yml

# Wait for application to be ready
kubectl wait --for=condition=ready pod -l app=two-tier-app --timeout=120s
```

### 3. Deployment Timeline

- **07:00:25** - EKS cluster creation started
- **07:10:29** - Node group creation started
- **07:41:23** - Node group became active
- **07:43:00** - MySQL deployment started
- **07:43:46** - MySQL pod ready
- **07:44:20** - Flask application deployment started
- **07:44:40** - Flask application pod ready
- **07:45:00** - LoadBalancer provisioned and ready

## Problems Faced and Solutions

### 1. Problem: Persistent Volume Compatibility
**Issue**: Original deployment used hostPath persistent volumes which are not suitable for EKS managed nodes.

**Solution**: 
- Removed PV/PVC dependencies
- Used ConfigMap for database initialization
- Implemented ephemeral storage for development/testing

### 2. Problem: Service Discovery
**Issue**: Hardcoded MySQL ClusterIP in original deployment wouldn't work in EKS.

**Solution**:
- Changed MYSQL_HOST to use service name "mysql"
- Leveraged Kubernetes DNS for automatic service discovery

### 3. Problem: External Access Method
**Issue**: NodePort service type requires specific port configuration and node access.

**Solution**:
- Changed service type to LoadBalancer
- Leveraged AWS Application Load Balancer for external access
- Simplified port configuration

### 4. Problem: Security Best Practices
**Issue**: Passwords stored as plain text in environment variables.

**Solution**:
- Implemented Kubernetes Secrets for sensitive data
- Used base64 encoding for password storage
- Referenced secrets in deployment manifests

## Verification and Testing

### 1. Infrastructure Verification

```bash
# Verify cluster status
kubectl get nodes -o wide
# Output: 2 nodes in Ready state

# Verify all resources
kubectl get all -o wide
# Output: All pods running, services accessible

# Check persistent volumes (none expected)
kubectl get pv,pvc
# Output: No resources found (as expected)
```

### 2. Application Health Checks

```bash
# Check application logs
kubectl logs deployment/two-tier-app-deployment --tail=20
# Output: Flask app running on 0.0.0.0:5000

# Check MySQL logs
kubectl logs deployment/mysql --tail=20
# Output: MySQL ready for connections on port 3306

# Test internal connectivity
kubectl exec deployment/two-tier-app-deployment -- getent hosts mysql
# Output: 10.100.240.1 mysql.default.svc.cluster.local

# Test MySQL connection
kubectl exec deployment/mysql -- mysql -u root -padmin -e "SHOW DATABASES;"
# Output: mydb database present
```

### 3. External Access Testing

```bash
# Get LoadBalancer URL
kubectl get service two-tier-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
# Output: a3c46adff141f469e862fc2762d77433-1854184287.ap-south-1.elb.amazonaws.com

# Test HTTP connectivity
curl -s -o /dev/null -w "%{http_code}" http://a3c46adff141f469e862fc2762d77433-1854184287.ap-south-1.elb.amazonaws.com
# Output: 200 (Success)
```

### 4. Load Balancer Verification

**AWS Load Balancer Details:**
- **Type**: Application Load Balancer (ALB)
- **DNS Name**: a3c46adff141f469e862fc2762d77433-1854184287.ap-south-1.elb.amazonaws.com
- **Port**: 80 → 5000 (Flask application)
- **Health Check**: HTTP GET / (200 OK)

## Final Deployment Status

### Successful Deployment Summary

✅ **EKS Cluster**: Anshuman-cluster - ACTIVE  
✅ **Node Group**: standard-workers - ACTIVE (2/2 nodes)  
✅ **MySQL Pod**: Running (1/1 Ready)  
✅ **Flask App Pod**: Running (1/1 Ready)  
✅ **MySQL Service**: ClusterIP - Internal access working  
✅ **Flask App Service**: LoadBalancer - External access working  
✅ **Database Connection**: Flask ↔ MySQL connectivity verified  
✅ **External Access**: HTTP 200 response via LoadBalancer  

### Resource Summary

**Kubernetes Resources Created:**
- 1 Secret (mysql-secret)
- 1 ConfigMap (mysql-initdb-config)
- 2 Deployments (mysql, two-tier-app-deployment)
- 2 Services (mysql, two-tier-app-service)
- 2 ReplicaSets (auto-created)
- 2 Pods (auto-created)

**AWS Resources Created:**
- 1 EKS Cluster
- 1 EKS Node Group
- 2 EC2 Instances (t3.small)
- 1 Application Load Balancer
- VPC, Subnets, Security Groups (auto-created by eksctl)
- IAM Roles and Policies (auto-created by eksctl)

## Cleanup and Deletion

### 1. Application Cleanup

```bash
# Delete all Kubernetes resources
kubectl delete all --all
# Output: All pods, services, deployments deleted

# Verify cleanup
kubectl get all
# Output: Only kubernetes service remaining
```

### 2. EKS Cluster Deletion

```bash
# Delete entire EKS cluster and associated resources
eksctl delete cluster --name Anshuman-cluster --region ap-south-1
```

**Deletion Process:**
- **Duration**: ~8 minutes
- **Process**: 
  1. Drain unmanaged nodegroups (none found)
  2. Delete Fargate profiles (none found)
  3. Clean up AWS load balancers
  4. Delete node group "standard-workers"
  5. Delete cluster control plane
  6. Clean up CloudFormation stacks

**Deletion Timeline:**
- **07:52:36** - Deletion process started
- **07:52:38** - Node group deletion started
- **08:00:17** - Node group deletion completed
- **08:00:17** - Cluster control plane deletion started
- **08:00:17** - All cluster resources deleted

### 3. Verification of Complete Cleanup

```bash
# Verify cluster deletion
aws eks list-clusters --region ap-south-1
# Output: {"clusters": []} (empty)

# Verify CloudFormation stacks
aws cloudformation list-stacks --region ap-south-1 --stack-status-filter DELETE_COMPLETE
# Output: Both eksctl stacks in DELETE_COMPLETE status

# Verify kubectl context cleanup
kubectl config get-contexts
# Output: No contexts (automatically cleaned up)
```

## Cost Analysis

### Deployment Costs (Actual)
- **EKS Cluster**: $0.10/hour × 1 hour = $0.10
- **EC2 Instances**: 2 × t3.small ($0.0208/hour) × 1 hour = $0.0416
- **Load Balancer**: $0.0225/hour × 1 hour = $0.0225
- **Data Transfer**: Minimal (< $0.01)
- **Total Cost**: ~$0.16 for 1-hour deployment

### Production Considerations
- **Persistent Storage**: Add EBS volumes for MySQL data persistence
- **High Availability**: Multi-AZ deployment with RDS MySQL
- **Monitoring**: CloudWatch Container Insights
- **Security**: Private subnets, WAF, security groups refinement
- **Scaling**: Horizontal Pod Autoscaler (HPA) and Cluster Autoscaler

## Key Differences: Kind vs EKS

| Aspect | Kind (Local) | EKS (Cloud) |
|--------|-------------|-------------|
| **Infrastructure** | Docker containers | AWS managed Kubernetes |
| **Networking** | NodePort (30004) | LoadBalancer (ALB) |
| **Storage** | hostPath PV | EBS volumes (recommended) |
| **Access** | EC2 IP:30004 | Load Balancer DNS |
| **Scalability** | Single node | Multi-node, auto-scaling |
| **Cost** | Free (local) | Pay-per-use AWS resources |
| **Production Ready** | Development only | Production ready |
| **High Availability** | No | Multi-AZ support |
| **Security** | Basic | AWS IAM, VPC, Security Groups |
| **Monitoring** | Manual | CloudWatch integration |

## Lessons Learned

1. **Service Discovery**: Using service names instead of hardcoded IPs provides better portability
2. **Security**: Kubernetes Secrets are essential for production deployments
3. **Storage**: Persistent volumes require careful planning in cloud environments
4. **Networking**: LoadBalancer services simplify external access in cloud environments
5. **Resource Management**: EKS provides better resource isolation and management
6. **Cost Optimization**: Proper resource sizing is crucial for cost management
7. **Automation**: eksctl significantly simplifies EKS cluster management

## Conclusion

The two-tier application was successfully migrated from Kind (local Kubernetes) to Amazon EKS, demonstrating the transition from development to production-ready infrastructure. The deployment showcased:

- **Successful Migration**: All application functionality preserved
- **Cloud-Native Features**: Leveraged AWS Load Balancer and EKS managed services
- **Security Improvements**: Implemented Kubernetes Secrets and proper service discovery
- **Scalability**: Prepared foundation for horizontal scaling and high availability
- **Cost Efficiency**: Demonstrated cost-effective deployment with proper resource sizing

The deployment serves as a solid foundation for production workloads with additional considerations for persistent storage, monitoring, and security hardening.

---

**Deployment Date**: August 18, 2025  
**Duration**: ~1 hour (including cluster creation and testing)  
**Status**: Successfully completed and cleaned up  
**Documentation**: Complete and verified
