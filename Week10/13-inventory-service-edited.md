# Inventory Service with Flyway Migrations - 11AM Section

## Introduction

This guide adapts the inventory-service to use **Flyway database migrations** for the 11AM section codebase. The inventory-service already exists but currently uses Hibernate's `ddl-auto=update`. This guide will convert it to use Flyway for production-ready database version control.

**What you'll update:**
- Add Flyway dependencies to existing inventory-service
- Create Flyway migration scripts
- Update configuration to use Flyway instead of Hibernate DDL
- Add inventory-service to docker-compose.yml
- Create integration tests with TestContainers

**Time:** 60-90 minutes

---

## Current State Analysis

**Existing inventory-service:**
- ✅ Model, Repository, Service, Controller already exist
- ✅ PostgreSQL configuration (localhost:5432)
- ⚠️ Uses `ddl-auto=update` (not production-ready)
- ❌ Not included in docker-compose.yml
- ❌ No Flyway migrations

**Project Details:**
- Spring Boot: 3.5.6
- Java: 21
- Network: microservices-network

---

## Step 1: Update build.gradle.kts

Open `inventory-service/build.gradle.kts`

Add Flyway dependencies to the dependencies block:

```kotlin
dependencies {
	implementation("org.springframework.boot:spring-boot-starter-actuator")
	implementation("org.springframework.boot:spring-boot-starter-data-jpa")
	implementation("org.springframework.boot:spring-boot-starter-web")

	// ADD THESE TWO LINES FOR FLYWAY
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
```

**Reload Gradle** in IntelliJ after making changes.

---

## Step 2: Update Application Configuration

### 2.1 Update application.properties

Open `inventory-service/src/main/resources/application.properties`

**CHANGE THIS LINE:**
```properties
spring.jpa.hibernate.ddl-auto=update
```

**TO:**
```properties
spring.jpa.hibernate.ddl-auto=none
```

**Full file should look like:**
```properties
spring.application.name=inventory-service

server.port=8083

spring.datasource.url=jdbc:postgresql://localhost:5432/inventory-service
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# CHANGED: Use Flyway for migrations instead of Hibernate DDL
spring.jpa.hibernate.ddl-auto=none

spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
management.endpoints.web.exposure.include=health,info
```

### 2.2 Update application-docker.properties

Open `inventory-service/src/main/resources/application-docker.properties`

**CHANGE THIS LINE:**
```properties
spring.jpa.hibernate.ddl-auto=update
```

**TO:**
```properties
spring.jpa.hibernate.ddl-auto=none
```

**Full file should look like:**
```properties
spring.application.name=inventory-service

server.port=8083

spring.datasource.url=jdbc:postgresql://postgres:5432/inventory-service
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# CHANGED: Use Flyway for migrations instead of Hibernate DDL
spring.jpa.hibernate.ddl-auto=none

spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
management.endpoints.web.exposure.include=health,info
```

---

## Step 3: Create Flyway Migration Scripts

### 3.1 Create Migration Directory

Create this directory structure:

```
inventory-service/src/main/resources/
└── db/
    └── migration/
```

**In IntelliJ:** Right-click on `src/main/resources` → New → Directory → Enter `db/migration`

### 3.2 Understanding Flyway Naming Convention

Flyway uses a specific naming pattern:

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

### 3.3 Create V1__init.sql (Schema Creation)

**In IntelliJ:** Right-click on `db/migration` → New → File → Enter `V1__init.sql`

```sql
CREATE TABLE t_inventory (
    id BIGSERIAL PRIMARY KEY,
    sku_code VARCHAR(255),
    quantity INTEGER
);
```

**What this does:**
- Creates `t_inventory` table
- `BIGSERIAL` = auto-incrementing Long (PostgreSQL equivalent of AUTO_INCREMENT)
- Matches the existing `Inventory.java` entity

### 3.4 Create V2__add_inventory.sql (Data Seeding)

**In IntelliJ:** Right-click on `db/migration` → New → File → Enter `V2__add_inventory.sql`

```sql
INSERT INTO t_inventory (sku_code, quantity)
VALUES
    ('SKU001', 100),
    ('SKU002', 200),
    ('SKU003', 300),
    ('SKU004', 400),
    ('SKU005', 500),
    ('SKU006', 600);
```

**What this does:**
- Inserts 6 sample inventory items
- SKU001 has 100 units, SKU002 has 200 units, etc.

---

## Step 4: Test Locally

### 4.1 Drop Existing Table (if running before)

