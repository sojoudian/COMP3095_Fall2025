# API Documentation with Swagger/OpenAPI

## Overview

Integrate Swagger/OpenAPI documentation into your microservices using SpringDoc. This provides interactive API documentation accessible through web UI and JSON endpoints.

**Time:** 60-90 minutes

---

## Prerequisites

- ✅ All microservices running (product-service, order-service, inventory-service)
- ✅ Basic understanding of REST APIs
- ✅ Gradle project setup

---

## Background

API documentation is essential for microservices architectures. **Swagger** and **OpenAPI** are industry-standard tools that automatically generate, visualize, and provide interactive testing for REST APIs.

**SpringDoc** is a library that integrates Swagger/OpenAPI with Spring Boot applications, automatically generating documentation from your controllers, models, and annotations.

### Benefits

- **Auto-generated** documentation from code
- **Interactive** API testing through Swagger UI
- **Standardized** OpenAPI specification (JSON/YAML)
- **Easy onboarding** for new developers and API consumers

---

## Step 1: Add Swagger Dependencies to product-service

### 1.1 Update build.gradle.kts

**Location:** `product-service/build.gradle.kts`

Add SpringDoc OpenAPI dependencies:

```kotlin
dependencies {
    // Existing dependencies...

    // Week 6.1 - Swagger/OpenAPI Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")
}
```

**Dependencies Explained:**

- `springdoc-openapi-starter-webmvc-ui`: Provides Swagger UI web interface
- `springdoc-openapi-starter-webmvc-api`: Provides OpenAPI JSON/YAML endpoints

### 1.2 Reload Gradle Project

1. In IntelliJ IDEA, right-click on project root
2. Select **Gradle** → **Reload Gradle Project**
3. Wait for dependencies to download

Or use terminal:

```bash
cd product-service
./gradlew build --refresh-dependencies
```

---

## Step 2: Configure Swagger UI Path

### 2.1 Update application.properties

**Location:** `product-service/src/main/resources/application.properties`

Add Swagger configuration:

```properties
spring.application.name=product-service
product-service.version=v1.0

server.port=8084

# Existing MongoDB and Redis config...

# Week 6.1 - Swagger Documentation
# Swagger UI accessible at: http://localhost:8084/swagger-ui
springdoc.swagger-ui.path=/swagger-ui
# OpenAPI JSON accessible at: http://localhost:8084/api-docs
springdoc.api-docs.path=/api-docs
```

**Properties Explained:**

- `springdoc.swagger-ui.path`: Custom URL path for Swagger UI interface
- `springdoc.api-docs.path`: Custom URL path for OpenAPI JSON specification

---

## Step 3: Test Basic Documentation

### 3.1 Run product-service

```bash
cd product-service
./gradlew bootRun
```

### 3.2 Access Swagger UI

Open browser and navigate to:

```
http://localhost:8084/swagger-ui/index.html
```

or

```
http://localhost:8084/swagger-ui
```

You should see auto-generated API documentation showing:
- **product-controller** with all endpoints (GET, POST, PUT, DELETE)
- **Schemas** for ProductRequest and ProductResponse
- Interactive "Try it out" buttons for testing endpoints

### 3.3 Explore Documentation

The Swagger UI displays:

1. **Servers**: Base URL for API (e.g., `http://localhost:8084`)
2. **product-controller**: All REST endpoints
   - `GET /api/product` - Get all products
   - `POST /api/product` - Create product
   - `PUT /api/product/{productId}` - Update product
   - `DELETE /api/product/{productId}` - Delete product
3. **Schemas**: Data transfer objects (DTOs)
   - ProductRequest (id, name, description, price)
   - ProductResponse (id, name, description, price)

---

## Step 4: Customize API Documentation

### 4.1 Create config Package

Create new package for configuration classes:

1. Navigate to `product-service/src/main/java/ca/gbc/productservice`
2. Right-click on `ca.gbc.productservice`
3. Select **New** → **Package**
4. Name: `config`
5. Click **OK**

### 4.2 Create OpenAPIConfig Class

Create configuration class:

