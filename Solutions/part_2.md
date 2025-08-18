# Part 2: 2️⃣ Advanced Joins & Set Operations
🚀 This is where things get really **Data Engineer interview flavored** — because joins & set operations come up *all the time*. Let’s break these 5 down one by one:

---

## 2️⃣ Advanced Joins & Set Operations

### **Q1: What’s the difference between UNION and UNION ALL?**

* **UNION**

  * Combines results of two queries.
  * Removes duplicates (like `DISTINCT`).
  * Slower than `UNION ALL` because of the deduplication step.
* **UNION ALL**

  * Combines results, **keeps duplicates**.
  * Faster since it skips the distinct step.
* 💡 **Rule of thumb:** Use `UNION ALL` unless you *must* remove duplicates (better performance for ETL).

**Example:**

```sql
SELECT city FROM customers
UNION
SELECT city FROM suppliers;

-- vs
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

---

### **Q2: How would you write a query to find records in table A but not in table B?**

Several ways:

1. **Using `LEFT JOIN … WHERE NULL`:**

```sql
SELECT a.*
FROM table_a a
LEFT JOIN table_b b
  ON a.id = b.id
WHERE b.id IS NULL;
```

2. **Using `NOT IN`:**

```sql
SELECT *
FROM table_a
WHERE id NOT IN (SELECT id FROM table_b);
```

3. **Using `EXCEPT` (Postgres only, not in SQLite):**

```sql
SELECT id FROM table_a
EXCEPT
SELECT id FROM table_b;
```

💡 **Best practice:** Use `LEFT JOIN … WHERE NULL` for readability and safety (since `NOT IN` breaks if `NULL` exists).

---

### **Q3: Explain when you’d use a FULL OUTER JOIN instead of LEFT or RIGHT join.**

* **LEFT JOIN:** All rows from left + matching rows from right.
* **RIGHT JOIN:** All rows from right + matching rows from left.
* **FULL OUTER JOIN:** All rows from both sides. If no match, fills with `NULL`.

👉 Use **FULL OUTER JOIN** when:

* You want to see **all records**, regardless of match.
* Example: comparing data across two systems to see mismatches.

```sql
SELECT a.id, b.id
FROM table_a a
FULL OUTER JOIN table_b b
  ON a.id = b.id;
```

💡 **Snowflake & Postgres support it; SQLite does not.**

---

### **Q4: How do you detect and remove duplicate rows using SQL?**

1. **Detect duplicates (based on all columns):**

```sql
SELECT col1, col2, COUNT(*)
FROM my_table
GROUP BY col1, col2
HAVING COUNT(*) > 1;
```

2. **Delete duplicates (Postgres example using `CTE`):**

```sql
WITH duplicates AS (
  SELECT id,
         ROW_NUMBER() OVER (PARTITION BY col1, col2 ORDER BY id) AS row_num
  FROM my_table
)
DELETE FROM my_table
WHERE id IN (
  SELECT id FROM duplicates WHERE row_num > 1
);
```

💡 In Data Engineering pipelines, deduplication is super common, especially with **streaming data**.

---

### **Q5: What are semi-joins and anti-joins?**

These are **logical concepts** — SQL doesn’t have `SEMI JOIN`/`ANTI JOIN` keywords (except in some engines), but they’re implemented using `EXISTS` or `NOT EXISTS`.

* **Semi-Join**: Return rows from table A **where a match exists** in table B, but without returning B’s columns.

```sql
SELECT a.*
FROM table_a a
WHERE EXISTS (SELECT 1 FROM table_b b WHERE a.id = b.id);
```

💡 Like an `INNER JOIN`, but doesn’t return columns from `b`.

* **Anti-Join**: Return rows from table A **where no match exists** in table B.

```sql
SELECT a.*
FROM table_a a
WHERE NOT EXISTS (SELECT 1 FROM table_b b WHERE a.id = b.id);
```

💡 In ETL pipelines, anti-joins are used for **finding new or missing records**.

---

✅ **Quick Reference Table**

| Concept                | Key Point                                          | Example Usage                              |
| ---------------------- | -------------------------------------------------- | ------------------------------------------ |
| **UNION vs UNION ALL** | UNION removes dups, UNION ALL keeps them (faster). | Combining datasets (logs, customers).      |
| **A but not in B**     | `LEFT JOIN … WHERE NULL` or `NOT EXISTS`.          | Finding missing customers.                 |
| **FULL OUTER JOIN**    | Returns everything, NULL where no match.           | Comparing two systems’ data.               |
| **Remove Duplicates**  | `ROW_NUMBER()` + delete extra rows.                | Cleaning raw ingested data.                |
| **Semi/Anti Joins**    | `EXISTS` / `NOT EXISTS`.                           | Semi = find overlaps, Anti = find missing. |

---

🔥 These are **100% real-world data engineering join scenarios** — like CDC (change data capture), deduping logs, or comparing staging vs prod tables.


Got it 👍 — you want the **next block fully broken down** just like I did for **Advanced Joins & Set Operations**. Since you said “next block (2️⃣ Querying & Joins section)” → I’ll assume you mean the **core Querying & Joins interview section** (before we move deeper into window functions).

Let’s dive in 🚀

---

## 🔹 2️⃣ Querying & Joins (Core Scenarios)

### **Q1: How do you find the second highest salary from an Employee table?**

* **Classic Problem** (asked in almost every SQL interview).
* **Solutions:**

  1. **Using `LIMIT / OFFSET` (Postgres style):**

  ```sql
  SELECT DISTINCT salary
  FROM employees
  ORDER BY salary DESC
  LIMIT 1 OFFSET 1;
  ```

  2. **Using Window Functions:**

  ```sql
  SELECT salary
  FROM (
      SELECT salary,
             DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
      FROM employees
  ) t
  WHERE rnk = 2;
  ```

  ✅ Window functions are safer if duplicates exist.

---

### **Q2: How do you join three or more tables in SQL?**

* **Just extend `JOIN` clauses.**

```sql
SELECT o.order_id, c.customer_name, p.product_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id;
```

💡 Real world: data engineers join staging tables (orders, customers, products) to build fact tables.

---

### **Q3: Difference between INNER JOIN, LEFT JOIN, RIGHT JOIN, and FULL OUTER JOIN?**

* **INNER JOIN** → matching rows only.
* **LEFT JOIN** → all from left, matching from right.
* **RIGHT JOIN** → all from right, matching from left.
* **FULL OUTER JOIN** → all from both sides, `NULL` if no match.

**Example:** Customer vs Orders table.

* `INNER JOIN` → Customers who placed orders.
* `LEFT JOIN` → All customers, even those without orders.
* `RIGHT JOIN` → All orders, even if customer record missing (rare but possible in dirty data).
* `FULL` → Audit mismatches between systems.

---

### **Q4: How do you get top-N records per group?**

* **Scenario:** Get top 2 orders per customer by amount.

```sql
SELECT customer_id, order_id, amount
FROM (
    SELECT customer_id, order_id, amount,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rn
    FROM orders
) t
WHERE rn <= 2;
```

💡 This is **super common** in analytics (e.g., top 5 products per category).

---

### **Q5: How to find customers who bought products from *both* Category A and Category B?**

* **INTERSECT method (Postgres only):**

```sql
SELECT customer_id
FROM orders
WHERE category = 'A'
INTERSECT
SELECT customer_id
FROM orders
WHERE category = 'B';
```

* **Self-Join method (portable):**

```sql
SELECT DISTINCT o1.customer_id
FROM orders o1
JOIN orders o2
  ON o1.customer_id = o2.customer_id
