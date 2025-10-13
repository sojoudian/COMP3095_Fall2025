# Integration Testing with TestContainers - Lab Guide

## Introduction

This guide will help you add integration testing to your Spring Boot microservices using **TestContainers**. Integration tests verify that your entire application works correctly with real databases, ensuring your code functions properly in production-like environments.

**What You'll Learn:**
- How to add TestContainers dependencies to your project
- How to write integration tests using REST Assured
- How to test REST APIs with real databases
- Best practices for testing microservices

**Time Required:** 60-75 minutes

---

## Table of Contents

1. [What is TestContainers?](#what-is-testcontainers)
2. [Step 1: Add Dependencies](#step-1-add-dependencies)
3. [Step 2: Configure TestContainer](#step-2-configure-testcontainer)
4. [Step 3: Write Integration Tests](#step-3-write-integration-tests)
5. [Step 4: Run and Verify Tests](#step-4-run-and-verify-tests)
6. [Understanding Key Concepts](#understanding-key-concepts)
7. [Common Issues and Solutions](#common-issues-and-solutions)

---

## What is TestContainers?

**TestContainers** is a Java library that provides lightweight, throwaway instances of databases, message brokers, and other services that can run in Docker containers during your tests.

### Why Use TestContainers?

✅ **Real Database Testing:** Tests run against actual PostgreSQL/MongoDB, not mocks
✅ **Isolation:** Each test run gets a fresh database
✅ **Confidence:** Catches bugs that unit tests miss
✅ **CI/CD Ready:** Works in automated build pipelines

### How It Works

```
Your Test → TestContainers → Docker Container (PostgreSQL) → Your Application → Test Passes/Fails
```

---

## Step 1: Add Dependencies

### Open your `build.gradle.kts` file

For **order-service**, open:
```
microservices-parent/order-service/build.gradle.kts
```

### Current Dependencies (Before Adding TestContainers)

Your `dependencies {}` block currently looks like this:

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    compileOnly("org.projectlombok:lombok")
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    runtimeOnly("org.postgresql:postgresql")
    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

### Complete Dependencies (After Adding TestContainers)

**Replace your entire `dependencies {}` block with this:**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    compileOnly("org.projectlombok:lombok")
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    runtimeOnly("org.postgresql:postgresql")
    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")

    // ========================================
    // TestContainers Dependencies (NEW)
    // ========================================

    // TestContainers BOM (Bill of Materials) - manages versions
    testImplementation(platform("org.testcontainers:testcontainers-bom:1.21.3"))

    // Spring Boot TestContainers support - REQUIRED FOR @ServiceConnection
    testImplementation("org.springframework.boot:spring-boot-testcontainers")

    // TestContainers PostgreSQL module
    testImplementation("org.testcontainers:postgresql")

    // TestContainers JUnit 5 integration
    testImplementation("org.testcontainers:junit-jupiter")

    // REST Assured for API testing
    testImplementation("io.rest-assured:rest-assured")
}
```

### What Each Dependency Does

| Dependency | Purpose |
|------------|---------|
| `testcontainers-bom` | Manages versions of all TestContainers modules |
| `spring-boot-testcontainers` | Provides @ServiceConnection support for automatic configuration |
| `testcontainers:postgresql` | Provides PostgreSQL container support |
| `testcontainers:junit-jupiter` | Integrates TestContainers with JUnit 5 |
| `rest-assured` | HTTP client for testing REST APIs |

### Reload Gradle Dependencies

**In IntelliJ IDEA:**
1. Click the "Reload Gradle Project" button (elephant icon with circular arrow)
2. Wait for dependencies to download

**Via Command Line:**
```bash
./gradlew --refresh-dependencies
```

---

## Step 2: Configure TestContainer

### Open Your Test File

Navigate to:
```
microservices-parent/order-service/src/test/java/ca/gbc/comp3095/orderservice/OrderServiceApplicationTests.java
```

### Replace the Entire File Contents

Copy and paste this complete test class:

```java
package ca.gbc.comp3095.orderservice;

import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.hamcrest.Matchers;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.http.HttpStatus;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderServiceApplicationTests {

    // PostgreSQL TestContainer - runs once for all tests
    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgresContainer =
        new PostgreSQLContainer<>("postgres:latest")
            .withDatabaseName("order-service-test")
            .withUsername("test")
            .withPassword("test");

    @LocalServerPort
    private Integer port;

    @BeforeEach
    void setUp() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
    }

    static {
        postgresContainer.start();
    }

    @Test
    void placeOrderTest() {
        String requestBody = """
            {
                "orderLineItemDtoList": [
                    {
                        "skuCode": "sku_123456",
                        "price": 999.99,
                        "quantity": 2
                    },
                    {
                        "skuCode": "sku_789012",
                        "price": 499.99,
                        "quantity": 1
                    }
                ]
            }
            """;

        RestAssured.given()
            .contentType(ContentType.JSON)
            .body(requestBody)
            .when()
            .post("/api/order")
            .then()
            .log().all()
            .statusCode(HttpStatus.CREATED.value())
            .body(Matchers.equalTo("Order Placed Successfully"));
    }
}
```

**That's it!** This test class is complete and ready to run

---

## Step 3: Write Integration Tests

### Test Structure: Given-When-Then (BDD Pattern)

```java
RestAssured.given()      // GIVEN: Setup (body, headers)
    .contentType(ContentType.JSON)
    .body(requestBody)
    .when()              // WHEN: Action (HTTP request)
    .post("/api/order")
    .then()              // THEN: Assertions (verify response)
    .statusCode(HttpStatus.CREATED.value());
```

### Basic Test: Place an Order

```java
@Test
void placeOrderTest() {
    // Prepare request body
    String requestBody = """
        {
            "orderLineItemDtoList": [
                {
                    "skuCode": "sku_123456",
                    "price": 999.99,
                    "quantity": 2
                }
            ]
        }
        """;

    // Execute request and verify response
    RestAssured.given()
        .contentType(ContentType.JSON)
        .body(requestBody)
        .when()
        .post("/api/order")
        .then()
        .log().all()
        .statusCode(HttpStatus.CREATED.value())
        .body(Matchers.equalTo("Order Placed Successfully"));
}
```

### Additional Test Examples

#### Test with Multiple Items

```java
@Test
void placeOrderWithMultipleItemsTest() {
    String requestBody = """
        {
            "orderLineItemDtoList": [
                {
                    "skuCode": "laptop_001",
                    "price": 1200.00,
                    "quantity": 1
                },
                {
                    "skuCode": "mouse_002",
                    "price": 25.99,
                    "quantity": 2
                },
                {
                    "skuCode": "keyboard_003",
                    "price": 75.50,
                    "quantity": 1
                }
            ]
        }
        """;

    RestAssured.given()
        .contentType(ContentType.JSON)
        .body(requestBody)
        .when()
        .post("/api/order")
        .then()
        .statusCode(HttpStatus.CREATED.value());
}
```

#### Test Empty Order (Edge Case)

```java
@Test
void placeOrderWithEmptyItemsTest() {
    String requestBody = """
        {
            "orderLineItemDtoList": []
        }
        """;

    RestAssured.given()
        .contentType(ContentType.JSON)
        .body(requestBody)
        .when()
        .post("/api/order")
        .then()
        .statusCode(HttpStatus.CREATED.value());
}
```

---

## Step 4: Run and Verify Tests

### Option 1: Run from IntelliJ

1. Right-click on `OrderServiceApplicationTests` class
2. Select **"Run 'OrderServiceApplicationTests'"**
3. Watch the test console

### Option 2: Run from Command Line

```bash
cd microservices-parent/order-service
./gradlew test
```

### Expected Output

```
OrderServiceApplicationTests > placeOrderTest() STARTED
Creating container for image: postgres:latest
Container postgres:latest is starting...
Container postgres:latest started in PT5.234S
...
OrderServiceApplicationTests > placeOrderTest() PASSED
```

### View Test Report

Open in browser:
```
order-service/build/reports/tests/test/index.html
```

**You should see:**
- ✅ 1 test passed
- ⏱️ Total execution time
- 📊 Test details

---

## Understanding Key Concepts

### 1. Singleton Pattern for Containers

**Why use `static`?**

```java
// ❌ BAD: New container for each test (slow)
PostgreSQLContainer<?> container = new PostgreSQLContainer<>("postgres:latest");

// ✅ GOOD: One container for all tests (fast)
static PostgreSQLContainer<?> container = new PostgreSQLContainer<>("postgres:latest");
```

**Performance comparison:**
- Without static: 3 tests = 15-30 seconds (container starts 3 times)
- With static: 3 tests = 7-10 seconds (container starts once)

### 2. @ServiceConnection Magic

This annotation automatically configures your Spring Boot application:

```java
@ServiceConnection
static PostgreSQLContainer<?> postgresContainer = new PostgreSQLContainer<>("postgres:latest");
```

**What it does automatically:**
- Sets `spring.datasource.url` to the container's JDBC URL
- Sets `spring.datasource.username` to container's username
- Sets `spring.datasource.password` to container's password

**Without @ServiceConnection, you'd need:**

```java
@DynamicPropertySource
static void configureDatabase(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgresContainer::getJdbcUrl);
    registry.add("spring.datasource.username", postgresContainer::getUsername);
    registry.add("spring.datasource.password", postgresContainer::getPassword);
}
```

### 3. BDD Testing Style

**Given-When-Then** makes tests readable:

```java
// GIVEN: I have an order request
String requestBody = "{ ... }";

// WHEN: I POST to /api/order
RestAssured.given()
    .body(requestBody)
    .when()
    .post("/api/order")

// THEN: I expect 201 CREATED status
    .then()
    .statusCode(201);
```

### 4. Docker Requirements

TestContainers requires Docker to be running:

**Check Docker:**
```bash
docker --version
docker ps
```

**If Docker isn't running:**
- Mac: Start Docker Desktop
- Linux: `sudo systemctl start docker`
- Windows: Start Docker Desktop

---

## Common Issues and Solutions

### Issue 1: "Could not find or load main class org.testcontainers..."

**Solution:** Reload Gradle dependencies
```bash
./gradlew --refresh-dependencies
```

### Issue 2: "Container startup failed"

**Solution:** Check Docker is running
```bash
docker ps
```

### Issue 3: "Port already in use"

**Solution:** Use `RANDOM_PORT` (already configured in example)
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
```

### Issue 4: Tests are slow (>30 seconds)

**Solution:** Use static container (already configured in example)
```java
static PostgreSQLContainer<?> postgresContainer = ...
```

### Issue 5: "Connection refused" errors

**Solution:** Ensure PostgreSQL container has enough time to start
```java
static {
    postgresContainer.start();
}
```

### Issue 6: Import errors

**Solution:** Let IntelliJ auto-import (Alt+Enter) or add manually:
```java
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import io.restassured.RestAssured;
```

---

## What's Different for MongoDB (Product-Service)?

If you're working with **product-service** (uses MongoDB), the setup is similar:

```java
@Container
@ServiceConnection
static MongoDBContainer mongoDBContainer = new MongoDBContainer("mongo:latest");
```

**Dependencies difference:**
```kotlin
// For MongoDB
testImplementation("org.testcontainers:mongodb")

// For PostgreSQL
testImplementation("org.testcontainers:postgresql")
```

---

## Next Steps

Once you've completed this lab:

1. ✅ Add more test cases (invalid data, missing fields, etc.)
2. ✅ Test GET, PUT, DELETE endpoints
3. ✅ Add tests for edge cases
4. ✅ Commit your changes to Git

**Git Commit Example:**
```bash
git add .
git commit -m "Add TestContainers integration tests for order-service"
git push
```

---

## Summary

You've learned how to:

✅ Add TestContainers dependencies to a Spring Boot project
✅ Configure PostgreSQL TestContainer
✅ Write integration tests using REST Assured
✅ Use the Singleton pattern for efficient testing
✅ Follow BDD (Given-When-Then) testing style

**Key Takeaway:** Integration tests with TestContainers give you confidence that your application works correctly with real databases, catching bugs that unit tests might miss.

---

## Additional Resources

- **TestContainers Documentation:** https://testcontainers.com/
- **REST Assured Documentation:** https://rest-assured.io/
- **Spring Boot Testing Guide:** https://spring.io/guides/gs/testing-web/

---

**END OF LAB GUIDE**
