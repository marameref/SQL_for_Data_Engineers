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
