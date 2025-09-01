# Comprehensive Docker Guide: From Private Registries to Multi-Tier Applications

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
version: '3'
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
version: '3.8'
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

## 6. Docker Project: Java-Based Application

### Spring Boot Application Containerization

#### Traditional Approach
```dockerfile
FROM eclipse-temurin:17-jdk
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

#### Multi-Stage Build for Java
```dockerfile
# Build Stage
FROM maven:3.8-openjdk-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime Stage
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Gradle Alternative
```dockerfile
# Build Stage
FROM gradle:7.6-jdk17 AS builder
WORKDIR /app
COPY build.gradle settings.gradle ./
COPY gradle ./gradle
RUN gradle dependencies
COPY src ./src
RUN gradle build -x test

# Runtime Stage
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 7. Docker Project: Multi-Tier .NET + MongoDB

### .NET Core Application with MongoDB

#### Application Structure
```
dotnet-mongodb-app/
├── docker-compose.yml
├── Dockerfile
├── src/
│   ├── Controllers/
│   ├── Models/
│   ├── Services/
│   └── Program.cs
└── appsettings.json
```

#### Dockerfile for .NET Application
```dockerfile
# Build Stage
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder
WORKDIR /app
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o out

# Runtime Stage
FROM mcr.microsoft.com/dotnet/aspnet:7.0-alpine
WORKDIR /app
COPY --from=builder /app/out .
EXPOSE 80
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

#### Docker Compose Configuration
```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "5000:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__MongoDB=mongodb://mongodb:27017/myapp
    depends_on:
      - mongodb
    networks:
      - app-network

  mongodb:
    image: mongo:6.0
    container_name: mongodb
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
      - MONGO_INITDB_DATABASE=myapp
    volumes:
      - mongodb_data:/data/db
      - ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
    ports:
      - "27017:27017"
    networks:
      - app-network

volumes:
  mongodb_data:

networks:
  app-network:
    driver: bridge
```

#### MongoDB Initialization Script
```javascript
// init-mongo.js
db = db.getSiblingDB('myapp');
db.createUser({
  user: 'appuser',
  pwd: 'apppassword',
  roles: [{ role: 'readWrite', db: 'myapp' }]
});

db.products.insertMany([
  { name: 'Product 1', price: 29.99 },
  { name: 'Product 2', price: 39.99 }
]);
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
version: '3.8'
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

## 9. Docker Project: Multi-Tier Python + PostgreSQL

### FastAPI Application with PostgreSQL

#### Project Structure
```
python-postgres-app/
├── docker-compose.yml
├── requirements.txt
├── Dockerfile
├── app/
│   ├── main.py
│   ├── models.py
│   ├── database.py
│   └── crud.py
└── database/
    └── init.sql
```

#### FastAPI Application Dockerfile
```dockerfile
# Build Stage
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production Stage
FROM python:3.11-slim
WORKDIR /app

# Copy installed packages from builder stage
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Copy application code
COPY . .

# Create non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Django Alternative Dockerfile
```dockerfile
FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Collect static files
RUN python manage.py collectstatic --noinput

EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "myproject.wsgi:application"]
```

#### Docker Compose Configuration
```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://appuser:apppassword@postgres:5432/myapp
      - PYTHONPATH=/app
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - .:/app
    networks:
      - app-network

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=apppassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d myapp"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

#### Application Code Example (FastAPI)
```python
# app/main.py
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from . import crud, models, database

models.Base.metadata.create_all(bind=database.engine)

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello from FastAPI with PostgreSQL!"}

@app.get("/products/")
def read_products(skip: int = 0, limit: int = 100, db: Session = Depends(database.get_db)):
    products = crud.get_products(db, skip=skip, limit=limit)
    return products

@app.post("/products/")
def create_product(product: dict, db: Session = Depends(database.get_db)):
    return crud.create_product(db=db, product=product)
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