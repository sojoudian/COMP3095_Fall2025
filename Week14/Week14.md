# Week 14 - Kafka Event-Driven Architecture & Schema Registry

## settings.gradle.kts
```kotlin
rootProject.name = "microservices-parent"

include("product-service", "order-service", "inventory-service", "api-gateway", "notification-service")
include("shared-schema")
```

---

## shared-schema/build.gradle.kts
```kotlin
plugins {
    id("java-library") // Library plugin
    id("com.github.davidmc24.gradle.plugin.avro") version "1.9.1" // Add Avro Gradle Plugin
}

group = "ca.gbc"
version = "1.0.0"

repositories {
    maven { url = uri("https://packages.confluent.io/maven/") }
    mavenCentral()
}

dependencies {
    implementation("org.apache.avro:avro:1.12.0")
}

avro {
    isCreateSetters = true
    fieldVisibility = "PRIVATE"
    isEnableDecimalLogicalType = true
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

---

## shared-schema/src/main/avro/order-placed.avsc
```json
{
  "type": "record",
  "name": "OrderPlacedEvent",
  "namespace": "ca.gbc.orderservice.event",
  "fields": [
    { "name": "orderNumber", "type": "string" },
    { "name": "email", "type": "string" },
    { "name": "firstName", "type": "string" },
    { "name": "lastName", "type": "string" }
  ]
}
```

---

## notification-service/build.gradle.kts
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
    implementation("org.springframework.boot:spring-boot-starter-mail")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.kafka:spring-kafka")
    compileOnly("org.projectlombok:lombok")

    //Week 10 - Day 1
    implementation("org.apache.avro:avro:1.12.0")
    implementation("io.confluent:kafka-schema-registry-client:8.0.0")
    implementation("io.confluent:kafka-avro-serializer:8.0.0")

    //Week 10 - Day 1 - Shared Schema Project
    implementation(project(":shared-schema"))

    //Week 10 - Day 1 - REMOVED FOR Week 10 - Day 1
    //developmentOnly("org.springframework.boot:spring-boot-devtools")

    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("org.springframework.kafka:spring-kafka-test")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("org.testcontainers:kafka")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## notification-service/Dockerfile
```dockerfile
# ------------------
#  Build Stage
# ------------------

# Start from the Gradle 8 image with JDK 22. This image provides both Gradle and JDK 22.
# We use this stage to build the application within the container.
FROM gradle:8-jdk21 AS builder

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
FROM openjdk:21-jdk

# Create a directory inside the container where the application will be stored.
# This directory is where we will place the packaged .jar file built in the previous stage.
RUN mkdir /app

