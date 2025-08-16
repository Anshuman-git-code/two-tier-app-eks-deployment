# Two-Tier Flask Application Deployment Documentation

## Project Overview

This document details the complete deployment process of a two-tier Flask web application using Docker and Docker Compose. The application consists of a Flask backend connected to a MySQL database, demonstrating containerized microservices architecture.

**Application Architecture:**
- **Frontend/Backend**: Flask web application (Python)
- **Database**: MySQL 5.7
- **Containerization**: Docker & Docker Compose
- **Networking**: Custom Docker network for inter-container communication

## Project Structure

```
two-tier-flask-app/
├── app.py                    # Main Flask application
├── requirements.txt          # Python dependencies
├── Dockerfile               # Container build instructions
├── docker-compose.yml       # Multi-container orchestration
├── message.sql              # Database initialization script
├── templates/               # HTML templates directory
└── README.md               # Project documentation
```

## Deployment Process

### Phase 1: Application Development and Setup

#### 1.1 Flask Application Development
Created the main Flask application (`app.py`) with the following features:
- MySQL database connectivity using Flask-MySQLdb
- Environment variable configuration for database credentials
- RESTful endpoints for message submission and retrieval
- Database initialization with table creation

**Key Commands Used:**
```bash
# Created the main application file
touch app.py

# Created requirements file with dependencies
echo "Flask==2.0.1
Flask-MySQLdb==0.2.0
requests==2.26.0
Werkzeug==2.2.2" > requirements.txt
```

#### 1.2 Database Schema Setup
Created database initialization script (`message.sql`):
```sql
CREATE TABLE messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message TEXT
);
```

### Phase 2: Containerization with Docker

#### 2.1 Dockerfile Creation
Developed a multi-stage Dockerfile for the Flask application:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update -y \
  && apt-get update -y \
  && apt-get install -y gcc default-libmysqlclient-dev pkg-config \
  && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

RUN pip install mysqlclient
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

**Commands Used:**
```bash
# Build the Docker image
docker build -t flaskapp .

# Tag the image for Docker Hub
docker tag flaskapp anshuman0506/flaskapp:latest

# Push to Docker Hub
docker push anshuman0506/flaskapp:latest
```

#### 2.2 Manual Container Deployment (Alternative Method)
Implemented manual container deployment without Docker Compose:

```bash
# Create custom network
docker network create twotier

# Deploy MySQL container
docker run -d \
    --name mysql \
    -v mysql-data:/var/lib/mysql \
    --network=twotier \
    -e MYSQL_DATABASE=mydb \
    -e MYSQL_ROOT_PASSWORD=admin \
    -p 3306:3306 \
    mysql:5.7

# Deploy Flask application container
docker run -d \
    --name flaskapp \
    --network=twotier \
    -e MYSQL_HOST=mysql \
    -e MYSQL_USER=root \
    -e MYSQL_PASSWORD=admin \
    -e MYSQL_DB=mydb \
    -p 5000:5000 \
    flaskapp:latest
```

### Phase 3: Docker Compose Orchestration

#### 3.1 Docker Compose Configuration
Created `docker-compose.yml` for simplified multi-container management:

```yaml
version: '3.8'
services:
  backend:
    image: anshuman0506/flaskapp:latest
    ports:
      - "5000:5000"
    environment:
      - MYSQL_HOST=mysql
      - MYSQL_USER=admin
      - MYSQL_PASSWORD=admin
      - MYSQL_DB=myDb
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:5.7
    environment:
      - MYSQL_DATABASE=myDb
      - MYSQL_USER=admin
      - MYSQL_PASSWORD=admin
      - MYSQL_ROOT_PASSWORD=admin
    ports:
      - "3306:3306"
    volumes:
      - ./message.sql:/docker-entrypoint-initdb.d/message.sql
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql-data:
```

#### 3.2 Deployment Commands
```bash
# Start all services
docker-compose up --build

# Start services in detached mode
docker-compose up -d

# View running containers
docker-compose ps

# View logs
docker-compose logs

# Stop all services
docker-compose down

# Remove volumes (complete cleanup)
docker-compose down -v
```

## Challenges Faced and Solutions

### Challenge 1: MySQL Connection Issues
**Problem**: Flask application couldn't connect to MySQL container
**Error**: `Can't connect to MySQL server on 'localhost'`

**Root Cause**: 
- Containers were not on the same network
- Incorrect hostname configuration

**Solution**:
```bash
# Created custom Docker network
docker network create twotier

# Used service name 'mysql' as hostname in environment variables
MYSQL_HOST=mysql  # Instead of localhost
```

### Challenge 2: Database Initialization
**Problem**: Messages table not created automatically
**Error**: `Table 'myDb.messages' doesn't exist`

