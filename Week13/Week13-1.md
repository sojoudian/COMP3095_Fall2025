# Migrating from OpenFeign to Spring REST Client

## Overview

Refactor the order-service to replace Spring Cloud OpenFeign with Spring's declarative REST Client. This migration simplifies dependencies and uses Spring Boot's native HTTP client capabilities with declarative service interfaces.

**Time:** 45-60 minutes

---

## Prerequisites

- ✅ Completed Week 12 (Keycloak Security and Swagger Documentation)
- ✅ All microservices running (product-service, order-service, inventory-service, api-gateway)
- ✅ Docker Desktop running
- ✅ Understanding of REST client communication

---

## Background

### Why Migrate from OpenFeign?

**OpenFeign** is a declarative web service client that makes writing HTTP clients easier. However, Spring Framework 6+ introduced a native declarative HTTP interface using `@HttpExchange` annotations, reducing the need for external dependencies.

### Benefits of Spring REST Client

- **Native Spring Support**: Built into Spring Framework 6+
- **Simplified Dependencies**: No need for Spring Cloud OpenFeign
- **Better Integration**: Seamless integration with Spring Boot 3.x
- **Modern API**: Uses Java's HttpClient under the hood
- **Type-Safe**: Compile-time checking of service interfaces

### What Changes?

| Component | OpenFeign (Old) | Spring REST Client (New) |
|-----------|----------------|-------------------------|
| **Dependency** | `spring-cloud-starter-openfeign` | Built into Spring Boot 3+ |
| **Annotation** | `@FeignClient` | `@GetExchange`, `@PostExchange` |
| **Configuration** | `@EnableFeignClients` | `RestClient` + `HttpServiceProxyFactory` |
| **Client Interface** | Annotated with `@FeignClient` | Plain interface with `@HttpExchange` |

---

## Step 1: Remove OpenFeign Dependencies

### 1.1 Update order-service build.gradle.kts

**Location:** `order-service/build.gradle.kts`

Remove the OpenFeign dependency:

```kotlin
// Remove this line
implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
```

Add Spring Cloud Contract Stub Runner for WireMock testing:

```kotlin
// Week 13 - Spring Cloud for REST Client and WireMock testing
implementation("org.springframework.cloud:spring-cloud-starter-contract-stub-runner")
```

### 1.2 Reload Gradle Project

**IntelliJ IDEA:**
1. Right-click on project root
2. Select **Gradle** → **Reload Gradle Project**
3. Wait for dependencies to download

---

## Step 2: Update InventoryClient Interface

### 2.1 Refactor Client Interface

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/client/InventoryClient.java`

```java
package ca.gbc.comp3095.orderservice.client;

import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.service.annotation.GetExchange;


public interface InventoryClient {

    @GetExchange("/api/inventory")
    boolean isInStock(@RequestParam String skuCode, @RequestParam Integer quantity);

}
```

**How It Works:**

Spring's `@GetExchange` creates a proxy that implements the interface at runtime. The actual HTTP client configuration happens in a separate configuration class (next step).

---

## Step 3: Verify OrderRequest DTO Structure

### 3.1 Check OrderRequest

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/dto/OrderRequest.java`

Verify the `OrderRequest` record has the correct structure:

```java
package ca.gbc.comp3095.orderservice.dto;

import java.math.BigDecimal;

public record OrderRequest(
        Long id,
        String orderNumber,
        String skuCode,
        BigDecimal price,
        Integer quantity) {}
```

**Important:** This simple DTO structure is required for the REST Client migration. It should have direct fields (`skuCode`, `price`, `quantity`), not nested objects.

---

## Step 4: Create RestClientConfig

### 4.1 Create RestClientConfig Class

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/config/RestClientConfig.java`

Create new class in the `config` package:

1. Right-click on `config` package
2. Select **New** → **Java Class**
3. Name: `RestClientConfig`
4. Click **OK**

**Code:**

```java
package ca.gbc.comp3095.orderservice.config;

import ca.gbc.comp3095.orderservice.client.InventoryClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.support.RestClientAdapter;
import org.springframework.web.service.invoker.HttpServiceProxyFactory;

@Configuration
public class RestClientConfig {

    @Value("${inventory.service.url}")
    private String inventoryServiceUrl;

