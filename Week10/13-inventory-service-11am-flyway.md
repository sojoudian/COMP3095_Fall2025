# Inventory Service with Flyway Migrations

## Introduction

Add Flyway database migrations to inventory-service for comp3095_fall2025_11am.

**Time:** 60-90 minutes

---

## Step 1: Update build.gradle.kts

Open `inventory-service/build.gradle.kts`

Add Flyway dependencies:

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
	testImplementation("org.springframework.boot:spring-boot-testcontainers")
	testImplementation("org.testcontainers:junit-jupiter")
	testImplementation("org.testcontainers:postgresql")
	testImplementation("io.rest-assured:rest-assured")
	testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

Reload Gradle.

---

## Step 2: Update application.properties

Open `inventory-service/src/main/resources/application.properties`

Replace entire file:

```properties
spring.application.name=inventory-service

server.port=8083

spring.datasource.url=jdbc:postgresql://localhost:5433/inventory-service
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
management.endpoints.web.exposure.include=health,info
```

---

## Step 3: Update application-docker.properties

Open `inventory-service/src/main/resources/application-docker.properties`

Replace entire file:

```properties
spring.application.name=inventory-service

server.port=8083

spring.datasource.url=jdbc:postgresql://postgres-inventory:5433/inventory-service
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
management.endpoints.web.exposure.include=health,info
```

---

## Step 4: Create Flyway Migration Scripts

Create directory: `inventory-service/src/main/resources/db/migration`

### Create V1__init.sql

File: `src/main/resources/db/migration/V1__init.sql`

```sql
CREATE TABLE t_inventory (
    id BIGSERIAL PRIMARY KEY,
    sku_code VARCHAR(255),
    quantity INTEGER
);
```

### Create V2__add_inventory.sql

File: `src/main/resources/db/migration/V2__add_inventory.sql`

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

---

## Step 5: Test Locally

### Start postgres-inventory container

```bash
docker run -d --name postgres-inventory -p 5433:5432 \
  -e POSTGRES_DB=inventory-service \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  postgres:latest
```

### Run inventory-service

In IntelliJ: Run `InventoryServiceApplication`

### Verify data

```bash
docker exec -it postgres-inventory psql -U admin -d inventory-service
```

```sql
SELECT * FROM t_inventory;
SELECT * FROM flyway_schema_history;
\q
```

### Test API

```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU001&quantity=50"
```

---

## Step 6: Update Integration Tests

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

Delete `TestcontainersConfiguration.java`.

Run tests:

```bash
cd microservices-parent/inventory-service
./gradlew test
```

---

## Step 7: Create Dockerfile

Create `inventory-service/Dockerfile`:

```dockerfile
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /app

COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts .
COPY settings.gradle.kts .
COPY src src

RUN chmod +x gradlew
RUN ./gradlew clean build -x test

FROM eclipse-temurin:21-jre

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

---

## Step 8: Update docker-compose.yml

Open `microservices-parent/docker-compose.yml`

Add postgres-inventory service:

```yaml
  postgres-inventory:
    image: postgres:latest
    container_name: postgres-inventory
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: inventory-service
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-inventory-data:/var/lib/postgresql/data
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d inventory-service"]
      interval: 10s
      timeout: 5s
      retries: 5
```

Add inventory-service:

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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-inventory:5433/inventory-service
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: none
    depends_on:
      postgres-inventory:
        condition: service_healthy
    networks:
      - microservices-network
```

Add volume to volumes section:

```yaml
volumes:
  mongo-data:
    driver: local
  postgres-data:
    driver: local
  postgres-inventory-data:
    driver: local
  pgadmin-data:
    driver: local
```

---

## Step 9: Run with Docker Compose

```bash
cd microservices-parent
docker-compose up --build -d
```

Check services:

```bash
docker-compose ps
```

Check logs:

```bash
docker-compose logs -f inventory-service
```

Verify database:

```bash
docker exec -it postgres-inventory psql -U admin -d inventory-service
```

```sql
SELECT * FROM t_inventory;
SELECT * FROM flyway_schema_history;
\q
```

Test API:

```bash
curl "http://localhost:8083/api/inventory?skuCode=SKU001&quantity=50"
```

Stop services:

```bash
docker-compose down
```

---

## Summary

- Local: postgres-inventory on port 5433
- Docker: postgres-inventory service
- Flyway migrations: V1 (schema), V2 (data)
