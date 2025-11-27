# Week 13 Complete Configuration

## API Gateway Files

### File 1: api-gateway/build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.7"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "ca.gbc.comp3095"
version = "0.0.1-SNAPSHOT"
description = "api-gateway"

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

extra["springCloudVersion"] = "2025.0.0"

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.cloud:spring-cloud-starter-gateway-server-webmvc")
    compileOnly("org.projectlombok:lombok")
    developmentOnly("org.springframework.boot:spring-boot-devtools")
    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")

    // Week 12 - Security
    compileOnly("jakarta.servlet:jakarta.servlet-api:6.1.0")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("org.springframework.boot:spring-boot-starter-security")

    // Week 13 - Swagger Documentation Aggregation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.9")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.9")

    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j:3.3.0")

}

dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:${property("springCloudVersion")}")
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### File 2: api-gateway/src/main/java/ca/gbc/comp3095/apigateway/routes/Routes.java

```java
package ca.gbc.comp3095.apigateway.routes;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gateway.server.mvc.filter.CircuitBreakerFilterFunctions;
import org.springframework.cloud.gateway.server.mvc.handler.GatewayRouterFunctions;
import org.springframework.cloud.gateway.server.mvc.handler.HandlerFunctions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpStatus;
import org.springframework.web.servlet.function.*;

import java.net.URI;

import static org.springframework.cloud.gateway.server.mvc.filter.FilterFunctions.setPath;
import static org.springframework.cloud.gateway.server.mvc.handler.GatewayRouterFunctions.route;

/**
 * Configuration class for defining API Gateway routes.
 * This class sets up routing rules to forward requests to the product, order, and inventory microservices
 * using Spring Cloud Gateway's functional routing approach.
 */
@Configuration
@Slf4j
public class Routes {

    @Value("${services.product-url}")
    private String productServiceUrl;

    @Value("${services.order-url}")
    private String orderServiceUrl;

    @Value("${services.inventory-url}")
    private String inventoryServiceUrl;

    @Bean
    public RouterFunction<ServerResponse> productServiceRoute() {
        return GatewayRouterFunctions.route("product_service")
                .route(
                        RequestPredicates.path("/api/product"),
                        HandlerFunctions.http(productServiceUrl)
                )
                .filter(CircuitBreakerFilterFunctions.circuitBreaker("productServiceCircuitBreaker",
                        URI.create("forward:/fallbackRoute")))
                .build();
    }

    @Bean
    public RouterFunction<ServerResponse> productServiceSwaggerRoute() {
        return GatewayRouterFunctions.route("product_service_swagger")
                .route(RequestPredicates.path("/aggregate/product-service/v3/api-docs"),
                        HandlerFunctions.http(productServiceUrl))
                .filter(CircuitBreakerFilterFunctions.circuitBreaker("productServiceSwaggerCircuitBreaker",
                        URI.create("forward:/fallbackRoute")))
                .filter(setPath("/api-docs"))
                .build();
    }

    @Bean
    public RouterFunction<ServerResponse> orderServiceRoute() {
        return GatewayRouterFunctions.route("order_service")
                .route(
                        RequestPredicates.path("/api/order"),
                        HandlerFunctions.http(orderServiceUrl)
                )
                .filter(CircuitBreakerFilterFunctions.circuitBreaker("orderServiceCircuitBreaker",
                        URI.create("forward:/fallbackRoute")))
                .build();
    }

    @Bean
    public RouterFunction<ServerResponse> orderServiceSwaggerRoute() {
        return GatewayRouterFunctions.route("order_service_swagger")
                .route(RequestPredicates.path("/aggregate/order-service/v3/api-docs"),
                        HandlerFunctions.http(orderServiceUrl))
                .filter(CircuitBreakerFilterFunctions.circuitBreaker("orderServiceSwaggerCircuitBreaker",
                        URI.create("forward:/fallbackRoute")))
                .filter(setPath("/api-docs"))
                .build();
    }

    @Bean
    public RouterFunction<ServerResponse> inventoryServiceRoute() {
        return GatewayRouterFunctions.route("inventory_service")
                .route(
                        RequestPredicates.path("/api/inventory"),
                        HandlerFunctions.http(inventoryServiceUrl)
                )
                .filter(CircuitBreakerFilterFunctions.circuitBreaker("inventoryServiceCircuitBreaker",
                        URI.create("forward:/fallbackRoute")))
                .build();
    }

    @Bean
    public RouterFunction<ServerResponse> inventoryServiceSwaggerRoute() {
        return GatewayRouterFunctions.route("inventory_service_swagger")
                .route(RequestPredicates.path("/aggregate/inventory-service/v3/api-docs"),
                        HandlerFunctions.http(inventoryServiceUrl))
                .filter(CircuitBreakerFilterFunctions.circuitBreaker("inventoryServiceSwaggerCircuitBreaker",
                        URI.create("forward:/fallbackRoute")))
                .filter(setPath("/api-docs"))
                .build();
    }

    @Bean
    public RouterFunction<ServerResponse> fallbackRoute() {
        log.info("Fallback route triggered");
        return route("fallbackRoute")
                .GET("/fallbackRoute", request -> ServerResponse.status(HttpStatus.SERVICE_UNAVAILABLE)
                        .body("Service Unavailable, please try again in a few minutes"))
                .POST("/fallbackRoute", request -> ServerResponse.status(HttpStatus.SERVICE_UNAVAILABLE)
                        .body("Service Unavailable, please try again in a few minutes"))
                .build();
    }
}
```