    /**
     * Creates and configures an InventoryClient bean for interaction with the Inventory Service.
     * <p>
     * The method sets up a RestClient with the specified base URL, wraps it in a RestClientAdapter,
     * and uses an HttpServiceProxyFactory to create a proxy implementation of the InventoryClient interface.
     * </p>
     *
     * @return InventoryClient - a proxy instance configured to communicate with the Inventory Service.
     */
    @Bean
    public InventoryClient inventoryClient() {
        // Create a RestClient with the base URL from inventoryServiceUrl
        RestClient restClient = RestClient.builder()
                .baseUrl(inventoryServiceUrl)
                .build();

        // Wrap the RestClient in a RestClientAdapter for use with the HttpServiceProxyFactory
        var restClientAdapter = RestClientAdapter.create(restClient);

        // Create an HttpServiceProxyFactory, which generates a proxy instance of InventoryClient
        var httpServiceProxyFactory = HttpServiceProxyFactory.builderFor(restClientAdapter).build();

        // Return the proxy instance of InventoryClient for dependency injection
        return httpServiceProxyFactory.createClient(InventoryClient.class);
    }

}
```

**Configuration Breakdown:**

1. **@Configuration**: Marks this as a Spring configuration class
2. **@Value("${inventory.service.url}")**: Injects inventory service URL from application.properties
3. **RestClient.builder()**: Creates a RestClient instance (Spring's HTTP client)
4. **RestClientAdapter.create()**: Adapts RestClient for use with HttpServiceProxyFactory
5. **HttpServiceProxyFactory**: Creates dynamic proxy implementations of declarative HTTP interfaces
6. **createClient(InventoryClient.class)**: Generates implementation of InventoryClient interface

**How It Works:**

At runtime, Spring creates a proxy object that implements `InventoryClient`. When you call `isInStock()`, the proxy:
1. Constructs the HTTP request using `@GetExchange` metadata
2. Sends the request via `RestClient`
3. Deserializes the response
4. Returns the result

---

## Step 5: Update OrderServiceApplication

### 5.1 Remove @EnableFeignClients Annotation

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/OrderServiceApplication.java`

Remove the `@EnableFeignClients` annotation and its import:

```java
package ca.gbc.comp3095.orderservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }

}
```

**Why?**

Spring REST Client doesn't require component scanning annotations. The `RestClientConfig` bean factory handles client creation automatically.

---

## Step 6: Update Order Model

### 6.1 Simplify Order Entity Structure

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/model/Order.java`

For Week 6.2, we simplify the Order model to have direct fields instead of nested OrderLineItem collections. This matches the simplified OrderRequest structure.

Replace the Order model with:

```java
package ca.gbc.comp3095.orderservice.model;

import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;

@Entity
@Table(name="t_orders")
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class Order {

    /**
     * The @GeneratedValue annotation specifies how the primary key (ID) will be generated automatically by the database.
     *
     * strategy = GenerationType.IDENTITY:
     * - The database will handle auto-incrementing the primary key.
     * - Hibernate relies on the underlying database's auto-increment feature to generate unique IDs.
     * - Each time a new record is inserted, the database assigns the next available primary key value.
     *
     * Other strategies:
     *
     * 1. GenerationType.AUTO:
     * - Hibernate chooses the strategy based on the underlying database's capabilities (could be SEQUENCE, TABLE, or IDENTITY).
     * - This is the default strategy if none is specified.
     *
     * 2. GenerationType.SEQUENCE:
     * - Uses a database sequence object to generate values for the primary key.
     * - Commonly used in databases that support sequences (e.g., PostgreSQL, Oracle).
     * - Allows preallocation of ID values in batches for performance improvements.
     *
     * 3. GenerationType.TABLE:
     * - Uses a separate table in the database to generate primary key values.
     * - Can be used across multiple tables, but it is generally slower compared to other strategies.
     * - It provides a more flexible approach but is less commonly used.
     */
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String orderNumber;
    private String skuCode;
    private BigDecimal price;
    private Integer quantity;

}
```

**Key Changes:**

1. **Removed**: `List<OrderLineItem> orderLineItems` (nested collection)
2. **Added**: Direct fields for `skuCode`, `price`, and `quantity`
3. **Changed**: Table name from `"orders"` to `"t_orders"`
4. **Changed**: Lombok annotations from `@Data` to `@Getter` and `@Setter` (more explicit)

**Why Simplify?**

- **Direct Mapping**: OrderRequest fields map directly to Order fields (no transformation needed)
- **Simpler Database Schema**: Single table instead of join with OrderLineItem table
- **Easier Testing**: Fewer entities to manage in tests
- **Focus on Learning**: Emphasizes REST Client migration, not complex data modeling

### 6.2 Delete Unused Files

The simplified Order model no longer uses nested OrderLineItem structures. Delete the unused files:

```bash
cd microservices-parent
rm order-service/src/main/java/ca/gbc/comp3095/orderservice/dto/OrderLineItemDto.java
rm order-service/src/main/java/ca/gbc/comp3095/orderservice/model/OrderLineItem.java
```

---

## Step 7: Simplify OrderServiceImpl (Optional)

### 7.1 Remove Inventory Check (Week 6.2 Approach)

For Week 6.2, the order service is simplified to focus on REST Client migration without inventory validation.

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/service/OrderServiceImpl.java`

