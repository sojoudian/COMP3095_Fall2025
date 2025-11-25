# API Gateway Documentation Aggregation

## Overview

Aggregate Swagger/OpenAPI documentation from all microservices into a single unified documentation interface accessible through the API Gateway. This provides developers with one-stop access to documentation for all services in the microservices architecture.

**Time:** 60-75 minutes

---

## Prerequisites

- Completed Week 12-3 (Swagger Documentation for all services)
- Completed Week 13-1 (REST Client Migration)
- All microservices have SpringDoc OpenAPI configured
- API Gateway operational with Keycloak security
- Docker Desktop running

---

## Background

### Why Aggregate Documentation?

In a microservices architecture, each service exposes its own Swagger UI at different ports:

- Product Service: `http://localhost:8084/swagger-ui/index.html`
- Order Service: `http://localhost:8082/swagger-ui/index.html`
- Inventory Service: `http://localhost:8083/swagger-ui/index.html`

This fragmentation creates challenges:
- Developers must remember multiple URLs
- No unified view of the entire API ecosystem
- Difficult to understand inter-service dependencies
- Keycloak authentication must be configured separately for each

### Solution: API Gateway Aggregation

By aggregating documentation in the API Gateway:
- **Single Entry Point**: One URL for all documentation
- **Unified Interface**: Dropdown selector for different services
- **Centralized Security**: One authentication point
- **Professional Presentation**: Clean, organized API catalog
- **Developer Productivity**: Faster onboarding and API discovery

### Architecture

```
Browser → API Gateway (localhost:9000)
          ↓
          ├─→ Product Service Docs (/aggregate/product-service/v3/api-docs)
          ├─→ Order Service Docs (/aggregate/order-service/v3/api-docs)
          └─→ Inventory Service Docs (/aggregate/inventory-service/v3/api-docs)
```

---

## Step 1: Add SpringDoc to API Gateway

### 1.1 Update api-gateway build.gradle.kts

**Location:** `api-gateway/build.gradle.kts`

Add SpringDoc OpenAPI dependencies:

```kotlin
// Week 13 - Swagger Documentation Aggregation
implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.9")
testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.9")
```

**Version Note:** Using 2.8.9 (slightly newer than 2.8.8 used in services) for latest bug fixes.

### 1.2 Reload Gradle Project

**IntelliJ IDEA:**
1. Right-click on project root
2. Select **Gradle** → **Reload Gradle Project**
3. Wait for dependencies to download

---

## Step 2: Configure Swagger Aggregation

### 2.1 Update application.properties

**Location:** `api-gateway/src/main/resources/application.properties`

Add Swagger configuration with service aggregation:

```properties
spring.application.name=api-gateway

server.port=9000

# Services
services.product-url=http://localhost:8084
services.order-url=http://localhost:8082
services.inventory-url=http://localhost:8083

# Week 12 - Keycloak Security
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:8080/realms/spring-microservices-security-realm

# Week 13 - Swagger UI Configuration
springdoc.swagger-ui.path=/swagger-ui
# Aggregate documentation from all microservices
springdoc.swagger-ui.urls[0].name=Product Service
springdoc.swagger-ui.urls[0].url=/aggregate/product-service/v3/api-docs
springdoc.swagger-ui.urls[1].name=Order Service
springdoc.swagger-ui.urls[1].url=/aggregate/order-service/v3/api-docs
springdoc.swagger-ui.urls[2].name=Inventory Service
springdoc.swagger-ui.urls[2].url=/aggregate/inventory-service/v3/api-docs
```

**Configuration Breakdown:**

- **springdoc.swagger-ui.path**: Custom path for Swagger UI (`/swagger-ui`)
- **springdoc.swagger-ui.urls[0].name**: Display name in service dropdown
- **springdoc.swagger-ui.urls[0].url**: Path to service's OpenAPI JSON specification

**How It Works:**

