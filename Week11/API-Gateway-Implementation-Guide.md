# API Gateway Implementation Guide

## Overview

This guide provides step-by-step instructions to transform your existing microservices project (`comp3095_fall2025_11am`) to include an API Gateway service, matching the architecture of the `comp3095-s2025` (branch week5.1) reference implementation.

The API Gateway will act as a single entry point for all client requests, routing them to the appropriate microservices (Product Service and Order Service).

---

## Prerequisites

Before starting, ensure you have:

1. **Existing microservices project** at `/Users/maziar/temp/comp3095_fall2025_11am/microservices-parent/`
   - `product-service` (running on port 8084)
   - `order-service` (running on port 8082)
   - `inventory-service` (running on port 8083)

2. **Development tools:**
   - Java 21 JDK
   - Gradle 8.x
   - Docker and Docker Compose
   - Your preferred IDE (IntelliJ IDEA, Eclipse, or VS Code)

3. **Reference implementation** at `/Users/maziar/temp/comp3095-s2025/microservices-parent/` (branch week5.1)

---

## Architecture Overview

```
Client Requests
      ↓
API Gateway (Port 9000)
      ↓
   ┌──┴──┐
   ↓     ↓
Product  Order
Service  Service
(8084)   (8082)
```

---

## Step-by-Step Implementation

### **Step 1: Create API Gateway Module Directory Structure**

Create the following directory structure inside your `microservices-parent/` folder:

```
microservices-parent/
└── api-gateway/
    └── src/
        ├── main/
        │   ├── java/
        │   │   └── ca/
        │   │       └── gbc/
        │   │           └── apigateway/
        │   │               ├── ApiGatewayApplication.java
        │   │               └── routes/
        │   │                   └── Routes.java
        │   └── resources/
        │       ├── application.properties
        │       └── application-docker.properties
        └── test/
            └── java/
                └── ca/
                    └── gbc/
                        └── apigateway/
```

**Commands to create the directory structure:**

```bash
cd /Users/maziar/temp/comp3095_fall2025_11am/microservices-parent/
mkdir -p api-gateway/src/main/java/ca/gbc/apigateway/routes
mkdir -p api-gateway/src/main/resources
mkdir -p api-gateway/src/test/java/ca/gbc/apigateway
```

**Important Note on Package Naming:**
- The reference project (`comp3095-s2025` week5.1) uses package name: `ca.gbc.apigateway`
- Your current 11am project uses: `ca.gbc.comp3095.productservice`, `ca.gbc.comp3095.orderservice`
- For the API Gateway, use the simpler package structure: `ca.gbc.apigateway`
- You do **NOT** need to refactor your existing services' package names

---

### **Step 2: Create `build.gradle.kts` for API Gateway**

Create a new file at `api-gateway/build.gradle.kts`:

**File:** `microservices-parent/api-gateway/build.gradle.kts`

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.4.4"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "ca.gbc"
version = "0.0.1-SNAPSHOT"

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

extra["springCloudVersion"] = "2024.0.1"

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.cloud:spring-cloud-starter-gateway-mvc")
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
- `spring-cloud-starter-gateway-mvc`: Provides Spring Cloud Gateway MVC functionality
- Spring Cloud version: `2024.0.1`
- Spring Boot version: `3.4.4`

---

### **Step 3: Create Main Application Class**

Create the main Spring Boot application class for the API Gateway.

**File:** `microservices-parent/api-gateway/src/main/java/ca/gbc/apigateway/ApiGatewayApplication.java`

```java
package ca.gbc.apigateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ApiGatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }

}
```

This is a standard Spring Boot application class with no special configuration needed.

---

### **Step 4: Create Routes Configuration Class**

Create the Routes class that defines routing rules for the API Gateway.

**File:** `microservices-parent/api-gateway/src/main/java/ca/gbc/apigateway/routes/Routes.java`

