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

```bash
  1. shouldReturnTrueWhenItemIsInStock (line 205) - Tests SKU001 with quantity 100, expects true
  2. shouldReturnFalseWhenItemDoesNotExist (line 218) - Tests NON_EXISTENT_SKU, expects false
```

Replace with:

```java
package ca.gbc.comp3095.inventoryservice;

import io.restassured.RestAssured;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.jdbc.core.JdbcTemplate;
import org.testcontainers.containers.PostgreSQLContainer;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.is;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class InventoryServiceApplicationTests {

    @ServiceConnection
    static PostgreSQLContainer<?> postgreSQLContainer = new PostgreSQLContainer<>("postgres:latest");

    @LocalServerPort
    private Integer port;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @BeforeEach
    void setup() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;

        jdbcTemplate.execute("DELETE FROM t_inventory;");
        jdbcTemplate.execute("INSERT INTO t_inventory (sku_code, quantity) VALUES ('SKU001', 200);");
        jdbcTemplate.execute("INSERT INTO t_inventory (sku_code, quantity) VALUES ('SKU002', 50);");
    }

    static {
        postgreSQLContainer.start();
    }

    @Test
    void shouldReturnTrueWhenItemIsInStock() {
        given()
                .queryParam("skuCode", "SKU001")
                .queryParam("quantity", 100)
                .when()
                .get("/api/inventory")
                .then()
                .log().all()
                .statusCode(200)
                .body(is("true"));
    }

    @Test
    void shouldReturnFalseWhenItemDoesNotExist() {
        given()
                .queryParam("skuCode", "NON_EXISTENT_SKU")
                .queryParam("quantity", 100)
                .when()
                .get("/api/inventory")
                .then()
                .log().all()
                .statusCode(200)
                .body(is("false"));
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
FROM gradle:8-jdk21 AS builder

COPY --chown=gradle:gradle . /home/gradle/src

WORKDIR /home/gradle/src

RUN gradle build -x test

FROM eclipse-temurin:21-jdk

RUN mkdir /app

COPY --from=builder /home/gradle/src/build/libs/*.jar /app/inventory-service.jar

ENV POSTGRES_USER=admin \
    POSTGRES_PASSWORD=password

EXPOSE 8083

ENTRYPOINT ["java", "-jar", "/app/inventory-service.jar"]
```

---

## Step 8: Update docker-compose.yml

Replace `microservices-parent/docker-compose.yml` with:

```yaml
version: '3.8'

services:
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
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
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