**SIMPLIFIED VERSION (Week 6.2):**

```java
package ca.gbc.comp3095.orderservice.service;

import ca.gbc.comp3095.orderservice.dto.OrderRequest;
import ca.gbc.comp3095.orderservice.model.Order;
import ca.gbc.comp3095.orderservice.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional
public class OrderServiceImpl implements OrderService {

    //We use the @RequiredArgsConstructor to resolve the dependency via constructor injection
    private final OrderRepository orderRepository;

    @Override
    public void placeOrder(OrderRequest orderRequest) {

        //Create Order object from mapping incoming OrderRequest dto
        //Builder pattern
        Order order = Order.builder()
                .orderNumber(UUID.randomUUID().toString())
                .price(orderRequest.price())
                .skuCode(orderRequest.skuCode())
                .quantity(orderRequest.quantity())
                .build();

        orderRepository.save(order);
    }


}
```

**Key Changes:**
- **Removed**: `InventoryClient` dependency injection
- **Removed**: Inventory check logic (`inventoryClient.isInStock()`)
- **Simplified**: Direct order placement without validation

**Why Simplify?**

This approach:
- Focuses on REST Client migration learning objectives
- Reduces complexity for testing
- Demonstrates service decoupling
- Will be re-integrated in future labs with proper error handling

---

## Step 8: Verify Configuration Files

### 8.1 Update application.properties

**Location:** `order-service/src/main/resources/application.properties`

Change the database name from `order-service` to `order_service` (hyphen to underscore):

```properties
# Change this line:
spring.datasource.url=jdbc:postgresql://localhost:5432/order-service

# To this:
spring.datasource.url=jdbc:postgresql://localhost:5432/order_service
```

**Why?** The database name must match the Docker configuration and initialization scripts which use `order_service` (with underscore). Using `order-service` (with hyphen) will cause connection failures.

### 8.2 Update application-docker.properties (order-service)

**Location:** `order-service/src/main/resources/application-docker.properties`

Add the Flyway configuration properties:

```properties
# Week 13
spring.flyway.baseline-on-migrate=true
spring.flyway.locations=classpath:db/migration
spring.flyway.enabled=true
```

**Why?** These properties ensure Flyway database migrations run properly in the Docker environment.

### 8.3 Update application-docker.properties (inventory-service)

**Location:** `inventory-service/src/main/resources/application-docker.properties`

Add the Flyway configuration properties:

```properties
# Week 13
spring.flyway.baseline-on-migrate=true
spring.flyway.locations=classpath:db/migration
spring.flyway.enabled=true
```

**Why?** Same as order-service - Flyway migrations must be enabled for Docker environment.

---

## Step 9: Update Flyway Migration

### 9.1 Update order-service V1__init.sql

**Location:** `order-service/src/main/resources/db/migration/V1__init.sql`

Replace the entire file content with:

```sql
CREATE TABLE t_orders (
   id BIGSERIAL NOT NULL,
   order_number VARCHAR(255) DEFAULT NULL,
   sku_code VARCHAR(255),
   price DECIMAL(19, 2),
   quantity INT,
   PRIMARY KEY (id)
);
```

**Key Changes:**

1. **Table name**: Changed from `orders` to `t_orders` (matches Order.java `@Table(name="t_orders")`)
2. **Added columns**: `sku_code`, `price`, `quantity` (matches simplified Order model)
3. **Removed table**: `order_line_items` table no longer needed (simplified model)
4. **Removed constraint**: Foreign key constraint no longer needed

**Why?** The migration must create the exact table structure that the Order entity expects. Mismatches between Flyway schema and JPA entity mapping cause application startup failures.

---

## Step 10: Update Tests with WireMock

### 10.1 Update Test Configuration

**Location:** `order-service/src/test/resources/application.properties`

Add the following properties:

```properties
# Week 13
spring.datasource.driver-class-name=org.testcontainers.jdbc.ContainerDatabaseDriver
spring.datasource.url=jdbc:tc:postgresql:15-alpine:///order_service
```

---

## Step 11: Test the Migration

### 11.1 Run Unit Tests

Test locally before deploying to Docker using IntelliJ IDEA:

1. Right-click on `order-service` project
2. Select **Run 'All Tests'**
3. Wait for tests to complete

**If Tests Fail:**

