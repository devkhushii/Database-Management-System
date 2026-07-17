# Part 3 — The Complete SQL Handbook
## Module 7: DML (Data Manipulation)

> **Goal of this module:** Master every way to put data into, change data within, and remove data from tables — `INSERT` (single, multi-row, and from a query), `UPSERT`/conflict resolution, `MERGE`, `UPDATE`, `DELETE`, and the `RETURNING` clause. This is where the schemas from Modules 3–6 finally come alive with real data.

---

### 📍 Where This Module Fits

```
Module 6: Rules data must follow  →  Module 7: Putting/changing/removing data (You are here)  →  Module 8: Querying it
```

---

## 7.1 What is DML, Revisited

Recall **Module 1, Section 1.8**: DML (**Data Manipulation Language**) modifies the *data* inside tables, as opposed to DDL's modification of *structure*. Unlike most DDL (Module 3–4), DML statements are **always transactional** in both PostgreSQL and MySQL — they can be rolled back if wrapped in an explicit transaction (transactions get full dedicated treatment in Part 4, but we'll use `BEGIN`/`COMMIT`/`ROLLBACK` informally in this module's examples where useful).

---

## 7.2 INSERT — Single Row

### Basic Syntax

```sql
INSERT INTO employees (name, department, salary)
VALUES ('Ananya', 'Engineering', 95000);
```

### Column List — Why You Should Always Specify It

```sql
-- Works, but fragile:
INSERT INTO employees VALUES (DEFAULT, 'Ananya', 'Engineering', 95000, 'active', CURRENT_DATE);

-- Robust, explicit, and recommended:
INSERT INTO employees (name, department, salary)
VALUES ('Ananya', 'Engineering', 95000);
```

> ⚠️ **Extremely common real-world mistake:** Omitting the column list and relying on positional ordering. This works *until* someone adds, removes, or reorders a column via `ALTER TABLE` (Module 4) — at which point every unqualified `INSERT` silently starts writing values into the **wrong columns**, often without any error at all if the types happen to coincidentally match. **Always specify the column list explicitly** in production code; this single habit prevents an entire category of silent, hard-to-diagnose bugs.

### Internal Working

1. The parser validates the statement and resolves the target table/columns.
2. For any omitted columns, the engine substitutes their `DEFAULT` value (Module 6, Section 6.8), or `NULL` if no default exists and the column is nullable, or **raises an error** if the column is `NOT NULL` with no default.
3. All relevant constraints (Module 6) are checked: `NOT NULL`, `CHECK`, `UNIQUE`, `FOREIGN KEY`.
4. If any check fails, the **entire statement** is rejected — no partial insert occurs.
5. On success, any `GENERATED ALWAYS AS IDENTITY` (Module 4) or generated columns (Module 4, Section 4.10) are computed automatically.
6. The new row is written to the table's storage, and any indexes (Part 4) are updated to include it.

---

## 7.3 INSERT — Multiple Rows

```sql
INSERT INTO employees (name, department, salary) VALUES
    ('Rohan', 'Sales', 55000),
    ('Divya', 'Marketing', 60000),
    ('Karan', 'Engineering', 88000);
```

### Why Multi-Row INSERT Matters for Performance

> ⚠️ **Genuine, frequently-tested performance point:** Issuing 1,000 separate single-row `INSERT` statements (1,000 round-trips to the server, each with its own parsing/planning/constraint-checking/transaction overhead) is **dramatically slower** than issuing one multi-row `INSERT` containing 1,000 value tuples (a single round-trip, a single parse/plan, and typically more efficient batched constraint checking and index updates). This is a very common, very real optimization every backend engineer eventually learns the hard way when a naive loop-of-inserts becomes a performance bottleneck.

```
1,000 individual INSERTs:                    One multi-row INSERT (1,000 tuples):
Client → Server (x1000 round trips)          Client → Server (x1 round trip)
   parse, plan, execute (x1000)                  parse, plan, execute (x1, batched)
   ⏱ Slow — network + overhead dominates          ⏱ Much faster
```

