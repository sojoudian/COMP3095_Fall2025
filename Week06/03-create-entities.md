# Create JPA Entities

## Overview

In this section, you will create JPA entities to model the order domain. You'll create two entities:
1. **OrderLineItem** - Represents individual items within an order
2. **Order** - Represents a customer order

**IMPORTANT:** Create `OrderLineItem` first because `Order` references it.

---

## Step 1: Create OrderLineItem Entity

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

**Class-Level Annotations:**
- `@Entity` - Marks class as JPA entity
- `@Table(name = "order_line_items")` - Specifies database table name
- `@Data` - Lombok: generates getters, setters, toString, equals, hashCode
- `@NoArgsConstructor` - Generates no-argument constructor (required by JPA)
- `@AllArgsConstructor` - Generates constructor with all fields
- `@Builder` - Implements Builder pattern

**Field-Level Annotations:**

**Primary Key:**
```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```
- `@Id` - Marks field as primary key
- `@GeneratedValue` - Auto-generates value using database auto-increment

**SKU Code Field:**
```java
@Column(name = "sku_code", nullable = false)
private String skuCode;
```
- SKU = Stock Keeping Unit (product identifier)
- `nullable = false` - Database constraint (NOT NULL)

**Price Field:**
```java
@Column(nullable = false, precision = 19, scale = 2)
private BigDecimal price;
```
- `BigDecimal` - Used for currency (accurate decimal arithmetic)
- `precision = 19` - Total number of digits
- `scale = 2` - Number of digits after decimal point

**Quantity Field:**
```java
@Column(nullable = false)
private Integer quantity;
```
- Number of items in the order line

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

**Relationship Field:**

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "order_id")
private List<OrderLineItem> orderLineItems;
```

This is a **unidirectional one-to-many relationship**.

**Annotations Explained:**

**`@OneToMany`:**
- One Order has Many OrderLineItems
- Defines parent-child relationship

**`cascade = CascadeType.ALL`:**
- All operations cascade to children
- When you save Order → OrderLineItems are saved automatically
- When you delete Order → OrderLineItems are deleted automatically

**`orphanRemoval = true`:**
- Removes OrderLineItems not in the list
- If you remove an item from `orderLineItems` list and save → That item is deleted from database

**`@JoinColumn(name = "order_id")`:**
- Specifies foreign key column in `order_line_items` table
- Column name: `order_id`
- Links OrderLineItem back to Order

**Other Fields:**

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

---

## Summary

### What You Created:

**OrderLineItem Entity:**
- ✅ Represents items within an order
- ✅ Has primary key (id)
- ✅ Has SKU code, price, and quantity
- ✅ Uses BigDecimal for price precision

**Order Entity:**
- ✅ Represents customer orders
- ✅ Has primary key (id)
- ✅ Has order number
- ✅ Has one-to-many relationship with OrderLineItems
- ✅ Configured with cascade operations

---

## Next Steps

Continue to [04-create-dtos-and-services.md](04-create-dtos-and-services.md)

---

**Files Created:**
- ✅ `OrderLineItem.java` - Order line item entity
- ✅ `Order.java` - Order entity with one-to-many relationship
