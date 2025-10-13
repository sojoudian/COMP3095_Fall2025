# Deployment - Docker and Docker Compose

## Overview

In this final section, you will containerize the order-service and deploy the complete microservices architecture using Docker Compose. You'll create a multistage Dockerfile and update docker-compose.yml to orchestrate all services.

---

## Part 1: Create Dockerfile for order-service

## Step 1: Create Dockerfile

### 1.1 Create File

**Location:** `order-service/Dockerfile`

**Steps:**
1. Right-click on `order-service` folder in IntelliJ
2. Select **New → File**
3. Name: `Dockerfile`
4. Click **OK**

### 1.2 Write Dockerfile

**IMPORTANT:** Use `eclipse-temurin` base images instead of `openjdk` images. The `openjdk` images are missing required utilities like `xargs` that Gradle needs, which will cause build failures with the error "xargs is not available".

**Complete Dockerfile:**

```dockerfile
# ============================================
# Stage 1: Build Stage
# ============================================
FROM eclipse-temurin:21-jdk AS builder

# Set working directory
WORKDIR /app

# Copy Gradle wrapper and build files
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .

# Copy source code
COPY src src

# Make gradlew executable
RUN chmod +x gradlew

# Build the application (skip tests for faster builds)
RUN ./gradlew clean build -x test

# ============================================
# Stage 2: Runtime Stage
# ============================================
FROM eclipse-temurin:21-jre

# Set working directory
WORKDIR /app

# Copy JAR from build stage
COPY --from=builder /app/build/libs/*.jar app.jar

# Create non-root user for security
RUN addgroup --system spring && adduser --system --group spring

# Change ownership of app directory
RUN chown -R spring:spring /app

# Switch to non-root user
USER spring:spring

# Expose port
EXPOSE 8082

# Set environment variables
ENV SPRING_PROFILES_ACTIVE=docker

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8082/actuator/health || exit 1

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Understanding the Dockerfile:**

**Stage 1 - Build Stage:**
- `FROM eclipse-temurin:21-jdk AS builder` - Use Eclipse Temurin JDK 21 (includes all build tools)
- Copies Gradle wrapper and source code
- `RUN ./gradlew clean build -x test` - Builds the application (skips tests for faster builds)

**Stage 2 - Runtime Stage:**
- `FROM eclipse-temurin:21-jre` - Use Eclipse Temurin JRE 21 (smaller, runtime-only)
- Copies only the compiled JAR from build stage
- Creates non-root user `spring` for security
- Exposes port 8082
- Sets `SPRING_PROFILES_ACTIVE=docker` to use Docker-specific configuration
- Includes health check for container orchestration

**Why Eclipse Temurin?**
- ✅ Official OpenJDK builds from Eclipse Foundation
- ✅ Includes all necessary utilities (xargs, etc.)
- ✅ Well-maintained and secure
- ✅ Replaces deprecated openjdk images
- ✅ Fully compatible with Gradle builds

**Why Multistage Build?**
- ✅ Smaller final image (JRE vs JDK: ~300MB vs ~600MB)
- ✅ Better security (no build tools in production)
- ✅ Faster deployments
- ✅ Clear separation between build and runtime

---

## Step 2: Build Docker Image

### 2.1 Build Image Using IntelliJ Docker Plugin

**Steps:**
1. Right-click on `Dockerfile`
2. Select **Modify Run Configuration...**
3. **Image tag:** `order-service:latest`
4. Click **OK**
5. Click **Run**

### 2.2 Build Image Using Command Line

**Navigate to order-service Directory:**
```bash
cd order-service
```

**Build Command:**
```bash
docker build -t order-service:latest .
```

**Expected Output:**
```
[+] Building 45.2s (18/18) FINISHED

Step 1/17 : FROM eclipse-temurin:21-jdk AS builder
...
Step 17/17 : ENTRYPOINT ["java", "-jar", "app.jar"]

