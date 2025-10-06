# Docker Setup - PostgreSQL and pgAdmin

## Overview

In this section, you will set up PostgreSQL and pgAdmin as Docker containers. PostgreSQL will serve as the database for order-service, and pgAdmin (optional) provides a web-based GUI for database management.

---

## Understanding Docker Setup

### Why Docker for Databases?

**Benefits:**
- ✅ **Consistent Environment** - Same database version everywhere
- ✅ **Easy Setup** - No manual installation required
- ✅ **Isolation** - Database runs in isolated container
- ✅ **Portability** - Works on any machine with Docker
- ✅ **Version Control** - Easy to switch PostgreSQL versions
- ✅ **Clean Removal** - Delete container to remove completely

**Traditional Installation vs Docker:**

| Aspect | Traditional Install | Docker Container |
|--------|-------------------|------------------|
| **Setup Time** | 30-60 minutes | 2-5 minutes |
| **Configuration** | Complex, OS-specific | Simple, unified |
| **Multiple Versions** | Difficult | Easy |
| **Cleanup** | Manual uninstall | Delete container |
| **Portability** | OS-dependent | Cross-platform |

---

## Docker Network Architecture

### How Containers Communicate

**Docker Network:**
```
Docker Network: microservices-network
├── order-service (container)
│   └── Connects to: postgres:5432
├── postgres (container)
│   └── Port: 5432 (internal)
│   └── Exposed: 5433 → 5432 (host access)
└── pgadmin (container)
    └── Port: 80 (internal)
    └── Exposed: 5050 → 80 (host access)
```

**Connection Patterns:**

**From Host Machine (Your Computer):**
```
localhost:5433 → postgres container (port 5432)
localhost:5050 → pgAdmin container (port 80)
```

**From order-service Container:**
```
postgres:5432 → postgres container
(Uses service name, not localhost!)
```

**Key Concept:**
- Inside Docker network: Use service names (`postgres`, `order-service`)
- From host machine: Use `localhost` with mapped ports

---

## Step 1: Pull PostgreSQL Docker Image

### 1.1 Using IntelliJ Docker Plugin

**Open Docker Tool Window:**
1. Click **View → Tool Windows → Services** (or Alt+8)
2. Expand **Docker** node
3. You should see your Docker connection

**Pull PostgreSQL Image:**

**Option A: Using IntelliJ UI**
1. Right-click on **Docker** connection
2. Select **Pull Image...**
3. Enter image name: `postgres:latest`
4. Click **OK**
5. Wait for download to complete

**Option B: Using Terminal**
```bash
docker pull postgres:latest
```

### 1.2 Verify Image Downloaded

**Check in IntelliJ:**
1. Expand **Docker → Images**
2. Look for `postgres:latest`
3. Note the image size (approximately 400MB)

**Check via Command Line:**
```bash
docker images | grep postgres
```

**Expected Output:**
```
postgres    latest    a123b456c789    2 days ago    379MB
```

---

## Step 2: Create PostgreSQL Container

### 2.1 Understanding PostgreSQL Configuration

**Environment Variables:**

| Variable | Value | Purpose |
|----------|-------|---------|
| `POSTGRES_DB` | `order-service` | Creates database on startup |
| `POSTGRES_USER` | `admin` | Database username |
| `POSTGRES_PASSWORD` | `password` | Database password |

**Port Mapping:**
- Container Port: `5432` (PostgreSQL default)
- Host Port: `5433` (to avoid conflict with local PostgreSQL if installed)
- Format: `host_port:container_port`

**Why 5433 instead of 5432?**
- You might have PostgreSQL already installed locally on 5432
- Using 5433 avoids port conflicts
- Container still uses 5432 internally

### 2.2 Create Container Using IntelliJ

**Step-by-Step:**

1. **Right-click on `postgres:latest` image**
2. **Select "Create Container"**

**Container Configuration:**

**General Tab:**
- **Container Name:** `postgres-container`
- **Image:** `postgres:latest`

**Environment Variables Tab:**

Add these environment variables (click + button for each):
```
POSTGRES_DB=order-service
POSTGRES_USER=admin
POSTGRES_PASSWORD=password
```

**Port Bindings Tab:**

Add port binding:
```
Host Port: 5433
Container Port: 5432
Protocol: tcp
```

**Command Line Tab (Optional):**

If you want to see logs immediately:
```
-c log_statement=all
```

3. **Click "Run"**

### 2.3 Create Container Using Command Line

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
- `postgres:latest` - Image to use