### File 3: api-gateway/src/main/java/ca/gbc/comp3095/apigateway/config/SecurityConfig.java

```java
package ca.gbc.comp3095.apigateway.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityCustomizer;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.web.SecurityFilterChain;

@Slf4j
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // Week 13 - Whitelist documentation AND API endpoints for testing (no authentication required)
    private final String[] noauthResourceUrls = {
            "/swagger-ui",
            "/swagger-ui/**",
            "/v3/api-docs/**",
            "/swagger-resources/**",
            "/api-docs/**",
            "/aggregate/**",
            "/api/product",
            "/api/product/**",
            "/api/order",
            "/api/order/**",
            "/api/inventory",
            "/api/inventory/**"
    };

    /**
     * WebSecurityCustomizer to completely bypass Spring Security for whitelisted paths.
     * These paths will not pass through the security filter chain at all.
     */
    @Bean
    public WebSecurityCustomizer webSecurityCustomizer() {
        log.info("Configuring WebSecurityCustomizer to ignore whitelisted paths...");
        return (web) -> web.ignoring().requestMatchers(noauthResourceUrls);
    }

    /**
     * Configures the security filter chain for HTTP requests.
     * Week 13: Temporarily disable OAuth2 for testing - all paths use permitAll
     */
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {

        log.info("Initializing Security Filter Chain...");

        return httpSecurity
                .csrf(AbstractHttpConfigurer::disable)
                .authorizeHttpRequests(authorize -> authorize
                        .anyRequest().permitAll())
                .build();
    }

}
```

### File 4: api-gateway/src/main/resources/application.properties

```properties
spring.application.name=api-gateway

server.port=9000

#services
services.product-url=http://localhost:8084
services.order-url=http://localhost:8082
services.inventory-url=http://localhost:8083


# Week 12 - Keycloak Security
# The API Gateway uses Keycloak's public key to validate JWT tokens locally
# Keycloak Security
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://keycloak:8080/realms/spring-microservices-security-realm

# Week 13 - Swagger UI Configuration
springdoc.swagger-ui.path=/swagger-ui
# Aggregate documentation from all microservices
springdoc.swagger-ui.urls[0].name=Product Service
springdoc.swagger-ui.urls[0].url=/aggregate/product-service/v3/api-docs
springdoc.swagger-ui.urls[1].name=Order Service
springdoc.swagger-ui.urls[1].url=/aggregate/order-service/v3/api-docs
springdoc.swagger-ui.urls[2].name=Inventory Service
springdoc.swagger-ui.urls[2].url=/aggregate/inventory-service/v3/api-docs

management.health.circuitbreakers.enabled=true
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

resilience4j.circuitbreaker.configs.default.registerHealthIndicator=true
resilience4j.circuitbreaker.configs.default.event-consumer-buffer-size=10
resilience4j.circuitbreaker.configs.default.slidingWindowType=COUNT_BASED
resilience4j.circuitbreaker.configs.default.slidingWindowSize=10
resilience4j.circuitbreaker.configs.default.failureRateThreshold=50
resilience4j.circuitbreaker.configs.default.waitDurationInOpenState=5s
resilience4j.circuitbreaker.configs.default.permittedNumberOfCallsInHalfOpenState=3
resilience4j.circuitbreaker.configs.default.automaticTransitionFromOpenToHalfOpenEnabled=true

resilience4j.timelimiter.configs.default.timeout-duration=3s
resilience4j.circuitbreaker.configs.default.minimum-number-of-calls=5

resilience4j.retry.configs.default.max-attempts=3
resilience4j.retry.configs.default.wait-duration=2s
```

