# Order Service Setup - Complete Guide

## Step 1: Update build.gradle.kts

Open `microservices-parent/order-service/build.gradle.kts`

Add these dependencies after the existing testRuntimeOnly line:

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

    // TestContainers Dependencies
    testImplementation(platform("org.testcontainers:testcontainers-bom:1.21.3"))
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("org.testcontainers:postgresql")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("io.rest-assured:rest-assured")
}
```

## Step 2: Update OrderServiceApplicationTests.java

Open `microservices-parent/order-service/src/test/java/ca/gbc/comp3095/orderservice/OrderServiceApplicationTests.java`

Replace entire file with:

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

## Step 3: Update application.properties

Open `microservices-parent/order-service/src/main/resources/application.properties`

Change PostgreSQL port from 5441 to 5432:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/order-service
```

## Step 4: Update docker-compose.yml

Open `microservices-parent/docker-compose.yml`

Replace entire file with:

```yaml
version: '3.8'

services:
  # ============================================
  # MongoDB Database (for product-service)
  # ============================================
  mongodb:
    image: mongo:latest
    container_name: mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - mongo-data:/data/db
      - ./init/mongo/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================
  # Mongo Express (MongoDB GUI)
  # ============================================
  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_SERVER: mongodb
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: password
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - microservices-network

  # ============================================
  # PostgreSQL Database (for order-service)
  # ============================================
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
      - postgres-data:/var/lib/postgresql/data
      - ./init/postgres/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d order-service"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================
  # pgAdmin (PostgreSQL GUI)
  # ============================================
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

  # ============================================
  # Product Service (Spring Boot - MongoDB)
  # ============================================
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

  # ============================================
  # Order Service (Spring Boot - PostgreSQL)
  # ============================================
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

# ============================================
# Networks
# ============================================
networks:
  microservices-network:
    driver: bridge

# ============================================
# Volumes
# ============================================
volumes:
  mongo-data:
  postgres-data:
  pgadmin-data:
```

## Step 5: Create Dockerfile for order-service

Create `microservices-parent/order-service/Dockerfile`:

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8082
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Step 6: Create PostgreSQL Init Directory

Create directory structure:

```bash
mkdir -p microservices-parent/init/postgres/docker-entrypoint-initdb.d
```

## Step 7: Configure IntelliJ Test Runner

Open IntelliJ IDEA:
1. Go to **Settings/Preferences → Build, Execution, Deployment → Build Tools → Gradle**
2. Find **"Run tests using:"** dropdown
3. Change to **"IntelliJ IDEA"**
4. Click OK

## Step 8: Reload Gradle

In IntelliJ:
- Click Gradle reload button (elephant icon with circular arrow)

Or via command line:
```bash
./gradlew --refresh-dependencies
```

## Step 9: Run Tests

**From Gradle Tool Window:**
1. Open Gradle tool window (View → Tool Windows → Gradle)
2. Navigate to: `microservices-parent > order-service > Tasks > verification > test`
3. Double-click "test"

**From Command Line:**
```bash
cd microservices-parent/order-service
./gradlew test
```

## Verification

After completing all steps:
- Tests should pass in IntelliJ
- Docker Compose should start all services correctly
- Order service accessible at http://localhost:8082
- Product service accessible at http://localhost:8084
- Mongo Express at http://localhost:8081
- pgAdmin at http://localhost:8888