> 💡 **Best practice for bulk loads:** For truly large data loads (tens of thousands+ rows), even multi-row `INSERT` batching has limits — dedicated bulk-load tools are typically faster still: PostgreSQL's `COPY` command (reads directly from a file/stream, bypassing much of the per-row SQL overhead) and MySQL's `LOAD DATA INFILE` are the industry-standard choices for genuinely large imports, well beyond what hand-written `INSERT` statements should handle.

---

## 7.4 INSERT ... SELECT

```sql
INSERT INTO archived_employees (name, department, salary)
SELECT name, department, salary
FROM employees
WHERE status = 'terminated';
```

**What it does:** Inserts the **result of a query** directly into a table, without ever materializing the data on the client side — the entire operation happens server-side in one atomic statement.

> 🧠 **Real-life analogy:** This is like directly photocopying pages from one filing cabinet drawer into another, rather than manually reading each page aloud and re-typing it — the data never has to "leave" the database and come back.

### Why It's Preferred Over "Read Then Write" from Application Code

Fetching rows into your application, then looping to `INSERT` them back (even into the same database) is **strictly slower and more error-prone** than a single `INSERT ... SELECT` — it involves unnecessary network round-trips, and loses the atomicity guarantee (Part 4) that a single server-side statement provides.

---

## 7.5 UPSERT (INSERT ... ON CONFLICT / ON DUPLICATE KEY UPDATE)

### The Problem It Solves

You often want: *"insert this row, but if a row with this key already exists, update it instead"* — a single atomic operation, rather than a racy "check if it exists, then decide to insert or update" sequence from application code (which is vulnerable to a **race condition**: another concurrent request could insert the same row between your check and your write).

### PostgreSQL: `ON CONFLICT`

```sql
INSERT INTO inventory (product_id, quantity)
VALUES (101, 50)
ON CONFLICT (product_id)
DO UPDATE SET quantity = inventory.quantity + EXCLUDED.quantity;
```

- `ON CONFLICT (product_id)` — specifies which constraint (typically a `UNIQUE` or `PRIMARY KEY` column) triggers the conflict-handling logic.
- `EXCLUDED` — a special, standard PostgreSQL pseudo-table referring to the **row that was proposed for insertion** (the values from your `VALUES` clause) — letting you reference "the new value that would have been inserted" inside the `UPDATE` branch.
- `DO NOTHING` is also available, to silently skip the insert on conflict instead of updating:

```sql
INSERT INTO inventory (product_id, quantity)
VALUES (101, 50)
ON CONFLICT (product_id) DO NOTHING;
```

### MySQL: `ON DUPLICATE KEY UPDATE`

```sql
INSERT INTO inventory (product_id, quantity)
VALUES (101, 50)
ON DUPLICATE KEY UPDATE quantity = quantity + VALUES(quantity);
```

