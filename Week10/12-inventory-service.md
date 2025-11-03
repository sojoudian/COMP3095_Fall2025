# Inventory Service Setup Guide (Production Approach)

## Introduction

This guide walks you through creating a new microservice called **inventory-service** that tracks product stock levels using **production-ready practices** including Flyway database migrations.

**What you'll build:**
- New Spring Boot microservice
- PostgreSQL database integration with Flyway migrations
- REST API to check inventory
- Docker containerization
- Integration tests with TestContainers

**Time:** 90-120 minutes

**Key Difference from Week 7:**
- Uses **Flyway migrations** for database schema and data management (production standard)
- No CommandLineRunner/DataLoader approach

---

## Step 1: Create Inventory Service

### 1.1 Generate Project

Go to [Spring Initializr](https://start.spring.io/):

**Project Settings:**
- Project: Gradle - Kotlin
- Language: Java
- Spring Boot: 3.4.4
- Java: 22
- Group: `ca.gbc`
- Artifact: `inventory-service`
- Name: `inventory-service`
- Package name: `ca.gbc.inventoryservice`
- Packaging: Jar

**Dependencies:**
- Lombok
- Spring Web
- Spring Data JPA
- PostgreSQL Driver
- Spring Boot DevTools
- Spring Boot Actuator
- Testcontainers

Click **Generate** and extract the zip file.

### 1.2 Add to Multi-Module Project

Move the `inventory-service` folder to:
```
microservices-parent/inventory-service/
```

Update `microservices-parent/settings.gradle.kts`:

```kotlin
rootProject.name = "microservices-parent"

include("product-service")
include("order-service")
include("inventory-service")
```

---

## Step 2: Package Structure

Create these packages in `src/main/java/ca/gbc/inventoryservice/`:

```
inventoryservice/
├── controller/
├── service/
├── model/
└── repository/
```

**Note:** No `bootstrap` package needed - we'll use Flyway migrations instead.

Also create this directory for database migrations:

```
src/main/resources/
└── db/
    └── migration/
```

---

## Step 3: Configure PostgreSQL

### 3.1 Update application.properties

Open `src/main/resources/application.properties`:

```properties
# Application Configuration
spring.application.name=inventory-service

# Server Configuration
server.port=8083

# PostgreSQL Configuration for LOCAL development
spring.datasource.url=jdbc:postgresql://localhost:5432/inventory_service
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA/Hibernate Configuration
# IMPORTANT: Use 'none' because Flyway manages schema
spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Actuator Configuration
management.endpoints.web.exposure.include=health,info
```

### 3.2 Create application-docker.properties

Create `src/main/resources/application-docker.properties`:

```properties
# Application Configuration
spring.application.name=inventory-service

# Server Configuration
server.port=8083

# PostgreSQL Configuration for DOCKER
spring.datasource.url=jdbc:postgresql://postgres-inventory:5432/inventory_service
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA/Hibernate Configuration
# IMPORTANT: Use 'none' because Flyway manages schema
spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Actuator Configuration
management.endpoints.web.exposure.include=health,info
```

**Key Point:** `ddl-auto=none` because Flyway will manage the database schema, not JPA/Hibernate.

---

## Step 4: Update build.gradle.kts

### 4.1 Add Flyway Dependencies

Open `inventory-service/build.gradle.kts`

Update dependencies section:

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")

    // Flyway for Database Migrations
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    compileOnly("org.projectlombok:lombok")
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    runtimeOnly("org.postgresql:postgresql")
    annotationProcessor("org.projectlombok:lombok")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")

    // TestContainers Dependencies
    testImplementation(platform("org.testcontainers:testcontainers-bom:1.21.3"))
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("org.testcontainers:postgresql")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("io.rest-assured:rest-assured")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

**Reload Gradle** after making changes.

**What is Flyway?**
- Database migration tool
- Version control for your database
- Manages schema changes and data seeding
- Production standard for database management

---

## Step 5: Create Inventory Model

**In IntelliJ:** Right-click on `model` package → New → Java Class → Select **Class**

Create `model/Inventory.java`:

```java
package ca.gbc.inventoryservice.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "t_inventory")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Inventory {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String skuCode;

    private Integer quantity;
}
```

**What this does:**
- Represents `t_inventory` table in database
- Each row has: id, skuCode, quantity
- Auto-generates primary key
- **Note:** No `@Builder` annotation - we'll use SQL to insert data

---

## Step 6: Create Inventory Repository

**In IntelliJ:** Right-click on `repository` package → New → Java Class → Select **Interface**

Create `repository/InventoryRepository.java`:

```java
package ca.gbc.inventoryservice.repository;

import ca.gbc.inventoryservice.model.Inventory;
import org.springframework.data.jpa.repository.JpaRepository;

/**
 * The InventoryRepository interface extends JpaRepository to provide basic CRUD operations
 * for the Inventory entity. JpaRepository is part of Spring Data JPA, which simplifies
 * the process of working with relational databases by generating queries from method names.
 */
public interface InventoryRepository extends JpaRepository<Inventory, Long> {

    /**
     * Custom query method generated by Spring Data JPA. Based on the method name, Spring Data JPA
     * automatically derives the appropriate SQL query to check if an inventory item with the given
     * SKU code exists and whether its quantity is greater than or equal to the specified amount.
     *
     * Method name breakdown:
     * - "existsBy": This tells Spring Data JPA that we're checking for the existence of records.
     * - "SkuCode": Refers to the 'skuCode' field in the Inventory entity, which will be matched
     *    against the provided 'skuCode' parameter.
     * - "AndQuantityGreaterThanEqual": This tells Spring Data JPA to also compare the 'quantity'
     *    field in the entity, ensuring it is greater than or equal to the provided 'quantity' parameter.
     *
     * Spring Data JPA will automatically translate this method into an SQL query that looks something like:
     *
     * SELECT CASE WHEN COUNT(i) > 0 THEN TRUE ELSE FALSE END
     * FROM t_inventory i
     * WHERE i.sku_code = ?1 AND i.quantity >= ?2;
     *
     * @param skuCode the SKU code of the inventory item.
     * @param quantity the minimum quantity required to determine if the item is in stock.
     * @return true if an item with the given SKU code exists and its quantity is greater than or equal to the given value, false otherwise.
     */
    boolean existsBySkuCodeAndQuantityGreaterThanEqual(String skuCode, Integer quantity);
}
```

**What this does:**
- Provides database operations (save, find, delete)
- Custom method: `existsBySkuCodeAndQuantityGreaterThanEqual()`
- Spring Data JPA automatically creates SQL: checks if product exists AND has enough quantity

---

## Step 7: Create Inventory Service

### 7.1 Create Service Interface

**In IntelliJ:** Right-click on `service` package → New → Java Class → Select **Interface**

Create `service/InventoryService.java`:

```java
package ca.gbc.inventoryservice.service;

public interface InventoryService {

    boolean isInStock(String skuCode, Integer quantity);

}
```

### 7.2 Create Service Implementation

**In IntelliJ:** Right-click on `service` package → New → Java Class → Select **Class**

Create `service/InventoryServiceImpl.java`:

```java
package ca.gbc.inventoryservice.service;

import ca.gbc.inventoryservice.repository.InventoryRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class InventoryServiceImpl implements InventoryService {

    private final InventoryRepository inventoryRepository;

    @Override
    public boolean isInStock(String skuCode, Integer quantity) {
        // Return the result of the check for stock availability
        return inventoryRepository.existsBySkuCodeAndQuantityGreaterThanEqual(skuCode, quantity);
    }
}
```

**What this does:**
- Interface defines the contract: `isInStock(skuCode, quantity)`
- Implementation checks if product exists AND has enough quantity
- Returns `true` if in stock with sufficient quantity, `false` otherwise

---

## Step 8: Create Inventory Controller

**In IntelliJ:** Right-click on `controller` package → New → Java Class → Select **Class**

Create `controller/InventoryController.java`:

```java
package ca.gbc.inventoryservice.controller;

import ca.gbc.inventoryservice.service.InventoryService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/inventory")
@RequiredArgsConstructor
public class InventoryController {

    private final InventoryService inventoryService;

    @GetMapping
    @ResponseStatus(HttpStatus.OK)
    public boolean isInStock(@RequestParam String skuCode, @RequestParam Integer quantity) {
        return inventoryService.isInStock(skuCode, quantity);
    }

}
```

**What this does:**
- Exposes REST endpoint: `GET /api/inventory?skuCode=SKU001&quantity=50`
- Takes two parameters: product code and quantity needed
- Returns boolean: `true` if enough stock, `false` otherwise

---

## Step 9: Create Flyway Migration Scripts

### 9.1 Understanding Flyway Migration Naming

Flyway uses a specific naming convention:

```
V<version>__<description>.sql

Examples:
V1__init.sql           (Version 1: Initial schema)
V2__add_inventory.sql  (Version 2: Add inventory data)
V3__add_indexes.sql    (Version 3: Add indexes)
```

**Rules:**
- Prefix: `V` (uppercase)
- Version: Number (1, 2, 3, etc.)
- Separator: `__` (double underscore)
- Description: Lowercase with underscores
- Extension: `.sql`

### 9.2 Create V1__init.sql (Schema Creation)

Create `src/main/resources/db/migration/V1__init.sql`:

```sql
-- ============================================
-- Flyway Migration V1: Create Inventory Table
-- ============================================

CREATE TABLE t_inventory (
    id BIGSERIAL PRIMARY KEY,           -- Auto-incrementing primary key
    sku_code VARCHAR(255) DEFAULT NULL, -- Product SKU code
    quantity INT                        -- Available quantity
);

-- Add comments for documentation
COMMENT ON TABLE t_inventory IS 'Inventory table storing product stock levels';
COMMENT ON COLUMN t_inventory.id IS 'Auto-generated unique identifier';
COMMENT ON COLUMN t_inventory.sku_code IS 'Stock Keeping Unit code (product identifier)';
COMMENT ON COLUMN t_inventory.quantity IS 'Available quantity in stock';
```

**What this does:**
- Creates `t_inventory` table
- `BIGSERIAL` = auto-incrementing Long
- Adds documentation via comments

### 9.3 Create V2__add_inventory.sql (Data Seeding)

Create `src/main/resources/db/migration/V2__add_inventory.sql`:

```sql
-- ============================================
-- Flyway Migration V2: Seed Inventory Data
-- ============================================

INSERT INTO t_inventory (quantity, sku_code)
VALUES
    (100, 'SKU001'),
    (200, 'SKU002'),
    (300, 'SKU003'),
    (400, 'SKU004'),
    (500, 'SKU005'),
    (600, 'SKU006');

-- Log the seeding
DO $$
BEGIN
  RAISE NOTICE 'Inventory data seeded successfully: 6 items added';
END $$;
```

**What this does:**
- Inserts 6 inventory items
- SKU001 through SKU006
- Quantities: 100, 200, 300, 400, 500, 600

**Why Flyway is Better:**
- ✅ Version controlled
- ✅ Idempotent (won't run twice)
- ✅ Tracks migration history
- ✅ Production standard
- ✅ Works across all environments

---

## Step 10: Create PostgreSQL Init Script

PostgreSQL doesn't auto-create databases. We need to create an initialization script.

### 10.1 Create Init Script Directory

Create the directory structure:

```bash
mkdir -p microservices-parent/docker/integrated/postgres/inventory-service/init
```

### 10.2 Create init.sql Script

Create `microservices-parent/docker/integrated/postgres/inventory-service/init/init.sql`:

```sql
-- ============================================
-- Inventory Service Database Initialization
-- ============================================

-- Create the inventory_service database
CREATE DATABASE inventory_service;

-- Create admin user (if not exists)
DO
$$
BEGIN
  IF NOT EXISTS (SELECT FROM pg_user WHERE usename = 'admin') THEN
    CREATE USER admin WITH PASSWORD 'password';
  END IF;
END
$$;

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE inventory_service TO admin;

-- Log success
DO $$
BEGIN
  RAISE NOTICE 'Database inventory_service initialized successfully';
END $$;
```

**Important Notes:**
- Database name uses underscore: `inventory_service` (not hyphen)
- This matches the database URL in application.properties from Step 3
- This script will run automatically when postgres-inventory container starts
- The script is idempotent (safe to run multiple times)

---

## Step 11: Run and Test Locally

**Note:** For local testing, we'll use the shared postgres container. When we containerize with docker-compose, we'll use a dedicated postgres-inventory container.

### 11.1 Start PostgreSQL Container

```bash
docker start postgres
```

### 11.2 Create Database (if not already created)

```bash
docker exec -it postgres psql -U admin -c "CREATE DATABASE inventory_service;"
```

If it already exists, you'll get an error message (safe to ignore).

### 11.3 Run inventory-service

In IntelliJ: Run `InventoryServiceApplication`

Or via command line:
```bash
cd microservices-parent/inventory-service
./gradlew bootRun
```

**Watch the logs** - you should see:

```
Flyway: Migrating schema "public" to version "1 - init"
Flyway: Migrating schema "public" to version "2 - add inventory"
Flyway: Successfully applied 2 migrations
```

This means Flyway:
1. Created the `t_inventory` table
2. Inserted 6 inventory items

### 11.4 Verify Flyway Migrations

Check database:
```bash
docker exec -it postgres psql -U admin -d inventory_service
```

Check Flyway migration history:
```sql
SELECT * FROM flyway_schema_history;
```

Expected output:
```
installed_rank | version | description   | type | script              | checksum    | installed_by | installed_on        | execution_time | success
---------------|---------|---------------|------|---------------------|-------------|--------------|---------------------|----------------|--------
             1 | 1       | init          | SQL  | V1__init.sql        | 1234567890  | admin        | 2024-01-01 10:00:00 |            45  | t
             2 | 2       | add inventory | SQL  | V2__add_inventory.sql| 9876543210 | admin        | 2024-01-01 10:00:01 |            23  | t
```

This table is automatically created by Flyway to track which migrations have been applied.

### 11.5 Verify Inventory Data

Query:
```sql
SELECT * FROM t_inventory;
```

Expected output:
```
 id | sku_code | quantity
----|----------|----------
  1 | SKU001   |      100
  2 | SKU002   |      200
  3 | SKU003   |      300
  4 | SKU004   |      400
  5 | SKU005   |      500
  6 | SKU006   |      600
```

Exit psql:
```sql
\q
```

### 11.6 Test with Postman

**Request 1: Check if 50 units of SKU001 are available (has 100)**
```
GET http://localhost:8083/api/inventory?skuCode=SKU001&quantity=50
```

**Expected Response:**
```
true
```

**Request 2: Check if 150 units of SKU001 are available (has only 100)**
```
GET http://localhost:8083/api/inventory?skuCode=SKU001&quantity=150
```

**Expected Response:**
```
false
```

**Request 3: Check if 300 units of SKU003 are available (has exactly 300)**
```
GET http://localhost:8083/api/inventory?skuCode=SKU003&quantity=300
```

**Expected Response:**
```
true
```

---

## Step 12: Add TestContainers

### 12.1 Create Integration Test

Open `src/test/java/ca/gbc/inventoryservice/InventoryServiceApplicationTests.java`

Replace with:

```java
package ca.gbc.inventoryservice;

import io.restassured.RestAssured;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.http.HttpStatus;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.hamcrest.Matchers.equalTo;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class InventoryServiceApplicationTests {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgresContainer =
            new PostgreSQLContainer<>("postgres:latest")
                    .withDatabaseName("inventory-service-test")
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
    void shouldReturnTrueWhenSufficientStock() {
        // Given: SKU001 has quantity=100 (seeded by Flyway V2 migration)
        String skuCode = "SKU001";
        Integer quantity = 50;

        // When: Check if 50 units are available
        // Then: Should return true (100 >= 50)
        RestAssured.given()
                .queryParam("skuCode", skuCode)
                .queryParam("quantity", quantity)
                .when()
                .get("/api/inventory")
                .then()
                .log().all()
                .statusCode(HttpStatus.OK.value())
                .body(equalTo("true"));
    }

    @Test
    void shouldReturnFalseWhenInsufficientStock() {
        // Given: SKU001 has quantity=100
        String skuCode = "SKU001";
        Integer quantity = 150;

        // When: Check if 150 units are available
        // Then: Should return false (100 < 150)
        RestAssured.given()
                .queryParam("skuCode", skuCode)
                .queryParam("quantity", quantity)
                .when()
                .get("/api/inventory")
                .then()
                .log().all()
                .statusCode(HttpStatus.OK.value())
                .body(equalTo("false"));
    }

    @Test
    void shouldReturnTrueWhenExactStock() {
        // Given: SKU003 has quantity=300
        String skuCode = "SKU003";
        Integer quantity = 300;

        // When: Check if 300 units are available
        // Then: Should return true (300 >= 300)
        RestAssured.given()
                .queryParam("skuCode", skuCode)
                .queryParam("quantity", quantity)
                .when()
                .get("/api/inventory")
                .then()
                .log().all()
                .statusCode(HttpStatus.OK.value())
                .body(equalTo("true"));
    }

    @Test
    void shouldReturnFalseWhenProductNotFound() {
        // Given: Non-existent product
        String skuCode = "NONEXISTENT";
        Integer quantity = 1;

        // When: Check if product is in stock
        // Then: Should return false (product doesn't exist)
        RestAssured.given()
                .queryParam("skuCode", skuCode)
                .queryParam("quantity", quantity)
                .when()
                .get("/api/inventory")
                .then()
                .log().all()
                .statusCode(HttpStatus.OK.value())
                .body(equalTo("false"));
    }
}
```

**What this does:**
- Spins up PostgreSQL TestContainer automatically
- Flyway runs migrations automatically in test container
- Tests against seeded data (SKU001-SKU006)
- 4 test scenarios covering all cases

### 12.2 Run Tests

```bash
cd microservices-parent/inventory-service
./gradlew test
```

All tests should pass. Look for:

```
InventoryServiceApplicationTests > shouldReturnTrueWhenSufficientStock() PASSED
InventoryServiceApplicationTests > shouldReturnFalseWhenInsufficientStock() PASSED
InventoryServiceApplicationTests > shouldReturnTrueWhenExactStock() PASSED
InventoryServiceApplicationTests > shouldReturnFalseWhenProductNotFound() PASSED

BUILD SUCCESSFUL
```

---

## Step 13: Create Dockerfile

Create `inventory-service/Dockerfile`:

```dockerfile
# ============================================
# Stage 1: Build Stage
# ============================================
FROM eclipse-temurin:22-jdk AS builder

WORKDIR /app

COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .
COPY src src

RUN chmod +x gradlew
RUN ./gradlew clean build -x test

# ============================================
# Stage 2: Runtime Stage
# ============================================
FROM eclipse-temurin:22-jre

WORKDIR /app

COPY --from=builder /app/build/libs/*.jar app.jar

RUN addgroup --system spring && adduser --system --group spring
RUN chown -R spring:spring /app

USER spring:spring

EXPOSE 8083

ENV SPRING_PROFILES_ACTIVE=docker

HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8083/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**What this does:**
- **Stage 1 (Builder):** Compiles the application with Java 22 JDK
- **Stage 2 (Runtime):** Runs the application with Java 22 JRE (smaller image)
- Creates non-root `spring` user for security
- Sets Docker profile automatically
- Health check via Actuator endpoint

---

## Step 14: Update docker-compose.yml

Open `microservices-parent/docker-compose.yml`

### 14.1 Add Dedicated PostgreSQL Container for Inventory Service

**Important:** Each microservice should have its own database instance for proper isolation.

Add the postgres-inventory service:

```yaml
  # ============================================
  # PostgreSQL for Inventory Service
  # ============================================
  postgres-inventory:
    container_name: postgres-inventory
    image: postgres:latest
    environment:
      POSTGRES_DB: inventory_service
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      PGDATA: /data/postgres
    volumes:
      - ./docker/integrated/postgres/data/postgres-inventory:/data/postgres
      - ./docker/integrated/postgres/inventory-service/init/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5434:5432"  # External port 5434 to avoid conflict with other postgres instances
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d inventory_service"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - microservices-network
    restart: unless-stopped
```

### 14.2 Add Inventory Service

Add the inventory-service:

```yaml
  # ============================================
  # Inventory Service (Spring Boot - PostgreSQL)
  # ============================================
  inventory-service:
    build:
      context: ./inventory-service
      dockerfile: Dockerfile
    image: inventory-service:latest
    container_name: inventory-service
    ports:
      - "8083:8083"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-inventory:5432/inventory_service
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: none
    depends_on:
      postgres-inventory:
        condition: service_healthy
    networks:
      - microservices-network
    restart: unless-stopped
```

**Key Points:**
- `postgres-inventory` uses **port 5434** externally (to avoid conflicts)
- Init script automatically creates `inventory_service` database
- Inventory service waits for postgres-inventory to be healthy before starting
- Database name is `inventory_service` (underscore, not hyphen)
- Separate data volume for isolation: `postgres-inventory`
- Flyway migrations run automatically on startup

---

## Step 15: Run with Docker Compose

### 15.1 Build and Start All Services

```bash
cd microservices-parent
docker-compose up --build -d
```

The `--build` flag ensures Docker rebuilds the inventory-service image with your latest code.

### 15.2 Verify All Services are Running

```bash
docker-compose ps
```

Expected output should show:
- ✅ postgres-inventory (healthy)
- ✅ inventory-service (running)
- Plus your other services (product-service, order-service, etc.)

### 15.3 Check Logs

View inventory-service logs:
```bash
docker-compose logs -f inventory-service
```

You should see:
- "Started InventoryServiceApplication"
- Flyway migration messages:
  ```
  Flyway: Migrating schema "public" to version "1 - init"
  Flyway: Migrating schema "public" to version "2 - add inventory"
  Flyway: Successfully applied 2 migrations
  ```

Press `Ctrl+C` to exit logs.

### 15.4 Verify Database

Connect to postgres-inventory container:
```bash
docker exec -it postgres-inventory psql -U admin -d inventory_service
```

Check Flyway migrations:
```sql
SELECT version, description, installed_on, success
FROM flyway_schema_history
ORDER BY installed_rank;
```

Check inventory data:
```sql
SELECT * FROM t_inventory;
```

Expected output:
```
 id | sku_code | quantity
----|----------|----------
  1 | SKU001   |      100
  2 | SKU002   |      200
  3 | SKU003   |      300
  4 | SKU004   |      400
  5 | SKU005   |      500
  6 | SKU006   |      600
```

Exit:
```sql
\q
```

### 15.5 Test API with Postman or cURL

**Test 1: Item in stock**
```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU001&quantity=50"
```
Expected: `true`

**Test 2: Insufficient stock**
```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU001&quantity=150"
```
Expected: `false`

**Test 3: Exact stock match**
```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU006&quantity=600"
```
Expected: `true`

### 15.6 Stop Services

When done testing:
```bash
docker-compose down
```

To remove volumes (clears database data):
```bash
docker-compose down -v
```

**Note:** If you remove volumes, Flyway will re-run migrations when you start again.

---

## Summary

### What You've Built

You've successfully created a production-ready inventory microservice with:

**Core Functionality:**
- ✅ New inventory-service microservice
- ✅ Dedicated PostgreSQL database instance (postgres-inventory)
- ✅ REST API to check stock availability
- ✅ **Flyway database migrations** for schema and data management
- ✅ Spring Data JPA custom query methods
- ✅ Integration tests with TestContainers
- ✅ Multistage Docker containerization

**Production Best Practices Implemented:**
- ✅ **Flyway Migrations**: Version-controlled database changes
- ✅ **Database per Service Pattern**: Each microservice has its own database instance
- ✅ **Service Isolation**: Inventory service can be deployed/scaled independently
- ✅ **Health Checks**: PostgreSQL health checks ensure proper startup order
- ✅ **Environment-based Configuration**: Separate configs for local vs Docker
- ✅ **Container Orchestration**: Docker Compose manages all services
- ✅ **Migration Tracking**: Flyway tracks which migrations have been applied

### Architecture Overview

**Service Ports:**
- product-service: 8084
- order-service: 8082
- **inventory-service: 8083** (NEW)

**Database Ports:**
- MongoDB (product-service): 27017
- PostgreSQL (order-service): 5432
- **PostgreSQL (inventory-service): 5434** (NEW)

### Flyway Benefits

**Why Flyway over CommandLineRunner?**

| Aspect | CommandLineRunner | Flyway |
|--------|------------------|--------|
| **Production Ready** | ❌ No | ✅ Yes |
| **Version Control** | ❌ No tracking | ✅ Full history |
| **Idempotent** | ⚠️ Needs custom logic | ✅ Built-in |
| **Team Collaboration** | ❌ Difficult | ✅ Easy |
| **Rollback Support** | ❌ No | ✅ Yes (paid) |
| **CI/CD Integration** | ⚠️ Complex | ✅ Simple |
| **Schema Evolution** | ❌ Hard to track | ✅ Automatic |

### Key Flyway Concepts Learned

1. **Versioned Migrations**: `V1__`, `V2__`, etc.
2. **Naming Convention**: `V<version>__<description>.sql`
3. **Migration History**: `flyway_schema_history` table
4. **Automatic Execution**: Runs on application startup
5. **Schema Management**: `ddl-auto=none` - Flyway controls everything

### Next Steps

Now that you have a production-ready inventory service, you can:

1. **Add More Migrations:**
   ```sql
   V3__add_indexes.sql      -- Add performance indexes
   V4__add_audit_columns.sql -- Add created_at, updated_at
   V5__add_location.sql     -- Add warehouse location
   ```

2. **Integrate with Order Service:**
   - Check inventory before placing orders
   - Reserve inventory during order processing
   - Release inventory if order fails

3. **Add More Endpoints:**
   - `PUT /api/inventory/{id}` - Update quantity
   - `POST /api/inventory` - Add new product
   - `GET /api/inventory/{skuCode}` - Get specific item

4. **Implement Business Logic:**
   - Inventory reservation (temporary holds)
   - Low stock alerts
   - Reorder point calculations
   - Stock movement history

### Troubleshooting

**If Flyway migrations fail:**
- Check `flyway_schema_history` table for errors
- Look at console logs for specific error messages
- Verify SQL syntax in migration files
- Ensure version numbers are sequential

**If inventory-service won't start:**
- Check postgres-inventory is healthy: `docker-compose ps`
- View logs: `docker-compose logs postgres-inventory`
- Verify init script ran: `docker exec -it postgres-inventory psql -U admin -l`

**If data isn't seeded:**
- Check Flyway logs in application startup
- Query `flyway_schema_history` to see if V2 migration ran
- Verify `t_inventory` table exists

**Database connection errors:**
- Ensure `SPRING_DATASOURCE_URL` points to `postgres-inventory:5432`
- Check username/password match in docker-compose and application-docker.properties
- Verify database name is `inventory_service` (underscore, not hyphen)

**To re-run migrations (DANGER - only for development):**
```sql
-- Connect to database
docker exec -it postgres-inventory psql -U admin -d inventory_service

-- Delete migration history (CAUTION!)
TRUNCATE flyway_schema_history;

-- Drop and recreate tables
DROP TABLE t_inventory;

-- Restart service to re-run migrations
docker-compose restart inventory-service
```

### Learning Resources

- [Flyway Documentation](https://flywaydb.org/documentation/)
- [Spring Boot Flyway Integration](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.data-initialization.migration-tool.flyway)
- [Database Migration Best Practices](https://flywaydb.org/documentation/concepts/migrations)

---

**Congratulations!** You've built a production-ready inventory microservice using industry-standard tools and practices. This approach scales from development to production seamlessly.