Check logs for:
- Missing dependency errors (verify Spring Cloud Contract is added)
- Connection refused errors (verify WireMock configuration)
- Bean creation errors (verify RestClientConfig is correct)

### 11.2 Run Service Locally

Start dependencies (PostgreSQL, MongoDB, Redis):

```bash
cd microservices-parent/docker/standalone/postgres
docker-compose up -d

cd ../mongo-redis
docker-compose up -d
```

Start order-service:

1. In IntelliJ IDEA, navigate to `OrderServiceApplication.java`
2. Right-click on the file
3. Select **Run 'OrderServiceApplication'**

Verify startup logs show:

```
INFO  RestClientConfig : Creating InventoryClient with URL: http://localhost:8083
INFO  OrderServiceApplication : Started OrderServiceApplication in 3.21 seconds
```

### 11.3 Test with Postman

**Create Order (POST):**

```
POST http://localhost:8082/api/order
Content-Type: application/json

{
  "skuCode": "SKU-001",
  "price": 99.99,
  "quantity": 2
}
```

**Expected Response:** `201 Created`

```json
{
  "id": 1,
  "orderNumber": "550e8400-e29b-41d4-a716-446655440000",
  "skuCode": "SKU-001",
  "price": 99.99,
  "quantity": 2
}
```

**Get Orders (GET):**

```
GET http://localhost:8082/api/order
```

**Expected Response:** `200 OK` with list of orders

---

## Step 12: Build and Deploy with Docker

### 12.1 Rebuild Docker Containers

Stop existing containers:

```bash
cd microservices-parent
docker-compose down
```

Rebuild with updated order-service:

```bash
docker-compose -p microservices-parent up -d --build
```

Wait ~60-90 seconds for services to start.

### 12.2 Verify Containers

```bash
docker ps
```

Expected containers:
- keycloak
- postgres-keycloak
- api-gateway
- product-service
- order-service (rebuilt)
- inventory-service
- postgres-order
- postgres-inventory
- mongodb
- mongo-express
- redis
- redis-insight

### 12.3 Check order-service Logs

```bash
docker logs order-service
```

Look for successful startup:

```
INFO  RestClientConfig : Creating InventoryClient with URL: http://inventory-service:8083
INFO  OrderServiceApplication : Started OrderServiceApplication in 5.123 seconds
```

### 12.4 Test Through API Gateway

Since Keycloak security is enabled, obtain token first (see Week 12-1 for details):

**Postman OAuth2 Configuration:**
1. Authorization → OAuth 2.0
2. Grant Type: Client Credentials
3. Access Token URL: `http://keycloak:8080/realms/spring-microservices-security-realm/protocol/openid-connect/token`
4. Client ID: `spring-client-credentials-id`
5. Client Secret: `<your-secret>`
6. Get New Access Token → Use Token

**Test Order Creation:**

```
POST http://localhost:9000/api/order
Content-Type: application/json
Authorization: Bearer <token>

{
  "skuCode": "SKU-001",
  "price": 99.99,
  "quantity": 2
}
```

**Expected Response:** `201 Created`

---

## Step 13: Verify Swagger Documentation

### 13.1 Access Order Service Swagger

Navigate to:

```
http://localhost:8082/swagger-ui/index.html
```

Verify:
- All endpoints are documented
- InventoryClient is not visible (internal service communication)
- Order endpoints are interactive

### 13.2 Test via Swagger UI

1. Expand `POST /api/order`
2. Click **Try it out**
3. Enter request body:

```json
{
  "skuCode": "SKU-002",
  "price": 149.99,
  "quantity": 1
}
```

4. Click **Execute**
5. Verify `201 Created` response

---

## Troubleshooting

### Bean Creation Error: InventoryClient

**Error:**

```
Error creating bean with name 'inventoryClient': FactoryBean threw exception on object creation
```

**Solution:**

1. Verify `RestClientConfig` is in a package scanned by Spring:
   - Must be under `ca.gbc.comp3095.orderservice.config`
2. Check `@Configuration` annotation is present
3. Verify `inventory.service.url` property exists in application.properties
4. Ensure Spring Cloud Contract Stub Runner dependency is added

### Connection Refused to Inventory Service

**Error:**

```
java.net.ConnectException: Connection refused: localhost:8083
```

**Solution:**

**If running locally:**
1. Start inventory-service:
   - In IntelliJ IDEA, navigate to `InventoryServiceApplication.java`
   - Right-click on the file
   - Select **Run 'InventoryServiceApplication'**

**If running in Docker:**
1. Verify inventory-service container is running:
   ```bash
   docker ps | grep inventory-service
   ```
