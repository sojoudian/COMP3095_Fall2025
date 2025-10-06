# Setup Order Service Module

## Overview

In this section, you will create the `order-service` module using Spring Initializr. This module will be added to your existing `microservices-parent` project alongside the `product-service`.

---

## Step 1: Create order-service Module

### Using Spring Initializr in IntelliJ

1. **Right-click** on `microservices-parent` project in Project Explorer
2. Select **New → Module**
3. Choose **Spring Initializr** from the generators list
4. Click **Next**

### Project Configuration

Configure your new module with the following settings:

| Setting | Value | Notes |
|---------|-------|-------|
| **Name** | `order-service` | Module name |
| **Group** | `ca.gbc.comp3095` | Same as product-service |
| **Artifact** | `order-service` | Artifact ID |
| **Package name** | `ca.gbc.comp3095.orderservice` | Base package |
| **Type** | Gradle - Kotlin | Build system |
| **Language** | Java | Programming language |
| **Java** | 21 | Java version |
| **Packaging** | Jar | Packaging type |
| **Spring Boot** | 3.5.6 | Framework version |

**Screenshot Reference:** Similar to product-service creation in Week 2

### Select Dependencies

On the dependencies screen, add the following dependencies:

#### **1. Developer Tools**
- ✅ **Lombok** - Reduce boilerplate code
- ✅ **Spring Boot DevTools** - Hot reload and LiveReload

#### **2. Web**
- ✅ **Spring Web** - Build web applications including RESTful services

#### **3. SQL**
- ✅ **Spring Data JPA** - Persist data in SQL stores with Java Persistence API
- ✅ **PostgreSQL Driver** - PostgreSQL JDBC driver

#### **4. OPS**
- ✅ **Spring Boot Actuator** - Production-ready features

#### **5. SQL (Database Migration)**
- ✅ **Flyway Migration** - Version control for database schema (optional but recommended)

**Complete Dependency List:**
```
Developer Tools:
  - Lombok
  - Spring Boot DevTools

Web:
  - Spring Web

SQL:
  - Spring Data JPA
  - PostgreSQL Driver
  - Flyway Migration

Ops:
  - Spring Boot Actuator
```

### Complete Module Creation

1. Click **Next** after selecting dependencies
2. Review the configuration
3. Click **Finish**
4. Wait for IntelliJ to generate the module and sync Gradle

---

## Step 2: Verify Module Structure

After creation, your project structure should look like:

```
microservices-parent/
├── product-service/              (Existing)
│   ├── src/
│   ├── build.gradle.kts
│   └── Dockerfile
├── order-service/                (NEW)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/
│   │   │   │   └── ca/gbc/comp3095/orderservice/
│   │   │   │       └── OrderServiceApplication.java
│   │   │   └── resources/
│   │   │       └── application.properties
│   │   └── test/
│   │       └── java/
│   │           └── ca/gbc/comp3095/orderservice/
│   │               └── OrderServiceApplicationTests.java
│   ├── gradle/
│   ├── build.gradle.kts
│   ├── gradlew
│   ├── gradlew.bat
│   └── settings.gradle.kts
├── build.gradle.kts
└── settings.gradle.kts
```

---

## Step 3: Review Generated Files

### OrderServiceApplication.java

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/OrderServiceApplication.java`

**Generated Code:**
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

**Explanation:**
- `@SpringBootApplication` - Combines `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`
- `main()` method - Entry point for the application
- `SpringApplication.run()` - Bootstraps Spring Boot application

### build.gradle.kts

**Location:** `order-service/build.gradle.kts`

**Generated Content:**
```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.6"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "ca.gbc.comp3095"
version = "0.0.1-SNAPSHOT"

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
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

**Key Dependencies:**
- `spring-boot-starter-data-jpa` - JPA support
- `spring-boot-starter-web` - REST API support
- `postgresql` - PostgreSQL driver
- `flyway-core` - Database migrations
- `lombok` - Boilerplate reduction
- `spring-boot-devtools` - Hot reload

### application.properties

**Location:** `order-service/src/main/resources/application.properties`

**Initial Content:**
```properties
# Application will be configured in Step 5
```

**Note:** Initially empty or with minimal configuration. We'll add database configuration later.

---

## Step 4: Create Package Structure

Create the following package structure within `ca.gbc.comp3095.orderservice`:

### 4.1 Create controller Package

1. **Right-click** on `ca.gbc.comp3095.orderservice`
2. Select **New → Package**
3. Enter name: `controller`
4. Press **Enter**

### 4.2 Create service Package

