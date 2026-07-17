# Part 3 — The Complete SQL Handbook
## Module 14: Views

> **Goal of this module:** Understand views as stored, reusable queries — regular views, an introduction to materialized views, updatable views, and view-based security patterns. Views are where query logic itself becomes a reusable, named database object.

---

### 📍 Where This Module Fits

```
Module 13: Nested queries  →  Module 14: Naming/saving queries as objects (You are here)  →  Module 15: Execution Order
```

---

## 14.1 What Is a View?

A **view** is a **named, stored `SELECT` query** that behaves like a virtual table — you can `SELECT` from it, filter it, join it with other tables, exactly as if it were a real table, but it stores **no data of its own** (with the notable exception of materialized views, Section 14.5). Every time you query a regular view, the database **re-runs the underlying `SELECT` statement** against the live, current data.

```sql
CREATE VIEW active_employees AS
SELECT id, name, department, salary
FROM employees
WHERE status = 'active';

-- Query the view exactly like a table:
SELECT * FROM active_employees WHERE department = 'Engineering';
```

> 🧠 **Real-life analogy:** A view is like a **saved search filter** in an email client or a photo gallery app — you don't create a *copy* of your emails when you save a "Starred + Unread" filter; you're saving the **definition** of that filter, and every time you open it, it re-evaluates against your current, live inbox. A view works exactly the same way, but for SQL queries.

### Why Views Exist

| Purpose | Explanation |
|---|---|
| **Simplify complexity** | Hide a complicated join/aggregation behind a simple, reusable name — `SELECT * FROM active_employees` instead of repeating the full filtering logic everywhere |
| **Consistency** | Centralize business logic (e.g., "what counts as an 'active' employee") in one place, rather than duplicating the same `WHERE` condition across dozens of application queries |
| **Security** | Expose only specific columns/rows to certain users, without granting them direct access to the full underlying table (Section 14.4) |
| **Abstraction / backward compatibility** | Let the underlying table structure evolve while keeping a stable, consistent interface for consumers of the view |

---

## 14.2 Internal Working

When `CREATE VIEW` runs, the engine stores only the view's **query definition** (its parsed representation) in the system catalog — **no data is copied or materialized**. When you later query the view:

```
SELECT * FROM active_employees WHERE department = 'Engineering';
```

1. The parser (**Module 1**) recognizes `active_employees` as a view, not a base table.
2. It **substitutes the view's stored query definition** in place of the view name — this is often called **"view expansion"** or **"query rewriting"**.
3. The **outer query's conditions get combined with the view's own definition**, and the whole thing is optimized and executed as a single, unified query against the real underlying tables.

```
Your query:                          Effectively rewritten internally to:

SELECT * FROM active_employees        SELECT id, name, department, salary
WHERE department = 'Engineering';     FROM employees
                                       WHERE status = 'active'
                                         AND department = 'Engineering';
```

> 💡 **Genuinely important performance implication:** Because the view is expanded and merged with your outer query **before** the optimizer (Module 1) makes its final execution plan decisions, the optimizer can often use indexes on the underlying table (e.g., an index on `department`) exactly as if you'd written the combined query by hand — a **regular view generally adds no meaningful performance overhead** compared to writing the equivalent full query directly, since it's genuinely just a stored, reusable text substitution, not a separate execution step.

---

## 14.3 Basic View Operations

```sql
-- Create
CREATE VIEW view_name AS SELECT ...;

-- Create or replace (avoids DROP + CREATE for simple redefinitions)
CREATE OR REPLACE VIEW view_name AS SELECT ...;   -- both PostgreSQL and MySQL support this

-- Drop
DROP VIEW IF EXISTS view_name;

-- Inspect a view's definition
SELECT definition FROM pg_views WHERE viewname = 'view_name';   -- PostgreSQL
SHOW CREATE VIEW view_name;                                      -- MySQL
```

> ⚠️ **A real, worth-knowing limitation:** `CREATE OR REPLACE VIEW` in **both** engines can only change the query definition — it **cannot** remove columns or reorder existing ones in a way that changes their position, and generally cannot change column data types. Adding new columns at the **end** of the column list is fine; removing or reordering existing ones requires a full `DROP VIEW` + `CREATE VIEW`.

### Views Built on Joins and Aggregates