2. Check `application-docker.properties` uses `http://inventory-service:8083` (not localhost)

### Missing HttpServiceProxyFactory

**Error:**

```
java.lang.ClassNotFoundException: org.springframework.web.service.invoker.HttpServiceProxyFactory
```

**Solution:**

This class requires Spring Framework 6.0+ (included in Spring Boot 3.0+). Verify:

```kotlin
plugins {
    id("org.springframework.boot") version "3.4.4"  // Must be 3.0+
}
```

If using older Spring Boot version (2.x), upgrade to Spring Boot 3.x or stick with OpenFeign.

### Tests Failing with WireMock

**Error:**

```
org.springframework.web.client.ResourceAccessException: I/O error on GET request
```

**Solution:**

1. Verify `spring-cloud-starter-contract-stub-runner` dependency is added
2. Check test configuration includes WireMock setup
3. Ensure test properties use `${wiremock.server.port}` placeholder
4. Verify `@AutoConfigureWireMock` annotation is present in test class (if using)

### Gradle Build Fails

**Error:**

```
Could not resolve org.springframework.cloud:spring-cloud-starter-contract-stub-runner
```

**Solution:**

Add Spring Cloud BOM to `build.gradle.kts`:

```kotlin
dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2024.0.1")
    }
}
```

---

## Understanding the Architecture

### Request Flow with Spring REST Client

1. **OrderController** receives POST request
2. **OrderServiceImpl** calls `inventoryClient.isInStock()`
3. **HttpServiceProxyFactory proxy** intercepts call
4. **RestClient** constructs HTTP GET to `http://inventory-service:8083/api/inventory?skuCode=SKU-001&quantity=2`
5. **InventoryController** (inventory-service) responds with `true` or `false`
6. **Proxy** deserializes response to `boolean`
7. **OrderServiceImpl** receives result and proceeds

### Comparison: OpenFeign vs Spring REST Client

**OpenFeign Architecture:**

```
OrderServiceImpl
  → @FeignClient annotated interface
  → Feign proxy
  → Ribbon load balancer
  → HTTP request
```

**Spring REST Client Architecture:**

```
OrderServiceImpl
  → @GetExchange annotated interface
  → HttpServiceProxyFactory proxy
  → RestClient
  → HTTP request
```

**Key Difference:**

Spring REST Client is simpler and requires fewer dependencies. OpenFeign provides advanced features like load balancing and circuit breakers (now handled by Spring Cloud LoadBalancer and Resilience4j separately).

---

## Summary

You now have:

- ✅ Migrated from Spring Cloud OpenFeign to Spring REST Client
- ✅ Simplified dependency management (removed external library)
- ✅ Implemented declarative HTTP interfaces with `@GetExchange`
- ✅ Created RestClient bean factory configuration
- ✅ Updated application to use native Spring 6 features
- ✅ Maintained WireMock test compatibility
- ✅ Verified migration with local and Docker deployments

**Key Benefits:**
- **Reduced Dependencies**: No need for Spring Cloud OpenFeign
- **Native Spring Support**: Uses built-in Spring Framework 6 capabilities
- **Modern API**: Leverages Java's HttpClient
- **Simplified Configuration**: Less boilerplate code
- **Future-Proof**: Aligned with Spring Boot 3.x direction

**Key Achievements:**
- Understand the benefits of Spring REST Client over OpenFeign
- Implement declarative HTTP interfaces using `@HttpExchange` annotations
- Configure RestClient with HttpServiceProxyFactory
- Migrate existing Feign clients to Spring REST Client
- Maintain test compatibility with WireMock

**Next Steps:**
- Add error handling and retry logic to REST Client
- Implement circuit breaker pattern with Resilience4j
- Add request/response logging for debugging
- Configure timeout and connection pool settings
- Implement load balancing with Spring Cloud LoadBalancer
- Add metrics and monitoring for REST Client calls

---

## Additional Resources

**Spring Documentation:**
- REST Clients: https://docs.spring.io/spring-framework/reference/integration/rest-clients.html
- HTTP Interface: https://docs.spring.io/spring-framework/reference/integration/rest-clients.html#rest-http-interface

**Spring Boot 3 Migration:**
- Spring Boot 3.0 Migration Guide: https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide

**Comparison Articles:**
- OpenFeign vs RestClient: https://spring.io/blog/2023/06/07/spring-boot-3-1-0-available-now

**Best Practices:**
- Always use `@Value` injection for service URLs (avoid hardcoding)
- Configure appropriate timeouts for production use
- Implement proper error handling for HTTP failures
- Use WireMock for integration testing
- Monitor REST Client performance with actuator metrics
