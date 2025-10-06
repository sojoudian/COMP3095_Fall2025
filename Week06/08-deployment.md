# Deployment - Docker and Docker Compose

## Overview

In this final section, you will containerize the order-service and deploy the complete microservices architecture using Docker Compose. You'll create a multistage Dockerfile, update docker-compose.yml to orchestrate all services, and verify the entire system works together.

---

## Understanding the Complete Architecture

### What We're Building

**Complete System:**
```
Docker Network: microservices-network
├── order-service (Spring Boot app)
│   └── Connects to: postgres:5432
├── product-service (Spring Boot app)
│   └── Connects to: mongodb:27017
├── postgres (PostgreSQL database)
│   └── Ports: 5432 (internal), 5433 (external)
├── mongodb (MongoDB database)
│   └── Ports: 27017 (internal), 27018 (external)
├── mongo-express (MongoDB GUI)
│   └── Ports: 8081 (external)
└── pgadmin (PostgreSQL GUI)
    └── Ports: 5050 (external)
```

**Service Communication:**
```
External Access (from your browser/Postman):
- order-service:    http://localhost:8082
- product-service:  http://localhost:8084
- pgAdmin:          http://localhost:5050
- mongo-express:    http://localhost:8081

Internal Communication (between containers):
- order-service → postgres:5432
- product-service → mongodb:27017
- pgadmin → postgres:5432
- mongo-express → mongodb:27017
```

---

## Part 1: Create Dockerfile for order-service

### Understanding Multistage Dockerfile

**Why Multistage Build?**

**Single Stage (Not Recommended):**
```
Base Image: openjdk:21-jdk (800MB)
+ Build tools (Gradle, source code)
+ Runtime application
= Final Image: ~1.2GB
```

**Multistage Build (Recommended):**
```
Stage 1 (Build): openjdk:21-jdk (800MB)
- Compiles source code
- Creates JAR file
- Discarded after build

Stage 2 (Runtime): openjdk:21-jre-slim (200MB)
+ Only JAR file
+ No build tools
= Final Image: ~400MB
```

**Benefits:**
- ✅ 60-70% smaller image size
- ✅ Faster deployment
- ✅ Reduced attack surface (no build tools in production)
- ✅ Security best practices

---

## Step 1: Create Dockerfile

### 1.1 Create File

**Location:** `order-service/Dockerfile`

**Steps:**
1. Right-click on `order-service` folder in IntelliJ
2. Select **New → File**
3. Name: `Dockerfile` (exact spelling, no extension)
4. Click **OK**

### 1.2 Write Dockerfile

**Complete Dockerfile:**

