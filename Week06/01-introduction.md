# Order Service Implementation - Introduction

## Lab Overview

In this lab, you will design and implement a new microservice called **order-service** as part of your growing microservices architecture. This lab walks through the complete development workflow—from modeling entities using JPA, to integrating a PostgreSQL database, to writing automated integration tests using TestContainers, and finally deploying the service using Docker and Docker Compose.

By the end of the lab, you will have a fully functional, containerized service capable of handling order placement and persistence operations in a relational database.

---

## Background

### Why Microservices?

Microservice-based systems encourage **separation of concerns** and **modular deployment**. In a commerce-oriented system, separating the product catalog (product-service) from the order-handling logic (order-service) provides several key benefits:

**Benefits:**
- ✅ **Independent Evolution:** Services can evolve independently without affecting each other
- ✅ **Different Scaling:** Scale product catalog differently from order processing
- ✅ **Loose Coupling:** Services remain loosely coupled through well-defined APIs
- ✅ **Technology Diversity:** Use the best database for each service's needs
- ✅ **Team Autonomy:** Different teams can own different services
- ✅ **Fault Isolation:** Failure in one service doesn't cascade to others

### Product Service vs Order Service

| Aspect | Product Service | Order Service |
|--------|----------------|---------------|
| **Purpose** | Manage product catalog | Handle customer orders |
| **Database** | MongoDB (NoSQL) | PostgreSQL (SQL) |
| **Data Model** | Simple documents | Complex relationships |
| **Schema** | Flexible, schema-less | Fixed, schema-based |
| **Relationships** | Embedded documents | JPA entity relationships |
| **Transactions** | Not critical | Critical (ACID) |
| **Queries** | Document-based | Relational joins |
| **Scaling** | Read-heavy | Write-heavy |

### What You'll Build

In this lab, you will:

#### **1. Use Spring Data JPA**
- Persist relational data with PostgreSQL
- Define entity relationships using JPA annotations
- Implement repository pattern for data access

#### **2. Establish Entity Relationships**
- Use `@OneToMany` for Order → OrderLineItems
- Use `@ManyToOne` for OrderLineItem → Order
- Configure cascade operations with `CascadeType.ALL`
- Understand bidirectional relationships

#### **3. Perform CRUD Operations**
- Create complex domain models (Order with multiple OrderLineItems)
- Save cascading entities
- Query related data
- Manage transactions

#### **4. Configure PostgreSQL Docker Container**
- Setup PostgreSQL as backend database
- Configure database connection
- Create initialization scripts
- Use pgAdmin for database management

#### **5. Author Multistage Dockerfile**
- Build optimized Docker images
- Separate build and runtime stages
- Reduce image size
- Follow security best practices

#### **6. Update docker-compose.yml**
- Add order-service to existing architecture
- Configure PostgreSQL service
- Setup pgAdmin service
- Manage service dependencies

#### **7. Add Integration Testing**
- Use TestContainers for PostgreSQL
- Write integration tests with real database
- Validate application behavior in containerized environment
- Test entity relationships and cascading

---

## Learning Objectives

By the end of this lab, you will be able to:

### **Objective 1: Design and Implement Microservice**
**Skills:**
- Create a new microservice (order-service) using Spring Boot
- Use Spring Data JPA for data persistence
- Apply Spring Boot best practices

**Deliverables:**
- Fully functional order-service microservice
- RESTful API endpoint for placing orders
- Complete layered architecture implementation

### **Objective 2: Model Relational Entities**
**Skills:**
- Use JPA annotations: `@Entity`, `@Table`, `@Id`, `@GeneratedValue`
- Define relationships: `@OneToMany`, `@ManyToOne`
- Configure cascading behavior: `CascadeType.ALL`
- Understand fetch strategies: `LAZY` vs `EAGER`

**Deliverables:**
- Order entity with proper annotations
- OrderLineItem entity with relationship mapping
- Bidirectional relationship between entities

### **Objective 3: Construct Layered Architecture**
**Skills:**
- Design DTOs for request/response separation
- Implement service layer for business logic
- Create repository layer for data access
- Build controller layer for REST endpoints
- Apply dependency injection