```java
package ca.gbc.apigateway.routes;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gateway.server.mvc.handler.GatewayRouterFunctions;
import org.springframework.cloud.gateway.server.mvc.handler.HandlerFunctions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.function.*;

/**
 * Configuration class for defining API Gateway routes.
 * This class sets up routing rules to forward requests to the product and order microservices
 * using Spring Cloud Gateway's functional routing approach.
 */
@Configuration
@Slf4j
public class Routes {

    // URL of the product service, injected from application properties
    @Value("${services.product-url}")
    private String productServiceUrl;

    // URL of the order service, injected from application properties
    @Value("${services.order-url}")
    private String orderServiceUrl;

    /**
     * Defines the routing configuration for the product service.
     * Routes requests with the path "/api/product" to the product service URL.
     *
     * @return RouterFunction that handles product service requests
     */
    @Bean
    public RouterFunction<ServerResponse> productServiceRoute() {
        // Log the initialization of the product service route
        log.info("Initializing product service route with URL: {}", productServiceUrl);

        // Configure route for product service
        return GatewayRouterFunctions.route("product_service")
                .route(
                        // Match requests with path starting with "/api/product"
                        RequestPredicates.path("/api/product"),
                        request -> {
                            // Log incoming request details
                            log.info("Received request for product service: {}", request.uri());
                            try {
                                // Forward request to product service and get response
                                ServerResponse response = HandlerFunctions.http(productServiceUrl).handle(request);
                                // Log response status
                                log.info("Response status: {}", response.statusCode());
                                return response;
                            } catch (Exception e) {
                                // Log any errors during request processing
                                log.error("Error occurred while routing request: {}", e.getMessage(), e);
                                // Return 500 error response to client
                                return ServerResponse.status(500).body("An error occurred");
                            }
                        })
                .build();
    }

    /**
     * Defines the routing configuration for the order service.
     * Routes requests with the path "/api/order" to the order service URL.
     *
     * @return RouterFunction that handles order service requests
     */
    @Bean
    public RouterFunction<ServerResponse> orderServiceRoute() {
        // Log the initialization of the order service route
        log.info("Initializing order service route with URL: {}", orderServiceUrl);

        // Configure route for order service
        return GatewayRouterFunctions.route("order_service")
                .route(
                        // Match requests with path starting with "/api/order"
                        RequestPredicates.path("/api/order"),
                        request -> {
                            // Log incoming request details
                            log.info("Received request for order service: {}", request.uri());
                            try {
                                // Forward request to order service and get response
                                ServerResponse response = HandlerFunctions.http(orderServiceUrl).handle(request);
                                // Log response status
                                log.info("Response status: {}", response.statusCode());
                                return response;
                            } catch (Exception e) {
                                // Log any errors during request processing
                                log.error("Error occurred while routing request: {}", e.getMessage(), e);
                                // Return 500 error response to client
                                return ServerResponse.status(500).body("An error occurred");
                            }
                        })
                .build();
    }
}
```

**Key Concepts:**
- `@Configuration`: Marks this class as a source of Bean definitions
- `@Value`: Injects service URLs from application.properties
- `GatewayRouterFunctions.route()`: Creates a routing function with a specific route ID
- `RequestPredicates.path()`: Matches incoming requests based on path pattern
- `HandlerFunctions.http()`: Forwards requests to the target service URL
- Error handling with try-catch returns 500 status on errors

---

### **Step 5: Create Application Properties Files**

#### **5.1 Create `application.properties` (Local Development)**

**File:** `microservices-parent/api-gateway/src/main/resources/application.properties`

```properties
spring.application.name=api-gateway

server.port=9000

#services
services.product-url=http://localhost:8084
services.order-url=http://localhost:8082
```

**Explanation:**
- `spring.application.name`: Identifies the application as "api-gateway"
- `server.port=9000`: API Gateway runs on port 9000
- `services.product-url`: Points to product-service running locally on port 8084
- `services.order-url`: Points to order-service running locally on port 8082

---

#### **5.2 Create `application-docker.properties` (Docker Environment)**

**File:** `microservices-parent/api-gateway/src/main/resources/application-docker.properties`

```properties
spring.application.name=api-gateway

server.port=9000

#services
services.product-url=http://product-service:8084
services.order-url=http://order-service:8082
```

**Explanation:**
- Uses Docker service names (`product-service`, `order-service`) instead of `localhost`
- Docker Compose allows containers to communicate using service names as hostnames
- This file is activated when `SPRING_PROFILES_ACTIVE=docker` environment variable is set

---

### **Step 6: Create Dockerfile for API Gateway**

Create a multi-stage Dockerfile to build and package the API Gateway.

**File:** `microservices-parent/api-gateway/Dockerfile`

