# Week 14 - Kafka Event-Driven Architecture & Schema Registry

---

## Important Notes (Read Before Starting)

1. **Open the correct folder in IntelliJ:** Always open `microservices-parent` folder, NOT the parent folder.

2. **Delete settings.gradle.kts from submodules:** When creating modules via Spring Initializr, IntelliJ may create a `settings.gradle.kts` inside the new module. **Delete it.** Only `microservices-parent/settings.gradle.kts` should exist.

3. **If Gradle elephant icon is missing:** Right-click on `settings.gradle.kts` in the project tree → **Link Gradle Project**

4. **If you get "Project with path ':shared-schema' could not be found":**
   - Check that only ONE `settings.gradle.kts` exists (in microservices-parent)
   - Delete any `settings.gradle.kts` files inside submodules
   - Delete `.idea` folder and re-import the project

---

## Step 0: Delete settings.gradle.kts from Existing Submodules

**IMPORTANT:** The cloned project may have `settings.gradle.kts` files inside existing submodules. These MUST be deleted first.

Check and delete these files if they exist:
- `api-gateway/settings.gradle.kts`
- `inventory-service/settings.gradle.kts`
- `order-service/settings.gradle.kts`
- `product-service/settings.gradle.kts`

**To delete:**
1. In IntelliJ project tree, navigate to each submodule
2. If `settings.gradle.kts` exists, right-click → **Delete**
3. Confirm deletion

**Only `microservices-parent/settings.gradle.kts` should exist.**

**After deleting, reset IntelliJ's Gradle configuration:**
1. Close IntelliJ
2. Delete the `.idea` folder in `microservices-parent`
3. Reopen IntelliJ and open the `microservices-parent` folder
4. Link Gradle Project:
   - Right-click on `settings.gradle.kts` → **Link Gradle Project**
5. Sync Gradle (click the elephant icon → refresh)

---

## Step 1: Create shared-schema Module (Non-Spring Boot)

### 1.1 Create Module in IntelliJ

1. **Right-click** on `microservices-parent`
2. Select **New → Module**
3. Choose **New Module** (NOT Spring Initializr)
4. Select **Gradle** with **Kotlin DSL**
5. Set **Name** to `shared-schema`
6. Click **Create**

### 1.2 Delete settings.gradle.kts from shared-schema (if exists)

If IntelliJ created a `settings.gradle.kts` inside `shared-schema/`, delete it:

1. Check if `shared-schema/settings.gradle.kts` exists
2. If yes, right-click → **Delete**

### 1.3 Update shared-schema/build.gradle.kts

```kotlin
plugins {
    id("java-library")
    id("com.github.davidmc24.gradle.plugin.avro") version "1.9.1"
}

group = "ca.gbc.comp3095"
version = "1.0.0"

repositories {
    maven { url = uri("https://packages.confluent.io/maven/") }
    mavenCentral()
}

dependencies {
    implementation("org.apache.avro:avro:1.12.0")
}

avro {
    isCreateSetters.set(true)
    fieldVisibility.set("PRIVATE")
    isEnableDecimalLogicalType.set(true)
}

sourceSets {
    main {
        java.srcDir(layout.buildDirectory.dir("generated-main-avro-java"))
    }
}

tasks.named<com.github.davidmc24.gradle.plugin.avro.GenerateAvroJavaTask>("generateAvroJava") {
    source(fileTree("src/main/avro"))
    setOutputDir(layout.buildDirectory.dir("generated-main-avro-java").get().asFile)
}
```

### 1.4 Create Directory Structure

```
shared-schema/
├── build.gradle.kts
└── src/
    └── main/
        └── avro/
            └── order-placed.avsc
```

To create the `avro` directory:
1. Right-click on `shared-schema/src/main`
2. Select **New → Directory**
3. Name: `avro`
4. Click **OK**

### 1.5 Create shared-schema/src/main/avro/order-placed.avsc

```json
{
  "type": "record",
  "name": "OrderPlacedEvent",
  "namespace": "ca.gbc.comp3095.orderservice.event",
  "fields": [
    { "name": "orderNumber", "type": "string" },
    { "name": "email", "type": "string" },
    { "name": "firstName", "type": "string" },
    { "name": "lastName", "type": "string" }
  ]
}
```

