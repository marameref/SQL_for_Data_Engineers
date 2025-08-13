We’ll **start with SQLite** (so you can run everything right now in DB Browser for SQLite), but I’ll keep in mind that **later we’ll port it to PostgreSQL** so you get production-ready practice too.

---

## **Step 1 – SQLite Practice Database Schema**

I’ll create **4 schemas** worth of tables in one SQLite file so you can cover:

* **Sales/Orders** (classic e-commerce joins, aggregations, windows)
* **HR** (ranking, date functions, filtering)
* **Yelp-style JSON** (semi-structured queries)
* **ETL staging** (deduplication, transformations)

---

### **SQL Script – Create & Seed Database**

You can paste this into DB Browser for SQLite **Execute SQL** tab to create the practice DB.

```sql
-- ================================
-- SCHEMA 1: Sales / Orders
-- ================================
DROP TABLE IF EXISTS customers;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS order_details;

CREATE TABLE customers (
    customer_id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    city TEXT,
    signup_date DATE
);

CREATE TABLE products (
    product_id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    category TEXT,
    price REAL
);

CREATE TABLE orders (
    order_id INTEGER PRIMARY KEY,
    customer_id INTEGER,
    order_date DATE,
    status TEXT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_details (
    order_detail_id INTEGER PRIMARY KEY,
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

INSERT INTO customers (name, email, city, signup_date) VALUES
('Alice Johnson', 'alice@example.com', 'New York', '2023-01-15'),
('Bob Smith', 'bob@example.com', 'Chicago', '2023-03-10'),
('Carol Lee', 'carol@example.com', 'San Francisco', '2022-11-20'),
('David Kim', 'david@example.com', 'Los Angeles', '2023-02-05');

INSERT INTO products (name, category, price) VALUES
('Laptop', 'Electronics', 1200.00),
('Headphones', 'Electronics', 150.00),
('Office Chair', 'Furniture', 300.00),
('Coffee Mug', 'Kitchenware', 15.00),
('Standing Desk', 'Furniture', 500.00);

INSERT INTO orders (customer_id, order_date, status) VALUES
(1, '2023-05-01', 'Shipped'),
(2, '2023-05-03', 'Pending'),
(1, '2023-05-05', 'Shipped'),
(3, '2023-05-07', 'Cancelled');

INSERT INTO order_details (order_id, product_id, quantity) VALUES
(1, 1, 1),
(1, 2, 2),
(2, 3, 1),
(3, 5, 1),
(4, 4, 4);

-- ================================
-- SCHEMA 2: HR
-- ================================
DROP TABLE IF EXISTS employees;
DROP TABLE IF EXISTS departments;

CREATE TABLE departments (
    dept_id INTEGER PRIMARY KEY,
    dept_name TEXT
);

CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY,
    name TEXT,
    dept_id INTEGER,
    hire_date DATE,
    salary REAL,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);

INSERT INTO departments (dept_name) VALUES
('Engineering'), ('HR'), ('Sales');

INSERT INTO employees (name, dept_id, hire_date, salary) VALUES
('Emma Davis', 1, '2020-01-15', 90000),
('Frank Wilson', 1, '2019-06-20', 120000),
('Grace Lee', 2, '2021-03-12', 70000),
('Henry Clark', 3, '2022-09-01', 65000);

-- ================================
-- SCHEMA 3: Yelp-style JSON
-- ================================
DROP TABLE IF EXISTS yelp_business_data;

CREATE TABLE yelp_business_data (
    business_id TEXT PRIMARY KEY,
    name TEXT,
    categories TEXT,
    attributes TEXT,
    hours TEXT,
    city TEXT,
    stars REAL,
    review_count INTEGER
);

INSERT INTO yelp_business_data VALUES
('b1', 'Joe’s Pizza', 'Restaurant, Italian', '{"DogsAllowed": "True", "BusinessAcceptsCreditCards": "True"}', '{"Saturday": "11:00-23:00", "Sunday": "12:00-21:00"}', 'Philadelphia', 5, 320),
('b2', 'Healthy Eats', 'Restaurant, Vegan', '{"DogsAllowed": "False", "BusinessAcceptsCreditCards": "True"}', '{"Saturday": "09:00-21:00", "Sunday": "Closed"}', 'Philadelphia', 4.5, 150);

-- ================================
-- SCHEMA 4: ETL Stage
-- ================================
DROP TABLE IF EXISTS staging_orders;

CREATE TABLE staging_orders (
    order_id INTEGER,
    customer_name TEXT,
    product_name TEXT,
    order_date TEXT
);

INSERT INTO staging_orders VALUES
(1, 'Alice Johnson', 'Laptop', '2023-05-01'),
(1, 'Alice Johnson', 'Laptop', '2023-05-01'), -- Duplicate
(2, 'Bob Smith', 'Office Chair', '2023-05-03');
```

---

## **Step 2 – How We’ll Use It**

This DB lets you practice:

* Joins (`INNER`, `LEFT`, `RIGHT` equivalent in SQLite via `LEFT JOIN`)
* Aggregations & window functions
* CTEs & subqueries
* JSON-like data handling in SQLite (`json_extract`)
* ETL cleaning tasks (deduplication, transformations)

---
