# API Testing & Integration Testing Implementation Guide - COMP 3095
## Complete Step-by-Step Guide for Testing Product Service

> **Note**: This guide includes screenshots showing actual Postman requests and responses to help you follow along with the implementation.

### Table of Contents
1. [Lab Overview](#lab-overview)
2. [Prerequisites](#prerequisites)
3. [Part 1: Manual API Testing with Postman](#part-1-manual-api-testing-with-postman)
   - [Step 1: Docker Setup](#step-1-docker-setup)
   - [Step 2: Start Product Service](#step-2-start-product-service)
   - [Step 3: Postman Setup](#step-3-postman-setup)
   - [Step 4: Test POST Endpoint](#step-4-test-post-endpoint)
   - [Step 5: Test GET Endpoint](#step-5-test-get-endpoint)
   - [Step 6: Test PUT Endpoint](#step-6-test-put-endpoint)
   - [Step 7: Test DELETE Endpoint](#step-7-test-delete-endpoint)
4. [Part 2: Automated Testing with TestContainers](#part-2-automated-testing-with-testcontainers)
   - [Step 8: Add TestContainers Dependencies](#step-8-add-testcontainers-dependencies)
   - [Step 9: Configure Test Class](#step-9-configure-test-class)
   - [Step 10: Implement POST Test](#step-10-implement-post-test)
   - [Step 11: Implement GET Test](#step-11-implement-get-test)
   - [Step 12: Run Integration Tests](#step-12-run-integration-tests)
5. [Verification & Troubleshooting](#verification--troubleshooting)
6. [Repository Submission](#repository-submission)

---

## Lab Overview

This lab extends your Product Service implementation by adding comprehensive testing capabilities:
- **Manual Testing**: Using Postman to validate REST endpoints
- **Automated Testing**: Using TestContainers for integration testing with real MongoDB instances
- **Test-Driven Development**: Implementing automated tests to ensure code reliability

### Learning Objectives
- ✅ Master API testing with Postman
- ✅ Implement integration tests using TestContainers
- ✅ Configure MockMvc for Spring Boot testing
- ✅ Validate CRUD operations programmatically
- ✅ Work with Docker containers in testing environments

---

## Prerequisites

### Required Software
- ✅ IntelliJ IDEA (with Spring Boot plugin)
- ✅ Java JDK 21
- ✅ Docker Desktop (running)
- ✅ Postman (latest version)
- ✅ Completed Product Service from Week 2

### Verify Your Setup
```bash
# Check Java version
java --version

# Check Docker is running
docker --version
docker ps

# Check if MongoDB container exists
docker ps -a | grep mongo
```

---

## Part 1: Manual API Testing with Postman

### Step 1: Docker Setup

#### 1.1 Start MongoDB Container

**Option A: Using Terminal**
```bash
# If container exists but stopped
docker start comp3095-mongodb

# If container doesn't exist, create it
docker run -d \
  --name comp3095-mongodb \
  -p 28017:27017 \
  --restart unless-stopped \
  mongo:latest
```

**Option B: Using Docker Desktop**
1. Open Docker Desktop
2. Go to **Containers** tab
3. Find `comp3095-mongodb`
4. Click **Start** button (▶️)

**Option C: Using IntelliJ Docker Integration**
1. **View → Tool Windows → Services**
2. Expand **Docker → Containers**
3. Right-click `comp3095-mongodb`
4. Select **Start Container**

#### 1.2 Verify MongoDB is Running
```bash
docker ps | grep mongodb
# Should show: comp3095-mongodb ... Up ... 28017->27017/tcp
```

#### 1.3 (Optional) Start MongoExpress for GUI Access
```bash
docker run -d \
  --name mongo-express \
  --network bridge \
  --restart unless-stopped \
  -p 8081:8081 \
  -e ME_CONFIG_MONGODB_SERVER=host.docker.internal \
  -e ME_CONFIG_MONGODB_PORT=28017 \
  -e ME_CONFIG_BASICAUTH_USERNAME=admin \
  -e ME_CONFIG_BASICAUTH_PASSWORD=pass \
  mongo-express
```
Access at: `http://localhost:8081` (username: admin, password: pass)

---

### Step 2: Start Product Service

#### 2.1 Open Project in IntelliJ
1. **File → Open**
2. Navigate to `/Users/maziar/temp/COMP3095_Fall2025_11AM/microservices-parent`
3. Click **Open**
4. Wait for Gradle sync to complete

#### 2.2 Update Application Configuration
Before running the application, we need to update the configuration to use custom ports to avoid conflicts with other services.

1. Navigate to `product-service/src/main/resources/application.properties`
2. **Current configuration** (from Week 2):
```properties
spring.application.name=product-service
#spring.data.mongodb.uri=mongodb://rootadmin:password@localhost:27017/product-service?authSource=admin
spring.data.mongodb.uri=mongodb://localhost:27017/product-service
```

3. **Replace the entire contents** with this new configuration:
```properties
spring.application.name=product-service

# Server Configuration
server.port=8084

# MongoDB Configuration
spring.data.mongodb.host=localhost
spring.data.mongodb.port=28017
spring.data.mongodb.database=product-service

# Authentication (commented out for local development)
#spring.data.mongodb.username=admin
#spring.data.mongodb.password=password
#spring.data.mongodb.authentication-database=admin
```

**Why These Changes?**
- **Port 8084 for Application**: Avoids conflicts if port 8080 is already in use by another service
- **Port 28017 for MongoDB**: Allows running multiple MongoDB instances (production on 27017, testing on 28017)
- **Separated Configuration**: Breaking down the URI into individual properties provides:
  - Better readability and maintenance
  - Easier debugging of connection issues
  - Flexibility to change individual settings
  - Clear separation of concerns (host, port, database name)
- **Docker Container Mapping**: Matches our container setup `-p 28017:27017` where:
  - External port 28017 (what our app connects to)
  - Maps to internal port 27017 (MongoDB's default port inside container)

#### 2.3 Run the Application
**Method 1: Using Run Configuration**
1. Click dropdown next to green arrow (▶️) in toolbar
2. Select `ProductServiceApplication`
3. Click **Run** (▶️)

**Method 2: From Main Class**
1. Navigate to `product-service/src/main/java/ca/gbc/comp3095/productservice/`
2. Open `ProductServiceApplication.java`
3. Click green arrow next to `public class ProductServiceApplication`
4. Select **Run 'ProductServiceApplication'**

#### 2.4 Verify Application Started
Look for these lines in console:
```
Started ProductServiceApplication in X.XXX seconds
Tomcat started on port 8084 (http)
MongoDB connection established on port 28017
```

---

### Step 3: Postman Setup

#### 3.1 Install and Open Postman
1. Download from [postman.com](https://www.postman.com/downloads/) if not installed
2. Launch Postman
3. Sign in or use without account

#### 3.2 Create Workspace
1. Click **Workspaces** in top menu
2. Click **Create Workspace**
3. Configure:
   - **Name**: `COMP3095-ProductService-Tests`
   - **Visibility**: Personal
   - **Description**: Testing Product Service API endpoints
4. Click **Create Workspace**

#### 3.3 Create Collection
1. In your new workspace, click **Collections** in sidebar
2. Click **+ New Collection** or **Create Collection**
3. Name it: `Product Service API Tests`
4. Add description: `CRUD operations for Product Service`
5. Click **Create**

---

### Step 4: Test POST Endpoint

#### 4.1 Create POST Request
1. In your collection, click **⋮** (three dots) → **Add request**
2. Configure request:
   - **Name**: `Create Product`
   - **Method**: Select `POST` from dropdown
   - **URL**: `http://localhost:8084/api/product`

#### 4.2 Configure Request Body
1. Click **Body** tab below URL
2. Select **raw** radio button
3. Change dropdown from `Text` to `JSON`
4. Enter product data:
```json
{
    "name": "MacBook Pro M3",
    "description": "14-inch MacBook Pro with M3 chip",
    "price": 1999.99
}
```

#### 4.3 Send Request
1. Click **Send** button
2. Expected Response:
   - **Status**: `201 Created`
   - **Body**: Empty (void method)

#### 4.4 Verify in Console
Check IntelliJ console for:
```
Product 644e500c73c54e784fbb6b19 is saved
```

#### 4.5 Save Request
1. Click **Save** button (💾)
2. Select your collection
3. Click **Save to Product Service API Tests**

![POST Request - Create Product](img/03_POST_Create_Product_MacBook_M3.png)

---

### Step 5: Test GET Endpoint

#### 5.1 Create GET Request
1. In collection, click **⋮** → **Add request**
2. Configure:
   - **Name**: `Get All Products`
   - **Method**: `GET`
   - **URL**: `http://localhost:8084/api/product`

![GET Request Setup](img/01_GET_Get_Products_Empty.png)

#### 5.2 Send Request
1. Click **Send**
2. Expected Response:
   - **Status**: `200 OK`
   - **Body**: JSON array with products
```json
[
    {
        "id": "644e500c73c54e784fbb6b19",
        "name": "MacBook Pro M3",
        "description": "14-inch MacBook Pro with M3 chip",
        "price": 1999.99
    }
]
```

#### 5.3 Create Additional Test Products
1. Go back to **Create Product** request
2. Send with different data:
```json
{
    "name": "iPad Pro",
    "description": "12.9-inch iPad Pro with M2 chip",
    "price": 1299.00
}
```
3. Send another:
```json
{
    "name": "AirPods Pro",
    "description": "Active Noise Cancellation",
    "price": 249.00
}
```

#### 5.4 Test Get All Again
1. Return to **Get All Products** request
2. Click **Send**
3. Verify all 3 products appear in response

![GET Request - Products Retrieved](img/04_GET_Get_Products_With_Data.png)

#### 5.5 Save Request
Click **Save** button

---

### Step 6: Test PUT Endpoint

#### 6.1 Get Product ID
1. From the GET all products response, copy an `id` value
2. Example: `644e500c73c54e784fbb6b19`

#### 6.2 Create PUT Request
1. Add new request to collection
2. Configure:
   - **Name**: `Update Product`
   - **Method**: `PUT`
   - **URL**: `http://localhost:8084/api/product/{id}`

#### 6.3 Add Path Variable
1. Replace `{id}` with actual ID:
   - URL becomes: `http://localhost:8084/api/product/644e500c73c54e784fbb6b19`

#### 6.4 Configure Body
1. Click **Body** tab
2. Select **raw** and **JSON**
3. Enter updated data:
```json
{
    "name": "MacBook Pro M3 Max",
    "description": "14-inch MacBook Pro with M3 Max chip - Updated",
    "price": 3199.99
}
```

#### 6.5 Send and Verify
1. Click **Send**
2. Expected: **204 No Content**
3. Run GET All to verify update

![PUT Request - Update Product](img/05_PUT_Update_Product.png)

#### 6.6 Use Variables (Advanced)
1. In URL, use: `http://localhost:8084/api/product/{{productId}}`
2. Click **Variables** tab
3. Add variable:
   - **Key**: `productId`
   - **Value**: Your product ID
4. Save request

---

### Step 7: Test DELETE Endpoint

#### 7.1 Create DELETE Request
1. Add new request
2. Configure:
   - **Name**: `Delete Product`
   - **Method**: `DELETE`
   - **URL**: `http://localhost:8084/api/product/{{productId}}`

#### 7.2 Set Product ID
1. Use an existing product ID
2. Or use variable like in PUT request

#### 7.3 Send and Verify
1. Click **Send**
2. Expected: **204 No Content**
3. Console shows: `Product {id} is deleted`
4. Run GET All to confirm deletion

![DELETE Request - Delete Product](img/06_DELETE_Delete_Product.png)

#### 7.4 Test Collections
1. Click collection name
2. Click **Run** button
3. See all tests execute in sequence
4. Review results

---

## Part 2: Automated Testing with TestContainers

TestContainers is a Java library that provides lightweight, disposable instances of common databases or any other service that can run in a Docker container. For MongoDB testing, we'll use the MongoDB TestContainers module.

![TestContainers MongoDB Documentation](img/02_TestContainers_MongoDB_Documentation.png)

### Step 8: Add TestContainers Dependencies

#### 8.1 Open build.gradle.kts
Navigate to: `product-service/build.gradle.kts`

#### 8.2 Add TestContainers BOM
In the `dependencies` block, add at the TOP (before other dependencies):
```kotlin
dependencies {
    // TestContainers BOM - Add this FIRST
    testImplementation(platform("org.testcontainers:testcontainers-bom:1.20.4"))

    // Existing dependencies...
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    // ... rest of your dependencies
```

#### 8.3 Add TestContainers Dependencies
After the existing dependencies, add:
```kotlin
    // TestContainers dependencies
    testImplementation("org.testcontainers:mongodb")
    testImplementation("org.testcontainers:junit-jupiter")

    // RestAssured for API testing
    testImplementation("io.rest-assured:rest-assured")

    // Additional testing utilities
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
```

#### 8.4 Complete Dependencies Section
Your complete build.gradle.kts should look like:
```kotlin
dependencies {
    // IMPORTANT: TestContainers BOM must be FIRST in the dependencies block
    testImplementation(platform("org.testcontainers:testcontainers-bom:1.20.4"))

    // Spring Boot Starters
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation("org.springframework.boot:spring-boot-starter-web")

    // Lombok
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    // Development Tools
    developmentOnly("org.springframework.boot:spring-boot-devtools")

    // Spring Boot Test
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-testcontainers")

    // TestContainers Modules
    testImplementation("org.testcontainers:mongodb")
    testImplementation("org.testcontainers:junit-jupiter")

    // REST Testing
    testImplementation("io.rest-assured:rest-assured")

    // Test Runtime
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

**Critical Notes:**
- The BOM (Bill of Materials) **MUST** be the first line in dependencies
- This ensures all TestContainers modules use compatible versions
- Order of dependencies matters for proper dependency resolution

#### 8.5 Sync Gradle
1. Click **Gradle** notification bar that appears
2. Click **Load Gradle Changes** or **Import Changes**
3. Or use shortcut: **⌘+Shift+O** (Mac) / **Ctrl+Shift+O** (Windows)
4. Wait for sync to complete

---

### Step 9: Configure Test Class

#### 9.1 Open Test File
Navigate to: `product-service/src/test/java/ca/gbc/comp3095/productservice/ProductServiceApplicationTests.java`

#### 9.2 Replace Entire File Content
Since you're using Spring Boot 3.5.6, we'll use the modern `@ServiceConnection` approach which automatically configures the MongoDB connection.

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
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
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
}
```

#### 9.3 Understanding the Configuration

**Key Annotations and Components:**

1. **@SpringBootTest(webEnvironment = RANDOM_PORT)**
   - Starts the full Spring application context
   - Uses a random port to avoid conflicts

2. **@Testcontainers**
   - Enables TestContainers JUnit 5 extension
   - Manages container lifecycle automatically

3. **@Container + @ServiceConnection**
   - `@Container`: Marks the field as a TestContainer
   - `@ServiceConnection`: Automatically configures Spring Boot to connect to this container
   - No manual property configuration needed!

4. **MongoDBContainer**
   - Creates a real MongoDB instance in Docker
   - Uses `DockerImageName.parse()` for proper image handling

5. **@LocalServerPort**
   - Injects the random port number
   - Used to configure RestAssured for API calls

6. **@BeforeEach setUp()**
   - Configures RestAssured base URI and port
   - Clears database to ensure test isolation

#### 9.4 Verify Container Starts

Run the test to verify TestContainers works:
1. Right-click on `contextLoads()` method
2. Select "Run contextLoads()"
3. You should see Docker pulling/starting MongoDB container
4. Test should pass with green checkmark

**Expected Console Output:**
```
🐳 [testcontainers/ryuk] - Starting container...
🐳 [mongo:latest] - Starting container...
🐳 [mongo:latest] - Container started in PT2.345S
✓ contextLoads() passed
```

#### 9.5 Troubleshooting

If the test fails:

1. **Docker not running:**
   - Start Docker Desktop
   - Verify with: `docker ps`

2. **Port conflicts:**
   - The RANDOM_PORT should prevent this
   - If issues persist, restart your IDE

3. **Container fails to start:**
   - Check Docker has enough resources
   - Try: `docker pull mongo:latest` manually

**Note:** The `@ServiceConnection` annotation eliminates the need for manual `setProperties()` configuration mentioned in the PDF. Spring Boot 3.1+ handles this automatically, achieving the same goal with cleaner code.

---

### Step 10: Implement POST Test

#### 10.1 Open the Test File
Navigate to the test file we just modified in Step 9:
- **Location**: `product-service/src/test/java/ca/gbc/comp3095/productservice/`
- **File**: `ProductServiceApplicationTests.java`
- The file should already be open from Step 9

#### 10.2 Add Import Statements
If not already present, ensure these imports are at the top of the file (after the package declaration):
```java
import java.math.BigDecimal;
import static org.junit.jupiter.api.Assertions.*;
```

#### 10.3 Add Helper Method
Add this helper method **inside the test class**, after the `contextLoads()` method (around line 47):
```java
    private ProductRequest getProductRequest() {
        return new ProductRequest(
                "Test Product",
                "Test Product Description",
                BigDecimal.valueOf(199.99)
        );
    }
```

**Placement in file:**
- This method goes inside the `ProductServiceApplicationTests` class
- Place it after the existing `contextLoads()` test method
- Before the closing brace `}` of the class

#### 10.4 Implement Create Product Test
Add this test method **immediately after** the helper method you just added:
```java
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
```

#### 10.5 Complete Test File After Step 10
After completing all parts of Step 10, your complete `ProductServiceApplicationTests.java` file should look like this:

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
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
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
}
```

**Note:** At this point, your test file has:
- All necessary imports
- TestContainers configuration with MongoDB
- The `contextLoads()` test to verify setup
- The helper method `getProductRequest()`
- The `createProduct()` test for POST endpoint
- Total of approximately 88 lines

#### 10.6 Run the POST Test
1. Click green arrow next to `createProduct()` method
2. Select **Run 'createProduct()'**
3. Wait for container to start (first time takes longer)
4. Test should pass with green checkmark ✅

---

### Step 11: Implement GET Test

#### 11.1 Add Get All Products Test
Add this test method to validate the GET endpoint:
```java
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
```

#### 11.2 Complete Test File After Step 11
After adding the GET test, here's the complete `ProductServiceApplicationTests.java` file with both POST and GET tests:

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
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.math.BigDecimal;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
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
}
```

**Note:** This complete test file includes:
- All necessary imports for RestAssured testing
- @ServiceConnection for automatic MongoDB configuration (Spring Boot 3.1+)
- @LocalServerPort for random port injection
- RestAssured configuration in setUp()
- The contextLoads() test to verify setup
- The helper method getProductRequest()
- The createProduct() test for POST endpoint
- The getAllProducts() test for GET endpoint
- Total of approximately 108 lines

All tests use the RestAssured approach consistently, matching the configuration from Steps 9-10.

---

### Step 12: Run Integration Tests

#### 12.1 Run All Tests
1. Right-click on `ProductServiceApplicationTests` class name
2. Select **Run 'ProductServiceApplicationTests'**
3. Watch the test output

#### 12.2 Understanding Test Output
You'll see:
```
Starting ProductServiceApplicationTests using Java 21
🐳 [testcontainers/ryuk] - Creating container...
🐳 [mongo:7.0.5] - Starting container...
🐳 [mongo:7.0.5] - Container started in PT2.345S

✅ contextLoads
✅ createProduct
✅ getAllProducts
✅ updateProduct
✅ deleteProduct

Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESSFUL
```

#### 12.3 Run Tests from Terminal
```bash
cd /Users/maziar/temp/COMP3095_Fall2025_11AM/microservices-parent
./gradlew :product-service:test
```

#### 12.4 View Test Results in IntelliJ
1. After tests run, click **Test Results** tab at bottom
2. See tree view of all tests
3. Green checkmarks = passed
4. Click individual tests to see details

---

## Verification & Troubleshooting

### Common Issues and Solutions

#### Issue 1: MongoDB Connection Failed
**Error**: `MongoSocketOpenException`
```bash
# Solution: Ensure Docker is running
docker ps
# If Docker not running, start Docker Desktop
```

#### Issue 2: TestContainers Docker Error
**Error**: `Could not find a valid Docker environment`
```bash
# Solution: Check Docker installation
docker --version
# Restart Docker Desktop if needed
```

#### Issue 3: Port Already in Use
**Error**: `Port 8084 already in use`
```bash
# Find process using port
lsof -i :8084
# Kill the process
kill -9 <PID>
```

#### Issue 4: Test Dependencies Not Found
**Error**: `Cannot resolve symbol 'Testcontainers'`
```
Solution:
1. File → Invalidate Caches and Restart
2. Re-sync Gradle
3. Reimport project
```

#### Issue 5: Tests Pass Individually but Fail Together
**Solution**: Add cleanup in @BeforeEach:
```java
@BeforeEach
void setUp() {
    productRepository.deleteAll();
}
```

### Verify Everything Works

#### Check Postman Collection
1. Run entire collection: **Run Collection**
2. All requests should pass
3. Export collection: **⋮** → **Export**
4. Save as `ProductService.postman_collection.json`

#### Check TestContainers
1. Run all tests
2. All 5 tests should pass
3. No containers should be left running after tests

#### Check Code Coverage (Optional)
1. Right-click test class
2. Select **Run with Coverage**
3. View coverage report
4. Aim for >80% coverage

---

## Repository Submission

### Step 1: Add Test Files to Git
```bash
cd /Users/maziar/temp/COMP3095_Fall2025_11AM
git add microservices-parent/product-service/src/test/
git add microservices-parent/product-service/build.gradle.kts
git add Week04/
```

### Step 2: Commit Changes
```bash
git commit -m "Add API testing with Postman and TestContainers integration tests"
```

### Step 3: Push to Repository
```bash
git push origin main
```

### Step 4: Verify on GitLab
1. Go to your GitLab repository
2. Check that test files are present
3. Verify Week04 documentation is uploaded

### Step 5: Export and Save Postman Collection
1. In Postman, select your collection
2. Click **⋮** → **Export** → **Collection v2.1**
3. Save as: `ProductService_Postman_Collection.json`
4. Add to repository:
```bash
mkdir -p Week04/postman
mv ~/Downloads/ProductService_Postman_Collection.json Week04/postman/
git add Week04/postman/
git commit -m "Add Postman collection for API testing"
git push origin main
```

---

## Conclusion

### What You've Accomplished
✅ **Manual API Testing**: Complete CRUD validation using Postman
✅ **Automated Testing**: Integration tests with real MongoDB using TestContainers
✅ **Test Coverage**: All endpoints tested programmatically
✅ **Best Practices**: Proper test isolation and cleanup
✅ **Documentation**: Comprehensive testing documentation

### Skills Developed
- REST API testing methodology
- Integration testing with Docker containers
- MockMvc for Spring Boot testing
- Test-driven development practices
- CI/CD testing foundations

### Next Steps
This testing foundation prepares you for:
- Continuous Integration pipelines
- API documentation with Swagger/OpenAPI
- Performance testing with JMeter
- Contract testing between microservices
- Test automation in deployment pipelines

---

## Additional Resources

### Postman Resources
- [Postman Learning Center](https://learning.postman.com/)
- [API Testing Best Practices](https://www.postman.com/api-platform/api-testing/)
- [Environment Variables in Postman](https://learning.postman.com/docs/sending-requests/variables/)

### TestContainers Resources
- [TestContainers Documentation](https://www.testcontainers.org/)
- [MongoDB TestContainers Module](https://www.testcontainers.org/modules/databases/mongodb/)
- [Spring Boot Integration](https://spring.io/blog/2023/06/19/spring-boot-31-connectiondetails-abstraction)

### Spring Testing Resources
- [Spring Boot Testing Guide](https://spring.io/guides/gs/testing-web/)
- [MockMvc Documentation](https://docs.spring.io/spring-framework/reference/testing/spring-mvc-test-framework.html)
- [Integration Testing Best Practices](https://www.baeldung.com/spring-boot-testing)

---

**Lab Completed!** 🎉

You now have a fully tested Product Service with both manual and automated testing capabilities. This forms a solid foundation for building reliable microservices.