```dockerfile
# ------------------
#  Build Stage
# ------------------

# Start from the Gradle 8 image with JDK 22. This image provides both Gradle and JDK 22.
# We use this stage to build the application within the container.
FROM gradle:8-jdk21-alpine AS builder

# Copy the application files from the host machine to the image filesystem.
# We use the '--chown=gradle:gradle' flag to ensure proper file permissions for the Gradle user.
COPY --chown=gradle:gradle . /home/gradle/src

# Set the working directory inside the container to /home/gradle/src.
# All future commands will be executed from this directory.
WORKDIR /home/gradle/src

# Run the Gradle build inside the container, skipping the tests (-x test).
# This command compiles the code, resolves dependencies, and packages the application as a .jar file.
# Note: The command applies to the image only, not to the host machine.
RUN gradle build -x test

# ------------------
#  Package Stage
# ------------------

# Start from a lightweight OpenJDK 22 Alpine image. This will be our runtime image.
# Alpine images are much smaller, which helps keep the final image size down.
FROM openjdk:21-jdk

# Create a directory inside the container where the application will be stored.
# This directory is where we will place the packaged .jar file built in the previous stage.
RUN mkdir /app

# Copy the built .jar file from the build stage to the /app directory in the final image.
# We use the '--from=builder' instruction to reference the "builder" stage.
COPY --from=builder /home/gradle/src/build/libs/*.jar /app/api-gateway.jar

# Expose port 8084 to allow communication with the containerized application.
# EXPOSE does not actually make the port accessible to the host machine; it's documentation for the image.
EXPOSE 9000

# The ENTRYPOINT instruction defines the command to run when the container starts.
# In this case, we are telling Docker to run the Java command with the packaged JAR file.
ENTRYPOINT ["java", "-jar", "/app/api-gateway.jar"]
```

**Key Points:**
- **Multi-stage build**: Separates build and runtime environments
- **Builder stage**: Uses `gradle:8-jdk21-alpine` to compile the application
- **Runtime stage**: Uses `openjdk:21-jdk` to run the application (smaller image)
- **Port 9000**: Exposed for API Gateway service
- **JAR naming**: The built JAR is copied to `/app/api-gateway.jar`

---

### **Step 7: Update `settings.gradle.kts`**

Add the `api-gateway` module to the root Gradle settings file.

**File:** `microservices-parent/settings.gradle.kts`

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

Or use a more concise format:

```kotlin
rootProject.name = "microservices-parent"

include("product-service", "order-service", "inventory-service", "api-gateway")
```

This tells Gradle to include the `api-gateway` module in the multi-module project.

---

### **Step 8: Update `docker-compose.yml`**

Add the API Gateway service to your Docker Compose configuration.

**File:** `microservices-parent/docker-compose.yml`

Add the following service definition **at the beginning** of the `services:` section (after the comments but before other services):

```yaml
  api-gateway:
    image: api-gateway
    #image: ssantilli/api-gateway:latest
    #pull_policy: always
    ports:
      - "9000:9000"
    build: # Build the image using the Dockerfile in the current directory.
      context: ./api-gateway                   # The context provides the build context.
      dockerfile: ./Dockerfile                 # The dockerfile specifies the name of the Dockerfile to use.
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_APPLICATION_JSON={"logging":{"level":{"root":"INFO","ca.gbc.apigateway":"DEBUG"}}}:
    container_name: api-gateway
    networks:
      - spring
```

**Key Configuration:**
- **Ports**: Maps port 9000 on host to port 9000 in container
- **Build context**: Points to `./api-gateway` directory
- **Environment**:
  - `SPRING_PROFILES_ACTIVE: docker` - Activates the Docker profile (uses `application-docker.properties`)
  - Logging configuration sets DEBUG level for `ca.gbc.apigateway` package
- **Networks**: Connects to the `spring` network (same as other services)

**Full docker-compose.yml structure:**
```yaml
services:

  api-gateway:
    # ... (configuration shown above)

  redis:
    # ... (existing configuration)

  redis-insight:
    # ... (existing configuration)

  inventory-service:
    # ... (existing configuration)

  order-service:
    # ... (existing configuration)

  product-service:
    # ... (existing configuration)

  mongodb:
    # ... (existing configuration)

  mongo-express:
    # ... (existing configuration)

  postgres-inventory:
    # ... (existing configuration)

  postgres-order:
    # ... (existing configuration)

  pgpadmin:
    # ... (existing configuration)

volumes:
  # ... (existing volumes)

networks:
  # ... (existing networks)
```

---

## Step 9: Build and Test the API Gateway

### **9.1 Build the Project**

From the `microservices-parent/` directory:

```bash
cd /Users/maziar/temp/comp3095_fall2025_11am/microservices-parent/
./gradlew clean build
```