### File 5: api-gateway/src/main/resources/application-docker.properties

```properties
spring.application.name=api-gateway

server.port=9000

#services
services.product-url=http://product-service:8084
services.order-url=http://order-service:8082
services.inventory-url=http://inventory-service:8083

# Week 12 - Keycloak Security (Docker)
# Keycloak Security (Docker)
# Week 13 - Temporarily disable OAuth2 auto-configuration for testing
# spring.security.oauth2.resourceserver.jwt.issuer-uri=http://keycloak:8080/realms/spring-microservices-security-realm


# Week 13 - Swagger UI Configuration
springdoc.swagger-ui.path=/swagger-ui
# Aggregate documentation from all microservices
springdoc.swagger-ui.urls[0].name=Product Service
springdoc.swagger-ui.urls[0].url=/aggregate/product-service/v3/api-docs
springdoc.swagger-ui.urls[1].name=Order Service
springdoc.swagger-ui.urls[1].url=/aggregate/order-service/v3/api-docs
springdoc.swagger-ui.urls[2].name=Inventory Service
springdoc.swagger-ui.urls[2].url=/aggregate/inventory-service/v3/api-docs
```

## Order Service Files

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
    //    implementation("org.springframework.cloud:spring-cloud-starter-contract-stub-runner")
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
    implementation("org.springframework.cloud:spring-cloud-starter-contract-stub-runner")
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j:3.3.0")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### File 7: order-service/src/main/java/ca/gbc/orderservice/client/InventoryClient.java

```java
package ca.gbc.orderservice.client;

import io.github.resilience4j.circuitbreaker.CallNotPermittedException;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.service.annotation.GetExchange;

public interface InventoryClient {
    Logger log = LoggerFactory.getLogger(InventoryClient.class);

    @GetExchange("/api/inventory")
    @CircuitBreaker(name = "inventory", fallbackMethod = "fallbackMethod")
    @Retry(name = "inventory")
    boolean isInStock(@RequestParam String skuCode, @RequestParam Integer quantity);

    default boolean fallbackMethod(String skuCode, Integer quantity, Throwable throwable) {
        if (throwable instanceof CallNotPermittedException) {
            log.warn("Circuit breaker is open for SKU code {}: {}", skuCode, throwable.getMessage());
        } else {
            log.warn("Fallback triggered for SKU code {}, reason: {}", skuCode, throwable.getMessage());
        }
        return false;
    }
}
```

### File 8: order-service/src/main/java/ca/gbc/orderservice/config/RestClientConfig.java

```java
package ca.gbc.orderservice.config;

import ca.gbc.orderservice.client.InventoryClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.JdkClientHttpRequestFactory;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.support.RestClientAdapter;
import org.springframework.web.service.invoker.HttpServiceProxyFactory;

import java.net.http.HttpClient;
import java.time.Duration;

@Configuration
public class RestClientConfig {

    private static final Logger log = LoggerFactory.getLogger(RestClientConfig.class);

    @Value("${inventory.service.url}")
    private String inventoryServiceUrl;

    /**
     * ✅ Week 9 - Day 1
     * Creates and configures an InventoryClient bean for interaction with the Inventory Service.
     * <p>
     * The method sets up a RestClient with the specified base URL and timeouts,
     * wraps it in a RestClientAdapter, and uses an HttpServiceProxyFactory to create
     * a proxy implementation of the InventoryClient interface.
     * </p>
     *
     * @return InventoryClient - a proxy instance configured to communicate with the Inventory Service.
     */
    @Bean
    public InventoryClient inventoryClient() {
        //✅ Week 9 - Day 1
        // Configure HttpClient with timeouts
        HttpClient httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(5)) // 5 seconds for connection
                .build();

        //✅ Week 9 - Day 1
        // Create JdkClientHttpRequestFactory with the configured HttpClient
        JdkClientHttpRequestFactory requestFactory = new JdkClientHttpRequestFactory(httpClient);
        // Set read timeout (applies to response body reading)
        requestFactory.setReadTimeout(Duration.ofSeconds(5)); // 5 seconds for reading response

        // Create RestClient with base URL and request factory
        RestClient restClient = RestClient.builder()
                .baseUrl(inventoryServiceUrl)
                .requestFactory(requestFactory)
                .build();

        // Wrap the RestClient in a RestClientAdapter
        var restClientAdapter = RestClientAdapter.create(restClient);

        // Create HttpServiceProxyFactory for InventoryClient
        var httpServiceProxyFactory = HttpServiceProxyFactory.builderFor(restClientAdapter).build();

        return httpServiceProxyFactory.createClient(InventoryClient.class);
    }
}
```

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
spring.datasource.url=jdbc:postgresql://localhost:5432/order-service
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
management.health.circuitbreakers.enabled=true
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
```

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

# Week 13 - Circuit Breaker Configuration
management.health.circuitbreakers.enabled=true
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
```

