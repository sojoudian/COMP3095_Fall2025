# Testing - Postman and TestContainers

## Overview

In this section, you will test the order-service using two approaches:
1. **Manual Testing** - Using Postman to test the REST API endpoints
2. **Automated Testing** - Using TestContainers to write integration tests with real PostgreSQL database

---

## Part 1: Manual Testing with Postman

## Step 1: Start order-service

### 1.1 Ensure PostgreSQL is Running

```bash
docker ps | grep postgres
```

**If Not Running:**
```bash
docker start postgres-container
```

### 1.2 Start order-service Application

**In IntelliJ:**
1. Open `OrderServiceApplication.java`
2. Click green play button
3. Wait for "Started OrderServiceApplication" message

**Via Command Line:**
```bash
cd order-service
./gradlew bootRun
```

**Verify Application Started:**
```bash
curl http://localhost:8082/actuator/health
```

**Expected:**
```json
{"status":"UP"}
```

---

## Step 2: Test with Postman

### 2.1 Open Postman

**If First Time:**
1. Open Postman application
2. Skip sign-in (or create free account)

### 2.2 Create New Request

**Steps:**
1. Click **New** button
2. Select **HTTP Request**

### 2.3 Configure Request

**1. Set HTTP Method:**
- Change dropdown to `POST`

**2. Enter URL:**
```
http://localhost:8082/api/order
```

**3. Set Headers:**
- Click **Headers** tab
- Add header:
  - Key: `Content-Type`
  - Value: `application/json`

**4. Set Request Body:**
- Click **Body** tab
- Select **raw** radio button
- Ensure dropdown shows **JSON**
- Enter JSON:

```json
{
  "orderLineItemDtoList": [
    {
      "skuCode": "iphone_15_pro",
      "price": 1299.99,
      "quantity": 1
    },
    {
      "skuCode": "airpods_pro",
      "price": 249.99,
      "quantity": 2
    }
  ]
}
```

### 2.4 Send Request

**Click "Send" Button**

**Expected Response:**

**Status:**
```
201 Created
```

**Response Body:**
```
Order Placed Successfully
```

### 2.5 Save Request to Collection

**Create Collection:**
1. Click **Save** button
2. Click **+ Create Collection**
3. Name: `COMP3095 - Order Service`
4. Click **Create**

**Save Request:**
1. Request Name: `Place Order`
2. Select collection: `COMP3095 - Order Service`
3. Click **Save**

---

## Step 3: Verify Data in Database

### 3.1 Using pgAdmin

**Open pgAdmin:**
```
http://localhost:8888
```

**Login:**
- Email: `admin@admin.com`
- Password: `admin`

**Query orders Table:**
1. Navigate: **Servers → order-service-local → Databases → order-service → Schemas → public → Tables**
2. Right-click **orders** → **View/Edit Data → All Rows**

**Expected Data:**
```
| id | order_number                         |
|----|--------------------------------------|
| 1  | 123e4567-e89b-12d3-a456-426614174000 |
```

**Query order_line_items Table:**
1. Right-click **order_line_items** → **View/Edit Data → All Rows**

**Expected Data:**
```
| id | sku_code        | price   | quantity | order_id |
|----|----------------|---------|----------|----------|
| 1  | iphone_15_pro  | 1299.99 | 1        | 1        |
| 2  | airpods_pro    | 249.99  | 2        | 1        |
```

### 3.2 Using psql

**Connect to Database:**
```bash
docker exec -it postgres-container psql -U admin -d order-service
```

**Query orders:**
```sql
SELECT * FROM orders;
```

**Query order_line_items:**
```sql
SELECT * FROM order_line_items;
```

**Join Query:**
```sql
SELECT
    o.id AS order_id,
    o.order_number,
    oli.sku_code,
    oli.price,
    oli.quantity,
    (oli.price * oli.quantity) AS line_total
FROM orders o
JOIN order_line_items oli ON o.id = oli.order_id;
```

**Exit:**
```sql
\q
```

---

## Step 4: Test Multiple Orders

### 4.1 Place Second Order

**In Postman:**

