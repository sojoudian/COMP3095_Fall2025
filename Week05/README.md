# Week 05: Docker Containerization

## Overview

This week introduces **Docker containerization** for the Product Service microservice. Students learn to package applications into portable containers that run consistently across any environment.

## Learning Materials

### Main Guide
- **[docker-containerization-guide.md](docker-containerization-guide.md)** - Complete step-by-step implementation guide (comprehensive, 2,000+ lines)

### Topics Covered

1. **Docker Fundamentals**
   - Images vs Containers
   - Docker architecture
   - Container lifecycle

2. **Multistage Dockerfiles**
   - Build stage (JDK)
   - Runtime stage (JRE)
   - Image optimization

3. **Docker Compose**
   - Multi-container orchestration
   - Service dependencies
   - Networks and volumes

4. **MongoDB in Docker**
   - Container configuration
   - Authentication setup
   - Data persistence
   - Initialization scripts

5. **Spring Boot Configuration**
   - Environment-specific properties
   - Spring Profiles
   - Container networking

6. **Testing Containerized Applications**
   - Postman API testing
   - MongoDB shell verification
   - Container debugging

## Prerequisites

**Required:**
- ✅ Docker Desktop installed and running
- ✅ Completed Week 2/3 (Product Service)
- ✅ Completed Week 4 (Testing)
- ✅ Postman installed

**Knowledge:**
- Basic command line usage
- Understanding of REST APIs
- Spring Boot application structure

## Lab Structure

### Part 1: Dockerfile (45-60 minutes)
- Understand multistage builds
- Create Dockerfile
- Build Docker image

### Part 2: Docker Compose (45-60 minutes)
- Create docker-compose.yml
- Configure services (product-service, mongodb, mongo-express)
- Create MongoDB initialization scripts

### Part 3: Configuration (30 minutes)
- Update application properties
- Create Docker-specific profile
- Understand local vs Docker settings

### Part 4: Run and Test (60-90 minutes)
- Clean existing containers
- Start Docker Compose network
- Test all CRUD operations with Postman
- Verify data in MongoDB

**Total estimated time:** 3-4 hours

## Key Files Created

Students will create these files:

```
microservices-parent/
├── docker-compose.yml                    (New - 130 lines)
├── init/
│   └── mongo/
│       └── docker-entrypoint-initdb.d/
│           └── mongo-init.js             (New - 30 lines)
└── product-service/
    ├── Dockerfile                         (New - 50 lines)
    └── src/main/resources/
        ├── application.properties         (Updated)
        └── application-docker.properties  (New)
```

## Learning Outcomes

By the end of this lab, students will be able to:

- ✅ Create optimized multistage Dockerfiles for Java applications
- ✅ Write docker-compose.yml files for multi-container applications
- ✅ Configure MongoDB with authentication in containers
- ✅ Manage Docker networks and volumes for data persistence
- ✅ Use Spring Profiles for environment-specific configuration
- ✅ Debug and troubleshoot containerized applications
- ✅ Test REST APIs in containerized environments

## Technology Stack

- **Docker:** 20.10+
- **Docker Compose:** 2.x
- **Base Images:** eclipse-temurin:21-jdk, eclipse-temurin:21-jre
- **MongoDB:** latest (7.x)
- **Mongo Express:** latest
- **Spring Boot:** 3.5.6
- **Java:** 21

## Architecture Diagram

```
┌─────────────────────────────────────────┐
│      microservices-network (bridge)     │
│                                          │
│  ┌────────────────┐  ┌────────────────┐ │
│  │ product-service│  │    mongodb     │ │
│  │  (port 8084)   │→ │  (port 27017)  │ │
│  └────────────────┘  └────────────────┘ │
│          ↓                    ↓          │
│          └───────> mongo-express        │
│                    (port 8081)          │
└─────────────────────────────────────────┘
              ↓
     mongodb-data (volume)
```

## Quick Start Commands

```bash
# Build and start all services
cd microservices-parent
docker-compose up -d --build

# View logs
docker-compose logs -f

# Stop services
docker-compose down

# Clean everything
docker-compose down -v
docker rmi product-service:1.0
```

## Common Issues

| Issue | Solution |
|-------|----------|
| Port already in use | Change port in docker-compose.yml or kill process using port |
| MongoDB connection refused | Wait 30 seconds for MongoDB initialization |
| Authentication failed | Verify credentials match in all config files |
| Gradle build fails | Test locally first: `./gradlew clean build` |
| Changes not reflected | Rebuild: `docker-compose up -d --build` |

See [Troubleshooting section](docker-containerization-guide.md#troubleshooting) in main guide for detailed solutions.

## Testing Checklist

After completing the lab, verify:

- ✅ Three containers running (product-service, mongodb, mongo-express)
- ✅ POST creates products (201 Created)
- ✅ GET retrieves all products (200 OK)
- ✅ GET by ID retrieves single product (200 OK)
- ✅ PUT updates product (204 No Content)
- ✅ DELETE removes product (204 No Content)
- ✅ Data persists after `docker-compose restart`
- ✅ Can view data in Mongo Express (http://localhost:8081)
- ✅ Can query data with mongosh

## Next Steps

After completing this lab:

1. **Week 6 Preparation:** Spring Security and Authentication
2. **Optional Enhancements:**
   - Add health checks to services
   - Implement resource limits
   - Create backup/restore scripts
   - Add logging configuration

## Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Spring Boot Docker Guide](https://spring.io/guides/gs/spring-boot-docker/)
- [MongoDB Docker Hub](https://hub.docker.com/_/mongo)
- [Eclipse Temurin Images](https://hub.docker.com/_/eclipse-temurin)

## Support

If you encounter issues:

1. Check the [Troubleshooting section](docker-containerization-guide.md#troubleshooting)
2. Review container logs: `docker-compose logs <service-name>`
3. Verify all files match the guide examples
4. Ask instructor during office hours
5. Check Docker Desktop for visual status

## Version History

- **v1.0** (Week 5, Fall 2025) - Initial release
  - Multistage Dockerfile implementation
  - Docker Compose with 3 services
  - MongoDB authentication and persistence
  - Complete CRUD testing guide

---

**Note:** This guide follows the same comprehensive format as Week 2/3 and Week 4 guides, providing complete code examples, detailed explanations, troubleshooting, and verification steps.