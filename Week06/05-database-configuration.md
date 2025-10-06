# Database Configuration

## Overview

In this section, you will configure the PostgreSQL database connection for the order-service. You'll create two configuration files:
1. **application.properties** - For local development (localhost)
2. **application-docker.properties** - For Docker containerized environment

---

## Understanding PostgreSQL Configuration

### Connection String Format

**JDBC URL Format:**
```
jdbc:postgresql://[host]:[port]/[database]
```

**Examples:**
- Local: `jdbc:postgresql://localhost:5432/order-service`
- Docker: `jdbc:postgresql://postgres:5432/order-service`

**Components:**
- `jdbc:postgresql://` - JDBC protocol for PostgreSQL
- `localhost` or `postgres` - Database host
- `5432` - PostgreSQL default port
- `order-service` - Database name

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

### 1.3 Understanding Each Configuration

#### **Application Configuration**

```properties
spring.application.name=order-service
```
- Sets application name
- Used in logs and monitoring
- Identifies service in distributed systems

```properties
server.port=8082
```
- HTTP port for the application
- Different from product-service (8084)
- Avoids port conflicts

#### **PostgreSQL DataSource Configuration**

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/order-service
```
- Connection URL to PostgreSQL
- `localhost:5432` - Local PostgreSQL instance
- `order-service` - Database name

```properties
spring.datasource.username=admin
spring.datasource.password=password
```
- Database credentials
- Must match PostgreSQL user created later
- **Security Note:** Use environment variables in production

```properties
spring.datasource.driver-class-name=org.postgresql.Driver
```
- JDBC driver class
- Tells Spring which driver to use
- Usually auto-detected, but explicit is better

#### **JPA/Hibernate Configuration**

```properties
spring.jpa.hibernate.ddl-auto=update
```
- Controls schema generation
- `update` - Updates schema if needed, preserves data
- Other options:
  - `create` - Drops and creates schema on startup (loses data)
  - `create-drop` - Creates on startup, drops on shutdown
  - `validate` - Only validates schema, no changes
  - `none` - No schema management

**Development:** `update`
**Production:** `validate` or `none` (use Flyway for migrations)

```properties
spring.jpa.show-sql=true
```
- Prints SQL statements to console
- Useful for debugging
- Disable in production for performance

```properties
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```
- SQL dialect for PostgreSQL
- Optimizes SQL for PostgreSQL specific features
- Auto-detected, but explicit is better

```properties
spring.jpa.properties.hibernate.format_sql=true
```
- Pretty-prints SQL in console
- Makes SQL easier to read
- Useful for debugging

#### **Logging Configuration**

```properties
logging.level.ca.gbc.comp3095=INFO
```
- Sets log level for application packages
- `INFO` - General informational messages
- Other levels: `DEBUG`, `TRACE`, `WARN`, `ERROR`

```properties
logging.level.org.hibernate.SQL=DEBUG
```
- Shows SQL statements in logs
- Helpful for debugging queries

```properties
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```
- Shows parameter values in SQL
- Example: `binding parameter [1] as [VARCHAR] - [sku_12334A]`
- Very detailed, use for debugging only

#### **Actuator Configuration**

```properties
management.endpoints.web.exposure.include=health,info,metrics
```
- Exposes actuator endpoints
- `health` - Health check endpoint
- `info` - Application information
- `metrics` - Application metrics

```properties
management.endpoint.health.show-details=always
```
- Shows detailed health information
- Includes database connection status
- Useful for monitoring

**Actuator Endpoints:**
- `http://localhost:8082/actuator/health` - Health status
- `http://localhost:8082/actuator/info` - App info
- `http://localhost:8082/actuator/metrics` - Metrics

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

### 2.3 Key Differences from Local Configuration

**The ONLY Difference:**
```properties
# Local (application.properties)
spring.datasource.url=jdbc:postgresql://localhost:5432/order-service

# Docker (application-docker.properties)
spring.datasource.url=jdbc:postgresql://postgres:5432/order-service
                                       ^^^^^^^^
                                    Service name!
```

**Why "postgres" instead of "localhost"?**
- Docker containers use Docker's internal DNS
- Service names (from docker-compose.yml) resolve to container IPs
- `postgres` is the service name we'll define in docker-compose.yml
- Containers on same network can communicate by service name

