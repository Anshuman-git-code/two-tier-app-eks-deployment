# 🚀 Two-Tier Flask Application - Complete DevOps Deployment Guide

A comprehensive DevOps project demonstrating the deployment of a two-tier Flask and MySQL application across multiple platforms: Docker, Kubernetes (Kind), Amazon EKS, and Helm packaging.

## 📋 Project Overview

This project showcases a complete two-tier web application with multiple deployment strategies:

- **Frontend/Backend**: Flask web application (Python) with MySQL database
- **Containerization**: Docker and Docker Compose
- **Local Kubernetes**: Kind (Kubernetes in Docker)
- **Cloud Kubernetes**: Amazon EKS
- **Package Management**: Helm Charts
- **Infrastructure**: AWS EC2, EKS, Load Balancers

## 🏗️ Architecture

### Application Architecture
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Flask App     │───▶│   MySQL DB      │    │  Persistent     │
│   (Port 5000)   │    │   (Port 3306)   │───▶│  Storage        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Deployment Architectures

#### Docker Deployment
```
Internet → EC2:5000 → Docker Network → Flask Container ↔ MySQL Container
```

#### Kubernetes (Kind) Deployment
```
Internet → EC2:30004 → Kind Container → NodePort Service → Flask Pod ↔ MySQL Pod
```

#### EKS Deployment
```
Internet → AWS Load Balancer → EKS Cluster → LoadBalancer Service → Flask Pod ↔ MySQL Pod
```

## 🛠️ Technologies Used

- **Application**: Python Flask, MySQL 5.7
- **Containerization**: Docker, Docker Compose
- **Orchestration**: Kubernetes, Kind, Amazon EKS
- **Package Management**: Helm Charts
- **Cloud Platform**: AWS (EC2, EKS, ALB)
- **Infrastructure**: Terraform (for EKS setup)
- **Networking**: Kubernetes Services, AWS Load Balancer

## 📁 Project Structure

```
two-tier-app-eks-deployment/
├── app.py                                  # Flask application
├── requirements.txt                        # Python dependencies
├── Dockerfile                             # Container build instructions
├── docker-compose.yml                     # Multi-container orchestration
├── message.sql                            # Database initialization
├── templates/
│   └── index.html                         # Frontend template
├── k8s/                                   # Kubernetes manifests (Kind)
│   ├── mysql-pv.yml                      # MySQL Persistent Volume
│   ├── mysql-pvc.yml                     # MySQL Persistent Volume Claim
│   ├── mysql-deployment.yml              # MySQL Deployment
│   ├── mysql-svc.yml                     # MySQL Service
│   ├── two-tier-app-deployment.yml       # Flask App Deployment
│   ├── two-tier-app-svc.yml             # Flask App Service (NodePort)
│   ├── two-tier-app-pod.yml             # Flask App Pod
│   └── kind-config.yaml                  # Kind cluster configuration
├── eks-manifests/                         # EKS-specific manifests
│   ├── mysql-secrets.yml                 # MySQL credentials (base64)
│   ├── mysql-configmap.yml               # Database initialization
│   ├── mysql-deployment.yml              # MySQL Deployment (EKS)
│   ├── mysql-svc.yml                     # MySQL Service (EKS)
│   ├── two-tier-app-deployment.yml       # Flask App Deployment (EKS)
│   └── two-tier-app-svc.yml             # Flask App Service (LoadBalancer)
├── flask-app-chart/                       # Helm chart for Flask app
├── mysql-chart/                           # Helm chart for MySQL
├── flask-app-chart-0.1.0.tgz             # Packaged Flask Helm chart
├── mysql-chart-0.1.0.tgz                 # Packaged MySQL Helm chart
├── DEPLOYMENT_WITH_DOCKER_DOCUMENTATION.md
├── DEPLOYMENT_WITH_K8S_DOCUMENTATION.md
├── DEPLOYMENT_WITH_EKS_DOCUMENTATION.md
├── HELM_PACKAGING_DOCUMENTATION.md
└── README.md                              # This file
```

## 🚀 Deployment Methods

### Method 1: Docker Deployment

#### Quick Start with Docker Compose
```bash
# Clone repository
git clone https://github.com/Anshuman-git-code/two-tier-app-eks-deployment.git
cd two-tier-app-eks-deployment

# Start application
docker-compose up --build

# Access application
curl http://localhost:5000
```

