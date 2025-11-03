# Inventory Service Setup Guide

## Introduction

This guide walks you through creating a new microservice called **inventory-service** that tracks product stock levels using Flyway database migrations.

**What you'll build:**
- New Spring Boot microservice
- PostgreSQL database integration with Flyway migrations
- REST API to check inventory
- Docker containerization
- Integration tests with TestContainers

**Time:** 90-120 minutes

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
spring.application.name=inventory-service

server.port=8083

spring.datasource.driver-class-name=org.postgresql.Driver

# default is 5432 for postgres
spring.datasource.url=jdbc:postgresql://localhost:5432/inventory-service
spring.datasource.username=admin
spring.datasource.password=password
# Using Flyway
spring.jpa.hibernate.ddl-auto=none
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

### 3.2 Create application-docker.properties

Create `src/main/resources/application-docker.properties`:

```properties
spring.application.name=inventory-service

server.port=8083

spring.datasource.driver-class-name=org.postgresql.Driver

# default is 5432 for postgres
spring.datasource.url=jdbc:postgresql://postgres-inventory:5434/inventory-service
spring.datasource.username=admin
spring.datasource.password=password
# Using Flyway
spring.jpa.hibernate.ddl-auto=none
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

**Note:** The Docker URL includes port 5434 and uses `inventory-service` as database name.

---

## Step 4: Update build.gradle.kts

### 4.1 Add Flyway Dependencies

Open `inventory-service/build.gradle.kts`

Replace the entire file with:

```kotlin
plugins {
	java
	id("org.springframework.boot") version "3.4.4"
	id("io.spring.dependency-management") version "1.1.7"
}

group = "ca.gbc"
version = "0.0.1-SNAPSHOT"