```dockerfile
# ============================================
# Stage 1: Build Stage
# ============================================
FROM openjdk:21-jdk AS builder

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
FROM openjdk:21-jre-slim

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

# Set environment variables (can be overridden at runtime)
ENV SPRING_PROFILES_ACTIVE=docker

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8082/actuator/health || exit 1

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 1.3 Understanding Each Section

#### **Stage 1: Build Stage**

```dockerfile
FROM openjdk:21-jdk AS builder
```
- Base image with full JDK (needed for compilation)
- Named `builder` to reference later
- ~800MB image size

```dockerfile
WORKDIR /app
```
- Sets working directory to `/app`
- All subsequent commands run from this directory

```dockerfile
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .
```
- Copy Gradle wrapper and build configuration
- Allows container to build project
- `.` means copy to current directory (`/app`)

```dockerfile
COPY src src
```
- Copy source code
- Into `/app/src` directory

```dockerfile
RUN chmod +x gradlew
```
- Make Gradle wrapper executable
- Required on Unix-based systems

```dockerfile
RUN ./gradlew clean build -x test
```
- Compile source code
- Create JAR file
- `-x test` skips tests (faster build)
- JAR created at: `/app/build/libs/order-service-0.0.1-SNAPSHOT.jar`

#### **Stage 2: Runtime Stage**

```dockerfile
FROM openjdk:21-jre-slim
```
- New base image with only JRE (no compiler)
- `slim` variant = minimal dependencies
- ~200MB image size
- **Previous stage (builder) is discarded**

```dockerfile
COPY --from=builder /app/build/libs/*.jar app.jar
```
- Copy JAR from build stage to runtime stage
- `--from=builder` references previous stage
- Source: `/app/build/libs/*.jar` in builder stage
- Destination: `/app/app.jar` in runtime stage

```dockerfile
RUN addgroup --system spring && adduser --system --group spring
```
- Create non-root user named `spring`
- Security best practice (don't run as root)
- `--system` creates system user (no login shell)

```dockerfile
RUN chown -R spring:spring /app
```
- Change ownership of `/app` to `spring` user
- Allows `spring` user to read JAR file

```dockerfile
USER spring:spring
```
- Switch from root to `spring` user
- All subsequent commands run as `spring` user
- Application runs as non-root

```dockerfile
EXPOSE 8082
```
- Documents that container listens on port 8082
- **Note:** This is documentation only, doesn't actually open port
- Port mapping done in docker-compose.yml

```dockerfile
ENV SPRING_PROFILES_ACTIVE=docker
```
- Sets environment variable
- Activates `docker` Spring profile
- Tells application to use `application-docker.properties`
- Can be overridden in docker-compose.yml

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8082/actuator/health || exit 1
```
- Docker health check configuration
- `--interval=30s` - Check every 30 seconds
- `--timeout=3s` - Fail if no response in 3 seconds
- `--start-period=30s` - Wait 30s before first check (startup time)
- `--retries=3` - Retry 3 times before marking unhealthy
- Uses Actuator health endpoint

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```
- Command to run when container starts
- Starts Spring Boot application
- Array format (exec form) recommended

---

## Step 2: Build Docker Image

### 2.1 Build Image Using IntelliJ Docker Plugin

**Steps:**
1. Right-click on `Dockerfile`
2. Select **Run 'Dockerfile'**
3. Wait for build to complete

**Or Configure Run Configuration:**
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

Step 1/17 : FROM openjdk:21-jdk AS builder
 ---> a1b2c3d4e5f6
Step 2/17 : WORKDIR /app
 ---> Running in 1a2b3c4d5e6f
 ---> 2b3c4d5e6f7a
Step 3/17 : COPY gradlew .
 ---> 3c4d5e6f7a8b
...
Step 17/17 : ENTRYPOINT ["java", "-jar", "app.jar"]
 ---> 9a8b7c6d5e4f

Successfully built 9a8b7c6d5e4f
Successfully tagged order-service:latest
```

### 2.3 Verify Image Created

**List Docker Images:**
```bash
docker images | grep order-service
```

**Expected Output:**
```
REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
order-service     latest    9a8b7c6d5e4f   2 minutes ago    387MB
```

**Verify Image Layers:**
```bash
docker history order-service:latest
```

**Shows multistage build worked:**
- Base layer: openjdk:21-jre-slim
- Added layer: app.jar
- No build tools in final image

---

## Step 3: Test Docker Image Locally

### 3.1 Run Container Standalone

**Important:** Stop local order-service application first (if running)

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

**Explanation:**
- `-d` - Run in detached mode (background)
- `--name` - Container name
- `-p 8082:8082` - Port mapping
- `-e` - Environment variables override application.properties
- `host.docker.internal` - Docker's DNS name for host machine
- Port `5433` - Our PostgreSQL container's mapped port

### 3.2 Check Container Status

```bash
docker ps | grep order-service
```

**Expected:**
```
CONTAINER ID   IMAGE                    STATUS         PORTS                    NAMES
1a2b3c4d5e6f   order-service:latest     Up 10 seconds  0.0.0.0:8082->8082/tcp   order-service-test
```

### 3.3 View Container Logs

```bash
docker logs -f order-service-test
```

**Expected (Successful Startup):**
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.5.6)

Started OrderServiceApplication in 8.456 seconds
```

### 3.4 Test Endpoint

**Health Check:**
```bash
curl http://localhost:8082/actuator/health
```

**Expected:**
```json
{"status":"UP"}
```

**Place Order:**
```bash
curl -X POST http://localhost:8082/api/order \
  -H "Content-Type: application/json" \
  -d '{
    "orderLineItemDtoList": [
      {
        "skuCode": "test_product",
        "price": 99.99,
        "quantity": 1
      }
    ]
  }'
```

**Expected:**
```
Order Placed Successfully
```

### 3.5 Clean Up Test Container

```bash
docker stop order-service-test
docker rm order-service-test
```

---

## Part 2: Update docker-compose.yml

### Understanding the Update

**Current docker-compose.yml (from Week 5):**
- ✅ product-service
- ✅ mongodb
- ✅ mongo-express

**What We're Adding:**
- 🆕 order-service
- 🆕 postgres
- 🆕 pgadmin

**Total Services: 6**

---

## Step 4: Update docker-compose.yml

### 4.1 Locate File

**Location:** `microservices-parent/docker-compose.yml`

**Current Content (from Week 5):**
```yaml
services:
  mongodb:
    # ... existing mongodb config

  mongo-express:
    # ... existing mongo-express config

  product-service:
    # ... existing product-service config
```

### 4.2 Complete Updated docker-compose.yml

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
      - ./init-scripts:/docker-entrypoint-initdb.d
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
      - "5050:80"
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

### 4.3 Understanding the Configuration

#### **PostgreSQL Service**

```yaml
postgres:
  image: postgres:latest
  container_name: postgres
  ports:
    - "5433:5432"
```
- Uses official PostgreSQL image
- Names container `postgres`
- Maps host port 5433 to container port 5432

```yaml
environment:
  POSTGRES_DB: order-service
  POSTGRES_USER: admin
  POSTGRES_PASSWORD: password
```
- Auto-creates database `order-service`
- Creates user `admin` with password `password`
- Must match application configuration

```yaml
volumes:
  - postgres-data:/var/lib/postgresql/data
  - ./init-scripts:/docker-entrypoint-initdb.d
```
- `postgres-data` volume - Persists database data
- `./init-scripts` - Optional initialization SQL scripts
- Scripts in `docker-entrypoint-initdb.d` run on first startup

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U admin -d order-service"]
  interval: 10s
  timeout: 5s
  retries: 5
```
- Health check using `pg_isready` command
- Checks every 10 seconds
- Retries 5 times before marking unhealthy
- Other services wait for this before starting

#### **pgAdmin Service**

```yaml
pgadmin:
  image: dpage/pgadmin4:latest
  container_name: pgadmin
  ports:
    - "5050:80"
```
- Web interface on port 5050
- Container runs on port 80 internally

```yaml
environment:
  PGADMIN_DEFAULT_EMAIL: admin@admin.com
  PGADMIN_DEFAULT_PASSWORD: admin
```
- Login credentials for web interface

```yaml
depends_on:
  postgres:
    condition: service_healthy
```
- Waits for PostgreSQL to be healthy before starting
- Ensures database is ready when pgAdmin starts

#### **Order Service**

```yaml
order-service:
  build:
    context: ./order-service
    dockerfile: Dockerfile
  image: order-service:latest
```
- Builds from local Dockerfile
- Tags image as `order-service:latest`
- `context` is the build context (directory)

```yaml
ports:
  - "8082:8082"
```
- Exposes REST API on port 8082

```yaml
environment:
  SPRING_PROFILES_ACTIVE: docker
  SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/order-service
  SPRING_DATASOURCE_USERNAME: admin
  SPRING_DATASOURCE_PASSWORD: password
  SPRING_JPA_HIBERNATE_DDL_AUTO: update
```
- Activates `docker` profile
- Overrides datasource configuration
- Uses service name `postgres` (Docker DNS)
- Uses internal port 5432 (not 5433!)

```yaml
depends_on:
  postgres:
    condition: service_healthy
```
- Waits for PostgreSQL to be ready
- Prevents connection errors on startup

#### **Networks**

```yaml
networks:
  microservices-network:
    driver: bridge
```
- All services on same network
- Enables service-to-service communication
- DNS resolution by service name

#### **Volumes**

```yaml
volumes:
  mongodb-data:
  postgres-data:
  pgadmin-data:
```
- Named volumes for data persistence
- Data survives container restarts
- Data survives `docker-compose down`
- Deleted only with `docker-compose down -v`

---

## Step 5: Create Initialization Scripts (Optional)

### 5.1 Create init-scripts Directory

**Location:** `microservices-parent/init-scripts/`

```bash
cd microservices-parent
mkdir init-scripts
```

### 5.2 Create Initialization SQL Script

**File:** `init-scripts/01-init.sql`

**Content:**
```sql
-- ============================================
-- Order Service Database Initialization
-- ============================================

-- This script runs automatically on first PostgreSQL startup
-- Only executes if database is being created for the first time

-- Verify database exists
SELECT 'Database order-service is ready!' AS message;

-- Create indexes for better query performance (optional)
-- Note: JPA already creates tables, so we don't need CREATE TABLE here

-- Index on order_number for faster lookups
-- This will be created after JPA creates the tables
-- For now, this is a placeholder for future optimizations

-- You can add any custom SQL initialization here:
-- - Create additional indexes
-- - Insert seed data
-- - Create stored procedures
-- - Grant permissions

-- Example: Insert seed data (uncomment if needed)
-- Note: This will fail on first run because JPA hasn't created tables yet
-- These statements are examples only

/*
INSERT INTO orders (order_number) VALUES ('SEED-ORDER-001');
INSERT INTO order_line_items (sku_code, price, quantity, order_id)
VALUES ('seed_product', 9.99, 1, 1);
*/

