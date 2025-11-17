# Standalone Docker Configurations

## Overview

Create standalone Docker Compose configurations for local development. This allows running databases independently without the full microservices stack.

**Time:** 30-45 minutes

---

## Prerequisites

- ✅ Completed Week 11 (API Gateway)
- ✅ Docker Desktop running
- ✅ All microservices created (product-service, order-service, inventory-service, api-gateway)

---

## Architecture

```
docker/
├── standalone/
│   ├── mongo-redis/          # MongoDB + Redis for product-service
│   │   ├── docker-compose.yml
│   │   └── init/
│   │       ├── mongo-init.js
│   │       └── redis.conf
│   └── postgres/             # PostgreSQL for order + inventory services
│       ├── docker-compose.yml
│       ├── inventory-service/
│       │   └── init/init.sql
│       └── order-service/
│           └── init/init.sql
```

---

## Step 1: Create Directory Structure

```bash
cd microservices-parent
mkdir -p docker/standalone/mongo-redis/init
mkdir -p docker/standalone/postgres/inventory-service/init
mkdir -p docker/standalone/postgres/order-service/init
```

---

## Step 2: MongoDB + Redis Configuration

### 2.1 Create mongo-init.js

**Location:** `docker/standalone/mongo-redis/init/mongo-init.js`

```javascript
db = db.getSiblingDB('admin');
db.auth('admin', 'password');

db = db.getSiblingDB('product-service');

db.createUser({
    user: 'productAdmin',
    pwd: 'password',
    roles: [
        {
            role: 'readWrite',
            db: 'product-service'
        }
    ]
});

db.createCollection('product');
db.product.createIndex({ name: 1 });
```

### 2.2 Create redis.conf

**Location:** `docker/standalone/mongo-redis/init/redis.conf`

```conf
requirepass password

save 900 1
save 300 10
save 60 10000

appendonly yes
appendfilename "appendonly.aof"
```

### 2.3 Create docker-compose.yml

**Location:** `docker/standalone/mongo-redis/docker-compose.yml`

```yaml
services:
  mongodb:
    image: mongo:latest
    container_name: standalone-mongodb
    ports:
      - "27017:27017"
    restart: no
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
      - MONGO_INITDB_DATABASE=product-service
    volumes:
      - ./init/mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro
      - ./db-data:/data/db
    command: mongod --auth
    networks:
      - backend

  mongo-express:
    image: mongo-express
    container_name: standalone-mongoexpress
    ports:
      - "8081:8081"
    restart: no
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=admin
      - ME_CONFIG_MONGODB_ADMINPASSWORD=password
      - ME_CONFIG_MONGODB_SERVER=mongodb
    depends_on:
      - mongodb
    networks:
      - backend

  redis:
    image: redis:7.4.3
    container_name: standalone-redis
    ports:
      - "6379:6379"
    restart: no
    volumes:
      - ./init/redis.conf:/usr/local/etc/redis/redis.conf
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    networks:
      - backend

  redisinsight:
    image: redislabs/redisinsight:1.14.0
    container_name: standalone-redisinsight
    ports:
      - "8001:8001"
    restart: no
    depends_on:
      - redis
    networks:
      - backend

networks:
  backend:
    driver: bridge
```

---

## Step 3: PostgreSQL Configuration

### 3.1 Create inventory-service init.sql

**Location:** `docker/standalone/postgres/inventory-service/init/init.sql`

```sql
CREATE DATABASE "inventory-service";
GRANT ALL PRIVILEGES ON DATABASE "inventory-service" TO admin;
```

### 3.2 Create order-service init.sql

**Location:** `docker/standalone/postgres/order-service/init/init.sql`

```sql
CREATE DATABASE "order-service";
GRANT ALL PRIVILEGES ON DATABASE "order-service" TO admin;
```

### 3.3 Create docker-compose.yml

**Location:** `docker/standalone/postgres/docker-compose.yml`