1. Right-click on `config` package
2. Select **New** → **Java Class**
3. Name: `OpenAPIConfig`
4. Click **OK**

**Location:** `product-service/src/main/java/ca/gbc/productservice/config/OpenAPIConfig.java`

```java
package ca.gbc.productservice.config;

import io.swagger.v3.oas.models.ExternalDocumentation;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenAPIConfig {

    @Value("${product-service.version}")
    private String version;

    @Bean
    public OpenAPI productServiceAPI() {

        return new OpenAPI()
                .info(new Info()
                        .title("Product Service API")
                        .description("This is the REST API for Product Service")
                        .version(version)
                        .license(new License().name("Apache 2.0")))
                .externalDocs(new ExternalDocumentation()
                        .description("Product Service Confluence Documentation")
                        .url("https://mycompany.ca/product-service/docs"));
    }
}
```

**Configuration Breakdown:**

- `@Configuration`: Marks class as Spring configuration
- `@Value`: Injects version from application.properties
- `@Bean`: Creates OpenAPI bean for Spring context
- `.title()`: Sets API title displayed in Swagger UI
- `.description()`: Provides API description
- `.version()`: Uses parameterized version from properties
- `.license()`: Specifies API license (Apache 2.0)
- `.externalDocs()`: Links to external documentation

### 4.3 Verify Configuration

1. Restart product-service
2. Refresh Swagger UI: `http://localhost:8084/swagger-ui/index.html`

You should now see:
- **Title**: "Product Service API" (instead of default)
- **Description**: "This is the REST API for Product Service"
- **Version**: "v1.0"
- **License**: "Apache 2.0"
- **External Link**: "Product Service Confluence Documentation"

---

## Step 5: Access OpenAPI JSON Specification

### 5.1 Access API Docs Endpoint

Open browser and navigate to:

```
http://localhost:8084/api-docs
```

This returns the OpenAPI specification in JSON format.

**Example Response:**

```json
{
  "openapi": "3.0.1",
  "info": {
    "title": "Product Service API",
    "description": "This is the REST API for Product Service",
    "license": {
      "name": "Apache 2.0"
    },
    "version": "v1.0"
  },
  "externalDocs": {
    "description": "Product Service Confluence Documentation",
    "url": "https://mycompany.ca/product-service/docs"
  },
  "servers": [
    {
      "url": "http://localhost:8084",
      "description": "Generated server url"
    }
  ],
  "paths": {
    "/api/product/{productId}": {
      "put": { ... },
      "delete": { ... }
    },
    "/api/product": {
      "get": { ... },
      "post": { ... }
    }
  },
  "components": {
    "schemas": {
      "ProductRequest": { ... },
      "ProductResponse": { ... }
    }
  }
}
```

**Use Cases:**

- Import into Postman collections
- Generate client SDKs
- Automated API testing tools
- API gateway integration

---

## Step 6: Add Documentation to order-service

### 6.1 Update build.gradle.kts

**Location:** `order-service/build.gradle.kts`

Add SpringDoc dependencies:

```kotlin
dependencies {
    // Existing dependencies...

    // Week 6.1 - Swagger/OpenAPI Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")
}
```

Reload Gradle project.

### 6.2 Update application.properties

**Location:** `order-service/src/main/resources/application.properties`

Add configuration:

```properties
spring.application.name=order-service
order-service.version=v1.0

server.port=8082

# Existing PostgreSQL config...

# Week 6.1 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs
```

### 6.3 Create config Package

1. Navigate to `order-service/src/main/java/ca/gbc/orderservice`
2. Right-click on `ca.gbc.orderservice`
3. Select **New** → **Package**
4. Name: `config`
5. Click **OK**

### 6.4 Create OpenAPIConfig Class

1. Right-click on `config` package
2. Select **New** → **Java Class**
3. Name: `OpenAPIConfig`
4. Click **OK**

**Location:** `order-service/src/main/java/ca/gbc/orderservice/config/OpenAPIConfig.java`