### File 11: order-service/src/main/resources/db/migration/V1__init.sql

```sql
CREATE TABLE t_orders (
    id BIGSERIAL NOT NULL,
    order_number VARCHAR(255) DEFAULT NULL,
    sku_code VARCHAR(255),
    price DECIMAL(19, 2),
    quantity INT,
    PRIMARY KEY (id)
);
```

## Inventory Service Files

### File 12: inventory-service/build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.6"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "ca.gbc.comp3095"
version = "0.0.1-SNAPSHOT"
description = "inventory-service"

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


    // Week 12 - Swagger/OpenAPI Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")
}


tasks.withType<Test> {
    useJUnitPlatform()
}
```

### File 13: inventory-service/src/main/resources/application.properties

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

# Week 12 - API version for documentation
inventory-service.version=v1.0

# Week 12 - Swagger Documentation
# Swagger UI accessible at: http://localhost:8083/swagger-ui
springdoc.swagger-ui.path=/swagger-ui
# OpenAPI JSON accessible at: http://localhost:8083/api-docs
springdoc.api-docs.path=/api-docs
```

### File 14: inventory-service/src/main/resources/application-docker.properties

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

# Week 12 - API version for documentation
inventory-service.version=v1.0

# Week 12 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs

spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect

# Week 13
spring.flyway.baseline-on-migrate=true
spring.flyway.locations=classpath:db/migration
spring.flyway.enabled=true
```

### File 15: inventory-service/src/main/resources/db/migration/V1__init.sql

```sql
CREATE TABLE t_inventory (
    id BIGSERIAL PRIMARY KEY,
    sku_code VARCHAR(255),
    quantity INTEGER
);
```

### File 16: inventory-service/src/main/resources/db/migration/V2__add_inventory.sql

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

## Product Service Files

### File 17: product-service/build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.5.6"
    id("io.spring.dependency-management") version "1.1.7"
}

group = "ca.gbc.comp3095"
version = "0.0.1-SNAPSHOT"
description = "product-service"

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
    // TestContainers Bill-of-Materials (BOM)
    testImplementation(platform("org.testcontainers:testcontainers-bom:1.21.3"))

    // Spring Boot Starters
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")

    // Lombok
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    // Spring Boot DevTools
    developmentOnly("org.springframework.boot:spring-boot-devtools")

    // Spring Boot Testing Starters
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-testcontainers")

    // TestContainers Modules
    testImplementation("org.testcontainers:mongodb")
    testImplementation("org.testcontainers:junit-jupiter")
    //testImplementation("org.testcontainers:redis")

    // REST Assured for testing
    testImplementation("io.rest-assured:rest-assured")

    // Test Platform Launcher
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")

    // Week 12 - Swagger/OpenAPI Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.8")
    testImplementation("org.springdoc:springdoc-openapi-starter-webmvc-api:2.8.8")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### File 18: product-service/src/main/resources/application.properties

```properties
# ============================================
# Product Service - Local Configuration
# Use this when running from IntelliJ IDE
# ============================================

# Application name
spring.application.name=product-service

# Server configuration
server.port=8084

# MongoDB configuration for LOCAL development
# Connects to MongoDB on localhost (your machine)
spring.data.mongodb.host=localhost
spring.data.mongodb.port=27017
spring.data.mongodb.database=product-service

# Note: No authentication for local MongoDB
# This assumes you have MongoDB running locally without auth
# If you have auth enabled locally, add:
# spring.data.mongodb.username=admin
# spring.data.mongodb.password=password
# spring.data.mongodb.authentication-database=admin

# Logging configuration
logging.level.ca.gbc.comp3095=INFO
logging.level.org.springframework.data.mongodb=DEBUG


