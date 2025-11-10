# API Gateway Implementation

## Introduction

Add API Gateway to your microservices architecture. The gateway acts as a single entry point, routing client requests to product-service and order-service.

**Time:** 90-120 minutes

---

## Table of Contents

- [Step 1: Create api-gateway Module](#step-1-create-api-gateway-module)
- [Step 2: Configure Dependencies](#step-2-configure-dependencies)
- [Step 3: Create Application Class](#step-3-create-application-class)
- [Step 4: Create Routes Configuration](#step-4-create-routes-configuration)
- [Step 5: Configure application.properties](#step-5-configure-applicationproperties)
- [Step 6: Configure application-docker.properties](#step-6-configure-application-dockerproperties)
- [Step 7: Create Dockerfile](#step-7-create-dockerfile)
- [Step 8: Update settings.gradle.kts](#step-8-update-settingsgradlekts)
- [Step 9: Update docker-compose.yml](#step-9-update-docker-composeyml)
- [Step 10: Test Locally](#step-10-test-locally)
- [Step 11: Test with Docker Compose](#step-11-test-with-docker-compose)
- [Summary](#summary)

---

## Prerequisites

### Required Completed Labs:
- ✅ Product Service with MongoDB and Redis
- ✅ Order Service with PostgreSQL and OpenFeign
- ✅ Inventory Service with Flyway

### Required Software:
- ✅ IntelliJ IDEA 2024.1+
- ✅ Java JDK 21
- ✅ Docker Desktop (running)
- ✅ Postman

---

## Architecture Overview

```
Client → API Gateway (9000) → Product Service (8084)
                            → Order Service (8082)
```

**Benefits:**
- ✅ Single entry point for all clients
- ✅ Simplified client configuration
- ✅ Centralized request logging
- ✅ Future: Authentication, rate limiting, load balancing

---

## Step 1: Create api-gateway Module

### 1.1 Using Spring Initializr in IntelliJ

1. **Right-click** on `microservices-parent`
2. Select **New → Module**
3. Choose **Spring Initializr**
4. Click **Next**

### 1.2 Project Configuration

| Setting | Value |
|---------|-------|
| **Name** | `api-gateway` |
| **Group** | `ca.gbc.comp3095` |
| **Artifact** | `api-gateway` |
| **Package name** | `ca.gbc.comp3095.apigateway` |
| **Type** | Gradle - Kotlin |
| **Language** | Java |
| **Java** | 21 |
| **Packaging** | Jar |
| **Spring Boot** | 3.5.7 |

**Note:** Use `ca.gbc.comp3095` for Group to match your existing services (product-service, order-service, inventory-service).

### 1.3 Select Dependencies

#### **Developer Tools**
- ✅ **Lombok**
- ✅ **Spring Boot DevTools**

#### **Web**
- ✅ **Spring Web**

#### **Ops**
- ✅ **Spring Boot Actuator**

#### **Spring Cloud Routing**
- ✅ Expand **Spring Cloud Routing** section
- ✅ Select **Gateway**

**Important:** Do NOT select "Reactive Gateway".

### 1.4 Complete Creation

1. Click **Next**
2. Click **Finish**
3. Wait for Gradle sync

---

## Step 2: Configure Dependencies

### 2.1 Update build.gradle.kts

**Location:** `microservices-parent/api-gateway/build.gradle.kts`

Replace entire file:

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.7"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "ca.gbc.comp3095"
version = "0.0.1-SNAPSHOT"
description = "api-gateway"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

configurations {
    compileOnly {
        extendsFrom(configurations.annotationProcessor.get())
    }
}

repositories {
    mavenCentral()
}

extra["springCloudVersion"] = "2025.0.0"

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.cloud:spring-cloud-starter-gateway-server-webmvc")
    compileOnly("org.projectlombok:lombok")
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:${property("springCloudVersion")}")
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

**Key Dependencies:**
- `spring-cloud-starter-gateway-server-webmvc` - Gateway functionality
- `spring-boot-starter-web` - Web support

**Reload Gradle.**

---

## Step 3: Create Application Class

### 3.1 Verify Main Application

**Location:** `api-gateway/src/main/java/ca/gbc/comp3095/apigateway/ApiGatewayApplication.java`

Spring Initializr should have created:

```java
package ca.gbc.comp3095.apigateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ApiGatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }

}
```

No changes needed.

---

## Step 4: Create Routes Configuration

### 4.1 Create routes Package

1. **Right-click** on `ca.gbc.comp3095.apigateway`
2. Select **New → Package**
3. Name: `routes`
4. Press **Enter**

### 4.2 Create Routes Class

1. **Right-click** on `routes` package
2. Select **New → Java Class**
3. Name: `Routes`
4. Click **OK**

**Location:** `api-gateway/src/main/java/ca/gbc/comp3095/apigateway/routes/Routes.java`

Replace entire file with:

```java
package ca.gbc.comp3095.apigateway.routes;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gateway.server.mvc.handler.GatewayRouterFunctions;
import org.springframework.cloud.gateway.server.mvc.handler.HandlerFunctions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.function.*;

@Configuration
@Slf4j
public class Routes {

    @Value("${services.product-url}")
    private String productServiceUrl;

    @Value("${services.order-url}")
    private String orderServiceUrl;

    @Bean
    public RouterFunction<ServerResponse> productServiceRoute() {
        log.info("Initializing product service route with URL: {}", productServiceUrl);

        return GatewayRouterFunctions.route("product_service")
                .route(
                        RequestPredicates.path("/api/product"),
                        request -> {
                            log.info("Received request for product service: {}", request.uri());
                            try {
                                ServerResponse response = HandlerFunctions.http(productServiceUrl).handle(request);
                                log.info("Response status: {}", response.statusCode());
                                return response;
                            } catch (Exception e) {
                                log.error("Error occurred while routing request: {}", e.getMessage(), e);
                                return ServerResponse.status(500).body("An error occurred");
                            }
                        })
                .build();
    }

    @Bean
    public RouterFunction<ServerResponse> orderServiceRoute() {
        log.info("Initializing order service route with URL: {}", orderServiceUrl);

        return GatewayRouterFunctions.route("order_service")
                .route(
                        RequestPredicates.path("/api/order"),
                        request -> {
                            log.info("Received request for order service: {}", request.uri());
                            try {
                                ServerResponse response = HandlerFunctions.http(orderServiceUrl).handle(request);
                                log.info("Response status: {}", response.statusCode());
                                return response;
                            } catch (Exception e) {
                                log.error("Error occurred while routing request: {}", e.getMessage(), e);
                                return ServerResponse.status(500).body("An error occurred");
                            }
                        })
                .build();
    }
}
```

**Explanation:**
- `@Configuration` - Spring configuration class
- `@Value` - Injects URLs from properties
- `GatewayRouterFunctions.route()` - Creates route with ID
- `RequestPredicates.path()` - Matches incoming path
- `HandlerFunctions.http()` - Forwards to backend service

---

## Step 5: Configure application.properties

**Location:** `api-gateway/src/main/resources/application.properties`

Replace with:

```properties
spring.application.name=api-gateway

server.port=9000

#services
services.product-url=http://localhost:8084
services.order-url=http://localhost:8082
```

**Configuration:**
- Gateway runs on port **9000**
- Routes to localhost (local development)

---

## Step 6: Configure application-docker.properties

**Location:** `api-gateway/src/main/resources/application-docker.properties`

Create new file:

```properties
spring.application.name=api-gateway

server.port=9000

#services
services.product-url=http://product-service:8084
services.order-url=http://order-service:8082
```

**Configuration:**
- Uses Docker service names instead of localhost
- Activated with `SPRING_PROFILES_ACTIVE=docker`

---

## Step 7: Create Dockerfile

**Location:** `api-gateway/Dockerfile`

```dockerfile
# ------------------
#  Build Stage
# ------------------
FROM gradle:8-jdk21-alpine AS builder

COPY --chown=gradle:gradle . /home/gradle/src

WORKDIR /home/gradle/src

RUN gradle build -x test

# ------------------
#  Package Stage
# ------------------
FROM openjdk:21-jdk

RUN mkdir /app

COPY --from=builder /home/gradle/src/build/libs/*.jar /app/api-gateway.jar

EXPOSE 9000

ENTRYPOINT ["java", "-jar", "/app/api-gateway.jar"]
```

**Multi-stage Build:**
- **Stage 1:** Build with gradle:8-jdk21-alpine
- **Stage 2:** Runtime with openjdk:21-jdk
- **Exposed Port:** 9000

---

## Step 8: Update settings.gradle.kts

**Location:** `microservices-parent/settings.gradle.kts`

**Before:**
```kotlin
rootProject.name = "microservices-parent"
include("product-service")
include("order-service")
include("inventory-service")
```

**After:**
```kotlin
rootProject.name = "microservices-parent"
include("product-service")
include("order-service")
include("inventory-service")
include("api-gateway")
```

**Reload Gradle.**

---

## Step 9: Update docker-compose.yml

**Location:** `microservices-parent/docker-compose.yml`

Add api-gateway service at the **very beginning** of services section (before mongodb):

```yaml
  api-gateway:
    image: api-gateway
    ports:
      - "9000:9000"
    build:
      context: ./api-gateway
      dockerfile: ./Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: docker
    container_name: api-gateway
    networks:
      - microservices-network
```

**Complete docker-compose.yml:**

```yaml
version: '3.8'

services:

  api-gateway:              # ADD THIS SERVICE FIRST
    image: api-gateway
    ports:
      - "9000:9000"
    build:
      context: ./api-gateway
      dockerfile: ./Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: docker
    container_name: api-gateway
    networks:
      - microservices-network

  mongodb:
    image: mongo:latest
    container_name: mongodb
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
    volumes:
      - mongo-data:/data/db
      - ./init/mongo/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
    command: mongod --auth
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    ports:
      - "8081:8081"
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=admin
      - ME_CONFIG_MONGODB_ADMINPASSWORD=password
      - ME_CONFIG_MONGODB_SERVER=mongodb
      - ME_CONFIG_BASICAUTH_USERNAME=admin
      - ME_CONFIG_BASICAUTH_PASSWORD=password
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - microservices-network

  postgres:
    image: postgres:latest
    container_name: postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: order-service
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql
      - ./init/postgres/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d order-service"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-inventory:
    image: postgres:latest
    container_name: postgres-inventory
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: inventory-service
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      PGDATA: /data/postgres
    volumes:
      - postgres-inventory-data:/data/postgres
    networks:
      - microservices-network

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

  redis:
    image: redis:latest
    ports:
      - "6379:6379"
    volumes:
      - ./init/redis/redis.conf:/usr/local/etc/redis/redis.conf
      - redis-data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    container_name: redis
    networks:
      - microservices-network

  redis-insight:
    image: redislabs/redisinsight:1.14.0
    ports:
      - "8001:8001"
    container_name: redis-insight
    depends_on:
      - redis
    networks:
      - microservices-network

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

  inventory-service:
    image: inventory-service
    ports:
      - "8083:8083"
    build:
      context: ./inventory-service
      dockerfile: ./Dockerfile
    container_name: inventory-service
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-inventory/inventory-service
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: none
    depends_on:
      - postgres-inventory
    networks:
      - microservices-network

networks:
  microservices-network:
    driver: bridge

volumes:
  mongo-data:
    driver: local
  postgres-data:
    driver: local
  postgres-inventory-data:
    driver: local
  pgadmin-data:
    driver: local
  redis-data:
    driver: local
```

---

## Step 10: Test Locally

### 10.1 Build Project

```bash
cd microservices-parent
./gradlew clean build
```

**Expected:**
```
BUILD SUCCESSFUL
```

### 10.2 Start Backend Services

**Terminal 1 - Product Service:**
```bash
cd product-service
./gradlew bootRun
```

Wait for: `Started ProductServiceApplication`

**Terminal 2 - Order Service:**
```bash
cd order-service
./gradlew bootRun
```

Wait for: `Started OrderServiceApplication`

### 10.3 Start API Gateway

**Terminal 3 - API Gateway:**
```bash
cd api-gateway
./gradlew bootRun
```

**Expected logs:**
```
Initializing product service route with URL: http://localhost:8084
Initializing order service route with URL: http://localhost:8082
Started ApiGatewayApplication on port 9000
```

### 10.4 Test with Postman

#### Test 1: Get Products Through Gateway

**Request:**
- Method: `GET`
- URL: `http://localhost:9000/api/product`

**Expected Response:**
- Status: `200 OK`
- Body: JSON array of products

#### Test 2: Create Product Through Gateway

**Request:**
- Method: `POST`
- URL: `http://localhost:9000/api/product`
- Headers: `Content-Type: application/json`
- Body:
```json
{
  "name": "Samsung TV",
  "description": "Samsung TV - 2025",
  "price": 2000.00
}
```

**Expected Response:**
- Status: `201 Created`
- Body: Product object with ID

#### Test 3: Place Order Through Gateway

**Request:**
- Method: `POST`
- URL: `http://localhost:9000/api/order`
- Headers: `Content-Type: application/json`
- Body:
```json
{
  "orderLineItemDtoList": [
    {
      "skuCode": "SKU001",
      "price": 100.00,
      "quantity": 2
    }
  ]
}
```

**Expected Response:**
- Status: `201 Created`
- Body: `Order Placed Successfully`

### 10.5 Verify Gateway Logs

Check Terminal 3 for routing logs:

```
Received request for product service: http://localhost:9000/api/product
Response status: 200 OK
```

---

## Step 11: Test with Docker Compose

### 11.1 Stop Local Services

Stop all three terminals (Ctrl+C).

### 11.2 Build Docker Images

```bash
cd microservices-parent
docker-compose build
```

**Expected:**
```
Successfully built api-gateway
Successfully built product-service
Successfully built order-service
Successfully built inventory-service
```

### 11.3 Start All Services

```bash
docker-compose up -d
```

### 11.4 Verify Services Running

```bash
docker-compose ps
```

**Expected:**
```
NAME               STATUS
api-gateway        Up
product-service    Up
order-service      Up
inventory-service  Up
mongodb            Up
postgres           Up
postgres-inventory Up
redis              Up
... (other services)
```

### 11.5 Check Gateway Logs

```bash
docker logs api-gateway
```

**Expected:**
```
Initializing product service route with URL: http://product-service:8084
Initializing order service route with URL: http://order-service:8082
Started ApiGatewayApplication
```

### 11.6 Test with Postman

Repeat tests from Step 10.4, same endpoints:
- `GET http://localhost:9000/api/product`
- `POST http://localhost:9000/api/product`
- `POST http://localhost:9000/api/order`

All should work identically.

### 11.7 Test with cURL (Optional)

```bash
# Get products
curl http://localhost:9000/api/product

# Create product
curl -X POST http://localhost:9000/api/product \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Samsung TV",
    "description": "Samsung TV - 2025",
    "price": 2000.00
  }'

# Place order
curl -X POST http://localhost:9000/api/order \
  -H "Content-Type: application/json" \
  -d '{
    "orderLineItemDtoList": [
      {
        "skuCode": "SKU001",
        "price": 100.00,
        "quantity": 2
      }
    ]
  }'
```

---

## Troubleshooting

### Issue 1: Gateway Returns 404

**Problem:** `GET http://localhost:9000/api/product` returns 404

**Solution:**
1. Verify Routes.java has correct path: `RequestPredicates.path("/api/product")`
2. Check logs for route initialization
3. Verify backend services are running

### Issue 2: Connection Refused

**Problem:** Gateway logs show "Connection refused"

**Solution:**
1. Verify product-service and order-service are running
2. Check service URLs in application properties
3. For Docker: verify all services on same network

### Issue 3: Port 9000 Already in Use

**Problem:** `Port 9000 is already in use`

**Solution:**
```bash
# Find process using port 9000
lsof -ti:9000

# Kill the process
lsof -ti:9000 | xargs kill -9
```

### Issue 4: Gradle Dependency Error

**Problem:** Cannot resolve spring-cloud-starter-gateway-server-webmvc

**Solution:**
1. Verify Spring Cloud version: `extra["springCloudVersion"] = "2025.0.0"`
2. Reload Gradle: `./gradlew clean build --refresh-dependencies`

---

## Summary

### What You Created:
- ✅ api-gateway module with Spring Cloud Gateway MVC
- ✅ Routes configuration for product and order services
- ✅ Local and Docker configuration files
- ✅ Multi-stage Dockerfile
- ✅ Updated docker-compose.yml

### Dependencies Added:
- ✅ spring-cloud-starter-gateway-server-webmvc
- ✅ spring-boot-starter-web
- ✅ Spring Boot Actuator
- ✅ Lombok
- ✅ DevTools

### Verified:
- ✅ Gateway routes to product-service (port 8084)
- ✅ Gateway routes to order-service (port 8082)
- ✅ Works locally and in Docker
- ✅ All services communicate through gateway

### Architecture:
```
Before:
Client → product-service:8084
Client → order-service:8082

After:
Client → api-gateway:9000 → product-service:8084
                          → order-service:8082
```

### Next Steps (Optional):
- ⏳ Add authentication/authorization
- ⏳ Add rate limiting
- ⏳ Add request/response logging
- ⏳ Add circuit breakers
- ⏳ Add load balancing

---

## Success Criteria

Your implementation is successful when:

- ✅ api-gateway starts without errors
- ✅ `GET http://localhost:9000/api/product` returns products
- ✅ `POST http://localhost:9000/api/product` creates product
- ✅ `POST http://localhost:9000/api/order` places order
- ✅ Gateway logs show routing messages
- ✅ All services run in Docker Compose
- ✅ Direct service access still works (8084, 8082)
- ✅ No errors in any service logs

---

**Total Services:** 11 containers (api-gateway, product-service, order-service, inventory-service, mongodb, mongo-express, postgres, postgres-inventory, pgadmin, redis, redis-insight)

**Ports:**
- API Gateway: 9000
- Product Service: 8084
- Order Service: 8082
- Inventory Service: 8083
