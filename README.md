# Comprehensive Docker Guide:
Virtualization and containerization are both technologies that enable resource sharing and application isolation, but they operate at different levels and offer distinct advantages.

## Docker Architecture
Docker Daemon (dockerd): The core engine that runs as a background process on the host system. It manages the lifecycle of Docker containers, builds images, and handles API requests from clients. The daemon exposes a REST API for communication and can interact with other daemons in distributed environments.

Docker Client: The primary interface for user interaction, typically through a command-line interface (CLI). It translates user commands like docker run and docker build into REST API requests sent to the Docker daemon. The client can communicate with multiple daemons and doesn't need to run on the same host as the daemon.

Docker Registry: A stateless, scalable storage system for Docker images. The most common public registry is Docker Hub, but organizations often use private registries for proprietary images. Registries enable image sharing, distribution, and version control.

## Understanding Docker

Docker Images
A Docker image is an immutable, read-only template that contains all the instructions and files needed to create a container. Images are composed of multiple layers, where each layer represents a change or instruction from the Dockerfile. These layers are cached and can be shared among multiple containers, optimizing storage efficiency.

Dockerfile
A Dockerfile is a text-based configuration file containing a series of instructions that Docker uses to build an image automatically. It defines the base image, application dependencies, environment configuration, and runtime commands.
```text
FROM: Specifies the base image

WORKDIR: Sets the working directory

COPY/ADD: Copies files from host to image

RUN: Executes commands during build

EXPOSE: Documents port usage

CMD/ENTRYPOINT: Defines container startup behavior
```
Docker Containers
A Docker container is a lightweight, runnable instance of a Docker image. Containers include all necessary components to run an application: code, runtime, system tools, libraries, and settings. Unlike images, containers are mutable and include an additional writable layer on top of the image layers.

## Install & Setup Docker With Proper Permissions

```bash
# Create docker group
sudo groupadd docker

# Add user to docker group
sudo usermod -aG docker $USER

# Apply group changes
newgrp docker

# Verify permissions
docker run hello-world
```
## Writing Optimized Dockerfiles
Separate build and runtime environments to exclude unnecessary build tools and dependencies from the final image
```text
# Build stage
FROM golang:1.19 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Runtime stage  
FROM alpine:latest
WORKDIR /root/
COPY --from=builder /app/myapp .
CMD ["./myapp"]

```
## Minimizing Image Size and Build Times
Dependency Management: Use package managers efficiently and clean up unnecessary files
```text
RUN apt-get update && apt-get install -y \
    package1 \
    package2 \
    && rm -rf /var/lib/apt/lists/*

```

## 2. Multi-Stage Dockerfiles

Multi-stage builds optimize image sizes by separating build and runtime environments, eliminating unnecessary build tools from final images.

### Benefits

- **Smaller Images**: Remove build dependencies from production images
- **Single Dockerfile**: Consolidate complex build processes
- **Better Security**: Fewer components mean reduced attack surface
- **Improved Performance**: Faster deployment and reduced storage costs

### Basic Multi-Stage Example

```dockerfile
# Build Stage
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production Stage
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Advanced Multi-Stage Pattern

```dockerfile
# Dependencies Stage
FROM node:20 AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Build Stage
FROM node:20 AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Runtime Stage
FROM node:20-alpine
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package*.json ./
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Best Practices for Multi-Stage Builds

- Place frequently changing stages last
- Use specific image tags for build stages
- Leverage build cache effectively
- Copy only necessary artifacts between stages
- Choose appropriate base images for each stage

## 3. Docker Errors & Troubleshooting

### Common Docker Errors and Solutions

#### Container Lifecycle Issues

**Problem**: Container fails to start
**Solutions:**

- Check logs: `docker logs [container_id]`
- Inspect configuration: `docker inspect [container_id]`
- Verify Dockerfile syntax and dependencies
- Check port conflicts

#### Application Errors

**Problem**: Application crashes inside container
**Debugging Steps:**

1. Access container shell: `docker exec -it [container] /bin/bash`
2. Check application logs within container
3. Verify environment variables
4. Test connectivity to external services

#### Networking Issues

**Problem**: Containers cannot communicate
**Solutions:**

- Inspect networks: `docker network inspect [network]`
- Test connectivity: `docker exec -it [container] ping [target]`
- Check port mappings and firewall settings
- Verify DNS resolution

#### Build Issues

**Problem**: Docker build fails
**Troubleshooting:**

- Review build output for error messages
- Disable BuildKit temporarily: `DOCKER_BUILDKIT=0 docker build`
- Run intermediate layers: `docker run -it [layer_id] sh`
- Check .dockerignore file

### Performance Debugging

**Monitor Resource Usage:**

```bash
docker stats
docker system df
docker system prune
```

**Debug Network Connectivity:**

```bash
docker exec -it container_name ping target_host
docker exec -it container_name nc -zv target_ip port
```

## 4. Docker Compose with Projects

Docker Compose simplifies multi-container application deployment by defining services, networks, and volumes in a single YAML file.

### Basic Docker Compose Structure

```yaml
version: "3.8"
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
    image: postgres:14
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Multi-Container Application Example

```yaml
version: "3.8"
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - web1
      - web2

  web1:
    build: .
    environment:
      - NODE_ENV=production
    depends_on:
      - redis
      - db

  web2:
    build: .
    environment:
      - NODE_ENV=production
    depends_on:
      - redis
      - db

  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data

  db:
    image: postgres:14
    environment:
      - POSTGRES_DB=myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  redis_data:
  postgres_data:

networks:
  default:
    driver: bridge
```

### Docker Compose Commands

```bash
# Start services
docker-compose up -d

# Build and start
docker-compose up -d --build

# Stop services
docker-compose down

