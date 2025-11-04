# Order Service with OpenFeign

## Introduction

Add OpenFeign for inter-service communication between order-service and inventory-service for comp3095_fall2025_11am.

**Time:** 60-90 minutes

---

## Table of Contents

- [Step 1: Update build.gradle.kts](#step-1-update-buildgradlekts)
- [Step 2: Create InventoryClient](#step-2-create-inventoryclient)
- [Step 3: Enable Feign Clients](#step-3-enable-feign-clients)
- [Step 4: Update OrderService](#step-4-update-orderservice)
- [Step 5: Update application.properties](#step-5-update-applicationproperties)
- [Step 6: Update application-docker.properties](#step-6-update-application-dockerproperties)
- [Step 7: Create InventoryClientStub](#step-7-create-inventoryclientstub)
- [Step 8: Create test application.properties](#step-8-create-test-applicationproperties)
- [Step 9: Update OrderServiceApplicationTests](#step-9-update-orderserviceapplicationtests)
- [Step 10: Test Locally](#step-10-test-locally)
- [Step 11: Test with Docker Compose](#step-11-test-with-docker-compose)
- [Summary](#summary)

---

## Step 1: Update build.gradle.kts

Open `microservices-parent/order-service/build.gradle.kts`

Add Spring Cloud BOM and OpenFeign dependencies:

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.6"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "ca.gbc.comp3095"
version = "0.0.1-SNAPSHOT"
description = "order-service"

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

dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2025.0.0")
    }
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("org.springframework.cloud:spring-cloud-starter-contract-stub-runner")
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

Reload Gradle.

---

## Step 2: Create InventoryClient

Create directory: `microservices-parent/order-service/src/main/java/ca/gbc/comp3095/orderservice/client`

```
microservices-parent
└── order-service
    └── src
        └── main
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── orderservice
                                └── client
                                    └── InventoryClient.java
```

Create `InventoryClient.java` (interface):

```java
package ca.gbc.comp3095.orderservice.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(value = "inventory", url = "${inventory.service.url}")
public interface InventoryClient {

    @RequestMapping(method = RequestMethod.GET, value = "/api/inventory")
    boolean isInStock(@RequestParam String skuCode, @RequestParam Integer quantity);
}
```

---

## Step 3: Enable Feign Clients

```
microservices-parent
└── order-service
    └── src
        └── main
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── orderservice
                                └── OrderServiceApplication.java
```

Open `microservices-parent/order-service/src/main/java/ca/gbc/comp3095/orderservice/OrderServiceApplication.java`

Replace with:

```java
package ca.gbc.comp3095.orderservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }

}
```

---

## Step 4: Update OrderService

```
microservices-parent
└── order-service
    └── src
        └── main
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── orderservice
                                └── service
                                    └── OrderService.java
```

Open `microservices-parent/order-service/src/main/java/ca/gbc/comp3095/orderservice/service/OrderService.java`

Replace with:

```java
package ca.gbc.comp3095.orderservice.service;

import ca.gbc.comp3095.orderservice.client.InventoryClient;
import ca.gbc.comp3095.orderservice.dto.OrderLineItemDto;
import ca.gbc.comp3095.orderservice.dto.OrderRequest;
import ca.gbc.comp3095.orderservice.model.Order;
import ca.gbc.comp3095.orderservice.model.OrderLineItem;
import ca.gbc.comp3095.orderservice.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;

    public void placeOrder(OrderRequest orderRequest) {
        // Check inventory for each item
        boolean allInStock = orderRequest.orderLineItemDtoList()
                .stream()
                .allMatch(item -> inventoryClient.isInStock(item.skuCode(), item.quantity()));

        if (!allInStock) {
            throw new RuntimeException("One or more products are not in stock");
        }

        // Generate unique order number
        String orderNumber = UUID.randomUUID().toString();

        // Convert DTOs to Entities
        List<OrderLineItem> orderLineItems = orderRequest.orderLineItemDtoList()
                .stream()
                .map(this::mapToOrderLineItem)
                .toList();

        // Create Order entity
        Order order = Order.builder()
                .orderNumber(orderNumber)
                .orderLineItems(orderLineItems)
                .build();

        // Save order (cascade will save line items)
        orderRepository.save(order);

        log.info("Order {} placed successfully with {} items",
                orderNumber, orderLineItems.size());
    }

    private OrderLineItem mapToOrderLineItem(OrderLineItemDto dto) {
        return OrderLineItem.builder()
                .skuCode(dto.skuCode())
                .price(dto.price())
                .quantity(dto.quantity())
                .build();
    }
}
```

---

## Step 5: Update application.properties

```
microservices-parent
└── order-service
    └── src
        └── main
            └── resources
                └── application.properties
```

Open `microservices-parent/order-service/src/main/resources/application.properties`

Add at the end:

```properties
inventory.service.url=http://localhost:8083
```

---

## Step 6: Update application-docker.properties

```
microservices-parent
└── order-service
    └── src
        └── main
            └── resources
                └── application-docker.properties
```

Open `microservices-parent/order-service/src/main/resources/application-docker.properties`

Add at the end:

```properties
inventory.service.url=http://inventory-service:8083
```

---

## Step 7: Create InventoryClientStub

Create directory: `microservices-parent/order-service/src/test/java/ca/gbc/comp3095/orderservice/stubs`

```
microservices-parent
└── order-service
    └── src
        └── test
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── orderservice
                                └── stubs
                                    └── InventoryClientStub.java
```

Create `InventoryClientStub.java` (class):

```java
package ca.gbc.comp3095.orderservice.stubs;

import static com.github.tomakehurst.wiremock.client.WireMock.*;

public class InventoryClientStub {

    public static void stubInventoryCall(String skuCode, Integer quantity){
        stubFor(get(urlEqualTo("/api/inventory?skuCode=" + skuCode + "&quantity=" + quantity))
                .willReturn(aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("true")));
    }
}
```

---

## Step 8: Create test application.properties

```
microservices-parent
└── order-service
    └── src
        └── test
            └── resources
                └── application.properties
```

Create file: `microservices-parent/order-service/src/test/resources/application.properties`

```properties
inventory.service.url=http://localhost:${wiremock.server.port}
```

---

## Step 9: Update OrderServiceApplicationTests

```
microservices-parent
└── order-service
    └── src
        └── test
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── orderservice
                                └── OrderServiceApplicationTests.java
```

Open `microservices-parent/order-service/src/test/java/ca/gbc/comp3095/orderservice/OrderServiceApplicationTests.java`

Replace with:

```java
package ca.gbc.comp3095.orderservice;

import ca.gbc.comp3095.orderservice.stubs.InventoryClientStub;
import io.restassured.RestAssured;
import org.hamcrest.Matchers;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.cloud.contract.wiremock.AutoConfigureWireMock;
import org.testcontainers.containers.PostgreSQLContainer;

import static org.hamcrest.MatcherAssert.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWireMock(port = 0)
class OrderServiceApplicationTests {

    @ServiceConnection
    static PostgreSQLContainer<?> postgreSQLContainer = new PostgreSQLContainer<>("postgres:latest");

    @LocalServerPort
    private Integer port;

    static {
        postgreSQLContainer.start();
    }

    @BeforeEach
    void setup() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
    }

    @Test
    void placeOrderTest() {
        String orderJson = """
                {
                  "orderLineItemDtoList": [
                    {
                      "skuCode": "SKU001",
                      "price": 100.00,
                      "quantity": 2
                    }
                  ]
                }
                """;

        InventoryClientStub.stubInventoryCall("SKU001", 2);

        var responseBodyString = RestAssured
                .given()
                .contentType("application/json")
                .body(orderJson)
                .when()
                .post("/api/order")
                .then()
                .log().all()
                .statusCode(201)
                .extract()
                .body().asString();

        assertThat(responseBodyString, Matchers.is("Order Placed Successfully"));
    }
}
```

---

## Step 10: Test Locally

### Start postgres container

```bash
docker run -d --name postgres -p 5432:5432 \
  -e POSTGRES_DB=order-service \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  postgres:latest
```

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

### Run order-service

In IntelliJ: Run `OrderServiceApplication`

### Test with curl

```bash
curl -X POST http://localhost:8082/api/order \
  -H "Content-Type: application/json" \
  -d '{
    "orderLineItemDtoList": [
      {
        "skuCode": "SKU001",
        "price": 100.00,
        "quantity": 2
      }
    ]
  }'
```

### Run tests

```bash
cd microservices-parent/order-service
./gradlew test
```

---

## Step 11: Test with Docker Compose

```bash
cd microservices-parent
docker-compose up --build -d
```

Check logs:

```bash
docker-compose logs -f order-service
```

Test API:

```bash
curl -X POST http://localhost:8082/api/order \
  -H "Content-Type: application/json" \
  -d '{
    "orderLineItemDtoList": [
      {
        "skuCode": "SKU001",
        "price": 100.00,
        "quantity": 2
      }
    ]
  }'
```

Stop services:

```bash
docker-compose down
```

---

## Summary

- Added Spring Cloud OpenFeign for declarative REST clients
- Created InventoryClient interface
- Updated OrderService to check inventory before placing orders
- Added WireMock for testing without running inventory-service
- Configured URLs for local and Docker environments