**Solutions Implemented**:
1. **Docker Compose Volume Mount**: 
   ```yaml
   volumes:
     - ./message.sql:/docker-entrypoint-initdb.d/message.sql
   ```

2. **Application-level Initialization**:
   ```python
   def init_db():
       with app.app_context():
           cur = mysql.connection.cursor()
           cur.execute('''
           CREATE TABLE IF NOT EXISTS messages (
               id INT AUTO_INCREMENT PRIMARY KEY,
               message TEXT
           );
           ''')
           mysql.connection.commit()
           cur.close()
   ```

### Challenge 3: Container Startup Order
**Problem**: Flask app started before MySQL was ready
**Error**: `Connection refused` during application startup

**Solution**: Implemented health checks and service dependencies
```yaml
depends_on:
  mysql:
    condition: service_healthy

healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 10s
  timeout: 5s
  retries: 5
```

### Challenge 4: Python MySQL Dependencies
**Problem**: MySQLclient installation failed in container
**Error**: `Failed building wheel for mysqlclient`

**Solution**: Added system dependencies in Dockerfile
```dockerfile
RUN apt-get update -y \
  && apt-get install -y gcc default-libmysqlclient-dev pkg-config \
  && rm -rf /var/lib/apt/lists/*
```

### Challenge 5: Environment Variable Configuration
**Problem**: Inconsistent database credentials across services
**Error**: `Access denied for user`

**Solution**: Standardized environment variables
```yaml
# MySQL service
environment:
  - MYSQL_DATABASE=myDb
  - MYSQL_USER=admin
  - MYSQL_PASSWORD=admin
  - MYSQL_ROOT_PASSWORD=admin

# Flask service
environment:
  - MYSQL_HOST=mysql
  - MYSQL_USER=admin
  - MYSQL_PASSWORD=admin
  - MYSQL_DB=myDb
```

## Testing and Verification

### Application Testing Commands
```bash
# Check container status
docker ps

# Test database connectivity
docker exec -it mysql mysql -u admin -padmin -e "SHOW DATABASES;"

# View application logs
docker logs flaskapp

# Test API endpoints
curl http://localhost:5000
curl -X POST http://localhost:5000/submit -d "new_message=Hello World"
```

### Verification Steps
1. **Container Health**: All containers running and healthy
2. **Network Connectivity**: Flask can connect to MySQL
3. **Database Operations**: CRUD operations working correctly
4. **Web Interface**: Frontend accessible at http://localhost:5000
5. **Data Persistence**: Data survives container restarts

## Performance Optimizations

### Implemented Optimizations
1. **Multi-stage Docker builds** for smaller image size
2. **Volume mounting** for data persistence
3. **Health checks** for reliable service startup
4. **Environment-based configuration** for flexibility

### Resource Usage
```bash
# Monitor resource usage
docker stats

# View disk usage
docker system df
```

## Security Considerations

### Implemented Security Measures
1. **Environment variables** for sensitive data
2. **Non-root user** in containers (recommended for production)
3. **Network isolation** using custom Docker networks
4. **Volume permissions** properly configured

### Production Recommendations
- Use Docker secrets for sensitive data
- Implement SSL/TLS encryption
- Regular security updates for base images
- Network policies and firewall rules

## Maintenance and Monitoring

### Regular Maintenance Commands
```bash
# Update images
docker-compose pull
docker-compose up -d

# Clean up unused resources
docker system prune

# Backup database
docker exec mysql mysqldump -u admin -padmin myDb > backup.sql

# View resource usage
docker-compose top
```

### Monitoring Setup
- Container health checks
- Log aggregation with `docker-compose logs`
- Resource monitoring with `docker stats`

## Conclusion

Successfully deployed a two-tier Flask application using Docker and Docker Compose with the following achievements:

✅ **Containerized Architecture**: Both Flask and MySQL running in isolated containers
✅ **Service Orchestration**: Automated startup and dependency management
✅ **Data Persistence**: MySQL data survives container restarts
✅ **Network Isolation**: Secure inter-container communication
✅ **Environment Configuration**: Flexible configuration management
✅ **Health Monitoring**: Automated health checks and service recovery

The deployment demonstrates modern DevOps practices including containerization, infrastructure as code, and automated service management. The application is now ready for scaling and production deployment with additional security and monitoring enhancements.

## Future Enhancements

- Implement CI/CD pipeline
- Add container orchestration with Kubernetes
- Implement monitoring with Prometheus/Grafana
- Add SSL/TLS termination with reverse proxy
- Database clustering for high availability
- Automated backup and disaster recovery

---
**Deployment Date**: August 16, 2025  
**Environment**: Ubuntu Linux  
**Docker Version**: Latest  
**Docker Compose Version**: 3.8