# View logs
docker-compose logs -f [service]

# Scale services
docker-compose up -d --scale web=3

# Execute commands
docker-compose exec web python manage.py migrate
```

## 5. Docker Networking & Volumes

### Docker Networking Types

#### Bridge Networks (Default)

- Create isolated networks between containers
- Containers get unique IP addresses
- Enable communication between containers on same host

```bash
docker network create my-bridge
docker run --network my-bridge --name app1 nginx
docker run --network my-bridge --name app2 alpine ping app1
```

#### Host Networks

- Remove network isolation
- Share host's network stack
- Better performance but reduced security

```bash
docker run --network host nginx
```

#### Overlay Networks

- Enable communication across multiple Docker hosts
- Used in Docker Swarm and Kubernetes
- Secure encrypted communication

```bash
docker network create -d overlay my-overlay
```

#### None Networks

- Disable all networking
- Complete network isolation
- Used for security-sensitive applications

```bash
docker run --network none alpine
```

### Docker Volumes for Persistent Storage

#### Volume Types

**Named Volumes (Recommended):**

```bash
docker volume create mydata
docker run -v mydata:/data nginx
```

**Bind Mounts:**

```bash
docker run -v /host/path:/container/path nginx
```

**tmpfs Mounts (Linux only):**

```bash
docker run --tmpfs /tmp nginx
```

#### Volume Management

```bash
# Create volume
docker volume create postgres_data

# List volumes
docker volume ls

# Inspect volume
docker volume inspect postgres_data

# Remove unused volumes
docker volume prune

# Remove specific volume
docker volume rm postgres_data
```

#### Volume Usage in Docker Compose

```yaml
services:
  db:
    image: postgres:14
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      - /host/logs:/var/log/postgresql

volumes:
  postgres_data:
    driver: local
```
## 1. Private Docker Registry

### Setting up a Private Docker Registry with Nexus

Private Docker registries provide secure, controlled environments for storing and managing container images, especially critical for commercial applications.

#### Benefits of Private Registries

- **Faster Build Times**: Caching features reduce download times for frequently used images
- **Security**: Keep proprietary applications and sensitive data within your organization
- **Cost Control**: Avoid Docker Hub rate limits and subscription costs
- **Enhanced Management**: Better control over image versions and access permissions

#### Nexus Repository Setup

**Prerequisites:**

- Docker and Docker Compose installed
- Ports 8084 (management) and 19001 (Docker registry) available

**Docker Compose Configuration:**

```yaml
version: "3"
services:
  nexus_oss:
    image: sonatype/nexus3:3.45.0
    container_name: nexus3
    ports:
      - 8084:8081
      - 19001:9001
    restart: always
    volumes:
      - nexus_data:/nexus-data
volumes:
  nexus_data:
```

**Setup Steps:**

1. Start Nexus: `docker-compose up -d`
2. Access web interface at `http://localhost:8084`
3. Retrieve initial admin password from `/nexus-data/admin.password`
4. Complete setup wizard (enable anonymous access recommended)
5. Create Docker repository (gear icon → repositories → create repository)

#### Using Docker Hub as Proxy

Configure Nexus as a Docker Hub proxy to cache images locally:

- Repository type: Docker Proxy
- Remote URL: `https://registry-1.docker.io`
- Port: 8082
- Enable anonymous pull

### Pushing and Pulling Images

**Login to Registry:**

```bash
docker login localhost:19001
```

**Tag and Push Images:**

```bash
docker tag myapp:latest localhost:19001/myapp:latest
docker push localhost:19001/myapp:latest
```

**Pull from Registry:**

```bash
docker pull localhost:19001/myapp:latest
```


## 8. Docker Project: Multi-Tier Node.js + MySQL

### Node.js Application with MySQL Database

#### Project Structure

```
nodejs-mysql-app/
├── docker-compose.yml
├── frontend/
│   ├── Dockerfile
│   └── package.json
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
└── database/
    └── init.sql
```

#### Backend Dockerfile

```dockerfile
FROM node:18-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS build
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package*.json ./
EXPOSE 5000
USER node
CMD ["node", "dist/server.js"]
```

#### Frontend Dockerfile

```dockerfile
# Build Stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production Stage
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### Docker Compose Configuration

```yaml
version: "3.8"
services:
  frontend:
    build:
      context: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - app-network

  backend:
    build:
      context: ./backend
    ports:
      - "5000:5000"
    environment:
      - NODE_ENV=production
      - DB_HOST=mysql
      - DB_USER=appuser
      - DB_PASSWORD=apppassword
      - DB_NAME=myapp
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - app-network

  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=myapp
      - MYSQL_USER=appuser
      - MYSQL_PASSWORD=apppassword
    volumes:
      - mysql_data:/var/lib/mysql
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10
    networks:
      - app-network

volumes:
  mysql_data:

networks:
  app-network:
    driver: bridge
```

#### Database Initialization

```sql
-- init.sql
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products (name, price, description) VALUES
('Product 1', 29.99, 'First product'),
('Product 2', 39.99, 'Second product');
```

### Best Practices Summary

1. **Security**

   - Use non-root users in containers
   - Implement health checks
   - Keep base images updated
   - Use secrets management

2. **Performance**

   - Leverage multi-stage builds
   - Optimize layer caching
   - Use .dockerignore files
   - Choose appropriate base images

3. **Maintainability**

   - Use Docker Compose for multi-container apps
   - Implement proper logging
   - Document configurations
   - Version control all Docker files

4. **Production Readiness**
   - Configure resource limits
   - Implement monitoring
   - Use persistent volumes for data
   - Plan for backup and recovery

This comprehensive guide covers all aspects of Docker from basic containerization to complex multi-tier applications, providing practical examples and best practices for real-world deployment scenarios.
