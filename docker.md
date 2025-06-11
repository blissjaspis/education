# Docker Education: From Beginner to Expert

## Table of Contents

1. [Beginner Level](#beginner-level)
   - [What is Docker?](#what-is-docker)
   - [Installation](#installation)
   - [Basic Commands](#basic-commands)
   - [Working with Images](#working-with-images)
   - [Working with Containers](#working-with-containers)

2. [Intermediate Level](#intermediate-level)
   - [Dockerfile](#dockerfile)
   - [Docker Compose](#docker-compose)
   - [Volumes and Data Persistence](#volumes-and-data-persistence)
   - [Networking](#networking)
   - [Environment Variables](#environment-variables)

3. [Advanced Level](#advanced-level)
   - [Multi-stage Builds](#multi-stage-builds)
   - [Docker Security](#docker-security)
   - [Container Orchestration](#container-orchestration)
   - [Production Best Practices](#production-best-practices)
   - [Monitoring and Logging](#monitoring-and-logging)

---

## Beginner Level

### What is Docker?

Docker is a containerization platform that allows you to package applications and their dependencies into lightweight, portable containers. Think of containers as isolated environments that can run anywhere Docker is installed.

**Key Concepts:**
- **Container**: A running instance of an image
- **Image**: A template used to create containers
- **Dockerfile**: A script containing instructions to build an image
- **Registry**: A storage location for Docker images (e.g., Docker Hub)

**Benefits:**
- Consistency across environments
- Resource efficiency
- Fast deployment
- Scalability
- Isolation

### Installation

#### macOS
```bash
# Install Docker Desktop
brew install --cask docker

# Or download from Docker website
# https://www.docker.com/products/docker-desktop
```

#### Linux (Ubuntu/Debian)
```bash
# Update package index
sudo apt-get update

# Install Docker
sudo apt-get install docker.io

# Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group (optional)
sudo usermod -aG docker $USER
```

#### Windows
Download Docker Desktop from the official website and follow the installation wizard.

**Verify Installation:**
```bash
docker --version
docker run hello-world
```

### Basic Commands

#### Docker Information
```bash
# Check Docker version
docker --version

# Display system-wide information
docker info

# Show Docker help
docker --help
```

#### Image Management
```bash
# List local images
docker images

# Pull an image from registry
docker pull nginx

# Remove an image
docker rmi nginx

# Search for images
docker search ubuntu
```

#### Container Lifecycle
```bash
# Run a container
docker run nginx

# Run container in background (detached)
docker run -d nginx

# Run container with custom name
docker run --name my-nginx nginx

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop a container
docker stop container-id

# Start a stopped container
docker start container-id

# Remove a container
docker rm container-id

# Remove all stopped containers
docker container prune
```

### Working with Images

#### Pulling Images
```bash
# Pull latest version
docker pull ubuntu

# Pull specific version/tag
docker pull ubuntu:20.04

# Pull from different registry
docker pull gcr.io/google-containers/busybox
```

#### Creating Images from Containers
```bash
# Make changes to a running container, then commit
docker run -it ubuntu bash
# (make changes inside container)
docker commit container-id my-custom-image
```

### Working with Containers

#### Interactive Containers
```bash
# Run container interactively
docker run -it ubuntu bash

# Execute command in running container
docker exec -it container-id bash

# Copy files to/from container
docker cp file.txt container-id:/path/to/destination
docker cp container-id:/path/to/file.txt ./
```

#### Port Mapping
```bash
# Map container port to host port
docker run -p 8080:80 nginx

# Map to random host port
docker run -P nginx

# Check port mappings
docker port container-id
```

#### Volume Mounting
```bash
# Mount host directory to container
docker run -v /host/path:/container/path ubuntu

# Mount current directory
docker run -v $(pwd):/app ubuntu
```

---

## Intermediate Level

### Dockerfile

A Dockerfile is a text file containing instructions to build a Docker image.

#### Basic Dockerfile Structure
```dockerfile
# Use official base image
FROM ubuntu:20.04

# Set working directory
WORKDIR /app

# Copy files from host to container
COPY . /app

# Install dependencies
RUN apt-get update && apt-get install -y python3

# Set environment variables
ENV PYTHONPATH=/app

# Expose port
EXPOSE 8000

# Define default command
CMD ["python3", "app.py"]
```

#### Common Dockerfile Instructions
```dockerfile
# FROM: Base image
FROM node:16-alpine

# LABEL: Add metadata
LABEL maintainer="your-email@example.com"

# RUN: Execute commands during build
RUN npm install

# COPY: Copy files/directories
COPY package.json .
COPY src/ ./src/

# ADD: Similar to COPY but can handle URLs and tar files
ADD https://example.com/file.tar.gz /tmp/

# WORKDIR: Set working directory
WORKDIR /usr/src/app

# ENV: Set environment variables
ENV NODE_ENV=production

# ARG: Build-time variables
ARG VERSION=1.0.0

# EXPOSE: Document port usage
EXPOSE 3000

# VOLUME: Create mount point
VOLUME ["/data"]

# USER: Set user for subsequent instructions
USER node

# CMD: Default command (can be overridden)
CMD ["npm", "start"]

# ENTRYPOINT: Always executed command
ENTRYPOINT ["docker-entrypoint.sh"]
```

#### Building Images
```bash
# Build image from Dockerfile
docker build -t my-app .

# Build with specific Dockerfile
docker build -f Dockerfile.prod -t my-app:prod .

# Build with build arguments
docker build --build-arg VERSION=2.0.0 -t my-app .

# Build without cache
docker build --no-cache -t my-app .
```

#### Example: Node.js Application
```dockerfile
FROM node:16-alpine

WORKDIR /usr/src/app

# Copy package files first (better caching)
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Change ownership
RUN chown -R nextjs:nodejs /usr/src/app
USER nextjs

EXPOSE 3000

CMD ["npm", "start"]
```

### Docker Compose

Docker Compose allows you to define and run multi-container applications using a YAML file.

#### Basic docker-compose.yml
```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    depends_on:
      - db

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

#### Docker Compose Commands
```bash
# Start services
docker-compose up

# Start in background
docker-compose up -d

# Stop services
docker-compose down

# Build and start
docker-compose up --build

# View logs
docker-compose logs

# Scale services
docker-compose up --scale web=3

# Execute command in service
docker-compose exec web bash
```

#### Advanced Compose Example
```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - api

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:

networks:
  default:
    driver: bridge
```

### Volumes and Data Persistence

#### Types of Volumes
1. **Anonymous Volumes**: Managed by Docker
2. **Named Volumes**: Persistent, managed by Docker
3. **Bind Mounts**: Direct host filesystem access

```bash
# Named volume
docker run -v myvolume:/data ubuntu

# Bind mount
docker run -v /host/path:/container/path ubuntu

# Anonymous volume
docker run -v /data ubuntu
```

#### Volume Management
```bash
# List volumes
docker volume ls

# Create volume
docker volume create myvolume

# Inspect volume
docker volume inspect myvolume

# Remove volume
docker volume rm myvolume

# Remove unused volumes
docker volume prune
```

#### Example: Database with Persistent Storage
```yaml
version: '3.8'

services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: myapp
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3306:3306"

volumes:
  mysql_data:
    driver: local
```

### Networking

#### Network Types
1. **Bridge**: Default network for containers
2. **Host**: Use host's network stack
3. **None**: No networking
4. **Overlay**: Multi-host networking

#### Network Commands
```bash
# List networks
docker network ls

# Create network
docker network create mynetwork

# Inspect network
docker network inspect mynetwork

# Connect container to network
docker network connect mynetwork container-id

# Disconnect container from network
docker network disconnect mynetwork container-id
```

#### Custom Networks in Compose
```yaml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      - frontend

  api:
    image: node:16
    networks:
      - frontend
      - backend

  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

### Environment Variables

#### Methods to Set Environment Variables
```bash
# Command line
docker run -e NODE_ENV=production myapp

# Environment file
docker run --env-file .env myapp

# Docker Compose
```

```yaml
services:
  app:
    image: myapp
    environment:
      - NODE_ENV=production
      - API_KEY=${API_KEY}
    env_file:
      - .env
```

#### Example .env file
```
NODE_ENV=production
DATABASE_URL=postgresql://user:pass@localhost:5432/db
API_KEY=your-secret-key
```

---

## Advanced Level

### Multi-stage Builds

Multi-stage builds help create smaller, more secure production images.

#### Example: Node.js Multi-stage Build
```dockerfile
# Build stage
FROM node:16-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM node:16-alpine AS production

WORKDIR /app

# Copy only production dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy built application from builder stage
COPY --from=builder /app/dist ./dist

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
USER nextjs

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

#### Example: Go Multi-stage Build
```dockerfile
# Build stage
FROM golang:1.19-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Production stage
FROM alpine:latest AS production

RUN apk --no-cache add ca-certificates
WORKDIR /root/

# Copy binary from builder stage
COPY --from=builder /app/main .

CMD ["./main"]
```

### Docker Security

#### Security Best Practices

1. **Use Official Base Images**
```dockerfile
# Good
FROM node:16-alpine

# Avoid
FROM ubuntu:latest
RUN curl -sL https://deb.nodesource.com/setup_16.x | bash -
RUN apt-get install -y nodejs
```

2. **Run as Non-root User**
```dockerfile
FROM node:16-alpine

# Create user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Set ownership
COPY --chown=nextjs:nodejs . .
USER nextjs
```

3. **Minimize Attack Surface**
```dockerfile
# Use minimal base images
FROM node:16-alpine

# Remove unnecessary packages
RUN apk del build-dependencies

# Use .dockerignore
```

4. **Scan Images for Vulnerabilities**
```bash
# Scan with Docker Scout
docker scout cves myapp:latest

# Scan with Trivy
trivy image myapp:latest
```

#### Security Configuration
```yaml
version: '3.8'

services:
  app:
    image: myapp
    # Read-only root filesystem
    read_only: true
    # Drop all capabilities
    cap_drop:
      - ALL
    # Add only needed capabilities
    cap_add:
      - NET_BIND_SERVICE
    # Security options
    security_opt:
      - no-new-privileges:true
    # Resource limits
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
```

### Container Orchestration

#### Introduction to Kubernetes
While Docker Compose is great for development, Kubernetes is the standard for production orchestration.

#### Docker Swarm (Simple Orchestration)
```bash
# Initialize swarm
docker swarm init

# Deploy stack
docker stack deploy -c docker-compose.yml mystack

# List services
docker service ls

# Scale service
docker service scale mystack_web=3

# Remove stack
docker stack rm mystack
```

#### Example Swarm Compose File
```yaml
version: '3.8'

services:
  web:
    image: myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    ports:
      - "80:3000"
    networks:
      - webnet

  db:
    image: postgres:13
    deploy:
      placement:
        constraints: [node.role == manager]
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - webnet

volumes:
  db-data:

networks:
  webnet:
```

### Production Best Practices

#### 1. Image Optimization
```dockerfile
# Use specific versions
FROM node:16.14.2-alpine

# Combine RUN commands
RUN apk update && \
    apk add --no-cache curl && \
    rm -rf /var/cache/apk/*

# Order layers by change frequency
COPY package.json .
RUN npm install
COPY . .
```

#### 2. Health Checks
```dockerfile
# Dockerfile health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

```yaml
# Compose health check
services:
  web:
    image: myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

#### 3. Resource Management
```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

#### 4. Secrets Management
```yaml
version: '3.8'

services:
  app:
    image: myapp
    secrets:
      - db_password
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    file: ./db_password.txt
```

#### 5. Configuration Management
```yaml
services:
  app:
    image: myapp
    configs:
      - source: app_config
        target: /app/config.yml

configs:
  app_config:
    file: ./config.yml
```

### Monitoring and Logging

#### Logging Best Practices
```dockerfile
# Log to stdout/stderr
RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log
```

#### Monitoring Stack with Prometheus
```yaml
version: '3.8'

services:
  app:
    image: myapp
    ports:
      - "3000:3000"

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

#### Log Aggregation with ELK Stack
```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.14.0
    environment:
      - discovery.type=single-node

  logstash:
    image: docker.elastic.co/logstash/logstash:7.14.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf

  kibana:
    image: docker.elastic.co/kibana/kibana:7.14.0
    ports:
      - "5601:5601"
```

## Practice Exercises

### Beginner Exercises
1. Create a simple web server using nginx
2. Build a custom image with a Python application
3. Use volumes to persist data

### Intermediate Exercises
1. Create a multi-service application with Docker Compose
2. Implement environment-specific configurations
3. Set up a database with persistent storage

### Advanced Exercises
1. Implement a multi-stage build for optimization
2. Set up monitoring and logging
3. Create a production-ready deployment

## Useful Commands Reference

### Cleanup Commands
```bash
# Remove all stopped containers
docker container prune

# Remove unused images
docker image prune

# Remove unused volumes
docker volume prune

# Remove unused networks
docker network prune

# Remove everything unused
docker system prune -a
```

### Debugging Commands
```bash
# View container logs
docker logs container-id

# Follow logs
docker logs -f container-id

# Inspect container
docker inspect container-id

# View container processes
docker top container-id

# Get container stats
docker stats container-id
```

## Additional Resources

- [Docker Official Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Play with Docker](https://labs.play-with-docker.com/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

---

*This documentation provides a comprehensive learning path for Docker. Start with the beginner section and gradually progress through intermediate and advanced topics. Practice with real projects to solidify your understanding.*
