**Solutions pack of 55 SQLite SQL tasks** using the practice DB we created (Sales/Orders, HR, Yelp-style JSON, ETL). Each item has a **ready-to-run query** and a super-brief note on what it returns. Everything is SQLite-friendly (no RIGHT/FULL JOINs, uses JSON1 where needed).

---

# 🧩 A. Joins & Aggregations (1–15)

**1) Total revenue (all time)**

```sql
SELECT ROUND(SUM(od.quantity * p.price), 2) AS total_revenue
FROM order_details od
JOIN products p ON p.product_id = od.product_id;
```

> Sum of quantity × price across all order lines.

**2) Revenue per order**

```sql
SELECT o.order_id,
       ROUND(SUM(od.quantity * p.price), 2) AS order_revenue
FROM orders o
JOIN order_details od ON od.order_id = o.order_id
JOIN products p ON p.product_id = od.product_id
GROUP BY o.order_id
ORDER BY order_revenue DESC;
```

> Each order’s revenue.

**3) Top 3 products by revenue**

```sql
SELECT p.product_id, p.name,
       ROUND(SUM(od.quantity * p.price), 2) AS revenue
FROM products p
LEFT JOIN order_details od ON od.product_id = p.product_id
GROUP BY p.product_id, p.name
ORDER BY revenue DESC
LIMIT 3;
```

> Includes products with zero sales.

**4) Average order value (AOV)**

```sql
WITH order_rev AS (
  SELECT o.order_id, SUM(od.quantity * p.price) AS rev
  FROM orders o
  JOIN order_details od ON od.order_id = o.order_id
  JOIN products p ON p.product_id = od.product_id
  GROUP BY o.order_id
)
SELECT ROUND(AVG(rev), 2) AS avg_order_value
FROM order_rev;
```

> Mean revenue per order.

**5) Orders per customer + revenue**

```sql
SELECT c.customer_id, c.name,
       COUNT(DISTINCT o.order_id) AS orders_count,
       ROUND(SUM(od.quantity * p.price), 2) AS total_revenue
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
LEFT JOIN order_details od ON od.order_id = o.order_id
LEFT JOIN products p ON p.product_id = od.product_id
GROUP BY c.customer_id, c.name
ORDER BY total_revenue DESC;
```

> Customer activity summary.

**6) Revenue by category**

```sql
SELECT p.category,
       ROUND(SUM(od.quantity * p.price), 2) AS revenue
FROM products p
JOIN order_details od ON od.product_id = p.product_id
GROUP BY p.category
ORDER BY revenue DESC;
```

> Category-level revenue.

**7) Conversion of “Cancelled” orders (count & % of all)**

```sql
SELECT 
  SUM(status='Cancelled') AS cancelled,
  COUNT(*) AS total,
  ROUND(100.0 * SUM(status='Cancelled') / COUNT(*), 2) AS pct_cancelled
FROM orders;
```

> SQLite treats boolean as 0/1—handy trick.

**8) Monthly revenue trend**

```sql
SELECT substr(o.order_date,1,7) AS year_month,
       ROUND(SUM(od.quantity * p.price), 2) AS revenue
FROM orders o
JOIN order_details od ON od.order_id = o.order_id
JOIN products p ON p.product_id = od.product_id
GROUP BY year_month
ORDER BY year_month;
```

> YYYY-MM revenue.

**9) Product mix within each order (share %)**

```sql
WITH lines AS (
  SELECT od.order_id, od.product_id, od.quantity * p.price AS line_rev
  FROM order_details od JOIN products p ON p.product_id = od.product_id
),
tot AS (
  SELECT order_id, SUM(line_rev) AS order_rev FROM lines GROUP BY order_id
)
SELECT l.order_id, l.product_id,
       ROUND(100.0 * l.line_rev / t.order_rev, 2) AS pct_of_order
FROM lines l JOIN tot t ON t.order_id = l.order_id
ORDER BY l.order_id, pct_of_order DESC;
```

> % contribution per product inside each order.

**10) Best city by total revenue**

```sql
SELECT c.city, ROUND(SUM(od.quantity * p.price), 2) AS revenue
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_details od ON od.order_id = o.order_id
JOIN products p ON p.product_id = od.product_id
GROUP BY c.city
ORDER BY revenue DESC;
```

> City leaderboard.

**11) Customers with no orders**

```sql
SELECT c.*
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```

> Unconverted customers.

**12) Products never ordered**