-- Log completion
SELECT 'Initialization script completed!' AS message;
```

**Note:** This script is optional. JPA will create tables automatically. Use init scripts for:
- Seeding initial data
- Creating indexes
- Setting up permissions
- Creating stored procedures

---

## Step 6: Deploy Complete Architecture

### 6.1 Stop Any Running Containers

**Stop individual containers:**
```bash
docker stop order-service-test postgres-container pgadmin-container
docker rm order-service-test postgres-container pgadmin-container
```

**Or stop everything:**
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

**Explanation:**
- `up` - Start services
- `--build` - Rebuild images before starting
- `-d` - Detached mode (run in background)

**Expected Output:**
```
[+] Building 45.3s (36/36) FINISHED
[+] Running 6/6
 ✔ Container mongodb           Started  2.1s
 ✔ Container postgres          Started  2.3s
 ✔ Container mongo-express     Started  4.5s
 ✔ Container pgadmin           Started  4.7s
 ✔ Container product-service   Started  6.8s
 ✔ Container order-service     Started  7.2s
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
pgadmin            dpage/pgadmin4:latest      Up 1 minute    0.0.0.0:5050->80/tcp
postgres           postgres:latest            Up 1 minute    0.0.0.0:5433->5432/tcp
order-service      order-service:latest       Up 30 seconds  0.0.0.0:8082->8082/tcp
product-service    product-service:latest     Up 30 seconds  0.0.0.0:8084->8084/tcp
```

**All STATUS should show "Up"**

### 6.4 Check Service Health

**View health status:**
```bash
docker-compose ps
```

**Look for "healthy" in STATUS:**
```
postgres    Up 2 minutes (healthy)
mongodb     Up 2 minutes (healthy)
```

### 6.5 View Logs

**View all logs:**
```bash
docker-compose logs -f
```

**View specific service logs:**
```bash
docker-compose logs -f order-service
```

**Expected (order-service):**
```
order-service  | Started OrderServiceApplication in 12.456 seconds
```

**Expected (product-service):**
```
product-service | Started ProductServiceApplication in 10.234 seconds
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

