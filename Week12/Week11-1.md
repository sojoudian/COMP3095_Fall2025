# Keycloak Security Integration

## Overview

Integrate Keycloak authentication into your microservices architecture. This lab secures all API Gateway endpoints using OAuth2 Client Credentials flow with JWT tokens.

**Time:** 90-120 minutes

---

## Prerequisites

- ✅ Completed Week 11 (API Gateway Implementation)
- ✅ Docker Desktop running
- ✅ All microservices operational (product-service, order-service, inventory-service, api-gateway)
- ✅ Postman installed

---

## Background

Currently, all microservices endpoints are publicly accessible without authentication. Keycloak is an open-source Identity and Access Management (IAM) solution that provides centralized authentication and authorization using OAuth2 and OpenID Connect protocols.

### OAuth2 Client Credentials Flow

This lab uses the **Client Credentials** grant type for service-to-service authentication:

- API Gateway authenticates with Keycloak using **client ID** and **client secret**
- Keycloak issues a **JWT access token**
- Token is sent via `Authorization: Bearer <token>` header to protected endpoints

**Use Case:** Backend service-to-service communication without user involvement

**Not For:** User login scenarios (use Authorization Code Flow instead)

---

## Step 1: Add Keycloak to docker-compose.yml

### 1.1 Update Main docker-compose.yml

**Location:** `microservices-parent/docker-compose.yml`

Before showing the complete docker-compose.yml, here are the Keycloak-related services to add:

```yaml
services:
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
      - postgres-keycloak
    container_name: keycloak
    networks:
      - spring

  postgres-keycloak:
    image: postgres:15
    volumes:
      - ./docker/integrated/keycloak/db-data:/var/lib/postgresql/data
    ports:
      - "5434:5432"
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: password
    container_name: postgres-keycloak
    networks:
      - spring
```

Complete docker-compose.yml configuration:

```yaml
services:

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
      - postgres-keycloak
    container_name: keycloak
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
    networks:
      - spring

  redis:
    image: redis:latest
    ports:
      - "6379:6379"
    volumes:
      - ./docker/integrated/redis/init/redis.conf:/usr/local/etc/redis/redis.conf
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    container_name: redis
    networks:
      - spring

  redis-insight:
    image: redislabs/redisinsight:1.14.0
    ports:
      - "8001:8001"
    container_name: redis-insight
    depends_on:
      - redis
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
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-inventory/inventory-service
      - SPRING_DATASOURCE_USERNAME=admin
      - SPRING_DATASOURCE_PASSWORD=password
      - SPRING_JPA_HIBERNATE_DDL_AUTO=none
      - SPRING_PROFILES_ACTIVE=docker
    depends_on:
      - postgres-inventory
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
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-order/order-service
      - SPRING_DATASOURCE_USERNAME=admin
      - SPRING_DATASOURCE_PASSWORD=password
      - SPRING_JPA_HIBERNATE_DDL_AUTO=none
      - SPRING_PROFILES_ACTIVE=docker
    depends_on:
      - postgres-order
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
      - mongodb
    networks:
      - spring

  mongodb:
    image: mongo:latest
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
    volumes:
      - ./docker/integrated/mongo/data/mongo/products:/data/db
      - ./docker/integrated/mongo/init/mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js
    container_name: mongodb
    command: mongod --auth
    networks:
      - spring

  mongo-express:
    image: mongo-express
    ports:
      - "8081:8081"
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=admin
      - ME_CONFIG_MONGODB_ADMINPASSWORD=password
      - ME_CONFIG_MONGODB_SERVER=mongodb
      - ME_CONFIG_MONGODB_AUTH_DATABASE=admin
    container_name: mongo-express
    depends_on:
      - mongodb
    networks:
      - spring

  postgres-keycloak:
    image: postgres:15
    volumes:
      - ./docker/integrated/keycloak/db-data:/var/lib/postgresql/data
    ports:
      - "5434:5432"
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: password
    container_name: postgres-keycloak
    networks:
      - spring

  postgres-inventory:
    container_name: postgres-inventory
    image: postgres
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
    networks:
      - spring

  postgres-order:
    container_name: postgres-order
    image: postgres
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
    networks:
      - spring

  pgpadmin:
    image: dpage/pgadmin4:9.2
    ports:
      - "8888:80"
    environment:
      - PGADMIN_DEFAULT_EMAIL=user@domain.ca
      - PGADMIN_DEFAULT_PASSWORD=password
    container_name: pgadmin
    networks:
      - spring

volumes:
  mongo-db:
    driver: local

networks:
  spring:
    driver: bridge
```