java {
	toolchain {
		languageVersion = JavaLanguageVersion.of(22)
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
	testImplementation("org.springframework.boot:spring-boot-testcontainers")
	testImplementation("org.testcontainers:junit-jupiter")
	testImplementation("org.testcontainers:postgresql")
	testImplementation("io.rest-assured:rest-assured")
	testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.withType<Test> {
	useJUnitPlatform()
}
```

**Reload Gradle** after making changes.

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
@AllArgsConstructor
@NoArgsConstructor
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
- Spring Data JPA automatically creates SQL query

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
CREATE TABLE t_inventory (
      id BIGSERIAL PRIMARY KEY,  -- BIGSERIAL is auto-incrementing, and this is set as the primary key
      sku_code VARCHAR(255) DEFAULT NULL,  -- varchar for SKU code, default is NULL
      quantity INT  -- integer field for quantity
);
```

**What this does:**
- Creates `t_inventory` table
- `BIGSERIAL` = auto-incrementing Long
- Uses inline comments for documentation

### 9.3 Create V2__add_inventory.sql (Data Seeding)

Create `src/main/resources/db/migration/V2__add_inventory.sql`:

```sql
INSERT INTO t_inventory (quantity, sku_code)
VALUES
    (100, 'SKU001'),
    (200, 'SKU002'),
    (300, 'SKU003'),
    (400, 'SKU004'),
    (500, 'SKU005'),
    (600, 'SKU006');
```

**What this does:**
- Inserts 6 inventory items
- SKU001 through SKU006
- Quantities: 100, 200, 300, 400, 500, 600

---

## Step 10: Create PostgreSQL Init Script

### 10.1 Create Init Script Directory

Create the directory structure:

```bash
mkdir -p microservices-parent/docker/integrated/postgres/inventory-service/init
```

### 10.2 Create init.sql Script

Create `microservices-parent/docker/integrated/postgres/inventory-service/init/init.sql`:

```sql
-- Create the inventory_service database and user
CREATE DATABASE inventory_service;
CREATE USER admin WITH PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE inventory_service TO admin;
```

**Note:** This creates the database `inventory_service` with underscore.

---

## Step 11: Run and Test Locally

### 11.1 Start PostgreSQL Container

```bash
docker start postgres
```

### 11.2 Create Database

```bash
docker exec -it postgres psql -U admin -c "CREATE DATABASE \"inventory-service\";"
```

**Note:** Using hyphenated name for local development.

### 11.3 Run inventory-service

In IntelliJ: Run `InventoryServiceApplication`

Or via command line:
```bash
cd microservices-parent/inventory-service
./gradlew bootRun
```

**Watch the logs** - you should see Flyway migration messages.

### 11.4 Verify Data

Check database:
```bash
docker exec -it postgres psql -U admin -d inventory-service
```

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

Exit:
```sql
\q
```

### 11.5 Test with Postman

**Request 1:**
```
GET http://localhost:8083/api/inventory?skuCode=SKU001&quantity=50
```
Expected: `true`

**Request 2:**
```
GET http://localhost:8083/api/inventory?skuCode=SKU001&quantity=150
```
Expected: `false`

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
        String skuCode = "SKU001";
        Integer quantity = 50;

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
}
```

### 12.2 Run Tests

```bash
cd microservices-parent/inventory-service
./gradlew test
```

---

## Step 13: Create Dockerfile

Create `inventory-service/Dockerfile`:

```dockerfile
# ------------------
#  Build Stage
# ------------------

# Start from the Gradle 8 image with JDK 22. This image provides both Gradle and JDK 22.
# We use this stage to build the application within the container.
FROM gradle:8-jdk22 AS builder

# Copy the application files from the host machine to the image filesystem.
# We use the '--chown=gradle:gradle' flag to ensure proper file permissions for the Gradle user.
COPY --chown=gradle:gradle . /home/gradle/src

# Set the working directory inside the container to /home/gradle/src.
# All future commands will be executed from this directory.
WORKDIR /home/gradle/src

# Run the Gradle build inside the container, skipping the tests (-x test).
# This command compiles the code, resolves dependencies, and packages the application as a .jar file.
# Note: The command applies to the image only, not to the host machine.
RUN gradle build -x test

# ------------------
#  Package Stage
# ------------------

# Start from a lightweight OpenJDK 22 Alpine image. This will be our runtime image.
# Alpine images are much smaller, which helps keep the final image size down.
FROM openjdk:22-jdk

# Create a directory inside the container where the application will be stored.
# This directory is where we will place the packaged .jar file built in the previous stage.
RUN mkdir /app

# Copy the built .jar file from the build stage to the /app directory in the final image.
# We use the '--from=builder' instruction to reference the "builder" stage.
COPY --from=builder /home/gradle/src/build/libs/*.jar /app/inventory-service.jar

# Set environment variables for the application. These variables can be accessed within the application.
# These values could be replaced by Docker Compose or passed during runtime with the `docker run` command.
ENV POSTGRES_USER=admin \
    POSTGRES_USER=password

# Expose port 8084 to allow communication with the containerized application.
# EXPOSE does not actually make the port accessible to the host machine; it's documentation for the image.
EXPOSE 8082

# The ENTRYPOINT instruction defines the command to run when the container starts.
# In this case, we are telling Docker to run the Java command with the packaged JAR file.
ENTRYPOINT ["java", "-jar", "/app/inventory-service.jar"]
```

**Note:** This Dockerfile follows the pattern used in week3.2.

---

## Step 14: Update docker-compose.yml

Open `microservices-parent/docker-compose.yml`

### 14.1 Add PostgreSQL Container for Inventory Service

Add the postgres-inventory service:

```yaml
  postgres-inventory:
    container_name: postgres-inventory
    image: postgres
    environment:
      POSTGRES_DB: inventory-service
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      PGDATA: /data/postgres
    volumes:
      - ./docker/integrated/postgres/data/postgres-inventory:/data/postgres                                # mount the database
      - ./docker/integrated/postgres/inventory-service/init/init.sql:/docker-entrypoint-initdb.d/init.sql  # mount the init.sql script
    ports:
      - "5434:5432"
    networks:
      - spring
```

### 14.2 Add Inventory Service

Add the inventory-service:

```yaml
  inventory-service:
    image: inventory-service
    ports:
      - "8083:8083"
    build:
      context: ./inventory-service
      dockerfile: ./Dockerfile
    container_name: inventory-service
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-inventory/inventory-service
      - SPRING_DATASOURCE_USERNAME=admin
      - SPRING_DATASOURCE_PASSWORD=password
      - SPRING_JPA_HIBERNATE_DDL_AUTO=none
      - SPRING_PROFILES_ACTIVE=docker
    depends_on:
      - postgres-inventory
    networks:
      - spring
```

**Key Points:**
- postgres-inventory uses external port 5434
- Network name is `spring`
- Environment variables use list format with `-` prefix
- Simple depends_on without health check condition

---

## Step 15: Run with Docker Compose

### 15.1 Build and Start All Services

```bash
cd microservices-parent
docker-compose up --build -d
```

### 15.2 Verify Services

```bash
docker-compose ps
```

Expected output should show:
- postgres-inventory (running)
- inventory-service (running)

### 15.3 Check Logs

```bash
docker-compose logs -f inventory-service
```

Look for Flyway migration messages. Press `Ctrl+C` to exit.

### 15.4 Verify Database

```bash
docker exec -it postgres-inventory psql -U admin -d inventory-service
```

Check data:
```sql
SELECT * FROM t_inventory;
```

Exit:
```sql
\q
```

### 15.5 Test API

```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU001&quantity=50"
```
Expected: `true`

```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU001&quantity=150"
```
Expected: `false`

### 15.6 Stop Services

```bash
docker-compose down
```

---

## Summary

You've created:
- ✅ Inventory microservice with Spring Boot
- ✅ PostgreSQL database with dedicated container
- ✅ Flyway migrations for schema and data
- ✅ REST API for stock checking
- ✅ Integration tests with TestContainers
- ✅ Docker containerization

**Service Ports:**
- inventory-service: 8083
- postgres-inventory: 5434 (external)

**Next Steps:**
- Integrate with order-service
- Add more inventory management endpoints
- Implement stock reservation logic