**Docker Network Resolution:**
```
order-service container → "postgres:5432" → Docker DNS
                                          ↓
                          PostgreSQL container (IP: 172.18.0.3)
```

---

## Step 3: Understanding Spring Profiles

### What are Spring Profiles?

**Definition:**
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

**Examples:**
- `application-dev.properties` - Development settings
- `application-prod.properties` - Production settings
- `application-docker.properties` - Docker settings

### Activation

**Local (No Profile):**
```bash
# Run from IntelliJ or:
./gradlew bootRun
# Uses: application.properties only
```

**Docker (docker profile):**
```bash
# Set environment variable:
export SPRING_PROFILES_ACTIVE=docker
./gradlew bootRun
# Uses: application.properties + application-docker.properties
```

**In Dockerfile:**
```dockerfile
ENV SPRING_PROFILES_ACTIVE=docker
```

**In docker-compose.yml:**
```yaml
environment:
  SPRING_PROFILES_ACTIVE: docker
```

---

## Step 4: Configuration Comparison

### Side-by-Side Comparison

| Configuration | Local | Docker |
|--------------|-------|--------|
| **File** | application.properties | application-docker.properties |
| **Port** | 8082 | 8082 |
| **Database Host** | localhost | postgres (service name) |
| **Database Port** | 5432 | 5432 |
| **Username** | admin | admin |
| **Password** | password | password |
| **Schema Management** | update | update |
| **Show SQL** | true | true |

**Only Difference:** Database host (localhost vs postgres)

---

## Step 5: Environment-Specific Settings

### Development vs Production

**Development (Local):**
```properties
# Show SQL for debugging
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Auto-update schema
spring.jpa.hibernate.ddl-auto=update

# Detailed logging
logging.level.org.hibernate.SQL=DEBUG
```

**Production (Recommended):**
```properties
# Don't show SQL (performance)
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=false

# No automatic schema changes
spring.jpa.hibernate.ddl-auto=validate

# Less verbose logging
logging.level.org.hibernate.SQL=WARN

# Externalize credentials
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```

---

## Step 6: Connection Pool Configuration (Optional)

### HikariCP Configuration

Spring Boot uses HikariCP by default. You can configure it:

```properties
# Connection Pool Configuration (Optional)
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

**Explanation:**
- `maximum-pool-size=10` - Max 10 connections to database
- `minimum-idle=5` - Keep 5 idle connections ready
- `connection-timeout=30000` - 30 seconds to get connection
- `idle-timeout=600000` - 10 minutes before idle connection closed
- `max-lifetime=1800000` - 30 minutes max connection lifetime

**For this lab, default settings are sufficient.**

---

## Step 7: Verify Configuration

### 7.1 Check Both Files Exist

```
order-service/src/main/resources/
├── application.properties              ✅
└── application-docker.properties       ✅
```

### 7.2 Verify No Typos

**Common Mistakes:**
- ❌ `spring.datasource.ur` (missing 'l')
- ❌ `postgre` (missing 'sql')
- ❌ `spring.jpa.hiberate` (missing 'n')
- ❌ Wrong port: 5433 instead of 5432

**Correct:**
- ✅ `spring.datasource.url`
- ✅ `postgresql`
- ✅ `spring.jpa.hibernate`
- ✅ Port: 5432

---

## Summary

### What You Configured:

**application.properties (Local):**
- ✅ Server port: 8082
- ✅ PostgreSQL connection: localhost:5432
- ✅ Database credentials: admin/password
- ✅ JPA schema auto-update
- ✅ SQL logging enabled
- ✅ Hibernate dialect: PostgreSQL
- ✅ Actuator endpoints exposed

**application-docker.properties (Docker):**
- ✅ Same as local except:
- ✅ Database host: postgres (service name)

### Key Concepts:

- ✅ JDBC connection strings
- ✅ Spring profiles for environment-specific config
- ✅ JPA/Hibernate configuration
- ✅ Schema management with ddl-auto
- ✅ SQL logging for debugging
- ✅ Actuator for monitoring

### Configuration Files Comparison:

**Similarities:**
- Port, credentials, JPA settings, logging

**Difference:**
- Database host: `localhost` vs `postgres`

### Why Two Files?

**Scenario:**
1. Develop locally → Use application.properties → Connect to localhost
2. Deploy to Docker → Use application-docker.properties → Connect to postgres container
3. No code changes needed → Just activate profile

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
