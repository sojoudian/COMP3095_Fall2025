# Microservices Implementation Guide - COMP 3095
## Complete Step-by-Step Implementation of Product Service

### Table of Contents
1. [Project Overview](#project-overview)
2. [Prerequisites](#prerequisites)
3. [Project Setup](#project-setup)
4. [Module Configuration](#module-configuration)
5. [Database Configuration](#database-configuration)
6. [Implementation Steps](#implementation-steps)
7. [Testing](#testing)
8. [Troubleshooting](#troubleshooting)

---

## Project Overview

This guide provides a detailed implementation of a microservices architecture using Spring Boot, MongoDB, and Gradle with Kotlin DSL. The project implements a RESTful Product Service with full CRUD operations following enterprise-level best practices.

### Architecture Components
- **Parent Project**: `microservices-parent` - Container for all microservices
- **Service Module**: `product-service` - REST API for product management
- **Database**: MongoDB (containerized with Docker)
- **Build Tool**: Gradle with Kotlin DSL
- **Framework**: Spring Boot 3.5.5

---

## Prerequisites

### Required Software
1. **IntelliJ IDEA** (2024.1 or later)
2. **Java JDK 21**
3. **Docker Desktop** (for MongoDB container)
4. **Git** (for version control)
5. **Postman or curl** (for API testing)

### IntelliJ Plugins Required
- Lombok Plugin (usually pre-installed)
- Delombok Plugin
- MapStruct Support Plugin

---

## Project Setup

### Step 1: Create Git Repository
```bash
# Initialize git repository
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin [your-repository-url]
git push -u origin main
```

![Alt text](https://raw.githubusercontent.com/sojoudian/COMP3095_Fall2025/refs/heads/master/Week02/img/01_Create_and_Setup_Git_Repository.png)

### Step 2: Create Parent Project in IntelliJ

1. **File → New → Project**
2. Configure project settings:
   - **Type**: Java
   - **Name**: `microservices-parent`
   - **Location**: Your git folder
   - **Language**: Java
   - **Build system**: Gradle
   - **JDK**: 21
   - **Gradle DSL**: Kotlin
   - **Advanced Settings**:
     - GroupId: `ca.gbc`
     - ArtifactId: `microservices-parent`
     - Package name: `ca.gbc.microservicesparent`
3. **IMPORTANT**: Uncheck "Add sample code"
4. Click **Create**

![Alt text](https://raw.githubusercontent.com/sojoudian/COMP3095_Fall2025/refs/heads/master/Week02/img/02_Create_Parent_Project_in_IntelliJ.png)


### Step 3: Configure IntelliJ for Lombok

1. **Settings → Build, Execution, Deployment → Compiler → Annotation Processors**
   - Enable annotation processing ✓
2. **Settings → Plugins**
   - Verify Lombok is installed
   - Install Delombok plugin
   - Install MapStruct Support plugin

![Alt text](https://raw.githubusercontent.com/sojoudian/COMP3095_Fall2025/refs/heads/master/Week02/img/03_1_Plugins.png)
![Alt text](https://raw.githubusercontent.com/sojoudian/COMP3095_Fall2025/refs/heads/master/Week02/img/3_2_enableAnnotation.png)

---

## Module Configuration

### Step 4: Create Product Service Module

1. **Right-click** on `microservices-parent` → **New → Module**
2. Select **Spring Boot** (shown as "Spring Boot" in generators)
3. Configure module:
   - **Name**: `product-service`
   - **Language**: Java
   - **Type**: Gradle - Kotlin
   - **Group**: `ca.gbc`
   - **Artifact**: `product-service`
   - **Package name**: `ca.gbc.productservice`
   - **Java**: 21
4. Click **Next**
5. Select dependencies:
   - Lombok (Developer Tools)
   - Spring Web (Web)
   - Spring Data MongoDB (NoSQL)
   - Spring Boot DevTools (Developer Tools)
   - Spring Boot Actuator (Ops)
6. Click **Create**

![Alt text](https://raw.githubusercontent.com/sojoudian/COMP3095_Fall2025/refs/heads/master/Week02/img/04_CreateModule.png)


---

## Database Configuration

### Step 5: Configure MongoDB Connection

1. Navigate to: `product-service/src/main/resources/application.properties`
2. Add MongoDB configuration:
```properties
spring.application.name=product-service
spring.data.mongodb.uri=mongodb://localhost:27017/product-service
```

### Step 6: Setup MongoDB Container

```bash
# Create Docker network (optional, for mongo-express)
docker network create mynetwork
```
![Alt text](https://raw.githubusercontent.com/sojoudian/COMP3095_Fall2025/refs/heads/master/Week02/img/09_CreateDockerNetwork.png)


```bash
# Run MongoDB container
docker run -d \
  --name comp3095-mongodb \
  --network=mynetwork \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=rootadmin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  --restart unless-stopped \
  mongo:latest
```
![Alt text](https://raw.githubusercontent.com/sojoudian/COMP3095_Fall2025/refs/heads/master/Week02/img/10_Docker_Run_MongoDB.png)


```bash
# Optional: Run Mongo Express (GUI)
docker run -d \
  --name mongo-express \
  --network=mynetwork \
  --restart unless-stopped \
  -p 8081:8081 \
  -e ME_CONFIG_MONGODB_ADMINUSERNAME=rootadmin \
  -e ME_CONFIG_MONGODB_ADMINPASSWORD=password \
  -e ME_CONFIG_MONGODB_SERVER=comp3095-mongodb \
  mongo-express
```

---

## Implementation Steps

### Step 7: Create Product Model

1. Create package: `ca.gbc.productservice.model`
   - Right-click `ca.gbc.productservice` → New → Package → `model`
2. Create `Product.java` class:

```java
package ca.gbc.productservice.model;

import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.Builder;
import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import java.math.BigDecimal;

@AllArgsConstructor
@NoArgsConstructor
@Builder
@Data
@Document(value = "product")
public class Product {
    @Id
    private String id;
    private String name;
    private String description;
    private BigDecimal price;
}
```

**Key Annotations Explained:**
- `@Document(value = "product")`: Maps to MongoDB collection
- `@Id`: Marks primary key field
- `@Data`: Generates getters, setters, toString, equals, hashCode
- `@Builder`: Enables builder pattern
- `@AllArgsConstructor/@NoArgsConstructor`: Generate constructors

### Step 8: Create Repository Interface

1. **Create repository package:**
   - Right-click on `ca.gbc.productservice` (same location where you created `model`)
   - Select **New → Package** (or **New → Directory** if Package isn't showing)
   - Type: `repository`
   - Press Enter

2. **Create ProductRepository interface:**
   - Right-click on the new `repository` package/directory
   - Select **New → Java Class**
   - In the dialog:
     - **Name**: `ProductRepository`
     - **Kind**: Change from "Class" to **Interface**
   - Click OK/Create

3. **Replace the generated code with:**

```java
package ca.gbc.productservice.repository;

import ca.gbc.productservice.model.Product;
import org.springframework.data.mongodb.repository.MongoRepository;

public interface ProductRepository extends MongoRepository<Product, String> {
}
```

**Note**: MongoRepository provides all CRUD operations automatically

### Step 9: Create DTOs

1. Create package: `ca.gbc.productservice.dto`
2. Create `ProductRequest.java`:

```java
package ca.gbc.productservice.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.math.BigDecimal;

@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ProductRequest {
    private String name;
    private String description;
    private BigDecimal price;
}
```

3. Create `ProductResponse.java`:

```java
package ca.gbc.productservice.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.math.BigDecimal;

@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class ProductResponse {
    private String id;
    private String name;
    private String description;
    private BigDecimal price;
}
```

**DTO Pattern Benefits:**
- Decouples internal model from API contract
- Prevents exposing internal structure
- Allows API evolution without breaking changes

### Step 10: Implement Service Layer

1. Create package: `ca.gbc.productservice.service`
2. Create `ProductService.java`:

```java
package ca.gbc.productservice.service;

import ca.gbc.productservice.dto.ProductRequest;
import ca.gbc.productservice.dto.ProductResponse;
import ca.gbc.productservice.model.Product;
import ca.gbc.productservice.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class ProductService {
    
    private final ProductRepository productRepository;
    
    public void createProduct(ProductRequest productRequest) {
        Product product = Product.builder()
                .name(productRequest.getName())
                .description(productRequest.getDescription())
                .price(productRequest.getPrice())
                .build();
        
        productRepository.save(product);
        log.info("Product {} is saved", product.getId());
    }
    
    public List<ProductResponse> getAllProducts() {
        List<Product> products = productRepository.findAll();
        return products.stream().map(this::mapToProductResponse).toList();
    }
    
    private ProductResponse mapToProductResponse(Product product) {
        return ProductResponse.builder()
                .id(product.getId())
                .name(product.getName())
                .description(product.getDescription())
                .price(product.getPrice())
                .build();
    }
    
    // Additional methods for PUT and DELETE (optional)
    public void updateProduct(String id, ProductRequest productRequest) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found"));
        
        product.setName(productRequest.getName());
        product.setDescription(productRequest.getDescription());
        product.setPrice(productRequest.getPrice());
        
        productRepository.save(product);
        log.info("Product {} is updated", id);
    }
    
    public void deleteProduct(String id) {
        productRepository.deleteById(id);
        log.info("Product {} is deleted", id);
    }
}
```

### Step 11: Create REST Controller

1. Create package: `ca.gbc.productservice.controller`
2. Create `ProductController.java`:

```java
package ca.gbc.productservice.controller;

import ca.gbc.productservice.dto.ProductRequest;
import ca.gbc.productservice.dto.ProductResponse;
import ca.gbc.productservice.service.ProductService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/product")
@RequiredArgsConstructor
public class ProductController {
    
    private final ProductService productService;
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public void createProduct(@RequestBody ProductRequest productRequest) {
        productService.createProduct(productRequest);
    }
    
    @GetMapping
    @ResponseStatus(HttpStatus.OK)
    public List<ProductResponse> getAllProducts() {
        return productService.getAllProducts();
    }
    
    // Optional: PUT endpoint
    @PutMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void updateProduct(@PathVariable String id, 
                             @RequestBody ProductRequest productRequest) {
        productService.updateProduct(id, productRequest);
    }
    
    // Optional: DELETE endpoint
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteProduct(@PathVariable String id) {
        productService.deleteProduct(id);
    }
}
```

### Step 12: Clean Up Parent Project

1. **Delete** the `src` folder from `microservices-parent` root
2. **Update** `settings.gradle.kts` in microservices-parent:

```kotlin
rootProject.name = "microservices-parent"
include("product-service")
```

---

## Testing

### Step 13: Start and Test Application

1. **Start MongoDB** (if not running):
```bash
docker start comp3095-mongodb
```

2. **Run ProductServiceApplication** in IntelliJ

3. **Test Endpoints**:

#### Create Product (POST)
```bash
curl -X POST http://localhost:8080/api/product \
  -H "Content-Type: application/json" \
  -d '{
    "name": "MacBook Pro",
    "description": "Apple MacBook Pro 16-inch",
    "price": 2499.99
  }'
```

#### Get All Products (GET)
```bash
curl http://localhost:8080/api/product
```

#### Update Product (PUT)
```bash
curl -X PUT http://localhost:8080/api/product/{id} \
  -H "Content-Type: application/json" \
  -d '{
    "name": "MacBook Pro M3",
    "description": "Apple MacBook Pro 16-inch with M3",
    "price": 2999.99
  }'
```

#### Delete Product (DELETE)
```bash
curl -X DELETE http://localhost:8080/api/product/{id}
```

### Using Postman

1. **Create Product**:
   - Method: POST
   - URL: `http://localhost:8080/api/product`
   - Headers: `Content-Type: application/json`
   - Body (raw JSON):
   ```json
   {
     "name": "iPhone 15 Pro",
     "description": "Latest iPhone with A17 Pro chip",
     "price": 1199.99
   }
   ```

2. **Get Products**:
   - Method: GET
   - URL: `http://localhost:8080/api/product`

---

## Troubleshooting

### Common Issues and Solutions

#### 1. MongoDB Connection Failed
**Error**: `MongoSocketOpenException`
**Solution**:
```bash
# Check if MongoDB is running
docker ps

# If not running, start it
docker start comp3095-mongodb

# Or create new container
docker run -d -p 27017:27017 --name mongodb mongo:latest
```

#### 2. Lombok Not Working
**Error**: Getters/setters not found
**Solution**:
1. Enable annotation processing in IntelliJ
2. Rebuild project: Build → Rebuild Project
3. Invalidate caches: File → Invalidate Caches and Restart

#### 3. Port Already in Use
**Error**: `Port 8080 already in use`
**Solution**:
Add to application.properties:
```properties
server.port=8081
```

#### 4. Gradle Build Issues
**Solution**:
```bash
# Clean and rebuild
./gradlew clean build

# Or in IntelliJ
# View → Tool Windows → Gradle → Refresh
```

---

## Project Structure

Final project structure should look like:
```
microservices-parent/
├── .gradle/
├── .idea/
├── gradle/
├── product-service/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/
│   │   │   │   └── ca/gbc/productservice/
│   │   │   │       ├── controller/
│   │   │   │       │   └── ProductController.java
│   │   │   │       ├── dto/
│   │   │   │       │   ├── ProductRequest.java
│   │   │   │       │   └── ProductResponse.java
│   │   │   │       ├── model/
│   │   │   │       │   └── Product.java
│   │   │   │       ├── repository/
│   │   │   │       │   └── ProductRepository.java
│   │   │   │       ├── service/
│   │   │   │       │   └── ProductService.java
│   │   │   │       └── ProductServiceApplication.java
│   │   │   └── resources/
│   │   │       └── application.properties
│   │   └── test/
│   ├── build.gradle.kts
│   └── settings.gradle.kts
├── build.gradle.kts
├── settings.gradle.kts
├── gradlew
└── gradlew.bat
```

---

## GitLab Submission

### Add Instructor as Maintainer

1. Log into GitLab
2. Navigate to your project (COMP3095-20XX)
3. Go to: **Project information → Members**
4. Under "Invite member":
   - Username: `s.santilli`
   - Role: **Maintainer**
   - Leave expiration blank
5. Click **Invite**

### Final Commit
```bash
git add .
git commit -m "Complete product-service implementation with MongoDB"
git push origin main
```

---

## Next Steps

This foundation can be extended with:
- API documentation (OpenAPI/Swagger)
- Unit and integration tests
- Docker containerization
- Service discovery (Eureka)
- API Gateway
- Configuration server
- Circuit breakers (Hystrix/Resilience4j)
- Message queuing (RabbitMQ/Kafka)
- Monitoring and logging (ELK stack)

---

## Conclusion

This implementation demonstrates:
- ✅ Proper microservices architecture
- ✅ Clean code separation (Controller → Service → Repository)
- ✅ DTO pattern for API contracts
- ✅ MongoDB integration with Spring Data
- ✅ RESTful API design
- ✅ Gradle multi-module project structure
- ✅ Modern Java and Spring Boot features

The project is production-ready and follows enterprise best practices for microservices development.