Successfully built 9a8b7c6d5e4f
Successfully tagged order-service:latest
```

**Note:** First build will take longer (2-5 minutes) as Docker downloads base images. Subsequent builds are faster due to caching.

### 2.3 Verify Image Created

**List Docker Images:**
```bash
docker images | grep order-service
```

**Expected Output:**
```
REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
order-service     latest    9a8b7c6d5e4f   2 minutes ago    350MB
```

**Image Size Notes:**
- Expected size with Eclipse Temurin JRE: ~300-400MB
- If using JDK instead of JRE: ~600-800MB
- Multistage build reduces final size by 50-70%

---

## Step 3: Test Docker Image

### 3.1 Run Container

**Important:** Stop local order-service application first

**Run Container:**
```bash
docker run -d \
  --name order-service-test \
  -p 8082:8082 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://host.docker.internal:5433/order-service \
  -e SPRING_DATASOURCE_USERNAME=admin \
  -e SPRING_DATASOURCE_PASSWORD=password \
  order-service:latest
```

### 3.2 Test Endpoint

**Health Check:**
```bash
curl http://localhost:8082/actuator/health
```

**Expected:**
```json
{"status":"UP"}
```

### 3.3 Clean Up Test Container

```bash
docker stop order-service-test
docker rm order-service-test
```

---

## Part 2: Create Database Initialization Script (Optional)

## Step 4: Create PostgreSQL Initialization Script

### 4.1 Why Initialize Database?

**Two Approaches:**

**Approach 1 (Current):** Use `POSTGRES_DB` environment variable
- PostgreSQL automatically creates database on first run
- Simple, no script needed
- Used in our docker-compose.yml

**Approach 2 (Alternative):** Use init.sql script
- More control over database setup
- Can add initial tables, users, or data
- Useful for complex initialization

### 4.2 Create init.sql Script (Optional)

**Create Directory Structure:**

```bash
cd microservices-parent
mkdir -p init/postgres/docker-entrypoint-initdb.d
```

**Create File:**

**Location:** `microservices-parent/init/postgres/docker-entrypoint-initdb.d/init.sql`

**Content:**

```sql
-- ============================================
-- Order Service Database Initialization
-- ============================================

-- Check if database exists, create if not
SELECT 'CREATE DATABASE "order-service"'
WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'order-service')\gexec

-- Connect to database
\c order-service

-- Create extension for UUID support (optional)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE "order-service" TO admin;

-- Log successful initialization
DO $$
BEGIN
  RAISE NOTICE 'Database order-service initialized successfully';
END $$;
```

### 4.3 Update docker-compose.yml to Use Init Script (Optional)

**If using init script, modify postgres service:**

```yaml
postgres:
  image: postgres:latest
  container_name: postgres
  ports:
    - "5433:5432"
  environment:
    POSTGRES_USER: admin
    POSTGRES_PASSWORD: password
    # Remove POSTGRES_DB if using init script
  volumes:
    - postgres-data:/var/lib/postgresql/data
    - ./init/postgres/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d  # Add this line
  networks:
    - microservices-network
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U admin"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**Note:** For this lab, we use `POSTGRES_DB` environment variable (simpler). The init script approach is shown for reference.

---

## Part 3: Update docker-compose.yml

## Step 5: Update docker-compose.yml

### 5.1 Locate File

**Location:** `microservices-parent/docker-compose.yml`

### 5.2 Complete Updated docker-compose.yml

**IMPORTANT:** Ensure both `product-service/Dockerfile` and `order-service/Dockerfile` use Eclipse Temurin base images (`eclipse-temurin:21-jdk` and `eclipse-temurin:21-jre`). Docker Compose builds these images using the respective Dockerfiles.

**Replace entire file with:**

```yaml
version: '3.8'

services:

  # ============================================
  # MongoDB Database (for product-service)
  # ============================================
  mongodb:
    image: mongo:latest
    container_name: mongodb
    ports:
      - "27018:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - mongodb-data:/data/db
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================
  # Mongo Express (MongoDB GUI)
  # ============================================
  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_SERVER: mongodb
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: password
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - microservices-network

  # ============================================
  # PostgreSQL Database (for order-service)
  # ============================================
  postgres:
    image: postgres:latest
    container_name: postgres
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: order-service
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d order-service"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================
  # pgAdmin (PostgreSQL GUI)
  # ============================================
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    ports:
      - "8888:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    volumes:
      - pgadmin-data:/var/lib/pgadmin
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - microservices-network

  # ============================================
  # Product Service (Spring Boot - MongoDB)
  # ============================================
  product-service:
    build:
      context: ./product-service
      dockerfile: Dockerfile
    image: product-service:latest
    container_name: product-service
    ports:
      - "8084:8084"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATA_MONGODB_URI: mongodb://admin:password@mongodb:27017/product-service?authSource=admin
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - microservices-network

  # ============================================
  # Order Service (Spring Boot - PostgreSQL)
  # ============================================
  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    image: order-service:latest
    container_name: order-service
    ports:
      - "8082:8082"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/order-service
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - microservices-network

# ============================================
# Networks
# ============================================
networks:
  microservices-network:
    driver: bridge

# ============================================
# Volumes
# ============================================
volumes:
  mongodb-data:
  postgres-data:
  pgadmin-data:
```

