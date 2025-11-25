# Week 13 - Part 3: Circuit Breaker Pattern with Resilience4j

## Overview

Implement fault tolerance in microservices using Circuit Breaker Pattern with Resilience4j library. This pattern prevents cascading failures when downstream services become unavailable or slow.

## Circuit Breaker Pattern

The Circuit Breaker Pattern protects applications from failures by wrapping calls to external services. It monitors for failures and temporarily blocks requests when a threshold is exceeded.

**States:**
- **CLOSED**: Normal operation. Requests pass through. Failures are counted.
- **OPEN**: Circuit breaker is triggered. All requests fail immediately without calling the service.
- **HALF_OPEN**: After a wait period, limited requests are allowed to test if the service recovered.

**Why?**
Synchronous communication between microservices can cause cascading failures. When a downstream service fails or becomes slow, upstream services can become blocked waiting for responses, exhausting thread pools and causing system-wide failures.

---

## Step 1: Add Resilience4j Dependencies

### 1.1 Update API Gateway Dependencies

**Location:** `microservices-parent/api-gateway/build.gradle.kts`

Add the circuit breaker dependency:

```kotlin
implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j:3.3.0")
```

**Complete dependencies section:**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.cloud:spring-cloud-starter-gateway-mvc:4.2.1")
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j:3.3.0")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

### 1.2 Update Order Service Dependencies

**Location:** `microservices-parent/order-service/build.gradle.kts`

Add the circuit breaker dependency:

```kotlin
implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j:3.3.0")
```

**Complete dependencies section:**

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.7.0")
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j:3.3.0")
    implementation("org.springframework.cloud:spring-cloud-starter-contract-stub-runner")
    compileOnly("org.projectlombok:lombok")
    runtimeOnly("org.postgresql:postgresql")
    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-testcontainers")
    testImplementation("io.rest-assured:rest-assured")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("org.testcontainers:postgresql")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

---

## Step 2: Configure Spring Actuator

### 2.1 Enable Circuit Breaker Health Indicators in API Gateway

**Location:** `microservices-parent/api-gateway/src/main/resources/application.properties`

Add the following configuration:

```properties
management.health.circuitbreakers.enabled=true
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
```

**Why?**
Spring Actuator exposes circuit breaker metrics and health information through HTTP endpoints. This enables monitoring of circuit breaker states and statistics.

### 2.2 Enable Circuit Breaker Health Indicators in Order Service

**Location:** `microservices-parent/order-service/src/main/resources/application.properties`

Add the following configuration:

```properties
management.health.circuitbreakers.enabled=true
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
```

---

## Step 3: Configure Resilience4j in API Gateway

### 3.1 Add Default Circuit Breaker Configuration

**Location:** `microservices-parent/api-gateway/src/main/resources/application.properties`

Add the following configuration:

```properties
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

**How It Works:**
- `slidingWindowSize=10`: Tracks last 10 calls
- `failureRateThreshold=50`: Opens circuit if 50% of calls fail
- `minimum-number-of-calls=5`: Requires at least 5 calls before calculating failure rate
- `waitDurationInOpenState=5s`: Circuit stays open for 5 seconds
- `permittedNumberOfCallsInHalfOpenState=3`: Allows 3 test calls in half-open state
- `timeout-duration=3s`: Request times out after 3 seconds
- `max-attempts=3`: Retries failed requests up to 3 times
- `wait-duration=2s`: Waits 2 seconds between retries

---

## Step 4: Implement Circuit Breaker Routes in API Gateway

### 4.1 Update Routes Class

**Location:** `microservices-parent/api-gateway/src/main/java/ca/gbc/apigateway/routes/Routes.java`

Replace the entire file with the following code:

```java
package ca.gbc.apigateway.routes;

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

**Key Changes:**
- Added `CircuitBreakerFilterFunctions.circuitBreaker()` filter to all 6 routes
- Each circuit breaker has unique ID (productServiceCircuitBreaker, orderServiceCircuitBreaker, etc.)
- All circuit breakers forward to `/fallbackRoute` when open
- Added `fallbackRoute()` bean that returns HTTP 503 with message

**Why?**
The fallback route provides graceful degradation when downstream services are unavailable. Each route has its own circuit breaker instance to isolate failures.

