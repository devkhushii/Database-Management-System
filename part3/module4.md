# Part 3 — The Complete SQL Handbook
## Module 4: Table Operations

> **Goal of this module:** Create your first real tables. Learn every table-level DDL operation — creation, alteration, renaming, truncation, deletion — plus auto-generating primary key values (`SERIAL`/`IDENTITY`/`AUTO_INCREMENT`), computed/generated columns, and temporary tables. This module is the true hands-on foundation the rest of the handbook builds on.

---

### 📍 Where This Module Fits

```
Module 3: Database-level DDL  →  Module 4: Table-level DDL (You are here)  →  Module 5: Data Types (deep dive)
```

> 📝 **Note on sequencing:** This module necessarily *uses* data types (`INT`, `VARCHAR`, etc.) before Module 5 formally covers every type in depth. Treat the types used here as "enough to get moving" — Module 5 will circle back and explain each one exhaustively, including storage size, ranges, and engine-specific behavior.

---

## 4.1 What is a Table?

A **table** (formally, a **relation**, per Part 2's relational model) is the fundamental structure for storing data in a relational database: a named collection of **rows** (records/tuples) organized into **columns** (fields/attributes), where every row has the same set of columns.

```
                    Table: employees
        ┌────┬──────────┬────────────┬────────┐
Column ► │ id │   name   │ department │ salary │  ◄── Column headers (schema)
        ├────┼──────────┼────────────┼────────┤
Row ►    │  1 │ Asha     │ Engineering│  95000 │
Row ►    │  2 │ Ravi     │ Sales      │  62000 │
Row ►    │  3 │ Meera    │ Engineering│  105000│
        └────┴──────────┴────────────┴────────┘
```

> 🧠 **Real-life analogy:** A table is like a well-organized spreadsheet tab — but with the crucial difference that a database *enforces* the "shape" (schema) strictly: you cannot casually type text into a column defined as strictly numeric the way you might accidentally do in Excel.

---

## 4.2 CREATE TABLE

### Basic Syntax

```sql
CREATE TABLE table_name (
    column1 datatype constraints,
    column2 datatype constraints,
    ...
);
```

### Full Example (Both Engines)

```sql
-- PostgreSQL
CREATE TABLE employees (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    department  VARCHAR(50),
    salary      NUMERIC(10, 2) DEFAULT 0,
    hired_on    DATE NOT NULL DEFAULT CURRENT_DATE
);
```

```sql
-- MySQL
CREATE TABLE employees (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    department  VARCHAR(50),
    salary      DECIMAL(10, 2) DEFAULT 0,
    hired_on    DATE NOT NULL DEFAULT (CURRENT_DATE)
);
```

*(We'll fully explain `SERIAL`/`AUTO_INCREMENT` in Section 4.9, and constraints like `PRIMARY KEY`/`NOT NULL`/`DEFAULT` get their own dedicated deep dive in Module 6 — this module introduces them just enough to build working tables.)*

### Internal Working

When `CREATE TABLE` executes:

1. The parser (recall **Module 1, Section 1.9**) validates syntax and checks the table doesn't already exist in the target schema.
2. The engine writes a new entry into its **system catalog** (`pg_class`/`information_schema.tables` in PostgreSQL; the internal data dictionary in MySQL's `information_schema`/`SHOW`).
3. Column definitions (name, type, size, nullability, defaults) are recorded as catalog metadata.
4. The **storage engine** allocates the physical structures needed to eventually hold rows — in PostgreSQL, this means creating a new physical file (a "relation file") on disk *immediately*, even before any row is inserted; in MySQL/InnoDB, a new tablespace file is typically created (or a slot within a shared tablespace, depending on configuration).
5. No data pages are actually populated yet — an empty table still consumes minimal but nonzero storage for its metadata structures.

### IF NOT EXISTS

```sql
CREATE TABLE IF NOT EXISTS employees ( ... );
```

Both **PostgreSQL and MySQL support this** (unlike `CREATE DATABASE`, where we saw in Module 3 that PostgreSQL lacks it) — a useful, frequently tested asymmetry.

### CREATE TABLE ... AS SELECT (CTAS)

A powerful pattern: create a new table populated from the result of a query, in one step.

```sql
CREATE TABLE high_earners AS
SELECT * FROM employees WHERE salary > 100000;
```

This copies both the **structure** (inferred from the query's result columns) and the **data** — but notably, it does **not** automatically copy constraints, indexes, or defaults from the source table. This is a very common real-world gotcha: people assume a CTAS table is a full clone, but it's really just "structure + data of the result set," nothing more.

> ⚠️ **Common mistake:** Using CTAS to "back up" a table before a risky operation, then being surprised the backup table has no primary key, no indexes, and no constraints — because CTAS deliberately only copies what a `SELECT` naturally produces (column names + inferred types), not the full DDL definition.

---

## 4.3 DROP TABLE

```sql
DROP TABLE employees;
DROP TABLE IF EXISTS employees;      -- defensive version, supported by both engines
DROP TABLE IF EXISTS employees CASCADE;   -- PostgreSQL: also drops dependent objects (e.g., views, FKs referencing it)
```

### Internal Working

- All rows, all indexes, and all catalog metadata associated with the table are removed.
- If other tables have **foreign keys** (Module 6) referencing this table, the drop will **fail** unless those dependent constraints are handled first — or, in PostgreSQL, unless you explicitly use `CASCADE` to force-drop dependents too.

```
Attempting DROP TABLE departments; while employees.department_id
references departments.id:

PostgreSQL:  ERROR: cannot drop table departments because other
             objects depend on it (unless CASCADE is specified)

MySQL:       ERROR 1217 (23000): Cannot delete or update a parent row:
             a foreign key constraint fails (similar protective behavior)
```

> ⚠️ **`CASCADE` here is not the same as `ON DELETE CASCADE`** (a row-level foreign key behavior we'll cover in depth in Module 6) — `DROP TABLE ... CASCADE` cascades the **drop of dependent database objects** (views, constraints), a completely different, DDL-level concept. Confusing these two is an extremely common — and dangerous — mistake.

---

## 4.4 ALTER TABLE

`ALTER TABLE` is one of the most heavily used DDL statements in real production systems — schemas evolve constantly as applications grow.

### Adding a Column

```sql
ALTER TABLE employees ADD COLUMN email VARCHAR(255);
```

### Dropping a Column

```sql
ALTER TABLE employees DROP COLUMN email;
```

### Modifying a Column's Type

```sql
-- PostgreSQL
ALTER TABLE employees ALTER COLUMN salary TYPE NUMERIC(12, 2);

-- MySQL
ALTER TABLE employees MODIFY COLUMN salary DECIMAL(12, 2);
```

> ⚠️ **Notable syntax divergence:** PostgreSQL uses `ALTER COLUMN ... TYPE`; MySQL uses `MODIFY COLUMN` (or the more verbose `CHANGE COLUMN old_name new_name type`, which additionally allows renaming in the same statement). This is a genuinely common source of "works in one, fails in the other" copy-paste errors.

### Renaming a Column

```sql
-- PostgreSQL
ALTER TABLE employees RENAME COLUMN department TO dept;

-- MySQL (8.0+)
ALTER TABLE employees RENAME COLUMN department TO dept;
-- Older MySQL syntax (still valid today):
ALTER TABLE employees CHANGE department dept VARCHAR(50);
```

### Adding/Dropping Constraints

```sql
ALTER TABLE employees ADD CONSTRAINT chk_salary CHECK (salary >= 0);
ALTER TABLE employees DROP CONSTRAINT chk_salary;   -- PostgreSQL & MySQL 8.0.16+
```

*(Full constraint syntax and semantics are covered exhaustively in Module 6 — this is just the `ALTER` mechanics.)*

### Setting/Dropping a Default

```sql
ALTER TABLE employees ALTER COLUMN salary SET DEFAULT 30000;   -- PostgreSQL
ALTER TABLE employees ALTER COLUMN salary DROP DEFAULT;        -- PostgreSQL

ALTER TABLE employees ALTER salary SET DEFAULT 30000;          -- MySQL 8.0.13+
```

### Internal Working — Why Some `ALTER TABLE` Operations Are Fast and Others Are Dangerously Slow

This is a **genuine senior-level interview topic**, and one of the most practically important things in this entire module.

| Operation | Typical Behavior |
|---|---|
| Adding a nullable column with no default | **Fast** (metadata-only change) in modern PostgreSQL (11+) and MySQL (8.0+ with instant DDL support) |
| Adding a column with a non-null default (older engine versions) | Historically required **rewriting every row** in the table — extremely slow and lock-heavy on large tables |
| Changing a column's data type | Almost always requires a **full table rewrite** (every row's data must be re-encoded in the new type) and takes a heavy lock |
| Adding an index | Can lock the table for writes unless done **concurrently** (`CREATE INDEX CONCURRENTLY` in PostgreSQL — previewed here, covered fully with indexing in Part 4) |

```
ALTER TABLE Cost Spectrum:

Cheap (metadata-only)                                    Expensive (full table rewrite + lock)
├─────────────────────────────────────────────────────────────────────────►
Add nullable column        Rename column       Change column type      Add NOT NULL to
(modern engines)           (metadata-only)      (rewrites all rows)     existing column with
                                                                          no default (must
                                                                          validate every row)
```

> ⚠️ **Real production incident pattern:** An engineer runs `ALTER TABLE big_table ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active';` on a table with 500 million rows during peak traffic, expecting it to be instant like it was on their small local dev database — but on an older engine version (or under certain conditions even on newer ones, depending on the exact operation), this locks the table for a full rewrite, causing a production outage. This exact scenario is a classic "tell me about a time you caused an incident" interview story format, and understanding *why* it happens (Section above) is exactly what separates senior engineers from junior ones.

### Multiple Alterations in One Statement (PostgreSQL)

```sql
ALTER TABLE employees
    ADD COLUMN email VARCHAR(255),
    ALTER COLUMN salary SET DEFAULT 30000,
    DROP COLUMN dept;
```

PostgreSQL allows chaining multiple sub-operations in a single `ALTER TABLE` statement (comma-separated), applied atomically. MySQL also supports this pattern for many combinations.

---

## 4.5 RENAME TABLE

```sql
-- PostgreSQL
ALTER TABLE employees RENAME TO staff;

-- MySQL — two equivalent options
ALTER TABLE employees RENAME TO staff;
RENAME TABLE employees TO staff;   -- MySQL also has a dedicated RENAME TABLE statement,
                                     -- which additionally supports renaming multiple
                                     -- tables atomically in one statement:
RENAME TABLE a TO temp, b TO a, temp TO b;   -- atomic table swap!
```

> 💡 **MySQL-specific superpower:** MySQL's dedicated `RENAME TABLE` statement supports **atomic multi-table renames**, which is the standard trick used to swap two tables' names safely (e.g., swapping a freshly rebuilt table into production under the original name with zero downtime window). PostgreSQL doesn't have a direct multi-rename statement, though the same effect can be achieved using three sequential renames within a single **transaction** (since PostgreSQL's DDL is transactional, per Module 3, Section 3.1).

---

## 4.6 TRUNCATE TABLE

```sql
TRUNCATE TABLE employees;
```

### What It Does

Removes **all rows** from a table, extremely quickly — but keeps the table structure (columns, constraints, indexes) intact.

### Why `TRUNCATE` is DDL, Not DML (Revisited from Module 1)

Recall **Module 1, Section 1.8** flagged this as a common trick question. Now we can explain *why* in full:

| Aspect | `DELETE FROM employees;` (DML) | `TRUNCATE TABLE employees;` (DDL) |
|---|---|---|
| Mechanism | Removes rows **one at a time**, logged individually | **Deallocates entire data pages** at once — doesn't process row-by-row |
| Speed on large tables | Slow (proportional to row count) | Very fast (roughly constant time, regardless of row count) |
| Triggers fired | Row-level `DELETE` triggers **do** fire | Triggers generally **do NOT** fire (a real, commonly-tested gotcha) |
| `WHERE` clause | Supported (partial deletion possible) | **Not supported** — always removes everything |
| Transaction/rollback | Fully rollback-able like any DML | Rollback-able in PostgreSQL (transactional DDL); **not** rollback-able in MySQL (implicit commit) |
| Auto-increment counter | Left untouched (`DELETE` doesn't reset `SERIAL`/`AUTO_INCREMENT`) | Typically **resets** the auto-increment counter back to its starting value |
| Row count return | Reports exact number of rows deleted | Reports success/failure only, not a row count (it doesn't count rows individually) |

```
DELETE FROM employees WHERE department = 'Sales';
   → Engine scans/finds matching rows → deletes them individually → logs each removal

TRUNCATE TABLE employees;
   → Engine deallocates the table's entire storage extent in one operation
   → No row-by-row scanning needed at all
```

> ⚠️ **Common mistake:** Using `TRUNCATE` inside application logic expecting `ON DELETE CASCADE` foreign key triggers (Module 6) to fire on related tables — they typically **won't**, because `TRUNCATE` bypasses normal row-level delete processing. This has caused real production data-integrity bugs where "cleanup" scripts using `TRUNCATE` left orphaned rows in child tables that a `DELETE` would have properly cascaded.

### RESTART IDENTITY (PostgreSQL)

```sql
TRUNCATE TABLE employees RESTART IDENTITY;   -- explicitly reset the SERIAL counter
TRUNCATE TABLE employees CONTINUE IDENTITY;  -- keep the counter where it was (default)
```

---

## 4.7 COMMENT

Both engines let you attach human-readable documentation directly to database objects — genuinely useful in large production schemas where table/column *names* alone don't convey full business meaning.

### PostgreSQL

```sql
COMMENT ON TABLE employees IS 'Stores all current and former staff records.';
COMMENT ON COLUMN employees.salary IS 'Annual gross salary in INR, before tax deductions.';
```

### MySQL

```sql
CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    salary DECIMAL(10,2) COMMENT 'Annual gross salary in INR'
) COMMENT = 'Stores all current and former staff records.';

-- Or, added after the fact:
ALTER TABLE employees COMMENT = 'Updated description here.';
```

> 💡 **Industry usage:** Comments are frequently surfaced automatically by schema documentation tools (like dbdocs, SchemaSpy, or DBeaver's ER diagram viewer) — writing them consistently is a genuine best practice in real, long-lived production schemas maintained by large teams.

---

## 4.8 Temporary Tables

A **temporary table** is a table that exists only for the duration of a session (or, in some configurations, a transaction) and is automatically dropped afterward — never persisted permanently, and (by default) invisible to other sessions.

```sql
-- PostgreSQL
CREATE TEMPORARY TABLE temp_results (
    id INT,
    computed_value NUMERIC
);
-- or the shorthand:
CREATE TEMP TABLE temp_results (...);

-- MySQL
CREATE TEMPORARY TABLE temp_results (
    id INT,
    computed_value DECIMAL
);
```

### Why They Exist

- Storing **intermediate results** of a complex, multi-step computation without cluttering the permanent schema.
- Session-local **scratch space** for batch processing or ETL-style scripts.
- Avoiding the overhead of writing to fully durable, replicated permanent storage for data you know is disposable.

> 🧠 **Real-life analogy:** A temporary table is like a **scratchpad or whiteboard** in a meeting room — you jot down working notes during the discussion, and once everyone leaves (the session ends), the whiteboard gets wiped automatically. Nobody expects those notes to persist in a filing cabinet (permanent storage).

### Key Behaviors

| Aspect | Behavior |
|---|---|
| Visibility | Only visible to the session that created it (by default) — other concurrent sessions/users cannot see it, even if they query a table with the identical name |
| Lifespan | Automatically dropped at the end of the session (or, if declared `ON COMMIT DROP` in PostgreSQL, at the end of the current transaction) |
| Naming collisions | You *can* create a temp table with the same name as a permanent table — the temp one **shadows** the permanent one for that session, a subtle and occasionally dangerous behavior worth knowing |
| Performance | Generally lighter-weight than permanent tables — in PostgreSQL, temp table changes aren't written to the write-ahead log (WAL) used for crash recovery/replication, making them notably faster for heavy write workloads |

### PostgreSQL's `ON COMMIT` Options

```sql
CREATE TEMP TABLE staging (id INT) ON COMMIT DROP;     -- dropped when the transaction commits
CREATE TEMP TABLE staging (id INT) ON COMMIT DELETE ROWS; -- rows cleared, table structure kept, at each commit
CREATE TEMP TABLE staging (id INT) ON COMMIT PRESERVE ROWS; -- default: survives commits, dropped only at session end
```

> ⚠️ **Common mistake:** Assuming temporary tables are shared across connections/sessions for coordination between two different application processes — they are explicitly **session-scoped**, and using them for cross-session communication will silently fail (each session simply sees its own empty/nonexistent version).

---

## 4.9 Auto-Generating Primary Key Values: Identity Columns, Auto Increment, SERIAL, BIGSERIAL

This is one of the most practically important — and most confusingly *inconsistent-across-engines* — areas of table design. Nearly every real table needs a way to automatically generate unique, incrementing identifier values, and every major engine solved this problem slightly differently over time.

### The Problem This Solves

Without auto-generation, every `INSERT` would require the application to manually compute and supply a guaranteed-unique ID — a coordination nightmare, especially under concurrent inserts from multiple clients.

### MySQL: `AUTO_INCREMENT`

```sql
CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100)
);
```

- `AUTO_INCREMENT` is a **column attribute** — MySQL's storage engine (InnoDB) internally maintains a counter per table and automatically assigns the next value on insert if you omit that column (or explicitly pass `NULL`/`DEFAULT`).
- Only **one** `AUTO_INCREMENT` column is allowed per table, and it must be indexed (typically as the primary key).

### PostgreSQL (Historical/Legacy): `SERIAL` and `BIGSERIAL`

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,       -- effectively INTEGER, auto-incrementing
    name VARCHAR(100)
);

CREATE TABLE big_table (
    id BIGSERIAL PRIMARY KEY,    -- effectively BIGINT, auto-incrementing (for very large tables)
    name VARCHAR(100)
);
```

**What `SERIAL` really is (internal working — a favorite "gotcha" interview question):**

`SERIAL` is not a true data type at all — it's **syntactic sugar**. Writing `id SERIAL` is automatically expanded by PostgreSQL into:

```sql
id INTEGER NOT NULL DEFAULT nextval('employees_id_seq')
```

...plus the automatic creation of a separate **sequence object** (`employees_id_seq`) — a standalone counter object that exists independently of the table.

```
CREATE TABLE employees (id SERIAL PRIMARY KEY, ...);

    is REWRITTEN internally to:

CREATE SEQUENCE employees_id_seq;
CREATE TABLE employees (
    id INTEGER NOT NULL DEFAULT nextval('employees_id_seq') PRIMARY KEY,
    ...
);
ALTER SEQUENCE employees_id_seq OWNED BY employees.id;
```

> 💡 **Why this matters practically:** Because the sequence is a **separate object**, you can inspect/manipulate it directly: `SELECT currval('employees_id_seq');`, or manually reset it: `ALTER SEQUENCE employees_id_seq RESTART WITH 1000;` — a genuinely useful capability MySQL's simpler `AUTO_INCREMENT` doesn't expose in the same explicit, standalone way.

### The Modern Standard Way (SQL:2003+): `GENERATED ... AS IDENTITY`

Both PostgreSQL (10+) and MySQL/MariaDB ecosystems have moved toward supporting the **ANSI SQL-standard `IDENTITY` column** syntax, which `SERIAL` (being PostgreSQL-proprietary, non-standard sugar) never was.

```sql
-- PostgreSQL (10+) — the modern, standards-compliant, recommended approach
CREATE TABLE employees (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100)
);

-- GENERATED BY DEFAULT allows manual override (e.g., for data migrations)
CREATE TABLE employees (
    id INT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name VARCHAR(100)
);
```

| Variant | Behavior |
|---|---|
| `GENERATED ALWAYS AS IDENTITY` | The database **always** generates the value; manually supplying one raises an error unless you explicitly use `OVERRIDING SYSTEM VALUE` |
| `GENERATED BY DEFAULT AS IDENTITY` | The database generates a value **only if you don't supply one** — closer in spirit to `SERIAL`/`AUTO_INCREMENT`'s permissiveness |

> ⚠️ **Modern best practice, frequently tested in current interviews:** `SERIAL`/`BIGSERIAL` are considered **legacy** in PostgreSQL today. The **PostgreSQL project itself recommends `GENERATED ... AS IDENTITY`** for new schemas, since it's the ANSI-standard syntax (portable in spirit to other standards-compliant engines) and avoids some of `SERIAL`'s quirks (like orphaned sequences after certain operations, and permission-model oddities where sequence privileges are separate from table privileges). Expect senior-level interviewers to specifically ask "would you use `SERIAL` or `IDENTITY` in a new PostgreSQL table, and why?"

### Comparison Table — Auto-Generated Keys Across Engines

| Engine | Legacy/Native Syntax | Modern Standard Syntax | Underlying Mechanism |
|---|---|---|---|
| **PostgreSQL** | `SERIAL` / `BIGSERIAL` | `GENERATED ALWAYS/BY DEFAULT AS IDENTITY` | Sequence object (`nextval()`) |
| **MySQL** | `AUTO_INCREMENT` | *(Standard `IDENTITY` syntax not natively supported; `AUTO_INCREMENT` remains the idiomatic approach)* | Per-table internal counter (InnoDB) |
| **SQL Server** | `IDENTITY(seed, increment)` | Also supports `IDENTITY` natively (T-SQL's original design) | Per-table internal counter |
| **Oracle** | Historically manual `SEQUENCE` + trigger | `GENERATED ALWAYS AS IDENTITY` (12c+) | Sequence object |
| **SQLite** | `INTEGER PRIMARY KEY` (auto-behaves as rowid alias) | N/A | Built-in `rowid` mechanism |

### Internal Working — What Actually Happens on Insert

```sql
INSERT INTO employees (name) VALUES ('Priya');
```

**PostgreSQL (SERIAL/IDENTITY):**
1. The column's default expression `nextval('employees_id_seq')` fires.
2. The sequence atomically increments and returns a new value — this operation is specifically designed to be **fast and non-blocking even under heavy concurrent inserts** (sequences deliberately do **not** participate in normal transactional rollback semantics — more on this in the edge cases below).
3. That value is used as the row's `id`.

**MySQL (AUTO_INCREMENT):**
1. InnoDB consults its in-memory auto-increment counter for the table.
2. Depending on the `innodb_autoinc_lock_mode` setting, it either takes a brief table-level lock or uses a lighter-weight, interleaved-safe mechanism to claim the next value.
3. The counter increments, and that value is used as the row's `id`.

### Edge Case — Gaps in Sequences (A Very Commonly Misunderstood Behavior)

> ⚠️ **Critical, frequently-tested concept:** Auto-generated ID sequences are **not guaranteed to be gap-free**. If an `INSERT` fails, or a transaction using `nextval()` is **rolled back**, the consumed sequence value is **NOT reused** — it's permanently "burned." This is intentional: making sequences transactional/rollback-safe would require locking that would badly hurt concurrent insert performance, so the standard tradeoff (in both PostgreSQL sequences and MySQL's `AUTO_INCREMENT`, especially in higher-concurrency lock modes) is: **guaranteed uniqueness and good performance, at the cost of guaranteed contiguity.**

```
Session A: BEGIN; INSERT ... (gets id=101); ROLLBACK;
Session B: INSERT ... (gets id=102);

Result: id=101 is now PERMANENTLY skipped — never assigned to any row.
```

> 🧠 **Real-life analogy:** Think of a **"take a number" ticket dispenser** at a busy government office. If someone rips off ticket #47 and then walks away without ever getting served (a "rollback"), the dispenser doesn't rewind and hand #47 to the next person — it's simply gone. This is exactly why you should **never** assume auto-generated IDs are sequential without gaps, and never build business logic (like "count of customers = MAX(id)") on that false assumption.

---

## 4.10 Generated Columns (Computed Columns)

A **generated column** is a column whose value is **automatically computed** from other columns in the same row, rather than being directly written by `INSERT`/`UPDATE` statements.

```sql
-- PostgreSQL (12+)
CREATE TABLE order_items (
    quantity   INT NOT NULL,
    unit_price NUMERIC(10,2) NOT NULL,
    total_price NUMERIC(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

-- MySQL (5.7+)
CREATE TABLE order_items (
    quantity   INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);
```

### STORED vs VIRTUAL

| Type | Behavior |
|---|---|
| **STORED** | The computed value is **physically written to disk** alongside the row, recalculated automatically whenever the underlying columns change. Supported by both PostgreSQL and MySQL. |
| **VIRTUAL** | The value is **computed on the fly** every time it's read, never stored on disk. **MySQL supports this** (and it's actually MySQL's default if you omit `STORED`/`VIRTUAL`); **PostgreSQL currently only supports `STORED`** generated columns — a real, documented feature gap as of recent PostgreSQL versions. |

> ⚠️ **PostgreSQL vs MySQL — genuine feature asymmetry:** If you need a `VIRTUAL` (computed-on-read, never-stored) generated column, MySQL supports it natively; PostgreSQL does not (as of the versions this handbook targets) — the common PostgreSQL workaround is a regular **view** (Module 14) with the computed expression in the `SELECT` list, since a view is inherently "computed on read."

### Why Generated Columns Exist

- Guarantee **data consistency**: `total_price` can never drift out of sync with `quantity * unit_price`, because the database — not the application — enforces the relationship.
- Enable **indexing computed expressions**: you can build an index directly on a `STORED` generated column, something not straightforwardly possible on an arbitrary ad-hoc expression in older engine versions.
- Reduce duplicated computation logic scattered across multiple application services that all read/write the same table.

### Constraints on Generated Columns

- Cannot reference other generated columns, in most engines (avoids circular computation issues).
- Cannot be directly written to via `INSERT`/`UPDATE` — attempting to do so raises an error.
- Can still be read normally in any `SELECT`.

```sql
INSERT INTO order_items (quantity, unit_price) VALUES (3, 25.00);
-- total_price is automatically computed as 75.00 — you never supply it directly
```

---

## 4.11 Full Worked Example — Complete Table Lifecycle

```sql
-- 1. Create with modern identity + generated column
CREATE TABLE invoices (
    id           INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer     VARCHAR(100) NOT NULL,
    quantity     INT NOT NULL,
    unit_price   NUMERIC(10,2) NOT NULL,
    total        NUMERIC(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

COMMENT ON TABLE invoices IS 'Stores customer invoice line items.';

-- 2. Evolve the schema
ALTER TABLE invoices ADD COLUMN status VARCHAR(20) DEFAULT 'pending';
ALTER TABLE invoices RENAME COLUMN customer TO customer_name;

-- 3. Use a temp table for a one-off analysis
CREATE TEMP TABLE invoice_summary AS
SELECT customer_name, SUM(total) AS lifetime_value
FROM invoices
GROUP BY customer_name;

-- 4. Clean slate for testing (keeping structure)
TRUNCATE TABLE invoices RESTART IDENTITY;

-- 5. Eventually retire the table entirely
DROP TABLE IF EXISTS invoices CASCADE;
```

```
Execution Timeline:
CREATE TABLE ──► COMMENT ──► ALTER (add col) ──► ALTER (rename) ──► CREATE TEMP TABLE ──► TRUNCATE ──► DROP
     │                            │                     │                   │                 │           │
 catalog entry              metadata note        catalog updated     catalog updated    session-scoped   pages
 + storage alloc                                  (no rewrite if      (metadata-only)    scratch table    deallocated,
                                                    modern engine)                        created          catalog removed
```

---

## 4.12 Advantages / Disadvantages Summary

| Feature | Advantage | Disadvantage / Watch-out |
|---|---|---|
| `SERIAL`/`IDENTITY`/`AUTO_INCREMENT` | Automatic, concurrency-safe unique key generation | Gaps are normal and expected (Section 4.9) — never rely on contiguity |
| Generated columns | Guaranteed consistency, indexable, less app-side logic | PostgreSQL lacks `VIRTUAL` mode; can't be directly written |
| Temporary tables | Fast scratch space, auto-cleanup, session isolation | Easy to accidentally shadow a real table name; not for cross-session sharing |
| `TRUNCATE` | Extremely fast bulk-clear | Skips row-level triggers; not filterable with `WHERE` |
| `ALTER TABLE` | Schema evolves without recreating tables | Some operations (type changes, adding `NOT NULL` without defaults) can lock/rewrite huge tables |

## 4.13 Best Practices

- Prefer `GENERATED ... AS IDENTITY` over `SERIAL` in new PostgreSQL schemas (Section 4.9).
- Always test `ALTER TABLE` operations against a realistic data volume copy before running them on large production tables — "works instantly on my 50-row dev table" is not evidence it will be safe on 50 million rows.
- Use `IF EXISTS`/`IF NOT EXISTS` guards for idempotent migration scripts wherever the engine supports them.
- Document non-obvious columns with `COMMENT` in any schema maintained by more than one person.
- Use `STORED` generated columns for derived values that are frequently *read* and rarely *written to indirectly* — they trade a small amount of extra storage for guaranteed consistency and indexability.

## 4.14 Common Mistakes

- Assuming `TRUNCATE` behaves exactly like an unconditional `DELETE` (Section 4.6) — it skips triggers and isn't `WHERE`-filterable.
- Assuming auto-increment IDs are gap-free and using them for things like "total number of rows ever inserted" (Section 4.9's ticket-dispenser analogy).
- Running heavy `ALTER TABLE` operations on large production tables without understanding the locking/rewrite implications (Section 4.4).
- Forgetting that CTAS (`CREATE TABLE ... AS SELECT`) does not copy constraints/indexes from the source table (Section 4.2).

## 4.15 Edge Cases

- Renaming a table that views/foreign keys depend on can break those dependent objects in some engines/configurations — always check dependencies first (previewed here, foreign keys covered fully in Module 6).
- A temporary table can silently "shadow" a real permanent table of the same name for the rest of that session — queries will hit the temp table without any error, which can be deeply confusing to debug.
- In PostgreSQL, `DROP TABLE ... CASCADE` will also drop any views built on top of that table — an easy way to accidentally lose more than intended if you don't first check what depends on the table (`information_schema` or `pg_depend` can help here).

## 4.16 Performance Implications

- `TRUNCATE` vastly outperforms `DELETE` for full-table clears precisely because it skips row-by-row processing (Section 4.6) — a genuinely important optimization choice in ETL/batch pipelines.
- Adding columns with defaults or changing types on very large tables can cause significant production locking — this is one of the most common real-world causes of unplanned downtime, and a large part of why modern schema-migration tools (Liquibase, Flyway, gh-ost for MySQL, pg_repack for Postgres) exist specifically to perform these changes online/incrementally.
- Sequences (`SERIAL`/`IDENTITY`) are deliberately designed to avoid locking bottlenecks under concurrent inserts, at the cost of potential gaps — a textbook space/performance tradeoff worth being able to explain clearly.

## 4.17 Industry Usage

Table-level DDL is executed constantly in real engineering organizations — but almost always through **migration frameworks** rather than manual, ad-hoc commands typed directly into production, precisely because of the locking/safety concerns in Section 4.4. Understanding the raw SQL underneath those tools, however, is exactly what's expected in interviews and what's needed to debug migration failures when they inevitably happen.

## 4.18 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Basic `CREATE TABLE` syntax, data types, primary keys |
| Placement | "What's the difference between `DELETE` and `TRUNCATE`?" (a top-5 most-asked SQL interview question industry-wide) |
| Mid-level | "How does `SERIAL` actually work internally?" / "Why would a column have gaps in its auto-increment values?" |
| Senior SDE-2/SDE-3 | "How would you safely add a `NOT NULL` column to a 500-million-row production table with zero downtime?" / "Explain `STORED` vs `VIRTUAL` generated columns and when you'd use each." |

---

## 4.19 Chapter Summary

- `CREATE TABLE` defines structure; `ALTER TABLE` evolves it — but not all `ALTER TABLE` operations are equally cheap, and understanding which ones require full table rewrites is a genuine senior-engineer skill.
- `TRUNCATE` is DDL, not DML: it's fast, skips triggers, isn't filterable, and (in PostgreSQL) can reset identity counters — fundamentally different from `DELETE`.
- Auto-generated primary keys evolved from MySQL's `AUTO_INCREMENT` and PostgreSQL's `SERIAL`/`BIGSERIAL` (non-standard sugar around a `SEQUENCE`) toward the modern, ANSI-standard `GENERATED ... AS IDENTITY` syntax — and **gaps in these sequences are normal and expected**, not a bug.
- **Generated columns** compute derived values automatically and consistently; PostgreSQL supports only `STORED`, while MySQL supports both `STORED` and `VIRTUAL`.
- **Temporary tables** provide session-scoped scratch space, automatically cleaned up, and can be configured (in PostgreSQL) to drop or clear on transaction commit.

---

## 4.20 Revision Notes / Cheat Sheet

```sql
CREATE TABLE t (id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, ...);  -- modern, standard
CREATE TABLE t (id SERIAL PRIMARY KEY, ...);        -- PostgreSQL legacy
CREATE TABLE t (id INT AUTO_INCREMENT PRIMARY KEY, ...);  -- MySQL

ALTER TABLE t ADD COLUMN c TYPE;
ALTER TABLE t DROP COLUMN c;
ALTER TABLE t RENAME COLUMN old TO new;
ALTER TABLE t RENAME TO new_table_name;

TRUNCATE TABLE t [RESTART IDENTITY];   -- fast, DDL, no WHERE, skips most triggers
DELETE FROM t WHERE ...;               -- slow-ish, DML, filterable, fires triggers

CREATE TEMP TABLE t (...) [ON COMMIT DROP|DELETE ROWS|PRESERVE ROWS];  -- session-scoped

col TYPE GENERATED ALWAYS AS (expr) STORED;   -- computed column (both engines)
col TYPE GENERATED ALWAYS AS (expr) VIRTUAL;  -- MySQL only

COMMENT ON TABLE t IS '...';           -- PostgreSQL
CREATE TABLE t (...) COMMENT='...';    -- MySQL
```

---

## 4.21 Multiple Choice Questions (MCQs)

1. Which statement is **DDL**, not DML, despite deleting all data?
   a) `DELETE FROM t;` b) **`TRUNCATE TABLE t;`** c) `DROP COLUMN c;` d) `UPDATE t SET x=0;`

2. What does PostgreSQL's `SERIAL` actually expand into internally?
   a) A native data type b) **`INTEGER` + `DEFAULT nextval()` + an auto-created sequence** c) A trigger-based counter d) A view

3. Which is TRUE about auto-increment/identity values?
   a) They're always perfectly contiguous b) **Gaps can occur due to rolled-back transactions or failed inserts** c) They reset automatically on every `DELETE` d) Only one row can ever use them

4. Which generated column mode does PostgreSQL currently lack (that MySQL supports)?
   a) `STORED` b) **`VIRTUAL`** c) `GENERATED ALWAYS` d) `DEFAULT`

5. `CREATE TABLE new_t AS SELECT * FROM old_t;` copies:
   a) Structure, data, indexes, and constraints b) **Structure and data only — not constraints/indexes** c) Only structure d) Only data

---

## 4.22 Exercises

### 🟢 Beginner
1. Create a `products` table with `id` (auto-generated primary key), `name`, `price`, and `created_at` (defaulting to the current date), in both PostgreSQL and MySQL.
2. Add a `description` column to your `products` table using `ALTER TABLE`.
3. Rename the `products` table to `items`.

### 🟡 Intermediate
4. Create a `sales` table with `quantity`, `unit_price`, and a `STORED` generated column `total` computed as `quantity * unit_price`. Insert a row and verify `total` computes correctly.
5. Compare `DELETE FROM items;` vs `TRUNCATE TABLE items;` by inserting sample rows, deleting them each way, and observing (a) speed difference conceptually, and (b) what happens to the auto-increment counter afterward.
6. Create a temporary table summarizing data from `items`, confirm it disappears after your session ends (reconnect and query for it).

### 🔴 Advanced / Debugging
7. Explain, using Section 4.9's "ticket dispenser" analogy, why the following assumption is wrong: *"Since my table has 100 rows and uses `SERIAL`, the highest `id` must be 100."*
8. A colleague ran `ALTER TABLE orders ADD COLUMN region VARCHAR(50) NOT NULL DEFAULT 'unknown';` on a 200-million-row production MySQL table and caused a multi-minute outage. Using Section 4.4, explain what likely happened and how you'd approach this change more safely.

### 🧩 Scenario-Based
9. You need to guarantee `total_price` in an `order_items` table can never be out of sync with `quantity * unit_price`, even if a buggy application tries to write a mismatched value directly. Which Module 4 feature solves this at the database level, and why is it more robust than enforcing the rule only in application code?

### 🏗️ Mini Project
10. Design and create (with real `CREATE TABLE` statements) a small 3-table mini-schema of your choice (e.g., `students`, `courses`, `enrollments`), using: an `IDENTITY`/`SERIAL` primary key in each table, at least one `COMMENT`, and at least one generated column somewhere in the schema. You'll extend this schema with constraints in Module 6.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [PostgreSQL Docs — Identity Columns](https://www.postgresql.org/docs/current/sql-createtable.html)
- [MySQL Docs — AUTO_INCREMENT](https://dev.mysql.com/doc/refman/8.0/en/example-auto-increment.html)
- [SQLBolt — Creating Tables](https://sqlbolt.com/)

---

## ✅ Module 4 Complete

You can now create, evolve, document, and safely tear down tables — including modern identity columns, generated columns, and temporary scratch tables — and you understand *why* some schema changes are instant while others can take down production if done carelessly.

**Next up: Module 5 — SQL Data Types**, where every type introduced loosely in this module (`INT`, `VARCHAR`, `NUMERIC`, `TIMESTAMP`, and many more you haven't seen yet — `JSON`, `ENUM`, `UUID`, arrays) gets a full, exhaustive deep dive.

---

*Reply "continue" or "next module" when you're ready for Module 5.*