---

## Step 6: Deploy Complete Architecture

### 6.1 Stop Any Running Containers

```bash
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
```

### 6.2 Build and Start All Services

**Navigate to parent directory:**
```bash
cd microservices-parent
```

**Build and start services:**
```bash
docker-compose up --build -d
```

**Expected Output:**
```
[+] Building 45.3s (36/36) FINISHED
[+] Running 6/6
 ✔ Container mongodb           Started
 ✔ Container postgres          Started
 ✔ Container mongo-express     Started
 ✔ Container pgadmin           Started
 ✔ Container product-service   Started
 ✔ Container order-service     Started
```

### 6.3 Verify All Containers Running

```bash
docker-compose ps
```

**Expected Output:**
```
NAME               IMAGE                      STATUS         PORTS
mongodb            mongo:latest               Up 1 minute    0.0.0.0:27018->27017/tcp
mongo-express      mongo-express:latest       Up 1 minute    0.0.0.0:8081->8081/tcp
pgadmin            dpage/pgadmin4:latest      Up 1 minute    0.0.0.0:8888->80/tcp
postgres           postgres:latest            Up 1 minute    0.0.0.0:5433->5432/tcp
order-service      order-service:latest       Up 30 seconds  0.0.0.0:8082->8082/tcp
product-service    product-service:latest     Up 30 seconds  0.0.0.0:8084->8084/tcp
```

### 6.4 View Logs

**View all logs:**
```bash
docker-compose logs -f
```

**View specific service logs:**
```bash
docker-compose logs -f order-service
```

**Expected:**
```
order-service  | Started OrderServiceApplication in 12.456 seconds
```

**Exit logs:** Press `Ctrl+C`

---

## Step 7: Test Complete System

### 7.1 Test Health Endpoints

**Order Service:**
```bash
curl http://localhost:8082/actuator/health
```

**Expected:**
```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL"
      }
    }
  }
}
```

**Product Service:**
```bash
curl http://localhost:8084/actuator/health
```

### 7.2 Test Order Service API

**Place Order:**
```bash
curl -X POST http://localhost:8082/api/order \
  -H "Content-Type: application/json" \
  -d '{
    "orderLineItemDtoList": [
      {
        "skuCode": "docker_test_product",
        "price": 199.99,
        "quantity": 3
      }
    ]
  }'
```

**Expected:**
```
Order Placed Successfully
```

### 7.3 Test Product Service API

**Create Product:**
```bash
curl -X POST http://localhost:8084/api/product \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Docker Test Product",
    "description": "Testing from Docker deployment",
    "price": 199.99
  }'
```

**Get All Products:**
```bash
curl http://localhost:8084/api/product
```

### 7.4 Access Web Interfaces

**pgAdmin:**
```
URL: http://localhost:8888
Email: admin@admin.com
Password: admin
```

**Register PostgreSQL Server:**
- Name: `order-service-docker`
- Host: `postgres`
- Port: `5432`
- Database: `order-service`
- Username: `admin`
- Password: `password`

**Mongo Express:**
```
URL: http://localhost:8081
Username: admin
Password: password
```

---

## Step 8: Verify Data Persistence

### 8.1 Stop All Services

```bash
docker-compose down
```

**Note:** Volumes are NOT deleted

### 8.2 Start Services Again

```bash
docker-compose up -d
```

### 8.3 Verify Data Still Exists

**Check orders in PostgreSQL:**
```bash
docker exec -it postgres psql -U admin -d order-service -c "SELECT * FROM orders;"
```

**Expected:** Orders from previous test still exist

---

## Step 9: Clean Shutdown and Cleanup

### 9.1 Stop Services (Keep Data)

```bash
docker-compose down
```
- Stops containers
- Removes containers
- **Keeps volumes (data persists)**

### 9.2 Remove Everything (Including Data)

**WARNING:** This deletes all data!