### 1.6 Update microservices-parent/settings.gradle.kts

Add `pluginManagement` block and `shared-schema` to settings:

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

rootProject.name = "microservices-parent"
include("product-service")
include("order-service")
include("inventory-service")
include("api-gateway")
include("shared-schema")
```

### 1.7 Sync Gradle

1. Click the **Gradle elephant icon** in the right sidebar
2. Click the **Refresh** button at the top
3. Wait for sync to complete

---

## Step 2: Create notification-service Module

### 2.1 Using Spring Initializr in IntelliJ

1. **Right-click** on `microservices-parent`
2. Select **New → Module**
3. Choose **Spring Initializr**
4. Click **Next**

### 2.2 Project Configuration

| Setting | Value |
|---------|-------|
| **Name** | `notification-service` |
| **Group** | `ca.gbc.comp3095` |
| **Artifact** | `notification-service` |
| **Package name** | `ca.gbc.comp3095.notificationservice` |
| **Type** | Gradle - Kotlin |
| **Language** | Java |
| **Java** | 21 |
| **Packaging** | Jar |
| **Spring Boot** | 4.0.0 |

### 2.3 Select Dependencies

#### **Developer Tools**
- ✅ **Lombok**

#### **Web**
- ✅ **Spring Web**

#### **Ops**
- ✅ **Spring Boot Actuator**

#### **Messaging**
- ✅ **Spring for Apache Kafka**

#### **I/O**
- ✅ **Java Mail Sender**

### 2.4 Complete Creation

1. Click **Next**
2. Click **Finish**
3. Wait for Gradle sync

### 2.5 Delete settings.gradle.kts from notification-service

Spring Initializr creates a `settings.gradle.kts` inside `notification-service/`. **Delete it:**

1. In project tree, find `notification-service/settings.gradle.kts`
2. Right-click → **Delete**
3. Confirm deletion

### 2.6 Update microservices-parent/settings.gradle.kts

Replace the content of `microservices-parent/settings.gradle.kts` with:

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

rootProject.name = "microservices-parent"
include("product-service")
include("order-service")
include("inventory-service")
include("api-gateway")
include("shared-schema")
include("notification-service")
```

### 2.7 Sync Gradle

1. Click the **Gradle elephant icon** in the right sidebar
2. Click the **Refresh** button at the top
3. Wait for sync to complete

---