**Deliverables:**
- OrderRequest and OrderLineItemDto classes
- OrderService with transaction management
- OrderRepository interface
- OrderController with REST endpoints

### **Objective 4: Configure PostgreSQL Database**
**Skills:**
- Configure PostgreSQL connection in Spring Boot
- Use environment variables for configuration
- Understand JDBC connection properties
- Setup database initialization scripts

**Deliverables:**
- application.properties with PostgreSQL configuration
- application-docker.properties for containerized environment
- Working connection to PostgreSQL database

### **Objective 5: Use IntelliJ's Docker Plugin**
**Skills:**
- Pull Docker images from Docker Hub
- Create and configure containers
- Set environment variables
- Bind ports for container access
- Inspect container logs and status

**Deliverables:**
- Running PostgreSQL container
- Running pgAdmin container (optional)
- Verified connectivity to databases

### **Objective 6: Write Integration Tests**
**Skills:**
- Use TestContainers library
- Write integration tests with real database
- Validate `placeOrder()` endpoint behavior
- Test in isolated environment
- Use REST Assured for API testing

**Deliverables:**
- OrderServiceApplicationTests class
- Passing integration tests with TestContainers
- Verified order placement functionality

### **Objective 7: Build Multistage Dockerfile**
**Skills:**
- Create multistage Docker builds
- Optimize image size (JDK → JRE)
- Follow security best practices (non-root user)
- Configure environment variables
- Define entry points

**Deliverables:**
- Dockerfile with build and runtime stages
- Optimized order-service Docker image (<400MB)
- Production-ready containerized application

### **Objective 8: Extend Docker Compose Configuration**
**Skills:**
- Add new services to docker-compose.yml
- Configure service dependencies
- Setup networked communication
- Manage environment variables
- Use volumes for data persistence

**Deliverables:**
- Updated docker-compose.yml with 6 services
- order-service integrated into network
- postgres and pgAdmin services configured
- All services communicating properly

### **Objective 9: Execute End-to-End API Tests**
**Skills:**
- Use Postman for API testing
- Create and send HTTP requests
- Validate response status and body
- Test in Dockerized environment
- Verify data persistence

**Deliverables:**
- Successful POST request to /api/order
- 201 Created response received
- Order data verified in PostgreSQL database
- Postman collection for order-service

### **Objective 10: Apply Developer Tooling Features**
**Skills:**
- Configure Spring Boot DevTools
- Enable automatic restarts
- Use LiveReload server
- Streamline development workflow

**Deliverables:**
- DevTools configured and working
- Automatic application restart on changes
- Improved development experience

---

## Prerequisites

### **Required Completed Labs:**
- ✅ **Week 1:** Environment setup (Java 21, IntelliJ, Docker, Git, Gradle)
- ✅ **Week 2-3:** Product Service with MongoDB (complete CRUD operations)
- ✅ **Week 4:** API Testing with TestContainers (MongoDB tests)
- ✅ **Week 5:** Docker Containerization (product-service in Docker)

### **Required Software:**
- ✅ **IntelliJ IDEA** 2024.1 or later
- ✅ **Java JDK 21**
- ✅ **Docker Desktop** 20.10 or later (running)
- ✅ **Postman** latest version
- ✅ **Git** for version control

### **Required Knowledge:**
- ✅ Understanding of REST APIs and HTTP methods
- ✅ Basic SQL and relational database concepts
- ✅ Spring Boot application structure
- ✅ Docker and Docker Compose basics
- ✅ Gradle build tool usage

### **Verification Commands:**

```bash
# Check Java version
java --version
# Expected: openjdk 21.x.x

# Check Docker
docker --version
# Expected: Docker version 20.10.x or later

# Check Docker is running
docker ps
# Expected: CONTAINER ID   IMAGE   ... (should not error)

# Check Gradle
./gradlew --version
# Expected: Gradle 8.x

# Check Git
git --version
# Expected: git version 2.x.x
```

---

## Technology Stack

### **Core Technologies:**
- **Java 21** - Modern LTS version with latest features
- **Spring Boot 3.5.6** - Latest stable framework
- **Spring Data JPA** - Object-Relational Mapping
- **PostgreSQL Latest** - Relational database
- **Gradle (Kotlin DSL)** - Build automation

