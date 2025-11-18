# API Documentation with Swagger/OpenAPI

## Overview

Integrate Swagger/OpenAPI documentation into your microservices using SpringDoc. This provides interactive API documentation accessible through web UI and JSON endpoints, making APIs more accessible and user-friendly for both internal and external developers.

**Time:** 60-90 minutes

---

## Prerequisites

- ✅ Completed Week 11-1 (Keycloak Client Credentials)
- ✅ Completed Week 11-2 (User Authentication with Authorization Code Flow)
- ✅ All microservices running (product-service, order-service, inventory-service)
- ✅ Docker and docker-compose installed
- ✅ Basic understanding of REST APIs

---

## Background

API documentation is essential for microservices architectures. **Swagger** and **OpenAPI** are industry-standard tools that automatically generate, visualize, and interact with REST APIs. This lab introduces **SpringDoc**, a library that integrates Swagger with Spring Boot applications, enabling students to create, configure, and expose API documentation easily.

### Benefits

- **Auto-generated** documentation from code
- **Interactive** API testing through Swagger UI
- **Standardized** OpenAPI specification (JSON/YAML)
- **Easy onboarding** for new developers and API consumers
- **Professional** API presentation

### What is SpringDoc?

**SpringDoc** is a library that automatically generates OpenAPI 3 specification for Spring Boot applications. It scans your controllers, models, and annotations to create comprehensive API documentation without additional coding effort.

---

## Step 1: Add Swagger Dependencies to product-service

### 1.1 Update build.gradle.kts

**Location:** `product-service/build.gradle.kts`

Add SpringDoc OpenAPI dependencies to the dependencies block:

```kotlin
dependencies {
    // Existing dependencies...

    // Week 12 - Swagger/OpenAPI Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")
}
```

**Dependencies Explained:**

- `springdoc-openapi-starter-webmvc-ui`: Provides Swagger UI web interface for interactive API testing
- `springdoc-openapi-starter-webmvc-api`: Provides OpenAPI JSON/YAML endpoints for API specification

### 1.2 Reload Gradle Project

**Option 1: Using IntelliJ IDEA**

1. In IntelliJ IDEA, right-click on project root
2. Select **Gradle** → **Reload Gradle Project**
3. Wait for dependencies to download

---

## Step 2: Configure Swagger UI Path

### 2.1 Update application.properties

**Location:** `product-service/src/main/resources/application.properties`

Add Swagger configuration properties:

```properties
# Week 11 - API version for documentation
product-service.version=v1.0

# Week 11 - Swagger Documentation
# Swagger UI accessible at: http://localhost:8084/swagger-ui
springdoc.swagger-ui.path=/swagger-ui
# OpenAPI JSON accessible at: http://localhost:8084/api-docs
springdoc.api-docs.path=/api-docs
```

**Properties Explained:**

- `product-service.version`: Version string used in API documentation (parameterized)
- `springdoc.swagger-ui.path`: Custom URL path for Swagger UI interface
- `springdoc.api-docs.path`: Custom URL path for OpenAPI JSON specification

### 2.2 Update application-docker.properties

**Location:** `product-service/src/main/resources/application-docker.properties`

Add the same Swagger configuration for Docker environment:

```properties
# Week 11 - API version for documentation
product-service.version=v1.0

# Week 11 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs
```

---

## Step 3: Customize API Documentation

### 3.1 Create OpenAPIConfig Class

Create configuration class for customizing API documentation:

1. Navigate to `product-service/src/main/java/ca/gbc/comp3095/productservice/config`
2. Right-click on `config` package
3. Select **New** → **Java Class**
4. Name: `OpenAPIConfig`
5. Click **OK**

**Location:** `product-service/src/main/java/ca/gbc/comp3095/productservice/config/OpenAPIConfig.java`