```java
package ca.gbc.orderservice.config;

import io.swagger.v3.oas.models.ExternalDocumentation;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenAPIConfig {

    @Value("${order-service.version}")
    private String version;

    @Bean
    public OpenAPI orderServiceAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Order Service API")
                        .description("This is the REST API for Order Service")
                        .version(version)
                        .license(new License().name("Apache 2.0")))
                .externalDocs(new ExternalDocumentation()
                        .description("Order Service Confluence Documentation")
                        .url("https://mycompany.ca/order-service/docs"));
    }
}
```

### 6.5 Test order-service Documentation

Run order-service:

```bash
cd order-service
./gradlew bootRun
```

Access documentation:

- **Swagger UI**: `http://localhost:8082/swagger-ui/index.html`
- **OpenAPI JSON**: `http://localhost:8082/api-docs`

---

## Step 7: Add Documentation to inventory-service

### 7.1 Update build.gradle.kts

**Location:** `inventory-service/build.gradle.kts`

Add SpringDoc dependencies:

```kotlin
dependencies {
    // Existing dependencies...

    // Week 6.1 - Swagger/OpenAPI Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")
}
```

Reload Gradle project.

### 7.2 Update application.properties

**Location:** `inventory-service/src/main/resources/application.properties`

Add configuration:

```properties
spring.application.name=inventory-service
inventory-service.version=v1.0

server.port=8083

# Existing PostgreSQL config...

# Week 6.1 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs
```

### 7.3 Create config Package

1. Navigate to `inventory-service/src/main/java/ca/gbc/inventoryservice`
2. Right-click on `ca.gbc.inventoryservice`
3. Select **New** → **Package**
4. Name: `config`
5. Click **OK**

### 7.4 Create OpenAPIConfig Class

1. Right-click on `config` package
2. Select **New** → **Java Class**
3. Name: `OpenAPIConfig`
4. Click **OK**

**Location:** `inventory-service/src/main/java/ca/gbc/inventoryservice/config/OpenAPIConfig.java`

```java
package ca.gbc.inventoryservice.config;

import io.swagger.v3.oas.models.ExternalDocumentation;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenAPIConfig {

    @Value("${inventory-service.version}")
    private String version;

    @Bean
    public OpenAPI inventoryServiceAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Inventory Service API")
                        .description("This is the REST API for Inventory Service")
                        .version(version)
                        .license(new License().name("Apache 2.0")))
                .externalDocs(new ExternalDocumentation()
                        .description("Inventory Service Confluence Documentation")
                        .url("https://mycompany.ca/inventory-service/docs"));
    }
}
```

### 7.5 Test inventory-service Documentation

Run inventory-service:

```bash
cd inventory-service
./gradlew bootRun
```

Access documentation:

- **Swagger UI**: `http://localhost:8083/swagger-ui/index.html`
- **OpenAPI JSON**: `http://localhost:8083/api-docs`

---

## Step 8: Update Docker Configuration

### 8.1 Update application-docker.properties

Update Docker configuration for all three services to include Swagger properties.

**Location:** `product-service/src/main/resources/application-docker.properties`

```properties
spring.application.name=product-service
product-service.version=v1.0

server.port=8084

# MongoDB Docker configuration
spring.data.mongodb.host=mongodb
spring.data.mongodb.port=27017
spring.data.mongodb.database=product-service
spring.data.mongodb.username=admin
spring.data.mongodb.password=password
spring.data.mongodb.authentication-database=admin

# Redis Docker configuration
spring.data.redis.host=redis
spring.data.redis.port=6379
spring.data.redis.password=password
spring.cache.type=redis
spring.cache.redis.time-to-live=60s

# Week 6.1 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs
```

**Location:** `order-service/src/main/resources/application-docker.properties`

```properties
spring.application.name=order-service
order-service.version=v1.0

server.port=8082

# PostgreSQL Docker configuration
spring.datasource.url=jdbc:postgresql://postgres-order:5432/order-service
spring.datasource.username=admin
spring.datasource.password=password
spring.jpa.hibernate.ddl-auto=none

# Week 6.1 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs
```

**Location:** `inventory-service/src/main/resources/application-docker.properties`