This will:
- Build all modules including the new `api-gateway`
- Run tests
- Generate JAR files

### **9.2 Build Docker Images**

Build all Docker images including the API Gateway:

```bash
docker-compose build
```

Or to force rebuild:

```bash
docker-compose build --no-cache
```

### **9.3 Start All Services**

```bash
docker-compose up -d
```

This starts all services in detached mode.

### **9.4 Verify Services Are Running**

Check all containers are running:

```bash
docker-compose ps
```

You should see all services including `api-gateway` with status "Up".

Check the API Gateway logs:

```bash
docker logs api-gateway
```

You should see log messages like:
```
Initializing product service route with URL: http://product-service:8084
Initializing order service route with URL: http://order-service:8082
```

---

## Step 10: Test with Postman (or cURL)

### **10.1 Test Product Service Through API Gateway**

**Without API Gateway (Direct):**
```
GET http://localhost:8084/api/product
```

**With API Gateway:**
```
GET http://localhost:9000/api/product
```

**Postman Configuration:**
- Method: `GET`
- URL: `http://localhost:9000/api/product`
- Headers: None required
- Body: None

**Expected Response:**
- Status: `200 OK`
- Body: JSON array of products

---

### **10.2 Test Creating a Product Through API Gateway**

**Without API Gateway (Direct):**
```
POST http://localhost:8084/api/product
```

**With API Gateway:**
```
POST http://localhost:9000/api/product
```

**Postman Configuration:**
- Method: `POST`
- URL: `http://localhost:9000/api/product`
- Headers:
  - `Content-Type: application/json`
- Body (raw JSON):
```json
{
  "name": "Samsung TV",
  "description": "Samsung TV - 2025 Model",
  "price": 2000.00
}
```

**Expected Response:**
- Status: `201 Created`
- Body: JSON object with created product details including `id`

---

### **10.3 Test Order Service Through API Gateway**

**Without API Gateway (Direct):**
```
POST http://localhost:8082/api/order
```

**With API Gateway:**
```
POST http://localhost:9000/api/order
```

**Postman Configuration:**
- Method: `POST`
- URL: `http://localhost:9000/api/order`
- Headers:
  - `Content-Type: application/json`
- Body (raw JSON):
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

---

### **10.4 Using cURL (Alternative to Postman)**

**Test GET Products:**
```bash
curl -X GET http://localhost:9000/api/product
```

**Test POST Product:**
```bash
curl -X POST http://localhost:9000/api/product \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Samsung TV",
    "description": "Samsung TV - 2025 Model",
    "price": 2000.00
  }'
```

**Test POST Order:**
```bash
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

## Step 11: Verify Routing in Logs

Check the API Gateway logs to see the routing in action:

```bash
docker logs -f api-gateway
```

When you make a request, you should see log messages like:

```
INFO  c.g.apigateway.routes.Routes : Received request for product service: http://localhost:9000/api/product
INFO  c.g.apigateway.routes.Routes : Response status: 200 OK
```

This confirms that the API Gateway is successfully routing requests to the backend services.

---

## Troubleshooting

### **Issue 1: API Gateway Container Won't Start**

**Check logs:**
```bash
docker logs api-gateway
```

**Common causes:**
- Port 9000 already in use
- Service URLs misconfigured in `application-docker.properties`
- Build failed (check for compilation errors)

**Solution:**
- Stop any service using port 9000: `lsof -ti:9000 | xargs kill -9`
- Verify service names in docker-compose.yml match the URLs in application-docker.properties
- Rebuild: `docker-compose build api-gateway --no-cache`

---

### **Issue 2: 404 Not Found When Accessing Through Gateway**

**Symptoms:**
- Direct access to services works: `http://localhost:8084/api/product` ✅
- Gateway access fails: `http://localhost:9000/api/product` ❌

**Common causes:**
- Routes not properly configured
- Path predicates don't match

**Solution:**
- Check Routes.java has correct path: `RequestPredicates.path("/api/product")`
- Verify services are accessible from within gateway container
- Check network configuration in docker-compose.yml

---

### **Issue 3: Connection Refused to Backend Services**

**Symptoms:**
- Gateway logs show: "Connection refused" or "Unknown host"

**Common causes:**
- Backend services not running
- Wrong service URLs in application-docker.properties

**Solution:**
- Verify all services are up: `docker-compose ps`
- Check service names match container names
- Verify all services are on the same Docker network (`spring`)
- Test connectivity: `docker exec api-gateway ping product-service`