### **Additional Dependencies:**
- **Lombok** - Reduce boilerplate code
- **Spring Web** - RESTful web services
- **Spring Actuator** - Production-ready features
- **Spring Boot DevTools** - Development productivity
- **Flyway Migration** - Database version control (optional)

### **Testing Tools:**
- **JUnit 5** - Testing framework
- **TestContainers 1.21.3** - Container-based testing
- **REST Assured** - API testing library
- **Spring Boot Test** - Test support

### **DevOps Tools:**
- **Docker** - Containerization
- **Docker Compose** - Multi-container orchestration
- **pgAdmin** - PostgreSQL GUI tool (optional)

---

## Key Concepts

### **1. JPA (Java Persistence API)**

**What is JPA?**
- Specification for object-relational mapping (ORM)
- Maps Java objects to database tables
- Provides abstraction over SQL

**Key Annotations:**
- `@Entity` - Marks class as JPA entity
- `@Table` - Specifies database table name
- `@Id` - Marks primary key field
- `@GeneratedValue` - Auto-generates primary key values
- `@OneToMany` - One-to-many relationship
- `@ManyToOne` - Many-to-one relationship
- `@JoinColumn` - Specifies foreign key column

**Example:**
```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(cascade = CascadeType.ALL)
    private List<OrderLineItem> orderLineItems;
}
```

### **2. Entity Relationships**

**One-to-Many:**
- One Order has Many OrderLineItems
- Parent (Order) owns the relationship
- Children (OrderLineItems) reference parent

**Cascade Operations:**
- `CascadeType.ALL` - All operations cascade
- `CascadeType.PERSIST` - Save operation cascades
- `CascadeType.REMOVE` - Delete operation cascades

**Bidirectional vs Unidirectional:**
- Bidirectional: Both entities reference each other
- Unidirectional: Only one entity references the other

### **3. Transaction Management**

**What are Transactions?**
- Group of operations that succeed or fail together
- ACID properties (Atomicity, Consistency, Isolation, Durability)

**`@Transactional` Annotation:**
- Marks method/class for transaction management
- Automatic rollback on exception
- Ensures data consistency

**Example:**
```java
@Service
@Transactional
public class OrderService {
    public void placeOrder(OrderRequest request) {
        // All database operations in one transaction
        // Rollback if any exception occurs
    }
}
```

### **4. DTO Pattern (Data Transfer Object)**

**What are DTOs?**
- Objects used to transfer data between layers
- Decouple internal model from external API
- Control what data is exposed

**Benefits:**
- Security: Don't expose internal IDs
- Versioning: API can evolve independently
- Validation: Add constraints without polluting domain model

**Example:**
```java
// DTO for API request
public class OrderRequest {
    private List<OrderLineItemDto> orderLineItemDtoList;
}

// Entity for database
@Entity
public class Order {
    @Id
    private Long id;  // Not exposed in DTO
    private List<OrderLineItem> orderLineItems;
}
```

### **5. Repository Pattern**

**What is Repository?**
- Abstraction layer over data access
- Provides CRUD operations
- Hides database implementation details

**Spring Data JPA:**
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    // No implementation needed!
    // Spring provides: save(), findById(), findAll(), delete(), etc.
}
```

### **6. Layered Architecture**

**Layers:**
```
Controller Layer (REST API)
    ↓
Service Layer (Business Logic)
    ↓
Repository Layer (Data Access)
    ↓