```sql
SELECT p.*
FROM products p
LEFT JOIN order_details od ON od.product_id = p.product_id
WHERE od.order_detail_id IS NULL;
```

> Inventory with no sales.

**13) Orders with only one line item**

```sql
SELECT o.order_id
FROM orders o
JOIN order_details od ON od.order_id = o.order_id
GROUP BY o.order_id
HAVING COUNT(*) = 1;
```

> Single-line orders.

**14) Average items per order**

```sql
SELECT ROUND(AVG(items), 2) AS avg_items_per_order
FROM (
  SELECT o.order_id, SUM(od.quantity) AS items
  FROM orders o JOIN order_details od ON od.order_id = o.order_id
  GROUP BY o.order_id
);
```

> Quantity perspective.

**15) Median product price (approx via percentile)**

```sql
-- SQLite: emulate median with window rank
WITH ranked AS (
  SELECT price,
         ROW_NUMBER() OVER (ORDER BY price) AS rn,
         COUNT(*) OVER () AS cnt
  FROM products
)
SELECT AVG(price) AS median_price
FROM ranked
WHERE rn IN ((cnt+1)/2, (cnt+2)/2);
```

> Median of `products.price`.

---

# 🪜 B. Window Functions (16–25)

**16) Rank products by revenue**

```sql
WITH pr AS (
  SELECT p.product_id, p.name, SUM(od.quantity) * p.price AS revenue
  FROM products p LEFT JOIN order_details od ON od.product_id = p.product_id
  GROUP BY p.product_id, p.name
)
SELECT *, RANK() OVER (ORDER BY revenue DESC) AS rev_rank
FROM pr
ORDER BY rev_rank;
```

**17) Cumulative revenue by date**

```sql
WITH daily AS (
  SELECT o.order_date, SUM(od.quantity * p.price) AS rev
  FROM orders o
  JOIN order_details od ON od.order_id = o.order_id
  JOIN products p ON p.product_id = od.product_id
  GROUP BY o.order_date
)
SELECT order_date,
       rev,
       SUM(rev) OVER (ORDER BY order_date) AS cum_rev
FROM daily
ORDER BY order_date;
```

**18) Percent of total revenue per product**

```sql
WITH pr AS (
  SELECT p.product_id, p.name, SUM(od.quantity * p.price) AS revenue
  FROM products p JOIN order_details od ON od.product_id = p.product_id
  GROUP BY p.product_id, p.name
),
tot AS (SELECT SUM(revenue) AS t FROM pr)
SELECT pr.*,
       ROUND(100.0 * pr.revenue / tot.t, 2) AS pct_total
FROM pr, tot
ORDER BY pct_total DESC;
```

**19) Customer order frequency rank (dense)**

```sql
WITH oc AS (
  SELECT c.customer_id, c.name, COUNT(DISTINCT o.order_id) AS orders_cnt
  FROM customers c LEFT JOIN orders o ON o.customer_id = c.customer_id
  GROUP BY c.customer_id, c.name
)
SELECT *, DENSE_RANK() OVER (ORDER BY orders_cnt DESC) AS freq_rank
FROM oc
ORDER BY freq_rank, name;
```

**20) Salary z-score by department (HR)**

```sql
SELECT e.emp_id, e.name, d.dept_name, e.salary,
       ROUND((e.salary - AVG(e.salary) OVER (PARTITION BY e.dept_id)) /
             NULLIF(STDDEV(e.salary) OVER (PARTITION BY e.dept_id),0), 2) AS z_score
FROM employees e JOIN departments d ON d.dept_id = e.dept_id;
```

**21) Top earner per department**

```sql
SELECT *
FROM (
  SELECT e.*, d.dept_name,
         ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY salary DESC) AS rn
  FROM employees e JOIN departments d ON d.dept_id = e.dept_id
) t
WHERE rn = 1;
```

**22) Running count of orders per customer by date**

```sql
SELECT o.customer_id, o.order_id, o.order_date,
       COUNT(*) OVER (PARTITION BY o.customer_id ORDER BY o.order_date
                      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_orders
FROM orders o
ORDER BY o.customer_id, o.order_date;
```

**23) 7-day moving average of daily revenue (if daily granularity existed)**

```sql
WITH daily AS (
  SELECT o.order_date AS dt, SUM(od.quantity * p.price) AS rev
  FROM orders o
  JOIN order_details od ON od.order_id = o.order_id
  JOIN products p ON p.product_id = od.product_id
  GROUP BY o.order_date
)
SELECT dt, rev,
       ROUND(AVG(rev) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS mov_avg_7
FROM daily
ORDER BY dt;
```