#### Manual Docker Deployment
```bash
# Create custom network
docker network create twotier

# Deploy MySQL
docker run -d \
    --name mysql \
    -v mysql-data:/var/lib/mysql \
    --network=twotier \
    -e MYSQL_DATABASE=mydb \
    -e MYSQL_ROOT_PASSWORD=admin \
    -p 3306:3306 \
    mysql:5.7

# Deploy Flask app
docker run -d \
    --name flaskapp \
    --network=twotier \
    -e MYSQL_HOST=mysql \
    -e MYSQL_USER=root \
    -e MYSQL_PASSWORD=admin \
    -e MYSQL_DB=mydb \
    -p 5000:5000 \
    anshuman0506/flaskapp:latest
```

### Method 2: Kubernetes (Kind) Deployment

#### Prerequisites
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

#### Deployment Steps
```bash
# Create Kind cluster with port mapping
kind create cluster --name two-tier-cluster --config k8s/kind-config.yaml

# Deploy MySQL components
kubectl apply -f k8s/mysql-pv.yml
kubectl apply -f k8s/mysql-pvc.yml
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/mysql-svc.yml

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app=mysql --timeout=120s

# Deploy Flask application
kubectl apply -f k8s/two-tier-app-deployment.yml
kubectl apply -f k8s/two-tier-app-svc.yml

# Wait for application to be ready
kubectl wait --for=condition=ready pod -l app=two-tier-app --timeout=120s

# Access application
echo "Application accessible at: http://$(curl -s ifconfig.me):30004"
```

### Method 3: Amazon EKS Deployment

#### Prerequisites
```bash
# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS credentials
aws configure
```

#### EKS Cluster Creation
```bash
# Create EKS cluster
eksctl create cluster \
  --name Anshuman-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 3 \
  --managed

# Verify cluster
kubectl get nodes -o wide
```

#### Application Deployment
```bash
# Deploy MySQL components
kubectl apply -f eks-manifests/mysql-secrets.yml
kubectl apply -f eks-manifests/mysql-configmap.yml
kubectl apply -f eks-manifests/mysql-deployment.yml
kubectl apply -f eks-manifests/mysql-svc.yml

# Wait for MySQL
kubectl wait --for=condition=ready pod -l app=mysql --timeout=120s

# Deploy Flask application
kubectl apply -f eks-manifests/two-tier-app-deployment.yml
kubectl apply -f eks-manifests/two-tier-app-svc.yml

# Get LoadBalancer URL
kubectl get service two-tier-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Method 4: Helm Deployment

#### Create and Package Charts
```bash
# Create Helm charts
helm create mysql-chart
helm create flask-app-chart

# Package charts
helm package mysql-chart
helm package flask-app-chart
```

#### Deploy with Helm
```bash
# Install MySQL chart
helm install mysql-chart ./mysql-chart

# Wait for MySQL to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=mysql-chart --timeout=120s

# Install Flask app chart
helm install flask-app-chart ./flask-app-chart

# Check deployment status
helm list
```

## 🔧 Key Features Implemented

### ✅ Multi-Platform Deployment
- Docker and Docker Compose
- Kubernetes with Kind (local)
- Amazon EKS (cloud)
- Helm package management

### ✅ Infrastructure as Code
- Kubernetes manifests
- Helm charts with templating
- EKS cluster automation with eksctl

### ✅ Service Discovery
- Docker network communication
- Kubernetes DNS resolution
- Service-to-service connectivity

### ✅ Data Persistence
- Docker volumes
- Kubernetes Persistent Volumes
- EBS volumes in EKS

### ✅ Security Best Practices
- Kubernetes Secrets for credentials
- Network isolation
- Service account management

### ✅ Scalability
- Horizontal pod scaling
- Load balancer integration
- Multi-node cluster support

## 🔍 Challenges Faced and Solutions

### Challenge 1: Container Networking
**Problem**: Flask app couldn't connect to MySQL in Docker
**Solution**: Created custom Docker network and used service names

### Challenge 2: Kubernetes External Access
**Problem**: NodePort not accessible from external IP in Kind
**Solution**: Configured Kind cluster with proper port mapping

### Challenge 3: EKS Service Discovery
**Problem**: Hardcoded IPs breaking after cluster recreation
**Solution**: Used Kubernetes service names for DNS resolution

### Challenge 4: MySQL Health Checks
**Problem**: HTTP health checks failing for MySQL in Helm
**Solution**: Changed to TCP health checks on port 3306

### Challenge 5: Persistent Storage in EKS
**Problem**: hostPath volumes not suitable for managed nodes
**Solution**: Used ConfigMaps for initialization and ephemeral storage

## 📊 Deployment Comparison

| Feature | Docker | Kind | EKS | Helm |
|---------|--------|------|-----|------|
| **Setup Complexity** | Low | Medium | High | Medium |
| **Production Ready** | No | No | Yes | Yes |
| **External Access** | Direct | NodePort | LoadBalancer | Configurable |
| **Scalability** | Manual | Manual | Auto | Auto |
| **Cost** | Free | Free | Pay-per-use | Depends on platform |
| **High Availability** | No | No | Yes | Yes |
| **Package Management** | No | No | No | Yes |

## 🧪 Testing and Verification

### Application Health Checks
```bash
# Check all pods
kubectl get pods -o wide