If you've run inventory-service before with `ddl-auto=update`, drop the old table:

```bash
docker exec -it postgres psql -U admin -d inventory-service
```

Run:
```sql
DROP TABLE IF EXISTS t_inventory;
\q
```

### 4.2 Run inventory-service

In IntelliJ: Run `InventoryServiceApplication`

**Watch the console logs** - you should see:

```
Flyway Community Edition 9.x.x
Database: jdbc:postgresql://localhost:5432/inventory-service
Successfully validated 2 migrations
Current version of schema "public": << Empty Schema >>
Migrating schema "public" to version "1 - init"
Migrating schema "public" to version "2 - add inventory"
Successfully applied 2 migrations
```

### 4.3 Verify Data

Check the database:

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

Check Flyway metadata:
```sql
SELECT * FROM flyway_schema_history;
```

Expected:
```
installed_rank | version | description   | type | script               | success
---------------|---------|---------------|------|---------------------|--------
             1 |       1 | init          | SQL  | V1__init.sql        | t
             2 |       2 | add inventory | SQL  | V2__add_inventory.sql| t
```

Exit:
```sql
\q
```

### 4.4 Test API

```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU001&quantity=50"
```
Expected: `true` (SKU001 has 100, so 50 is available)

```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU001&quantity=150"
```
Expected: `false` (SKU001 only has 100, not enough for 150)

---

## Step 5: Update Integration Tests

Open `inventory-service/src/test/java/ca/gbc/comp3095/inventoryservice/InventoryServiceApplicationTests.java`

Replace with:

```java
package ca.gbc.comp3095.inventoryservice;

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

    static {
        postgresContainer.start();
    }

    @BeforeEach
    void setUp() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
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

    @Test
    void shouldReturnFalseWhenInsufficientStock() {
        String skuCode = "SKU001";
        Integer quantity = 150;

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

**You can delete** `TestcontainersConfiguration.java` since we're using inline configuration now.

### 5.1 Run Tests

In IntelliJ: Right-click on test class → Run Tests

Or via command line:
```bash
cd microservices-parent/inventory-service
./gradlew test
```

**Expected:** Both tests should pass. Flyway will run migrations in the TestContainer.

---

## Step 6: Create Dockerfile

Create `inventory-service/Dockerfile`:

```dockerfile
# ------------------
#  Build Stage
# ------------------
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /app

# Copy Gradle wrapper and build files
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .

# Copy source code
COPY src src

# Make gradlew executable and build
RUN chmod +x gradlew
RUN ./gradlew clean build -x test

# ------------------
#  Runtime Stage
# ------------------
FROM eclipse-temurin:21-jre

WORKDIR /app

