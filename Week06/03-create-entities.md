# Create JPA Entities

## Overview

In this section, you will create JPA entities to model the order domain. You'll create two entities:
1. **Order** - Represents a customer order
2. **OrderLineItem** - Represents individual items within an order

These entities will demonstrate a **one-to-many relationship** where one Order can have many OrderLineItems.

---

## Understanding JPA Entities

### What is an Entity?

**Definition:**
- A lightweight persistent domain object
- Typically represents a table in a relational database
- Each entity instance corresponds to a row in the table

**Key Characteristics:**
- Must have a primary key
- Should have a no-argument constructor
- Can have relationships with other entities

### JPA Annotations

| Annotation | Purpose | Example |
|------------|---------|---------|
| `@Entity` | Marks class as JPA entity | `@Entity` |
| `@Table` | Specifies table name | `@Table(name = "orders")` |
| `@Id` | Marks primary key field | `@Id` |
| `@GeneratedValue` | Auto-generate primary key | `@GeneratedValue(strategy = GenerationType.IDENTITY)` |
| `@Column` | Specifies column details | `@Column(name = "order_number")` |
| `@OneToMany` | One-to-many relationship | `@OneToMany(cascade = CascadeType.ALL)` |
| `@ManyToOne` | Many-to-one relationship | `@ManyToOne` |
| `@JoinColumn` | Foreign key column | `@JoinColumn(name = "order_id")` |

---

## Step 1: Create OrderLineItem Entity First

**IMPORTANT:** Create `OrderLineItem` first because `Order` references it. Creating them in the wrong order will cause compilation errors.

### 1.1 Create OrderLineItem Class

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/model/OrderLineItem.java`

**Steps:**
1. Right-click on `model` package
2. Select **New → Java Class**
3. Name: `OrderLineItem`
4. Click **OK**

### 1.2 Implement OrderLineItem Entity

**Complete OrderLineItem.java:**

```java
package ca.gbc.comp3095.orderservice.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;

@Entity
@Table(name = "order_line_items")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class OrderLineItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "sku_code", nullable = false)
    private String skuCode;

    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer quantity;
}
```

### 1.3 Understanding the Code

#### **Annotations Explained:**

**Class-Level Annotations:**
```java
@Entity
```
- Marks class as JPA entity
- JPA will manage instances of this class
- Required for all JPA entities

```java
@Table(name = "order_line_items")
```
- Specifies database table name
- Without this, table name would be "orderlineitem"
- Using snake_case convention: "order_line_items"

**Lombok Annotations:**
```java
@Data
```
- Generates getters, setters, `toString()`, `equals()`, `hashCode()`
- Reduces boilerplate code significantly

```java
@NoArgsConstructor
```
- Generates no-argument constructor
- Required by JPA for entity instantiation

```java
@AllArgsConstructor
```
- Generates constructor with all fields
- Useful for creating instances manually

```java
@Builder
```
- Implements Builder pattern
- Allows fluent object creation: `OrderLineItem.builder().skuCode("ABC").build()`

#### **Field-Level Annotations:**

**Primary Key:**
```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```
- `@Id` - Marks field as primary key
- `@GeneratedValue` - Auto-generates value
- `GenerationType.IDENTITY` - Uses database auto-increment
- Type `Long` - Standard for auto-generated IDs

**SKU Code Field:**
```java
@Column(name = "sku_code", nullable = false)
private String skuCode;
```
- `@Column` - Maps to database column
- `name = "sku_code"` - Column name (snake_case convention)
- `nullable = false` - Database constraint (NOT NULL)
- SKU = Stock Keeping Unit (product identifier)

**Price Field:**
```java
@Column(nullable = false, precision = 19, scale = 2)
private BigDecimal price;
```
- `BigDecimal` - Used for currency (accurate decimal arithmetic)
- `precision = 19` - Total number of digits
- `scale = 2` - Number of digits after decimal point
- Example: 9999999999999999.99 (max value)
- Never use `double` or `float` for money!

**Quantity Field:**
```java
@Column(nullable = false)
private Integer quantity;
```
- `Integer` - Number of items
- `nullable = false` - Must have a value

### 1.4 Generated SQL (for reference)

When Hibernate creates the schema, it generates SQL like:

```sql
CREATE TABLE order_line_items (
    id BIGSERIAL PRIMARY KEY,
    sku_code VARCHAR(255) NOT NULL,
    price NUMERIC(19, 2) NOT NULL,
    quantity INTEGER NOT NULL,
    order_id BIGINT  -- Foreign key (added when Order is created)
);
```

---

## Step 2: Create Order Entity

### 2.1 Create Order Class

**Location:** `order-service/src/main/java/ca/gbc/comp3095/orderservice/model/Order.java`

**Steps:**
1. Right-click on `model` package
2. Select **New → Java Class**
3. Name: `Order`
4. Click **OK**

### 2.2 Implement Order Entity

**Complete Order.java:**

```java
package ca.gbc.comp3095.orderservice.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@Entity
@Table(name = "orders")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_number", nullable = false)
    private String orderNumber;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderLineItem> orderLineItems;
}
```

### 2.3 Understanding the Code

#### **Relationship Field:**

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "order_id")
private List<OrderLineItem> orderLineItems;
```

