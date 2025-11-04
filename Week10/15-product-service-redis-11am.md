# Product Service with Redis Caching

## Introduction

Add Redis caching to product-service for comp3095_fall2025_11am.

**Time:** 90-120 minutes

---

## Table of Contents

- [Step 1: Update build.gradle.kts](#step-1-update-buildgradlekts)
- [Step 2: Update Docker Compose](#step-2-update-docker-compose)
- [Step 3: Create redis.conf](#step-3-create-redisconf)
- [Step 4: Enable Caching](#step-4-enable-caching)
- [Step 5: Update application.properties](#step-5-update-applicationproperties)
- [Step 6: Update application-docker.properties](#step-6-update-application-dockerproperties)
- [Step 7: Create RedisConfig](#step-7-create-redisconfig)
- [Step 8: Create ProductService Interface and ProductServiceImpl](#step-8-create-productservice-interface-and-productserviceimpl)
- [Step 9: Create ProductServiceApplicationCacheTests](#step-9-create-productserviceapplicationcachetests)
- [Step 10: Test Locally](#step-10-test-locally)
- [Step 11: Test with Docker Compose](#step-11-test-with-docker-compose)
- [Summary](#summary)

---

## Step 1: Update build.gradle.kts

Open `microservices-parent/product-service/build.gradle.kts`

Replace entire file with:

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
    testImplementation("org.testcontainers:redis")

    // REST Assured for testing
    testImplementation("io.rest-assured:rest-assured")

    // Test Platform Launcher
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

Reload Gradle.

---

## Step 2: Update Docker Compose

```
microservices-parent
└── docker-compose.yml
```

Open `microservices-parent/docker-compose.yml`

Add redis and redis-insight services after mongodb service:

```yaml
  redis:
    image: redis:latest
    ports:
      - "6379:6379"
    volumes:
      - ./init/redis/redis.conf:/usr/local/etc/redis/redis.conf
      - redis-data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    container_name: redis
    networks:
      - microservices-network

  redis-insight:
    image: redislabs/redisinsight:1.14.0
    ports:
      - "8001:8001"
    container_name: redis-insight
    depends_on:
      - redis
    networks:
      - microservices-network
```

Add redis-data volume in volumes section:

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
  redis-data:
    driver: local
```

Complete `docker-compose.yml`:

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

  redis:
    image: redis:latest
    ports:
      - "6379:6379"
    volumes:
      - ./init/redis/redis.conf:/usr/local/etc/redis/redis.conf
      - redis-data:/data
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    container_name: redis
    networks:
      - microservices-network

  redis-insight:
    image: redislabs/redisinsight:1.14.0
    ports:
      - "8001:8001"
    container_name: redis-insight
    depends_on:
      - redis
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
  redis-data:
    driver: local
```

---

## Step 3: Create redis.conf

Create directory: `microservices-parent/init/redis`

```
microservices-parent
└── init
    └── redis
        └── redis.conf
```

Create file: `microservices-parent/init/redis/redis.conf`

```
# This enables password authentication
requirepass password

# Save a snapshot every 900 seconds if at least 1 key changes
save 900 1

# Enable Append-Only File (AOF) persistence
appendonly yes
```

---

## Step 4: Enable Caching

```
microservices-parent
└── product-service
    └── src
        └── main
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── productservice
                                └── ProductServiceApplication.java
```

Open `microservices-parent/product-service/src/main/java/ca/gbc/comp3095/productservice/ProductServiceApplication.java`

Replace with:

```java
package ca.gbc.comp3095.productservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching
public class ProductServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProductServiceApplication.class, args);
    }

}
```

---

## Step 5: Update application.properties

```
microservices-parent
└── product-service
    └── src
        └── main
            └── resources
                └── application.properties
```

Open `microservices-parent/product-service/src/main/resources/application.properties`

Add at the end:

```properties
# Redis Configuration
spring.cache.type=redis
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.data.redis.password=password
spring.cache.redis.time-to-live=60s
```

---

## Step 6: Update application-docker.properties

```
microservices-parent
└── product-service
    └── src
        └── main
            └── resources
                └── application-docker.properties
```

Open `microservices-parent/product-service/src/main/resources/application-docker.properties`

Add at the end:

```properties
# Redis Configuration
spring.data.redis.host=redis
spring.data.redis.port=6379
spring.data.redis.password=password
spring.cache.type=redis
spring.cache.redis.time-to-live=60s
```

---

## Step 7: Create RedisConfig

Create directory: `microservices-parent/product-service/src/main/java/ca/gbc/comp3095/productservice/config`

```
microservices-parent
└── product-service
    └── src
        └── main
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── productservice
                                └── config
                                    └── RedisConfig.java
```

Create `RedisConfig.java` (class):

```java
package ca.gbc.comp3095.productservice.config;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.jsontype.BasicPolymorphicTypeValidator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;

import java.time.Duration;

@Configuration
public class RedisConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {

        ObjectMapper objectMapper = new ObjectMapper();

        objectMapper.activateDefaultTyping(
                BasicPolymorphicTypeValidator.builder()
                        .allowIfSubType(Object.class)
                        .build(),
                ObjectMapper.DefaultTyping.EVERYTHING,
                JsonTypeInfo.As.PROPERTY
        );

        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        Jackson2JsonRedisSerializer<Object> serializer =
                new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);

        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration
                .defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues()
                .serializeValuesWith(
                        RedisSerializationContext.SerializationPair.fromSerializer(serializer)
                );

        return RedisCacheManager
                .builder(connectionFactory)
                .cacheDefaults(redisCacheConfiguration)
                .build();
    }
}
```

---

## Step 8: Create ProductService Interface and ProductServiceImpl

### 8.1: Create ProductService Interface

```
microservices-parent
└── product-service
    └── src
        └── main
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── productservice
                                └── service
                                    └── ProductService.java
```

Rename existing `ProductService.java` to `ProductServiceOld.java`

Create `ProductService.java` (interface):

```java
package ca.gbc.comp3095.productservice.service;

import ca.gbc.comp3095.productservice.dto.ProductRequest;
import ca.gbc.comp3095.productservice.dto.ProductResponse;

import java.util.List;

public interface ProductService {

    ProductResponse createProduct(ProductRequest productRequest);
    List<ProductResponse> getAllProducts();
    String updateProduct(String productId, ProductRequest productRequest);
    void deleteProduct(String productId);

}
```

### 8.2: Create ProductServiceImpl

```
microservices-parent
└── product-service
    └── src
        └── main
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── productservice
                                └── service
                                    └── ProductServiceImpl.java
```

Create `ProductServiceImpl.java` (class):

```java
package ca.gbc.comp3095.productservice.service;

import ca.gbc.comp3095.productservice.dto.ProductRequest;
import ca.gbc.comp3095.productservice.dto.ProductResponse;
import ca.gbc.comp3095.productservice.model.Product;
import ca.gbc.comp3095.productservice.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@Slf4j
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;
    private final MongoTemplate mongoTemplate;
    private final CacheManager cacheManager;

    @Override
    @CachePut(value = "PRODUCT_CACHE", key = "#result.id()")
    public ProductResponse createProduct(ProductRequest productRequest) {

        log.debug("Creating new product {}", productRequest);

        Product product = Product.builder()
                .name(productRequest.name())
                .description(productRequest.description())
                .price(productRequest.price())
                .build();

        Product savedProduct = productRepository.save(product);
        log.debug("Saved product {}", product);

        return new ProductResponse(product.getId(), product.getName(),
                product.getDescription(), product.getPrice());
    }

    @Override
    @Cacheable(value = "PRODUCT_CACHE", key = "'ALL_PRODUCTS'")
    public List<ProductResponse> getAllProducts() {
        log.debug("Returning a List of all products");
        List<Product> products = productRepository.findAll();

        return products.stream()
                .map(this::maptoProductResponse).toList();
    }

    private ProductResponse maptoProductResponse(Product product) {
        return new ProductResponse(product.getId(), product.getName(),
                product.getDescription(), product.getPrice());
    }

    @Override
    @CachePut(value = "PRODUCT_CACHE", key = "#result")
    public String updateProduct(String productId, ProductRequest productRequest) {

        log.debug("Updating product with Id {}", productId);

        Query query = new Query();
        query.addCriteria(Criteria.where("id").is(productId));
        Product product = mongoTemplate.findOne(query, Product.class);

        if(product != null){
            product.setName(productRequest.name());
            product.setDescription(productRequest.description());
            product.setPrice(productRequest.price());
            return productRepository.save(product).getId();
        }
        return productId;
    }

    @Override
    @CacheEvict(value = "PRODUCT_CACHE", key = "#productId")
    public void deleteProduct(String productId) {
        log.debug("Deleting product with Id {}", productId);
        productRepository.deleteById(productId);
    }

}
```

---

## Step 9: Create ProductServiceApplicationCacheTests

Create directory: `microservices-parent/product-service/src/test/java/ca/gbc/comp3095/productservice`

```
microservices-parent
└── product-service
    └── src
        └── test
            └── java
                └── ca
                    └── gbc
                        └── comp3095
                            └── productservice
                                └── ProductServiceApplicationCacheTests.java
```

Create `ProductServiceApplicationCacheTests.java` (class):

```java
package ca.gbc.comp3095.productservice;

import ca.gbc.comp3095.productservice.dto.ProductRequest;
import ca.gbc.comp3095.productservice.dto.ProductResponse;
import ca.gbc.comp3095.productservice.model.Product;
import ca.gbc.comp3095.productservice.repository.ProductRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.containers.wait.strategy.Wait;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.math.BigDecimal;
import java.time.Duration;


import static org.junit.jupiter.api.Assertions.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@Testcontainers
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
public class ProductServiceApplicationCacheTests {

    @Container
    @ServiceConnection(name = "redis")
    static GenericContainer<?> redisContainer = new GenericContainer(DockerImageName.parse("redis:7.4.3"))
            .withExposedPorts(6379)
            .waitingFor(Wait.forListeningPort())
            .withStartupTimeout(Duration.ofSeconds(120));

    @Container
    @ServiceConnection(name = "mongodb")
    static MongoDBContainer mongodbContainer = new MongoDBContainer(DockerImageName.parse("mongo:5.0"))
            .withStartupTimeout(Duration.ofSeconds(120));

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private CacheManager cacheManager;

    @Autowired
    ProductRepository productRepository;

    @MockitoSpyBean
    ProductRepository productRepositorySpy;

    @Autowired
    private final ObjectMapper objectMapper = new ObjectMapper();

    @BeforeEach
    void setup(){
        productRepository.deleteAll();
        Cache cache = cacheManager.getCache("PRODUCT_CACHE");
        if (cache != null) {
            cache.clear();
        }
        System.out.println("Redis container port: " + redisContainer.getMappedPort(6379));
        System.out.println("MongoDB container URI: " + mongodbContainer.getReplicaSetUrl());
    }

    @Test
    void testCreateProductAndCacheIt() throws Exception {

        ProductRequest productRequest =
                new ProductRequest("Samsung TV", "Samsung TV - 2025", BigDecimal.valueOf(2000));

        MvcResult result = mockMvc.perform(post("/api/product")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(productRequest)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").exists())
                .andExpect(jsonPath("$.name").value(productRequest.name()))
                .andReturn();

        ProductResponse createdProduct =
                objectMapper.readValue(result.getResponse().getContentAsString(), ProductResponse.class);
        String productId = createdProduct.id();

        assertTrue(productRepository.findById(productId).isPresent(), "Product should exist in database");

        Cache cache = cacheManager.getCache("PRODUCT_CACHE");
        assertNotNull(cache, "Cache 'PRODUCT_CACHE' should exist");
        ProductResponse cachedProduct = cache.get(productId, ProductResponse.class);
        assertNotNull(cachedProduct, "Product should be cached");
        assertEquals(productId, cachedProduct.id(), "Cached product ID should match");

    }

    @Test
    void testGetProductsAndVerifyCache() throws Exception {

        Product product = Product.builder()
                .name("Text Book")
                .description("COMP3095 Text Book")
                .price(BigDecimal.valueOf(100))
                .build();

        productRepository.save(product);

        mockMvc.perform(get("/api/product"))
        .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].name").value(product.getName()));

        Mockito.verify(productRepositorySpy, Mockito.times(1)).findAll();

        Mockito.clearInvocations(productRepositorySpy);

        mockMvc.perform(get("/api/product"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].name").value(product.getName()));

        Mockito.verify(productRepositorySpy, Mockito.times(0)).findAll();
    }

    @Test
    void testUpdateProductAndVerifyCache() throws Exception {

        Product product = Product.builder()
                .name("Laptop")
                .description("Basic Laptop")
                .price(BigDecimal.valueOf(2000L))
                .build();

        productRepository.save(product);

        ProductRequest updatedProductRequest = new ProductRequest(
                "Gaming Laptop", "Gaming Laptop Pro", BigDecimal.valueOf(3000L));

        mockMvc.perform(MockMvcRequestBuilders.put("/api/product/{productId}", product.getId())
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(updatedProductRequest)))
                .andExpect(status().isNoContent());

        Cache cache = cacheManager.getCache("PRODUCT_CACHE");
        assertNotNull(cache);

        String cachedProductId = cache.get(product.getId(), String.class);

        assertNotNull(cachedProductId);
        Assertions.assertEquals(product.getId(), cachedProductId);

    }

    @Test
    void testDeleteProductAndVerifyCacheEviction() throws Exception {

        Product product = Product.builder()
                .name("Laptop")
                .description("Basic Laptop")
                .price(BigDecimal.valueOf(2000L))
                .build();
        productRepository.save(product);

        Cache cache = cacheManager.getCache("PRODUCT_CACHE");
        assertNotNull(cache);
        cache.put(product.getId(), product.getId());

        mockMvc.perform(MockMvcRequestBuilders.delete("/api/product/{productId}", product.getId()))
                .andExpect(status().isNoContent());

        Cache.ValueWrapper cachedProduct = cache.get(product.getId());
        assertNull(cachedProduct, "Product should have been evicted from cache after delete.");

    }

}
```

---

## Step 10: Test Locally

### Start mongodb container

```bash
docker run -d --name mongodb -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  mongo:latest
```

### Start redis container

```bash
docker run -d --name redis -p 6379:6379 \
  redis:latest redis-server --requirepass password
```

### Run product-service

In IntelliJ: Run `ProductServiceApplication`

### Run tests

```bash
cd microservices-parent/product-service
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
docker-compose logs -f product-service
```

Access RedisInsight:

```
http://localhost:8001
```

Connect to Redis:
- Click: "I already have a database"
- Host: host.docker.internal
- Port: 6379
- Name: product-service-cache
- Username: default
- Password: password

Stop services:

```bash
docker-compose down
```

---

## Summary

- Added Spring Data Redis for caching
- Created RedisConfig for cache configuration
- Updated ProductServiceImpl with @CachePut, @Cacheable, @CacheEvict
- Added Redis and RedisInsight containers to Docker Compose
- Created integration tests with TestContainers
- Configured TTL and JSON serialization for cached data
