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
- [Step 9: Refactor Order Service to Interface Pattern](#step-9-refactor-order-service-to-interface-pattern)
- [Step 10: Create Flyway Migration for Order Service](#step-10-create-flyway-migration-for-order-service)
- [Step 11: Update order-service Configuration](#step-11-update-order-service-configuration)
- [Step 12: Fix Product Service Test Configuration](#step-12-fix-product-service-test-configuration)
- [Step 13: Update docker-compose.yml](#step-13-update-docker-composeyml)
- [Step 13.1: Reset Docker Environment](#step-131-reset-docker-environment)
- [Step 14: Test Locally](#step-14-test-locally)
- [Step 15: Test with Docker Compose](#step-15-test-with-docker-compose)
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
FROM eclipse-temurin:21-jdk

RUN mkdir /app

COPY --from=builder /home/gradle/src/build/libs/*.jar /app/api-gateway.jar

EXPOSE 9000

ENTRYPOINT ["java", "-jar", "/app/api-gateway.jar"]
```

**Multi-stage Build:**
- **Stage 1:** Build with gradle:8-jdk21-alpine
- **Stage 2:** Runtime with eclipse-temurin:21-jdk
- **Exposed Port:** 9000

**Note:** Use `eclipse-temurin:21-jdk` instead of `openjdk:21-jdk` because the official OpenJDK Docker images have been deprecated. Eclipse Temurin is the recommended replacement from the Eclipse Adoptium project.

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

## Step 9: Refactor Order Service to Interface Pattern

Your order-service currently has a direct `OrderService` class. Refactor it to use the interface + implementation pattern for consistency with other services.

### 9.1 Create OrderService Interface

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/service/OrderService.java`

1. **Right-click** on the existing `OrderService.java` file
2. Select **Refactor → Rename**
3. Rename to `OrderServiceImpl`
4. Click **Refactor**

Now create the interface:

1. **Right-click** on `service` package
2. Select **New → Java Class**
3. Change type to **Interface**
4. Name: `OrderService`
5. Click **OK**

Replace content with:

```java
package ca.gbc.comp3095.orderservice.service;

import ca.gbc.comp3095.orderservice.dto.OrderRequest;

public interface OrderService {
    void placeOrder(OrderRequest orderRequest);
}
```

### 9.2 Update OrderServiceImpl

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/service/OrderServiceImpl.java`

Make OrderServiceImpl implement the interface:

**Before:**
```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class OrderServiceImpl {
```

**After:**
```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class OrderServiceImpl implements OrderService {
```

Add `@Override` annotation to `placeOrder` method:

```java
@Override
public void placeOrder(OrderRequest orderRequest) {
    // existing implementation
}
```

**Complete OrderServiceImpl.java:**

```java
package ca.gbc.comp3095.orderservice.service;

import ca.gbc.comp3095.orderservice.client.InventoryClient;
import ca.gbc.comp3095.orderservice.dto.OrderLineItemDto;
import ca.gbc.comp3095.orderservice.dto.OrderRequest;
import ca.gbc.comp3095.orderservice.model.Order;
import ca.gbc.comp3095.orderservice.model.OrderLineItem;
import ca.gbc.comp3095.orderservice.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class OrderServiceImpl implements OrderService {

    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;

    @Override
    public void placeOrder(OrderRequest orderRequest) {

        // Check if all products are in stock
        boolean allInStock = orderRequest.orderLineItemDtoList()
                .stream()
                .allMatch(orderLineItem ->
                        inventoryClient.isInStock(orderLineItem.skuCode(), orderLineItem.quantity()));

        if (!allInStock) {
            throw new RuntimeException("One or more products are not in stock");
        }

        // Map OrderLineItemDto to OrderLineItem entity
        List<OrderLineItem> orderLineItems = orderRequest.orderLineItemDtoList()
                .stream()
                .map(this::mapOrderLineItemDtoToOrderLineItem)
                .toList();

        // Create order
        Order order = Order.builder()
                .orderNumber(UUID.randomUUID().toString())
                .orderLineItems(orderLineItems)
                .build();

        // Save order
        orderRepository.save(order);
        log.info("Order {} placed successfully", order.getOrderNumber());
    }

    private OrderLineItem mapOrderLineItemDtoToOrderLineItem(OrderLineItemDto orderLineItemDto) {
        return OrderLineItem.builder()
                .skuCode(orderLineItemDto.skuCode())
                .price(orderLineItemDto.price())
                .quantity(orderLineItemDto.quantity())
                .build();
    }
}
```

---

## Step 10: Create Flyway Migration for Order Service

**Important:** Your order-service has a **two-table design** (Order + OrderLineItem entities) which is better than the reference implementation's single-table design. You need a migration that matches YOUR entities.

### 10.1 Create Migration Directory

```bash
cd microservices-parent/order-service
mkdir -p src/main/resources/db/migration
```

### 10.2 Create Migration File

**Location:** `order-service/src/main/resources/db/migration/V1__init.sql`

```sql
CREATE TABLE orders (
    id BIGSERIAL NOT NULL,
    order_number VARCHAR(255) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE order_line_items (
    id BIGSERIAL NOT NULL,
    sku_code VARCHAR(255) NOT NULL,
    price DECIMAL(19, 2) NOT NULL,
    quantity INTEGER NOT NULL,
    order_id BIGINT,
    PRIMARY KEY (id),
    CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);

CREATE INDEX idx_order_line_items_order_id ON order_line_items(order_id);
```

**Why this migration:**
- ✅ Matches your `Order` entity (`@Table(name = "orders")`)
- ✅ Matches your `OrderLineItem` entity (`@Table(name = "order_line_items")`)
- ✅ Includes foreign key relationship (`@OneToMany` in Order)
- ✅ Adds index for performance
- ✅ Uses `CASCADE DELETE` for data integrity

**Why this is needed:**
- Tests use **TestContainers** which creates isolated PostgreSQL containers
- TestContainers **only** uses Flyway migrations (not docker-compose init scripts)
- Without this migration, tests fail with: `ERROR: relation "orders" does not exist`

---

## Step 11: Update order-service Configuration

### 11.1 Update application.properties

**Location:** `order-service/src/main/resources/application.properties`

Change `ddl-auto` from `update` to `none`:

**Before:**
```properties
spring.jpa.hibernate.ddl-auto=update
```

**After:**
```properties
spring.jpa.hibernate.ddl-auto=none
```

**Why:** Flyway now manages the schema, not Hibernate.

### 11.2 Update application-docker.properties

**Location:** `order-service/src/main/resources/application-docker.properties`

Change `ddl-auto` from `update` to `none`:

**Before:**
```properties
spring.jpa.hibernate.ddl-auto=update
```

**After:**
```properties
spring.jpa.hibernate.ddl-auto=none
```

---

## Step 12: Fix Product Service Test Configuration

Your product-service has Redis caching enabled, but tests fail because Redis is not available during testing. We need to make Redis configuration conditional and configure tests to disable it.

### 12.1 Make RedisConfig Conditional

**Location:** `product-service/src/main/java/ca/gbc/comp3095/productservice/config/RedisConfig.java`

Add `@ConditionalOnProperty` annotation to make Redis configuration only load when caching is enabled:

**Before:**
```java
@Configuration
public class RedisConfig {
```

**After:**
```java
@Configuration
@ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis", matchIfMissing = true)
public class RedisConfig {
```

**Add import:**
```java
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
```

**Why:**
- Makes RedisConfig only load when `spring.cache.type=redis` (or not set at all)
- When tests set `spring.cache.type=none`, RedisConfig won't load
- Avoids needing Redis during testing

**Complete RedisConfig.java:**

```java
package ca.gbc.comp3095.productservice.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.jsontype.BasicPolymorphicTypeValidator;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;

import java.time.Duration;

@Configuration
@ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis", matchIfMissing = true)
public class RedisConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {

        ObjectMapper objectMapper = new ObjectMapper();

        objectMapper.activateDefaultTyping(
                BasicPolymorphicTypeValidator.builder()
                        .allowIfSubType(Object.class)
                        .build(),
                ObjectMapper.DefaultTyping.EVERYTHING,
                JsonTypeInfo.As.PROPERTY
        );

        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        Jackson2JsonRedisSerializer<Object> serializer =
                new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);

        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration
                .defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues()
                .serializeValuesWith(
                        RedisSerializationContext.SerializationPair.fromSerializer(serializer)
                );

        return RedisCacheManager
                .builder(connectionFactory)
                .cacheDefaults(redisCacheConfiguration)
                .build();
    }
}
```

### 12.2 Create Test Configuration

**Location:** `product-service/src/test/resources/application-test.properties`

Create new file:

```properties
spring.cache.type=none
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
```

**Why:**
- `spring.cache.type=none` - Disables caching (and prevents RedisConfig from loading)
- Excludes `RedisAutoConfiguration` - Prevents Spring Boot from auto-configuring Redis beans

### 12.3 Update ProductServiceApplicationTests

**Location:** `product-service/src/test/java/ca/gbc/comp3095/productservice/ProductServiceApplicationTests.java`

Add `@ActiveProfiles("test")` annotation:

**Before:**
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class ProductServiceApplicationTests {
```

**After:**
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class ProductServiceApplicationTests {
```

**Add import:**
```java
import org.springframework.test.context.ActiveProfiles;
```

**Why:** Activates the test profile which disables Redis.

**Note:** ProductServiceApplicationCacheTests.java already has Redis TestContainer, so it doesn't need this change.

**Complete ProductServiceApplicationTests.java:**

```java
package ca.gbc.comp3095.productservice;

import ca.gbc.comp3095.productservice.dto.ProductRequest;
import ca.gbc.comp3095.productservice.dto.ProductResponse;
import ca.gbc.comp3095.productservice.repository.ProductRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.restassured.RestAssured;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.test.context.ActiveProfiles;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class ProductServiceApplicationTests {

    @Container
    @ServiceConnection
    static MongoDBContainer mongoDBContainer = new MongoDBContainer(
            DockerImageName.parse("mongo:latest")
    );

    @LocalServerPort
    private Integer port;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private ProductRepository productRepository;

    @BeforeEach
    void setUp() {
        // Configure RestAssured for API testing
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;

        // Clear database before each test
        productRepository.deleteAll();
    }

    @Test
    void contextLoads() {
        // Verify container is running
        assertTrue(mongoDBContainer.isRunning());
        assertNotNull(port);
    }

    private ProductRequest getProductRequest() {
        return new ProductRequest(
                "Test Product",
                "Test Product Description",
                BigDecimal.valueOf(199.99)
        );
    }

    @Test
    void createProduct() {
        ProductRequest productRequest = getProductRequest();

        RestAssured.given()
                .contentType("application/json")
                .body(productRequest)
                .when()
                .post("/api/product")
                .then()
                .statusCode(201);

        // Verify product was saved to database
        assertEquals(1, productRepository.findAll().size());

        // Verify the saved product details
        var savedProduct = productRepository.findAll().get(0);
        assertEquals("Test Product", savedProduct.getName());
        assertEquals("Test Product Description", savedProduct.getDescription());
        assertEquals(BigDecimal.valueOf(199.99), savedProduct.getPrice());
    }

    @Test
    void getAllProducts() {
        // First, create a product
        ProductRequest productRequest = getProductRequest();

        RestAssured.given()
                .contentType("application/json")
                .body(productRequest)
                .when()
                .post("/api/product")
                .then()
                .statusCode(201);

        // Then, get all products
        List<ProductResponse> products = RestAssured.given()
                .when()
                .get("/api/product")
                .then()
                .statusCode(200)
                .extract()
                .body()
                .jsonPath()
                .getList(".", ProductResponse.class);

        // Verify we have 1 product
        assertEquals(1, products.size());
        assertEquals("Test Product", products.get(0).name());
        assertEquals("Test Product Description", products.get(0).description());
        assertEquals(BigDecimal.valueOf(199.99), products.get(0).price());
    }

    @Test
    void updateProduct() {
        // First, create a product
        ProductRequest productRequest = getProductRequest();

        RestAssured.given()
                .contentType("application/json")
                .body(productRequest)
                .when()
                .post("/api/product")
                .then()
                .statusCode(201);

        // Get the created product ID
        String productId = productRepository.findAll().get(0).getId();

        // Update the product
        ProductRequest updateRequest = new ProductRequest(
                "Updated Product",
                "Updated Description",
                BigDecimal.valueOf(299.99)
        );

        RestAssured.given()
                .contentType("application/json")
                .body(updateRequest)
                .when()
                .put("/api/product/" + productId)
                .then()
                .statusCode(204);

        // Verify the update
        var updatedProduct = productRepository.findById(productId).orElseThrow();
        assertEquals("Updated Product", updatedProduct.getName());
        assertEquals("Updated Description", updatedProduct.getDescription());
        assertEquals(BigDecimal.valueOf(299.99), updatedProduct.getPrice());
    }

    @Test
    void deleteProduct() {
        // First, create a product
        ProductRequest productRequest = getProductRequest();

        RestAssured.given()
                .contentType("application/json")
                .body(productRequest)
                .when()
                .post("/api/product")
                .then()
                .statusCode(201);

        // Get the created product ID
        String productId = productRepository.findAll().get(0).getId();

        // Delete the product
        RestAssured.given()
                .when()
                .delete("/api/product/" + productId)
                .then()
                .statusCode(204);

        // Verify deletion
        assertEquals(0, productRepository.findAll().size());
        assertTrue(productRepository.findById(productId).isEmpty());
    }
}
```

---

## Step 13: Update docker-compose.yml

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

**Update order-service environment:**

Change:
```yaml
  order-service:
    # ... existing config ...
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/order-service
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: update  # ❌ CHANGE THIS
```

To:
```yaml
  order-service:
    # ... existing config ...
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/order-service
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: none  # ✅ CHANGED TO none
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
      SPRING_JPA_HIBERNATE_DDL_AUTO: none
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

## Step 13.1: Reset Docker Environment

Before testing, clean up any existing Docker containers and volumes to ensure a fresh start:

```bash
cd microservices-parent
clear && docker-compose down -v && sleep 5 && docker-compose up --build -d
```

**What this does:**
- `clear` - Clears the terminal
- `docker-compose down -v` - Stops all containers and removes volumes (fresh database state)
- `sleep 5` - Waits 5 seconds for cleanup to complete
- `docker-compose up --build -d` - Rebuilds images and starts all services in background

**Expected output:**
```
Successfully built api-gateway
Successfully built product-service
Successfully built order-service
Successfully built inventory-service
...
Container api-gateway  Started
Container order-service  Started
Container product-service  Started
Container inventory-service  Started
```

**Why this is needed:**
- Removes old Flyway schema history (prevents checksum mismatch errors)
- Ensures all services use the latest code
- Provides a clean slate for testing

Wait 10-15 seconds for all services to fully start before testing.

---

## Step 14: Test Locally

### 14.1 Build Project

```bash
cd microservices-parent
./gradlew clean build
```

**Expected:**
```
BUILD SUCCESSFUL
```

**If you see test failures:** Review Steps 10-12 carefully.

### 14.2 Start Backend Services

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

### 14.3 Start API Gateway

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

### 14.4 Test with Postman

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

### 14.5 Verify Gateway Logs

Check Terminal 3 for routing logs:

```
Received request for product service: http://localhost:9000/api/product
Response status: 200 OK
```

---

## Step 15: Test with Docker Compose

### 15.1 Stop Local Services

Stop all three terminals (Ctrl+C).

### 15.2 Build Docker Images

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

### 15.3 Start All Services

```bash
docker-compose up -d
```

### 15.4 Verify Services Running

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

### 15.5 Check Gateway Logs

```bash
docker logs api-gateway
```

**Expected:**
```
Initializing product service route with URL: http://product-service:8084
Initializing order service route with URL: http://order-service:8082
Started ApiGatewayApplication
```

### 15.6 Test with Postman

Repeat tests from Step 14.4, same endpoints:
- `GET http://localhost:9000/api/product`
- `POST http://localhost:9000/api/product`
- `POST http://localhost:9000/api/order`

All should work identically.

### 15.7 Test with cURL (Optional)

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