# Redis Configuration
spring.cache.type=redis
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.data.redis.password=password
spring.cache.redis.time-to-live=60s

# Week 12 - API version for documentation
product-service.version=v1.0

# Week 12 - Swagger Documentation
# Swagger UI accessible at: http://localhost:8084/swagger-ui
springdoc.swagger-ui.path=/swagger-ui
# OpenAPI JSON accessible at: http://localhost:8084/api-docs
springdoc.api-docs.path=/api-docs
```

### File 19: product-service/src/main/resources/application-docker.properties

```properties
# ============================================
# Product Service - Docker Configuration
# Activated when SPRING_PROFILES_ACTIVE=docker
# ============================================

# Application name
spring.application.name=product-service

# Server configuration
server.port=8084

# MongoDB configuration for DOCKER environment
# Important: Use service name "mongodb" instead of "localhost"
spring.data.mongodb.host=mongodb
spring.data.mongodb.port=27017
spring.data.mongodb.database=product-service

# MongoDB authentication
# Credentials must match those in docker-compose.yml
spring.data.mongodb.username=admin
spring.data.mongodb.password=password
spring.data.mongodb.authentication-database=admin

# Logging configuration
logging.level.ca.gbc.comp3095=INFO
logging.level.org.springframework.data.mongodb=DEBUG

# Redis Configuration
spring.data.redis.host=redis
spring.data.redis.port=6379
spring.data.redis.password=password
spring.cache.type=redis
spring.cache.redis.time-to-live=60s

# Week 12 - API version for documentation
product-service.version=v1.0

# Week 12 - Swagger Documentation
springdoc.swagger-ui.path=/swagger-ui
springdoc.api-docs.path=/api-docs
```

### File 20: product-service/src/main/java/ca/gbc/comp3095/productservice/config/OpenAPIConfig.java

```java
package ca.gbc.comp3095.productservice.config;

import io.swagger.v3.oas.models.servers.Server;
import java.util.List;
import io.swagger.v3.oas.models.ExternalDocumentation;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenAPIConfig {

    @Value("${product-service.version}")
    private String version;

    @Bean
    public OpenAPI productServiceAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Product Service API")
                        .description("This is the REST API for Product Service")
                        .version(version)
                        .license(new License().name("Apache 2.0")))
                .externalDocs(new ExternalDocumentation()
                        .description("Product Service Confluence Documentation")
                        .url("https://mycompany.ca/product-service/docs"))
                .servers(List.of(
                        new Server()
                                .url("http://localhost:9000")
                                .description("API Gateway")
                ));
    }
}
```

### File 21: order-service/src/main/java/ca/gbc/comp3095/orderservice/config/OpenAPIConfig.java

```java
package ca.gbc.comp3095.orderservice.config;

import io.swagger.v3.oas.models.servers.Server;
import java.util.List;
import io.swagger.v3.oas.models.ExternalDocumentation;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration
public class OpenAPIConfig {

    @Value("${order-service.version}")
    private String version;

    @Bean
    public OpenAPI orderServiceAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Order Service API")
                        .description("This is the REST API for Order Service")
                        .version(version)
                        .license(new License().name("Apache 2.0")))
                .externalDocs(new ExternalDocumentation()
                        .description("Order Service Confluence Documentation")
                        .url("https://mycompany.ca/order-service/docs"))
                .servers(List.of(
                        new Server()
                                .url("http://localhost:9000")
                                .description("API Gateway")
                ));
    }
}
```

### File 22: inventory-service/src/main/java/ca/gbc/comp3095/inventoryservice/config/OpenAPIConfig.java

```java
package ca.gbc.comp3095.inventoryservice.config;

import io.swagger.v3.oas.models.ExternalDocumentation;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import io.swagger.v3.oas.models.servers.Server;
import java.util.List;

@Configuration
public class OpenAPIConfig {

    @Value("${inventory-service.version}")
    private String version;

    @Bean
    public OpenAPI inventoryServiceAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Inventory Service API")
                        .description("This is the REST API for Inventory Service")
                        .version(version)
                        .license(new License().name("Apache 2.0")))
                .externalDocs(new ExternalDocumentation()
                        .description("Inventory Service Confluence Documentation")
                        .url("https://mycompany.ca/inventory-service/docs"))
                .servers(List.of(
                        new Server()
                                .url("http://localhost:9000")
                                .description("API Gateway")
                ));
    }
}
```