```sql
CREATE VIEW department_summary AS
SELECT
    d.name AS department,
    COUNT(e.id) AS headcount,
    ROUND(AVG(e.salary), 2) AS avg_salary
FROM departments d
LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.name;

SELECT * FROM department_summary WHERE headcount > 5;
```

This directly builds on **Modules 11–12** — a view can encapsulate arbitrarily complex joins, aggregations, and filters behind one simple, reusable name.

---

## 14.4 Views for Security

A genuinely important, real-world use case: views let you grant **restricted access** to a subset of a table's columns or rows, without exposing the full underlying table.

```sql
-- Underlying table has sensitive columns (salary, ssn, etc.)
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    salary NUMERIC(10,2),
    ssn VARCHAR(20)
);

-- A restricted view exposing only non-sensitive columns
CREATE VIEW employee_directory AS
SELECT id, name, department
FROM employees;

-- Grant access to the VIEW only, not the underlying table (DCL, briefly previewed here)
GRANT SELECT ON employee_directory TO hr_readonly_role;
```

With this setup, `hr_readonly_role` can query `employee_directory` freely but has **no way to see `salary` or `ssn`** at all — those columns simply don't exist from that role's perspective, since the view's definition never includes them. This is a genuinely powerful, real, commonly-used security pattern, especially useful when a single underlying table mixes sensitive and non-sensitive data that different consumers need different levels of access to.

> 💡 **Row-level security preview:** Views can also restrict **rows**, not just columns (e.g., `CREATE VIEW my_region_orders AS SELECT * FROM orders WHERE region = 'APAC'`) — genuinely useful for multi-tenant or regional access patterns. PostgreSQL additionally offers a more granular, native **Row-Level Security (RLS)** feature (policies enforced directly on base tables, not just via views) — a more advanced topic outside this handbook's core SQL scope, but worth knowing exists for genuinely fine-grained security requirements.

---

## 14.5 Materialized Views (Introduction)

A **materialized view**, unlike a regular view, **physically stores its query's result set on disk** at creation time — it behaves like a genuine snapshot/cache of the query's output, not a live re-computation on every access.

```sql
-- PostgreSQL
CREATE MATERIALIZED VIEW department_summary_mv AS
SELECT d.name AS department, COUNT(e.id) AS headcount, AVG(e.salary) AS avg_salary
FROM departments d
LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.name;

-- Querying it is instant — reads the stored snapshot, no recomputation
SELECT * FROM department_summary_mv;

-- Refreshing it (re-runs the query, updates the stored snapshot)
REFRESH MATERIALIZED VIEW department_summary_mv;
REFRESH MATERIALIZED VIEW CONCURRENTLY department_summary_mv;   -- allows reads during refresh (PostgreSQL)
```

### Regular View vs Materialized View — The Core Tradeoff

| Aspect | Regular View | Materialized View |
|---|---|---|
| Data storage | None — always computed live | **Physically stored** on disk |
| Freshness | Always up-to-date (reflects live data instantly) | **Stale** until explicitly refreshed |
| Query speed | Same as running the underlying query directly | **Very fast** — just reads the pre-computed snapshot |
| Cost | No storage cost; recomputation cost on every query | Storage cost; refresh cost, but cheap reads |
| Best for | Simplifying/reusing query logic on frequently-changing data | Expensive aggregations/reports on large data where slight staleness is acceptable |

> 🧠 **Real-life analogy:** A regular view is like asking someone to **recalculate a complex answer from scratch every single time you ask** — always perfectly accurate, but potentially slow if the calculation is expensive. A materialized view is like **writing that answer down on a sticky note** after calculating it once — instantly readable afterward, but the note can go stale until you deliberately recalculate and rewrite it.

### PostgreSQL vs MySQL — Materialized Views