### 1.2 Create Directory Structure

```bash
cd microservices-parent
mkdir -p docker/integrated/keycloak/realms
mkdir -p docker/standalone/keycloak/realms
```

### 1.3 Start Keycloak Standalone (Initial Setup)

For initial configuration, use standalone Keycloak:

**Location:** `docker/standalone/keycloak/docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:15
    volumes:
      - ./db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: password
    container_name: local-postgres-keycloak

  keycloak:
    image: quay.io/keycloak/keycloak:24.0.1
    command: start-dev
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: password
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: password
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    container_name: local-keycloak
```

Start standalone Keycloak:

```bash
# Stop integrated docker-compose if running (to avoid port conflicts)
cd microservices-parent
docker-compose -p microservices-comp3095 down

# Start standalone Keycloak
cd docker/standalone/keycloak
docker-compose -p keycloak-standalone up -d
```

---

## Step 2: Configure Keycloak Realm and Client

### 2.1 Access Keycloak Admin Console

1. Navigate to `http://localhost:8080`
2. Click **Administration Console**
3. Login with:
   - Username: `admin`
   - Password: `password`

### 2.2 Create Realm

1. Click dropdown **Master** (top-left)
2. Click **Add realm**
3. Enter Realm name: `spring-microservices-security-realm`
4. Click **Create**

### 2.3 Create OAuth2 Client

1. Click **Clients** (left sidebar)
2. Click **Create Client**

**Client Settings (Step 1):**
- Client Type: `OpenID Connect`
- Client ID: `spring-client-credentials-id`
- Click **Next**

**Client Settings (Step 2):**
- Client Authentication: **ON**
- Standard Flow: **OFF**
- Direct Access Grants: **OFF**
- Service Accounts: **ON**
- Click **Next**

**Client Settings (Step 3):**
- Root URL: (leave blank)
- Home URL: (leave blank)
- Click **Save**

### 2.4 Obtain Client Secret

