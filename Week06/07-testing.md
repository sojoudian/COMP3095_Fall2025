# Testing - Postman and TestContainers

## Overview

In this section, you will test the order-service using two approaches:
1. **Manual Testing** - Using Postman to test the REST API endpoints
2. **Automated Testing** - Using TestContainers to write integration tests with real PostgreSQL database

---

## Part 1: Manual Testing with Postman

### Understanding the API Endpoint

**Endpoint Details:**
```
Method: POST
URL: http://localhost:8082/api/order
Content-Type: application/json
Body: OrderRequest JSON
```

**Request Structure:**
```json
{
  "orderLineItemDtoList": [
    {
      "skuCode": "product_sku_code",
      "price": 99.99,
      "quantity": 2
    }
  ]
}
```

**Expected Response:**
```
Status: 201 Created
Body: "Order Placed Successfully"
```

---

## Step 1: Start order-service

### 1.1 Ensure PostgreSQL is Running

**Check Container Status:**
```bash
docker ps | grep postgres
```

**Expected:**
```
CONTAINER ID   IMAGE             STATUS         PORTS                    NAMES
a1b2c3d4e5f6   postgres:latest   Up 10 minutes  0.0.0.0:5433->5432/tcp   postgres-container
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

**Check Logs for:**
```
Hibernate: create table if not exists orders (...)
Hibernate: create table if not exists order_line_items (...)
Started OrderServiceApplication in 5.321 seconds (process running on 12345)
```

**Test Actuator:**
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
3. Close any tutorial popups

### 2.2 Create New Request

**Steps:**
1. Click **New** button (top left)
2. Select **HTTP Request**
3. Or click **+** tab to create new request

### 2.3 Configure Request

**Request Configuration:**

**1. Set HTTP Method:**
- Change dropdown from `GET` to `POST`

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

**Response Time:**
```
~200-500 ms
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
http://localhost:5050
```

**Login:**
- Email: `admin@admin.com`
- Password: `admin`

**Navigate to Data:**
1. Expand: **Servers → order-service-local**
2. Expand: **Databases → order-service**
3. Expand: **Schemas → public**
4. Expand: **Tables**

**Query orders Table:**
1. Right-click **orders** → **View/Edit Data → All Rows**

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

**Verify Relationship:**
- Both order line items have `order_id = 1`
- This references the order with `id = 1`
- One-to-many relationship is working correctly

### 3.2 Using psql

**Connect to Database:**
```bash
docker exec -it postgres-container psql -U admin -d order-service
```

**Query orders:**
```sql
SELECT * FROM orders;
```

**Expected Output:**
```
 id |             order_number
----+--------------------------------------
  1 | 123e4567-e89b-12d3-a456-426614174000
(1 row)
```

**Query order_line_items:**
```sql
SELECT * FROM order_line_items;
```

**Expected Output:**
```
 id |    sku_code    |  price  | quantity | order_id
----+----------------+---------+----------+----------
  1 | iphone_15_pro  | 1299.99 |        1 |        1
  2 | airpods_pro    |  249.99 |        2 |        1
(2 rows)
```

**Join Query to See Complete Order:**
```sql
SELECT
    o.id AS order_id,
    o.order_number,
    oli.id AS line_item_id,
    oli.sku_code,
    oli.price,
    oli.quantity,
    (oli.price * oli.quantity) AS line_total
FROM orders o
JOIN order_line_items oli ON o.id = oli.order_id
ORDER BY o.id, oli.id;
```

**Expected Output:**
```
 order_id |             order_number              | line_item_id |    sku_code    |  price  | quantity | line_total
----------+---------------------------------------+--------------+----------------+---------+----------+------------
        1 | 123e4567-e89b-12d3-a456-426614174000 |            1 | iphone_15_pro  | 1299.99 |        1 |    1299.99
        1 | 123e4567-e89b-12d3-a456-426614174000 |            2 | airpods_pro    |  249.99 |        2 |     499.98