```properties
spring.application.name=inventory-service
inventory-service.version=v1.0

server.port=8083

# PostgreSQL Docker configuration
spring.datasource.url=jdbc:postgresql://postgres-inventory:5432/inventory-service
spring.datasource.username=admin
spring.datasource.password=password
spring.jpa.hibernate.ddl-auto=none

# Week 6.1 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs
```

### 8.2 Rebuild Docker Containers

```bash
cd microservices-parent
docker-compose -p microservices-comp3095 -f docker-compose.yml up -d --build
```

Wait for containers to start (~30 seconds).

### 8.3 Access Documentation in Docker

Access each service's documentation directly (not through API Gateway):

**Product Service:**
- Swagger UI: `http://localhost:8084/swagger-ui/index.html`
- OpenAPI JSON: `http://localhost:8084/api-docs`

**Order Service:**
- Swagger UI: `http://localhost:8082/swagger-ui/index.html`
- OpenAPI JSON: `http://localhost:8082/api-docs`

**Inventory Service:**
- Swagger UI: `http://localhost:8083/swagger-ui/index.html`
- OpenAPI JSON: `http://localhost:8083/api-docs`

**Note:** Accessing through API Gateway requires additional configuration (covered in a future lab).

---

## Step 9: Using Swagger UI

### 9.1 Explore Endpoints

In Swagger UI for any service:

1. **Expand** any endpoint (e.g., `GET /api/product`)
2. Click **Try it out** button
3. Click **Execute** button
4. View response with:
   - Response body (JSON)
   - Response code (200, 404, etc.)
   - Response headers

### 9.2 Test POST Endpoint

Example: Create a product via Swagger UI

1. Navigate to `http://localhost:8084/swagger-ui/index.html`
2. Find `POST /api/product` endpoint
3. Click **Try it out**
4. Enter request body:

```json
{
  "name": "Test Product",
  "description": "Created via Swagger UI",
  "price": 29.99
}
```

5. Click **Execute**
6. View response with created product ID

### 9.3 Test with Authentication

If Keycloak security is enabled (from previous labs):

1. Click **Authorize** button (top-right in Swagger UI)
2. Enter Bearer token obtained from Keycloak
3. Click **Authorize**
4. All subsequent requests will include authentication header

---

## Troubleshooting

### Swagger UI Not Loading

**Problem:** 404 error when accessing `/swagger-ui/index.html`

**Solution:**
1. Verify SpringDoc dependencies in `build.gradle.kts`
2. Check `springdoc.swagger-ui.path` in `application.properties`
3. Ensure application is running on correct port
4. Try accessing without `/index.html`: `http://localhost:8084/swagger-ui`

### Version Property Not Found

**Problem:** Error: "Could not resolve placeholder 'product-service.version'"

**Solution:**
1. Add version property to `application.properties`:
```properties
product-service.version=v1.0
```
2. Ensure property name matches `@Value("${product-service.version}")`

### OpenAPI JSON Returns Empty

**Problem:** `/api-docs` returns minimal JSON without endpoints

**Solution:**
1. Ensure controllers have proper `@RestController` annotation
2. Verify `@RequestMapping` paths are defined
3. Restart application to regenerate documentation

### Port Conflicts

**Problem:** Service won't start due to port already in use

**Solution:**
```bash
# Find process using port (e.g., 8084)
lsof -i :8084

# Kill process
kill -9 <PID>
```

---

## Summary

You now have:

- ✅ Swagger/OpenAPI documentation for all microservices
- ✅ Interactive Swagger UI for testing endpoints
- ✅ OpenAPI JSON specification for API consumption
- ✅ Customized API metadata (title, description, version)
- ✅ Docker-ready documentation configuration

**Key Benefits:**
- Auto-generated documentation from code
- Interactive API testing without Postman
- Standardized OpenAPI specification
- Easy API onboarding for developers
- Professional API presentation

**Next Steps:**
- Configure API Gateway to aggregate all service documentation
- Add detailed endpoint descriptions with `@Operation` annotations
- Document request/response schemas with `@Schema` annotations
- Add security schemes to Swagger UI
- Generate client SDKs from OpenAPI specification