The Swagger UI dropdown will show:
- Product Service (loads docs from `/aggregate/product-service/v3/api-docs`)
- Order Service (loads docs from `/aggregate/order-service/v3/api-docs`)
- Inventory Service (loads docs from `/aggregate/inventory-service/v3/api-docs`)

### 2.2 Update application-docker.properties

**Location:** `api-gateway/src/main/resources/application-docker.properties`

Add the same configuration for Docker environment:

```properties
spring.application.name=api-gateway

server.port=9000

# Services
services.product-url=http://product-service:8084
services.order-url=http://order-service:8082
services.inventory-url=http://inventory-service:8083

# Week 12 - Keycloak Security (Docker)
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://keycloak:8080/realms/spring-microservices-security-realm

# Week 13 - Swagger UI Configuration
springdoc.swagger-ui.path=/swagger-ui
# Aggregate documentation from all microservices
springdoc.swagger-ui.urls[0].name=Product Service
springdoc.swagger-ui.urls[0].url=/aggregate/product-service/v3/api-docs
springdoc.swagger-ui.urls[1].name=Order Service
springdoc.swagger-ui.urls[1].url=/aggregate/order-service/v3/api-docs
springdoc.swagger-ui.urls[2].name=Inventory Service
springdoc.swagger-ui.urls[2].url=/aggregate/inventory-service/v3/api-docs
```

**Note:** The aggregation URLs remain the same for both local and Docker environments. Only the service URLs change.

---

## Step 3: Add Documentation Routes

### 3.1 Update Routes Configuration

**Location:** `api-gateway/src/main/java/ca/gbc/comp3095/apigateway/routes/Routes.java`

Replace the entire file with this complete code:

```java
package ca.gbc.comp3095.apigateway.routes;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gateway.server.mvc.handler.GatewayRouterFunctions;
import org.springframework.cloud.gateway.server.mvc.handler.HandlerFunctions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.function.*;

import static org.springframework.cloud.gateway.server.mvc.filter.FilterFunctions.setPath;

@Configuration
@Slf4j
public class Routes {

    @Value("${services.product-url}")
    private String productServiceUrl;

    @Value("${services.order-url}")
    private String orderServiceUrl;

    @Value("${services.inventory-url}")
    private String inventoryServiceUrl;

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

    @Bean
    public RouterFunction<ServerResponse> inventoryServiceRoute() {
        log.info("Initializing inventory service route with URL: {}", inventoryServiceUrl);

        return GatewayRouterFunctions.route("inventory_service")
                .route(RequestPredicates.path("/api/inventory"), request -> {
                    log.info("Received request for inventory service: {}", request.uri());
                    try {
                        ServerResponse response = HandlerFunctions.http(inventoryServiceUrl).handle(request);
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
    public RouterFunction<ServerResponse> productServiceSwaggerRoute(){
        return GatewayRouterFunctions.route("product_service_swagger")
                .route(RequestPredicates.path("/aggregate/product-service/v3/api-docs"),
                        HandlerFunctions.http(productServiceUrl))
                .filter(setPath("/api-docs"))
                .build();
    }

    @Bean
    public RouterFunction<ServerResponse> orderServiceSwaggerRoute(){
        return GatewayRouterFunctions.route("order_service_swagger")
                .route(RequestPredicates.path("/aggregate/order-service/v3/api-docs"),
                        HandlerFunctions.http(orderServiceUrl))
                .filter(setPath("/api-docs"))
                .build();
    }

    @Bean
    public RouterFunction<ServerResponse> inventoryServiceSwaggerRoute(){
        return GatewayRouterFunctions.route("inventory_service_swagger")
                .route(RequestPredicates.path("/aggregate/inventory-service/v3/api-docs"),
                        HandlerFunctions.http(inventoryServiceUrl))
                .filter(setPath("/api-docs"))
                .build();
    }
}
```

**How Documentation Routes Work:**