1. Click **spring-client-credentials-id** client
2. Click **Credentials** tab
3. **Copy** the Client Secret value (you'll need this for Postman)

### 2.5 Get Issuer URI

The issuer URI follows this pattern:

```
http://localhost:8080/realms/spring-microservices-security-realm
```

---

## Step 3: Configure API Gateway Security

### 3.1 Add OAuth2 Dependency

**Location:** `api-gateway/build.gradle.kts`

Add dependencies:

```kotlin
dependencies {
    // Existing dependencies...

    // Week 5.2 - Security
    compileOnly("jakarta.servlet:jakarta.servlet-api:6.1.0")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("org.springframework.boot:spring-boot-starter-security")
}
```

### 3.2 Update application.properties

**Location:** `api-gateway/src/main/resources/application.properties`

Add issuer URI configuration:

```properties
spring.application.name=api-gateway

server.port=9000

#services
services.product-url=http://localhost:8084
services.order-url=http://localhost:8082

# Week 5.2 - Keycloak Security
# The API Gateway uses Keycloak's public key to validate JWT tokens locally
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:8080/realms/spring-microservices-security-realm
```

### 3.3 Update application-docker.properties

**Location:** `api-gateway/src/main/resources/application-docker.properties`

```properties
spring.application.name=api-gateway

server.port=9000

#services
services.product-url=http://product-service:8084
services.order-url=http://order-service:8082

# Week 5.2 - Keycloak Security (Docker)
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://keycloak:8080/realms/spring-microservices-security-realm
```

### 3.4 Create SecurityConfig Class

Create new package:

1. Right-click on `ca.gbc.apigateway`
2. Select **New → Package**
3. Name: `config`
4. Click **OK**

Create SecurityConfig:

1. Right-click on `config` package
2. Select **New → Java Class**
3. Name: `SecurityConfig`
4. Click **OK**

**Location:** `api-gateway/src/main/java/ca/gbc/apigateway/config/SecurityConfig.java`

```java
package ca.gbc.apigateway.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.web.SecurityFilterChain;

@Slf4j
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    /**
     * Configures the security filter chain for HTTP requests.
     * All requests require authentication and JWT token validation via Keycloak.
     */
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {

        log.info("Initializing Security Filter Chain...");

        return httpSecurity
                .csrf(AbstractHttpConfigurer::disable)
                // Require authentication for all requests
                .authorizeHttpRequests(authorize -> authorize
                        .anyRequest().authenticated())
                // Configure OAuth2 resource server with JWT validation
                .oauth2ResourceServer(oauth2 -> oauth2
                        .jwt(Customizer.withDefaults()))
                .build();
    }

}
```

### 3.5 Create RequestLoggingFilter (Optional)

Create filter class:

1. Right-click on `config` package
2. Select **New → Java Class**
3. Name: `RequestLoggingFilter`
4. Click **OK**

**Location:** `api-gateway/src/main/java/ca/gbc/apigateway/config/RequestLoggingFilter.java`

```java
package ca.gbc.apigateway.config;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Slf4j
@Component
public class RequestLoggingFilter implements Filter {

    /**
     * Logs incoming HTTP requests (method, URI, remote address)
     */
    @Override
    public void doFilter(ServletRequest request, ServletResponse servletResponse, FilterChain filterChain)
            throws IOException, ServletException {

        HttpServletRequest httpServletRequest = (HttpServletRequest) request;

        log.info("Incoming request: Method = {}, URI = {}, Remote Address = {}",
                httpServletRequest.getMethod(),
                httpServletRequest.getRequestURI(),
                httpServletRequest.getRemoteAddr());

        filterChain.doFilter(request, servletResponse);
    }

    @Override
    public void destroy() {
        Filter.super.destroy();
        log.info("Destroying RequestLoggingFilter...");
    }
}
```

### 3.6 Create FilterConfig (Optional)

Create configuration class:

1. Right-click on `config` package
2. Select **New → Java Class**
3. Name: `FilterConfig`
4. Click **OK**

**Location:** `api-gateway/src/main/java/ca/gbc/apigateway/config/FilterConfig.java`

```java
package ca.gbc.apigateway.config;

import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;

/**
 * Registers custom filters for the Spring Boot application
 */
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<RequestLoggingFilter> loggingFilter() {
        FilterRegistrationBean<RequestLoggingFilter> registrationBean = new FilterRegistrationBean<>();

        registrationBean.setFilter(new RequestLoggingFilter());
        registrationBean.addUrlPatterns("/*");

        return registrationBean;
    }
}
```

---

## Step 4: Export Keycloak Realm Configuration

### 4.1 Export Realm from Keycloak UI

1. Login to Keycloak Admin Console
2. Select **spring-microservices-security-realm**
3. Click **Realm settings** (left sidebar)
4. Click **Action** dropdown (top-right)
5. Select **Partial export**
6. Check **Export clients**
7. Click **Export**
8. Save as `realm-export.json`

### 4.2 Store Realm Files

Copy `realm-export.json` to both locations:

```bash
# Integrated (for docker-compose)
cp realm-export.json microservices-parent/docker/integrated/keycloak/realms/

# Standalone (for local development)
cp realm-export.json microservices-parent/docker/standalone/keycloak/realms/
```

### 4.3 Update .gitignore

**Location:** `microservices-parent/.gitignore`

Add:

```
# Docker data directories
docker/standalone/mongo-redis/db-data/
docker/standalone/postgres/inventory-service/db-data/
docker/integrated/mongo/data/
docker/integrated/postgres/data/
docker/standalone/postgres/order-service/db-data/
docker/standalone/keycloak/db-data/
```

---

## Step 5: Test Security Configuration

### 5.1 Rebuild and Start Services

Stop standalone Keycloak:

```bash
cd docker/standalone/keycloak
docker-compose down
```

Rebuild all services:

```bash
cd microservices-parent
docker-compose -p microservices-comp3095 -f docker-compose.yml up -d --build
```

Wait ~30 seconds for all services to stabilize.

### 5.2 Test Without Authentication

Open Postman and attempt:

```
GET http://localhost:9000/api/product
```

**Expected Result:** `401 Unauthorized`

This confirms security is working.

### 5.3 Configure Postman OAuth2

1. In Postman, click **Authorization** tab
2. Type: Select **OAuth 2.0**
3. Click **Configure New Token**

**Token Configuration:**
- Token Name: `Token`
- Grant Type: `Client Credentials`
- Access Token URL: `http://localhost:8080/realms/spring-microservices-security-realm/protocol/openid-connect/token`
- Client ID: `spring-client-credentials-id`
- Client Secret: `<paste your client secret from Keycloak>`
- Scope: (leave blank)
- Client Authentication: `Send as Basic Auth header`

4. Click **Get New Access Token**
5. Click **Use Token**

### 5.4 Test With Authentication

Make request again:

```
GET http://localhost:9000/api/product
```

**Expected Result:** `200 OK` with product data

### 5.5 Test All Endpoints

Test the following endpoints (update token for each request):

- `GET http://localhost:9000/api/product`
- `POST http://localhost:9000/api/product`
- `GET http://localhost:9000/api/order`
- `POST http://localhost:9000/api/order`

**Note:** JWT tokens typically expire after 5 minutes. Click **Get New Access Token** if you receive `401 Unauthorized`.

---

## Step 6: Update Redis Configuration (Cache TTL)

### 6.1 Update product-service Redis Config

**Location:** `product-service/src/main/resources/application.properties`

Add Redis cache expiration:

```properties
# Existing Redis config
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.data.redis.password=password
spring.cache.type=redis

# Week 5.2 - Add cache expiration
spring.cache.redis.time-to-live=60s
```

### 6.2 Update Standalone Redis Config

**Location:** `docker/standalone/redis/redis.conf`

```conf
requirepass password
```

### 6.3 Update Integrated Redis Config

**Location:** `docker/integrated/redis/init/redis.conf`

Remove persistence settings and keep only:

```conf
requirepass password
```

---

## Step 7: Standalone Keycloak Configuration

### 7.1 Standalone Docker Compose

**Location:** `docker/standalone/keycloak/docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:15
    volumes:
      - ./db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: password
    container_name: local-postgres-keycloak

  keycloak:
    image: quay.io/keycloak/keycloak:24.0.1
    command: start-dev
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: password
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: password
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    container_name: local-keycloak
```

**Usage:**

```bash
cd docker/standalone/keycloak
docker-compose -p keycloak-standalone up -d
```

**Access:**
- Keycloak Admin: `http://localhost:8080`
- Login: `admin` / `password`

**Stop:**

```bash
docker-compose down
```

---

## Step 8: Understanding OAuth2 Flows

### Client Credentials Flow (This Lab)

Used for **service-to-service** authentication:

1. Client sends `client_id` + `client_secret` to Keycloak
2. Keycloak returns JWT access token
3. Client uses token in `Authorization: Bearer <token>` header

**When to Use:**
- Backend microservices communication
- No user interaction required

### Authorization Code Flow (Future Labs)

Used for **user authentication**:

1. User redirected to Keycloak login page
2. User enters username/password
3. Keycloak redirects with authorization code
4. Client exchanges code for access token
5. Token used for API requests with user context

**When to Use:**
- Web applications with user login
- Mobile apps
- SPAs (Single Page Applications)

---

## Troubleshooting

### Port Conflicts

If port 8080 is in use:

```bash
# Check what's using port 8080
lsof -i :8080

# Kill the process
kill -9 <PID>
```

### Token "iss claim not valid" Error

This occurs when Keycloak hostname doesn't match. Update Postman Access Token URL from `localhost` to `keycloak` when testing Docker environment.

### 401 Unauthorized (with token)

Token expired. Click **Get New Access Token** in Postman.

### Services Not Starting

Check logs:

```bash
docker logs keycloak
docker logs api-gateway
docker logs product-service
```

---

## Summary

You now have:

- ✅ Keycloak IAM server integrated with microservices
- ✅ OAuth2 Client Credentials authentication flow
- ✅ JWT token-based security on all endpoints
- ✅ Centralized security configuration via API Gateway
- ✅ Realm export/import for automated Keycloak setup
- ✅ Standalone and integrated Docker configurations

**Key Benefits:**
- Centralized authentication management
- No security code in individual services
- Scalable security architecture
- Industry-standard OAuth2/OpenID Connect

**Next Steps:**
- Implement Authorization Code Flow for user authentication
- Add role-based access control (RBAC)
- Configure refresh tokens
- Implement Single Sign-On (SSO)
