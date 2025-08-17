# Two-Tier Application Kubernetes Deployment

This project demonstrates the deployment of a two-tier web application using Flask and MySQL on Kubernetes. The application is containerized and deployed using Kubernetes manifests with Kind (Kubernetes in Docker).

## Architecture

The application consists of two main tiers:
1. **Frontend**: Flask web application (Python)
2. **Backend**: MySQL database with persistent storage

**Deployment Architecture:**
```
Internet → EC2 (13.126.0.103:30004) → Kind Container (30004) → 
NodePort Service (30004) → Flask Pod (5000) → MySQL Service (3306) → MySQL Pod
```

## Features

- Kubernetes-native deployment
- Persistent storage for MySQL data
- Service discovery and networking
- External access via NodePort
- Scalable architecture
- Container orchestration

## Prerequisites

- Ubuntu EC2 instance
- Docker installed and running
- Kind (Kubernetes in Docker)
- kubectl CLI tool
- Git

## Quick Start

1. Clone the repository:
```bash
git clone https://github.com/Anshuman-git-code/two-tier-app-k8s-deployment.git
cd two-tier-app-k8s-deployment
```

2. Create Kind cluster with port mapping:
```bash
kind create cluster --name two-tier-cluster --config k8s/kind-config.yaml
```

3. Deploy the application:
```bash
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
```

4. Access the application:
```
http://<your-ec2-public-ip>:30004
```

## Kubernetes Components

### Storage Components
- **PersistentVolume**: `mysql-pv.yml` - 256Mi storage for MySQL data
- **PersistentVolumeClaim**: `mysql-pvc.yml` - Storage claim for MySQL

### MySQL Components
- **Deployment**: `mysql-deployment.yml` - MySQL database deployment
- **Service**: `mysql-svc.yml` - ClusterIP service for internal access

### Application Components
- **Deployment**: `two-tier-app-deployment.yml` - Flask application deployment
- **Service**: `two-tier-app-svc.yml` - NodePort service for external access
- **Pod**: `two-tier-app-pod.yml` - Alternative pod configuration

### Cluster Configuration
- **Kind Config**: `kind-config.yaml` - Cluster setup with port mapping

## Environment Variables

The Flask application uses these environment variables:
- `MYSQL_HOST`: MySQL service IP address
- `MYSQL_USER`: MySQL username (root)
- `MYSQL_PASSWORD`: MySQL password (admin)
- `MYSQL_DB`: MySQL database name (mydb)

## File Structure

```
.
├── k8s/                                    # Kubernetes manifests
│   ├── mysql-pv.yml                       # MySQL Persistent Volume
│   ├── mysql-pvc.yml                      # MySQL Persistent Volume Claim
│   ├── mysql-deployment.yml               # MySQL Deployment
│   ├── mysql-svc.yml                      # MySQL Service
│   ├── two-tier-app-deployment.yml        # Flask App Deployment
│   ├── two-tier-app-svc.yml              # Flask App Service
│   ├── two-tier-app-pod.yml              # Flask App Pod (alternative)
│   └── kind-config.yaml                   # Kind cluster configuration
├── app.py                                  # Flask application
├── requirements.txt                        # Python dependencies
├── Dockerfile                             # Docker configuration
├── docker-compose.yml                     # Docker Compose (legacy)
├── templates/                             # HTML templates
├── DEPLOYMENT_WITH_K8S_DOCUMENTATION.md   # Detailed deployment guide
└── README.md                              # This file
```

## Deployment Commands

### Verify Deployment
```bash
# Check all resources
kubectl get all -o wide

# Check persistent volumes
kubectl get pv,pvc

# Check application logs
kubectl logs deployment/two-tier-app-deployment

# Check MySQL logs
kubectl logs deployment/mysql
```

### Troubleshooting
```bash
# Test internal connectivity
kubectl exec deployment/two-tier-app-deployment -- getent hosts mysql

# Test MySQL connection
kubectl exec deployment/mysql -- mysql -u root -padmin -e "SHOW DATABASES;"

# Port forward for testing
kubectl port-forward service/two-tier-app-service 8080:80 --address=0.0.0.0
```

## Scaling

Scale the application:
```bash
kubectl scale deployment two-tier-app-deployment --replicas=3
```

## Cleanup

Remove the deployment:
```bash
kubectl delete -f k8s/
kind delete cluster --name two-tier-cluster
```

## Legacy Docker Deployment

For Docker-based deployment (legacy), see the docker-compose.yml file:

```bash
# Using Docker Compose
docker-compose up --build

# Manual Docker deployment
docker network create twotier

# MySQL container
docker run -d \
    --name mysql \
    -v mysql-data:/var/lib/mysql \
    --network=twotier \
    -e MYSQL_DATABASE=mydb \
    -e MYSQL_ROOT_PASSWORD=admin \
    -p 3306:3306 \
    mysql:5.7

# Flask application container
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

## Documentation

For detailed deployment process, challenges faced, and solutions, see:
- [DEPLOYMENT_WITH_K8S_DOCUMENTATION.md](DEPLOYMENT_WITH_K8S_DOCUMENTATION.md)

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

This project is open source and available under the [MIT License](LICENSE).