**Expected:**
```json
{
  "status": "UP",
  "components": {
    "mongo": {
      "status": "UP"
    }
  }
}
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
URL: http://localhost:5050
Email: admin@admin.com
Password: admin
```

**Register PostgreSQL Server:**
- Name: `order-service-docker`
- Host: `postgres` (service name)
- Port: `5432` (internal port)
- Database: `order-service`
- Username: `admin`
- Password: `password`

**Mongo Express:**
```
URL: http://localhost:8081
Username: admin
Password: password
```

**Navigate to:**
- Databases → product-service → products collection
- View documents

---

## Step 8: Verify Data Persistence

### 8.1 Stop All Services

```bash
docker-compose down
```

**Expected:**
```
[+] Running 6/6
 ✔ Container order-service     Removed
 ✔ Container product-service   Removed
 ✔ Container pgadmin           Removed
 ✔ Container mongo-express     Removed
 ✔ Container postgres          Removed
 ✔ Container mongodb           Removed
```

**Note:** Volumes are NOT deleted (data persists)

### 8.2 Start Services Again

```bash
docker-compose up -d
```

### 8.3 Verify Data Still Exists

**Check orders in PostgreSQL:**
```bash
docker exec -it postgres psql -U admin -d order-service -c "SELECT * FROM orders;"
```

