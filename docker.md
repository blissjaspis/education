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
   - [Multi-Platform Builds](#multi-platform-builds)
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

### Multi-Platform Builds

Building Docker images for multiple CPU architectures (AMD64, ARM64, etc.) ensures your application runs on different hardware platforms like x86 servers, Apple Silicon, and ARM-based devices.

#### Understanding Platform Architectures

Common platforms:
- **linux/amd64**: Standard x86_64 architecture (Intel/AMD processors)
- **linux/arm64**: 64-bit ARM architecture (Apple Silicon, AWS Graviton, Raspberry Pi 4+)
- **linux/arm/v7**: 32-bit ARM architecture (Older Raspberry Pi)
- **linux/arm/v6**: ARMv6 architecture (Raspberry Pi Zero)

#### Setting Up Docker Buildx

Docker Buildx is the extended build capabilities that support multi-platform builds.

```bash
# Check if buildx is available
docker buildx version

# Create a new builder instance
docker buildx create --name multiplatform-builder --use

# Bootstrap the builder (downloads necessary components)
docker buildx inspect --bootstrap

# List available builders
docker buildx ls
```

#### Building for Multiple Platforms

##### Basic Multi-Platform Build
```bash
# Build for AMD64 and ARM64
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .

# Build and push to registry (required for multi-platform)
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t username/myapp:latest \
  --push .

# Build and load locally (single platform only)
docker buildx build \
  --platform linux/amd64 \
  -t myapp:latest \
  --load .
```

##### Build with Tags
```bash
# Multiple tags for multi-platform image
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t username/myapp:latest \
  -t username/myapp:v1.0.0 \
  --push .
```

#### Multi-Platform Dockerfile Best Practices

##### Platform-Specific Instructions
```dockerfile
# Use BUILDPLATFORM and TARGETPLATFORM
FROM --platform=$BUILDPLATFORM node:16-alpine AS builder

# Set architecture-specific variables
ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

WORKDIR /app

# Display build information
RUN echo "Building on $BUILDPLATFORM for $TARGETPLATFORM"

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM node:16-alpine

WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

CMD ["node", "dist/index.js"]
```

##### Architecture-Specific Dependencies
```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.19-alpine AS builder

ARG TARGETARCH
ARG TARGETOS

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Build for specific target architecture
RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    CGO_ENABLED=0 \
    go build -o main .

# Use platform-neutral final image
FROM alpine:latest

RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /app/main .

CMD ["./main"]
```

#### Advanced Multi-Platform Examples

##### Example: Python Application
```dockerfile
FROM --platform=$BUILDPLATFORM python:3.11-slim AS builder

ARG TARGETPLATFORM

WORKDIR /app

# Install platform-specific dependencies if needed
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

COPY . .

# Final stage
FROM python:3.11-slim

WORKDIR /app

# Copy Python packages from builder
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app /app

# Ensure scripts are in PATH
ENV PATH=/root/.local/bin:$PATH

CMD ["python", "app.py"]
```

##### Example: Rust Multi-Platform Build
```dockerfile
FROM --platform=$BUILDPLATFORM rust:1.70-alpine AS builder

ARG TARGETARCH
ARG TARGETOS

WORKDIR /app

# Install cross-compilation tools if needed
RUN apk add --no-cache musl-dev

# Copy dependency files
COPY Cargo.toml Cargo.lock ./

# Build dependencies separately for better caching
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src

# Copy actual source
COPY src ./src

# Build for target architecture
RUN cargo build --release

FROM alpine:latest

WORKDIR /app

COPY --from=builder /app/target/release/myapp .

CMD ["./myapp"]
```

#### Docker Compose with Multi-Platform

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      platforms:
        - linux/amd64
        - linux/arm64
    image: myapp:latest
    ports:
      - "3000:3000"
```

#### Building and Testing Multi-Platform Images

```bash
# Build for all platforms and push
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t username/myapp:latest \
  --push .

# Inspect the multi-platform manifest
docker buildx imagetools inspect username/myapp:latest

# Test specific platform locally
docker buildx build \
  --platform linux/arm64 \
  -t myapp:arm64 \
  --load .

docker run --platform linux/arm64 myapp:arm64
```

#### GitHub Actions Multi-Platform Build

```yaml
name: Docker Multi-Platform Build

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          platforms: linux/amd64,linux/arm64,linux/arm/v7
          push: true
          tags: |
            username/myapp:latest
            username/myapp:${{ github.sha }}
          cache-from: type=registry,ref=username/myapp:latest
          cache-to: type=inline
```

#### Best Practices for Multi-Platform Builds

1. **Use Platform-Aware Base Images**
```dockerfile
# Official images typically support multiple platforms
FROM node:16-alpine  # Supports amd64, arm64, arm/v7, etc.
```

2. **Leverage Build Cache**
```bash
# Use cache from registry
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=username/myapp:latest \
  --cache-to type=inline \
  -t username/myapp:latest \
  --push .
```

3. **Test on Target Platforms**
```bash
# Use QEMU to test different architectures locally
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# Run ARM container on AMD64 host
docker run --platform linux/arm64 myapp:latest
```

4. **Optimize Build Time**
```dockerfile
# Use BUILDPLATFORM for build tools
FROM --platform=$BUILDPLATFORM node:16-alpine AS deps

# This stage runs on the build machine's architecture (faster)
RUN npm ci

# Only the final image needs to match TARGETPLATFORM
FROM node:16-alpine
COPY --from=deps /app/node_modules ./node_modules
```

5. **Handle Platform-Specific Dependencies**
```dockerfile
ARG TARGETARCH

# Install architecture-specific binaries
RUN case ${TARGETARCH} in \
      "amd64")  ARCH=x86_64 ;; \
      "arm64")  ARCH=aarch64 ;; \
      "arm")    ARCH=armv7l ;; \
      *)        echo "Unsupported architecture"; exit 1 ;; \
    esac && \
    wget https://example.com/binary-${ARCH}.tar.gz
```

#### Common Issues and Solutions

**Issue: Build is slow**
```bash
# Solution: Use native builders when possible
docker buildx create --name fast-builder \
  --driver docker-container \
  --platform linux/amd64 \
  --use
```

**Issue: Cannot load multi-platform image locally**
```bash
# Solution: Build for single platform with --load
docker buildx build \
  --platform linux/amd64 \
  -t myapp:latest \
  --load .

# Or extract specific platform from registry
docker pull --platform linux/arm64 username/myapp:latest
```

**Issue: Cross-compilation errors**
```dockerfile
# Solution: Use emulation or native compilation
FROM --platform=$BUILDPLATFORM golang:1.19-alpine AS builder

# Install cross-compilation tools
RUN apk add --no-cache gcc musl-dev

# Enable CGO with proper cross-compilation
ARG TARGETARCH
ENV CGO_ENABLED=1
ENV GOARCH=${TARGETARCH}
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
2. Build and deploy a multi-platform image for AMD64 and ARM64
3. Set up monitoring and logging
4. Create a production-ready deployment with health checks and resource limits

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