Database (PostgreSQL)
```

**Responsibilities:**
- **Controller:** Handle HTTP requests/responses
- **Service:** Implement business logic, manage transactions
- **Repository:** Perform database operations
- **Model:** Represent domain entities

---

## What Makes This Lab Different?

### **Comparison with Product Service:**

| Feature | Product Service (Weeks 2-3) | Order Service (This Week) |
|---------|----------------------------|---------------------------|
| **Database** | MongoDB (NoSQL) | PostgreSQL (SQL) |
| **Data Access** | Spring Data MongoDB | Spring Data JPA |
| **Entity Type** | Document (`@Document`) | Relational (`@Entity`) |
| **ID Type** | String (ObjectId) | Long (auto-increment) |
| **Schema** | Flexible, schema-less | Fixed, schema-based |
| **Relationships** | Embedded documents | JPA relationships (`@OneToMany`) |
| **Transactions** | Not emphasized | Critical (`@Transactional`) |
| **Query Language** | MongoDB Query | SQL/JPQL |
| **Complexity** | Single entity | Multiple related entities |
| **Port** | 8084 | 8082 |

### **New Concepts Introduced:**

1. **Relational Database Design**
   - Primary keys and foreign keys
   - Table relationships
   - SQL DDL (Data Definition Language)

2. **JPA Entity Relationships**
   - Bidirectional mapping
   - Cascade operations
   - Fetch strategies

3. **Transaction Management**
   - ACID properties
   - Rollback behavior
   - Transaction isolation

4. **Database Initialization**
   - Manual database creation (PostgreSQL requirement)
   - SQL initialization scripts
   - Flyway migrations (optional)

5. **GUI Database Management**
   - pgAdmin web interface
   - Visual query execution
   - Database inspection and debugging

---

## Expected Outcomes

### **By the End of This Lab:**

**You will have:**
- ✅ A fully functional order-service microservice
- ✅ PostgreSQL database with proper schema
- ✅ JPA entities with relationships
- ✅ RESTful API endpoint for placing orders
- ✅ Integration tests with TestContainers
- ✅ Dockerized deployment with docker-compose
- ✅ Multi-database microservices architecture

**You will understand:**
- ✅ How to model relational data with JPA
- ✅ How entity relationships work in JPA
- ✅ How transactions ensure data consistency
- ✅ How to test with real databases using TestContainers
- ✅ How to orchestrate multiple databases with Docker Compose
- ✅ The differences between NoSQL and SQL approaches

**You will be prepared for:**
- ✅ Building more complex microservices
- ✅ Implementing inter-service communication
- ✅ Adding API gateways
- ✅ Implementing distributed transactions
- ✅ Working on enterprise microservices projects

---

## Lab Workflow Overview

### **Phase 1: Setup (30 minutes)**
1. Create order-service module with Spring Initializr
2. Configure dependencies
3. Setup package structure
4. Verify project builds successfully

### **Phase 2: Implementation (90 minutes)**
1. Create JPA entities (Order, OrderLineItem)
2. Define entity relationships
3. Create DTOs (OrderRequest, OrderLineItemDto)
4. Implement service layer with transaction management
5. Create repository interface
6. Implement REST controller

### **Phase 3: Database Configuration (45 minutes)**
1. Configure PostgreSQL connection
2. Setup PostgreSQL Docker container
3. Setup pgAdmin container (optional)
4. Create order-service database
5. Test connection and schema creation

### **Phase 4: Local Testing (45 minutes)**
1. Run order-service locally
2. Test with Postman
3. Verify data persistence
4. Debug any issues

### **Phase 5: Integration Testing (60 minutes)**
1. Add TestContainers dependencies
2. Configure PostgreSQL TestContainer
3. Write integration test for placeOrder()
4. Run tests and verify they pass

### **Phase 6: Containerization (90 minutes)**
1. Create multistage Dockerfile
2. Build Docker image
3. Update docker-compose.yml
4. Add initialization scripts
5. Deploy complete architecture
6. Test in containerized environment

### **Phase 7: Verification and Submission (30 minutes)**
1. Verify all services running
2. Test complete workflow
3. Review and clean code
4. Commit and push to repository
5. Document any customizations

**Total Time:** 6-7 hours (can be spread across multiple sessions)

---

## Success Criteria

Your lab is successful when:

- ✅ order-service starts without errors
- ✅ POST /api/order returns 201 Created
- ✅ Order data persists in PostgreSQL
- ✅ Order line items are saved with cascade
- ✅ Integration tests pass (green checkmarks)
- ✅ Docker image builds successfully
- ✅ All 6 containers run in docker-compose
- ✅ Services communicate through Docker network
- ✅ Data persists after container restart
- ✅ Code is clean and follows best practices
- ✅ Repository is updated with all changes

---

## Next Section

Continue to [02-setup-order-service.md](02-setup-order-service.md) to begin creating the order-service module.

---

**Note:** This lab builds directly upon concepts from previous weeks. Ensure you have completed Weeks 1-5 before proceeding.