> ⚠️ **Genuine, significant PostgreSQL vs MySQL feature gap:** **PostgreSQL has native `MATERIALIZED VIEW` support** (since version 9.3). **MySQL has no native materialized view feature at all.** The standard MySQL workaround is to **manually simulate** one: create a **real table**, populate it with `INSERT ... SELECT` (**Module 7**), and periodically refresh it using a scheduled job (a `TRUNCATE` + `INSERT ... SELECT`, or MySQL's built-in **`EVENT`** scheduler) — genuinely more manual, error-prone bookkeeping compared to PostgreSQL's dedicated, purpose-built feature.

```sql
-- MySQL manual "materialized view" simulation
CREATE TABLE department_summary_snapshot (
    department VARCHAR(100),
    headcount INT,
    avg_salary DECIMAL(10,2)
);

-- Refresh logic (run periodically via a scheduled job/EVENT):
TRUNCATE TABLE department_summary_snapshot;
INSERT INTO department_summary_snapshot
SELECT d.name, COUNT(e.id), AVG(e.salary)
FROM departments d LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.name;
```

### Why Materialized Views Matter for Performance

Materialized views are a genuinely important, real tool for expensive, frequently-run aggregation queries — dashboards that would otherwise re-scan and re-aggregate millions of rows on every page load can instead read an instantly-fast, pre-computed snapshot, refreshed on a schedule (e.g., every 15 minutes, or nightly) appropriate to how "fresh" the data genuinely needs to be for that use case.

---

## 14.6 Updatable Views

Under certain conditions, you can `INSERT`/`UPDATE`/`DELETE` **through** a view, and the changes are automatically applied to the underlying base table.

```sql
CREATE VIEW active_employees AS
SELECT id, name, department, salary
FROM employees
WHERE status = 'active';

UPDATE active_employees SET salary = 95000 WHERE id = 1;
-- This actually updates the underlying `employees` table's row where id = 1
```

### Conditions for a View to Be Updatable (ANSI Standard Rules, Both Engines)

A view is generally updatable **only if**:
- It's based on a **single table** (no joins) — though PostgreSQL, via `INSTEAD OF` triggers (a more advanced topic), can make even join-based views updatable with custom logic.
- It doesn't use `DISTINCT`, `GROUP BY`, `HAVING`, aggregate functions, `UNION`, or window functions.
- Every `NOT NULL` column without a `DEFAULT` in the underlying table is included in the view's column list (otherwise an `INSERT` through the view would violate that constraint with no way to supply a value).

> ⚠️ **A genuinely subtle, real gotcha:** A view like `active_employees` (filtering `WHERE status = 'active'`) is updatable — but what happens if you `UPDATE active_employees SET status = 'inactive' WHERE id = 1`? The underlying row is updated correctly, but it **no longer matches the view's own `WHERE` clause** — it effectively "disappears" from the view immediately after the update. PostgreSQL and MySQL both support an optional `WITH CHECK OPTION` clause specifically to **prevent** this kind of self-contradicting update:

```sql
CREATE VIEW active_employees AS
SELECT id, name, department, salary, status
FROM employees
WHERE status = 'active'
WITH CHECK OPTION;

UPDATE active_employees SET status = 'inactive' WHERE id = 1;
-- ERROR: new row violates check option for view "active_employees"
-- (WITH CHECK OPTION explicitly prevents updates that would make a row
--  no longer satisfy the view's own WHERE condition)
```

---

## 14.7 Full Worked Example

```sql
-- 1. A security-oriented view, hiding sensitive columns
CREATE VIEW public_employee_list AS
SELECT id, name, department
FROM employees;

-- 2. A reporting view, encapsulating a join + aggregation
CREATE VIEW department_headcount AS
SELECT d.name AS department, COUNT(e.id) AS headcount
FROM departments d
LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.name;

-- 3. A materialized view for an expensive, infrequently-changing report
CREATE MATERIALIZED VIEW yearly_payroll_summary AS
SELECT EXTRACT(YEAR FROM hired_on) AS hire_year, SUM(salary) AS total_payroll
FROM employees
GROUP BY EXTRACT(YEAR FROM hired_on);

-- Refresh the materialized view on a schedule (e.g., nightly via a cron job)
REFRESH MATERIALIZED VIEW yearly_payroll_summary;

-- 4. An updatable view with a safety guard
CREATE VIEW active_employees AS
SELECT id, name, department, salary, status
FROM employees
WHERE status = 'active'
WITH CHECK OPTION;
```

---

## 14.8 Advantages / Disadvantages

| Feature | Advantage | Watch-out |
|---|---|---|
| Regular views | No storage cost, always live/current, encapsulates complexity | Adds no performance benefit by itself — still runs the full underlying query each time |
| Materialized views | Very fast reads on expensive queries | Data can be stale; requires deliberate refresh management; not supported at all in MySQL |
| Security-oriented views | Restricts sensitive columns/rows cleanly | Requires disciplined DCL (`GRANT`) management alongside the view definition |
| Updatable views | Convenient, transparent write access | Easy to accidentally "update a row out of view" without `WITH CHECK OPTION` |

## 14.9 Best Practices

- Use views to **centralize** frequently-repeated business logic (e.g., "what counts as an active user") rather than duplicating `WHERE` conditions across many queries/applications.
- Use materialized views for genuinely expensive aggregations where **some staleness is acceptable** — never assume they're automatically fresh.
- Always add `WITH CHECK OPTION` to updatable views whose `WHERE` clause could be violated by the very updates performed through them.
- Prefer views over duplicating column-restriction logic for security purposes — a single, well-defined view is much easier to audit than scattered, ad-hoc column lists across many queries.

## 14.10 Common Mistakes

- Assuming a materialized view is always up-to-date without a defined refresh strategy (Section 14.5).
- Forgetting `WITH CHECK OPTION`, leading to confusing "my update succeeded but the row vanished from this view" behavior (Section 14.6).
- Attempting to `INSERT`/`UPDATE` through a view that joins multiple tables or uses aggregation, without realizing it's not updatable by default (Section 14.6).
- Assuming views themselves provide a performance boost — a regular view is just a stored query definition, not a cache (contrast directly with materialized views, Section 14.5).

## 14.11 Edge Cases

- Dropping or altering a base table's columns that a view depends on can break the view — PostgreSQL will generally prevent dropping a column that a view depends on unless `CASCADE` is used (connecting back to **Module 4, Section 4.3**'s discussion of `DROP TABLE ... CASCADE`).
- Nesting views (a view built on top of another view) is technically allowed in both engines but can create deeply hidden performance and readability issues — each layer of view expansion adds another layer of query rewriting complexity for the optimizer and for future readers to untangle.
- `REFRESH MATERIALIZED VIEW` without `CONCURRENTLY` (PostgreSQL) locks the materialized view against reads during the refresh — a real, practical operational consideration for materialized views feeding live-traffic dashboards.