---

### **Issue 4: Build Fails with Dependency Resolution Error**

**Symptoms:**
```
Could not resolve org.springframework.cloud:spring-cloud-starter-gateway-mvc
```

**Solution:**
- Ensure Spring Cloud version is correctly set: `extra["springCloudVersion"] = "2024.0.1"`
- Verify dependency management block is present in build.gradle.kts
- Run: `./gradlew clean build --refresh-dependencies`

---

## Project Structure Summary

After completing all steps, your project structure should look like this:

```
microservices-parent/
├── api-gateway/
│   ├── build.gradle.kts
│   ├── Dockerfile
│   └── src/
│       ├── main/
│       │   ├── java/
│       │   │   └── ca/
│       │   │       └── gbc/
│       │   │           └── apigateway/
│       │   │               ├── ApiGatewayApplication.java
│       │   │               └── routes/
│       │   │                   └── Routes.java
│       │   └── resources/
│       │       ├── application.properties
│       │       └── application-docker.properties
│       └── test/
│           └── java/
│               └── ca/
│                   └── gbc/
│                       └── apigateway/
├── product-service/
│   └── ... (existing files)
├── order-service/
│   └── ... (existing files)
├── inventory-service/
│   └── ... (existing files)
├── docker-compose.yml (updated)
├── settings.gradle.kts (updated)
└── build.gradle.kts
```

---

## Key Differences Between Your Project and Reference

### **Package Naming**

| Service | Your 11am Project | Reference week5.1 |
|---------|------------------|-------------------|
| Product Service | `ca.gbc.comp3095.productservice` | `ca.gbc.productservice` |
| Order Service | `ca.gbc.comp3095.orderservice` | `ca.gbc.orderservice` |
| Inventory Service | `ca.gbc.comp3095.inventoryservice` | `ca.gbc.inventoryservice` |
| API Gateway | **N/A (new)** | `ca.gbc.apigateway` |

**Recommendation:**
- Keep your existing services with `ca.gbc.comp3095.*` package naming
- Use `ca.gbc.apigateway` for the new API Gateway service
- No refactoring needed for existing services

---

## Additional Enhancements (Optional)

### **1. Add Health Checks**

API Gateway already includes Spring Boot Actuator. Access health endpoint:

```
http://localhost:9000/actuator/health
```

### **2. Add CORS Configuration**

If you need to allow cross-origin requests, add to `ApiGatewayApplication.java`:

```java
@Bean
public CorsFilter corsFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("*"));
    config.setAllowedMethods(List.of("*"));
    config.setAllowedHeaders(List.of("*"));

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);

    return new CorsFilter(source);
}
```

### **3. Add Rate Limiting**

Consider adding rate limiting to prevent abuse of your API.

### **4. Add Authentication/Authorization**

Implement Spring Security to add authentication at the gateway level.

---

## Testing Checklist

- [ ] All modules build successfully: `./gradlew clean build`
- [ ] Docker images build successfully: `docker-compose build`
- [ ] All containers start: `docker-compose up -d`
- [ ] API Gateway container is running: `docker ps | grep api-gateway`
- [ ] Can access product service through gateway: `GET http://localhost:9000/api/product`
- [ ] Can create product through gateway: `POST http://localhost:9000/api/product`
- [ ] Can access order service through gateway: `POST http://localhost:9000/api/order`
- [ ] Gateway logs show routing messages: `docker logs api-gateway`
- [ ] Health endpoint accessible: `GET http://localhost:9000/actuator/health`

---

## Conclusion

You have successfully added an API Gateway to your microservices architecture!

**What you've accomplished:**
1. Created a new `api-gateway` Spring Boot module
2. Configured routing to forward requests to Product and Order services
3. Set up Docker containerization for the API Gateway
4. Integrated the API Gateway into your docker-compose setup
5. Tested the complete flow from client → API Gateway → Backend Services

**Next Steps:**
- Consider adding more routes for the inventory service
- Implement security and authentication at the gateway level
- Add monitoring and logging aggregation
- Explore advanced gateway features like circuit breakers, retry logic, and load balancing

---

## References

- [Spring Cloud Gateway MVC Documentation](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/)
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- COMP3095 - In-Class Instructions 5.1.pdf

---

**Document Version:** 1.0
**Last Updated:** November 10, 2025
**Author:** Based on COMP3095 Course Materials