```bash
docker-compose down -v
```
- **Removes volumes (data deleted)**

---

## Docker Compose Commands Reference

### Starting Services

```bash
# Start services (use existing images)
docker-compose up -d

# Build images and start services
docker-compose up --build -d
```

### Stopping Services

```bash
# Stop services (containers removed)
docker-compose down

# Stop services (keep containers)
docker-compose stop
```

### Viewing Status

```bash
# List running services
docker-compose ps

# View logs (all services)
docker-compose logs -f

# View logs (specific service)
docker-compose logs -f order-service
```

### Restarting Services

```bash
# Restart all services
docker-compose restart

# Restart specific service
docker-compose restart order-service
```

---

## Troubleshooting

### Issue 1: Container Won't Start

**Check Logs:**
```bash
docker-compose logs order-service
```

**Common Errors:**

**Database Connection Refused:**
```
Connection refused: postgres:5432
```
**Solution:** Wait for PostgreSQL to be healthy

**Port Already in Use:**
```
Bind for 0.0.0.0:8082 failed
```
**Solution:** Stop conflicting container

### Issue 2: Cannot Connect Between Services

**Verify Network:**
```bash
docker network inspect microservices-parent_microservices-network
```

**Verify Service Name:**
- Internal: Use service name `postgres`
- Internal port: Use `5432` (not `5433`)

### Issue 3: Data Not Persisting

**Don't Use `-v` Flag:**
```bash
docker-compose down     # Good - keeps volumes
docker-compose down -v  # Bad - deletes volumes
```

### Issue 4: "xargs is not available" Build Error

**Error:**
```
xargs is not available
ERROR: failed to build: process "/bin/sh -c ./gradlew clean build -x test" did not complete successfully: exit code: 1
```

**Cause:** Using `openjdk` base images which lack required utilities

**Solution:**
Replace in Dockerfile:
- Line 4: Change `FROM openjdk:21-jdk AS builder` to `FROM eclipse-temurin:21-jdk AS builder`
- Line 27: Change `FROM openjdk:21-jre-slim` to `FROM eclipse-temurin:21-jre`

Eclipse Temurin images include all necessary build utilities that Gradle requires.

---

## Summary

### What You Completed:

**Dockerfile Creation:**
- ✅ Created multistage Dockerfile for order-service
- ✅ Build stage with full JDK
- ✅ Runtime stage with slim JRE
- ✅ Non-root user for security
- ✅ Health check configuration

**Docker Image:**
- ✅ Built order-service:latest image using Eclipse Temurin
- ✅ Verified image size (~300-400MB with multistage build)
- ✅ Tested image locally

**docker-compose.yml:**
- ✅ Added order-service configuration
- ✅ Added PostgreSQL database
- ✅ Added pgAdmin GUI
- ✅ Configured service dependencies
- ✅ Setup health checks
- ✅ Configured volumes for persistence

**Complete Architecture:**
- ✅ 6 services running
- ✅ 2 databases (PostgreSQL, MongoDB)
- ✅ 2 microservices (order-service, product-service)
- ✅ 2 GUI tools (pgAdmin, Mongo Express)

**Testing and Verification:**
- ✅ Health endpoints responding
- ✅ REST APIs functional
- ✅ Data persisting in databases
- ✅ Web interfaces accessible
- ✅ Data surviving container restarts

### Final Architecture:

```
Docker Network: microservices-network
├── order-service (Spring Boot)
│   └── PostgreSQL :8082
├── product-service (Spring Boot)
│   └── MongoDB :8084
├── postgres :5433
├── mongodb :27018
├── pgadmin :8888
└── mongo-express :8081
```

### Files Created/Modified:

**Created:**
- ✅ `order-service/Dockerfile`

**Modified:**
- ✅ `docker-compose.yml`

---

## Congratulations!

You have successfully completed the lab and deployed a complete microservices architecture with:

✅ **order-service microservice** with Spring Data JPA and PostgreSQL

✅ **JPA entities with relationships** (@OneToMany, cascade operations)

✅ **Complete layered architecture** (Controller, Service, Repository, DTO)

✅ **PostgreSQL database** with local and Docker configurations

✅ **Integration tests** with TestContainers

✅ **Docker deployment** with multistage builds

✅ **docker-compose orchestration** of all services

---

Continue to next week's material for advanced microservices patterns.

---

**🎉 Lab Complete! 🎉**