(2 rows)
```

**Calculate Order Total:**
```sql
SELECT
    o.order_number,
    SUM(oli.price * oli.quantity) AS order_total
FROM orders o
JOIN order_line_items oli ON o.id = oli.order_id
GROUP BY o.order_number;
```

**Expected Output:**
```
             order_number              | order_total
---------------------------------------+-------------
 123e4567-e89b-12d3-a456-426614174000 |     1799.97
(1 row)
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

**Expected Response:**
```
201 Created
"Order Placed Successfully"
```

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
    },
    {
      "skuCode": "magic_keyboard",
      "price": 299.99,
      "quantity": 1
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

**Expected Output:**
```
 id |             order_number              | item_count | total_quantity | order_total
----+---------------------------------------+------------+----------------+-------------
  1 | 123e4567-e89b-12d3-a456-426614174000 |          2 |              3 |     1799.97
  2 | 987f6543-e21c-34b5-d678-538726185111 |          1 |              1 |     2499.00
  3 | 456a1234-b67d-89e1-f234-649837296222 |          3 |              5 |     1759.95
(3 rows)
```

---

## Step 5: Test Edge Cases

### 5.1 Empty Order (Should Work)

**Request Body:**
```json
{
  "orderLineItemDtoList": []
}
```

**Expected:**
- ✅ 201 Created
- Order created with no line items

**Verify:**
```sql
SELECT * FROM orders WHERE id NOT IN (SELECT DISTINCT order_id FROM order_line_items WHERE order_id IS NOT NULL);
```

### 5.2 Missing Required Fields (Should Fail)

**Request Body:**
```json
{
  "orderLineItemDtoList": [
    {
      "skuCode": "test_product"
      // Missing price and quantity
    }
  ]
}
```

**Expected:**
- ❌ 400 Bad Request (if validation added)
- Or ❌ 500 Internal Server Error (database constraint violation)

**Note:** This will fail because `price` and `quantity` are NOT NULL in database.

### 5.3 Negative Price (Should Be Handled)

**Request Body:**
```json
{
  "orderLineItemDtoList": [
    {
      "skuCode": "test_product",
      "price": -100.00,
      "quantity": 1
    }
  ]
}
```

**Current Behavior:**
- ✅ Accepts negative price (no validation in current implementation)

**Best Practice:** Add validation in OrderLineItemDto:
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderLineItemDto {
    private Long id;

    @NotBlank(message = "SKU code is required")
    private String skuCode;

    @DecimalMin(value = "0.01", message = "Price must be positive")
    private BigDecimal price;

    @Min(value = 1, message = "Quantity must be at least 1")
    private Integer quantity;
}
```

**Update Controller to Enable Validation:**
```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public String placeOrder(@Valid @RequestBody OrderRequest orderRequest) {
    orderService.placeOrder(orderRequest);
    return "Order Placed Successfully";
}
```

**Add Dependency to build.gradle.kts:**
```kotlin
implementation("org.springframework.boot:spring-boot-starter-validation")
```

---

## Part 2: Automated Testing with TestContainers

### Understanding TestContainers

**What is TestContainers?**
- Java library for integration testing
- Spins up real Docker containers during tests
- Provides real database environment
- Automatically cleans up after tests

**Benefits:**
- ✅ Test with real database (not H2 mock)
- ✅ Exact same database as production (PostgreSQL)
- ✅ Automatic container lifecycle management
- ✅ Isolated test environment
- ✅ Parallel test execution support

**How It Works:**
```
Test Starts
    ↓
TestContainers starts PostgreSQL container
    ↓
Application connects to test database
    ↓
Test executes
    ↓
Test completes
    ↓
TestContainers stops and removes container
```

---

## Step 6: Add TestContainers Dependencies

### 6.1 Update build.gradle.kts

**Location:** `order-service/build.gradle.kts`

**Add TestContainers Dependencies:**

```kotlin
dependencies {
    // ... existing dependencies ...

    // TestContainers
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("org.testcontainers:postgresql")

    // REST Assured for API testing
    testImplementation("io.rest-assured:rest-assured:5.5.0")

    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

**Complete dependencies Section:**

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

### 6.2 Sync Gradle Project

**In IntelliJ:**
1. Click **Gradle** notification that appears
2. Or click **Reload All Gradle Projects** button in Gradle tab
3. Wait for dependencies to download

**Via Command Line:**
```bash
./gradlew clean build --refresh-dependencies
```

---

## Step 7: Create Integration Test

### 7.1 Locate Test Class

**File:** `order-service/src/test/java/ca/gbc/comp3095/orderservice/OrderServiceApplicationTests.java`

**Generated Content (Delete and Replace):**
```java
package ca.gbc.comp3095.orderservice;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class OrderServiceApplicationTests {

    @Test
    void contextLoads() {
    }

}
```

### 7.2 Write Complete Integration Test

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

### 7.3 Understanding the Test Code

**Class Annotations:**

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
```
- Starts full Spring Boot application
- Uses random available port (avoids conflicts)
- Complete application context loaded

```java
@Testcontainers
```
- Enables TestContainers support
- Manages container lifecycle automatically

**TestContainer Setup:**

```java
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgreSQLContainer = new PostgreSQLContainer<>("postgres:latest")
        .withDatabaseName("order-service-test")
        .withUsername("test")
        .withPassword("test");
```

**Explanation:**
- `@Container` - Marks field as TestContainer
- `@ServiceConnection` - Auto-configures Spring datasource to use this container
- `static` - Container shared across all tests in class (faster)
- `.withDatabaseName()` - Creates database in container
- Container starts before any test runs
- Container stops after all tests complete

**Port Injection:**

```java
@LocalServerPort
private Integer port;
```
- Injects the random port Spring Boot is using
- Allows REST Assured to connect to correct port

**REST Assured Setup:**

```java
@BeforeEach
void setUp() {
    RestAssured.baseURI = "http://localhost";
    RestAssured.port = port;
}
```
- Runs before each test method
- Configures REST Assured with application URL and port

**Test Method Structure:**

```java
@Test
void shouldPlaceOrder() {
    String requestBody = """
            { ... JSON ... }
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
```

**Breakdown:**
- `given()` - Request setup
  - `.contentType()` - Sets Content-Type header
  - `.body()` - Sets request body
- `when()` - Action
  - `.post()` - HTTP POST to endpoint
- `then()` - Assertions
  - `.log().all()` - Logs response (helpful for debugging)
  - `.statusCode(201)` - Assert HTTP 201
  - `.body()` - Assert response body content

---

## Step 8: Run Integration Tests

### 8.1 Run Tests in IntelliJ

**Steps:**
1. Right-click `OrderServiceApplicationTests.java`
2. Select **Run 'OrderServiceApplicationTests'**
3. Watch Test Results window

**What Happens:**
1. TestContainers pulls `postgres:latest` image (if not already present)
2. TestContainers starts PostgreSQL container
3. Spring Boot application starts
4. Application connects to test database
5. JPA creates tables (orders, order_line_items)
6. Tests execute one by one
7. Tests complete
8. Spring Boot application stops
9. TestContainers stops and removes container

**Expected Output:**

```
OrderServiceApplicationTests > shouldPlaceOrder() PASSED
OrderServiceApplicationTests > shouldPlaceOrderWithSingleItem() PASSED
OrderServiceApplicationTests > shouldPlaceOrderWithMultipleItems() PASSED
OrderServiceApplicationTests > shouldPlaceEmptyOrder() PASSED

BUILD SUCCESSFUL in 25s
4 tests completed, 4 passed
```

### 8.2 Run Tests via Command Line

**Run All Tests:**
```bash
cd order-service
./gradlew test
```

**Run with Detailed Output:**
```bash
./gradlew test --info
```

**Expected Output:**
```
> Task :test

OrderServiceApplicationTests > shouldPlaceOrder() PASSED
OrderServiceApplicationTests > shouldPlaceOrderWithSingleItem() PASSED
OrderServiceApplicationTests > shouldPlaceOrderWithMultipleItems() PASSED
OrderServiceApplicationTests > shouldPlaceEmptyOrder() PASSED

BUILD SUCCESSFUL in 28s
```

### 8.3 View Test Report

**Location:**
```
order-service/build/reports/tests/test/index.html
```

**Open in Browser:**
```bash
open order-service/build/reports/tests/test/index.html
```

**Report Shows:**
- Total tests: 4
- Passed: 4 (green)
- Failed: 0
- Duration: ~25 seconds
- Detailed logs for each test

---

## Step 9: Understanding Test Logs

### 9.1 Container Startup Logs

**Look for in Console:**

```
Creating container for image: postgres:latest
Container postgres:latest is starting: a1b2c3d4e5f6
Container postgres:latest started in PT5.234S
Mapped port 5432 (container) → 51234 (host)
```

**What This Means:**
- TestContainers created container
- PostgreSQL started in ~5 seconds
- PostgreSQL running on random host port (51234)
- Application will connect to this port

### 9.2 Application Startup Logs

```
HikariPool-1 - Starting...
HikariPool-1 - Start completed.
Hibernate: create table if not exists orders (...)
Hibernate: create table if not exists order_line_items (...)
Started OrderServiceApplicationTests in 8.456 seconds
```

**What This Means:**
- Spring Boot connected to test database
- JPA created tables automatically
- Application ready for testing

### 9.3 Test Execution Logs

**When `log().all()` is enabled:**

```
Request method:   POST
Request URI:      http://localhost:51234/api/order
Headers:          Content-Type=application/json
Body:
{
    "orderLineItemDtoList": [
        {
            "skuCode": "iphone_15_pro",
            "price": 1299.99,
            "quantity": 1
        }
    ]
}

Response:
HTTP/1.1 201
Content-Type: text/plain;charset=UTF-8
Content-Length: 25

Order Placed Successfully
```

**What This Means:**
- REST Assured sent POST request
- Application processed request
- Returned 201 Created
- Response body matches expected

### 9.4 Container Cleanup Logs

```
Stopping container: a1b2c3d4e5f6
Removing container: a1b2c3d4e5f6
```

**What This Means:**
- TestContainers stopped container
- TestContainers deleted container
- No leftover containers

---

## Step 10: Verify Test Data Isolation

### 10.1 Run Tests Multiple Times

**Run 1:**
```bash
./gradlew test
```

**Expected:** ✅ All tests pass

**Run 2 (Immediately After):**
```bash
./gradlew test
```

**Expected:** ✅ All tests pass again

**Why?**
- Each test run uses fresh container
- No data persists between test runs
- Complete isolation

### 10.2 Verify No Impact on Local Database

**Check your local PostgreSQL container:**

```bash
docker exec -it postgres-container psql -U admin -d order-service -c "SELECT COUNT(*) FROM orders;"
```

**Expected:**
- Shows only orders from Postman tests
- TestContainers used separate container
- Local database unaffected

---

## Troubleshooting

### Issue 1: TestContainers Cannot Start Container

**Error:**
```
Could not start container
org.testcontainers.containers.ContainerLaunchException
```

**Solution:**

**A. Verify Docker is Running:**
```bash
docker ps
```

**B. Ensure Docker Socket is Accessible:**
```bash
# Mac/Linux
ls -la /var/run/docker.sock

# Should show readable socket file
```

**C. Check Docker Permissions:**
```bash
docker run hello-world
```

### Issue 2: Tests Timeout

**Error:**
```
Container startup failed after timeout
```

**Solution:**

**A. Increase Timeout (in test class):**
```java
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgreSQLContainer = new PostgreSQLContainer<>("postgres:latest")
        .withDatabaseName("order-service-test")
        .withUsername("test")
        .withPassword("test")
        .withStartupTimeout(Duration.ofMinutes(5));
```

**B. Check Docker Resources:**
- Docker Desktop → Settings → Resources
- Increase Memory to 4GB+
- Increase CPUs to 2+

### Issue 3: Port Already in Use

**Error:**
```
Bind for 0.0.0.0:5432 failed: port is already allocated
```

**Solution:**

**TestContainers uses random ports automatically**
- This error means TestContainers can't find available port
- Usually indicates Docker networking issue

**Fix:**
```bash
# Restart Docker
# Mac: Docker Desktop → Restart

# Or reset Docker networks
docker network prune
```

### Issue 4: Tests Pass but Application Logs Show Errors

**Error in Logs:**
```
org.hibernate.exception.ConstraintViolationException
```

**Solution:**

**A. Check Request Body:**
- Ensure all required fields present
- Verify data types match entity

**B. Check Entity Constraints:**
```java
@Column(nullable = false)  // This field is required!
```

### Issue 5: REST Assured Cannot Connect

**Error:**
```
java.net.ConnectException: Connection refused
```

**Solution:**

**A. Verify Port Injection:**
```java
@LocalServerPort
private Integer port;

@BeforeEach
void setUp() {
    RestAssured.port = port;  // Must be set!
}
```

**B. Check Application Started:**
- Look for "Started OrderServiceApplicationTests" in logs
- If missing, Spring Boot didn't start

### Issue 6: Gradle Cannot Resolve TestContainers

**Error:**
```
Could not find org.testcontainers:postgresql
```

**Solution:**

**A. Ensure Correct Version:**
```kotlin
testImplementation("org.springframework.boot:spring-boot-testcontainers")
testImplementation("org.testcontainers:junit-jupiter")
testImplementation("org.testcontainers:postgresql")
```

**B. Refresh Dependencies:**
```bash
./gradlew clean build --refresh-dependencies
```

---

## Summary

### What You Completed:

**Manual Testing with Postman:**
- ✅ Created POST requests to /api/order endpoint
- ✅ Sent multiple test orders with various scenarios
- ✅ Verified 201 Created responses
- ✅ Inspected persisted data in PostgreSQL
- ✅ Executed SQL queries to verify relationships
- ✅ Saved requests in Postman collection

**Database Verification:**
- ✅ Verified orders table contains order data
- ✅ Verified order_line_items table contains line item data
- ✅ Confirmed foreign key relationships working correctly
- ✅ Executed JOIN queries to see complete orders
- ✅ Calculated order totals with aggregate queries

**Automated Testing with TestContainers:**
- ✅ Added TestContainers dependencies to build.gradle.kts
- ✅ Created integration test class with 4 test cases
- ✅ Configured PostgreSQL TestContainer
- ✅ Implemented REST Assured API tests
- ✅ Verified all tests pass (green checkmarks)
- ✅ Confirmed test isolation (no data leakage)
- ✅ Validated automatic container lifecycle management

**Test Coverage:**
- ✅ Single order item test
- ✅ Multiple order items test
- ✅ Empty order test
- ✅ Order with various product types

### Key Concepts Learned:

- ✅ REST API testing with Postman
- ✅ HTTP status codes (201 Created)
- ✅ JSON request/response handling
- ✅ SQL queries for data verification
- ✅ JOIN operations in relational databases
- ✅ TestContainers for integration testing
- ✅ REST Assured for API testing
- ✅ Test isolation and container cleanup
- ✅ Given-When-Then test structure
- ✅ Test automation best practices

### Files Created/Modified:

**Modified:**
- ✅ build.gradle.kts - Added TestContainers and REST Assured dependencies

**Replaced:**
- ✅ OrderServiceApplicationTests.java - Complete integration test suite

**Created in Postman:**
- ✅ COMP3095 - Order Service collection
- ✅ Place Order request

### Test Results:

```
Total Tests: 4
Passed: 4 ✅
Failed: 0
Duration: ~25 seconds
```

---

## Next Steps

You have successfully tested the order-service using both manual and automated approaches. Next, you will:

1. Create Dockerfile for order-service
2. Build Docker image
3. Update docker-compose.yml to include all services
4. Deploy complete microservices architecture

Continue to [08-deployment.md](08-deployment.md)

---

**Testing Complete! All endpoints verified and integration tests passing.**
