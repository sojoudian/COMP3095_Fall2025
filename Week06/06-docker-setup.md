# Docker Setup - PostgreSQL and pgAdmin

## Overview

In this section, you will set up PostgreSQL and pgAdmin as Docker containers. PostgreSQL will serve as the database for order-service, and pgAdmin (optional) provides a web-based GUI for database management.

---

## Step 1: Pull PostgreSQL Docker Image

### 1.1 Using IntelliJ Docker Plugin

**Open Docker Tool Window:**
1. Click **View → Tool Windows → Services** (or Alt+8)
2. Expand **Docker** node

**Pull PostgreSQL Image:**

**Option A: Using IntelliJ UI**
1. Right-click on **Docker** connection
2. Select **Pull Image...**
3. Enter image name: `postgres:latest`
4. Click **OK**

**Option B: Using Terminal**
```bash
docker pull postgres:latest
```

### 1.2 Verify Image Downloaded

```bash
docker images | grep postgres
```

**Expected Output:**
```
postgres    latest    a123b456c789    2 days ago    379MB
```

---

## Step 2: Create PostgreSQL Container

### 2.1 Create Container Using IntelliJ

**Steps:**
1. Right-click on `postgres:latest` image
2. Select **Create Container**

**Container Configuration:**

**General Tab:**
- **Container Name:** `postgres-container`
- **Image:** `postgres:latest`

**Environment Variables Tab:**

Add these environment variables:
```
POSTGRES_DB=order-service
POSTGRES_USER=admin
POSTGRES_PASSWORD=password
```

**Port Bindings Tab:**
```
Host Port: 5433
Container Port: 5432
```

3. Click **Run**

### 2.2 Create Container Using Command Line

**Alternative: Docker CLI**

```bash
docker run -d \
  --name postgres-container \
  -e POSTGRES_DB=order-service \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  -p 5433:5432 \
  postgres:latest
```

**Explanation:**
- `-d` - Run in detached mode (background)
- `--name` - Container name
- `-e` - Environment variable
- `-p` - Port mapping (host:container)

### 2.3 Verify Container is Running

```bash
docker ps | grep postgres
```

**Expected Output:**
```
CONTAINER ID   IMAGE             STATUS         PORTS                    NAMES
a1b2c3d4e5f6   postgres:latest   Up 2 minutes   0.0.0.0:5433->5432/tcp   postgres-container
```

### 2.4 View Container Logs

**In IntelliJ:**
1. Right-click on `postgres-container`
2. Select **Show Logs**

**Via Command Line:**
```bash
docker logs postgres-container
```

**Look for:**
```
database system is ready to accept connections
```

---

## Step 3: Verify Database Created

### 3.1 Connect to PostgreSQL Using psql

**Using Docker Exec:**
```bash
docker exec -it postgres-container psql -U admin -d order-service
```

### 3.2 Verify Database Exists

**Inside psql Shell:**

```sql
-- List all databases
\l
```

**Expected Output:**
```
     Name      | Owner | Encoding
---------------+-------+----------
 order-service | admin | UTF8
```

**Verify Connection:**
```sql
-- Check current database
SELECT current_database();

-- Check current user
SELECT current_user;

-- Exit psql
\q
```

---

## Step 4: Test Connection from Host Machine

### 4.1 Using IntelliJ Database Tool

**Add Data Source:**

1. Open Database Tool Window (View → Tool Windows → Database)
2. Click **+** button → **Data Source → PostgreSQL**
3. Configure Connection:

```
Name: order-service (local)
Host: localhost
Port: 5433
Database: order-service
User: admin
Password: password
```

4. Click **Download Driver Files** (if prompted)
5. Click **Test Connection** → Should show "Succeeded"
6. Click **OK**

---

## Step 5: Setup pgAdmin (Optional)

### 5.1 Pull pgAdmin Image

**Using IntelliJ:**
1. Right-click **Docker** connection
2. Select **Pull Image...**
3. Enter: `dpage/pgadmin4:latest`
4. Click **OK**

**Using Command Line:**
```bash
docker pull dpage/pgadmin4:latest
```

### 5.2 Create pgAdmin Container

**Using IntelliJ:**

1. Right-click `dpage/pgadmin4:latest` image
2. Select **Create Container**

**Configuration:**

**General:**
- **Container Name:** `pgadmin-container`

**Environment Variables:**
```
PGADMIN_DEFAULT_EMAIL=admin@admin.com
PGADMIN_DEFAULT_PASSWORD=admin
```

**Port Bindings:**
```
Host Port: 5050
Container Port: 80
```

**Using Command Line:**

