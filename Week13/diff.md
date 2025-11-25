# COMPREHENSIVE COMPARISON REPORT

## Week6.2 (Source of Truth) vs P11 (Target)

---

## 1. CRITICAL MISSING FILES

### 1.1 Missing Docker Standalone Compose Files

**Location:** `docker/standalone/postgres/inventory-service/docker-compose-single.yml`
**Issue:** File exists in week6.2 but missing in p11
**Week6.2 has:** Standalone docker-compose for inventory-service postgres
**P11 has:** MISSING
**Action needed:** Copy file from week6.2

**Location:** `docker/standalone/postgres/order-service/docker-compose-single.yml`
**Issue:** File exists in week6.2 but missing in p11
**Week6.2 has:** Standalone docker-compose for order-service postgres
**P11 has:** MISSING
**Action needed:** Copy file from week6.2

### 1.2 InventoryClientStub Location Issue

**Location:** `order-service/src/test/java/ca/gbc/comp3095/orderservice/stubs/InventoryClientStub.java`
**Issue:** File is in wrong directory in p11
**Week6.2 has:** File in `src/test/java/` directory
**P11 has:** File in `src/main/java/` directory (WRONG LOCATION)
**Action needed:** Move from src/main/java to src/test/java

---

## 2. CONFIGURATION DIFFERENCES

### 2.1 Order Service - application.properties

**Location:** `order-service/src/main/resources/application.properties`

| Property | Week6.2 | P11 | Issue |
|----------|---------|-----|-------|
| **Port** | `localhost:5433` | `localhost:5432` | Different ports |
| **order-service.version** | At line 3 | At line 36 (Week 12 comment) | Different location |
| **spring.jpa.database-platform** | Present | Missing | P11 missing property |
| **Logging config** | Absent | Present (INFO/DEBUG/TRACE) | P11 has extra |
| **Actuator config** | Absent | Present | P11 has extra |
| **show-sql/format_sql** | Absent | Present | P11 has extra |

**Action needed:** Change port from 5432 to 5433 to match week6.2

---

### 2.2 Order Service - application-docker.properties

**Location:** `order-service/src/main/resources/application-docker.properties`

| Property | Week6.2 | P11 | Issue |
|----------|---------|-----|-------|
| **inventory.service.url** | `http://inventory-service` (NO PORT!) | `http://inventory-service:8083` | **CRITICAL: Week6.2 missing port** |
| **Flyway config** | ABSENT | Present (Week 13) | P11 has extra Flyway config |
| **spring.jpa.database-platform** | Present | Missing | P11 missing |
| **Logging/Actuator** | Absent | Present | P11 has extra |