- MySQL uses the function-like syntax `VALUES(column_name)` to refer to the proposed insert value (conceptually the same role as PostgreSQL's `EXCLUDED` pseudo-table, just different syntax).
- **Note:** `VALUES()` in this specific `ON DUPLICATE KEY UPDATE` context is **deprecated** as of MySQL 8.0.20 in favor of row aliasing syntax, though it remains widely used and functional:

```sql
-- Modern MySQL 8.0.19+ alternative syntax
INSERT INTO inventory (product_id, quantity)
VALUES (101, 50) AS new_row
ON DUPLICATE KEY UPDATE quantity = inventory.quantity + new_row.quantity;
```

### Internal Working

1. The engine attempts the `INSERT` as normal.
2. If it would violate a **unique constraint** (the one specified, for Postgres; any unique/primary key, for MySQL), instead of raising an error and aborting, the engine switches to executing the specified `UPDATE` logic against the conflicting existing row.
3. This entire sequence happens as a **single atomic operation** — no other concurrent transaction can "sneak in" between the failed insert attempt and the fallback update, which is exactly the race-condition protection this feature exists to provide.

> ⚠️ **Common mistake:** Trying to replicate UPSERT logic manually in application code with `SELECT ... then IF EXISTS UPDATE ELSE INSERT`. Under concurrent load, two simultaneous requests can both run the `SELECT`, both see "doesn't exist," and both attempt an `INSERT` — one succeeds, one fails with a unique constraint violation, or worse (depending on isolation level, covered in Part 4) you get a genuine duplicate. **Always prefer the database's native atomic UPSERT mechanism over hand-rolled check-then-act logic.**

### PostgreSQL vs MySQL — UPSERT Comparison

| Aspect | PostgreSQL | MySQL |
|---|---|---|
| Syntax | `ON CONFLICT (col) DO UPDATE/NOTHING` | `ON DUPLICATE KEY UPDATE` |
| Reference to proposed values | `EXCLUDED.column` | `VALUES(column)` (legacy) or row-alias syntax (modern) |
| Conflict target specification | Explicit — you name the constraint/column | Implicit — triggers on **any** unique/primary key violation |
| Skip-on-conflict option | `DO NOTHING` (clean, explicit) | No direct equivalent; commonly emulated with `INSERT IGNORE` (a broader, less precise statement that suppresses various errors, not just key conflicts) |

---

## 7.6 MERGE

`MERGE` is the **ANSI SQL-standard** statement (added in SQL:2003) for combining insert/update/delete logic based on whether a match exists between a target table and a source dataset — a more general, powerful cousin of UPSERT.

```sql
-- PostgreSQL (15+)
MERGE INTO inventory AS target
USING staging_updates AS source
ON target.product_id = source.product_id
WHEN MATCHED THEN
    UPDATE SET quantity = target.quantity + source.quantity
WHEN NOT MATCHED THEN
    INSERT (product_id, quantity) VALUES (source.product_id, source.quantity);
```

**MySQL does not support `MERGE`** at all (a genuine, real feature gap as of the versions covered in this handbook) — MySQL's `ON DUPLICATE KEY UPDATE` (Section 7.5) covers the most common single-table UPSERT case, but there's no equivalent to `MERGE`'s ability to reconcile against an arbitrary **source table/subquery** with multiple `WHEN MATCHED`/`WHEN NOT MATCHED` branches, including conditional deletes.

> 💡 **When `MERGE` is genuinely more powerful than UPSERT:** `MERGE` shines in data-warehousing/ETL scenarios — reconciling a full "staging" table of incoming changes against a production table in one atomic statement, potentially inserting new rows, updating changed rows, *and* deleting rows no longer present in the source, all in a single statement. This is a genuinely common real pattern in data engineering pipelines.

> ⚠️ **`MERGE` is relatively recent in PostgreSQL** (added in version 15) — many production PostgreSQL deployments still run older versions without it, making `INSERT ... ON CONFLICT` (Section 7.5) the more universally-available and still heavily-used tool for the common single-table UPSERT case even in Postgres-based systems.

---

## 7.7 UPDATE

### Basic Syntax

```sql
UPDATE employees
SET salary = salary * 1.10
WHERE department = 'Engineering';
```

> ⚠️ **The single most dangerous mistake in all of DML:** Omitting the `WHERE` clause. `UPDATE employees SET salary = salary * 1.10;` with **no `WHERE`** updates **every single row in the entire table** — a devastatingly common, real production incident pattern. **Always write and verify your `WHERE` clause — ideally by first running the equivalent `SELECT` with the same `WHERE` clause to confirm exactly which rows will be affected — before running the `UPDATE`.**

```sql
-- Best practice: verify first
SELECT * FROM employees WHERE department = 'Engineering';
-- Confirms the right rows... THEN run:
UPDATE employees SET salary = salary * 1.10 WHERE department = 'Engineering';
```

### Updating Multiple Columns

```sql
UPDATE employees
SET salary = 100000,
    status = 'active',
    department = 'Engineering'
WHERE id = 42;
```

### UPDATE with a JOIN (Engine-Specific Syntax Divergence)

A genuinely important, frequently-needed pattern: updating a table based on values from a *related* table.

```sql
-- PostgreSQL — uses FROM
UPDATE employees e
SET department = d.name
FROM departments d
WHERE e.department_id = d.id
  AND d.name = 'Engineering';

-- MySQL — uses JOIN directly in the UPDATE statement
UPDATE employees e
JOIN departments d ON e.department_id = d.id
SET e.department = d.name
WHERE d.name = 'Engineering';
```

> ⚠️ **A real, commonly-hit syntax divergence:** This is one of the most frequently-searched "why doesn't my query work in the other database" issues — PostgreSQL's `UPDATE ... FROM` syntax and MySQL's `UPDATE ... JOIN` syntax accomplish the identical logical result but are **not interchangeable syntax**; copy-pasting one into the other engine produces a syntax error.

### Internal Working

An `UPDATE` is conceptually (and, in many engines' internal implementation) closer to a **delete-then-insert** of the affected row than an in-place edit — especially relevant in PostgreSQL, whose **MVCC (Multi-Version Concurrency Control)** architecture (previewed here, covered fully in Part 4's transactions module) never truly overwrites a row in place: it writes an entirely **new row version** and marks the old version as obsolete for future cleanup (`VACUUM`). This is why heavy `UPDATE` workloads in PostgreSQL benefit significantly from regular vacuuming — a genuine operational concern in real production Postgres systems.

---

## 7.8 DELETE

### Basic Syntax

```sql
DELETE FROM employees WHERE status = 'terminated';
```

> ⚠️ **The `DELETE` twin of the `UPDATE` warning above:** `DELETE FROM employees;` with no `WHERE` clause removes **every row** in the table (though, unlike `TRUNCATE` from **Module 4, Section 4.6**, it does so row-by-row, is slower, and — critically — **does** fire any row-level triggers and **is** fully transactional/rollback-able in both engines, since it's genuine DML). Same best practice applies: verify with `SELECT` first.

### DELETE with a JOIN

```sql
-- PostgreSQL — uses USING
DELETE FROM employees e
USING departments d
WHERE e.department_id = d.id
  AND d.name = 'Discontinued';

-- MySQL — uses JOIN directly
DELETE e FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE d.name = 'Discontinued';
```

### DELETE vs TRUNCATE vs DROP — Full Comparison (Consolidating Module 3 & 4 Content)

| | `DELETE` | `TRUNCATE` | `DROP TABLE` |
|---|---|---|---|
| Category | DML | DDL | DDL |
| Removes | Specific rows (via `WHERE`) or all rows | All rows only | The entire table structure + all rows |
| Table structure afterward | Unchanged | Unchanged, empty | **Gone entirely** |
| Speed | Slower (row-by-row) | Fast | Fast |
| Triggers fired | Yes | Generally no | N/A |
| Rollback-able | Yes (both engines) | Yes (Postgres) / No, implicit commit (MySQL) | Yes (Postgres) / No (MySQL) |

---

## 7.9 RETURNING Clause

```sql
-- PostgreSQL (and MySQL only very recently/limitedly — see note below)
INSERT INTO employees (name, salary)
VALUES ('Neha', 72000)
RETURNING id, name, created_at;

UPDATE employees SET salary = salary * 1.05
WHERE department = 'Sales'
RETURNING id, name, salary;

DELETE FROM employees WHERE status = 'terminated'
RETURNING id, name;
```

**What it does:** Returns the affected row(s)' data **directly from the DML statement itself**, eliminating the need for a separate follow-up `SELECT` to retrieve, for example, an auto-generated ID immediately after an `INSERT`.

### Why It's a Genuinely Valuable Feature

Without `RETURNING`, retrieving a newly-generated `SERIAL`/`IDENTITY` value after an `INSERT` traditionally required a second round-trip — a follow-up query like PostgreSQL's `SELECT currval('employees_id_seq')` or MySQL's `SELECT LAST_INSERT_ID()`. `RETURNING` collapses this into a **single atomic round-trip**, which is both **faster** (no extra network round-trip) and **safer** (no risk of a race condition where a concurrent insert on the same connection/session changes what "the last inserted ID" refers to between your two separate statements).

> ⚠️ **Genuine, real PostgreSQL vs MySQL feature gap worth knowing:** `RETURNING` is a long-standing, fully-supported PostgreSQL feature (and part of the broader SQL:2003+ standard's optional feature set). **MySQL historically has no `RETURNING` support at all** for `INSERT`/`UPDATE`/`DELETE` (MySQL 8.0.21+ finally added table-level support in a much more limited context for certain use cases, but it remains far less commonly relied upon and less feature-complete than PostgreSQL's implementation). This is a real, practically significant reason some teams cite when comparing the two engines' ergonomics for application development. The idiomatic MySQL alternative for retrieving a newly-inserted ID remains the separate `SELECT LAST_INSERT_ID();` call.

---

## 7.10 Conflict Resolution — Consolidated View

We've now seen several related-but-distinct "what happens on conflict" mechanisms across this module — worth explicitly distinguishing them, since interviews sometimes probe whether candidates conflate them:

| Mechanism | Triggers On | Scope |
|---|---|---|
| `ON CONFLICT` (Postgres) / `ON DUPLICATE KEY UPDATE` (MySQL) | A single `INSERT` violating a unique/primary key | Handles the conflict inline, within the same statement |
| `MERGE` | Row-by-row matching against a source table/query | A full reconciliation operation across many rows, with insert/update/delete branches |
| Foreign key `ON DELETE`/`ON UPDATE` actions (**Module 6**) | A parent row being deleted/updated while child rows reference it | A referential-integrity-driven cascade, not a uniqueness conflict at all — a genuinely different kind of "conflict" |

> 💡 **Interview clarity tip:** If asked to "explain conflict resolution in SQL," it's a strong answer to explicitly name all three of these and clarify that they solve **different problems** — this demonstrates precise understanding rather than vague pattern-matching on the word "conflict."

---

## 7.11 Full Worked Example — DML Lifecycle

```sql
-- 1. Bulk insert initial data
INSERT INTO products (name, price, stock) VALUES
    ('Widget', 9.99, 100),
    ('Gadget', 19.99, 50),
    ('Gizmo', 14.99, 75);

-- 2. Atomic upsert on restock (PostgreSQL)
INSERT INTO products (name, price, stock)
VALUES ('Widget', 9.99, 25)
ON CONFLICT (name)
DO UPDATE SET stock = products.stock + EXCLUDED.stock
RETURNING name, stock;

-- 3. Bulk price adjustment with verification-first discipline
SELECT name, price FROM products WHERE price < 15;   -- verify affected rows first
UPDATE products SET price = price * 1.05 WHERE price < 15;

-- 4. Archive and remove discontinued items atomically
INSERT INTO archived_products SELECT * FROM products WHERE stock = 0;
DELETE FROM products WHERE stock = 0
RETURNING id, name;
```

---

## 7.12 Advantages / Disadvantages Summary

| Feature | Advantage | Watch-out |
|---|---|---|
| Multi-row `INSERT` | Far fewer round-trips, much faster bulk loads | Extremely large single statements can hit engine-specific size/packet limits |
| `INSERT ... SELECT` | Server-side, atomic, no client round-trip for the data itself | Can lock large portions of the source table during the read, depending on isolation level |
| UPSERT (`ON CONFLICT`/`ON DUPLICATE KEY UPDATE`) | Atomic, race-condition-free insert-or-update | Syntax is genuinely non-portable between engines |
| `MERGE` | Handles complex, multi-branch reconciliation in one statement | Not supported in MySQL at all; recent addition even in PostgreSQL (15+) |
| `RETURNING` | Avoids extra round-trip for generated values, race-condition-free | Very limited/absent in MySQL |

## 7.13 Best Practices

- Always specify the column list explicitly in `INSERT` statements (Section 7.2).
- Batch multi-row inserts for bulk loads; use `COPY`/`LOAD DATA INFILE` for very large imports.
- **Always** verify `WHERE` clauses with a `SELECT` before running `UPDATE`/`DELETE` in any environment where mistakes are costly (which is, realistically, every production environment).
- Prefer native UPSERT mechanisms over hand-rolled check-then-act application logic to avoid race conditions.
- Use `RETURNING` (PostgreSQL) wherever available to avoid unnecessary follow-up queries.

## 7.14 Common Mistakes

- Running `UPDATE`/`DELETE` without a `WHERE` clause (Sections 7.7–7.8) — arguably *the* most infamous SQL mistake in the industry.
- Relying on positional `INSERT` without a column list, breaking silently after a schema change (Section 7.2).
- Implementing UPSERT manually via `SELECT`-then-branch application logic, introducing race conditions under concurrency (Section 7.5).
- Assuming `ON DUPLICATE KEY UPDATE` and `ON CONFLICT` syntax are interchangeable copy-paste between MySQL and PostgreSQL (Section 7.5).

## 7.15 Edge Cases

- `ON CONFLICT DO UPDATE` in PostgreSQL still requires the `WHERE` clause of the conflict target (if any) to logically make sense — and you can further filter *which* conflicts trigger the update using `ON CONFLICT (col) DO UPDATE SET ... WHERE condition`, a genuinely powerful, lesser-known refinement.
- `INSERT ... SELECT` where the source and target are the **same table** requires care — depending on the engine and query, this can behave unexpectedly if the `SELECT` portion sees rows being inserted by the same statement (generally, engines snapshot the source read before applying inserts, but this is worth testing explicitly rather than assuming).
- Multi-row `INSERT ... ON CONFLICT` in PostgreSQL: if the *same* conflicting key appears **twice within the same multi-row VALUES list**, PostgreSQL raises an error (`ON CONFLICT DO UPDATE command cannot affect row a second time`) — a genuine, sometimes-surprising limitation worth knowing.

## 7.16 Performance Implications

- Multi-row `INSERT` vastly outperforms per-row `INSERT` loops (Section 7.3) — one of the highest-leverage, easy performance wins in everyday application code.
- `UPDATE`/`DELETE` on large, unfiltered result sets can hold locks for extended periods, blocking other transactions — batching large `UPDATE`/`DELETE` operations into smaller chunks (e.g., using `LIMIT` in a loop) is a common real-world technique to avoid long lock waits on production tables.
- PostgreSQL's MVCC-driven `UPDATE` behavior (Section 7.7) means heavy update workloads generate significant "dead tuple" buildup, requiring regular `VACUUM` (previewed here, covered in Part 4) to reclaim space and maintain performance.

## 7.17 Industry Usage

DML statements are executed continuously, at massive scale, in virtually every production backend system — from a single user signing up (`INSERT`) to a checkout flow atomically decrementing inventory (`UPDATE`) to a scheduled cleanup job purging old records (`DELETE`). UPSERT patterns specifically are extremely common in event-processing/idempotent-write systems (e.g., "record this event, but if we've already seen this event ID, just update its status" — a pattern used constantly in distributed systems to handle at-least-once delivery semantics gracefully).

## 7.18 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Basic `INSERT`/`UPDATE`/`DELETE` syntax |
| Placement | "What happens if you forget the `WHERE` clause in an `UPDATE`?" |
| Mid-level | "How would you implement an atomic upsert, and why not just check-then-insert from application code?" |
| Senior SDE-2/SDE-3 | "How would you safely perform a bulk `UPDATE` on a 100-million-row production table without causing lock contention?" / "Explain why PostgreSQL's `UPDATE` interacts with MVCC and `VACUUM`." |

---

## 7.19 Chapter Summary

- `INSERT` supports single-row, multi-row, and `INSERT ... SELECT` forms — always specify columns explicitly, and batch for performance.
- **UPSERT** (`ON CONFLICT`/`ON DUPLICATE KEY UPDATE`) provides atomic, race-condition-free insert-or-update logic — always prefer it over hand-rolled check-then-act code.
- **`MERGE`** offers the most general reconciliation logic (multi-branch insert/update/delete against a source) but isn't supported in MySQL and is only recent in PostgreSQL.
- **`UPDATE`/`DELETE` without a `WHERE` clause** affects the entire table — always verify with a `SELECT` first.
- `UPDATE ... FROM` (Postgres) vs `UPDATE ... JOIN` (MySQL) is a genuine, common syntax divergence for cross-table updates.
- **`RETURNING`** (PostgreSQL-strong, MySQL-limited) collapses a write + follow-up read into one atomic, efficient round-trip.

---

## 7.20 Revision Notes / Cheat Sheet

```sql
INSERT INTO t (col1, col2) VALUES (v1, v2), (v3, v4);          -- multi-row
INSERT INTO t (col1, col2) SELECT c1, c2 FROM other_t;          -- insert-select

-- UPSERT
INSERT INTO t (k, v) VALUES (1, 'x')
    ON CONFLICT (k) DO UPDATE SET v = EXCLUDED.v;                -- PostgreSQL
INSERT INTO t (k, v) VALUES (1, 'x')
    ON DUPLICATE KEY UPDATE v = VALUES(v);                        -- MySQL

UPDATE t SET col = val WHERE condition;      -- ⚠ NEVER omit WHERE unintentionally
DELETE FROM t WHERE condition;               -- ⚠ NEVER omit WHERE unintentionally

UPDATE a SET x = b.y FROM b WHERE a.id = b.id;         -- PostgreSQL join-update
UPDATE a JOIN b ON a.id = b.id SET a.x = b.y;           -- MySQL join-update

INSERT ... RETURNING col;    -- PostgreSQL: get back inserted/updated/deleted rows
UPDATE ... RETURNING col;
DELETE ... RETURNING col;
```

---

## 7.21 Multiple Choice Questions (MCQs)

1. What is the most significant risk of running `UPDATE employees SET salary = 0;` without a `WHERE` clause?
   a) Syntax error b) **Every row in the table gets updated** c) Nothing happens d) Only the first row updates

