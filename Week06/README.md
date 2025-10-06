# Week 06: Order Service Implementation with PostgreSQL and JPA

## Overview

This week introduces building a second microservice called **order-service** that uses **PostgreSQL** (relational database) and **Spring Data JPA** for data persistence. This complements the existing product-service which uses MongoDB.

## Learning Materials

### Complete Guide Structure

This guide is organized into the following sections:

1. **[01-introduction.md](01-introduction.md)** - Lab overview, background, and objectives
2. **[02-setup-order-service.md](02-setup-order-service.md)** - Creating the order-service module with Spring Initializr
3. **[03-create-entities.md](03-create-entities.md)** - Building JPA entities (Order, OrderLineItem)
4. **[04-create-dtos-and-services.md](04-create-dtos-and-services.md)** - DTOs, Services, Repositories, and Controllers
5. **[05-database-configuration.md](05-database-configuration.md)** - PostgreSQL configuration and setup
6. **[06-docker-setup.md](06-docker-setup.md)** - Docker containers for PostgreSQL and pgAdmin
7. **[07-testing.md](07-testing.md)** - Integration testing with TestContainers and Postman
8. **[08-deployment.md](08-deployment.md)** - Dockerfile, Docker Compose, and deployment

### Topics Covered

1. **Spring Data JPA**
   - Entity modeling with `@Entity`, `@Table`, `@Id`
   - Entity relationships (`@OneToMany`, `@ManyToOne`)
   - Cascade operations (`CascadeType.ALL`)
   - Transaction management (`@Transactional`)

2. **PostgreSQL Integration**
   - Relational database configuration
   - Docker container setup
   - Database initialization
   - pgAdmin GUI setup

3. **Layered Architecture**
   - DTOs for request/response
   - Service layer with business logic
   - Repository layer for data access
   - Controller layer for REST endpoints

4. **Testing**
   - TestContainers for PostgreSQL
   - Integration testing with REST Assured
   - Postman API testing

5. **Containerization**
   - Multistage Dockerfile
   - Docker Compose orchestration
   - Multi-database architecture

6. **Developer Tools**
   - Spring Boot DevTools
   - LiveReload
   - Flyway migration (optional)

## Prerequisites

**Required:**
- ✅ Completed Week 2/3 (Product Service with MongoDB)
- ✅ Completed Week 4 (API Testing with TestContainers)
- ✅ Completed Week 5 (Docker Containerization)
- ✅ Docker Desktop installed and running
- ✅ IntelliJ IDEA with Docker plugin
- ✅ Postman installed

**Knowledge:**
- Understanding of REST APIs
- Basic SQL and relational databases
- Docker and Docker Compose
- Spring Boot basics

## Technology Stack

- **Java:** 21
- **Spring Boot:** 3.5.6
- **Spring Data JPA:** Latest
- **PostgreSQL:** Latest
- **Docker:** 20.10+
- **Docker Compose:** 2.x
- **TestContainers:** 1.21.3
- **Gradle:** Kotlin DSL
- **pgAdmin:** Latest (optional GUI)

## Architecture Overview

### Before This Week (Product Service Only):
```
┌──────────────────────────────────────────┐
│      microservices-network               │
│                                          │
│  ┌────────────────┐  ┌────────────────┐  │
│  │ product-service│  │    mongodb     │  │
│  │  (port 8084)   │→ │  (port 27017)  │  │
│  └────────────────┘  └────────────────┘  │
│          ↓                    ↓          │
│          └───────> mongo-express         │
│                    (port 8081)           │
└──────────────────────────────────────────┘
```

### After This Week (Product + Order Services):
```
┌───────────────────────────────────────────────────────┐
│              microservices-network                    │
│                                                       │
│  ┌────────────────┐  ┌────────────────┐               │
│  │ product-service│  │    mongodb     │               │
│  │  (port 8084)   │→ │  (port 27017)  │               │
│  └────────────────┘  └────────────────┘               │
│          ↓                    ↓                       │
│          └───────> mongo-express (port 8081)          │
│                                                       │
│  ┌────────────────┐  ┌────────────────┐               │
│  │  order-service │  │   postgres     │               │
│  │  (port 8082)   │→ │  (port 5432)   │               │
│  └────────────────┘  └────────────────┘               │
│          ↓                    ↓                       │
│          └───────> pgadmin (port 8888)                │
└───────────────────────────────────────────────────────┘
```

## Lab Structure

### Part 1: Service Creation (60-90 minutes)
- Create order-service module
- Build package structure
- Implement JPA entities with relationships
- Create DTOs, services, repositories, controllers

### Part 2: Database Setup (45-60 minutes)
- Configure PostgreSQL connection
- Setup PostgreSQL Docker container
- Setup pgAdmin container (optional)
- Create database and test connection

### Part 3: Testing (45-60 minutes)
- Run order-service locally
- Test with Postman
- Write integration tests with TestContainers
- Verify all CRUD operations

### Part 4: Containerization (60-90 minutes)
- Create multistage Dockerfile
- Update docker-compose.yml
- Add initialization scripts
- Deploy and test complete architecture

**Total estimated time:** 4-6 hours

## Key Differences: MongoDB vs PostgreSQL

| Aspect | Product Service (MongoDB) | Order Service (PostgreSQL) |
|--------|---------------------------|----------------------------|
| **Database Type** | NoSQL (Document) | SQL (Relational) |
| **Data Access** | Spring Data MongoDB | Spring Data JPA |
| **Entity Annotation** | `@Document` | `@Entity` |
| **ID Type** | `String` (ObjectId) | `Long` (auto-increment) |
| **Schema** | Schema-less | Schema-based (DDL) |
| **Relationships** | Embedded documents | JPA relationships |
| **Transactions** | Not emphasized | `@Transactional` |
| **Query Language** | MongoDB Query | SQL/JPQL |
| **Port** | 27017 | 5432 |