**24) Order lines ranked by line revenue within each order**

```sql
SELECT od.order_id, p.name,
       od.quantity * p.price AS line_rev,
       RANK() OVER (PARTITION BY od.order_id ORDER BY od.quantity * p.price DESC) AS line_rank
FROM order_details od JOIN products p ON p.product_id = od.product_id
ORDER BY od.order_id, line_rank;
```

**25) Share of category revenue within category**

```sql
WITH cat_rev AS (
  SELECT p.category, p.product_id, p.name,
         SUM(od.quantity * p.price) AS rev
  FROM products p
  LEFT JOIN order_details od ON od.product_id = p.product_id
  GROUP BY p.category, p.product_id, p.name
),
cat_tot AS (
  SELECT category, SUM(rev) AS tot FROM cat_rev GROUP BY category
)
SELECT c.*, ROUND(100.0 * c.rev / t.tot, 2) AS pct_in_cat
FROM cat_rev c JOIN cat_tot t USING(category)
ORDER BY category, pct_in_cat DESC;
```

---

# 🧵 C. CTEs & Subqueries (26–35)

**26) Customers who spent above average**

```sql
WITH spend AS (
  SELECT c.customer_id, c.name, COALESCE(SUM(od.quantity*p.price),0) AS total_spend
  FROM customers c
  LEFT JOIN orders o ON o.customer_id = c.customer_id
  LEFT JOIN order_details od ON od.order_id = o.order_id
  LEFT JOIN products p ON p.product_id = od.product_id
  GROUP BY c.customer_id, c.name
),
avg_sp AS (SELECT AVG(total_spend) AS avg_spend FROM spend)
SELECT s.*
FROM spend s, avg_sp a
WHERE s.total_spend > a.avg_spend
ORDER BY s.total_spend DESC;
```

**27) Second-highest product price (no LIMIT)**

```sql
SELECT MAX(price) AS second_highest
FROM products
WHERE price < (SELECT MAX(price) FROM products);
```

**28) Orders that include Electronics AND Furniture**

```sql
SELECT o.order_id
FROM orders o
WHERE EXISTS (
  SELECT 1
  FROM order_details od
  JOIN products p ON p.product_id = od.product_id
  WHERE od.order_id = o.order_id AND p.category='Electronics'
)
AND EXISTS (
  SELECT 1
  FROM order_details od
  JOIN products p ON p.product_id = od.product_id
  WHERE od.order_id = o.order_id AND p.category='Furniture'
);
```

**29) Cheapest product in each category**

```sql
SELECT p1.*
FROM products p1
WHERE p1.price = (
  SELECT MIN(p2.price) FROM products p2 WHERE p2.category = p1.category
)
ORDER BY p1.category, p1.price;
```

**30) Top N (=2) customers by spend (parameterizable)**

```sql
WITH spend AS (
  SELECT c.customer_id, c.name, SUM(od.quantity*p.price) AS total_spend
  FROM customers c
  JOIN orders o ON o.customer_id = c.customer_id
  JOIN order_details od ON od.order_id = o.order_id
  JOIN products p ON p.product_id = od.product_id
  GROUP BY c.customer_id, c.name
)
SELECT * FROM (
  SELECT s.*, DENSE_RANK() OVER (ORDER BY total_spend DESC) AS rnk
  FROM spend s
) t
WHERE rnk <= 2
ORDER BY total_spend DESC;
```

**31) Products priced above category average**

```sql
WITH cat_avg AS (
  SELECT category, AVG(price) AS avg_price
  FROM products GROUP BY category
)
SELECT p.*
FROM products p JOIN cat_avg a USING(category)
WHERE p.price > a.avg_price
ORDER BY p.category, p.price DESC;
```

**32) Last order per customer (by date)**

```sql
SELECT o.*
FROM orders o
JOIN (
  SELECT customer_id, MAX(order_date) AS last_dt
  FROM orders
  GROUP BY customer_id
) x ON x.customer_id = o.customer_id AND x.last_dt = o.order_date;
```

**33) Customers who ordered only one distinct product ever**

```sql
WITH prod_counts AS (
  SELECT o.customer_id, COUNT(DISTINCT od.product_id) AS distinct_products
  FROM orders o JOIN order_details od ON od.order_id = o.order_id
  GROUP BY o.customer_id
)
SELECT c.*
FROM customers c
JOIN prod_counts pc ON pc.customer_id = c.customer_id
WHERE pc.distinct_products = 1;
```