```
Request:  GET /aggregate/product-service/v3/api-docs
          ↓
Filter:   setPath("/api-docs")  // Rewrites path
          ↓
Forward:  GET http://product-service:8084/api-docs
          ↓
Response: OpenAPI JSON specification
```

The `setPath()` filter transforms the aggregation URL into the actual service endpoint.

---

## Step 4: Update Security Configuration

### 4.1 Permit Unauthenticated Access to Documentation

**Location:** `api-gateway/src/main/java/ca/gbc/apigateway/config/SecurityConfig.java`

Add a whitelist array as a class field:

```java
// Week 13 - Whitelist documentation endpoints (no authentication required)
private final String[] noauthResourceUrls = {
        "/swagger-ui",
        "/swagger-ui/**",
        "/v3/api-docs/**",
        "/swagger-resources/**",
        "/api-docs/**",
        "/aggregate/**"
};
```

Update the `securityFilterChain` method to permit unauthenticated access to documentation:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {

    log.info("Initializing Security Filter Chain...");

    return httpSecurity
            .csrf(AbstractHttpConfigurer::disable)
            // Week 13 - UPDATE: Permit unauthenticated access to documentation
            .authorizeHttpRequests(authorize -> authorize
                    .requestMatchers(noauthResourceUrls)  // Add this line
                    .permitAll()                           // Add this line
                    .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                    .jwt(Customizer.withDefaults()))
            .build();
}
```

**Key Changes:**

1. **Added Whitelist Array** (`noauthResourceUrls`):
   - `/swagger-ui`: Swagger UI home page
   - `/swagger-ui/**`: All Swagger UI assets (CSS, JS, images)
   - `/v3/api-docs/**`: OpenAPI 3.0 JSON specification
   - `/swagger-resources/**`: Swagger configuration resources
   - `/api-docs/**`: Custom API documentation endpoint
   - `/aggregate/**`: Aggregated documentation from all services

2. **Updated Authorization Rules**:
   - `.requestMatchers(noauthResourceUrls).permitAll()`: Allow unauthenticated access to documentation
   - `.anyRequest().authenticated()`: All other requests require JWT token

**Why Permit Documentation?**

Documentation endpoints should be accessible without authentication for:
- Developer onboarding (no need to configure Keycloak first)
- Public API documentation
- Easy testing and exploration
- Third-party integration discovery

**Security Note:** In production, you can restrict documentation access by:
- Removing the whitelist
- Using IP whitelisting
- Requiring basic authentication
- Deploying documentation to internal network only

---

## Step 5: Test Aggregated Documentation

### 5.1 Run Services Locally

Start all dependencies:

```bash
# PostgreSQL
cd microservices-parent/docker/standalone/postgres
docker-compose up -d

# MongoDB + Redis
cd ../mongo-redis
docker-compose up -d

# Keycloak
cd ../keycloak
docker-compose up -d
```

Start microservices using IntelliJ IDEA:

**Product Service:**
1. Navigate to `ProductServiceApplication.java`
2. Right-click on the file
3. Select **Run 'ProductServiceApplication'**

**Order Service:**
1. Navigate to `OrderServiceApplication.java`
2. Right-click on the file
3. Select **Run 'OrderServiceApplication'**

**Inventory Service:**
1. Navigate to `InventoryServiceApplication.java`
2. Right-click on the file
3. Select **Run 'InventoryServiceApplication'**

**API Gateway:**
1. Navigate to `ApiGatewayApplication.java`
2. Right-click on the file
3. Select **Run 'ApiGatewayApplication'**

Wait for all services to start (~30-60 seconds).

### 5.2 Access Aggregated Documentation

Open browser and navigate to:

```
http://localhost:9000/swagger-ui/index.html
```

**Expected Result:**

You should see the Swagger UI interface with a dropdown menu in the top-right corner labeled **"Select a definition"** containing:
- Product Service
- Order Service
- Inventory Service

### 5.3 Test Service Selection

1. **Select Product Service** from dropdown
   - Should load Product Service endpoints (`GET /api/product`, `POST /api/product`)
   - Verify schemas show `Product`, `ProductRequest`, `ProductResponse`

2. **Select Order Service** from dropdown
   - Should load Order Service endpoints (`GET /api/order`, `POST /api/order`)
   - Verify schemas show `Order`, `OrderRequest`

3. **Select Inventory Service** from dropdown
   - Should load Inventory Service endpoint (`GET /api/inventory`)
   - Verify schemas show `Inventory`

### 5.4 Test Interactive Functionality

**Test Product Service - GET All Products:**

1. Select **Product Service** from dropdown
2. Expand `GET /api/product`
3. Click **Try it out**
4. Click **Execute**

**Expected Response:** `401 Unauthorized`

This is correct! API endpoints still require authentication (only documentation is public).

### 5.5 Test with Authentication

To test actual API calls through Swagger UI, you need to configure authentication.

**Important Note:** Configuring OAuth2 authentication in Swagger UI requires additional setup (Security Schemes configuration). For now, use Postman for authenticated API testing.

**Postman Alternative:**

Use Postman with Keycloak OAuth2 (see Week 12-1 and 12-2) to test actual API calls:

```
GET http://localhost:9000/api/product
Authorization: Bearer <token>
```

---

## Step 6: Deploy with Docker

### 6.1 Stop Local Services

Stop all locally running services (Ctrl+C in each terminal).

Stop standalone databases:

```bash
cd docker/standalone/postgres
docker-compose down

cd ../mongo-redis
docker-compose down

cd ../keycloak
docker-compose down
```

### 6.2 Rebuild Docker Containers

Rebuild all services with updated API Gateway:

```bash
cd microservices-parent
docker-compose -p microservices-parent up -d --build
```

Wait ~90-120 seconds for all containers to start and stabilize.

### 6.3 Verify Containers

```bash
docker ps
```

Expected containers (14 total):
- keycloak
- postgres-keycloak
- api-gateway (rebuilt with aggregation)
- product-service
- order-service
- inventory-service
- postgres-order
- postgres-inventory
- mongodb
- mongo-express
- redis
- redis-insight
- pgadmin

### 6.4 Check API Gateway Logs

```bash
docker logs api-gateway
```

Look for successful route initialization:

```
INFO  Routes : Initializing product service route with URL: http://product-service:8084
INFO  Routes : Initializing order service route with URL: http://order-service:8082
INFO  Routes : Initializing inventory service route with URL: http://inventory-service:8083
INFO  ApiGatewayApplication : Started ApiGatewayApplication in 5.234 seconds
```

### 6.5 Test Docker Deployment

Open browser:

```
http://localhost:9000/swagger-ui/index.html
```

Verify:
- Swagger UI loads successfully
- Dropdown shows all three services
- Switching services loads correct endpoints
- All service documentation is accessible

---

## Step 7: Verify Individual Service Documentation

### 7.1 Test OpenAPI JSON Endpoints

Each service's OpenAPI specification should be accessible through the gateway:

**Product Service API Docs:**

```
GET http://localhost:9000/aggregate/product-service/v3/api-docs
```

**Expected Response:** JSON OpenAPI specification

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
  "servers": [
    {
      "url": "http://localhost:8084",
      "description": "Generated server url"
    }
  ],
  "paths": {
    "/api/product": {
      "get": { ... },
      "post": { ... }
    }
  }
}
```

**Order Service API Docs:**

```
GET http://localhost:9000/aggregate/order-service/v3/api-docs
```

**Inventory Service API Docs:**

```
GET http://localhost:9000/aggregate/inventory-service/v3/api-docs
```

All should return valid OpenAPI JSON specifications.

### 7.2 Direct Service Access Still Works

Individual services still expose their own documentation:

**Product Service (Direct):**

```
http://localhost:8084/swagger-ui/index.html
```

**Order Service (Direct):**

```
http://localhost:8082/swagger-ui/index.html
```

**Inventory Service (Direct):**

```
http://localhost:8083/swagger-ui/index.html
```

Both access methods work:
- **Gateway**: Aggregated view with service selector
- **Direct**: Individual service documentation

---

## Troubleshooting

### Swagger UI Shows Empty Dropdown

**Problem:** Dropdown has no services listed

**Solution:**

1. Check `application.properties` configuration:
   ```properties
   springdoc.swagger-ui.urls[0].name=Product Service
   springdoc.swagger-ui.urls[0].url=/aggregate/product-service/v3/api-docs
   ```
2. Verify property syntax (array notation with square brackets)
3. Ensure no typos in service names
4. Restart api-gateway

### 404 Not Found for Documentation Routes

**Problem:** Accessing `/aggregate/product-service/v3/api-docs` returns 404

**Solution:**

1. Verify documentation routes exist in `Routes.java`:
   ```java
   @Bean
   public RouterFunction<ServerResponse> productServiceSwaggerRoute()
   ```
2. Check import statement:
   ```java
   import static org.springframework.cloud.gateway.server.mvc.filter.FilterFunctions.setPath;
   ```
3. Verify service URL configuration in `application.properties`:
   ```properties
   services.product-url=http://product-service:8084
   ```
4. Check logs for route initialization errors:
   ```bash
   docker logs api-gateway
   ```

### 401 Unauthorized for Documentation

**Problem:** Cannot access `/swagger-ui` - returns 401

**Solution:**

1. Verify `SecurityConfig` includes documentation whitelist:
   ```java
   private final String[] noauthResourceUrls = {
       "/swagger-ui",
       "/swagger-ui/**",
       "/aggregate/**"
   };
   ```
2. Check authorization rules:
   ```java
   .authorizeHttpRequests(authorize -> authorize
       .requestMatchers(noauthResourceUrls).permitAll()
       .anyRequest().authenticated())
   ```
3. Ensure whitelist is applied before `.anyRequest().authenticated()`
4. Restart api-gateway

### Service Documentation Not Loading

**Problem:** Selecting service from dropdown shows error "Failed to fetch"

**Solution:**

1. **Check service is running:**
   ```bash
   docker ps | grep product-service
   ```

2. **Verify service has SpringDoc dependency:**
   ```kotlin
   implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
   ```

3. **Check service OpenAPI endpoint works directly:**
   ```
   http://localhost:8084/api-docs
   ```

4. **Verify route filter configuration:**
   ```java
   .filter(setPath("/api-docs"))  // Transforms /aggregate/* to /api-docs
   ```

5. **Check CORS configuration** (if accessing from different domain)

### Services Start But Documentation Empty

**Problem:** Swagger UI loads but shows no endpoints

**Solution:**

1. Verify controllers are annotated:
   ```java
   @RestController
   @RequestMapping("/api/product")
   ```

2. Check OpenAPIConfig bean exists:
   ```java
   @Configuration
   public class OpenAPIConfig { ... }
   ```

3. Verify service application properties:
   ```properties
   springdoc.api-docs.path=/api-docs
   ```

4. Check service logs for SpringDoc initialization:
   ```bash
   docker logs product-service | grep springdoc
   ```

### Port Conflicts

**Problem:** Services fail to start with port conflicts

**Solution:**

```bash
# Check what's using port 9000
lsof -i :9000

# Kill process
kill -9 <PID>

# Or stop all Docker containers
docker-compose -p microservices-parent down
```

---

## Understanding the Architecture

### Documentation Flow

```
Browser Request: http://localhost:9000/swagger-ui/index.html
                 ↓
API Gateway: Loads Swagger UI with service URLs configuration
                 ↓
User Selects: "Product Service" from dropdown
                 ↓
Swagger UI Fetches: /aggregate/product-service/v3/api-docs
                 ↓
API Gateway Route: productServiceSwaggerRoute()
                 ↓
Filter: setPath("/api-docs")
                 ↓
Forward To: http://product-service:8084/api-docs
                 ↓
Product Service: Returns OpenAPI JSON
                 ↓
Swagger UI: Renders endpoints, schemas, examples
```

### Security Flow

**Documentation Access (No Auth Required):**

```
GET /swagger-ui/index.html
  → SecurityFilterChain
  → Matches noauthResourceUrls whitelist
  → permitAll()
  → Allow access
```

**API Access (Auth Required):**

```
GET /api/product
  → SecurityFilterChain
  → Does NOT match whitelist
  → Requires authentication
  → Validate JWT token
  → Allow access if valid
```

### Comparison: Before vs After

**Before Aggregation:**

```
Product Docs:   http://localhost:8084/swagger-ui/index.html
Order Docs:     http://localhost:8082/swagger-ui/index.html
Inventory Docs: http://localhost:8083/swagger-ui/index.html
```

**After Aggregation:**

```
All Docs: http://localhost:9000/swagger-ui/index.html
          (Select service from dropdown)
```

---

## Summary

You now have:

- Aggregated Swagger/OpenAPI documentation in API Gateway
- Unified documentation interface with service selector
- Unauthenticated access to documentation (authenticated API calls)
- Documentation routes for all microservices
- Security configuration allowing public documentation
- Both direct and aggregated documentation access
- Professional API documentation presentation

**Key Benefits:**
- **Single Entry Point**: One URL for all documentation
- **Improved Developer Experience**: Easy service discovery
- **Centralized Management**: Update docs in one place
- **Professional Presentation**: Clean, organized API catalog
- **Reduced Complexity**: No need to remember multiple URLs
- **Better Onboarding**: New developers find APIs faster

**Key Achievements:**
- Understand the value of documentation aggregation
- Configure SpringDoc OpenAPI in Spring Cloud Gateway
- Implement documentation proxy routes with path rewriting
- Whitelist documentation endpoints in security configuration
- Create unified API documentation interface
- Test aggregated documentation locally and in Docker

**Next Steps:**
- Add security schemes to Swagger UI (OAuth2 authentication)
- Implement API versioning with documentation
- Add detailed endpoint descriptions with `@Operation` annotations
- Configure CORS for external documentation access
- Add custom Swagger UI themes and branding
- Implement automated documentation generation in CI/CD
- Add request/response examples with `@ApiResponse`
- Create API changelog and deprecation notices
- Integrate with API documentation tools (Postman, Redocly)

---

## Additional Resources

**SpringDoc OpenAPI:**
- Official Docs: https://springdoc.org/
- Swagger UI Configuration: https://springdoc.org/#swagger-ui-properties
- OpenAPI 3 Specification: https://swagger.io/specification/

**Spring Cloud Gateway:**
- Routing: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway/gatewayfilter-factories/rewritepath-factory.html
- Filters: https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-mvc/filters.html

**API Documentation Best Practices:**
- OpenAPI Best Practices: https://swagger.io/blog/api-documentation/api-documentation-best-practices/
- REST API Design: https://restfulapi.net/

**Security:**
- OAuth2 in Swagger UI: https://springdoc.org/#swagger-ui-oauth-2
- Spring Security Configuration: https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html

**Tools:**
- Swagger Editor: https://editor.swagger.io/
- Postman: https://www.postman.com/
- Redocly: https://redocly.com/

**Best Practices:**
- Always document breaking changes
- Use semantic versioning for APIs
- Include authentication examples
- Provide sample requests and responses
- Document error codes and messages
- Keep documentation synchronized with code
- Review documentation in code reviews
- Make documentation accessible (public or internal portal)