## 14.12 Performance Implications

- Regular views add negligible overhead beyond the equivalent hand-written query, thanks to query rewriting/expansion (Section 14.2) — a genuinely important point that counters the common misconception that "views are inherently slow."
- Materialized views trade **write-time cost** (the refresh operation) for **read-time speed** — an explicit, deliberate tradeoff appropriate for read-heavy, refresh-tolerant reporting workloads.
- Deeply nested views (views built on views built on views) can, in some cases, confuse the optimizer's ability to produce an optimal combined plan — flattening overly nested view hierarchies is a legitimate, real performance troubleshooting technique.

## 14.13 Industry Usage

Views are used constantly in real production systems for exactly the purposes covered here: simplifying complex reporting logic into reusable names, restricting sensitive data access for different user roles, and (via materialized views, in PostgreSQL specifically) powering fast-loading dashboards over otherwise-expensive aggregate queries.

## 14.14 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Definition of a view; basic `CREATE VIEW` syntax |
| Placement | "What's the difference between a view and a table?" |
| Mid-level | "What's the difference between a regular view and a materialized view?" |
| Senior SDE-2/SDE-3 | "How would you design a security model using views to restrict column access for different roles?" / "MySQL has no materialized views — how would you simulate one, and what tradeoffs does that introduce?" |

---

## 14.15 Chapter Summary

- A **view** is a stored, named `SELECT` query — no data of its own, expanded/rewritten into the outer query at execution time, generally adding no meaningful overhead versus the equivalent hand-written query.
- Views are genuinely valuable for **simplifying complex logic**, **enforcing consistency**, and **restricting access** to sensitive columns/rows.
- **Materialized views** physically store their result set, trading data freshness for read speed — a real, significant **PostgreSQL-only feature** among the two engines this handbook covers; MySQL requires manual simulation via real tables and scheduled refresh jobs.
- **Updatable views** allow writes through to the underlying table under specific conditions (single table, no aggregation/`DISTINCT`/`GROUP BY`) — always pair filtering `WHERE` conditions with `WITH CHECK OPTION` to prevent "updating a row out of the view."

---

## 14.16 Revision Notes / Cheat Sheet

