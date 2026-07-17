# Part 3 — The Complete SQL Handbook
## Module 19: Industry Case Studies

> **Goal of this module:** Apply everything from Modules 1–18 to complete, realistic SQL solutions across six real-world domains — Banking, E-commerce, Student Management, Hospital, Inventory, and Food Delivery. Each case study includes a schema design (with deliberate constraint/type choices) and a set of representative business queries, explicitly citing which earlier modules each technique draws from.

---

### 📍 Where This Module Fits

```
Module 18: Interview rehearsal  →  Module 19: End-to-end applied practice (You are here)  →  Module 20: Preview of Part 4
```

---

## 19.1 Case Study: Banking System

### Schema Design

```sql
CREATE TABLE customers (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,   -- Module 4
    name        VARCHAR(150) NOT NULL,                            -- Module 5
    email       VARCHAR(255) NOT NULL UNIQUE,                     -- Module 6
    created_at  TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP     -- Module 5, Section 5.5
);

CREATE TABLE accounts (
    id           INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  INT NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,  -- Module 6: never orphan financial records
    account_type VARCHAR(20) NOT NULL CHECK (account_type IN ('savings', 'checking')),
    balance      NUMERIC(14,2) NOT NULL DEFAULT 0 CHECK (balance >= 0),      -- Module 5: NUMERIC for money, never FLOAT
    opened_on    DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE transactions (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, -- BIGINT: high-volume table (Module 5, Section 5.2)
    account_id    INT NOT NULL REFERENCES accounts(id) ON DELETE RESTRICT,
    txn_type      VARCHAR(10) NOT NULL CHECK (txn_type IN ('deposit', 'withdrawal', 'transfer')),
    amount        NUMERIC(14,2) NOT NULL CHECK (amount > 0),
    txn_timestamp TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

> 💡 **Design notes:** `ON DELETE RESTRICT` (Module 6, Section 6.4) is used deliberately throughout — financial records must never be silently cascaded away; a customer or account with transaction history should require explicit, deliberate handling before deletion. `NUMERIC` (never `FLOAT`) is non-negotiable for any monetary column (Module 5, Section 5.2).

### Representative Queries

**Q: Compute each account's running balance verification (sum of deposits minus withdrawals).**
```sql
SELECT
    a.id AS account_id,
    a.balance AS stated_balance,
    COALESCE(SUM(CASE WHEN t.txn_type = 'deposit' THEN t.amount ELSE 0 END), 0)
      - COALESCE(SUM(CASE WHEN t.txn_type = 'withdrawal' THEN t.amount ELSE 0 END), 0) AS computed_balance
FROM accounts a
LEFT JOIN transactions t ON t.account_id = a.id
GROUP BY a.id, a.balance;
```
*(Conditional aggregation, Module 10/11; `LEFT JOIN` to include accounts with zero transactions, Module 12.)*

**Q: Find customers with more than one account.**
```sql
SELECT c.name, COUNT(a.id) AS account_count
FROM customers c
JOIN accounts a ON a.customer_id = c.id
GROUP BY c.name
HAVING COUNT(a.id) > 1;
```
*(`GROUP BY`/`HAVING`, Module 11.)*

**Q: Safely transfer funds between two accounts (atomic UPSERT-adjacent pattern, transactional in spirit — full transaction syntax in Part 4).**
```sql
UPDATE accounts SET balance = balance - 500 WHERE id = 1 AND balance >= 500;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
INSERT INTO transactions (account_id, txn_type, amount) VALUES (1, 'transfer', 500);
```
*(The `WHERE balance >= 500` guard on the debit prevents an overdraft — this sequence should be wrapped in an explicit transaction, `BEGIN`/`COMMIT`, fully covered in Part 4, to guarantee atomicity.)*

---

## 19.2 Case Study: E-commerce System

### Schema Design

```sql
CREATE TABLE products (
    id       INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name     VARCHAR(200) NOT NULL,
    price    NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    stock    INT NOT NULL DEFAULT 0 CHECK (stock >= 0),
    tags     TEXT[]    -- PostgreSQL array, Module 5, Section 5.11
);