WHERE o1.category = 'A' AND o2.category = 'B';
```

💡 Useful in **loyalty programs / cross-selling**.

---

### **Q6: What’s the difference between CROSS JOIN and SELF JOIN?**

* **CROSS JOIN** → Cartesian product (all combinations).

```sql
SELECT a.name, b.color
FROM animals a
CROSS JOIN colors b;
```

* **SELF JOIN** → Joining a table to itself (with aliasing).

```sql
SELECT e1.name, e2.name AS manager
FROM employees e1
JOIN employees e2 ON e1.manager_id = e2.id;
```

💡 Cross joins are rare (sometimes used in generating time slots), but self-joins are very common in **hierarchies (employees, org structures, category trees).**

---

### **Q7: Difference between WHERE and HAVING?**

* **WHERE** → filters rows *before* grouping.
* **HAVING** → filters groups *after aggregation*.

```sql
-- Customers who spent more than $1000
SELECT customer_id, SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 1000;
```

💡 Trick question: You can’t use aggregates (`SUM`, `AVG`, etc.) inside `WHERE`.

---

### **Q8: How to find customers who never placed an order?**

```sql
SELECT c.customer_id, c.customer_name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

💡 This is **classic anti-join**.

---

### **Q9: What’s the difference between `EXISTS` and `IN`?**

* **`IN`** → compares a value against a list/subquery.
* **`EXISTS`** → checks if subquery returns *at least one row*.

```sql
-- Using IN
SELECT * FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders);

-- Using EXISTS
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

✅ **Rule of thumb:**

* `EXISTS` → better for correlated subqueries.
* `IN` → simple lookups.
* `NOT EXISTS` vs `NOT IN`: `NOT EXISTS` is safer when NULLs are involved.

---

### **Q10: Explain correlated vs non-correlated subqueries.**

* **Non-Correlated Subquery** → independent query.

```sql
SELECT name
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

* **Correlated Subquery** → depends on outer query.

```sql
SELECT name
FROM employees e
WHERE salary > (SELECT AVG(salary)
                FROM employees
                WHERE department_id = e.department_id);
```

💡 Correlated ones run *per row* → expensive, but sometimes necessary.

---

✅ **Quick Ref Table**

| Concept                  | Key Takeaway                   | Real-world use                |
| ------------------------ | ------------------------------ | ----------------------------- |
| 2nd highest salary       | Use `OFFSET` or `DENSE_RANK`.  | HR/payroll queries.           |
| Multi-table joins        | Chain `JOIN`s.                 | ETL fact/dimension building.  |
| INNER/LEFT/RIGHT/FULL    | Different completeness levels. | Customer/order analysis.      |
| Top-N per group          | `ROW_NUMBER()` in partition.   | Best-sellers, top spenders.   |
| A & B buyers             | INTERSECT or self-join.        | Cross-selling campaigns.      |
| CROSS vs SELF join       | Cartesian vs hierarchy.        | Generate slots vs org charts. |
| WHERE vs HAVING          | Before vs after grouping.      | Aggregated filters.           |
| Customers with no orders | LEFT JOIN + `IS NULL`.         | Retention analytics.          |
| EXISTS vs IN             | EXISTS safer with NULLs.       | Membership checks.            |
| Correlated subqueries    | Re-run per row.                | Department-wise comparisons.  |

---

💡 These **10 querying & joins Qs** are bread-and-butter for SQL interviews. If you master them, you’re already in the **top 20% of candidates**.