```sql
CREATE VIEW v AS SELECT ...;
CREATE OR REPLACE VIEW v AS SELECT ...;
DROP VIEW IF EXISTS v;

CREATE MATERIALIZED VIEW mv AS SELECT ...;    -- PostgreSQL only
REFRESH MATERIALIZED VIEW mv;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv;     -- allows concurrent reads during refresh

CREATE VIEW v AS SELECT ... WHERE ... WITH CHECK OPTION;   -- prevents "updating out of view"

⚠ MySQL has NO materialized views — simulate with a real table + scheduled refresh
⚠ Regular views ≠ performance boost — they're just stored query text (query rewriting)
⚠ Updatable views: single table only, no DISTINCT/GROUP BY/aggregates/UNION
```

---

## 14.17 Multiple Choice Questions (MCQs)

1. Does a regular (non-materialized) view store data?
   a) Yes, always b) **No — it's re-computed from the live base tables on every query** c) Only the first time it's queried d) Only if indexed

2. Which engine has NO native materialized view support?
   a) PostgreSQL b) **MySQL** c) Both support it identically d) Neither supports it

3. What does `WITH CHECK OPTION` prevent?
   a) Deleting rows through a view b) **Updating a row through a view in a way that makes it no longer match the view's own WHERE clause** c) Creating the view at all d) Reading from the view

4. Which of these disqualifies a view from being updatable?
   a) A single-column `SELECT` b) A `WHERE` clause c) **Using `GROUP BY`/aggregate functions** d) Having an alias

5. What's the main tradeoff of a materialized view compared to a regular view?
   a) Materialized views are always slower b) **Materialized views trade data freshness for read speed** c) Materialized views cannot be queried with `WHERE` d) There is no tradeoff

---

## 14.18 Exercises

### 🟢 Beginner
1. Create a view showing only `name` and `department` from `employees`, excluding `salary`.
2. Create a view that joins `employees` and `departments`, showing employee name alongside department name.
3. Drop the view you created in exercise 1 using `DROP VIEW IF EXISTS`.

### 🟡 Intermediate
4. Create an updatable view filtering `status = 'active'`, add `WITH CHECK OPTION`, and demonstrate the error that occurs when attempting to update a row's status to `'inactive'` through that view.
5. In PostgreSQL, create a materialized view summarizing total salary per department, query it, then modify the underlying data and demonstrate that the materialized view is stale until `REFRESH MATERIALIZED VIEW` is run.
6. Design a security-oriented view exposing only non-sensitive columns from a table of your choice, and write the `GRANT` statement that would give a restricted role access to it (referencing DCL conceptually).

### 🔴 Advanced / Debugging
7. A view built with a `JOIN` between two tables fails when someone attempts an `INSERT` through it. Explain why, referencing Section 14.6's updatable-view conditions.
8. Propose a MySQL-compatible design (using a real table and a refresh strategy) that simulates the materialized view you built in exercise 5, and discuss what operational responsibility this shifts onto you compared to PostgreSQL's native feature.

### 🧩 Scenario-Based
9. Your team needs a dashboard showing complex, expensive multi-table aggregate statistics that are queried very frequently (every page load) but only need to be accurate as of the last hour. Would you use a regular view or a materialized view? Justify your answer.

### 🏗️ Mini Project
10. Using your practice schema, design a small "view layer" consisting of: one simplifying view (hiding complexity), one security-oriented view (restricting columns), and one materialized view (or its MySQL-simulated equivalent) for a genuinely expensive aggregate report — documenting the purpose of each.

### 🔗 Practice Links
- [PostgreSQL Docs — CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html)
- [PostgreSQL Docs — Materialized Views](https://www.postgresql.org/docs/current/rules-materializedviews.html)
- [MySQL Docs — CREATE VIEW](https://dev.mysql.com/doc/refman/8.0/en/create-view.html)
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)

---

## ✅ Module 14 Complete

You can now encapsulate complex query logic, build security-restricted data access layers, and understand the real, meaningful tradeoff between always-live regular views and pre-computed materialized views — including MySQL's notable lack of native materialized view support.

**Next up: Module 15 — SQL Execution Order**, where we finally, formally resolve the "written order vs. execution order" question that's been foreshadowed since Module 1: the true logical processing sequence of `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`.

---

*Reply "continue" or "next module" when you're ready for Module 15.*