1. **Right-click** on `ca.gbc.comp3095.orderservice`
2. Select **New → Package**
3. Enter name: `service`
4. Press **Enter**

### 4.3 Create dto Package

1. **Right-click** on `ca.gbc.comp3095.orderservice`
2. Select **New → Package**
3. Enter name: `dto`
4. Press **Enter**

### 4.4 Create model Package

1. **Right-click** on `ca.gbc.comp3095.orderservice`
2. Select **New → Package**
3. Enter name: `model`
4. Press **Enter**

### 4.5 Create repository Package

1. **Right-click** on `ca.gbc.comp3095.orderservice`
2. Select **New → Package**
3. Enter name: `repository`
4. Press **Enter**

### Final Package Structure

```
ca.gbc.comp3095.orderservice/
├── controller/         (REST endpoints)
├── service/           (Business logic)
├── dto/               (Data Transfer Objects)
├── model/             (JPA Entities)
├── repository/        (Data access layer)
└── OrderServiceApplication.java
```

**Verification:**
- All packages should be visible in Project Explorer
- Each package should be empty initially
- Package icons should appear (folder with package symbol)

---

## Step 5: Update Parent Project Configuration

### 5.1 Update settings.gradle.kts

**Location:** `microservices-parent/settings.gradle.kts`

**Current Content:**
```kotlin
rootProject.name = "microservices-parent"
include("product-service")
```

**Updated Content:**
```kotlin
rootProject.name = "microservices-parent"
include("product-service")
include("order-service")  // ADD THIS LINE
```

**Explanation:**
- Registers `order-service` as a module in the parent project
- Enables multi-module Gradle build

### 5.2 Sync Gradle Project

1. Click the **Gradle** notification that appears
2. Or click **Gradle** tab (right side of IntelliJ)
3. Click **Reload All Gradle Projects** button (circular arrows icon)
4. Wait for sync to complete

**Expected Output:**
```
BUILD SUCCESSFUL in 5s
```

---

## Step 6: Verify Build

### 6.1 Build the Project

**Option A: Using Gradle Command**
```bash
cd /path/to/your/microservices-parent
./gradlew :order-service:build
```

**Option B: Using IntelliJ**
1. Open **Gradle** tab (right side)
2. Expand **order-service → Tasks → build**
3. Double-click **build**

**Expected Output:**
```
> Task :order-service:compileJava
> Task :order-service:processResources
> Task :order-service:classes
> Task :order-service:bootJar
> Task :order-service:jar
> Task :order-service:assemble
> Task :order-service:compileTestJava
> Task :order-service:processTestResources
> Task :order-service:testClasses
> Task :order-service:test
> Task :order-service:check
> Task :order-service:build

BUILD SUCCESSFUL in 12s
```

### 6.2 Verify JAR File Created

**Check build output:**
```bash
ls -la order-service/build/libs/
```

**Expected:**
```
order-service-0.0.1-SNAPSHOT.jar
order-service-0.0.1-SNAPSHOT-plain.jar
```

---

## Step 7: Test Run (Optional)

**Note:** This will fail because we haven't configured the database yet. This is expected.

### 7.1 Try Running the Application

**Run Command:**
```bash
cd order-service
./gradlew bootRun
```

**Expected Error:**
```
***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.

Reason: Failed to determine a suitable driver class
```

**Why This Happens:**
- We added `spring-boot-starter-data-jpa` dependency
- Spring Boot expects database configuration
- No database configuration provided yet
- This is normal and expected

**Solution:**
- We'll configure the database in Section 5 (Database Configuration)

---

## Step 8: Verify Dependencies

### 8.1 Check Lombok is Working

**Create Test Class:**

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/TestLombok.java`

```java
package ca.gbc.comp3095.orderservice;

import lombok.Data;

@Data
public class TestLombok {
    private String name;
    private Integer age;
}
```

**Verify:**
1. No red underlines on `@Data`
2. No errors in the class
3. Lombok is generating getters/setters (you won't see them, but they exist)

**Delete Test Class:**
- This was just for verification
- Delete `TestLombok.java`

### 8.2 Verify Spring Boot Actuator

**Check Actuator Dependency:**

Actuator will be available after we configure the database and run the application. We'll verify it later.

---

## Comparison: product-service vs order-service

### Dependency Comparison

| Dependency | product-service | order-service |
|------------|----------------|---------------|
| **Spring Web** | ✅ | ✅ |
| **Lombok** | ✅ | ✅ |
| **Spring Actuator** | ✅ | ✅ |
| **DevTools** | ✅ | ✅ |
| **Spring Data MongoDB** | ✅ | ❌ |
| **MongoDB Driver** | ✅ | ❌ |
| **Spring Data JPA** | ❌ | ✅ |
| **PostgreSQL Driver** | ❌ | ✅ |
| **Flyway Migration** | ❌ | ✅ |

### Key Differences

**product-service:**
- Uses MongoDB (NoSQL)
- Spring Data MongoDB for data access
- No schema management needed
- Document-based data model

**order-service:**
- Uses PostgreSQL (SQL)
- Spring Data JPA for data access
- Schema management with Flyway (optional)
- Relational data model with entity relationships

---

## Understanding Dependencies

### Spring Data JPA

**Purpose:**
- Provides JPA (Java Persistence API) support
- Object-Relational Mapping (ORM)
- Simplifies database operations

**Key Features:**
- Entity management
- Repository pattern
- Query derivation from method names
- Transaction management
- JPQL and native SQL support

**Example:**
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    // No implementation needed - Spring provides CRUD operations
}
```