# Copy JAR from builder stage
COPY --from=builder /app/build/libs/*.jar app.jar

# Create non-root user
RUN addgroup --system spring && adduser --system --group spring
RUN chown -R spring:spring /app

# Run as non-root user
USER spring:spring

# Expose port
EXPOSE 8083

# Set active profile
ENV SPRING_PROFILES_ACTIVE=docker

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8083/actuator/health || exit 1

# Run application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Note:** This matches the pattern used in order-service Dockerfile.

---

## Step 7: Update docker-compose.yml

Open `microservices-parent/docker-compose.yml`

### 7.1 Add inventory-service

Add this service after order-service:

```yaml
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/inventory-service
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: none
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - microservices-network
```

**Key Points:**
- Uses shared `postgres` service (not a separate postgres-inventory)
- Network: `microservices-network` (matches existing services)
- Environment variables in key-value format (matches order-service)
- Depends on postgres health check
- Port 8083 for inventory-service

### 7.2 Update PostgreSQL Init Script (Optional)

The existing `init/postgres/docker-entrypoint-initdb.d/init.sql` only creates `order-service` database.

**Option 1: Keep it as-is** (Hibernate will create the database when it connects)

**Option 2: Update init.sql to create both databases:**

```sql
-- Create order-service database
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_database WHERE datname = 'order-service') THEN
        CREATE DATABASE "order-service";
    END IF;
END $$;

-- Create inventory-service database
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_database WHERE datname = 'inventory-service') THEN
        CREATE DATABASE "inventory-service";
    END IF;
END $$;

-- Connect to order-service
\c order-service

-- Create extension for UUID support
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Connect to inventory-service
\c inventory-service

-- Log successful initialization
DO $$
BEGIN
  RAISE NOTICE 'Databases order-service and inventory-service initialized successfully';
END $$;
```

---

## Step 8: Run with Docker Compose

### 8.1 Build and Start All Services

```bash
cd microservices-parent
docker-compose up --build -d
```

### 8.2 Verify Services

```bash
docker-compose ps
```

Expected output should show:
```
NAME                  STATUS
inventory-service     running
mongodb              running
mongo-express        running
order-service        running
pgadmin              running
postgres             running
product-service      running
```

### 8.3 Check Logs

```bash
docker-compose logs -f inventory-service
```

Look for Flyway migration messages:
```
Flyway Community Edition
Successfully validated 2 migrations
Migrating schema "public" to version "1 - init"
Migrating schema "public" to version "2 - add inventory"
Successfully applied 2 migrations
```

Press `Ctrl+C` to exit.

### 8.4 Verify Database

Check the database using pgAdmin at http://localhost:8888:
- Email: admin@admin.com
- Password: admin

Or via command line:

```bash
docker exec -it postgres psql -U admin -d inventory-service
```

Query:
```sql
SELECT * FROM t_inventory;
SELECT * FROM flyway_schema_history;
\q
```

### 8.5 Test API

```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU001&quantity=50"
```
Expected: `true`

```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU006&quantity=700"
```
Expected: `false` (SKU006 only has 600)

### 8.6 Stop Services

```bash
docker-compose down
```

---

## Step 9: Understanding Flyway Benefits

### Why Flyway over Hibernate DDL?

**Hibernate DDL (`ddl-auto=update`):**
- ❌ Not production-safe (can't delete columns)
- ❌ No version tracking
- ❌ Can't rollback changes
- ❌ Team collaboration issues

**Flyway:**
- ✅ Version-controlled migrations
- ✅ Repeatable and auditable
- ✅ Works across all environments (dev, test, prod)
- ✅ Team-friendly (conflicts caught early)
- ✅ Can rollback with paid version

### Flyway Workflow

1. Developer creates `V3__add_new_column.sql`
2. Commit to Git
3. Application starts → Flyway checks `flyway_schema_history`
4. New migration detected → Executes V3
5. Records in `flyway_schema_history`
6. Future deployments skip V1, V2, V3 (already applied)

---

## Summary

You've successfully:
- ✅ Added Flyway to existing inventory-service
- ✅ Created version-controlled database migrations
- ✅ Updated configuration to use Flyway (ddl-auto=none)
- ✅ Added inventory-service to docker-compose.yml
- ✅ Tested with TestContainers
- ✅ Verified with Docker Compose

**Architecture:**
- inventory-service: port 8083
- Shared PostgreSQL: port 5432
- Network: microservices-network
- Database: inventory-service
- Flyway migrations: V1 (schema), V2 (data)

**Next Steps:**
- Create V3 migration to add new columns
- Integrate inventory checks with order-service
- Add inventory reservation logic
- Implement stock management endpoints (add/remove stock)

---

## Troubleshooting

### Issue: Flyway detects failed migration

**Error:**
```
Detected failed migration to version 1 (init)
```

**Solution:**
```bash
docker exec -it postgres psql -U admin -d inventory-service
DELETE FROM flyway_schema_history WHERE success = false;
\q
```

Then restart the service.

### Issue: Table already exists

**Error:**
```
ERROR: relation "t_inventory" already exists
```

**Solution:**
Drop the table and restart:
```bash
docker exec -it postgres psql -U admin -d inventory-service
DROP TABLE t_inventory;
\q
```

### Issue: Can't connect to postgres in Docker

**Check:**
1. Postgres container is running: `docker-compose ps`
2. Health check passed: `docker-compose logs postgres`
3. Database created: `docker exec -it postgres psql -U admin -l`

---

## Comparison with Original Code

**Before (Hibernate DDL):**
```properties
spring.jpa.hibernate.ddl-auto=update
```
- Hibernate automatically creates/updates tables
- No version tracking
- Not production-ready

**After (Flyway):**
```properties
spring.jpa.hibernate.ddl-auto=none
```
- Flyway manages all schema changes
- Version-controlled migrations in `db/migration/`
- Production-ready database versioning

**Files Added:**
- `src/main/resources/db/migration/V1__init.sql`
- `src/main/resources/db/migration/V2__add_inventory.sql`
- `Dockerfile`

**Files Modified:**
- `build.gradle.kts` (added Flyway dependencies)
- `application.properties` (changed ddl-auto to none)
- `application-docker.properties` (changed ddl-auto to none)
- `docker-compose.yml` (added inventory-service)
- `InventoryServiceApplicationTests.java` (updated tests)
