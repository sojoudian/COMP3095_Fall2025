# Week 13 Fixes

## File 1: api-gateway/src/main/java/ca/gbc/comp3095/apigateway/routes/Routes.java

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

## File 2: api-gateway/src/main/java/ca/gbc/comp3095/apigateway/config/SecurityConfig.java

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

## File 3: api-gateway/src/main/resources/application-docker.properties

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