### PostgreSQL Driver

**Purpose:**
- JDBC driver for PostgreSQL database
- Enables Java applications to connect to PostgreSQL
- Handles database communication

**Configuration:**
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/order-service
spring.datasource.driver-class-name=org.postgresql.Driver
```

### Flyway Migration

**Purpose:**
- Database version control
- Schema evolution tracking
- Repeatable migrations

**How It Works:**
1. Create SQL migration files (e.g., `V1__create_orders_table.sql`)
2. Flyway tracks which migrations have been applied
3. Automatically applies new migrations on startup

**Example Migration:**
```sql
-- V1__create_orders_table.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(255) NOT NULL
);
```

**Note:** We'll use JPA's `spring.jpa.hibernate.ddl-auto` for simplicity in this lab. Flyway is optional.

---

## Troubleshooting

### Issue 1: Module Not Showing in Project Explorer

**Problem:** After creating module, it doesn't appear in IntelliJ

**Solution:**
1. **File → Invalidate Caches → Invalidate and Restart**
2. Wait for IntelliJ to restart
3. Reload Gradle project

### Issue 2: Lombok Not Working

**Problem:** `@Data` annotation shows red underline

**Solution:**
1. **Settings → Plugins**
2. Verify **Lombok** plugin is installed and enabled
3. **Settings → Build, Execution, Deployment → Compiler → Annotation Processors**
4. Check **Enable annotation processing**
5. Restart IntelliJ

### Issue 3: Gradle Sync Fails

**Problem:** Gradle sync fails with dependency resolution error

**Solution:**
```bash
# Clean Gradle cache
cd order-service
./gradlew clean --refresh-dependencies

# Or delete cache and retry
rm -rf ~/.gradle/caches/
./gradlew build
```

### Issue 4: Java 21 Not Found

**Problem:** `Cannot resolve Java 21`

**Solution:**
1. **File → Project Structure**
2. **Project Settings → Project**
3. **SDK:** Select Java 21 or download if not available
4. **Language level:** 21 - Patterns in switch
5. Click **OK**

### Issue 5: Port Already in Use (for later)

**Problem:** When running, `Port 8080 is already in use`

**Solution:**
- We'll configure `order-service` to use port 8082 in application.properties
- This avoids conflict with product-service (8084) and other services

---

## What's Next?

You have successfully:
- ✅ Created the order-service module
- ✅ Configured dependencies
- ✅ Setup package structure
- ✅ Verified Gradle build

**Next Steps:**
1. Create JPA entities (Order, OrderLineItem)
2. Define entity relationships
3. Implement DTOs
4. Create service and repository layers
5. Build REST controller

Continue to [03-create-entities.md](03-create-entities.md) to start implementing the data model.

---

## Summary

### What You Created:
- ✅ order-service module in microservices-parent project
- ✅ Package structure: controller, service, dto, model, repository
- ✅ Gradle build configuration with all dependencies
- ✅ OrderServiceApplication main class

### Dependencies Added:
- ✅ Spring Data JPA (for relational data access)
- ✅ PostgreSQL Driver (database connectivity)
- ✅ Spring Web (REST API)
- ✅ Lombok (reduce boilerplate)
- ✅ Spring Actuator (production features)
- ✅ DevTools (development productivity)
- ✅ Flyway (database migrations - optional)

### Verified:
- ✅ Gradle builds successfully
- ✅ Lombok is working
- ✅ Package structure is correct
- ✅ Module is registered in parent project

### Not Yet Done:
- ⏳ Database configuration
- ⏳ Entity creation
- ⏳ Service implementation
- ⏳ Controller implementation
- ⏳ Testing

---

**Continue to:** [03-create-entities.md](03-create-entities.md)