**Request Body:**
```json
{
  "orderLineItemDtoList": [
    {
      "skuCode": "macbook_pro_16",
      "price": 2499.00,
      "quantity": 1
    }
  ]
}
```

**Send Request**

**Expected:** `201 Created`

### 4.2 Place Third Order

**Request Body:**
```json
{
  "orderLineItemDtoList": [
    {
      "skuCode": "ipad_air",
      "price": 599.99,
      "quantity": 2
    },
    {
      "skuCode": "apple_pencil",
      "price": 129.99,
      "quantity": 2
    }
  ]
}
```

**Send Request**

### 4.3 Verify All Orders

**Query All Orders:**
```sql
SELECT
    o.id,
    o.order_number,
    COUNT(oli.id) AS item_count,
    SUM(oli.quantity) AS total_quantity,
    SUM(oli.price * oli.quantity) AS order_total
FROM orders o
JOIN order_line_items oli ON o.id = oli.order_id
GROUP BY o.id, o.order_number
ORDER BY o.id;
```

---

## Part 2: Automated Testing with TestContainers

## Step 5: Add TestContainers Dependencies

### 5.1 Update build.gradle.kts

**Location:** `order-service/build.gradle.kts`

**Add TestContainers Dependencies:**

```kotlin
dependencies {
    // Spring Boot Starters
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")

    // Flyway Migration
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    // PostgreSQL Driver
    runtimeOnly("org.postgresql:postgresql")

    // Lombok
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    // DevTools
    developmentOnly("org.springframework.boot:spring-boot-devtools")

    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("org.testcontainers:postgresql")
    testImplementation("io.rest-assured:rest-assured:5.5.0")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

### 5.2 Sync Gradle Project

**In IntelliJ:**
1. Click **Gradle** notification
2. Or click **Reload All Gradle Projects** button in Gradle tab

**Via Command Line:**
```bash
./gradlew clean build --refresh-dependencies
```

---

## Step 6: Create Integration Test

### 6.1 Locate Test Class

**File:** `order-service/src/test/java/ca/gbc/comp3095/orderservice/OrderServiceApplicationTests.java`

### 6.2 Write Complete Integration Test

**Replace with:**

```java
package ca.gbc.comp3095.orderservice;

import io.restassured.RestAssured;
import org.hamcrest.Matchers;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.hamcrest.MatcherAssert.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderServiceApplicationTests {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgreSQLContainer = new PostgreSQLContainer<>("postgres:latest")
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

    @Test
    void shouldPlaceOrder() {
        String requestBody = """
                {
                    "orderLineItemDtoList": [
                        {
                            "skuCode": "iphone_15_pro",
                            "price": 1299.99,
                            "quantity": 1
                        },
                        {
                            "skuCode": "airpods_pro",
                            "price": 249.99,
                            "quantity": 2
                        }
                    ]
                }
                """;

        RestAssured.given()
                .contentType("application/json")
                .body(requestBody)
                .when()
                .post("/api/order")
                .then()
                .log().all()
                .statusCode(201)
                .body(Matchers.equalTo("Order Placed Successfully"));
    }

    @Test
    void shouldPlaceOrderWithSingleItem() {
        String requestBody = """
                {
                    "orderLineItemDtoList": [
                        {
                            "skuCode": "macbook_pro_16",
                            "price": 2499.00,
                            "quantity": 1
                        }
                    ]
                }
                """;

        RestAssured.given()
                .contentType("application/json")
                .body(requestBody)
                .when()
                .post("/api/order")
                .then()
                .log().all()
                .statusCode(201)
                .body(Matchers.equalTo("Order Placed Successfully"));
    }

    @Test
    void shouldPlaceOrderWithMultipleItems() {
        String requestBody = """
                {
                    "orderLineItemDtoList": [
                        {
                            "skuCode": "ipad_air",
                            "price": 599.99,
                            "quantity": 2
                        },
                        {
                            "skuCode": "apple_pencil",
                            "price": 129.99,
                            "quantity": 2
                        },
                        {
                            "skuCode": "magic_keyboard",
                            "price": 299.99,
                            "quantity": 1
                        }
                    ]
                }
                """;

        RestAssured.given()
                .contentType("application/json")
                .body(requestBody)
                .when()
                .post("/api/order")
                .then()
                .log().all()
                .statusCode(201)
                .body(Matchers.equalTo("Order Placed Successfully"));
    }

    @Test
    void shouldPlaceEmptyOrder() {
        String requestBody = """
                {
                    "orderLineItemDtoList": []
                }
                """;

        RestAssured.given()
                .contentType("application/json")
                .body(requestBody)
                .when()
                .post("/api/order")
                .then()
                .log().all()
                .statusCode(201)
                .body(Matchers.equalTo("Order Placed Successfully"));
    }
}
```

---

## Step 7: Run Integration Tests

### 7.1 Run Tests in IntelliJ

**Steps:**
1. Right-click `OrderServiceApplicationTests.java`
2. Select **Run 'OrderServiceApplicationTests'**
3. Watch Test Results window

**Expected Output:**
```
OrderServiceApplicationTests > shouldPlaceOrder() PASSED
OrderServiceApplicationTests > shouldPlaceOrderWithSingleItem() PASSED
OrderServiceApplicationTests > shouldPlaceOrderWithMultipleItems() PASSED
OrderServiceApplicationTests > shouldPlaceEmptyOrder() PASSED