2. Which PostgreSQL clause lets you retrieve inserted rows without a follow-up `SELECT`?
   a) `OUTPUT` b) **`RETURNING`** c) `RESULT` d) `FETCH`

3. What does `EXCLUDED` refer to in a PostgreSQL `ON CONFLICT DO UPDATE` clause?
   a) Rows that were skipped b) **The row that was proposed for insertion** c) Rows from a different table d) Deleted rows

4. Which statement is TRUE about `MERGE`?
   a) Supported identically in MySQL and PostgreSQL b) **Not supported in MySQL at all** c) Only works with `SELECT` d) Is a DDL statement

5. Why is `INSERT ... ON CONFLICT` generally safer than application-side "check then insert" logic?
   a) It's shorter to type b) **It avoids race conditions under concurrent execution** c) It uses less disk space d) It doesn't require a unique constraint

---

## 7.22 Exercises

### 🟢 Beginner
1. Insert three rows into a `products` table using a single multi-row `INSERT` statement.
2. Write an `UPDATE` statement that increases the price of all products in a specific category by 10%, first verifying the affected rows with a `SELECT`.
3. Delete all products with zero stock using a properly filtered `DELETE`.

### 🟡 Intermediate
4. Implement an UPSERT: insert a product, then re-run the same insert with a different stock value using `ON CONFLICT`/`ON DUPLICATE KEY UPDATE` so the existing row's stock is incremented rather than erroring.
5. Write an `INSERT ... SELECT` that copies all "discontinued" products into an `archived_products` table.
6. In PostgreSQL, use `RETURNING` to capture the auto-generated `id` of a newly inserted row in the same statement.