```java
package ca.gbc.comp3095.productservice.config;

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

- `@Configuration`: Marks class as Spring configuration component
- `@Value("${product-service.version}")`: Injects version from application.properties
- `@Bean`: Creates OpenAPI bean for Spring context
- `.title()`: Sets API title displayed in Swagger UI header
- `.description()`: Provides detailed API description
- `.version()`: Uses parameterized version from properties (allows version management in one place)
- `.license()`: Specifies API license information (Apache 2.0)
- `.externalDocs()`: Links to external documentation (Confluence, GitHub, etc.)

---

## Step 4: Add Documentation to order-service

### 4.1 Update build.gradle.kts

**Location:** `order-service/build.gradle.kts`

Add SpringDoc dependencies:

```kotlin
dependencies {
    // Existing dependencies...

    // Week 12 - Swagger/OpenAPI Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")
}
```

Reload Gradle project.

### 4.2 Update application.properties

**Location:** `order-service/src/main/resources/application.properties`

Add Swagger configuration:

```properties
# Week 11 - API version for documentation
order-service.version=v1.0

# Week 11 - Swagger Documentation
# Swagger UI accessible at: http://localhost:8082/swagger-ui
springdoc.swagger-ui.path=/swagger-ui
# OpenAPI JSON accessible at: http://localhost:8082/api-docs
springdoc.api-docs.path=/api-docs
```

### 4.3 Update application-docker.properties

**Location:** `order-service/src/main/resources/application-docker.properties`

Add the same Swagger configuration for Docker environment:

```properties
# Week 11 - API version for documentation
order-service.version=v1.0

# Week 11 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs
```

### 4.4 Create config Package

1. Navigate to `order-service/src/main/java/ca/gbc/comp3095/orderservice`
2. Right-click on `ca.gbc.comp3095.orderservice`
3. Select **New** → **Package**
4. Name: `config`
5. Click **OK**

### 4.5 Create OpenAPIConfig Class

1. Right-click on `config` package
2. Select **New** → **Java Class**
3. Name: `OpenAPIConfig`
4. Click **OK**

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/config/OpenAPIConfig.java`

```java
package ca.gbc.comp3095.orderservice.config;

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

---

## Step 5: Add Documentation to inventory-service

### 5.1 Update build.gradle.kts

**Location:** `inventory-service/build.gradle.kts`

Add SpringDoc dependencies:

```kotlin
dependencies {
    // Existing dependencies...

    // Week 12 - Swagger/OpenAPI Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")
}
```

Reload Gradle project.

### 5.2 Update application.properties

**Location:** `inventory-service/src/main/resources/application.properties`

Add Swagger configuration:

```properties
# Week 11 - API version for documentation
inventory-service.version=v1.0

# Week 11 - Swagger Documentation
# Swagger UI accessible at: http://localhost:8083/swagger-ui
springdoc.swagger-ui.path=/swagger-ui
# OpenAPI JSON accessible at: http://localhost:8083/api-docs
springdoc.api-docs.path=/api-docs
```

### 5.3 Update application-docker.properties

**Location:** `inventory-service/src/main/resources/application-docker.properties`

Add the same Swagger configuration for Docker environment:

```properties
# Week 11 - API version for documentation
inventory-service.version=v1.0

# Week 11 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs
```

### 5.4 Create config Package

1. Navigate to `inventory-service/src/main/java/ca/gbc/comp3095/inventoryservice`
2. Right-click on `ca.gbc.comp3095.inventoryservice`
3. Select **New** → **Package**
4. Name: `config`
5. Click **OK**

### 5.5 Create OpenAPIConfig Class

1. Right-click on `config` package
2. Select **New** → **Java Class**
3. Name: `OpenAPIConfig`
4. Click **OK**

**Location:** `inventory-service/src/main/java/ca/gbc/comp3095/inventoryservice/config/OpenAPIConfig.java`

```java
package ca.gbc.comp3095.inventoryservice.config;

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

---

## Step 6: Build All Services with Docker Compose

### 6.1 Rebuild Docker Containers

Rebuild all services with updated Swagger configuration:

```bash
cd microservices-parent
docker-compose -p microservices-parent -f docker-compose.yml up -d --build
```

Wait for containers to build and start (~60-90 seconds).

**Note:** The `--build` flag forces Docker to rebuild images with new dependencies.

### 6.2 Verify Containers are Running

```bash
docker ps
```

Expected containers:
- keycloak
- postgres-keycloak
- api-gateway
- product-service
- order-service
- inventory-service
- postgres-order
- postgres-inventory
- mongodb
- mongo-express
- redis
- redis-insight