**Action needed:**
- **IMPORTANT:** Remove `:8083` from inventory.service.url to match week6.2 (or week6.2 has a bug)
- Remove Flyway configuration (week6.2 doesn't have it)

---

### 2.3 Inventory Service - application-docker.properties

**Location:** `inventory-service/src/main/resources/application-docker.properties`

| Property | Week6.2 | P11 | Issue |
|----------|---------|-----|-------|
| **Port** | `:5434` | `:5433` | Different ports |
| **Database name** | `inventory_service` (underscore) | `inventory-service` (hyphen) | **CRITICAL BUG IN P11!** |
| **Flyway config** | ABSENT | Present (Week 13) | P11 has extra |
| **spring.jpa.database-platform** | Present | Missing | P11 missing |

**Action needed:**
- **CRITICAL:** Change database from `inventory-service` to `inventory_service` (must use underscore!)
- Change port from 5433 to 5434
- Remove Flyway configuration (week6.2 doesn't have it)

---

### 2.4 Docker Compose - Database Names

**Location:** `docker-compose.yml`

| Service | Week6.2 SPRING_DATASOURCE_URL | P11 SPRING_DATASOURCE_URL | Issue |
|---------|-------------------------------|---------------------------|-------|
| **inventory** | `postgres-inventory/inventory-service` (hyphen) | `postgres-inventory/inventory_service` (underscore) | **MISMATCH!** |
| **order** | `postgres-order/order-service` (hyphen) | `postgres-order/order_service` (underscore) | **MISMATCH!** |

**Action needed:**
- Week6.2 uses hyphens in docker-compose but underscores in properties files - this is inconsistent!
- P11 uses underscores everywhere (more consistent)
- **Recommendation:** Use underscores everywhere (database names cannot have hyphens)

---

### 2.5 Order Service - build.gradle.kts

**Location:** `order-service/build.gradle.kts`

| Property | Week6.2 | P11 | Issue |
|----------|---------|-----|-------|
| **Spring Boot version** | 3.4.4 | 3.5.6 | P11 using newer version |
| **Spring Cloud BOM** | 2024.0.1 | 2025.0.0 | P11 using newer version |
| **Group** | `ca.gbc` | `ca.gbc.comp3095` | Different package structure |
| **Duplicate dependency** | - | `spring-cloud-starter-contract-stub-runner` appears twice | **BUG IN P11!** |
| **TestContainers** | Minimal | Extended with junit-jupiter | P11 has more |

**Action needed:**
- Remove duplicate `spring-cloud-starter-contract-stub-runner` dependency (line 59)
- Consider downgrading Spring Boot to 3.4.4 to match week6.2
- Consider downgrading Spring Cloud to 2024.0.1 to match week6.2

---

### 2.6 Order Service Test - application.properties

**Location:** `order-service/src/test/resources/application.properties`

| Property | Week6.2 | P11 | Issue |
|----------|---------|-----|-------|
| **Content** | Only `inventory.service.url` | Has Testcontainers datasource config | P11 has extra |

**Action needed:** Remove Testcontainers configuration (Week 13 additions not in week6.2)

---

## 3. CODE DIFFERENCES

### 3.1 Extra Classes in P11 (Should NOT Exist)

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/dto/OrderLineItemDto.java`
**Issue:** Extra class that doesn't exist in week6.2
**Week6.2 has:** Does not exist
**P11 has:** File present
**Action needed:** DELETE this file (not part of simplified model)

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/model/OrderLineItem.java`
**Issue:** Extra class that doesn't exist in week6.2
**Week6.2 has:** Does not exist
**P11 has:** File present
**Action needed:** DELETE this file (not part of simplified model)

### 3.2 Extra Test Configuration Classes in P11

**Location:** `inventory-service/src/test/java/ca/gbc/comp3095/inventoryservice/TestcontainersConfiguration.java`
**Issue:** Extra test class
**Week6.2 has:** Does not exist
**P11 has:** File present
**Action needed:** Keep or remove (not critical, extra test infrastructure)

**Location:** `inventory-service/src/test/java/ca/gbc/comp3095/inventoryservice/TestInventoryServiceApplication.java`
**Issue:** Extra test class
**Week6.2 has:** Does not exist
**P11 has:** File present
**Action needed:** Keep or remove (not critical, extra test infrastructure)

### 3.3 Package Name Differences

**All Java files:**
- **Week6.2:** Uses package `ca.gbc.{servicename}` (e.g., `ca.gbc.orderservice`)
- **P11:** Uses package `ca.gbc.comp3095.{servicename}` (e.g., `ca.gbc.comp3095.orderservice`)
- **Issue:** Structural difference, not a bug
- **Action needed:** Decide on standard (both are valid)

### 3.4 Settings Files in P11

**Locations:**
- `api-gateway/settings.gradle.kts`
- `inventory-service/settings.gradle.kts`
- `order-service/settings.gradle.kts`
- `product-service/settings.gradle.kts`

**Issue:** These individual service settings files exist in p11 but not in week6.2
**Week6.2 has:** Only root `settings.gradle.kts`
**P11 has:** Individual service settings files
**Action needed:** Remove individual service settings files (use only root settings.gradle.kts)

---

## 4. DATABASE SCHEMA DIFFERENCES

### 4.1 All Flyway Migrations Match

✅ `order-service/src/main/resources/db/migration/V1__init.sql` - **MATCHES** (t_orders table with simplified schema)
✅ `inventory-service/src/main/resources/db/migration/V1__init.sql` - **MATCHES**
✅ `inventory-service/src/main/resources/db/migration/V2__add_inventory.sql` - **MATCHES**

**No action needed** for migration files.

---

## 5. DOCKER/INFRASTRUCTURE DIFFERENCES

### 5.1 Extra Init Script in P11

**Location:** `init/postgres/docker-entrypoint-initdb.d/init.sql`
**Issue:** Extra file in p11
**Week6.2 has:** Does not exist
**P11 has:** File present
**Action needed:** Check if used, likely safe to remove

---

## PRIORITY ACTION SUMMARY

### **CRITICAL (Must Fix):**

1. ✅ **DONE:** Move InventoryClientStub from src/main to src/test
2. ⚠️ **CRITICAL BUG:** inventory-service application-docker.properties database name must be `inventory_service` not `inventory-service`
3. ⚠️ **VERIFY:** docker-compose.yml database names have hyphens in week6.2 but should use underscores (database naming constraint)

### **HIGH Priority:**

4. Remove duplicate dependency in order-service build.gradle.kts
5. Delete OrderLineItemDto.java and OrderLineItem.java (not in week6.2)
6. Remove individual service settings.gradle.kts files
7. Fix port mismatches (5432→5433 for order, 5433→5434 for inventory)

### **MEDIUM Priority:**

8. Remove Flyway configuration from application-docker.properties (both services) if matching week6.2 exactly
9. Remove Testcontainers config from order-service test application.properties
10. Verify inventory.service.url port (week6.2 missing :8083 - likely a bug in week6.2)

### **LOW Priority (Enhancements in P11):**

11. Extra logging/actuator config (beneficial, can keep)
12. Extra test infrastructure classes (beneficial, can keep)
13. Newer Spring Boot/Cloud versions (beneficial, can keep or align with week6.2)