---

## Step 5: Configure Resilience4j in Order Service

### 5.1 Add Instance-Specific Configuration

**Location:** `microservices-parent/order-service/src/main/resources/application.properties`

Add the following configuration:

```properties
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

**Why?**
Instance-specific configuration for the `inventory` circuit breaker used in InventoryClient. This allows fine-tuning parameters for specific service interactions.

---

## Step 6: Implement Circuit Breaker in Inventory Client

### 6.1 Update InventoryClient Interface

**Location:** `microservices-parent/order-service/src/main/java/ca/gbc/orderservice/client/InventoryClient.java`

Replace the entire file with the following code:

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

**How It Works:**
- `@CircuitBreaker(name = "inventory")`: Wraps the method with circuit breaker named `inventory`
- `fallbackMethod = "fallbackMethod"`: Calls `fallbackMethod` when circuit breaker is open or call fails
- `@Retry(name = "inventory")`: Automatically retries failed calls before triggering fallback
- `CallNotPermittedException`: Thrown when circuit breaker is open
- Returns `false` from fallback to indicate item is not in stock when inventory service is unavailable

---

## Step 7: Configure HTTP Client Timeouts

### 7.1 Update RestClientConfig

**Location:** `microservices-parent/order-service/src/main/java/ca/gbc/orderservice/config/RestClientConfig.java`

Replace the entire file with the following code:

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

**Key Changes:**
- Added `HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(5))` for connection timeout
- Added `JdkClientHttpRequestFactory` with `setReadTimeout(Duration.ofSeconds(5))` for read timeout
- RestClient now uses configured `requestFactory` with timeout settings

**Why?**
Timeouts prevent the application from waiting indefinitely for unresponsive services. The `connectTimeout` limits connection establishment time, while `readTimeout` limits response reading time.

---

## Step 8: Testing Circuit Breaker

### 8.1 Test Scenario 1: Service Down

**Steps:**
1. Start all services using Docker Compose
2. Stop the inventory-service container: `docker stop inventory-service`
3. Place an order through API Gateway: `POST http://localhost:8080/api/order`
4. Observe the fallback behavior in order-service logs
5. Check circuit breaker state: `GET http://localhost:8082/actuator/health`

**Expected Result:**
- Order placement fails gracefully
- Fallback method logs warning message
- Circuit breaker opens after failure threshold is reached
- Subsequent requests fail immediately without calling inventory-service

### 8.2 Test Scenario 2: Service Timeout

**Steps:**
1. Add artificial delay in inventory-service to exceed timeout (3 seconds)
2. Place an order through API Gateway
3. Observe timeout exception in order-service logs
4. Verify retry attempts in logs (3 attempts with 2 second intervals)
5. Check circuit breaker state

**Expected Result:**
- Request times out after 3 seconds
- Retry mechanism attempts 3 times
- Fallback method is called after all retries fail
- Circuit breaker opens after multiple timeout failures

### 8.3 Test Scenario 3: Service Recovery

**Steps:**
1. With circuit breaker in OPEN state, wait 5 seconds
2. Restart inventory-service: `docker start inventory-service`
3. Circuit breaker transitions to HALF_OPEN automatically
4. Place an order (one of the 3 permitted test calls)
5. Verify successful order placement
6. Place 2 more orders
7. Check circuit breaker state

**Expected Result:**
- Circuit breaker transitions to HALF_OPEN after 5 seconds
- First 3 requests are allowed through
- If all 3 succeed, circuit breaker transitions to CLOSED
- Normal operation resumes

### 8.4 Test Scenario 4: API Gateway Circuit Breaker

**Steps:**
1. Stop inventory-service container
2. Call inventory-service directly through API Gateway: `GET http://localhost:8080/api/inventory?skuCode=SKU001&quantity=10`
3. Repeat multiple times to trigger circuit breaker
4. Observe fallback response from API Gateway
5. Check circuit breaker metrics: `GET http://localhost:8080/actuator/circuitbreakers`

**Expected Result:**
- First few requests attempt to connect to inventory-service
- After failure threshold, API Gateway circuit breaker opens
- Subsequent requests return "Service Unavailable, please try again in a few minutes"
- Response is immediate without attempting connection

---

## Step 9: Monitor Circuit Breaker with Actuator