```bash
docker run -d \
  --name pgadmin-container \
  -e PGADMIN_DEFAULT_EMAIL=admin@admin.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin \
  -p 5050:80 \
  dpage/pgadmin4:latest
```

### 5.3 Access pgAdmin Web Interface

**Open Browser:**
```
http://localhost:5050
```

**Login:**
- **Email:** `admin@admin.com`
- **Password:** `admin`

### 5.4 Add PostgreSQL Server to pgAdmin

**Add New Server:**

1. Right-click **Servers** → **Register → Server**

**General Tab:**
- **Name:** `order-service-local`

**Connection Tab:**
```
Host: host.docker.internal
Port: 5433
Maintenance Database: order-service
Username: admin
Password: password
```

**Save Password:** Check **"Save password?"**

Click **Save**

---

## Step 6: Run order-service Application

### 6.1 Start Application

**In IntelliJ:**
1. Open `OrderServiceApplication.java`
2. Click green play button
3. Select **Run 'OrderServiceApplication'**

**Via Command Line:**
```bash
cd order-service
./gradlew bootRun
```

### 6.2 Check Application Logs

**Look for:**
```
HikariPool-1 - Starting...
HikariPool-1 - Start completed.
Hibernate: create table if not exists orders (...)
Hibernate: create table if not exists order_line_items (...)
Started OrderServiceApplication in 5.321 seconds
```

### 6.3 Verify Tables Created

**Using pgAdmin:**
1. Right-click **Tables** → **Refresh**
2. You should see:
   - `orders`
   - `order_line_items`

**Using psql:**
```bash
docker exec -it postgres-container psql -U admin -d order-service -c "\dt"
```

**Expected Output:**
```
                 List of relations
 Schema |       Name        | Type  | Owner
--------+-------------------+-------+-------
 public | order_line_items  | table | admin
 public | orders            | table | admin
```

### 6.4 Test Actuator Health Endpoint

**Open Browser:**
```
http://localhost:8082/actuator/health
```

**Expected Response:**
```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL"
      }
    }
  }
}
```

---

## Troubleshooting

### Issue 1: Port Already in Use

**Error:**
```
Bind for 0.0.0.0:5433 failed: port is already allocated
```

**Solution: Change Host Port:**
```bash
docker run -d \
  --name postgres-container \
  -p 5434:5432 \
  postgres:latest
```

Update application.properties:
```properties
spring.datasource.url=jdbc:postgresql://localhost:5434/order-service
```

### Issue 2: Cannot Connect from Application

**Error:**
```
Connection refused: localhost:5433
```

**Checklist:**
1. ✅ Container is running: `docker ps`
2. ✅ Port mapping correct: `0.0.0.0:5433->5432/tcp`
3. ✅ Credentials match: admin/password
4. ✅ Database exists: `order-service`

### Issue 3: pgAdmin Cannot Connect

**Error:**
```
Unable to connect to server: Connection refused
```

**Solution:**
Use `host.docker.internal` instead of `localhost`:
- **Host:** `host.docker.internal`
- **Port:** `5433`

---

## Summary

### What You Completed:

**PostgreSQL Setup:**
- ✅ Pulled postgres:latest Docker image
- ✅ Created postgres-container
- ✅ Configured port mapping (5433:5432)
- ✅ Database `order-service` auto-created
- ✅ Verified database exists

**pgAdmin Setup (Optional):**
- ✅ Pulled dpage/pgadmin4:latest image
- ✅ Created pgadmin-container
- ✅ Accessed pgAdmin web interface (localhost:5050)
- ✅ Registered PostgreSQL server in pgAdmin

**Connection Verification:**
- ✅ Connected from IntelliJ Database Tool
- ✅ Connected from order-service application
- ✅ Verified Actuator health endpoint

**Schema Verification:**
- ✅ JPA created `orders` table automatically
- ✅ JPA created `order_line_items` table automatically
- ✅ Foreign key relationship established

### Configuration Summary:

**PostgreSQL Container:**
```
Container Name: postgres-container
Port Mapping: 5433:5432
Database: order-service
User: admin
Password: password
```

**pgAdmin Container:**
```
Container Name: pgadmin-container
Port Mapping: 5050:80
Email: admin@admin.com
Password: admin
```

---

## Next Steps

Continue to [07-testing.md](07-testing.md)

---

**Files Modified:**
- ✅ application.properties - Verified PostgreSQL connection settings

**Docker Resources Created:**
- ✅ postgres-container (PostgreSQL 16)
- ✅ pgadmin-container (pgAdmin 4)
- ✅ Database: order-service
- ✅ Tables: orders, order_line_items