**Expected:**
- Orders from previous test still exist
- Data survived container restart

**Check products in MongoDB:**
```bash
docker exec -it mongodb mongosh -u admin -p password --authenticationDatabase admin product-service --eval "db.products.find()"
```

**Expected:**
- Products from previous test still exist

---

## Step 9: Clean Shutdown and Cleanup

### 9.1 Stop Services (Keep Data)

```bash
docker-compose down
```
- Stops containers
- Removes containers
- Removes networks
- **Keeps volumes (data persists)**

### 9.2 Remove Everything (Including Data)

**WARNING:** This deletes all data!

```bash
docker-compose down -v
```
- Stops containers
- Removes containers
- Removes networks
- **Removes volumes (data deleted)**

### 9.3 Clean Up Images

**Remove unused images:**
```bash
docker image prune -a
```

**Remove specific images:**
```bash
docker rmi order-service:latest
docker rmi product-service:latest
```

---

## Docker Compose Commands Reference

### Starting Services

```bash
# Start services (use existing images)
docker-compose up -d

# Build images and start services
docker-compose up --build -d

# Start specific service
docker-compose up -d order-service
```

### Stopping Services

```bash
# Stop services (containers removed)
docker-compose down

# Stop services (keep containers)
docker-compose stop

# Stop specific service
docker-compose stop order-service
```

### Viewing Status

```bash
# List running services
docker-compose ps

# View logs (all services)
docker-compose logs -f

# View logs (specific service)
docker-compose logs -f order-service

# View last 100 lines
docker-compose logs --tail=100 order-service
```

### Restarting Services

```bash
# Restart all services
docker-compose restart

# Restart specific service
docker-compose restart order-service
```

### Rebuilding Services

```bash
# Rebuild specific service
docker-compose build order-service

# Rebuild and restart
docker-compose up --build -d order-service
```

### Executing Commands in Containers

```bash
# Execute command in running container
docker-compose exec order-service bash

# Execute SQL query
docker-compose exec postgres psql -U admin -d order-service
```

### Scaling Services

```bash
# Run multiple instances (if stateless)
docker-compose up -d --scale order-service=3
```

---

## Troubleshooting

### Issue 1: Container Won't Start

**Symptom:**
```bash
docker-compose ps
# Shows: Exited (1)
```

**Solution:**

**A. Check Logs:**
```bash
docker-compose logs order-service
```

**B. Common Errors:**

**Database Connection Refused:**
```
Connection refused: postgres:5432
```
**Fix:** Wait for PostgreSQL to be healthy:
```yaml
depends_on:
  postgres:
    condition: service_healthy
```

**Port Already in Use:**
```
Bind for 0.0.0.0:8082 failed: port is already allocated
```
**Fix:** Stop conflicting container or change port:
```yaml
ports:
  - "8083:8082"  # Use different host port
```

### Issue 2: Service Unhealthy

**Check Health Status:**
```bash
docker inspect --format='{{json .State.Health}}' order-service | jq
```

**Fix Health Check:**

**If health check fails, modify Dockerfile:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8082/actuator/health || exit 1
```
- Increased `--start-period=60s` (more startup time)

### Issue 3: Cannot Connect Between Services

**Symptom:**
```
order-service: Could not connect to postgres:5432
```

**Solution:**

**A. Verify Network:**
```bash
docker network inspect microservices-parent_microservices-network
```

**All services should be listed**

**B. Verify Service Name:**
```yaml
environment:
  SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/order-service
  # Must use service name "postgres", not "localhost"
```

**C. Verify Port:**
- Internal communication: Use container port (5432)
- External access: Use mapped port (5433)

### Issue 4: Docker Compose Build Fails

**Error:**
```
failed to solve: failed to read dockerfile
```

**Solution:**

**A. Verify Dockerfile Location:**
```yaml
build:
  context: ./order-service
  dockerfile: Dockerfile  # Must exist in ./order-service/