### 6.3 Access Documentation in Docker

Access each service's Swagger documentation directly (not through API Gateway):

**Product Service:**
- Swagger UI: `http://localhost:8084/swagger-ui/index.html`
- OpenAPI JSON: `http://localhost:8084/api-docs`

**Order Service:**
- Swagger UI: `http://localhost:8082/swagger-ui/index.html`
- OpenAPI JSON: `http://localhost:8082/api-docs`

**Inventory Service:**
- Swagger UI: `http://localhost:8083/swagger-ui/index.html`
- OpenAPI JSON: `http://localhost:8083/api-docs`

**Important Note:**
Documentation is currently accessible only by directly accessing each service on its port. Accessing through API Gateway (`http://localhost:9000/api/product/swagger-ui`) requires additional configuration that will be covered in a future lab.

---

## Step 7: Using Swagger UI for Interactive Testing

### 7.1 Explore Endpoints

In Swagger UI for any service:

1. **Expand** any endpoint (e.g., `GET /api/product`)
2. Click **Try it out** button
3. Modify parameters if needed
4. Click **Execute** button
5. View response:
   - **Response body** (JSON data)
   - **Response code** (200 OK, 404 Not Found, etc.)
   - **Response headers**
   - **Curl command** (copy for terminal use)

### 7.2 Test GET Endpoint

Example: Get all products

1. Navigate to `http://localhost:8084/swagger-ui/index.html`
2. Find `GET /api/product` endpoint
3. Click **Try it out**
4. Click **Execute**
5. View response with list of products

**Expected Response:** `200 OK`

```json
[
  {
    "id": "507f1f77bcf86cd799439011",
    "name": "Sample Product",
    "description": "Product description",
    "price": 29.99
  }
]
```

### 7.3 Test POST Endpoint

Example: Create a product via Swagger UI

1. Navigate to `http://localhost:8084/swagger-ui/index.html`
2. Find `POST /api/product` endpoint
3. Click **Try it out**
4. Enter request body in the text area:

```json
{
  "name": "Test Product",
  "description": "Created via Swagger UI",
  "price": 49.99
}
```

5. Click **Execute**
6. View response with created product including generated ID

**Expected Response:** `201 Created`

```json
{
  "id": "507f1f77bcf86cd799439012",
  "name": "Test Product",
  "description": "Created via Swagger UI",
  "price": 49.99
}
```

### 7.4 Test with Authentication (Keycloak)

If Keycloak security is enabled (from Week 11-1 and Week 11-2):

**Note:** Direct service access (port 8084, 8082, 8083) currently bypasses API Gateway authentication. To test with Keycloak authentication, you would need to:

1. Access services through API Gateway
2. Configure Swagger UI to support OAuth2 authorization
3. This will be covered in a future lab on API Gateway aggregation

For now, Swagger UI works without authentication when accessing services directly.

---

## Step 8: Understanding Swagger UI Components

### 8.1 Swagger UI Sections

**Header Section:**
- API Title and Description
- Version information
- External documentation links
- License information

**Servers Section:**
- Dropdown to select API base URL
- Useful for switching between dev, staging, production
- Currently shows `http://localhost:<port>`

**Controllers Section:**
- Groups endpoints by controller
- Expandable panels for each controller
- Color-coded HTTP methods:
  - **GET** (blue): Retrieve data
  - **POST** (green): Create data
  - **PUT** (orange): Update data
  - **DELETE** (red): Delete data

**Schemas Section:**
- Data model definitions
- Request/Response object structures
- Field types and validation rules

### 8.2 Endpoint Details

When you expand an endpoint:

- **Parameters**: Path variables, query params, headers
- **Request Body**: JSON schema with example values
- **Responses**: Status codes with response schemas
- **Try it out**: Interactive testing
- **Example Value**: Sample request payload
- **Schema**: Data structure definition

---

## Troubleshooting

### Swagger UI Not Loading

**Problem:** 404 error when accessing `/swagger-ui/index.html`

**Solution:**
1. Verify SpringDoc dependencies in `build.gradle.kts`
2. Check `springdoc.swagger-ui.path` in `application.properties`
3. Ensure application is running on correct port
4. Try accessing without `/index.html`: `http://localhost:8084/swagger-ui`
5. Check application logs for errors:
   ```bash
   docker logs product-service
   ```