## Step 3: Update notification-service/build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version "4.0.0"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "ca.gbc.comp3095"
version = "0.0.1-SNAPSHOT"
description = "notification-service"

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
    maven {
        url = uri("https://packages.confluent.io/maven/")
    }
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-kafka")
    implementation("org.springframework.boot:spring-boot-starter-mail")
    implementation("org.springframework.boot:spring-boot-starter-webmvc")
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    // Week 14 - Schema Registry
    implementation("org.apache.avro:avro:1.12.0")
    implementation("io.confluent:kafka-schema-registry-client:8.0.0")
    implementation("io.confluent:kafka-avro-serializer:8.0.0")

    // Week 14 - Shared Schema Project
    implementation(project(":shared-schema"))

    testImplementation("org.springframework.boot:spring-boot-starter-actuator-test")
    testImplementation("org.springframework.boot:spring-boot-starter-kafka-test")
    testImplementation("org.springframework.boot:spring-boot-starter-mail-test")
    testImplementation("org.springframework.boot:spring-boot-starter-webmvc-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## Step 4: Create service Package in notification-service

1. Right-click on `notification-service/src/main/java/ca/gbc/comp3095/notificationservice`
2. Select **New → Package**
3. Name: `service`
4. Click **OK**

Full path: `notification-service/src/main/java/ca/gbc/comp3095/notificationservice/service/`

---

## Step 5: Create notification-service Files

### File 1: notification-service/Dockerfile

```dockerfile
# ------------------
#  Build Stage
# ------------------

FROM gradle:8-jdk21 AS builder

COPY --chown=gradle:gradle . /home/gradle/src

WORKDIR /home/gradle/src

RUN gradle build -x test

# ------------------
#  Package Stage
# ------------------

FROM openjdk:21-jdk

RUN mkdir /app

COPY --from=builder /home/gradle/src/build/libs/*.jar /app/notification-service.jar

EXPOSE 8085

ENTRYPOINT ["java", "-jar", "/app/notification-service.jar"]
```

---

### File 2: notification-service/src/main/java/ca/gbc/comp3095/notificationservice/NotificationServiceApplication.java

```java
package ca.gbc.comp3095.notificationservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class NotificationServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(NotificationServiceApplication.class, args);
    }

}
```

---

### File 3: notification-service/src/main/java/ca/gbc/comp3095/notificationservice/service/NotificationService.java

```java
package ca.gbc.comp3095.notificationservice.service;

// Week 14 - Import from shared-schema generated class
import ca.gbc.comp3095.orderservice.event.OrderPlacedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Service;

import jakarta.mail.MessagingException;
import jakarta.mail.internet.MimeMessage;

@Slf4j
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final JavaMailSender mailSender;

    @Value("${spring.mail.from:comp3095@georgebrown.ca}")
    private String fromEmail;

    @KafkaListener(topics = "order-placed", groupId = "notificationService")
    public void handleOrderPlacedEvent(OrderPlacedEvent event) {
        log.info("Received OrderPlacedEvent for order: {}", event.getOrderNumber());

        MimeMessage message = mailSender.createMimeMessage();
        try {
            MimeMessageHelper helper = new MimeMessageHelper(message, false);
            helper.setFrom(fromEmail);
            helper.setTo(event.getEmail());
            helper.setSubject(String.format("Your Order (%s) was successfully placed", event.getOrderNumber()));
            helper.setText(String.format(
                    """
                    Good Day %s %s,

                    Your Order with order number %s was successfully placed

                    Thank-you for your business
                    COMP3095 Staff
                    """, event.getFirstName(), event.getLastName(), event.getOrderNumber()));
            mailSender.send(message);
            log.info("Order notification email sent to {}", event.getEmail());
        } catch (MessagingException e) {
            log.error("Failed to send email to {}: {}", event.getEmail(), e.getMessage());
            throw new RuntimeException("Failed to send email", e);
        }
    }
}
```

---

### File 4: notification-service/src/main/resources/application.properties

```properties
spring.application.name=notification-service

server.port=8085

# Kafka Consumer Properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=notificationService
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.auto-offset-reset=earliest

# Week 14 - Schema Registry Deserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
spring.kafka.consumer.properties.spring.deserializer.key.delegate.class=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.properties.spring.deserializer.value.delegate.class=io.confluent.kafka.serializers.KafkaAvroDeserializer
spring.kafka.consumer.properties.schema.registry.url=http://localhost:8087
spring.kafka.consumer.properties.specific.avro.reader=true

management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
logging.level.org.springframework.kafka=DEBUG
logging.level.ca.gbc.comp3095.notificationservice=DEBUG

# Week 14 - MailTrap SMTP Configuration
spring.mail.host=sandbox.smtp.mailtrap.io
spring.mail.port=2525
spring.mail.username=YOUR_MAILTRAP_USERNAME
spring.mail.password=YOUR_MAILTRAP_PASSWORD
spring.mail.from=noreply@comp3095.ca
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

---

### File 5: notification-service/src/main/resources/application-docker.properties

```properties
spring.application.name=notification-service

server.port=8085

# Kafka Consumer Properties (Docker)
spring.kafka.bootstrap-servers=broker:29092
spring.kafka.consumer.group-id=notificationService
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.type.mapping=event:ca.gbc.comp3095.notificationservice.events.OrderPlacedEvent
spring.kafka.consumer.auto-offset-reset=earliest

management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
logging.level.org.springframework.kafka=DEBUG
logging.level.ca.gbc.comp3095.notificationservice=DEBUG

# Week 14 - MailTrap SMTP Configuration
spring.mail.host=sandbox.smtp.mailtrap.io
spring.mail.port=2525
spring.mail.username=YOUR_MAILTRAP_USERNAME
spring.mail.password=YOUR_MAILTRAP_PASSWORD
spring.mail.from=noreply@comp3095.ca
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

---

## Step 6: Update order-service

### File 6: order-service/build.gradle.kts

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
    // Week 14 - Confluent Repository for Schema Registry
    maven {
        url = uri("https://packages.confluent.io/maven/")
    }
}