This is a **unidirectional one-to-many relationship**.

**Unidirectional:**
```java
Order → OrderLineItem (Order knows about OrderLineItem)
OrderLineItem ✗ Order (OrderLineItem doesn't know about Order)
```

**Annotations Explained:**

**`@OneToMany`:**
- One Order has Many OrderLineItems
- Defines parent-child relationship

**`cascade = CascadeType.ALL`:**
- All operations cascade to children
- When you save Order → OrderLineItems are saved automatically
- When you delete Order → OrderLineItems are deleted automatically
- Operations: PERSIST, MERGE, REMOVE, REFRESH, DETACH

**`orphanRemoval = true`:**
- Removes OrderLineItems not in the list
- If you remove an item from `orderLineItems` list and save → That item is deleted from database
- Example:
```java
order.getOrderLineItems().remove(0);  // Remove first item
orderRepository.save(order);          // Item deleted from database
```

**`@JoinColumn(name = "order_id")`:**
- Specifies foreign key column in `order_line_items` table
- Column name: `order_id`
- Links OrderLineItem back to Order

**Why Unidirectional?**

```java
Order → OrderLineItem  ✅ (Order knows about OrderLineItem)
OrderLineItem ✗ Order  ❌ (OrderLineItem doesn't know about Order)
```

**Advantages:**
- Simpler entity model
- No circular references
- Easier JSON serialization
- Sufficient for most use cases

**If Bidirectional Needed (Optional):**
```java
// In OrderLineItem.java
@ManyToOne
@JoinColumn(name = "order_id", insertable = false, updatable = false)
private Order order;
```

For this lab, unidirectional is sufficient.

#### **Other Fields:**

**Primary Key:**
```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```
- Auto-generated ID for each order

**Order Number:**
```java
@Column(name = "order_number", nullable = false)
private String orderNumber;
```
- Unique identifier for the order
- Example: UUID string like "123e4567-e89b-12d3-a456-426614174000"
- Generated in service layer, not by database

### 2.4 Generated SQL (for reference)

When Hibernate creates the schema, it generates SQL like:

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(255) NOT NULL
);

-- Foreign key is added to order_line_items table:
ALTER TABLE order_line_items
    ADD COLUMN order_id BIGINT,
    ADD CONSTRAINT fk_order
    FOREIGN KEY (order_id) REFERENCES orders(id);
```

---

## Step 3: Understanding Entity Relationships

### Relationship Visualization

```
┌─────────────────────┐
│      Order          │
│  (orders table)     │
├─────────────────────┤
│ id (PK)             │
│ order_number        │
└─────────────────────┘
         │
         │ @OneToMany
         │ (one order has many line items)
         │
         ↓