# Test application endpoint
curl -I http://<endpoint>

# Check database connectivity
kubectl exec deployment/mysql -- mysql -u root -padmin -e "SHOW DATABASES;"

# View application logs
kubectl logs deployment/two-tier-app-deployment --tail=20
```

### Performance Testing
```bash
# Response time test
curl -s -o /dev/null -w "Response Time: %{time_total}s\n" http://<endpoint>

# Load testing with multiple requests
for i in {1..10}; do curl -s http://<endpoint> > /dev/null & done
```

## 🔄 CI/CD Integration

The project is ready for CI/CD integration with:
- Automated Docker image builds
- Kubernetes manifest validation
- Helm chart testing
- Multi-environment deployments

## 📈 Monitoring and Observability

### Kubernetes Monitoring
```bash
# Resource usage
kubectl top pods
kubectl top nodes

# Events monitoring
kubectl get events --sort-by=.metadata.creationTimestamp

# Application metrics
kubectl port-forward service/two-tier-app-service 8080:80
```

### Application Logs
```bash
# Real-time logs
kubectl logs -f deployment/two-tier-app-deployment

# MySQL logs
kubectl logs -f deployment/mysql
```

## 🧹 Cleanup Commands

### Docker Cleanup
```bash
docker-compose down -v
docker system prune -a
```

### Kubernetes Cleanup
```bash
kubectl delete -f k8s/
kind delete cluster --name two-tier-cluster
```

### EKS Cleanup
```bash
kubectl delete all --all
eksctl delete cluster --name Anshuman-cluster --region ap-south-1
```

### Helm Cleanup
```bash
helm uninstall flask-app-chart
helm uninstall mysql-chart
```

## 💰 Cost Analysis

### EKS Deployment Costs (1 hour)
- **EKS Cluster**: $0.10/hour
- **EC2 Instances**: 2 × t3.small ($0.0208/hour) = $0.0416
- **Load Balancer**: $0.0225/hour
- **Total**: ~$0.16/hour

### Cost Optimization Tips
- Use spot instances for development
- Right-size instance types
- Implement cluster autoscaling
- Monitor resource utilization

## 🔐 Security Considerations

### Implemented Security Measures
- Kubernetes Secrets for sensitive data
- Network policies and service isolation
- Non-root container users
- Resource limits and quotas

### Production Recommendations
- Enable RBAC
- Use private subnets
- Implement Pod Security Standards
- Regular security updates

## 📚 Documentation

Detailed documentation for each deployment method:
- [Docker Deployment Guide](DEPLOYMENT_WITH_DOCKER_DOCUMENTATION.md)
- [Kubernetes Deployment Guide](DEPLOYMENT_WITH_K8S_DOCUMENTATION.md)
- [EKS Deployment Guide](DEPLOYMENT_WITH_EKS_DOCUMENTATION.md)
- [Helm Packaging Guide](HELM_PACKAGING_DOCUMENTATION.md)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🎯 Key Learnings

1. **Container Orchestration**: Understanding the progression from Docker to Kubernetes to managed services
2. **Service Discovery**: Importance of DNS-based service communication
3. **Storage Management**: Different storage solutions for different platforms
4. **Network Configuration**: Port mapping, service types, and load balancers
5. **Package Management**: Benefits of Helm for Kubernetes application management
6. **Cloud Migration**: Considerations when moving from local to cloud environments

## 🚀 Future Enhancements

- [ ] Implement CI/CD pipeline with GitHub Actions
- [ ] Add monitoring with Prometheus and Grafana
- [ ] Implement SSL/TLS termination
- [ ] Add database clustering for high availability
- [ ] Implement automated backup strategies
- [ ] Add comprehensive testing suite
- [ ] Implement GitOps with ArgoCD

---

**Project Status**: ✅ Complete  
**Last Updated**: August 28, 2025  
**Deployment Platforms**: Docker, Kubernetes (Kind), Amazon EKS, Helm  
**Application URL**: Varies by deployment method

This project demonstrates a complete DevOps journey from containerization to cloud-native deployment, showcasing modern practices in application deployment and orchestration.