### 2.4 Verify Container is Running

**Check in IntelliJ:**
1. Expand **Docker → Containers**
2. Look for `postgres-container` with green icon
3. Status should show "Up" with timestamp

**Check via Command Line:**
```bash
docker ps | grep postgres
```

**Expected Output:**
```
CONTAINER ID   IMAGE             STATUS         PORTS                    NAMES
a1b2c3d4e5f6   postgres:latest   Up 2 minutes   0.0.0.0:5433->5432/tcp   postgres-container
```

### 2.5 View Container Logs

**In IntelliJ:**
1. Right-click on `postgres-container`
2. Select **Show Logs**
3. Look for this message:

```
PostgreSQL init process complete; ready for start up.
database system is ready to accept connections
```

**Via Command Line:**
```bash
docker logs postgres-container
```

**Healthy Logs Should Show:**
```
2024-01-15 10:30:45.123 UTC [1] LOG:  database system is ready to accept connections
2024-01-15 10:30:45.234 UTC [45] LOG:  database "order-service" created
```

---

## Step 3: Verify Database Created

### 3.1 Connect to PostgreSQL Using psql

**Access PostgreSQL Shell:**

**Option A: Using Docker Exec in IntelliJ**
1. Right-click on `postgres-container`
2. Select **Exec → Create...**
3. Enter command: `/bin/bash`
4. Click **OK**
5. Terminal opens inside container

**Option B: Using Command Line**
```bash
docker exec -it postgres-container psql -U admin -d order-service
```

### 3.2 Verify Database Exists

**Inside psql Shell:**

```sql
-- List all databases
\l

-- You should see:
-- order-service | admin | UTF8 | ...
```

**Expected Output:**
```
                                   List of databases
     Name      | Owner | Encoding |  Collate   |   Ctype    | Access privileges
---------------+-------+----------+------------+------------+-------------------
 order-service | admin | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres      | admin | UTF8     | en_US.utf8 | en_US.utf8 |
 template0     | admin | UTF8     | en_US.utf8 | en_US.utf8 | =c/admin         +
               |       |          |            |            | admin=CTc/admin
 template1     | admin | UTF8     | en_US.utf8 | en_US.utf8 | =c/admin         +
               |       |          |            |            | admin=CTc/admin
(4 rows)
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

**Exit Container Shell:**
```bash
exit
```

---

## Step 4: Test Connection from Host Machine

### 4.1 Using IntelliJ Database Tool

**Add Data Source:**

1. **Open Database Tool Window**
   - View → Tool Windows → Database (or Alt+1)

2. **Add PostgreSQL Data Source**
   - Click **+** button
   - Select **Data Source → PostgreSQL**

3. **Configure Connection**

```
Name: order-service (local)
Host: localhost
Port: 5433
Database: order-service
User: admin
Password: password
```

4. **Download Drivers**
   - If prompted, click **Download Driver Files**
   - Wait for download to complete

5. **Test Connection**
   - Click **Test Connection**
   - Should show: "Succeeded"
   - Click **OK**

**Expected Result:**
```
Connected
Version: PostgreSQL 16.x
```

### 4.2 Using psql from Host Machine

**If you have psql installed locally:**

```bash
psql -h localhost -p 5433 -U admin -d order-service
```

**Enter password:** `password`

**Test Query:**
```sql
SELECT version();
```

---

## Step 5: Setup pgAdmin (Optional)

pgAdmin is a web-based GUI for PostgreSQL database management. It's optional but highly recommended for visual database interaction.

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

1. **Right-click `dpage/pgadmin4:latest` image**
2. **Select "Create Container"**

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

1. **Right-click "Servers" → Register → Server**

**General Tab:**
- **Name:** `order-service-local`

**Connection Tab:**

**IMPORTANT:** Use host machine's IP, not localhost!

**For Mac/Linux:**
```
Host: host.docker.internal
Port: 5433
Maintenance Database: order-service
Username: admin
Password: password
```

**For Windows:**
```
Host: host.docker.internal
Port: 5433
Maintenance Database: order-service
Username: admin
Password: password
```

**Why `host.docker.internal`?**
- pgAdmin runs in a container
- `localhost` inside pgAdmin container refers to itself, not host machine
- `host.docker.internal` is Docker's special DNS name for host machine
- This allows pgAdmin container to reach PostgreSQL container via host

**Alternative: Using Docker Network (Better Approach):**

If containers are on same Docker network:
```
Host: postgres-container
Port: 5432
(Use internal port, not mapped port!)
```

**Save Password:**
- Check **"Save password?"**
- Click **Save**

**Expected Result:**
- Server appears in left sidebar
- Expand: Servers → order-service-local → Databases → order-service
- You should see the database structure

---

## Step 6: Inspect Database Schema

### 6.1 Check Tables (Should be Empty for Now)

**Using pgAdmin:**
1. Expand: **order-service-local → Databases → order-service → Schemas → public → Tables**
2. Should be empty (we haven't run the application yet)

**Using psql:**
```bash
docker exec -it postgres-container psql -U admin -d order-service
```

```sql
-- List all tables
\dt