### Version Property Not Found

**Problem:** Error: "Could not resolve placeholder 'product-service.version'"

**Solution:**
1. Verify version property exists in `application.properties`:
   ```properties
   product-service.version=v1.0
   ```
2. Ensure property name matches `@Value("${product-service.version}")`
3. Check for typos in property name (hyphens vs underscores)
4. Verify `${}` syntax is used in `@Value` annotation

### OpenAPI JSON Returns Empty

**Problem:** `/api-docs` returns minimal JSON without endpoints

**Solution:**
1. Ensure controllers have proper `@RestController` annotation
2. Verify `@RequestMapping` or `@GetMapping/@PostMapping` paths are defined
3. Check that Spring Boot has scanned the controller packages
4. Restart application to regenerate documentation:
   ```bash
   docker-compose -p microservices-parent restart product-service
   ```

### Port Conflicts

**Problem:** Service won't start due to port already in use

**Solution:**

```bash
# Find process using port (e.g., 8084)
lsof -i :8084

# Kill process
kill -9 <PID>
```

Or stop all Docker containers:

```bash
cd microservices-parent
docker-compose -p microservices-parent down
```

### Container Build Fails

**Problem:** Docker build fails with Gradle errors

**Solution:**
1. Clean Gradle cache:
   ```bash
   cd product-service
   ./gradlew clean build
   ```
2. Remove old Docker images:
   ```bash
   docker rmi product-service order-service inventory-service
   ```
3. Rebuild containers:
   ```bash
   cd microservices-parent
   docker-compose -p microservices-parent up -d --build
   ```

### Documentation Shows Wrong Version

**Problem:** Swagger UI shows incorrect version number

**Solution:**
1. Check version in `application-docker.properties` matches `application.properties`
2. Verify Docker container is using `application-docker.properties`:
   ```bash
   docker logs product-service | grep "product-service.version"
   ```
3. Rebuild container with correct properties

---

## Summary

You now have:

- ✅ Swagger/OpenAPI documentation for all microservices
- ✅ Interactive Swagger UI for testing endpoints
- ✅ OpenAPI JSON specification for API consumption
- ✅ Customized API metadata (title, description, version, license)
- ✅ Docker-ready documentation configuration
- ✅ Separate configuration for local and Docker environments

**Key Benefits:**
- **Auto-generated** documentation from code - no manual updates needed
- **Interactive** API testing without Postman or curl
- **Standardized** OpenAPI 3.0 specification for tool compatibility
- **Easy onboarding** for new developers and API consumers
- **Professional** API presentation with customized branding
- **Developer productivity** through "Try it out" functionality

**Key Achievements:**
- Understand the role and benefits of API documentation
- Implement Swagger (OpenAPI) in Spring Boot using SpringDoc
- Customize Swagger documentation for clarity and consistency
- Access API documentation in both UI and JSON formats
- Deploy API documentation with Docker and ensure accessibility across services
- Test APIs interactively through Swagger UI

**Next Steps:**
- Configure API Gateway to aggregate all service documentation at single endpoint
- Add detailed endpoint descriptions with `@Operation` annotations
- Document request/response schemas with `@Schema` annotations
- Add security schemes to Swagger UI for OAuth2/Keycloak integration
- Generate client SDKs from OpenAPI specification
- Implement API versioning strategy with documentation
- Add example requests/responses with `@ApiResponse` annotations
- Configure CORS for Swagger UI in production environments

---

## Additional Resources

**SpringDoc Documentation:**
- Official Docs: https://springdoc.org/
- GitHub: https://github.com/springdoc/springdoc-openapi

**OpenAPI Specification:**
- OpenAPI 3.0 Spec: https://swagger.io/specification/
- Swagger Tools: https://swagger.io/tools/

**Best Practices:**
- Use semantic versioning (v1.0, v1.1, v2.0)
- Document all endpoints, even internal ones
- Include example requests/responses
- Keep external documentation links up to date
- Secure Swagger UI in production environments
- Use meaningful descriptions for endpoints and schemas