BUILD SUCCESSFUL
4 tests completed, 4 passed
```

### 7.2 Run Tests via Command Line

```bash
cd order-service
./gradlew test
```

**Expected Output:**
```
> Task :test

OrderServiceApplicationTests > shouldPlaceOrder() PASSED
OrderServiceApplicationTests > shouldPlaceOrderWithSingleItem() PASSED
OrderServiceApplicationTests > shouldPlaceOrderWithMultipleItems() PASSED
OrderServiceApplicationTests > shouldPlaceEmptyOrder() PASSED

BUILD SUCCESSFUL
```

### 7.3 View Test Report

**Location:**
```
order-service/build/reports/tests/test/index.html
```

**Open in Browser:**
```bash
open order-service/build/reports/tests/test/index.html
```

---

## Troubleshooting

### Issue 1: TestContainers Cannot Start Container

**Error:**
```
Could not start container
```

**Solution:**

**Verify Docker is Running:**
```bash
docker ps
```

### Issue 2: Tests Timeout

**Error:**
```
Container startup failed after timeout
```

**Solution:**

**Check Docker Resources:**
- Docker Desktop → Settings → Resources
- Increase Memory to 4GB+

### Issue 3: REST Assured Cannot Connect

**Error:**
```
java.net.ConnectException: Connection refused
```

**Solution:**

**Verify Port Injection:**
```java
@LocalServerPort
private Integer port;

@BeforeEach
void setUp() {
    RestAssured.port = port;  // Must be set!
}
```

---

## Summary

### What You Completed:

**Manual Testing with Postman:**
- ✅ Created POST requests to /api/order endpoint
- ✅ Sent multiple test orders
- ✅ Verified 201 Created responses
- ✅ Inspected persisted data in PostgreSQL
- ✅ Saved requests in Postman collection

**Database Verification:**
- ✅ Verified orders table contains order data
- ✅ Verified order_line_items table contains line item data
- ✅ Confirmed foreign key relationships working
- ✅ Executed JOIN queries

**Automated Testing with TestContainers:**
- ✅ Added TestContainers dependencies
- ✅ Created integration test class with 4 test cases
- ✅ Configured PostgreSQL TestContainer
- ✅ Implemented REST Assured API tests
- ✅ Verified all tests pass

**Test Coverage:**
- ✅ Single order item test
- ✅ Multiple order items test
- ✅ Empty order test

### Files Created/Modified:

**Modified:**
- ✅ build.gradle.kts - Added TestContainers and REST Assured dependencies
- ✅ OrderServiceApplicationTests.java - Complete integration test suite

**Test Results:**
```
Total Tests: 4
Passed: 4 ✅
Failed: 0
```

---

## Next Steps

Continue to [08-deployment.md](08-deployment.md)

---

**Testing Complete! All endpoints verified and integration tests passing.**