ext {
    set("testcontainers.version", "1.20.4")
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

    // Week 12 - Swagger/OpenAPI Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")

    // Week 13 - Spring Cloud for REST Client and WireMock testing
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j:3.3.0")

    // Week 14 - Kafka
    implementation("org.springframework.kafka:spring-kafka:3.3.7")
    testImplementation("org.springframework.kafka:spring-kafka-test:3.3.7")
    testImplementation("org.testcontainers:kafka:1.21.2")

    // Week 14 - Schema Registry
    implementation("org.apache.avro:avro:1.12.0")
    implementation("io.confluent:kafka-schema-registry-client:8.0.0")
    implementation("io.confluent:kafka-avro-serializer:8.0.0")
    implementation("org.zalando:problem-spring-web:0.29.1")

    // Week 14 - Shared Schema Project
    implementation(project(":shared-schema"))
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

### File 7: order-service/src/main/java/ca/gbc/comp3095/orderservice/dto/OrderRequest.java

```java
package ca.gbc.comp3095.orderservice.dto;

import java.math.BigDecimal;

public record OrderRequest(
        Long id,
        String orderNumber,
        String skuCode,
        BigDecimal price,
        Integer quantity,
        // Week 14 - User details for email notification
        UserDetails userDetails) {

    // Week 14 - Nested record for user information
    public record UserDetails(String email, String firstName, String lastName) {}

}
```

---

### File 8: order-service/src/main/java/ca/gbc/comp3095/orderservice/service/OrderServiceImpl.java

```java
package ca.gbc.comp3095.orderservice.service;

import ca.gbc.comp3095.orderservice.client.InventoryClient;
import ca.gbc.comp3095.orderservice.dto.OrderRequest;
// Week 14 - Import from shared-schema generated class
import ca.gbc.comp3095.orderservice.event.OrderPlacedEvent;
import ca.gbc.comp3095.orderservice.model.Order;
import ca.gbc.comp3095.orderservice.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional
public class OrderServiceImpl implements OrderService {

    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;

    // Week 14 - KafkaTemplate for sending events
    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;

    @Override
    public void placeOrder(OrderRequest orderRequest) {

        try {

            boolean isInStock = inventoryClient.isInStock(orderRequest.skuCode(), orderRequest.quantity());
            if (!isInStock) {
                log.error("Cannot place order for SKU {}: Insufficient inventory", orderRequest.skuCode());
                throw new IllegalStateException("Insufficient inventory for SKU: " + orderRequest.skuCode());
            }

            // Create and save order
            Order order = Order.builder()
                    .orderNumber(UUID.randomUUID().toString())
                    .price(orderRequest.price())
                    .skuCode(orderRequest.skuCode())
                    .quantity(orderRequest.quantity())
                    .build();

            orderRepository.save(order);
            log.info("Order placed successfully: {}", order.getOrderNumber());

            // Week 14 - Create Avro event using Schema Registry
            OrderPlacedEvent orderPlacedEvent = new OrderPlacedEvent();
            orderPlacedEvent.setOrderNumber(order.getOrderNumber());
            orderPlacedEvent.setEmail(orderRequest.userDetails().email());
            orderPlacedEvent.setFirstName(orderRequest.userDetails().firstName());
            orderPlacedEvent.setLastName(orderRequest.userDetails().lastName());

            // Week 14 - Send message to Kafka topic
            log.info("Start - Sending OrderPlacedEvent {} to Kafka topic 'order-placed'", orderPlacedEvent);
            kafkaTemplate.send("order-placed", orderPlacedEvent);
            log.info("End - OrderPlacedEvent {} sent to Kafka topic 'order-placed'", orderPlacedEvent);


        } catch(Exception ex) {
            log.error("Failed to place order: {}", ex.getMessage());
            throw new RuntimeException("Order placement failed");
        }
    }

}
```

---

### File 9: order-service/src/main/resources/application.properties

```properties
# ============================================
# Order Service - Local Configuration
# Use this when running from IntelliJ IDE
# ============================================

# Application Configuration
spring.application.name=order-service

# Server Configuration
server.port=8082

# PostgreSQL Configuration for LOCAL development
spring.datasource.url=jdbc:postgresql://localhost:5432/order_service
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA/Hibernate Configuration
spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.format_sql=true

# Logging Configuration
logging.level.ca.gbc.comp3095=INFO
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE

# Actuator Configuration
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

inventory.service.url=http://localhost:8083

# Week 12 - API version for documentation
order-service.version=v1.0

# Week 12 - Swagger Documentation
# Swagger UI accessible at: http://localhost:8082/swagger-ui
springdoc.swagger-ui.path=/swagger-ui
# OpenAPI JSON accessible at: http://localhost:8082/api-docs
springdoc.api-docs.path=/api-docs

# Week 13
spring.flyway.baseline-on-migrate=true
spring.flyway.locations=classpath:db/migration
spring.flyway.enabled=true

# Week 9 - Day 1 - Circuit Breaker Configuration
management.health.circuitbreakers.enabled=true

# Week 9 - Day 1 - Resilience4j Circuit Breaker Configuration
# Opens if 50% of 20 calls fail after at least 10 calls.
# Stays open for 10 seconds, then allows 5 calls in half-open state
resilience4j.circuitbreaker.instances.inventory.registerHealthIndicator=true
resilience4j.circuitbreaker.instances.inventory.event-consumer-buffer-size=10
resilience4j.circuitbreaker.instances.inventory.slidingWindowType=COUNT_BASED
resilience4j.circuitbreaker.instances.inventory.slidingWindowSize=10
resilience4j.circuitbreaker.instances.inventory.failureRateThreshold=50
resilience4j.circuitbreaker.instances.inventory.waitDurationInOpenState=5s
resilience4j.circuitbreaker.instances.inventory.permittedNumberOfCallsInHalfOpenState=3
resilience4j.circuitbreaker.instances.inventory.automaticTransitionFromOpenToHalfOpenEnabled=true

resilience4j.timelimiter.instances.inventory.timeout-duration=3s
resilience4j.circuitbreaker.instances.inventory.minimum-number-of-calls=5

resilience4j.retry.instances.inventory.max-attempts=3
resilience4j.retry.instances.inventory.wait-duration=2s

# Week 14 - Kafka Producer Properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.template.default-topic=order-placed
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer

# Week 14 - Schema Registry Serializer
spring.kafka.producer.value-serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
spring.kafka.producer.properties.schema.registry.url=http://127.0.0.1:8087
```

---

### File 10: order-service/src/main/resources/application-docker.properties

```properties
# ============================================
# Order Service - Docker Configuration
# Activated when SPRING_PROFILES_ACTIVE=docker
# ============================================

# Application Configuration
spring.application.name=order-service

# Server Configuration
server.port=8082

# PostgreSQL Configuration for DOCKER environment
# IMPORTANT: Use service name "postgres" instead of "localhost"
spring.datasource.url=jdbc:postgresql://postgres-order:5432/order_service

spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA/Hibernate Configuration
spring.jpa.hibernate.ddl-auto=none
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.format_sql=true

# Logging Configuration
logging.level.ca.gbc.comp3095=INFO
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE

# Actuator Configuration
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

inventory.service.url=http://inventory-service:8083

# Week 12 - API version for documentation
order-service.version=v1.0

# Week 12 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs

# Week 13
spring.flyway.baseline-on-migrate=true
spring.flyway.locations=classpath:db/migration
spring.flyway.enabled=true

# Week 9 - Day 1 - Circuit Breaker Configuration
management.health.circuitbreakers.enabled=true

# Week 9 - Day 1 - Resilience4j Circuit Breaker Configuration
resilience4j.circuitbreaker.instances.inventory.registerHealthIndicator=true
resilience4j.circuitbreaker.instances.inventory.event-consumer-buffer-size=10
resilience4j.circuitbreaker.instances.inventory.slidingWindowType=COUNT_BASED
resilience4j.circuitbreaker.instances.inventory.slidingWindowSize=10
resilience4j.circuitbreaker.instances.inventory.failureRateThreshold=50
resilience4j.circuitbreaker.instances.inventory.waitDurationInOpenState=5s
resilience4j.circuitbreaker.instances.inventory.permittedNumberOfCallsInHalfOpenState=3
resilience4j.circuitbreaker.instances.inventory.automaticTransitionFromOpenToHalfOpenEnabled=true

resilience4j.timelimiter.instances.inventory.timeout-duration=3s
resilience4j.circuitbreaker.instances.inventory.minimum-number-of-calls=5

resilience4j.retry.instances.inventory.max-attempts=3
resilience4j.retry.instances.inventory.wait-duration=2s

# Week 14 - Kafka Producer Properties (Docker)
spring.kafka.bootstrap-servers=broker:29092
spring.kafka.template.default-topic=order-placed
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.producer.properties.spring.json.type.mapping=event:ca.gbc.comp3095.orderservice.events.OrderPlacedEvent
```

---

## Step 7: Update docker-compose.yml

### File 11: microservices-parent/docker-compose.yml

```yaml
# -------------------------------------------
# Commands to run this compose file:
# - docker-compose -p microservices-comp3095 -f docker-compose.yml up -d
# - docker-compose -f docker-compose.yml up -d --build
# -------------------------------------------

services:
  # Week 14 - Notification Service
  notification-service:
    image: notification-service
    ports:
      - "8085:8085"
    build:
      context: ./notification-service
      dockerfile: ./Dockerfile
    container_name: notification-service
    environment:
      SPRING_PROFILES_ACTIVE: docker
    depends_on:
      broker:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8085/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  keycloak:
    image: quay.io/keycloak/keycloak:24.0.1
    command: ["start-dev", "--import-realm"]
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: postgres-keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: password
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: password
    ports:
      - "8080:8080"
    volumes:
      - ./docker/integrated/keycloak/realms/:/opt/keycloak/data/import/
    depends_on:
      postgres-keycloak:
        condition: service_healthy
    container_name: keycloak
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  api-gateway:
    image: api-gateway
    ports:
      - "9000:9000"
    build:
      context: ./api-gateway
      dockerfile: ./Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_APPLICATION_JSON: '{"logging":{"level":{"root":"INFO","ca.gbc.comp3095.apigateway":"DEBUG"}}}'
    container_name: api-gateway
    depends_on:
      keycloak:
        condition: service_healthy
      order-service:
        condition: service_healthy
      inventory-service:
        condition: service_healthy
      product-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  redis:
    image: redis:latest
    ports:
      - "6379:6379"
    volumes:
      - ./docker/integrated/redis/data:/data
      - ./docker/integrated/redis/init/redis.conf:/usr/local/etc/redis/redis.conf
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    container_name: redis
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  redis-insight:
    image: redislabs/redisinsight:1.14.0
    ports:
      - "8001:8001"
    container_name: redis-insight
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  inventory-service:
    image: inventory-service
    ports:
      - "8083:8083"
    build:
      context: ./inventory-service
      dockerfile: ./Dockerfile
    container_name: inventory-service
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-inventory:5432/inventory-service
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: none
      SPRING_PROFILES_ACTIVE: docker
    depends_on:
      postgres-inventory:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8083/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  order-service:
    image: order-service
    ports:
      - "8082:8082"
    build:
      context: ./order-service
      dockerfile: ./Dockerfile
    container_name: order-service
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres-order:5432/order-service
      SPRING_DATASOURCE_USERNAME: admin
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_JPA_HIBERNATE_DDL_AUTO: none
      SPRING_PROFILES_ACTIVE: docker
    depends_on:
      postgres-order:
        condition: service_healthy
      broker:
        condition: service_healthy
      inventory-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8082/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  product-service:
    image: product-service
    ports:
      - "8084:8084"
    build:
      context: ./product-service
      dockerfile: ./Dockerfile
    container_name: product-service
    environment:
      SPRING_PROFILES_ACTIVE: docker
    depends_on:
      mongodb:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8084/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  mongodb:
    image: mongo:latest
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - ./docker/integrated/mongo/data/mongo/products:/data/db
      - ./docker/integrated/mongo/init/mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js
    container_name: mongodb
    command: mongod --auth
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  mongo-express:
    image: mongo-express
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_SERVER: mongodb
      ME_CONFIG_MONGODB_AUTH_DATABASE: admin
    container_name: mongo-express
    depends_on:
      mongodb:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  postgres-keycloak:
    image: postgres:15
    volumes:
      - ./docker/integrated/keycloak/postgres/db-data:/var/lib/postgresql/data
    ports:
      - "5434:5432"
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: password
    container_name: postgres-keycloak
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak -d keycloak"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  postgres-inventory:
    container_name: postgres-inventory
    image: postgres:16
    environment:
      POSTGRES_DB: inventory-service
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      PGDATA: /data/postgres
    volumes:
      - ./docker/integrated/postgres/data/postgres-inventory:/data/postgres
      - ./docker/integrated/postgres/inventory-service/init/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5433:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d inventory-service"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  postgres-order:
    container_name: postgres-order
    image: postgres:16
    environment:
      POSTGRES_DB: order-service
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      PGDATA: /data/postgres
    volumes:
      - ./docker/integrated/postgres/data/postgres-order:/data/postgres
      - ./docker/integrated/postgres/order-service/init/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d order-service"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  pgpadmin:
    image: dpage/pgadmin4:9.2
    ports:
      - "8888:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: user@domain.ca
      PGADMIN_DEFAULT_PASSWORD: password
    container_name: pgadmin
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  # Week 14 - Zookeeper (Kafka dependency)
  zookeeper:
    container_name: zookeeper
    hostname: zookeeper
    image: confluentinc/cp-zookeeper:7.9.2
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - ./docker/integrated/kafka/zookeeper_data:/var/lib/zookeeper/data
      - ./docker/integrated/kafka/zookeeper_log:/var/lib/zookeeper/log
    healthcheck:
      test: ["CMD", "zookeeper-shell", "zookeeper:2181", "ls", "/"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  # Week 14 - Kafka Broker
  broker:
    container_name: broker
    image: confluentinc/cp-kafka:7.9.2
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      zookeeper:
        condition: service_healthy
    volumes:
      - ./docker/integrated/kafka/kafka_data:/var/lib/kafka/data
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions.sh", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  # Week 14 - Schema Registry
  schema-registry:
    container_name: schema-registry
    hostname: schema-registry
    image: confluentinc/cp-schema-registry:7.9.2
    ports:
      - "8087:8087"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://broker:29092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8087
    depends_on:
      broker:
        condition: service_healthy
    healthcheck:
      test: [ "CMD", "curl", "-f", "http://localhost:8087" ]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

  # Week 14 - Kafka UI
  kafka-ui:
    container_name: kafka-ui
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8086:8080"
    depends_on:
      broker:
        condition: service_healthy
    environment:
      KAFKA_CLUSTERS_NAME: local
      KAFKA_CLUSTERS_BOOTSTRAPSERVERS: broker:29092
      KAFKA_CLUSTERS_SCHEMAREGISTRY: http://schema-registry:8087
      DYNAMIC_CONFIG_ENABLED: 'true'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

volumes:
  mongo-db:
    driver: local
  zookeeper_data:
    driver: local
  zookeeper_log:
    driver: local
  kafka_data:
    driver: local
  postgres_order_data:
    driver: local
  postgres_inventory_data:
    driver: local
  postgres_keycloak_data:
    driver: local
  schema_registry_data:
    driver: local

networks:
  spring:
    driver: bridge
```

---

## Summary

| Step | Action |
|------|--------|
| 1 | Create `shared-schema` module, update build.gradle.kts, create avro schema, **sync Gradle** |
| 2 | Create `notification-service` module via Spring Initializr |
| 3 | Update `notification-service/build.gradle.kts` |
| 4 | Create `service` package in notification-service |
| 5 | Create notification-service files (Dockerfile, Application, Service, properties) |
| 6 | Update order-service files (build.gradle.kts, OrderRequest, OrderServiceImpl, properties) |
| 7 | Update docker-compose.yml |