```yaml
services:
  postgres-inventory:
    image: postgres
    restart: always
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=password
    volumes:
      - ./inventory-service/db-data:/var/lib/postgresql/data
      - ./inventory-service/init/init.sql:/docker-entrypoint-initdb.d/init.sql
    container_name: postgresdb-inventory

  postgres-order:
    image: postgres
    restart: always
    ports:
      - "5433:5432"
    environment:
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=password
    volumes:
      - ./order-service/db-data:/var/lib/postgresql/data
      - ./order-service/init/init.sql:/docker-entrypoint-initdb.d/init.sql
    container_name: postgresdb-order

  pgpadmin:
    image: dpage/pgadmin4:9.2
    restart: always
    ports:
      - "8888:80"
    environment:
      - PGADMIN_DEFAULT_EMAIL=user@domain.ca
      - PGADMIN_DEFAULT_PASSWORD=password
    container_name: pgadmingui
```

---

## Step 4: Usage

### 4.1 Start MongoDB + Redis

```bash
cd docker/standalone/mongo-redis
docker-compose -p mongo-redis-standalone up -d
```

**Services:**
- MongoDB: `localhost:27017` (admin/password)
- Mongo Express: `http://localhost:8081` (admin/password)
- Redis: `localhost:6379` (password: password)
- RedisInsight: `http://localhost:8001`

**Connect RedisInsight:**
1. Open `http://localhost:8001`
2. Click "I already have a database"
3. Connection details:
   - Host: `localhost`
   - Port: `6379`
   - Database Alias: `product-service-cache`
   - Username: `default`
   - Password: `password`
4. Click "Add Database"

### 4.2 Start PostgreSQL

```bash
cd docker/standalone/postgres
docker-compose -p postgres-combined up -d
```

**Services:**
- PostgreSQL Inventory: `localhost:5432` (admin/password)
- PostgreSQL Order: `localhost:5433` (admin/password)
- pgAdmin: `http://localhost:8888` (user@domain.ca/password)

**Connect pgAdmin:**
1. Open `http://localhost:8888`
2. Login: `user@domain.ca` / `password`
3. Right-click "Servers" → "Register" → "Server"
4. **Inventory Database:**
   - Name: `Inventory Service`
   - Host: `host.docker.internal`
   - Port: `5432`
   - Database: `inventory-service`
   - Username: `admin`
   - Password: `password`
5. **Order Database:**
   - Name: `Order Service`
   - Host: `host.docker.internal`
   - Port: `5433`
   - Database: `order-service`
   - Username: `admin`
   - Password: `password`

### 4.3 Run Microservices Locally

**Terminal 1:**
```bash
cd product-service
./gradlew bootRun
```

**Terminal 2:**
```bash
cd order-service
./gradlew bootRun
```

**Terminal 3:**
```bash
cd inventory-service
./gradlew bootRun
```

**Terminal 4:**
```bash
cd api-gateway
./gradlew bootRun
```

### 4.4 Stop Services

**Stop MongoDB + Redis:**
```bash
cd docker/standalone/mongo-redis
docker-compose down
```

**Stop PostgreSQL:**
```bash
cd docker/standalone/postgres
docker-compose down
```

**Stop with volume cleanup:**
```bash
docker-compose down -v
```

---

## Step 5: .gitignore

Add to `.gitignore`:

```
# Docker data directories
docker/standalone/mongo-redis/db-data/
docker/standalone/postgres/inventory-service/db-data/
docker/standalone/postgres/order-service/db-data/
```

---

## Summary

You now have:
- ✅ MongoDB + Redis standalone configuration
- ✅ PostgreSQL standalone configuration
- ✅ Database initialization scripts
- ✅ Management UIs (Mongo Express, RedisInsight, pgAdmin)
- ✅ Local development setup

**Benefits:**
- Run only needed databases
- Faster startup than full docker-compose
- Easy database inspection with management UIs
- Independent service development
