# Database Configuration

## Overview

In this section, you will configure the PostgreSQL database connection for the order-service. You'll create two configuration files:
1. **application.properties** - For local development (localhost)
2. **application-docker.properties** - For Docker containerized environment

---

## Step 1: Configure application.properties (Local Development)

### 1.1 Open application.properties

**Location:** `order-service/src/main/resources/application.properties`

### 1.2 Add Configuration

**Complete application.properties:**

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
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.format_sql=true

# Logging Configuration
logging.level.ca.gbc.comp3095=INFO
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE

# Actuator Configuration
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=always
```

---

## Step 2: Create application-docker.properties

### 2.1 Create File

**Location:** `order-service/src/main/resources/application-docker.properties`

**Steps:**
1. Right-click on `resources` folder
2. Select **New → File**
3. Name: `application-docker.properties`
4. Click **OK**

### 2.2 Add Docker Configuration

**Complete application-docker.properties:**

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
spring.datasource.url=jdbc:postgresql://postgres:5432/order-service
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA/Hibernate Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.format_sql=true

# Logging Configuration
logging.level.ca.gbc.comp3095=INFO
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE

# Actuator Configuration
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=always
```

---

## Summary

### What You Configured:

**Spring Profiles:**
- Way to configure different settings for different environments
- Activated with `SPRING_PROFILES_ACTIVE` environment variable

**How It Works:**
```
application.properties              ← Base configuration (always loaded)
    +
application-{profile}.properties   ← Profile-specific (overrides base)
    =
Final Configuration
```

**Note:** We'll run the application in the next section after PostgreSQL is set up.

### Configuration Comparison:

| Configuration | Local | Docker |
|--------------|-------|--------|
| **File** | application.properties | application-docker.properties |
| **Port** | 8082 | 8082 |
| **Database Host** | localhost | postgres (service name) |
| **Database Port** | 5433 (mapped from container) | 5432 (internal port) |
| **Username** | admin | admin |
| **Password** | password | password |
| **Schema Management** | update | update |
| **Show SQL** | true | true |

**Key Differences:**
- Database host: `localhost` vs `postgres`
- Database port: `5433` (host) vs `5432` (container)

### Why Two Files?

**Purpose:**
- **Local development (IntelliJ):** Connect to PostgreSQL on `localhost:5433`
- **Docker deployment:** Connect to postgres container using service name `postgres:5432`

**Benefit:**
- No code changes needed when switching environments
- Automatically select correct configuration based on profile

---

## Troubleshooting

### Issue 1: Application Won't Start - DataSource Not Configured

**Error:**
```
Failed to configure a DataSource: 'url' attribute is not specified
```

**Solution:**
- Verify application.properties exists
- Check datasource.url is correctly set
- Ensure no typos in property names

### Issue 2: Cannot Connect to PostgreSQL

**Error:**
```
Connection refused: localhost:5432
```

**Solution:**
- PostgreSQL container not running yet
- We'll start it in the next section
- This is expected at this stage

### Issue 3: Wrong Database Name

**Error:**
```
database "order-service" does not exist
```

**Solution:**
- Database must be created manually in PostgreSQL
- We'll create it in the next section
- PostgreSQL doesn't auto-create databases like MongoDB

### Issue 4: Authentication Failed

**Error:**
```
FATAL: password authentication failed for user "admin"
```

**Solution:**
- Username/password doesn't match PostgreSQL
- We'll configure this when creating PostgreSQL container
- Ensure credentials match in docker setup

---

## Next Steps

You have successfully configured the database connection settings. Next, you will:
1. Setup PostgreSQL Docker container
2. Setup pgAdmin Docker container (optional GUI)
3. Create the order-service database
4. Test the connection

Continue to [06-docker-setup.md](06-docker-setup.md)

---

**Files Created/Modified:**
- ✅ `application.properties` - Local development configuration
- ✅ `application-docker.properties` - Docker environment configuration

**Configuration Summary:**
- ✅ Port 8082 for order-service
- ✅ PostgreSQL connection configured
- ✅ JPA schema auto-update enabled
- ✅ SQL logging enabled for debugging
- ✅ Actuator endpoints configured
- ✅ Spring profiles setup for environment switching