-- Should show: "Did not find any relations."
-- (Tables will be created when application starts)
```

### 6.2 What Will Happen When order-service Starts?

**JPA Auto-Schema Generation:**

When order-service starts with `spring.jpa.hibernate.ddl-auto=update`, Hibernate will:

1. **Analyze Entity Classes:**
   - Order entity → `orders` table
   - OrderLineItem entity → `order_line_items` table

2. **Create Tables Automatically:**

```sql
-- Hibernate will generate and execute:

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(255) NOT NULL
);

CREATE TABLE order_line_items (
    id BIGSERIAL PRIMARY KEY,
    sku_code VARCHAR(255) NOT NULL,
    price NUMERIC(19, 2) NOT NULL,
    quantity INTEGER NOT NULL,
    order_id BIGINT,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

3. **Create Foreign Key Relationship:**
   - `order_id` column in `order_line_items`
   - References `id` in `orders`
   - Implements `@OneToMany` relationship

**You don't need to create these tables manually!** JPA will do it automatically.

---

## Step 7: Test Application Connection to Database

### 7.1 Update application.properties

**Verify Configuration:**

**Location:** `order-service/src/main/resources/application.properties`

**Ensure these lines are present:**
```properties
spring.datasource.url=jdbc:postgresql://localhost:5433/order-service
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

**Note:** We use port `5433` because that's the host port mapped to container's `5432`.

### 7.2 Run order-service

**In IntelliJ:**
1. Open `OrderServiceApplication.java`
2. Click green play button
3. Select **Run 'OrderServiceApplication'**

**Via Command Line:**
```bash
cd order-service
./gradlew bootRun
```

### 7.3 Check Application Logs

**Look for these log messages:**

```
HikariPool-1 - Starting...
HikariPool-1 - Start completed.
Hibernate:
    create table if not exists orders (
       id bigserial not null,
        order_number varchar(255) not null,
        primary key (id)
    )
Hibernate:
    create table if not exists order_line_items (
       id bigserial not null,
        price numeric(19,2) not null,
        quantity integer not null,
        sku_code varchar(255) not null,
        order_id bigint,
        primary key (id)
    )
Hibernate:
    alter table if exists order_line_items
       add constraint FK5k8u2vjsbv2bvxvhkavlaxdwe
       foreign key (order_id)
       references orders
Started OrderServiceApplication in 5.321 seconds
```

**Key Indicators:**
- ✅ HikariCP connection pool started
- ✅ Tables created (orders, order_line_items)
- ✅ Foreign key constraint added
- ✅ Application started successfully

**If You See Error:**
```
Failed to configure a DataSource
Connection refused: localhost:5433
```

**Solutions:**
1. Verify PostgreSQL container is running (`docker ps`)
2. Check port mapping is correct (5433:5432)
3. Verify credentials match (admin/password)
4. Check application.properties configuration

### 7.4 Verify Tables Created in Database

**Using pgAdmin:**
1. Right-click **Tables** → Refresh
2. You should now see:
   - `orders`
   - `order_line_items`

**Using psql:**
```bash
docker exec -it postgres-container psql -U admin -d order-service
```

```sql
\dt
```

**Expected Output:**
```
                 List of relations
 Schema |       Name        | Type  | Owner
--------+-------------------+-------+-------
 public | order_line_items  | table | admin
 public | orders            | table | admin
(2 rows)
```

**Inspect Table Structure:**
```sql
\d orders
```

**Expected:**
```
                                     Table "public.orders"
    Column     |          Type          | Collation | Nullable |              Default
---------------+------------------------+-----------+----------+------------------------------------
 id            | bigint                 |           | not null | nextval('orders_id_seq'::regclass)
 order_number  | character varying(255) |           | not null |
Indexes:
    "orders_pkey" PRIMARY KEY, btree (id)
Referenced by:
    TABLE "order_line_items" CONSTRAINT "fk..." FOREIGN KEY (order_id) REFERENCES orders(id)
```

```sql
\d order_line_items
```

**Expected:**
```
                                         Table "public.order_line_items"
   Column   |     Type      | Collation | Nullable |                     Default
------------+---------------+-----------+----------+--------------------------------------------------
 id         | bigint        |           | not null | nextval('order_line_items_id_seq'::regclass)
 price      | numeric(19,2) |           | not null |
 quantity   | integer       |           | not null |
 sku_code   | character varying(255) |           | not null |
 order_id   | bigint        |           |          |
Indexes:
    "order_line_items_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "fk..." FOREIGN KEY (order_id) REFERENCES orders(id)
```

---

## Step 8: Test Actuator Health Endpoint

### 8.1 Access Health Endpoint

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
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 500000000000,
        "free": 250000000000,
        "threshold": 10485760,
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

**Key Indicators:**
- ✅ Overall status: `"UP"`
- ✅ Database component: `"status": "UP"`
- ✅ Database type: `"PostgreSQL"`

**If Database Shows DOWN:**
```json
{
  "status": "DOWN",
  "components": {
    "db": {
      "status": "DOWN",
      "details": {
        "error": "Connection refused"
      }
    }
  }
}
```

**Solutions:**
1. Check PostgreSQL container is running
2. Verify connection settings in application.properties
3. Check PostgreSQL logs for errors

---

## Docker Container Management

### Starting and Stopping Containers

**Stop Container:**
```bash
# IntelliJ: Right-click container → Stop
# Or CLI:
docker stop postgres-container
docker stop pgadmin-container
```

**Start Container:**
```bash
# IntelliJ: Right-click container → Start
# Or CLI:
docker start postgres-container
docker start pgadmin-container
```

**Restart Container:**
```bash
docker restart postgres-container
```

### Viewing Container Details

**In IntelliJ:**
1. Right-click container
2. Select **Inspect**
3. View JSON configuration

**Via CLI:**
```bash
docker inspect postgres-container
```

### Accessing Container Shell

**Bash Shell:**
```bash
docker exec -it postgres-container /bin/bash
```

**PostgreSQL Shell:**
```bash
docker exec -it postgres-container psql -U admin -d order-service
```

### Viewing Container Logs

**Follow Logs (Real-time):**
```bash
docker logs -f postgres-container
```

**Last 100 Lines:**
```bash
docker logs --tail 100 postgres-container
```

---

## Data Persistence

### Understanding Docker Volumes

**Current Setup (No Volume):**
- Data stored inside container
- **Data is lost when container is deleted**
- OK for development, NOT for production

**With Volume (Recommended for Production):**

**Create Container with Volume:**
```bash
docker run -d \
  --name postgres-container \
  -e POSTGRES_DB=order-service \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  -p 5433:5432 \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:latest
```

**Benefits:**
- Data persists even if container is deleted
- Can backup volume
- Can attach volume to new container

**Check Volumes:**
```bash
docker volume ls
```

**Backup Volume:**
```bash
docker run --rm \
  -v postgres-data:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/postgres-backup.tar.gz /data
```

---

## Troubleshooting

### Issue 1: Port Already in Use

**Error:**
```
Bind for 0.0.0.0:5433 failed: port is already allocated
```

**Solutions:**

**A. Change Host Port:**
```bash
docker run -d \
  --name postgres-container \
  -e POSTGRES_DB=order-service \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  -p 5434:5432 \
  postgres:latest
```

**Don't forget to update application.properties:**
```properties
spring.datasource.url=jdbc:postgresql://localhost:5434/order-service
```

**B. Stop Conflicting Container:**
```bash
# Find what's using port 5433
docker ps | grep 5433

# Stop the container
docker stop <container_id>
```

**C. Stop Local PostgreSQL:**
```bash
# Mac
brew services stop postgresql

# Linux
sudo systemctl stop postgresql
```

### Issue 2: Container Starts Then Stops

**Check Logs:**
```bash
docker logs postgres-container
```

**Common Causes:**

**A. Incorrect Environment Variables**
```
Error: database "order-service" does not exist
```
**Solution:** Ensure `POSTGRES_DB=order-service` is set

**B. Permission Issues**
```
Error: could not create directory "/var/lib/postgresql/data"
```
**Solution:** Add volume with proper permissions

### Issue 3: Cannot Connect from Application

**Error in Application Logs:**
```
Connection refused: localhost:5433
Connection timeout
```

**Checklist:**
1. ✅ Container is running: `docker ps`
2. ✅ Port mapping correct: `0.0.0.0:5433->5432/tcp`
3. ✅ Application uses correct port: `jdbc:postgresql://localhost:5433/...`
4. ✅ Credentials match: admin/password
5. ✅ Database exists: `POSTGRES_DB=order-service`

**Test Connection Manually:**
```bash
docker exec -it postgres-container psql -U admin -d order-service -c "SELECT 1"
```

**Expected:**
```
 ?column?
----------
        1
(1 row)
```

### Issue 4: pgAdmin Cannot Connect

**Error:**
```
Unable to connect to server:
Connection refused
```

**Solution:**

**Use `host.docker.internal` instead of `localhost`**
- **Host:** `host.docker.internal`
- **Port:** `5433`

**Or:** Put containers on same Docker network (covered in docker-compose section)

### Issue 5: Tables Not Created

**Symptoms:**
- Application starts successfully
- No tables in database

**Checklist:**
1. ✅ `spring.jpa.hibernate.ddl-auto=update` in application.properties
2. ✅ Entities have `@Entity` annotation
3. ✅ Entities are in package scanned by Spring Boot
4. ✅ No errors in application logs

**Force Schema Recreation (Development Only):**
```properties
spring.jpa.hibernate.ddl-auto=create-drop
```

**Warning:** This deletes all data on startup!

### Issue 6: Authentication Failed

**Error:**
```
FATAL: password authentication failed for user "admin"
```

**Solutions:**

**A. Verify Environment Variables:**
```bash
docker inspect postgres-container | grep -A 5 Env
```

**Expected:**
```
"Env": [
    "POSTGRES_DB=order-service",
    "POSTGRES_USER=admin",
    "POSTGRES_PASSWORD=password",
    ...
]
```

**B. Recreate Container:**
```bash
docker stop postgres-container
docker rm postgres-container

docker run -d \
  --name postgres-container \
  -e POSTGRES_DB=order-service \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=password \
  -p 5433:5432 \
  postgres:latest
```

---

## Summary

### What You Completed:

**PostgreSQL Setup:**
- ✅ Pulled postgres:latest Docker image
- ✅ Created postgres-container with environment variables
- ✅ Configured port mapping (5433:5432)
- ✅ Database `order-service` auto-created
- ✅ Verified database exists and is accessible

**pgAdmin Setup (Optional):**
- ✅ Pulled dpage/pgadmin4:latest image
- ✅ Created pgadmin-container
- ✅ Accessed pgAdmin web interface (localhost:5050)
- ✅ Registered PostgreSQL server in pgAdmin
- ✅ Verified visual database access

**Connection Verification:**
- ✅ Connected from host machine using psql
- ✅ Connected from IntelliJ Database Tool
- ✅ Connected from order-service application
- ✅ Verified Actuator health endpoint shows database UP

**Schema Verification:**
- ✅ JPA created `orders` table automatically
- ✅ JPA created `order_line_items` table automatically
- ✅ Foreign key relationship established
- ✅ Inspected schema using pgAdmin and psql

### Key Concepts Learned:

- ✅ Docker container creation and management
- ✅ Environment variables for container configuration
- ✅ Port mapping (host:container)
- ✅ Docker networking (container-to-container communication)
- ✅ JPA automatic schema generation
- ✅ PostgreSQL connection strings (JDBC URLs)
- ✅ Docker volumes for data persistence
- ✅ Container inspection and log viewing

### Configuration Summary:

**PostgreSQL Container:**
```
Container Name: postgres-container
Image: postgres:latest
Port Mapping: 5433:5432
Database: order-service
User: admin
Password: password
```

**pgAdmin Container:**
```
Container Name: pgadmin-container
Image: dpage/pgadmin4:latest
Port Mapping: 5050:80
Email: admin@admin.com
Password: admin
```

**Application Connection (Local):**
```
URL: jdbc:postgresql://localhost:5433/order-service
User: admin
Password: password
```

---

## Next Steps

You have successfully set up PostgreSQL and pgAdmin Docker containers and verified database connectivity. Next, you will:

1. Test the order-service API using Postman
2. Write integration tests using TestContainers
3. Verify data persistence
4. Create automated tests

Continue to [07-testing.md](07-testing.md)

---

**Files Modified:**
- ✅ application.properties - Verified PostgreSQL connection settings

**Docker Resources Created:**
- ✅ postgres-container (running PostgreSQL 16)
- ✅ pgadmin-container (running pgAdmin 4)
- ✅ Database: order-service
- ✅ Tables: orders, order_line_items (auto-created by JPA)