```

**B. Check Dockerfile Syntax:**
```bash
cd order-service
docker build -t test .
# Tests Dockerfile without docker-compose
```

### Issue 5: Data Not Persisting

**Symptom:**
- Data disappears after `docker-compose down`

**Solution:**

**A. Verify Volumes Defined:**
```yaml
volumes:
  postgres-data:
```

**B. Verify Volume Mounted:**
```yaml
postgres:
  volumes:
    - postgres-data:/var/lib/postgresql/data
```

**C. Don't Use `-v` Flag:**
```bash
docker-compose down     # Good - keeps volumes
docker-compose down -v  # Bad - deletes volumes
```

### Issue 6: Out of Memory or CPU

**Symptom:**
- Slow builds
- Containers crashing
- High CPU usage

**Solution:**

**A. Increase Docker Resources:**
- Docker Desktop → Settings → Resources
- Memory: 6GB+
- CPUs: 4+

**B. Build Without Cache:**
```bash
docker-compose build --no-cache
```

**C. Prune Unused Resources:**
```bash
docker system prune -a
```

---

## Performance Optimization

### 1. Layer Caching

**Optimize Dockerfile for faster builds:**

```dockerfile
# Copy dependencies first (changes less frequently)
COPY build.gradle.kts .
COPY settings.gradle.kts .

# Download dependencies (cached if build.gradle unchanged)
RUN ./gradlew dependencies

# Copy source code last (changes most frequently)
COPY src src
```

### 2. Multi-Stage Build

**Already implemented in our Dockerfile:**
- Build stage: Full JDK
- Runtime stage: Slim JRE
- Result: 60% smaller image

### 3. Health Checks

**Prevent premature traffic:**
```dockerfile
HEALTHCHECK --start-period=30s
```
- Allows application time to fully start
- No health checks during startup period

### 4. Resource Limits

**Add to docker-compose.yml (optional):**
```yaml
order-service:
  deploy:
    resources:
      limits:
        cpus: '1.0'
        memory: 1G
      reservations:
        cpus: '0.5'
        memory: 512M
```

---

## Production Considerations

### Security Best Practices

**1. Don't Commit Secrets:**
```yaml
# Bad - hardcoded credentials
environment:
  POSTGRES_PASSWORD: password

# Good - use environment variables
environment:
  POSTGRES_PASSWORD: ${DB_PASSWORD}
```

**Use .env file:**
```bash
# .env (add to .gitignore)
DB_PASSWORD=super_secure_password
```

**2. Non-Root User:**
- ✅ Already implemented in Dockerfile
- Application runs as `spring` user

**3. Minimal Base Image:**
- ✅ Using `openjdk:21-jre-slim`
- Smaller attack surface

**4. Health Checks:**
- ✅ Implemented in Dockerfile
- Enables automatic restart on failure

### Monitoring and Logging

**1. Centralized Logging:**
```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

**2. Prometheus Metrics:**
- Spring Boot Actuator exposes metrics
- Add Prometheus scraping in production

### High Availability

**1. Multiple Instances:**
```bash
docker-compose up -d --scale order-service=3
```

**2. Load Balancer:**
- Add nginx or traefik for load balancing
- Distribute requests across instances

**3. Database Replication:**
- Primary-replica setup for PostgreSQL
- Read replicas for MongoDB

---

## Summary

### What You Completed:

**Dockerfile Creation:**
- ✅ Created multistage Dockerfile for order-service
- ✅ Build stage with full JDK (compilation)
- ✅ Runtime stage with slim JRE (production)
- ✅ Non-root user for security
- ✅ Health check configuration
- ✅ Optimized image size (~400MB)

**Docker Image:**
- ✅ Built order-service:latest image
- ✅ Verified image layers and size
- ✅ Tested image locally

**docker-compose.yml:**
- ✅ Added order-service configuration
- ✅ Added PostgreSQL database
- ✅ Added pgAdmin GUI
- ✅ Configured service dependencies
- ✅ Setup health checks
- ✅ Configured volumes for persistence
- ✅ Integrated with existing services