### 9.1 Health Endpoint

**URL:** `http://localhost:8082/actuator/health`

**Response:**
```json
{
  "status": "UP",
  "components": {
    "circuitBreakers": {
      "status": "UP",
      "details": {
        "inventory": {
          "status": "UP",
          "details": {
            "failureRate": "0.0%",
            "slowCallRate": "0.0%",
            "failureRateThreshold": "50.0%",
            "slowCallRateThreshold": "100.0%",
            "bufferedCalls": 10,
            "failedCalls": 0,
            "slowCalls": 0,
            "slowFailedCalls": 0,
            "notPermittedCalls": 0,
            "state": "CLOSED"
          }
        }
      }
    }
  }
}
```

### 9.2 Circuit Breakers Endpoint

**URL:** `http://localhost:8082/actuator/circuitbreakers`

**Response:**
```json
{
  "circuitBreakers": {
    "inventory": {
      "failureRate": "0.0%",
      "slowCallRate": "0.0%",
      "failureRateThreshold": "50.0%",
      "slowCallRateThreshold": "100.0%",
      "bufferedCalls": 10,
      "failedCalls": 0,
      "slowCalls": 0,
      "notPermittedCalls": 0,
      "state": "CLOSED"
    }
  }
}
```

### 9.3 Circuit Breaker Events Endpoint

**URL:** `http://localhost:8082/actuator/circuitbreakerevents`

**Response:**
```json
{
  "circuitBreakerEvents": [
    {
      "circuitBreakerName": "inventory",
      "type": "STATE_TRANSITION",
      "creationTime": "2025-11-25T10:30:00.000Z",
      "stateTransition": "CLOSED_TO_OPEN",
      "errorMessage": "java.net.ConnectException: Connection refused"
    }
  ]
}
```

---

## Step 10: Troubleshooting

### 10.1 Circuit Breaker Not Triggering

**Problem:** Circuit breaker remains CLOSED despite service failures

**Solution:**
- Verify `minimum-number-of-calls` threshold is met (at least 5 calls)
- Check `slidingWindowSize` matches expected call volume
- Confirm failure rate exceeds `failureRateThreshold` (50%)
- Review logs for exceptions being caught

### 10.2 Fallback Method Not Called

**Problem:** Fallback method signature does not match

**Solution:**
- Ensure fallback method parameters match original method parameters
- Add `Throwable` parameter as last argument
- Return type must match original method return type
- Method must be accessible (public or default in interface)

### 10.3 Timeouts Not Working

**Problem:** Requests wait longer than configured timeout

**Solution:**
- Verify `resilience4j.timelimiter.instances.inventory.timeout-duration=3s` is set
- Confirm `JdkClientHttpRequestFactory` has both `connectTimeout` and `readTimeout` configured
- Check HTTP client is using configured request factory
- Ensure timeout values are reasonable for service response times

### 10.4 Circuit Breaker Opens Too Quickly

**Problem:** Circuit breaker opens with few failures

**Solution:**
- Increase `minimum-number-of-calls` (default: 5)
- Increase `slidingWindowSize` (default: 10)
- Increase `failureRateThreshold` (default: 50%)
- Consider using TIME_BASED sliding window instead of COUNT_BASED

---

## Summary

Circuit Breaker Pattern implementation protects microservices from cascading failures:

**API Gateway Level:**
- Fallback route provides graceful degradation
- Circuit breaker filters protect all downstream services
- Default configuration applies to all circuit breakers
- Monitors and fails fast when services are unavailable

**Order Service Level:**
- `@CircuitBreaker` annotation protects inventory client calls
- `@Retry` annotation provides automatic retry mechanism
- Fallback method returns safe default when inventory unavailable
- Instance-specific configuration fine-tunes behavior
- HTTP client timeouts prevent indefinite waits

**Monitoring:**
- Spring Actuator exposes circuit breaker state and metrics
- Health endpoint shows current state (CLOSED, OPEN, HALF_OPEN)
- Events endpoint tracks state transitions and failures
- Real-time monitoring enables proactive issue detection

**Testing:**
- Simulate service failures by stopping containers
- Verify retry attempts and fallback behavior
- Monitor state transitions through actuator endpoints
- Confirm automatic recovery when services restore
