# Docker Containerization Guide - COMP 3095
## Complete Step-by-Step Implementation of Docker and Docker Compose

> **Note**: This guide builds upon the Product Service implementation from Weeks 2-3 and testing from Week 4. Ensure you have completed those labs before proceeding.

### Table of Contents
1. [Lab Overview](#lab-overview)
2. [Prerequisites](#prerequisites)
3. [Understanding Docker Concepts](#understanding-docker-concepts)
4. [Part 1: Create Multistage Dockerfile](#part-1-create-multistage-dockerfile)
   - [Step 1: Understanding Multistage Builds](#step-1-understanding-multistage-builds)
   - [Step 2: Create the Dockerfile](#step-2-create-the-dockerfile)
   - [Step 3: Build the Docker Image](#step-3-build-the-docker-image)
5. [Part 2: Create Docker Compose Configuration](#part-2-create-docker-compose-configuration)
   - [Step 4: Understanding Docker Compose](#step-4-understanding-docker-compose)
   - [Step 5: Create docker-compose.yml](#step-5-create-docker-composeyml)
   - [Step 6: Create MongoDB Initialization Scripts](#step-6-create-mongodb-initialization-scripts)
6. [Part 3: Configure Application Properties](#part-3-configure-application-properties)
   - [Step 7: Update Application Properties](#step-7-update-application-properties)
   - [Step 8: Create Docker-Specific Configuration](#step-8-create-docker-specific-configuration)
7. [Part 4: Run and Test Dockerized Application](#part-4-run-and-test-dockerized-application)
   - [Step 9: Clean Existing Containers](#step-9-clean-existing-containers)
   - [Step 10: Start Docker Compose Network](#step-10-start-docker-compose-network)
   - [Step 11: Verify Container Status](#step-11-verify-container-status)
8. [Part 5: Test with Postman](#part-5-test-with-postman)
   - [Step 12: Test POST Request](#step-12-test-post-request)
   - [Step 13: Verify Data in MongoDB](#step-13-verify-data-in-mongodb)
   - [Step 14: Test GET Request](#step-14-test-get-request)
   - [Step 15: Test PUT Request](#step-15-test-put-request)
   - [Step 16: Test DELETE Request](#step-16-test-delete-request)
9. [Troubleshooting](#troubleshooting)
10. [Docker Commands Reference](#docker-commands-reference)
11. [Repository Submission](#repository-submission)
12. [Next Steps](#next-steps)

---

## Lab Overview

This lab introduces **Docker containerization** for your Product Service microservice. You will learn to package your application and its dependencies into portable containers that can run consistently across any environment.

### What You'll Build
- ✅ **Multistage Dockerfile** for optimized image building
- ✅ **Docker Compose** configuration for multi-container orchestration
- ✅ **Containerized MongoDB** with authentication and data persistence
- ✅ **Containerized Product Service** connected to MongoDB
- ✅ **Mongo Express** web UI for database management (optional)
- ✅ **Complete testing** of all CRUD operations in containerized environment

### Learning Objectives
By the end of this lab, you will be able to:
- ✅ Understand Docker images, containers, and registries
- ✅ Create multistage Dockerfiles for Java applications
- ✅ Write docker-compose.yml files for multi-container applications
- ✅ Configure MongoDB with authentication in containers
- ✅ Manage Docker networks and volumes
- ✅ Debug containerized applications
- ✅ Test REST APIs in containerized environments

### Why Docker?
**Docker** solves the "works on my machine" problem by:
- **Consistency**: Same environment in development, testing, and production
- **Portability**: Run anywhere Docker is installed
- **Isolation**: Applications don't conflict with each other
- **Efficiency**: Lightweight compared to virtual machines
- **Scalability**: Easy to scale horizontally

---

## Prerequisites

### Required Software (Should Already Be Installed)
- ✅ **Docker Desktop** (20.10 or later) - Running and available
- ✅ **IntelliJ IDEA** (2024.1 or later)
- ✅ **Java JDK 21**
- ✅ **Postman** (for API testing)
- ✅ **Git** (for version control)

### Required Completed Work
- ✅ **Week 2/3**: Product Service with MongoDB (completed)
- ✅ **Week 4**: Integration testing with TestContainers (completed)

### Verify Docker Installation
```bash
# Check Docker version
docker --version
# Should show: Docker version 20.10.x or later

# Check Docker is running
docker ps
# Should show: CONTAINER ID   IMAGE   ... (may be empty, that's ok)

# Check Docker Compose version
docker-compose --version
# Should show: Docker Compose version v2.x.x or later
```

If any commands fail, please refer to Week 1 installation guides.

---

## Understanding Docker Concepts

Before we start building, let's understand key Docker concepts:

### Docker Images vs Containers

**Docker Image:**
- A **template** or **blueprint** for creating containers
- Contains your application code, runtime, libraries, and dependencies
- **Immutable** - once created, doesn't change
- Like a **class** in object-oriented programming

**Docker Container:**
- A **running instance** of an image
- **Mutable** - can be started, stopped, restarted
- Isolated from other containers
- Like an **object/instance** in object-oriented programming

**Analogy:**
```
Image = Recipe (instructions)
Container = Cake (made from recipe)

You can make many cakes from one recipe!
```

### Multistage Builds

**Problem:** Traditional Dockerfile includes build tools in final image
- Large image size (600-800 MB)
- Security risk (build tools in production)
- Slower deployments

**Solution:** Multistage builds separate build and runtime
```
Stage 1 (Build):  JDK + Build Tools → Compile → JAR file
                         ↓
Stage 2 (Runtime): JRE only + Copy JAR → Small image
```

**Benefits:**
- ✅ Smaller images (200-300 MB vs 600-800 MB)
- ✅ Faster deployments
- ✅ Better security (no build tools in production)
- ✅ Clear separation of concerns

### Docker Compose

**Docker Compose** is a tool for defining and running **multi-container** applications.

**Why Docker Compose?**
- Our application needs **two containers**: product-service + MongoDB
- Manually starting multiple containers is tedious
- Docker Compose defines all services in **one YAML file**
- Start/stop entire application with **one command**

**What Docker Compose Manages:**
- Services (containers)
- Networks (communication between containers)
- Volumes (data persistence)
- Environment variables
- Dependencies (start order)

---

## Part 1: Create Multistage Dockerfile

### Step 1: Understanding Multistage Builds

A multistage Dockerfile has multiple `FROM` statements, each starting a new stage. Only the **final stage** determines what goes into the final image.

**Our Strategy:**
1. **Build Stage**: Use full JDK to compile Java code → Create JAR
2. **Runtime Stage**: Use lightweight JRE, copy JAR from build stage → Final image

**File Location:** `product-service/Dockerfile` (at the root of product-service, **not** in src/)

### Step 2: Create the Dockerfile

#### 2.1 Navigate to product-service Directory

In IntelliJ or your terminal:
```bash
cd /path/to/your/microservices-parent/product-service
```

#### 2.2 Create Dockerfile

1. **Right-click** on `product-service` root directory (same level as `src/`, `build.gradle.kts`)
2. Select **New → File**
3. Name it: `Dockerfile` (exact spelling, no extension)
4. Click **OK**

#### 2.3 Add Dockerfile Contents

**Complete Dockerfile with Detailed Annotations:**

```dockerfile
# ============================================
# STAGE 1: BUILD
# Purpose: Compile Java code and create JAR
# ============================================
FROM eclipse-temurin:21-jdk AS build

# Set working directory inside container
WORKDIR /app

# Copy all project files into container
# This includes: src/, build.gradle.kts, gradlew, etc.
COPY . .

# Make gradlew executable (needed on Linux containers)
RUN chmod +x gradlew

# Run Gradle build
# -x test: Skip tests (we already ran them in Week 4)
# --no-daemon: Don't start Gradle daemon (not needed in container)
RUN ./gradlew clean build -x test --no-daemon

# ============================================
# STAGE 2: RUNTIME
# Purpose: Create minimal runtime image
# ============================================
FROM eclipse-temurin:21-jre

# Set working directory
WORKDIR /app

# Create a non-root user for security
# Running as root in containers is a security risk
RUN groupadd -r spring && useradd -r -g spring spring

# Copy JAR file from build stage
# --from=build: Copy from the "build" stage
# Only copy the JAR, not the source code or build tools
COPY --from=build /app/build/libs/*.jar app.jar

# Change ownership of the JAR to the spring user
RUN chown spring:spring app.jar

# Switch to non-root user
USER spring:spring

# Document which port the application listens on
# This doesn't actually open the port (docker-compose does that)
# It's documentation for other developers
EXPOSE 8084

# Set environment variable to activate Docker profile
# This will make Spring Boot use application-docker.properties
ENV SPRING_PROFILES_ACTIVE=docker

# Define the command to run when container starts
# java -jar app.jar: Start the Spring Boot application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### 2.4 Understanding Each Instruction

Let's break down the key Dockerfile instructions:

| Instruction | Purpose | Example |
|-------------|---------|---------|
| **FROM** | Specifies base image | `FROM eclipse-temurin:21-jdk` |
| **WORKDIR** | Sets working directory | `WORKDIR /app` |
| **COPY** | Copies files into image | `COPY . .` |
| **RUN** | Executes commands during build | `RUN ./gradlew build` |
| **EXPOSE** | Documents exposed ports | `EXPOSE 8084` |
| **ENV** | Sets environment variables | `ENV SPRING_PROFILES_ACTIVE=docker` |
| **USER** | Sets user for subsequent commands | `USER spring:spring` |
| **ENTRYPOINT** | Defines container startup command | `ENTRYPOINT ["java", "-jar", "app.jar"]` |

**Why eclipse-temurin?**
- Official OpenJDK builds from Eclipse Foundation
- Well-maintained and secure
- Replaces deprecated openjdk images

**Why create a non-root user?**
- **Security best practice**: If container is compromised, attacker doesn't have root access
- **Principle of least privilege**: Application doesn't need root permissions
- **Production requirement**: Most companies enforce non-root containers

### Step 3: Build the Docker Image

#### 3.1 Open Terminal in product-service Directory

**Option A: IntelliJ Terminal**
1. In IntelliJ, open **Terminal** (bottom toolbar)
2. Navigate: `cd product-service` (if not already there)

**Option B: System Terminal**
```bash
cd /path/to/your/microservices-parent/product-service
```

#### 3.2 Verify Dockerfile Location

```bash
ls -la
# You should see: Dockerfile, build.gradle.kts, src/, gradlew
```

#### 3.3 Build the Image

**Command:**
```bash
docker build -t product-service:1.0 .
```

**Breaking Down the Command:**
- `docker build`: Docker command to build an image
- `-t product-service:1.0`: Tag (name) the image
  - `product-service`: Image name
  - `1.0`: Version tag
- `.`: Build context (current directory)

**Expected Output:**
```
[+] Building 45.2s (15/15) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [internal] load metadata for docker.io/library/eclipse-temurin:21-jdk
 => [build 1/5] FROM docker.io/library/eclipse-temurin:21-jdk
 => [internal] load build context
 => [build 2/5] WORKDIR /app
 => [build 3/5] COPY . .
 => [build 4/5] RUN chmod +x gradlew
 => [build 5/5] RUN ./gradlew clean build -x test --no-daemon
 => [stage-1 1/5] FROM docker.io/library/eclipse-temurin:21-jre
 => [stage-1 2/5] WORKDIR /app
 => [stage-1 3/5] RUN groupadd -r spring && useradd -r -g spring spring
 => [stage-1 4/5] COPY --from=build /app/build/libs/*.jar app.jar
 => [stage-1 5/5] RUN chown spring:spring app.jar
 => exporting to image
 => => naming to docker.io/library/product-service:1.0
```

**Build Time:**
- First build: 2-5 minutes (downloads base images)
- Subsequent builds: 30-60 seconds (cached layers)

#### 3.4 Verify Image Was Created

**Command:**
```bash
docker images | grep product-service
```

**Expected Output:**
```
product-service   1.0       a1b2c3d4e5f6   2 minutes ago   300MB
```

**Alternative: Docker Desktop**
1. Open **Docker Desktop**
2. Go to **Images** tab
3. You should see: `product-service:1.0`

#### 3.5 Understanding the Image Size

**Check your image size:**
```bash
docker images product-service:1.0
```

**Typical sizes:**
- **Multistage build (JRE)**: 250-350 MB ✅
- **Single stage (JDK)**: 600-800 MB ❌

**Why the difference?**
```
JDK = Java Development Kit (full)
├── Compiler (javac)
├── Debugger
├── Development tools
└── JRE (Java Runtime Environment)

JRE = Java Runtime Environment (minimal)
└── Only what's needed to RUN Java apps

Production needs JRE only!
```

**If your image is 600+ MB:**
- Check you're using `eclipse-temurin:21-jre` in Stage 2
- Verify multistage build is working correctly

---

## Part 2: Create Docker Compose Configuration

### Step 4: Understanding Docker Compose

Docker Compose uses a **YAML file** to configure services, networks, and volumes.

**Our Application Architecture:**
```
┌─────────────────────────────────────────┐
│      microservices-network (bridge)     │
│                                          │
│  ┌────────────────┐  ┌────────────────┐ │
│  │ product-service│  │    mongodb     │ │
│  │  (port 8084)   │→ │  (port 27017)  │ │
│  └────────────────┘  └────────────────┘ │
│          ↓                    ↓          │
│          └───────> mongo-express        │
│                    (port 8081 - Web UI) │
└─────────────────────────────────────────┘
              ↓
     mongodb-data (volume)
     Persists database data
```

**Why Networks?**
- Containers on same network can communicate by **service name**
- `product-service` connects to `mongodb` (not `localhost`)

**Why Volumes?**
- Containers are ephemeral (data lost when stopped)
- Volumes persist data even after container deletion
- MongoDB data stored in `mongodb-data` volume

### Step 5: Create docker-compose.yml

#### 5.1 Navigate to microservices-parent Directory

This is the **parent** directory, one level up from product-service:
```bash
cd /path/to/your/microservices-parent
```

**Verify you're in the right place:**
```bash
ls -la
# You should see: product-service/, build.gradle.kts, settings.gradle.kts
```

#### 5.2 Create docker-compose.yml

1. **Right-click** on `microservices-parent` root directory
2. Select **New → File**
3. Name it: `docker-compose.yml`
4. Click **OK**

#### 5.3 Add docker-compose.yml Contents

**⚠️ CRITICAL: YAML is indent-sensitive!**
- Use **2 spaces** for indentation (not tabs!)
- Incorrect indentation causes errors
- Copy-paste carefully

**Complete docker-compose.yml with Annotations:**

```yaml
# Docker Compose version
version: '3.8'

# Services section: Define all containers
services:

  # ==========================================
  # SERVICE 1: MongoDB Database
  # ==========================================
  mongodb:
    # Use official MongoDB image from Docker Hub
    image: mongo:latest

    # Container name (optional, but helpful for docker commands)
    container_name: mongodb

    # Port mapping: host:container
    # Access MongoDB on localhost:27017
    ports:
      - "27017:27017"

    # Environment variables for MongoDB initialization
    environment:
      # Root username and password
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
      # Database to create on first startup
      MONGO_INITDB_DATABASE: product-service

    # Volumes: Persist data and mount init scripts
    volumes:
      # Named volume: Persist MongoDB data
      - mongodb-data:/data/db
      # Bind mount: MongoDB runs scripts in this directory on first startup
      - ./init/mongo/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d

    # Connect to custom network
    networks:
      - microservices-network

    # Restart policy: Restart container unless manually stopped
    restart: unless-stopped

  # ==========================================
  # SERVICE 2: Product Service Application
  # ==========================================
  product-service:
    # Build configuration
    build:
      # Directory containing Dockerfile
      context: ./product-service
      # Dockerfile name (default is "Dockerfile")
      dockerfile: Dockerfile

    # Tag the built image
    image: product-service:1.0

    # Container name
    container_name: product-service

    # Port mapping
    ports:
      - "8084:8084"

    # Dependency: Start MongoDB before product-service
    # Note: This only controls START order, not readiness
    depends_on:
      - mongodb

    # Environment variables
    environment:
      # Activate Docker Spring profile
      SPRING_PROFILES_ACTIVE: docker

    # Connect to network
    networks:
      - microservices-network

    # Restart policy
    restart: unless-stopped

  # ==========================================
  # SERVICE 3: Mongo Express (Optional Web UI)
  # ==========================================
  mongo-express:
    # Official Mongo Express image
    image: mongo-express:latest

    # Container name
    container_name: mongo-express

    # Port mapping: Access web UI at http://localhost:8081
    ports:
      - "8081:8081"

    # Environment variables
    environment:
      # Connect to MongoDB service
      ME_CONFIG_MONGODB_SERVER: mongodb
      # Use the admin credentials from MongoDB
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      # Web UI login credentials (different from MongoDB!)
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: pass

    # Start after MongoDB
    depends_on:
      - mongodb

    # Connect to network
    networks:
      - microservices-network

    # Restart policy
    restart: unless-stopped

# ==========================================
# VOLUMES: Define named volumes
# ==========================================
volumes:
  # MongoDB data persistence
  mongodb-data:
    driver: local

# ==========================================
# NETWORKS: Define custom networks
# ==========================================
networks:
  # Bridge network for inter-container communication
  microservices-network:
    driver: bridge
```

#### 5.4 Understanding docker-compose.yml Structure

**Key Sections:**

**1. services:**
Defines all containers in your application. Each service becomes a container.

**2. volumes:**
- **Named volumes**: Managed by Docker, persists data
- **Bind mounts**: Links host directory to container directory

**3. networks:**
- Allows containers to communicate by service name
- Isolated from other Docker networks

**Important docker-compose Concepts:**

| Concept | Purpose | Example |
|---------|---------|---------|
| **image** | Use pre-built image | `image: mongo:latest` |
| **build** | Build image from Dockerfile | `build: ./product-service` |
| **ports** | Map container port to host | `"8084:8084"` |
| **environment** | Set environment variables | `SPRING_PROFILES_ACTIVE: docker` |
| **depends_on** | Control startup order | `depends_on: - mongodb` |
| **volumes** | Persist data or mount files | `mongodb-data:/data/db` |
| **networks** | Connect containers | `networks: - microservices-network` |

**Port Mapping Explained:**
```
"8084:8084"
  ↑     ↑
  |     └─ Container port (inside Docker)
  └─ Host port (your machine)

Access: http://localhost:8084 → Container port 8084
```

**Service Name Resolution:**
```
product-service container:
  spring.data.mongodb.host=mongodb ← Service name!
                            ↓
  Docker DNS resolves to mongodb container's IP
```

### Step 6: Create MongoDB Initialization Scripts

MongoDB containers automatically run JavaScript files from `/docker-entrypoint-initdb.d/` on first startup. We'll use this to create our application user and database.

#### 6.1 Create Directory Structure

In the `microservices-parent` directory, create this structure:
```
microservices-parent/
└── init/
    └── mongo/
        └── docker-entrypoint-initdb.d/
            └── mongo-init.js
```

**Commands:**
```bash
# Make sure you're in microservices-parent
cd /path/to/your/microservices-parent

# Create nested directories
mkdir -p init/mongo/docker-entrypoint-initdb.d
```

#### 6.2 Create mongo-init.js

1. **Right-click** on `docker-entrypoint-initdb.d` folder
2. Select **New → File**
3. Name it: `mongo-init.js`
4. Click **OK**

#### 6.3 Add mongo-init.js Contents

**Complete MongoDB Initialization Script:**

```javascript
// ============================================
// MongoDB Initialization Script
// Purpose: Create application user and database
// Runs automatically on first container startup
// ============================================

// Switch to the product-service database
// This is created by MONGO_INITDB_DATABASE env variable
db = db.getSiblingDB('product-service');

// Create application user with read/write permissions
db.createUser({
  user: 'admin',
  pwd: 'password',
  roles: [
    {
      // Grant read and write permissions
      role: 'readWrite',
      // On the product-service database
      db: 'product-service'
    }
  ]
});

// Create product collection (table in SQL terms)
// MongoDB creates collections automatically, but explicit creation
// allows us to set options and validate it exists
db.createCollection('product');

// Optional: Create indexes for better query performance
// Index on product name for faster searches
db.product.createIndex({ name: 1 });

// Log success message (visible in docker logs)
print('MongoDB initialization completed successfully!');
print('Database: product-service');
print('User: admin');
print('Collection: product');
```

#### 6.4 Understanding MongoDB Initialization

**What happens when MongoDB container starts for the first time:**

1. MongoDB creates root user (from MONGO_INITDB_ROOT_USERNAME)
2. MongoDB looks for `*.js` files in `/docker-entrypoint-initdb.d/`
3. Executes each script in alphabetical order
4. Our script:
   - Switches to `product-service` database
   - Creates `admin` user with `readWrite` role
   - Creates `product` collection
   - Creates index on `name` field

**Why do we need this?**
- By default, MongoDB root user can't be used by applications (security)
- We need a database-specific user for our application
- This script runs automatically, no manual setup needed

**Security Note:**
- ⚠️ Using `admin/password` is OK for development/learning
- ✅ In production, use strong passwords and environment variables
- ✅ Store credentials in secrets management system (AWS Secrets Manager, HashiCorp Vault, etc.)

#### 6.5 Verify File Structure

**Check your directory structure:**
```bash
# From microservices-parent directory
tree init
# Or use ls:
ls -R init
```

**Expected output:**
```
init/
└── mongo
    └── docker-entrypoint-initdb.d
        └── mongo-init.js
```

**In IntelliJ, you should see:**
```
microservices-parent/
├── init/
│   └── mongo/
│       └── docker-entrypoint-initdb.d/
│           └── mongo-init.js
├── product-service/
├── docker-compose.yml
└── ... other files
```

---

## Part 3: Configure Application Properties

### Step 7: Update Application Properties

Our application needs **two different configurations**:
1. **Local configuration** - When running from IntelliJ (connects to localhost)
2. **Docker configuration** - When running in container (connects to mongodb service)

**Why two configurations?**
- **Local development**: MongoDB at `localhost:27017`
- **Docker container**: MongoDB at `mongodb:27017` (service name)

**Solution: Spring Profiles**
- `application.properties` - Default (local) configuration
- `application-docker.properties` - Docker-specific configuration
- Activated by `SPRING_PROFILES_ACTIVE=docker` environment variable

#### 7.1 Update application.properties (Local Configuration)

**Location:** `product-service/src/main/resources/application.properties`

**Open the file and update to:**

```properties
# ============================================
# Product Service - Local Configuration
# Use this when running from IntelliJ IDE
# ============================================

# Application name
spring.application.name=product-service

# Server configuration
server.port=8084

# MongoDB configuration for LOCAL development
# Connects to MongoDB on localhost (your machine)
spring.data.mongodb.host=localhost
spring.data.mongodb.port=27017
spring.data.mongodb.database=product-service

# Note: No authentication for local MongoDB
# This assumes you have MongoDB running locally without auth
# If you have auth enabled locally, add:
# spring.data.mongodb.username=admin
# spring.data.mongodb.password=password
# spring.data.mongodb.authentication-database=admin

# Logging configuration
logging.level.ca.gbc.comp3095=INFO
logging.level.org.springframework.data.mongodb=DEBUG
```

### Step 8: Create Docker-Specific Configuration

#### 8.1 Create application-docker.properties

1. **Right-click** on `product-service/src/main/resources/`
2. Select **New → File**
3. Name it: `application-docker.properties`
4. Click **OK**

#### 8.2 Add Docker Configuration

**Complete application-docker.properties:**

```properties
# ============================================
# Product Service - Docker Configuration
# Activated when SPRING_PROFILES_ACTIVE=docker
# ============================================

# Application name
spring.application.name=product-service

# Server configuration
server.port=8084

# MongoDB configuration for DOCKER environment
# Important: Use service name "mongodb" instead of "localhost"
spring.data.mongodb.host=mongodb
spring.data.mongodb.port=27017
spring.data.mongodb.database=product-service

# MongoDB authentication
# Credentials must match those in docker-compose.yml
spring.data.mongodb.username=admin
spring.data.mongodb.password=password
spring.data.mongodb.authentication-database=admin

# Logging configuration
logging.level.ca.gbc.comp3095=INFO
logging.level.org.springframework.data.mongodb=DEBUG
```

#### 8.3 Understanding the Key Difference

**The CRITICAL difference between local and Docker configuration:**

**Local (application.properties):**
```properties
spring.data.mongodb.host=localhost
```

**Docker (application-docker.properties):**
```properties
spring.data.mongodb.host=mongodb  ← Service name from docker-compose.yml!
```

**Why "mongodb" works in Docker:**
- Docker Compose creates a **custom network**
- Docker provides **automatic DNS resolution**
- Service name `mongodb` resolves to the MongoDB container's IP address

**Diagram:**
```
product-service container:
  |
  ├─ Try to connect to "mongodb:27017"
  |
  ├─ Docker DNS looks up "mongodb" service
  |
  ├─ Returns MongoDB container's IP (e.g., 172.18.0.2)
  |
  └─ Connection established!
```

#### 8.4 How Spring Profiles Work

**When you run the application:**

**From IntelliJ (local):**
```
SPRING_PROFILES_ACTIVE not set
→ Uses application.properties
→ Connects to localhost:27017
```

**In Docker container:**
```
SPRING_PROFILES_ACTIVE=docker (from docker-compose.yml)
→ Uses application.properties + application-docker.properties
→ application-docker.properties OVERRIDES values from application.properties
→ Connects to mongodb:27017
```

**Profile Override Rules:**
1. Spring loads `application.properties` (base configuration)
2. If profile is active, loads `application-{profile}.properties`
3. Profile-specific values **override** base values
4. Non-overridden values from base are still used

#### 8.5 Verify Your Configuration Files

**Check you have both files:**
```
product-service/
└── src/
    └── main/
        └── resources/
            ├── application.properties         ← Local
            └── application-docker.properties  ← Docker
```

**In IntelliJ:**
```
product-service/
└── src/
    └── main/
        └── resources/
            ├── application.properties
            └── application-docker.properties
```

---

## Part 4: Run and Test Dockerized Application

### Step 9: Clean Existing Containers

Before starting our Docker Compose setup, we need to stop and remove any existing containers and images that might conflict.

#### 9.1 Check Running Containers

```bash
docker ps
```

**If you see any of these containers:**
- mongodb
- product-service
- mongo-express
- Any containers from previous labs

**Stop them:**

#### 9.2 Stop and Remove Containers (If Running)

**Option A: Stop specific containers**
```bash
# Stop containers
docker stop mongodb product-service mongo-express

# Remove containers
docker rm mongodb product-service mongo-express
```

**Option B: Stop all containers (nuclear option)**
```bash
# Stop all running containers
docker stop $(docker ps -q)

# Remove all stopped containers
docker rm $(docker ps -aq)
```

**Option C: Using Docker Desktop**
1. Open **Docker Desktop**
2. Go to **Containers** tab
3. Click **trash icon** next to each container you want to remove

#### 9.3 Remove Old product-service Image

We need to rebuild the image because we added `application-docker.properties`.

**Command:**
```bash
docker rmi product-service:1.0
```

**If that fails (image in use):**
```bash
# Force remove
docker rmi -f product-service:1.0
```

**Or via Docker Desktop:**
1. Go to **Images** tab
2. Find `product-service:1.0`
3. Click **trash icon**

#### 9.4 Verify Cleanup

```bash
# Should not show product-service, mongodb, or mongo-express
docker ps -a

# Should not show product-service:1.0
docker images
```

### Step 10: Start Docker Compose Network

Now we'll use Docker Compose to build and start all our services with one command!

#### 10.1 Navigate to microservices-parent

```bash
cd /path/to/your/microservices-parent
```

**Verify you're in the right place:**
```bash
ls -la
# You should see: docker-compose.yml, product-service/, init/
```

#### 10.2 Start Docker Compose

**Command:**
```bash
docker-compose -f docker-compose.yml up -d --build
```

**Breaking down the command:**
- `docker-compose`: Docker Compose CLI tool
- `-f docker-compose.yml`: Specify compose file (optional if file is named docker-compose.yml)
- `up`: Create and start containers
- `-d`: Detached mode (run in background)
- `--build`: Build images before starting (forces rebuild)

**Alternative (simpler):**
```bash
# Since our file is named docker-compose.yml, this works:
docker-compose up -d --build
```

#### 10.3 Watch the Output

**Expected output:**
```
Building product-service
[+] Building 45.2s (15/15) FINISHED
 => [build 1/5] FROM docker.io/library/eclipse-temurin:21-jdk
 => [build 2/5] WORKDIR /app
 => [build 3/5] COPY . .
 => [build 4/5] RUN chmod +x gradlew
 => [build 5/5] RUN ./gradlew clean build -x test --no-daemon
 => [stage-1 1/5] FROM docker.io/library/eclipse-temurin:21-jre
 => [stage-1 2/5] WORKDIR /app
 => [stage-1 3/5] RUN groupadd -r spring && useradd -r -g spring spring
 => [stage-1 4/5] COPY --from=build /app/build/libs/*.jar app.jar
 => [stage-1 5/5] RUN chown spring:spring app.jar
 => exporting to image

Creating network "microservices-parent_microservices-network" ... done
Creating volume "microservices-parent_mongodb-data" ... done
Creating mongodb ... done
Creating product-service ... done
Creating mongo-express ... done
```

**Timeline:**
- **Building product-service**: 2-5 minutes (first time)
- **Pulling MongoDB image**: 1-2 minutes (first time)
- **Starting containers**: 10-30 seconds

**Total first-time startup**: ~5-7 minutes
**Subsequent startups** (no --build): ~10 seconds

### Step 11: Verify Container Status

#### 11.1 Check Running Containers

**Command:**
```bash
docker-compose ps
```

**Expected output:**
```
NAME              IMAGE                 COMMAND                  SERVICE           STATUS          PORTS
mongodb           mongo:latest          "docker-entrypoint.s…"   mongodb           Up 30 seconds   0.0.0.0:27017->27017/tcp
mongo-express     mongo-express:latest  "tini -- /docker-ent…"   mongo-express     Up 30 seconds   0.0.0.0:8081->8081/tcp
product-service   product-service:1.0   "java -jar app.jar"      product-service   Up 30 seconds   0.0.0.0:8084->8084/tcp
```

**All three services should show:**
- ✅ **STATUS**: Up X seconds
- ✅ **PORTS**: Correct port mappings

**Or using regular docker command:**
```bash
docker ps
```

#### 11.2 Verify in Docker Desktop

1. Open **Docker Desktop**
2. Go to **Containers** tab
3. You should see a **group** named `microservices-parent` (or your folder name)
4. Expand it to see:
   - ✅ mongodb (green, running)
   - ✅ product-service (green, running)
   - ✅ mongo-express (green, running)

**Screenshot location:** This is what it should look like (see Week 5 screenshots)

#### 11.3 Check Container Logs

**View logs for each service:**

**MongoDB logs:**
```bash
docker-compose logs mongodb
```

**Look for:**
```
MongoDB initialization completed successfully!
Database: product-service
User: admin
Collection: product
```
This confirms our `mongo-init.js` script ran successfully!

**Product Service logs:**
```bash
docker-compose logs product-service
```

**Look for:**
```
Started ProductServiceApplication in X seconds
Tomcat started on port 8084
```

**If you see errors about MongoDB connection:**
- Container might still be starting (wait 30 seconds)
- Check MongoDB is running: `docker ps | grep mongodb`
- Check logs: `docker-compose logs mongodb`

**Mongo Express logs:**
```bash
docker-compose logs mongo-express
```

**Look for:**
```
Mongo Express server listening at http://0.0.0.0:8081
```

#### 11.4 Test Mongo Express Web UI (Optional)

1. Open browser: http://localhost:8081
2. Login:
   - Username: `admin`
   - Password: `pass`
   (These are the web UI credentials from docker-compose.yml, NOT MongoDB credentials!)
3. You should see:
   - ✅ `product-service` database listed
   - ✅ Click on it to see `product` collection

**This provides a visual way to inspect your database!**

#### 11.5 Troubleshooting Container Startup

**Problem: Containers keep restarting**

Check logs:
```bash
docker-compose logs --tail=50
```

**Common issues:**

**1. MongoDB authentication errors:**
```
Authentication failed
```
**Solution:** Check credentials in `application-docker.properties` match `docker-compose.yml`

**2. Port already in use:**
```
Bind for 0.0.0.0:8084 failed: port is already allocated
```
**Solution:**
```bash
# Find what's using the port
lsof -i :8084
# Kill the process or change port in docker-compose.yml
```

**3. Application can't connect to MongoDB:**
```
MongoSocketOpenException: Exception opening socket
```
**Solution:**
- Wait 30 seconds (MongoDB might still be initializing)
- Check MongoDB is running: `docker ps | grep mongodb`
- Verify `spring.data.mongodb.host=mongodb` (not localhost!)

**4. Gradle build fails in Docker:**
```
FAILURE: Build failed with an exception
```
**Solution:**
```bash
# Rebuild locally first to ensure code compiles
cd product-service
./gradlew clean build
# Then rebuild Docker image
cd ..
docker-compose up -d --build
```

---

## Part 5: Test with Postman

Now that our containerized application is running, let's test all CRUD operations!

### Step 12: Test POST Request

#### 12.1 Open Postman

1. Launch **Postman**
2. Create a new workspace: `Docker Containerization Tests` (or use existing)
3. Create a new collection: `Product Service Docker Tests`

#### 12.2 Create POST Request

**Request Configuration:**
1. Click **+ New Request**
2. Name it: `Create Product (Docker)`
3. Method: **POST**
4. URL: `http://localhost:8084/api/product`

#### 12.3 Configure Request Body

1. Click **Body** tab
2. Select **raw**
3. Select **JSON** from dropdown (next to GraphQL)
4. Enter JSON data:

```json
{
  "name": "MacBook Pro M3 Max",
  "description": "14-inch MacBook Pro with M3 Max chip - Containerized",
  "price": 3199.99
}
```

#### 12.4 Send Request

1. Click **Send**
2. **Expected Response:**
   - Status: `201 Created`
   - Body: Empty (our controller returns void)

**Success indicator:**
```
Status: 201 Created
```

#### 12.5 Verify in Logs

**Check product-service logs:**
```bash
docker-compose logs -f product-service
```

**You should see:**
```
Product 644e500c73c54e784fbb6b19 is saved
```

The hexadecimal string is the MongoDB ObjectId of the created product.

**Press Ctrl+C to exit log viewing**

#### 12.6 Test Creating Multiple Products

Create 2-3 more products with different data:

**Product 2:**
```json
{
  "name": "iPad Pro 2025",
  "description": "12.9-inch iPad Pro with M3 chip",
  "price": 1299.00
}
```

**Product 3:**
```json
{
  "name": "AirPods Pro (3rd Gen)",
  "description": "Active Noise Cancellation with Spatial Audio",
  "price": 249.99
}
```

Each should return `201 Created`.

### Step 13: Verify Data in MongoDB

Let's verify the data was actually saved to MongoDB using **mongosh** (MongoDB Shell).

#### 13.1 Access MongoDB Container

**Open a shell inside the MongoDB container:**
```bash
docker exec -it mongodb mongosh
```

**Breaking down the command:**
- `docker exec`: Execute command in running container
- `-it`: Interactive terminal
- `mongodb`: Container name (from docker-compose.yml)
- `mongosh`: MongoDB shell command

**You should see:**
```
Current Mongosh Log ID: ...
Connecting to: mongodb://127.0.0.1:27017/...
Using MongoDB: 7.x.x
Using Mongosh: 2.x.x

test>
```

#### 13.2 Switch to product-service Database

```javascript
use product-service
```

**Output:**
```
switched to db product-service
```

#### 13.3 Authenticate as Admin User

```javascript
db.auth("admin", "password")
```

**Output:**
```
{ ok: 1 }
```

If you see `{ ok: 0 }`, authentication failed. Check credentials match `mongo-init.js`.

#### 13.4 Query Product Collection

**Find all products:**
```javascript
db.product.find()
```

**Expected output:**
```javascript
[
  {
    _id: ObjectId('644e500c73c54e784fbb6b19'),
    name: 'MacBook Pro M3 Max',
    description: '14-inch MacBook Pro with M3 Max chip - Containerized',
    price: Decimal128("3199.99"),
    _class: 'ca.gbc.comp3095.productservice.model.Product'
  },
  {
    _id: ObjectId('644e532a73c54e784fbb6b20'),
    name: 'iPad Pro 2025',
    description: '12.9-inch iPad Pro with M3 chip',
    price: Decimal128("1299.00"),
    _class: 'ca.gbc.comp3095.productservice.model.Product'
  },
  {
    _id: ObjectId('644e537b73c54e784fbb6b21'),
    name: 'AirPods Pro (3rd Gen)',
    description: 'Active Noise Cancellation with Spatial Audio',
    price: Decimal128("249.99"),
    _class: 'ca.gbc.comp3095.productservice.model.Product'
  }
]
```

**Understanding the output:**
- `_id`: MongoDB's unique identifier (auto-generated)
- `_class`: Spring Data MongoDB metadata (stores entity class)
- `price`: Stored as `Decimal128` (MongoDB's decimal type for financial data)

#### 13.5 Count Documents

```javascript
db.product.countDocuments()
```

**Output:**
```
3
```

#### 13.6 Find Specific Product

```javascript
db.product.findOne({ name: "MacBook Pro M3 Max" })
```

#### 13.7 Exit MongoDB Shell

```javascript
exit
```

Or press `Ctrl+D`

### Step 14: Test GET Request

#### 14.1 Create GET Request in Postman

**Request Configuration:**
1. New Request
2. Name: `Get All Products (Docker)`
3. Method: **GET**
4. URL: `http://localhost:8084/api/product`
5. No body needed

#### 14.2 Send Request

Click **Send**

**Expected Response:**
- Status: `200 OK`
- Body: JSON array of products

**Example response:**
```json
[
  {
    "id": "644e500c73c54e784fbb6b19",
    "name": "MacBook Pro M3 Max",
    "description": "14-inch MacBook Pro with M3 Max chip - Containerized",
    "price": 3199.99
  },
  {
    "id": "644e532a73c54e784fbb6b20",
    "name": "iPad Pro 2025",
    "description": "12.9-inch iPad Pro with M3 chip",
    "price": 1299.00
  },
  {
    "id": "644e537b73c54e784fbb6b21",
    "name": "AirPods Pro (3rd Gen)",
    "description": "Active Noise Cancellation with Spatial Audio",
    "price": 249.99
  }
]
```

**Verify:**
- ✅ Status is `200 OK`
- ✅ Response is valid JSON array
- ✅ All products created earlier are returned
- ✅ Each product has `id`, `name`, `description`, `price`

#### 14.3 Test GET by ID

**Request Configuration:**
1. New Request
2. Name: `Get Product by ID (Docker)`
3. Method: **GET**
4. URL: `http://localhost:8084/api/product/{id}`
   - Replace `{id}` with an actual ID from the GET all response
   - Example: `http://localhost:8084/api/product/644e500c73c54e784fbb6b19`

**Send request:**
- Expected: `200 OK` with single product object

**Example response:**
```json
{
  "id": "644e500c73c54e784fbb6b19",
  "name": "MacBook Pro M3 Max",
  "description": "14-inch MacBook Pro with M3 Max chip - Containerized",
  "price": 3199.99
}
```

### Step 15: Test PUT Request

#### 15.1 Get a Product ID

From the GET all products response, copy one product's ID.

**Example ID:** `644e500c73c54e784fbb6b19`

#### 15.2 Create PUT Request

**Request Configuration:**
1. New Request
2. Name: `Update Product (Docker)`
3. Method: **PUT**
4. URL: `http://localhost:8084/api/product/644e500c73c54e784fbb6b19`
   (Replace with your actual product ID)

#### 15.3 Configure Request Body

1. Click **Body** tab
2. Select **raw**
3. Select **JSON**
4. Enter updated data:

```json
{
  "name": "MacBook Pro M3 Max (Updated)",
  "description": "16-inch MacBook Pro with M3 Max chip - Updated in Docker",
  "price": 3499.99
}
```

#### 15.4 Send Request

Click **Send**

**Expected Response:**
- Status: `204 No Content`
- Body: Empty (204 means success with no response body)

**Success indicator:**
```
Status: 204 No Content
```

#### 15.5 Verify Update

**Option 1: GET request**
Send a GET request to the same product ID.

**Expected:** Product should show updated values.

**Option 2: MongoDB shell**
```bash
docker exec -it mongodb mongosh
use product-service
db.auth("admin", "password")
db.product.findOne({ _id: ObjectId("644e500c73c54e784fbb6b19") })
exit
```

**Expected:** Document should show new values.

**Option 3: Mongo Express**
1. Open http://localhost:8081
2. Navigate to product-service → product collection
3. Find the product and verify changes

### Step 16: Test DELETE Request

#### 16.1 Get a Product ID to Delete

From the GET all products response, choose a product to delete.

**Example ID:** `644e537b73c54e784fbb6b21` (AirPods Pro)

#### 16.2 Create DELETE Request

**Request Configuration:**
1. New Request
2. Name: `Delete Product (Docker)`
3. Method: **DELETE**
4. URL: `http://localhost:8084/api/product/644e537b73c54e784fbb6b21`
   (Replace with your actual product ID)
5. No body needed

#### 16.3 Send Request

Click **Send**

**Expected Response:**
- Status: `204 No Content`
- Body: Empty

**Success indicator:**
```
Status: 204 No Content
```

#### 16.4 Verify Deletion

**Option 1: GET all products**
Send GET request to `/api/product`

**Expected:** Product should no longer be in the list.

**Option 2: GET by ID (expect 500 error)**
Send GET request to `/api/product/644e537b73c54e784fbb6b21`

**Expected:** `500 Internal Server Error` (because product not found)

> **Note:** Our current implementation throws RuntimeException for not found. In Week 6, we'll improve this to return proper `404 Not Found`.

**Option 3: MongoDB shell**
```bash
docker exec -it mongodb mongosh
use product-service
db.auth("admin", "password")
db.product.countDocuments()  # Should be 2 (one less than before)
db.product.find()            # Product should not appear
exit
```

**Option 4: Check logs**
```bash
docker-compose logs product-service | grep "is deleted"
```

**Expected:**
```
Product 644e537b73c54e784fbb6b21 is deleted
```

#### 16.5 Final Verification

**At this point, you've successfully:**
- ✅ Created products (POST)
- ✅ Retrieved all products (GET)
- ✅ Retrieved single product (GET by ID)
- ✅ Updated a product (PUT)
- ✅ Deleted a product (DELETE)

**All in a fully containerized environment!** 🎉

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: Container Won't Start

**Error:**
```
Error response from daemon: driver failed programming external connectivity
```

**Cause:** Port already in use

**Solution:**
```bash
# Find what's using the port
lsof -i :8084

# Kill the process
kill -9 <PID>

# Or change port in docker-compose.yml:
ports:
  - "8085:8084"  # Use port 8085 on host
```

---

#### Issue 2: MongoDB Connection Refused

**Error in logs:**
```
com.mongodb.MongoSocketOpenException: Exception opening socket
```

**Solutions:**

**1. Check MongoDB is running:**
```bash
docker ps | grep mongodb
```

**2. Wait for MongoDB to fully start:**
```bash
# Watch MongoDB logs
docker-compose logs -f mongodb
# Wait for: "Waiting for connections"
```

**3. Verify network connectivity:**
```bash
# Shell into product-service container
docker exec -it product-service sh
# Try to ping MongoDB
ping mongodb
# Should get responses
```

**4. Check application-docker.properties:**
```properties
# Make sure it says:
spring.data.mongodb.host=mongodb  # NOT localhost!
```

---

#### Issue 3: Authentication Failed

**Error:**
```
MongoCommandException: Command failed with error 18 (AuthenticationFailed)
```

**Solutions:**

**1. Verify credentials match:**

**docker-compose.yml:**
```yaml
environment:
  MONGO_INITDB_ROOT_USERNAME: admin
  MONGO_INITDB_ROOT_PASSWORD: password
```

**application-docker.properties:**
```properties
spring.data.mongodb.username=admin
spring.data.mongodb.password=password
```

**2. Delete and recreate volume:**
```bash
# Stop containers
docker-compose down

# Remove volume (this deletes all data!)
docker volume rm microservices-parent_mongodb-data

# Restart (MongoDB will initialize fresh)
docker-compose up -d --build
```

---

#### Issue 4: mongo-init.js Didn't Run

**Symptom:** Can't authenticate with `admin/password` in mongosh

**Solutions:**

**1. Check script was mounted:**
```bash
docker exec -it mongodb ls /docker-entrypoint-initdb.d/
# Should show: mongo-init.js
```

**2. Check MongoDB logs for initialization:**
```bash
docker-compose logs mongodb | grep "initialization"
# Should show: "MongoDB initialization completed successfully!"
```

**3. Script only runs on FIRST startup:**
- If MongoDB data already exists, script doesn't run
- Delete volume and restart:
```bash
docker-compose down
docker volume rm microservices-parent_mongodb-data
docker-compose up -d
```

---

#### Issue 5: Gradle Build Fails in Docker

**Error:**
```
> Task :build FAILED
FAILURE: Build failed with an exception
```

**Solutions:**

**1. Test build locally first:**
```bash
cd product-service
./gradlew clean build
# Fix any errors before Docker build
```

**2. Check Dockerfile COPY command:**
```dockerfile
# Make sure you're copying all files:
COPY . .
```

**3. Rebuild with no cache:**
```bash
docker-compose build --no-cache product-service
docker-compose up -d
```

---

#### Issue 6: Changes Not Reflected

**Symptom:** You changed code but container still runs old version

**Solutions:**

**1. Rebuild the image:**
```bash
docker-compose up -d --build
```

**2. Force complete rebuild:**
```bash
# Stop and remove containers
docker-compose down

# Remove image
docker rmi product-service:1.0

# Rebuild and start
docker-compose up -d --build
```

---

#### Issue 7: YAML Syntax Error

**Error:**
```
yaml: line 23: mapping values are not allowed in this context
```

**Cause:** YAML indentation error

**Solutions:**

**1. Check indentation:**
- Use **2 spaces** for each level
- **NO tabs!** (Configure IntelliJ to use spaces)

**2. Use YAML validator:**
- http://www.yamllint.com/
- Paste your docker-compose.yml to check syntax

**3. Compare with working example:**
- Check this guide's docker-compose.yml
- Compare line-by-line

---

#### Issue 8: Volume Permission Errors

**Error:**
```
Permission denied: '/data/db'
```

**Solutions:**

**1. For MongoDB (usually not needed):**
```yaml
# Add to mongodb service in docker-compose.yml:
user: "1000:1000"  # Your user:group ID
```

**2. Check volume exists:**
```bash
docker volume ls | grep mongodb-data
```

**3. Reset volume:**
```bash
docker-compose down
docker volume rm microservices-parent_mongodb-data
docker-compose up -d
```

---

#### Issue 9: Can't Access Mongo Express

**Symptom:** http://localhost:8081 doesn't load

**Solutions:**

**1. Check container is running:**
```bash
docker ps | grep mongo-express
```

**2. Check logs:**
```bash
docker-compose logs mongo-express
```

**3. Verify port not in use:**
```bash
lsof -i :8081
```

**4. Try different browser or incognito mode**

**5. Check credentials:**
- Username: `admin`
- Password: `pass`
- (These are web UI credentials, NOT MongoDB credentials)

---

#### Issue 10: Container Keeps Restarting

**Symptom:** Container status shows "Restarting"

**Solutions:**

**1. Check logs for error:**
```bash
docker-compose logs --tail=100 product-service
```

**2. Common causes:**
- Application crash on startup
- MongoDB not ready (add healthcheck)
- Port conflict
- Out of memory

**3. Remove restart policy temporarily:**
```yaml
# Comment out in docker-compose.yml:
# restart: unless-stopped
```

**4. Run container interactively to debug:**
```bash
docker-compose up product-service
# (without -d flag, see output in terminal)
```

---

#### Issue 11: Postman Tests Fail

**Error:** `Could not get response` or `Connection refused`

**Solutions:**

**1. Verify container is running:**
```bash
docker ps | grep product-service
```

**2. Check correct port:**
- URL should be `http://localhost:8084` (not 8080!)

**3. Check application started:**
```bash
docker-compose logs product-service | grep "Started"
# Should show: "Started ProductServiceApplication"
```

**4. Test with curl:**
```bash
curl http://localhost:8084/api/product
# Should return product JSON
```

**5. Check firewall not blocking Docker**

---

#### Issue 12: "No such file or directory" Error

**Error building image:**
```
COPY failed: file not found in build context
```

**Solutions:**

**1. Check Dockerfile location:**
```bash
# Dockerfile should be at:
product-service/Dockerfile
# NOT at:
microservices-parent/Dockerfile
```

**2. Check build context in docker-compose.yml:**
```yaml
build:
  context: ./product-service  # Correct
  # NOT: context: .
```

**3. Verify files exist:**
```bash
cd product-service
ls -la
# Should show: Dockerfile, gradlew, build.gradle.kts, src/
```

---

## Docker Commands Reference

### Essential Docker Commands

**Container Management:**
```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Start container
docker start <container-name>

# Stop container
docker stop <container-name>

# Restart container
docker restart <container-name>

# Remove container
docker rm <container-name>

# Force remove running container
docker rm -f <container-name>

# Remove all stopped containers
docker container prune
```

**Image Management:**
```bash
# List images
docker images

# Remove image
docker rmi <image-name>

# Force remove image
docker rmi -f <image-name>

# Remove unused images
docker image prune

# Remove all unused images
docker image prune -a
```

**Logs and Debugging:**
```bash
# View container logs
docker logs <container-name>

# Follow logs (real-time)
docker logs -f <container-name>

# View last N lines
docker logs --tail=50 <container-name>

# Shell into running container
docker exec -it <container-name> sh
# Or for bash:
docker exec -it <container-name> bash

# View container details
docker inspect <container-name>

# View container resource usage
docker stats <container-name>
```

**Network Management:**
```bash
# List networks
docker network ls

# Inspect network
docker network inspect <network-name>

# Remove network
docker network rm <network-name>
```

**Volume Management:**
```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect <volume-name>

# Remove volume
docker volume rm <volume-name>

# Remove all unused volumes
docker volume prune
```

### Docker Compose Commands

**Basic Operations:**
```bash
# Start services (detached)
docker-compose up -d

# Start and rebuild images
docker-compose up -d --build

# Stop services
docker-compose stop

# Stop and remove containers
docker-compose down

# Stop and remove containers, networks, volumes
docker-compose down -v

# Restart services
docker-compose restart
```

**Service Management:**
```bash
# List services
docker-compose ps

# View logs
docker-compose logs

# View logs for specific service
docker-compose logs <service-name>

# Follow logs
docker-compose logs -f

# Build or rebuild services
docker-compose build

# Build without cache
docker-compose build --no-cache

# Run command in service
docker-compose exec <service-name> <command>
```

**Scaling:**
```bash
# Scale service (run multiple instances)
docker-compose up -d --scale product-service=3
```

**Validation:**
```bash
# Validate docker-compose.yml syntax
docker-compose config
```

### Useful Command Combinations

**Complete cleanup:**
```bash
# Stop everything
docker-compose down

# Remove containers, networks, volumes, images
docker-compose down --rmi all --volumes --remove-orphans

# Clean all Docker resources
docker system prune -a --volumes
```

**Quick restart with rebuild:**
```bash
docker-compose down && docker-compose up -d --build
```

**View resource usage:**
```bash
# All containers
docker stats

# Specific container
docker stats product-service
```

**Export/Import volumes:**
```bash
# Backup MongoDB data
docker run --rm -v microservices-parent_mongodb-data:/data -v $(pwd):/backup ubuntu tar czf /backup/mongodb-backup.tar.gz /data

# Restore MongoDB data
docker run --rm -v microservices-parent_mongodb-data:/data -v $(pwd):/backup ubuntu tar xzf /backup/mongodb-backup.tar.gz -C /
```

---

## Repository Submission

### Step 1: Verify All Files Are Present

**Check you have created:**
```
microservices-parent/
├── docker-compose.yml                                    ← New
├── init/                                                 ← New
│   └── mongo/
│       └── docker-entrypoint-initdb.d/
│           └── mongo-init.js
└── product-service/
    ├── Dockerfile                                        ← New
    └── src/
        └── main/
            └── resources/
                ├── application.properties                ← Updated
                └── application-docker.properties         ← New
```

### Step 2: Test One More Time

Before committing, verify everything works:

```bash
# Clean start
docker-compose down
docker volume rm microservices-parent_mongodb-data
docker-compose up -d --build

# Wait 30 seconds for startup

# Test in Postman
# - POST a product
# - GET all products
# - Verify data persisted
```

### Step 3: Stop Containers (Optional)

```bash
# Stop but keep data
docker-compose stop

# Or stop and remove containers (keeps volumes)
docker-compose down
```

### Step 4: Create .dockerignore (Optional but Recommended)

Create `product-service/.dockerignore` to exclude unnecessary files from Docker build:

```bash
# In product-service directory
cat > .dockerignore << 'EOF'
.gradle
build
.idea
*.iml
.git
README.md
.DS_Store
EOF
```

### Step 5: Commit and Push

```bash
# Navigate to project root
cd /path/to/your/microservices-parent

# Check status
git status
# Should show:
# - docker-compose.yml
# - init/ directory
# - product-service/Dockerfile
# - product-service/src/main/resources/application-docker.properties
# - Updated application.properties

# Add all new and modified files
git add .

# Commit with descriptive message
git commit -m "Add Docker containerization with multi-stage build and docker-compose

- Created multi-stage Dockerfile for optimized image size
- Implemented docker-compose.yml for multi-container orchestration
- Added MongoDB initialization script for user creation
- Configured Spring profiles for local vs Docker environments
- Added Mongo Express web UI for database management
- All CRUD operations tested and working in containers"

# Push to repository
git push origin main
```

### Step 6: Verify on GitLab/GitHub

1. Navigate to your repository in browser
2. Verify all new files are present:
   - ✅ docker-compose.yml
   - ✅ init/mongo/docker-entrypoint-initdb.d/mongo-init.js
   - ✅ product-service/Dockerfile
   - ✅ product-service/src/main/resources/application-docker.properties

### Step 7: Export Postman Collection (Optional)

1. In Postman, click **⋮** (three dots) on your collection
2. Select **Export**
3. Choose **Collection v2.1**
4. Save as: `Week05_Docker_Tests.postman_collection.json`
5. Add to repository:
```bash
mkdir -p Week05/postman
mv ~/Downloads/Week05_Docker_Tests.postman_collection.json Week05/postman/
git add Week05/
git commit -m "Add Postman collection for Docker testing"
git push origin main
```

---

## Next Steps

### What You've Accomplished

Congratulations! You've successfully:
- ✅ Created a multi-stage Dockerfile for optimized Java applications
- ✅ Implemented docker-compose for multi-container orchestration
- ✅ Configured MongoDB with authentication in containers
- ✅ Managed Docker networks and volumes for data persistence
- ✅ Tested complete CRUD operations in containerized environment
- ✅ Learned Docker debugging and troubleshooting techniques

### Skills Gained

You can now:
- ✅ Containerize Spring Boot applications
- ✅ Orchestrate multi-container applications
- ✅ Configure environment-specific settings
- ✅ Debug containerized applications
- ✅ Manage Docker images, containers, volumes, and networks
- ✅ Use Docker for consistent development environments

### Production Readiness

**What we have:**
- ✅ Containerized application
- ✅ Multi-stage builds (optimized images)
- ✅ Non-root user (security)
- ✅ Data persistence (volumes)
- ✅ Environment configuration (Spring profiles)

**What's still needed for production:**
- ⚠️ Security (Spring Security, JWT, HTTPS)
- ⚠️ Monitoring (Prometheus, Grafana)
- ⚠️ Logging (ELK stack, centralized logs)
- ⚠️ CI/CD pipeline (GitHub Actions, Jenkins)
- ⚠️ Container orchestration (Kubernetes, Docker Swarm)
- ⚠️ Load balancing and scaling
- ⚠️ Health checks and readiness probes
- ⚠️ Secrets management (not hardcoded passwords)

### Suggested Enhancements (Optional)

**1. Add Health Checks**

Update `docker-compose.yml`:
```yaml
product-service:
  # ... existing config
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8084/actuator/health"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s
```

**2. Add Resource Limits**

```yaml
product-service:
  # ... existing config
  deploy:
    resources:
      limits:
        cpus: '1.0'
        memory: 512M
      reservations:
        cpus: '0.5'
        memory: 256M
```

**3. Add Logging Configuration**

```yaml
product-service:
  # ... existing config
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "3"
```

**4. Use .env File for Secrets**

Create `.env`:
```env
MONGO_ROOT_USERNAME=admin
MONGO_ROOT_PASSWORD=your_secure_password
```

Update `docker-compose.yml`:
```yaml
environment:
  MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
  MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
```

**5. Add Second Microservice**

In future weeks, you might add:
- Order Service
- User Service
- API Gateway

Each will be added to `docker-compose.yml` as additional services!

### Learning Resources

**Docker Official Documentation:**
- https://docs.docker.com/
- https://docs.docker.com/compose/

**Docker Best Practices:**
- https://docs.docker.com/develop/dev-best-practices/
- https://docs.docker.com/develop/develop-images/dockerfile_best-practices/

**Spring Boot Docker Guide:**
- https://spring.io/guides/gs/spring-boot-docker/
- https://spring.io/guides/topicals/spring-boot-docker/

**Docker Hub (Official Images):**
- https://hub.docker.com/
- MongoDB: https://hub.docker.com/_/mongo
- Eclipse Temurin: https://hub.docker.com/_/eclipse-temurin

### Preparing for Week 6

**Topics you might cover next:**
- Spring Security and Authentication
- API Documentation with Swagger/OpenAPI
- Error Handling and Custom Exceptions
- Input Validation with Bean Validation
- Second Microservice (Order Service)
- Service-to-Service Communication

**Recommended preparation:**
- Keep Docker Desktop running
- Review your Postman collection
- Understand how containers communicate via networks
- Be comfortable with docker-compose commands

---

## Conclusion

You've successfully containerized your Product Service microservice! This is a major milestone in modern application development. Your application now runs in a consistent, portable environment that works the same on any machine with Docker installed.

**Key Takeaways:**

1. **Docker Images** are blueprints, **Containers** are running instances
2. **Multistage builds** reduce image size by 50-70%
3. **docker-compose** simplifies multi-container applications
4. **Spring Profiles** enable environment-specific configuration
5. **Volumes** persist data across container restarts
6. **Networks** allow containers to communicate by service name

**You now have a solid foundation for:**
- Microservices development
- Cloud-native applications
- DevOps practices
- Container orchestration (future: Kubernetes)

Keep practicing with Docker - it's an essential skill for modern software development!

---

**Lab Complete!** 🎉🐳

**Next:** Week 6 - Spring Security and Authentication (or instructor's topic)

---

**Version:** 1.0
**Last Updated:** Week 5, Fall 2025
**Author:** Based on COMP 3095 Course Materials
**Technologies:** Docker 20.10+, Docker Compose 2.x, Spring Boot 3.5.6, MongoDB latest