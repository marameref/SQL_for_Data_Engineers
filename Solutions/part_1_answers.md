# 🌟 💫 ⭐️ 50+ SQL Interview Solutions to Questions for Data Engineering

---

## 1️⃣ Schema & Table Design Questions & Answers

### **Q1: What’s the difference between PRIMARY KEY and UNIQUE constraints?**

* **PRIMARY KEY**

  * Ensures each row is uniquely identifiable.
  * **Automatically implies `NOT NULL`** (you cannot have nulls).
  * A table can only have **one primary key** (though it can span multiple columns → composite key).
* **UNIQUE**

  * Ensures all values in the column(s) are distinct.
  * **Allows a single NULL** (in most RDBMS).
  * A table can have **multiple unique constraints**.
* 💡 **In practice:** Use `PRIMARY KEY` to identify rows. Use `UNIQUE` to enforce business rules (e.g., email addresses must be unique, but they’re not the PK).

---

### **Q2: How would you design a table for a Slowly Changing Dimension (SCD) Type 2?**

SCD Type 2 is used to **preserve history** of changes in dimension tables.

**Typical schema design:**

```sql
CREATE TABLE customer_dim (
    customer_sk SERIAL PRIMARY KEY,       -- surrogate key
    customer_id INT NOT NULL,             -- natural key (business key)
    name VARCHAR(100),
    address VARCHAR(200),
    valid_from DATE NOT NULL,
    valid_to DATE,
    current_flag BOOLEAN DEFAULT TRUE
);
```

* Every time a change happens (e.g., customer moves), you **insert a new row** with:

  * `valid_from` = date of change
  * `valid_to` = NULL until superseded
  * `current_flag` = TRUE for latest record, FALSE for historical ones
* 💡 **In practice:** Common in data warehouses for dimensions like `Customer`, `Product`.

---

### **Q3: Explain the difference between normalized and denormalized database designs.**

* **Normalized Design (OLTP systems — e.g., PostgreSQL, MySQL for transactions)**

  * Data is broken into multiple tables with relationships (3NF+).
  * Reduces redundancy ✅
  * Ensures consistency ✅
  * Example: `orders`, `customers`, `products` separated, linked by keys.
* **Denormalized Design (OLAP systems — e.g., Snowflake, Redshift for analytics)**

  * Combines tables to reduce joins.
  * Faster reads 🚀
  * More redundancy (same data may exist in multiple rows).
  * Example: `order_fact` table containing order + customer + product info.
* 💡 **Rule of thumb:** Normalize for write-heavy systems, denormalize for read-heavy analytics.

---

### **Q4: What is a surrogate key and when would you use one?**

* **Surrogate Key**

  * A system-generated identifier (like `AUTO_INCREMENT` or `UUID`).
  * Has **no business meaning**.
* **Use Cases**

  * When natural keys (like SSN, Email) can change → surrogate ensures stability.
  * For **data warehouses**, surrogate keys are standard for dimension tables.
  * Makes joins faster and simpler (`INT` keys vs composite natural keys).
* Example:

```sql
customer_sk SERIAL PRIMARY KEY   -- surrogate key
customer_id VARCHAR(20)          -- natural key
```

---

### **Q5: How do you enforce referential integrity in a relational database?**

* **Referential integrity** = making sure relationships between tables stay consistent.
* Done using **FOREIGN KEYS**:

```sql
ALTER TABLE orders
ADD CONSTRAINT fk_customer
FOREIGN KEY (customer_id)
REFERENCES customers(customer_id)
ON DELETE CASCADE;
```

* Guarantees:

  * You can’t insert an order for a non-existent customer.
  * If a customer is deleted, all their orders are deleted (`CASCADE`).
* 💡 **In practice:** Sometimes avoided in *big data pipelines* (because of scale); instead handled at the ETL layer.

---

✅ **Summary Table**

| Concept                        | Key Point                                                                               | Real-World Usage                                |
| ------------------------------ | --------------------------------------------------------------------------------------- | ----------------------------------------------- |
| **PRIMARY KEY vs UNIQUE**      | PK = identity (not null, 1 per table). UNIQUE = multiple allowed, allows nulls.         | PK for row identity, UNIQUE for business rules. |
| **SCD Type 2**                 | Preserves history with `valid_from`, `valid_to`.                                        | Customer/product changes in DW.                 |
| **Normalized vs Denormalized** | Normalized = less redundancy, slower joins. Denormalized = fast reads, more redundancy. | OLTP (normalized), OLAP (denormalized).         |
| **Surrogate Key**              | Artificial key with no business meaning.                                                | Standard in DW for stability + performance.     |
| **Referential Integrity**      | Enforced with foreign keys (`FK`).                                                      | Ensures data consistency between tables.        |

---