**Complete Architecture:**
- ✅ 6 services running
- ✅ 2 databases (PostgreSQL, MongoDB)
- ✅ 2 microservices (order-service, product-service)
- ✅ 2 GUI tools (pgAdmin, Mongo Express)
- ✅ All services communicating on same network

**Testing and Verification:**
- ✅ Health endpoints responding
- ✅ REST APIs functional
- ✅ Data persisting in databases
- ✅ Web interfaces accessible
- ✅ Service-to-service communication working
- ✅ Data surviving container restarts

### Final Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Host Machine                       │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │         Docker Network: microservices-network         │  │
│  │                                                         │  │
│  │  ┌──────────────┐    ┌──────────────┐                │  │
│  │  │   MongoDB    │    │  PostgreSQL  │                │  │
│  │  │   :27017     │    │    :5432     │                │  │
│  │  └──────────────┘    └──────────────┘                │  │
│  │         ↑                    ↑                         │  │
│  │         │                    │                         │  │
│  │  ┌──────────────┐    ┌──────────────┐                │  │
│  │  │   product-   │    │    order-    │                │  │
│  │  │   service    │    │   service    │                │  │
│  │  │    :8084     │    │    :8082     │                │  │
│  │  └──────────────┘    └──────────────┘                │  │
│  │         ↑                    ↑                         │  │
│  │         │                    │                         │  │
│  │  ┌──────────────┐    ┌──────────────┐                │  │
│  │  │   mongo-     │    │   pgAdmin    │                │  │
│  │  │   express    │    │    :5050     │                │  │
│  │  │    :8081     │    │              │                │  │
│  │  └──────────────┘    └──────────────┘                │  │
│  │                                                         │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                               │
│  External Access:                                            │
│  • order-service:    http://localhost:8082                  │
│  • product-service:  http://localhost:8084                  │
│  • pgAdmin:          http://localhost:5050                  │
│  • mongo-express:    http://localhost:8081                  │
│  • PostgreSQL:       localhost:5433                         │
│  • MongoDB:          localhost:27018                        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Key Files Created/Modified:

**Created:**
- ✅ `order-service/Dockerfile` - Multistage build configuration

**Modified:**
- ✅ `docker-compose.yml` - Added order-service, postgres, pgadmin

**Optional:**
- ✅ `init-scripts/01-init.sql` - PostgreSQL initialization script

### Metrics:

**Image Sizes:**
```
order-service:latest      387 MB
product-service:latest    350 MB
postgres:latest           379 MB
mongo:latest              698 MB
```

**Container Startup Times:**
```
PostgreSQL:     ~5 seconds
MongoDB:        ~3 seconds
order-service:  ~12 seconds
product-service: ~10 seconds
```

**Resource Usage (All Services):**
```
CPU:    ~15-20%
Memory: ~2.5 GB
Disk:   ~2.5 GB (including volumes)
```

---

## Congratulations!

You have successfully:

✅ **Created order-service microservice** with Spring Data JPA and PostgreSQL

✅ **Implemented JPA entities** with relationships (@OneToMany, cascade operations)

✅ **Built complete layered architecture** (Controller, Service, Repository, Entity, DTO)

✅ **Configured PostgreSQL database** with local and Docker profiles

✅ **Setup Docker containers** for PostgreSQL and pgAdmin

✅ **Tested with Postman** and verified data persistence

✅ **Wrote integration tests** using TestContainers

✅ **Containerized the application** with multistage Dockerfile

✅ **Deployed complete architecture** with docker-compose

✅ **Verified the system** end-to-end

---

## Next Steps (Beyond This Lab)

**Week 7 and Beyond:**
- Service-to-service communication (REST, gRPC, message queues)
- API Gateway (Spring Cloud Gateway)
- Service discovery (Eureka, Consul)
- Configuration server (Spring Cloud Config)
- Distributed tracing (Zipkin, Jaeger)
- Circuit breakers (Resilience4j)
- Kubernetes deployment
- CI/CD pipelines

---

**🎉 Lab Complete! You now have a fully functional microservices architecture with two databases, two services, and complete Docker deployment! 🎉**