# Copy the built .jar file from the build stage to the /app directory in the final image.
# We use the '--from=builder' instruction to reference the "builder" stage.
COPY --from=builder /home/gradle/src/build/libs/*.jar /app/notification-service.jar

# Expose port 8084 to allow communication with the containerized application.
# EXPOSE does not actually make the port accessible to the host machine; it's documentation for the image.
EXPOSE 8085

# The ENTRYPOINT instruction defines the command to run when the container starts.
# In this case, we are telling Docker to run the Java command with the packaged JAR file.
ENTRYPOINT ["java", "-jar", "/app/notification-service.jar"]
```

---

## notification-service/src/main/java/ca/gbc/notificationservice/NotificationServiceApplication.java
```java
package ca.gbc.notificationservice;

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

## notification-service/src/main/java/ca/gbc/notificationservice/service/NotificationService.java
```java
package ca.gbc.notificationservice.service;

//Week 10 - Day 1
import ca.gbc.orderservice.event.OrderPlacedEvent;
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
            MimeMessageHelper helper = new MimeMessageHelper(message, false); // Non-multipart
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

## notification-service/src/main/resources/application.properties
```properties
spring.application.name=notification-service

server.port=8085

# Kafka Consumer Properties
# The address of the Kafka broker used to connect to the Kafka cluster.
spring.kafka.bootstrap-servers=localhost:9092
# The unique identifier for the consumer group to which this consumer belongs.
spring.kafka.consumer.group-id=notificationService
# The deserializer class used for deserializing the key of the messages from Kafka.
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
# Will ensure your consumer starts processing all unprocessed events from the beginning of the topic
spring.kafka.consumer.auto-offset-reset=earliest

# Week 10 - Day 2
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
spring.kafka.consumer.properties.spring.deserializer.key.delegate.class=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.properties.spring.deserializer.value.delegate.class=io.confluent.kafka.serializers.KafkaAvroDeserializer
spring.kafka.consumer.properties.schema.registry.url=http://localhost:8087
spring.kafka.consumer.properties.specific.avro.reader=true

management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
logging.level.org.springframework.kafka=DEBUG
logging.level.ca.gbc.notificationservice=DEBUG

spring.mail.host=sandbox.smtp.mailtrap.io
spring.mail.port=2525
spring.mail.username=68347f4128ddcf
spring.mail.password=75c591488d271c
spring.mail.from=noreply@comp3095.ca
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

---

## notification-service/src/main/resources/application-docker.properties
```properties
spring.application.name=notification-service

# Week 9 - Day 2
server.port=8085

# Week 9 - Day 2
# Kafka Consumer Properties
# The address of the Kafka broker used to connect to the Kafka cluster.
spring.kafka.bootstrap-servers=broker:29092
# The unique identifier for the consumer group to which this consumer belongs.
spring.kafka.consumer.group-id=notificationService
# The deserializer class used for deserializing the key of the messages from Kafka.
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
# The deserializer class used for deserializing the value of the messages from Kafka in JSON format.
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
# Maps a custom event type to a specific class for deserialization of incoming JSON messages.
spring.kafka.consumer.properties.spring.json.type.mapping=event:ca.gbc.notificationservice.events.OrderPlacedEvent
# Will ensure your consumer starts processing all unprocessed events from the beginning of the topic
spring.kafka.consumer.auto-offset-reset=earliest

management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
logging.level.org.springframework.kafka=DEBUG
logging.level.ca.gbc.notificationservice=DEBUG

# Week 9 - Day 2
spring.mail.host=sandbox.smtp.mailtrap.io
spring.mail.port=2525
spring.mail.username=68347f4128ddcf
spring.mail.password=75c591488d271c
spring.mail.from=noreply@comp3095.ca
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

---

## order-service/build.gradle.kts
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
    //Week 10 - Day 1
    maven {
        url = uri("https://packages.confluent.io/maven/")
    }
}

//Week 4 - Day 1
dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2024.0.1") // Replace with the latest or required version
    }
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")
    implementation("org.springframework.cloud:spring-cloud-starter-contract-stub-runner")
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j:3.3.0")
    implementation("org.springframework.kafka:spring-kafka:3.3.7")
    testImplementation("org.springframework.kafka:spring-kafka-test:3.3.7")
    testImplementation("org.testcontainers:kafka:1.21.2")

    //Week 10 - Day 1
    implementation("org.apache.avro:avro:1.12.0")
    implementation("io.confluent:kafka-schema-registry-client:8.0.0")
    implementation("io.confluent:kafka-avro-serializer:8.0.0")
    implementation("org.zalando:problem-spring-web:0.29.1")

    //Week 10 - Day 1 - Shared Schema Project
    implementation(project(":shared-schema"))

    compileOnly("org.projectlombok:lombok")

    //Week 10 - Day 1 - REMOVED FOR Week 10 - Day 1
    //developmentOnly("org.springframework.boot:spring-boot-devtools")

    runtimeOnly("org.postgresql:postgresql")
    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("org.testcontainers:postgresql")
    testImplementation("io.rest-assured:rest-assured")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## order-service/src/main/java/ca/gbc/orderservice/dto/OrderRequest.java
```java
package ca.gbc.orderservice.dto;

import java.math.BigDecimal;

public record OrderRequest(
        Long id,
        String orderNumber,
        String skuCode,
        BigDecimal price,
        Integer quantity,
        //Week 9 - Day 2
        UserDetails userDetails) {

    //Week 9 - Day 2
    public record UserDetails(String email, String firstName, String lastName) {}

}
```

---

## order-service/src/main/java/ca/gbc/orderservice/service/OrderServiceImpl.java
```java
package ca.gbc.orderservice.service;

import ca.gbc.orderservice.client.InventoryClient;
import ca.gbc.orderservice.dto.OrderRequest;
import ca.gbc.orderservice.event.OrderPlacedEvent;
import ca.gbc.orderservice.model.Order;
import ca.gbc.orderservice.repository.OrderRepository;
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

    //We use the @RequiredArgsConstructor to resolve the dependency via constructor injection
    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;

    //Week 9 - Day 2
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

            //Week 10 - Day 2 - end message to Kafka topic
            //Schema Registry
            OrderPlacedEvent orderPlacedEvent = new OrderPlacedEvent();
            orderPlacedEvent.setOrderNumber(order.getOrderNumber());
            orderPlacedEvent.setEmail(orderRequest.userDetails().email());
            orderPlacedEvent.setFirstName(orderRequest.userDetails().firstName());
            orderPlacedEvent.setLastName(orderRequest.userDetails().lastName());

            //Week 9 - Day 2 - end message to Kafka topic
            log.info("Start - Sending OrderPlacedEvent {} to Kakfa topic 'order-placed'", orderPlacedEvent);
            kafkaTemplate.send("order-placed", orderPlacedEvent);
            log.info("End - OrderPlacedEvent {} sent to Kakfa topic 'order-placed'", orderPlacedEvent);


        }catch(Exception ex){
            log.error("Failed to place order: {}", ex.getMessage());
            throw new RuntimeException("Order placement failed");
        }
    }


}
```

---

## order-service/src/main/resources/application.properties
```properties
spring.application.name=order-service
order-service.version=v1.0

server.port=8082

# default is 5432 for postgres
# local-connections
spring.datasource.url=jdbc:postgresql://localhost:5433/order_service

# for container based connections
#spring.datasource.url=jdbc:postgresql://host.docker.internal:5431/order-service

spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# none, validate, update, create, create-drop
# Options for controlling how Hibernate handles schema management
# none: No schema generation or validation is performed.
# none because we will be using flyaway
spring.jpa.hibernate.ddl-auto=none

# validate: Hibernate will validate the schema against the database without making any changes.
# This is useful to ensure that the database structure matches the entity mappings.
# Validation failure will throw an error.
#spring.jpa.hibernate.ddl-auto=validate

# update: Hibernate will modify the database schema to match the entity mappings.
# Only adds missing columns and tables, but does not remove anything.
# This is useful for iterative development but should be avoided in production.
#spring.jpa.hibernate.ddl-auto=update

# create: Hibernate will drop the existing schema and recreate it every time the application starts.
# This is useful during development but should not be used in production environments as it will
# erase all data on each startup.
#spring.jpa.hibernate.ddl-auto=create

# create-drop: Similar to "create", but in addition, it will drop the schema when the application stops.
# It's useful for integration tests where the schema only needs to exist for the duration of the app run.
#spring.jpa.hibernate.ddl-auto=create-drop

inventory.service.url=http://localhost:8083

# Swagger documentation location - ex. http://localhost:8082/swagger-ui
springdoc.swagger-ui.path=/swagger-ui
# Swagger documentation location - ex. http://localhost:8082/api-docs
springdoc.api-docs.path=/api-docs

management.health.circuitbreakers.enabled=true
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

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

# Kafka Producer Properties - Week 9 - Day 2
# The address of the Kafka broker used to connect to the Kafka cluster.
spring.kafka.bootstrap-servers=localhost:9092
# The default topic where messages will be sent if not explicitly specified.
spring.kafka.template.default-topic=order-placed
# The serializer class used for serializing the key of the messages to Kafka.
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer

# The serializer class used for serializing the value of the messages to Kafka in JSON format.
#spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
# Maps a custom event type to a specific class for deserialization of incoming JSON messages.
#spring.kafka.producer.properties.spring.json.type.mapping=event:ca.gbc.orderservice.event.OrderPlacedEvent

# Week 10 - Day 2
# Update for Schema Registry
spring.kafka.producer.value-serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
# Schema Registry Container Port
spring.kafka.producer.properties.schema.registry.url=http://127.0.0.1:8087
```

---

## order-service/src/main/resources/application-docker.properties
```properties
spring.application.name=order-service
#? Week 6 - Day 1
order-service.version=v1.0

server.port=8082

# default is 5432 for postgres
# docker network-connections
spring.datasource.url=jdbc:postgresql://postgres-order:5432/order_service

# for container based connections
#spring.datasource.url=jdbc:postgresql://host.docker.internal:5431/order-service

spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# none, validate, update, create, create-drop
# Options for controlling how Hibernate handles schema management
# none: No schema generation or validation is performed.
# none because we will be using flyaway
spring.jpa.hibernate.ddl-auto=none

# validate: Hibernate will validate the schema against the database without making any changes.
# This is useful to ensure that the database structure matches the entity mappings.
# Validation failure will throw an error.
#spring.jpa.hibernate.ddl-auto=validate

# update: Hibernate will modify the database schema to match the entity mappings.
# Only adds missing columns and tables, but does not remove anything.
# This is useful for iterative development but should be avoided in production.
#spring.jpa.hibernate.ddl-auto=update

# create: Hibernate will drop the existing schema and recreate it every time the application starts.
# This is useful during development but should not be used in production environments as it will
# erase all data on each startup.
#spring.jpa.hibernate.ddl-auto=create

# create-drop: Similar to "create", but in addition, it will drop the schema when the application stops.
# It's useful for integration tests where the schema only needs to exist for the duration of the app run.
#spring.jpa.hibernate.ddl-auto=create-drop

inventory.service.url=http://inventory-service:8083

#? Week 6 - Day 1 - Swagger documentation location
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs

# Week 9 - Day 1
management.health.circuitbreakers.enabled=true
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

# Week 9 - Day 1
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

# Kafka Producer Properties - Week 9 - Day 2
# The address of the Kafka broker used to connect to the Kafka cluster.
spring.kafka.bootstrap-servers=broker:29092
# The default topic where messages will be sent if not explicitly specified.
spring.kafka.template.default-topic=order-placed
# The serializer class used for serializing the key of the messages to Kafka.
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
# The serializer class used for serializing the value of the messages to Kafka in JSON format.
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
# Maps a custom event type to a specific class for deserialization of incoming JSON messages.
spring.kafka.producer.properties.spring.json.type.mapping=event:ca.gbc.orderservice.events.OrderPlacedEvent
```

---

## docker-compose.yml
```yaml
# -------------------------------------------
# Commands to run this compose file:
# - docker-compose -p microservices-comp3095 -f docker-compose.yml up -d
#   This command will start the containers in detached mode (-d) without rebuilding the images (if they already exist).
#
# - docker-compose -f docker-compose.yml up -d --build
#   This command forces the rebuild of images, even if they already exist, before starting the containers.
# -------------------------------------------

# Define services (containers) that will run as part of the microservices stack.
services:
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
      SPRING_APPLICATION_JSON: '{"logging":{"level":{"root":"INFO","ca.gbc.apigateway":"DEBUG"}}}'
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

  #Week 10 - Day 1
  # Schema Registry service for managing Avro schemas used by Kafka producers and consumers.
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
      #Week 10 - Day 1
      KAFKA_CLUSTERS_SCHEMAREGISTRY: http://schema-registry:8087
      DYNAMIC_CONFIG_ENABLED: 'true'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - spring

# Optional volumes section for persisting data.
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
  #Week 10 - Day 1
  schema_registry_data:
    driver: local

# Define a custom network called 'spring' using the bridge driver.
# Containers on this network can communicate with each other using their container names.
networks:
  spring:
    driver: bridge
```