┌─────────────────────┐
│  OrderLineItem      │
│ (order_line_items)  │
├─────────────────────┤
│ id (PK)             │
│ sku_code            │
│ price               │
│ quantity            │
│ order_id (FK)       │
└─────────────────────┘
```

### Database Relationship

**Tables:**
- `orders` - Parent table
- `order_line_items` - Child table with `order_id` foreign key

**Example Data:**

**orders table:**
| id | order_number |
|----|-------------|
| 1  | ORD-2025-001 |
| 2  | ORD-2025-002 |

**order_line_items table:**
| id | sku_code | price | quantity | order_id |
|----|----------|-------|----------|----------|
| 1  | sku_123  | 99.99 | 2        | 1        |
| 2  | sku_456  | 49.99 | 1        | 1        |
| 3  | sku_789  | 149.99| 3        | 2        |

---

## Step 4: Understanding Cascade Operations

### What is Cascading?

**Definition:**
- Propagating operations from parent entity to child entities
- Configured with `cascade` attribute in `@OneToMany`

### CascadeType Options

| CascadeType | Description | Example |
|-------------|-------------|---------|
| `PERSIST` | Save operation cascades | Saving Order saves OrderLineItems |
| `MERGE` | Update operation cascades | Updating Order updates OrderLineItems |
| `REMOVE` | Delete operation cascades | Deleting Order deletes OrderLineItems |
| `REFRESH` | Refresh operation cascades | Refreshing Order refreshes OrderLineItems |
| `DETACH` | Detach operation cascades | Detaching Order detaches OrderLineItems |
| `ALL` | All above operations cascade | All operations propagate |

### Example: CascadeType.ALL

**Code:**
```java
@OneToMany(cascade = CascadeType.ALL)
private List<OrderLineItem> orderLineItems;
```

**Behavior:**
```java
// Create Order with OrderLineItems
Order order = Order.builder()
    .orderNumber("ORD-001")
    .orderLineItems(List.of(
        OrderLineItem.builder()
            .skuCode("sku_123")
            .price(new BigDecimal("99.99"))
            .quantity(2)
            .build()
    ))
    .build();

// Save Order
orderRepository.save(order);
// ✅ Saves Order
// ✅ Automatically saves OrderLineItems (cascade)

// Delete Order
orderRepository.delete(order);
// ✅ Deletes Order
// ✅ Automatically deletes OrderLineItems (cascade)
```

### OrphanRemoval

**What is an Orphan?**
- Child entity no longer referenced by parent
- Example: Removing OrderLineItem from Order's list

**orphanRemoval = true:**
```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderLineItem> orderLineItems;
```

**Behavior:**
```java
Order order = orderRepository.findById(1L).orElseThrow();
order.getOrderLineItems().remove(0); // Remove first line item
orderRepository.save(order);
// ✅ OrderLineItem is deleted from database (orphan removed)
```

---

## Summary

### What You Created:

**Order Entity:**
- ✅ Represents customer orders
- ✅ Has primary key (id)
- ✅ Has order number
- ✅ Has one-to-many relationship with OrderLineItems
- ✅ Configured with cascade operations

**OrderLineItem Entity:**
- ✅ Represents items within an order
- ✅ Has primary key (id)
- ✅ Has SKU code, price, and quantity
- ✅ Uses BigDecimal for price precision
- ✅ Part of Order's one-to-many relationship

### Key Concepts Covered:

- ✅ JPA entity annotations
- ✅ Entity relationships (@OneToMany)
- ✅ Cascade operations
- ✅ Orphan removal
- ✅ Primary key generation
- ✅ Column constraints
- ✅ Lombok for boilerplate reduction

### Database Schema (Generated by JPA):

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(255) NOT NULL
);

CREATE TABLE order_line_items (
    id BIGSERIAL PRIMARY KEY,
    sku_code VARCHAR(255) NOT NULL,
    price DECIMAL(19,2) NOT NULL,
    quantity INTEGER NOT NULL,
    order_id BIGINT NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

---

## Next Steps

You have successfully created the JPA entities. Next, you will:
1. Create DTOs for API requests/responses
2. Create OrderRepository interface
3. Create OrderService with business logic
4. Create OrderController for REST endpoints

Continue to [04-create-dtos-and-services.md](04-create-dtos-and-services.md)

---

**Files Created:**
- ✅ `Order.java` - Order entity with one-to-many relationship
- ✅ `OrderLineItem.java` - Order line item entity