## Learning Outcomes

By the end of this week, you will be able to:

- ✅ Design and implement a JPA-based microservice
- ✅ Model relational entities with proper relationships
- ✅ Configure PostgreSQL with Docker
- ✅ Use pgAdmin for database management
- ✅ Write integration tests with TestContainers (PostgreSQL)
- ✅ Create optimized multistage Dockerfiles
- ✅ Orchestrate multi-database microservices with Docker Compose
- ✅ Apply transaction management in Spring
- ✅ Use Spring Boot DevTools for rapid development
- ✅ Test REST APIs comprehensively

## Quick Start Commands

```bash
# Navigate to project
cd /path/to/your/microservices-parent

# Start all services
docker-compose up -d --build

# View logs
docker-compose logs -f order-service

# Test order-service
curl -X POST http://localhost:8082/api/order \
  -H "Content-Type: application/json" \
  -d '{
    "orderLineItemDtoList": [
      {
        "skuCode": "sku_12334A",
        "price": 888.00,
        "quantity": 5
      }
    ]
  }'

# Stop all services
docker-compose down

# Clean everything
docker-compose down -v
docker rmi order-service:1.0
```

## File Structure

After completing this week, your project structure will be:

```
microservices-parent/
├── product-service/              (Existing - MongoDB)
│   ├── src/
│   ├── Dockerfile
│   └── build.gradle.kts
├── order-service/                (NEW - PostgreSQL)
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/ca/gbc/comp3095/orderservice/
│   │   │   │   ├── controller/
│   │   │   │   │   └── OrderController.java
│   │   │   │   ├── dto/
│   │   │   │   │   ├── OrderRequest.java
│   │   │   │   │   └── OrderLineItemDto.java
│   │   │   │   ├── model/
│   │   │   │   │   ├── Order.java
│   │   │   │   │   └── OrderLineItem.java
│   │   │   │   ├── repository/
│   │   │   │   │   └── OrderRepository.java
│   │   │   │   ├── service/
│   │   │   │   │   └── OrderService.java
│   │   │   │   └── OrderServiceApplication.java
│   │   │   └── resources/
│   │   │       ├── application.properties
│   │   │       └── application-docker.properties
│   │   └── test/
│   │       └── java/ca/gbc/comp3095/orderservice/
│   │           └── OrderServiceApplicationTests.java
│   ├── Dockerfile                (NEW)
│   └── build.gradle.kts
├── init/
│   ├── mongo/
│   │   └── docker-entrypoint-initdb.d/
│   │       └── mongo-init.js
│   └── postgres/                 (NEW)
│       └── docker-entrypoint-initdb.d/
│           └── init.sql
├── docker-compose.yml            (UPDATED)
├── build.gradle.kts
└── settings.gradle.kts           (UPDATED)
```

## Common Issues

| Issue | Solution |
|-------|----------|
| Port 8082 already in use | Change port in application.properties and docker-compose.yml |
| PostgreSQL connection refused | Wait 30 seconds for container initialization |
| Database not created | PostgreSQL requires manual creation or init script |
| TestContainers fails | Ensure Docker Desktop is running |
| Gradle build fails | Reload Gradle dependencies |
| Cascade not working | Check `@OneToMany(cascade = CascadeType.ALL)` |

See each section's troubleshooting guide for detailed solutions.

## Testing Checklist

After completing the lab, verify:

- [ ] order-service module created with all dependencies
- [ ] All entities, DTOs, services, repositories, controllers implemented
- [ ] PostgreSQL container running
- [ ] pgAdmin container running (optional)
- [ ] order-service database created
- [ ] Application starts without errors
- [ ] POST /api/order creates orders successfully (Postman: 201 Created)
- [ ] Data persists in PostgreSQL (verify via pgAdmin)
- [ ] Integration tests pass (TestContainers with PostgreSQL)
- [ ] Dockerfile builds successfully
- [ ] All containers running in docker-compose (6 services)
- [ ] order-service accessible via Docker network
- [ ] Data persists after container restart
- [ ] Code committed and pushed to repository

## Next Steps

After completing this week:

1. **Week 7 Preparation:** Inter-service communication (order-service calling product-service)
2. **Optional Enhancements:**
   - Add GET, PUT, DELETE endpoints to OrderController
   - Implement Flyway migrations for database versioning
   - Add validation annotations to DTOs
   - Implement custom exception handling
   - Add API documentation with Swagger/OpenAPI

## Additional Resources

- [Spring Data JPA Documentation](https://spring.io/projects/spring-data-jpa)
- [PostgreSQL Docker Hub](https://hub.docker.com/_/postgres)
- [pgAdmin Documentation](https://www.pgadmin.org/docs/)
- [TestContainers PostgreSQL Module](https://www.testcontainers.org/modules/databases/postgres/)
- [JPA Entity Relationships Guide](https://www.baeldung.com/jpa-one-to-many)
- [Transaction Management in Spring](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)

## Support

If you encounter issues:

1. Check the troubleshooting section in each guide
2. Review container logs: `docker-compose logs <service-name>`
3. Verify all files match the guide examples
4. Ensure Docker Desktop has sufficient resources (4GB+ RAM)
5. Ask instructor during office hours

## Version History

- **v1.0** (Week 6, Fall 2025) - Initial release
  - Order service with PostgreSQL and JPA
  - Complete entity relationship modeling
  - TestContainers integration tests
  - Docker Compose multi-database architecture
  - Complete code examples with explanations

---

**Author:** Maziar Sojoudian
**Course:** COMP3095 Fall 2025
**Institution:** George Brown College