**34) Orders with revenue > average order revenue**

```sql
WITH order_rev AS (
  SELECT o.order_id, SUM(od.quantity*p.price) AS rev
  FROM orders o JOIN order_details od ON od.order_id = o.order_id
  JOIN products p ON p.product_id = od.product_id
  GROUP BY o.order_id
),
avg_rev AS (SELECT AVG(rev) AS a FROM order_rev)
SELECT o.order_id, ROUND(o.rev,2) AS rev
FROM order_rev o, avg_rev a
WHERE o.rev > a.a;
```

**35) Category with highest avg product price**

```sql
WITH cat_avg AS (
  SELECT category, AVG(price) AS avg_price
  FROM products
  GROUP BY category
)
SELECT category, ROUND(avg_price,2)
FROM cat_avg
ORDER BY avg_price DESC
LIMIT 1;
```

---

# 📅 D. Dates & Times (36–40)

**36) Orders per month**

```sql
SELECT substr(order_date,1,7) AS year_month, COUNT(*) AS orders_cnt
FROM orders
GROUP BY year_month
ORDER BY year_month;
```

**37) Customers signed up in 2023**

```sql
SELECT *
FROM customers
WHERE substr(signup_date,1,4) = '2023';
```

**38) Average days from signup to first order**

```sql
WITH first_order AS (
  SELECT c.customer_id, c.signup_date,
         MIN(o.order_date) AS first_order_date
  FROM customers c
  LEFT JOIN orders o ON o.customer_id = c.customer_id
  GROUP BY c.customer_id
)
SELECT ROUND(AVG(julianday(first_order_date) - julianday(signup_date)), 2) AS avg_days_to_first_order
FROM first_order
WHERE first_order_date IS NOT NULL;
```

**39) Orders in last 7 days of available data (relative max date)**

```sql
WITH m AS (SELECT MAX(order_date) AS max_dt FROM orders)
SELECT o.*
FROM orders o, m
WHERE julianday(m.max_dt) - julianday(o.order_date) <= 7;
```

**40) Weekday distribution of orders**

```sql
SELECT strftime('%w', order_date) AS weekday_0_sun,
       COUNT(*) AS orders_cnt
FROM orders
GROUP BY weekday_0_sun
ORDER BY orders_cnt DESC;
```

---

# 🧰 E. JSON (Yelp-style) (41–45)

> In our table, `attributes` and `hours` are JSON text. SQLite JSON1: `json_extract(col, '$.Key')`.

**41) Saturday & Sunday hours (Philadelphia restaurants)**

```sql
SELECT name,
       review_count,
       json_extract(hours, '$.Saturday') AS saturday_hours,
       json_extract(hours, '$.Sunday')   AS sunday_hours
FROM yelp_business_data
WHERE categories LIKE '%Restaurant%'
  AND json_extract(hours, '$.Saturday') IS NOT NULL
  AND json_extract(hours, '$.Sunday') IS NOT NULL
  AND city = 'Philadelphia'
  AND stars = 5
ORDER BY review_count DESC;
```

**42) Dogs allowed & accepts credit cards**

```sql
SELECT business_id, name
FROM yelp_business_data
WHERE categories LIKE '%Restaurant%'
  AND json_extract(attributes, '$.DogsAllowed') = 'True'
  AND json_extract(attributes, '$.BusinessAcceptsCreditCards') = 'True'
  AND city LIKE '%Philadelphia%'
  AND stars = 5;
```

**43) Businesses open Sunday (non-null)**

```sql
SELECT name, json_extract(hours, '$.Sunday') AS sunday
FROM yelp_business_data
WHERE json_extract(hours, '$.Sunday') IS NOT NULL;
```

**44) Count 5-star restaurants that accept cards**

```sql
SELECT COUNT(*) AS cnt
FROM yelp_business_data
WHERE categories LIKE '%Restaurant%'
  AND stars = 5
  AND json_extract(attributes, '$.BusinessAcceptsCreditCards') = 'True';
```

**45) Filter Italian restaurants with dogs allowed**

```sql
SELECT name, city
FROM yelp_business_data
WHERE categories LIKE '%Italian%'
  AND json_extract(attributes, '$.DogsAllowed') = 'True';
```

---

# 🧹 F. ETL / Data Quality (46–50)

**46) Find duplicates in `staging_orders` by full row**