### 🔴 Advanced / Debugging
7. Explain, step by step, why the following application-code pattern is unsafe under concurrent requests: `SELECT * FROM inventory WHERE product_id = 5; -- if not found, INSERT; else UPDATE;` Then rewrite it as a single atomic UPSERT statement.
8. A teammate wrote `UPDATE orders o JOIN customers c ON o.customer_id = c.id SET o.status = 'flagged' WHERE c.country = 'XX';` and it failed in PostgreSQL. Explain why, and rewrite it in valid PostgreSQL syntax.

### 🧩 Scenario-Based
9. You're processing a stream of "page view" events that may occasionally be delivered more than once (at-least-once delivery). Design an `INSERT ... ON CONFLICT` statement that safely handles duplicate event IDs without creating duplicate rows or erroring.

### 🏗️ Mini Project
10. Using your mini-schema from earlier modules, write a complete DML script that: bulk-inserts sample data, performs at least one UPSERT, performs one cross-table `UPDATE`, and archives-then-deletes a subset of rows — all with `WHERE`-verified safety and (where using PostgreSQL) `RETURNING` clauses to confirm each step's effect.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [PostgreSQL Docs — INSERT ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)
- [MySQL Docs — INSERT ON DUPLICATE KEY UPDATE](https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html)

---

## ✅ Module 7 Complete

You can now confidently insert, update, delete, and atomically upsert data — including the safety habits (verify `WHERE` first) and performance habits (batch inserts, avoid check-then-act races) that separate careful engineers from those who cause production incidents.

**Next up: Module 8 — Basic Queries**, where we shift fully into `SELECT`: `FROM`, `WHERE`, `DISTINCT`, `ORDER BY`, `LIMIT`/`OFFSET`, and aliases.

---

*Reply "continue" or "next module" when you're ready for Module 8.*