CREATE TABLE customers (
    id    INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  VARCHAR(150) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE orders (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id INT REFERENCES customers(id) ON DELETE SET NULL,   -- Module 6: retain order history
    status      VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','shipped','delivered','cancelled')),
    ordered_at  TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    id         INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id   INT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,  -- Module 6: items meaningless without the order
    product_id INT NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    quantity   INT NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10,2) NOT NULL,
    line_total NUMERIC(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED   -- Module 4, Section 4.10
);
```

### Representative Queries

**Q: Top 5 best-selling products by total quantity sold.**
```sql
SELECT p.name, SUM(oi.quantity) AS total_sold
FROM order_items oi
JOIN products p ON p.id = oi.product_id
GROUP BY p.name
ORDER BY total_sold DESC
LIMIT 5;
```
*(`GROUP BY` + `ORDER BY` + `LIMIT`, Modules 8/11.)*

**Q: Total revenue per order, using the generated `line_total` column.**
```sql
SELECT o.id AS order_id, SUM(oi.line_total) AS order_total
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id;
```

**Q: Customers who have never placed an order (anti-join, Module 12/13).**
```sql
SELECT c.name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

**Q: Atomic restock UPSERT.**
```sql
INSERT INTO products (id, name, price, stock) VALUES (101, 'Widget', 9.99, 20)
ON CONFLICT (id) DO UPDATE SET stock = products.stock + EXCLUDED.stock;
```
*(Module 7, Section 7.5.)*

---

## 19.3 Case Study: Student Management System

### Schema Design

```sql
CREATE TABLE students (
    id         INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name       VARCHAR(150) NOT NULL,
    enrolled_on DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE courses (
    id     INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title  VARCHAR(200) NOT NULL,
    credits SMALLINT NOT NULL CHECK (credits > 0)
);

-- Junction table (Module 6/12) — models the many-to-many relationship
CREATE TABLE enrollments (
    student_id INT NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id  INT NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    grade      CHAR(2) CHECK (grade IN ('A','B','C','D','F') OR grade IS NULL),
    PRIMARY KEY (student_id, course_id)    -- composite primary key, Module 6, Section 6.2
);
```

### Representative Queries

**Q: Each student's grade point summary (assuming A=4, B=3, C=2, D=1, F=0).**
```sql
SELECT
    s.name,
    ROUND(AVG(
        CASE e.grade
            WHEN 'A' THEN 4 WHEN 'B' THEN 3 WHEN 'C' THEN 2 WHEN 'D' THEN 1 WHEN 'F' THEN 0
        END
    ), 2) AS gpa
FROM students s
JOIN enrollments e ON e.student_id = s.id
WHERE e.grade IS NOT NULL
GROUP BY s.name;
```
*(`CASE`-based mapping inside an aggregate, Modules 10/11.)*

**Q: Students enrolled in more courses than the average student.**
```sql
SELECT s.name, COUNT(e.course_id) AS course_count
FROM students s
JOIN enrollments e ON e.student_id = s.id
GROUP BY s.name
HAVING COUNT(e.course_id) > (
    SELECT AVG(course_count) FROM (
        SELECT COUNT(*) AS course_count FROM enrollments GROUP BY student_id
    ) AS per_student
);
```
*(Derived table + `HAVING` with a subquery, Modules 11/13.)*

**Q: Courses with no enrolled students.**
```sql
SELECT c.title FROM courses c
WHERE NOT EXISTS (SELECT 1 FROM enrollments e WHERE e.course_id = c.id);
```

---

## 19.4 Case Study: Hospital Management System

### Schema Design

```sql
CREATE TABLE patients (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        VARCHAR(150) NOT NULL,
    date_of_birth DATE NOT NULL,
    blood_type  VARCHAR(5)
);

CREATE TABLE doctors (
    id           INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name         VARCHAR(150) NOT NULL,
    specialization VARCHAR(100) NOT NULL
);

CREATE TABLE appointments (
    id             INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    patient_id     INT NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
    doctor_id      INT NOT NULL REFERENCES doctors(id) ON DELETE RESTRICT,   -- preserve appointment history
    scheduled_at   TIMESTAMPTZ NOT NULL,
    status         VARCHAR(20) NOT NULL DEFAULT 'scheduled'
                       CHECK (status IN ('scheduled','completed','cancelled','no_show'))
);
```

### Representative Queries

**Q: Each doctor's appointment load, broken down by status (conditional aggregation).**
```sql
SELECT
    d.name,
    COUNT(CASE WHEN a.status = 'completed' THEN 1 END) AS completed,
    COUNT(CASE WHEN a.status = 'cancelled' THEN 1 END) AS cancelled,
    COUNT(CASE WHEN a.status = 'no_show' THEN 1 END)   AS no_shows
FROM doctors d
LEFT JOIN appointments a ON a.doctor_id = d.id
GROUP BY d.name;
```

**Q: Patients with more than 3 appointments in the last 30 days.**
```sql
SELECT p.name, COUNT(*) AS appointment_count
FROM patients p
JOIN appointments a ON a.patient_id = p.id
WHERE a.scheduled_at >= CURRENT_DATE - INTERVAL '30 days'   -- Module 5/10 date arithmetic
GROUP BY p.name
HAVING COUNT(*) > 3;
```

**Q: Doctors who have never had a cancelled appointment (safe NULL-aware pattern).**
```sql
SELECT d.name FROM doctors d
WHERE NOT EXISTS (
    SELECT 1 FROM appointments a WHERE a.doctor_id = d.id AND a.status = 'cancelled'
);
```
*(Deliberately `NOT EXISTS`, not `NOT IN`, per Module 9's `NULL` safety lesson.)*

---

## 19.5 Case Study: Inventory Management System

### Schema Design

```sql
CREATE TABLE warehouses (
    id   INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE items (
    id      INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku     VARCHAR(50) NOT NULL UNIQUE,
    name    VARCHAR(200) NOT NULL
);

CREATE TABLE stock_levels (
    warehouse_id INT NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    item_id      INT NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    quantity     INT NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    PRIMARY KEY (warehouse_id, item_id)
);

CREATE TABLE stock_movements (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    item_id      INT NOT NULL REFERENCES items(id),
    warehouse_id INT NOT NULL REFERENCES warehouses(id),
    change_qty   INT NOT NULL,     -- positive = stock in, negative = stock out
    moved_at     TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Representative Queries

**Q: Items with total stock below a reorder threshold, across all warehouses.**
```sql
SELECT i.name, SUM(s.quantity) AS total_stock
FROM items i
JOIN stock_levels s ON s.item_id = i.id
GROUP BY i.name
HAVING SUM(s.quantity) < 50;
```

**Q: Verify stock_levels against the movement ledger (reconciliation query).**
```sql
SELECT
    sl.warehouse_id, sl.item_id, sl.quantity AS recorded_stock,
    COALESCE(SUM(sm.change_qty), 0) AS computed_stock
FROM stock_levels sl
LEFT JOIN stock_movements sm
    ON sm.warehouse_id = sl.warehouse_id AND sm.item_id = sl.item_id
GROUP BY sl.warehouse_id, sl.item_id, sl.quantity
HAVING sl.quantity <> COALESCE(SUM(sm.change_qty), 0);   -- flags discrepancies
```
*(A realistic, genuinely useful data-integrity-auditing query pattern.)*

**Q: UPSERT to atomically adjust stock after a shipment received.**
```sql
INSERT INTO stock_levels (warehouse_id, item_id, quantity)
VALUES (1, 42, 100)
ON CONFLICT (warehouse_id, item_id)
DO UPDATE SET quantity = stock_levels.quantity + EXCLUDED.quantity;
```

---

## 19.6 Case Study: Food Delivery System

### Schema Design

```sql
CREATE TABLE restaurants (
    id     INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name   VARCHAR(150) NOT NULL,
    rating NUMERIC(2,1) CHECK (rating BETWEEN 0 AND 5)
);

CREATE TABLE menu_items (
    id            INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    restaurant_id INT NOT NULL REFERENCES restaurants(id) ON DELETE CASCADE,
    name          VARCHAR(150) NOT NULL,
    price         NUMERIC(8,2) NOT NULL CHECK (price >= 0)
);

CREATE TABLE delivery_orders (
    id            INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_name VARCHAR(150) NOT NULL,
    restaurant_id INT NOT NULL REFERENCES restaurants(id) ON DELETE RESTRICT,
    status        VARCHAR(20) NOT NULL DEFAULT 'placed'
                     CHECK (status IN ('placed','preparing','out_for_delivery','delivered','cancelled')),
    placed_at     TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    delivered_at  TIMESTAMPTZ
);

CREATE TABLE order_line_items (
    order_id     INT NOT NULL REFERENCES delivery_orders(id) ON DELETE CASCADE,
    menu_item_id INT NOT NULL REFERENCES menu_items(id),
    quantity     INT NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, menu_item_id)
);
```

### Representative Queries

**Q: Average delivery time per restaurant (only for delivered orders).**
```sql
SELECT
    r.name,
    AVG(EXTRACT(EPOCH FROM (o.delivered_at - o.placed_at)) / 60) AS avg_delivery_minutes
FROM restaurants r
JOIN delivery_orders o ON o.restaurant_id = r.id
WHERE o.status = 'delivered'
GROUP BY r.name;
```
*(Date/time arithmetic via `EXTRACT(EPOCH FROM ...)`, Module 10, Section 10.5.)*

**Q: Top-rated restaurants with at least 10 delivered orders (combining a rating filter and a `HAVING` count filter).**
```sql
SELECT r.name, r.rating, COUNT(o.id) AS delivered_orders
FROM restaurants r
JOIN delivery_orders o ON o.restaurant_id = r.id AND o.status = 'delivered'
WHERE r.rating >= 4.0
GROUP BY r.name, r.rating
HAVING COUNT(o.id) >= 10
ORDER BY r.rating DESC;
```
*(Demonstrates `WHERE` for row-level filtering combined with `HAVING` for group-level filtering, Module 11, Section 11.5 — a realistic, natural use of both together.)*

**Q: Total revenue per restaurant from delivered orders.**
```sql
SELECT r.name, SUM(mi.price * oli.quantity) AS total_revenue
FROM restaurants r
JOIN delivery_orders o ON o.restaurant_id = r.id AND o.status = 'delivered'
JOIN order_line_items oli ON oli.order_id = o.id
JOIN menu_items mi ON mi.id = oli.menu_item_id
GROUP BY r.name
ORDER BY total_revenue DESC;
```

---

## 19.7 Cross-Case-Study Patterns Worth Noticing

Across all six domains, several recurring design patterns appear deliberately, reinforcing this handbook's cumulative lessons:

| Pattern | Appears In | Module Reference |
|---|---|---|
| `NUMERIC` for all monetary values, never `FLOAT` | Banking, E-commerce, Food Delivery | Module 5, Section 5.2 |
| `ON DELETE RESTRICT` for financial/audit-critical relationships | Banking, Hospital, Food Delivery | Module 6, Section 6.4 |
| `ON DELETE CASCADE` for genuinely dependent child rows | E-commerce (order_items), Inventory (stock_levels) | Module 6, Section 6.4 |
| `ON DELETE SET NULL` to preserve historical records | E-commerce (orders.customer_id) | Module 6, Section 6.4 |
| Composite primary keys for junction/many-to-many tables | Student Management, Inventory, Food Delivery | Module 6, Section 6.2 |
| `NOT EXISTS` anti-join over `NOT IN` | Student Management, Hospital, E-commerce | Module 9, Section 9.5 |
| Conditional aggregation (`CASE` inside aggregate functions) | Banking, Hospital, Student Management | Modules 10–11 |
| Generated/computed columns | E-commerce (`line_total`) | Module 4, Section 4.10 |
| Atomic UPSERT for stock/inventory adjustments | E-commerce, Inventory | Module 7, Section 7.5 |

> 💡 **This consolidation is itself the point:** Real schema design and query-writing is not about applying isolated tricks — it's about recognizing which of a **relatively small, well-understood set of patterns** (covered across Modules 1–18) applies to the specific relationship and business requirement in front of you.

---

## 19.8 Interview Importance

Case-study-style questions ("design a schema for X") are extremely common in system-design-adjacent interview rounds, particularly for backend and database-focused roles — being able to fluently justify constraint choices (why `CASCADE` here but `RESTRICT` there), type choices (`NUMERIC` vs `INT`), and query patterns (anti-joins, conditional aggregation) in the context of a *real, concrete domain* is a much stronger signal than reciting definitions in isolation.

---

## 19.9 Chapter Summary

This module applied the full breadth of Modules 1–18 to six realistic domains, deliberately varying constraint choices (`CASCADE` vs `RESTRICT` vs `SET NULL`) based on genuine business semantics, and reinforcing recurring patterns (anti-joins, conditional aggregation, generated columns, UPSERT) across different real-world contexts — the clearest possible demonstration that SQL mastery is about applying a compact set of principles fluently, not memorizing domain-specific tricks.

---

## 19.10 Exercises

### 🟢 Beginner
1. Implement the Banking schema (Section 19.1) in your own PostgreSQL/MySQL instance and insert sample data for at least 3 customers, 5 accounts, and 10 transactions.
2. Run the "running balance verification" query and confirm it matches your inserted data.

### 🟡 Intermediate
3. Implement the E-commerce schema (Section 19.2) and write a query showing each customer's total lifetime spend, including customers with zero orders (showing 0, not being omitted).
4. Implement the Student Management schema (Section 19.3) and write a query listing each course alongside its enrollment count, sorted descending.

### 🔴 Advanced
5. Implement the Hospital schema (Section 19.4) and write a query identifying doctors whose no-show rate (no_shows / total appointments) exceeds 20%, safely guarding against division by zero.
6. Implement the Inventory schema (Section 19.5) and write the reconciliation query, then deliberately insert a discrepancy and confirm the query flags it correctly.

### 🧩 Scenario-Based
7. For the Food Delivery schema (Section 19.6), design and justify the `ON DELETE` behavior for `delivery_orders.restaurant_id` if a restaurant closes down — should historical orders be preserved, cascaded, or nulled? Justify your choice.

### 🏗️ Mini Project
8. Choose one case study from this module and extend it with at least two additional tables and five additional realistic business queries not covered here, applying at least one pattern from each of Modules 6 (constraints), 9 (NULL safety), 11 (aggregation), and 12 (joins).

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [DataLemur](https://datalemur.com/)
- [StrataScratch](https://www.stratascratch.com/)

---

## ✅ Module 19 Complete

You've now applied the entire handbook's cumulative knowledge to six complete, realistic domains — the clearest demonstration yet that SQL mastery comes from fluently recognizing and combining a compact set of well-understood patterns, not memorizing isolated tricks per use case.

**Next up: Module 20 — Preview of Part 4**, where we look ahead to Relational Algebra, Relational Calculus, Window Functions, CTEs, Recursive Queries, and Query Optimization.

---

*Reply "continue" or "next module" when you're ready for Module 20 — the final module of Part 3.*