```sql
SELECT order_id, customer_name, product_name, order_date, COUNT(*) AS cnt
FROM staging_orders
GROUP BY order_id, customer_name, product_name, order_date
HAVING COUNT(*) > 1;
```

**47) Deduplicate `staging_orders` (keep first) into a clean table**

```sql
DROP TABLE IF EXISTS clean_orders;
CREATE TABLE clean_orders AS
WITH ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (
           PARTITION BY order_id, customer_name, product_name, order_date
           ORDER BY rowid
         ) AS rn
  FROM staging_orders
)
SELECT order_id, customer_name, product_name, order_date
FROM ranked
WHERE rn = 1;
```

**48) Standardize date format (YYYY-MM-DD)**

```sql
-- If order_date already 'YYYY-MM-DD', this is pass-through.
SELECT order_id,
       customer_name,
       product_name,
       date(order_date) AS order_date_std
FROM clean_orders;
```

**49) Map names to IDs (dimension lookups)**

```sql
-- Example: join clean_orders to products & customers
SELECT co.order_id,
       c.customer_id, c.name AS customer_name,
       p.product_id, p.name AS product_name,
       co.order_date
FROM clean_orders co
LEFT JOIN customers c ON c.name = co.customer_name
LEFT JOIN products p  ON p.name  = co.product_name;
```

**50) Build a facts table from `clean_orders` (1 qty per row)**

```sql
DROP TABLE IF EXISTS fact_clean_orders;
CREATE TABLE fact_clean_orders AS
SELECT c.customer_id,
       p.product_id,
       date(co.order_date) AS order_date,
       1 AS quantity
FROM clean_orders co
LEFT JOIN customers c ON c.name = co.customer_name
LEFT JOIN products p  ON p.name  = co.product_name;
```

---

# ➕ G. Extras & Edge Cases (51–55)

**51) Pivot-like: revenue per category per month (tall)**

```sql
SELECT substr(o.order_date,1,7) AS year_month,
       p.category,
       ROUND(SUM(od.quantity * p.price), 2) AS revenue
FROM orders o
JOIN order_details od ON od.order_id = o.order_id
JOIN products p ON p.product_id = od.product_id
GROUP BY year_month, p.category
ORDER BY year_month, revenue DESC;
```

**52) Product price banding**

```sql
SELECT name, price,
       CASE
         WHEN price >= 700 THEN 'High'
         WHEN price >= 100 THEN 'Medium'
         ELSE 'Low'
       END AS price_band
FROM products
ORDER BY price DESC;
```

**53) Customers whose first order contained Electronics**

```sql
WITH first_order AS (
  SELECT customer_id, MIN(order_date) AS first_dt
  FROM orders GROUP BY customer_id
)
SELECT DISTINCT c.customer_id, c.name
FROM customers c
JOIN first_order f ON f.customer_id = c.customer_id
JOIN orders o ON o.customer_id = c.customer_id AND o.order_date = f.first_dt
JOIN order_details od ON od.order_id = o.order_id
JOIN products p ON p.product_id = od.product_id
WHERE p.category = 'Electronics';
```

**54) Find orders with a product priced above category avg (any line)**

```sql
WITH cat_avg AS (
  SELECT category, AVG(price) AS avg_price
  FROM products GROUP BY category
)
SELECT DISTINCT o.order_id
FROM orders o
JOIN order_details od ON od.order_id = o.order_id
JOIN products p ON p.product_id = od.product_id
JOIN cat_avg a ON a.category = p.category
WHERE p.price > a.avg_price;
```

**55) Revenue per customer with rank + bucket top/mid/low**

```sql
WITH spend AS (
  SELECT c.customer_id, c.name, COALESCE(SUM(od.quantity*p.price),0) AS total_spend
  FROM customers c
  LEFT JOIN orders o ON o.customer_id = c.customer_id
  LEFT JOIN order_details od ON od.order_id = o.order_id
  LEFT JOIN products p ON p.product_id = od.product_id
  GROUP BY c.customer_id, c.name
),
r AS (
  SELECT *, DENSE_RANK() OVER (ORDER BY total_spend DESC) AS rnk
  FROM spend
)
SELECT customer_id, name, ROUND(total_spend,2) AS total_spend,
       CASE
         WHEN rnk <= 1 THEN 'Top'
         WHEN rnk <= 2 THEN 'Mid'
         ELSE 'Low'
       END AS bucket
FROM r
ORDER BY total_spend DESC;
```

---
