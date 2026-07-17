# The Complete DBMS Handbook — Part 4

## 📘 MODULE 8: ADVANCED DATABASE OBJECTS

> **Continuing from Module 7 (Set Operations).** This module gives full, standalone treatment to the objects that Modules 3 and 4 kept referencing as "coming up later": Views, Materialized Views, Sequences, Synonyms, and Temporary Tables — the reusable, persistent building blocks that complement the query-level constructs you've learned so far.

---

## 8.0 Chapter Roadmap

```
Module 8: Advanced Database Objects
│
├── 8.1 Views
├── 8.2 Materialized Views
├── 8.3 Views vs Materialized Views — Full Comparison
├── 8.4 Sequences
├── 8.5 Synonyms
├── 8.6 Temporary Tables
├── 8.7 Global Temporary Tables
├── 8.8 Full Comparison: CTE vs Derived Table vs View vs Materialized View vs Temp Table
├── 8.9 PostgreSQL vs MySQL Notes
├── 8.10 Best Practices
├── 8.11 Common Mistakes & Edge Cases
├── 8.12 Interview Questions & MCQs
├── 8.13 Cheat Sheet
└── 8.14 Exercises
```

---

## 8.1 Views

**Definition:** A **View** is a **named, stored `SELECT` query** saved in the database schema — it looks and behaves like a table when queried, but **stores no data of its own**; every time you query a view, the database **re-executes the underlying query** against the current, live base tables.

```sql
CREATE VIEW HighEarners AS
SELECT emp_id, name, salary, dept_id
FROM Employees
WHERE salary > 60000;
```

```sql
SELECT * FROM HighEarners WHERE dept_id = 20;
-- Internally rewritten (conceptually) to:
SELECT emp_id, name, salary, dept_id
FROM (SELECT emp_id, name, salary, dept_id FROM Employees WHERE salary > 60000) AS HighEarners
WHERE dept_id = 20;
```

### Why Views Exist

| Problem Solved | Explanation |
|---|---|
| **Abstraction / simplification** | Hide complex joins, filters, and business logic behind a simple, reusable name — application code queries `HighEarners`, not a 5-table join. |
| **Security / access control** | Grant users access to a view exposing only certain columns/rows (e.g., a view excluding `salary` for HR-restricted roles), without granting direct table access. |
| **Logical data independence** | If the underlying table structure changes (e.g., a column is renamed, a table is split), the view definition can be updated once, and all dependent queries/reports continue working unchanged. |
| **Reusability** | The same well-tested business logic (e.g., "active customer" definition) is defined once and reused everywhere, rather than duplicated (and potentially inconsistently reimplemented) across many queries. |

### Real-life Analogy
A view is like a **saved search filter** in an email client (e.g., Gmail's "Starred" or a custom filter) — it doesn't create a separate copy of your emails; it's just a **live, named lens** applied to your actual inbox every time you open it. Delete or archive an email, and it instantly disappears from the filtered view too, because the view isn't storing its own copy.

### Are Views Updatable?

Some views are **updatable** — `INSERT`/`UPDATE`/`DELETE` through the view propagate to the underlying base table — but only under specific conditions:

| Condition for Updatability | Explanation |
|---|---|
| View is based on a **single table** (no joins) | Multi-table views generally can't unambiguously map an update back to one base table |
| No `GROUP BY`, aggregate functions, `DISTINCT`, or set operations in the view definition | These transform/collapse rows, making a reverse mapping to individual base rows impossible or ambiguous |
| All `NOT NULL` columns without defaults are included in the view | Otherwise an `INSERT` through the view couldn't satisfy the base table's constraints |

```sql
-- Updatable (simple, single-table, no aggregation):
CREATE VIEW ActiveEmployees AS SELECT * FROM Employees WHERE is_active = true;
UPDATE ActiveEmployees SET salary = 70000 WHERE emp_id = 2;   -- ✅ Works, updates base Employees table

-- NOT updatable (aggregation collapses rows):
CREATE VIEW DeptAverages AS SELECT dept_id, AVG(salary) AS avg_sal FROM Employees GROUP BY dept_id;
UPDATE DeptAverages SET avg_sal = 999 WHERE dept_id = 20;      -- ❌ Error — no unambiguous mapping back
```

### WITH CHECK OPTION

`WITH CHECK OPTION` ensures that rows inserted/updated **through the view** must still **satisfy the view's own `WHERE` condition** — preventing a confusing scenario where a row you just inserted through a filtered view immediately "disappears" from that view.

```sql
CREATE VIEW HighEarners AS
SELECT * FROM Employees WHERE salary > 60000
WITH CHECK OPTION;

-- Without CHECK OPTION: this would silently succeed but the row wouldn't show up in HighEarners
-- With CHECK OPTION: this correctly FAILS with a constraint violation
INSERT INTO HighEarners (emp_id, name, salary) VALUES (10, 'Karan', 50000);
```

#### Advantages
- No storage overhead (beyond the stored query definition itself).
- Always reflects live, up-to-date data (since it re-runs the query every time).
- Strong tool for security/access-control layering and abstraction.

#### Disadvantages
- **Performance cost is paid every single query** — a view wrapping an expensive join/aggregation is exactly as expensive every time it's queried, since nothing is cached (contrast with Materialized Views, Section 8.2).
- Views **stacked on top of other views** ("nested views") can become very hard to reason about performance-wise, since the optimizer must unravel potentially many layers.

#### Interview Importance: ⭐⭐⭐⭐ — Understanding updatability rules and the View vs Materialized View distinction is a very common Senior Backend/Database Engineer interview topic.

---

## 8.2 Materialized Views

**Definition:** A **Materialized View** is like a regular view, but its result is **physically computed and stored on disk** at creation time (or refresh time) — querying it reads the **stored, precomputed data**, not a live re-execution of the underlying query.

```sql
CREATE MATERIALIZED VIEW DeptRevenueSummary AS
SELECT dept_id, SUM(revenue) AS total_revenue
FROM Sales
GROUP BY dept_id;
```

Querying `DeptRevenueSummary` is now as fast as querying any regular table with the same row count — the expensive `GROUP BY`/`SUM` computation happened **once**, at creation/refresh time, not on every query.

### The Staleness Tradeoff

Because the data is a **snapshot**, it goes **stale** as the underlying base tables change — it must be explicitly **refreshed** to pick up new data:

```sql
-- PostgreSQL:
REFRESH MATERIALIZED VIEW DeptRevenueSummary;

-- Concurrently (doesn't block reads during refresh, requires a unique index on the MV):
REFRESH MATERIALIZED VIEW CONCURRENTLY DeptRevenueSummary;
```

```
Timeline:

  t=0: MV created — snapshot taken, data is fresh
  t=1: New Sales rows inserted into base table — MV is now STALE (doesn't reflect new data)
  t=2: Query MV — still returns t=0 snapshot data (fast, but outdated!)
  t=3: REFRESH MATERIALIZED VIEW run — MV recomputed, now fresh again
  t=4: Query MV — now reflects data as of t=3
```

### Why Materialized Views Exist

| Problem Solved | Explanation |
|---|---|
| **Expensive aggregation reuse** | Dashboards/reports that run the same expensive `GROUP BY`/join repeatedly benefit enormously from precomputing it once and reusing the cached result. |
| **OLAP / data warehousing** | Star-schema fact/dimension rollups (pre-aggregated sales-by-region-by-month, etc.) are a textbook materialized view use case. |
| **Reducing load on source systems** | A materialized view can be refreshed on a schedule (e.g., nightly), decoupling expensive analytical queries from live transactional load. |

#### Advantages
- Dramatically faster reads for expensive underlying queries.
- Reduces repeated computational load on the database for frequently-run reports.

#### Disadvantages
- **Data can be stale** between refreshes — not suitable when strict real-time accuracy is required.
- **Consumes physical storage**, proportional to result size (unlike a regular view).
- Refresh itself can be expensive/slow for very large aggregations, and (without `CONCURRENTLY` support) may briefly lock the materialized view during refresh.

#### Best Practices
- Use materialized views for **expensive, infrequently-changing, frequently-queried** aggregate data (classic OLAP/reporting pattern).
- Schedule refreshes appropriately (e.g., nightly batch jobs, or triggered after ETL loads) based on how stale the data is acceptable to be for the business use case.
- Add a **unique index** on a materialized view if you need `REFRESH ... CONCURRENTLY` (PostgreSQL requirement) to avoid blocking readers during refresh.

#### Interview Importance: ⭐⭐⭐⭐⭐ — The View vs Materialized View tradeoff (freshness vs performance) is one of the most common Senior Backend/Database Engineer system-design-adjacent interview questions.

---

## 8.3 Views vs Materialized Views — Full Comparison

| Aspect | View | Materialized View |
|---|---|---|
| Data storage | ❌ None — re-executes query each time | ✅ Physically stored |
| Query performance | Same as running the underlying query directly | Fast — reads precomputed data |
| Data freshness | Always live/current | Can be stale until refreshed |
| Storage cost | Minimal (just the query definition) | Proportional to result size |
| Refresh needed? | No (always live) | Yes, explicitly (`REFRESH ...`) or on a schedule |
| Indexable? | ❌ No (only underlying base tables) | ✅ Yes — can create indexes on the materialized view itself |
| Ideal use case | Abstraction, security, logical simplification of moderately-cheap queries | Expensive aggregations/reports queried frequently, where some staleness is acceptable |

---

## 8.4 Sequences

**Definition:** A **Sequence** is a database object that generates a **sequence of unique numeric values**, typically used for generating primary key values — independent of any specific table.

```sql
CREATE SEQUENCE emp_id_seq
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    NO MAXVALUE
    CACHE 20;

-- Usage:
INSERT INTO Employees (emp_id, name) VALUES (nextval('emp_id_seq'), 'Aditi');
SELECT currval('emp_id_seq');   -- last value generated in this session
```

### Why Sequences Exist

| Problem Solved | Explanation |
|---|---|
| **Guaranteed uniqueness without locking contention** | Multiple concurrent transactions can safely obtain unique sequence values without blocking each other on a single "max ID" row — sequences are specifically designed for high-concurrency ID generation. |
| **Decoupled from any single table** | Unlike an auto-increment column tied to one table, a sequence object can (in principle) be shared across multiple tables or used independently of any table structure. |
| **Custom numbering control** | Fine-grained control over start value, increment step, min/max bounds, and cycling behavior — useful for invoice numbers, order IDs with specific business rules, etc. |

### Sequence vs AUTO_INCREMENT/IDENTITY

| Aspect | Sequence (PostgreSQL/Oracle) | AUTO_INCREMENT (MySQL) / IDENTITY (SQL Server) |
|---|---|---|
| Tied to a specific column/table? | ❌ No — independent object, can be used anywhere | ✅ Yes — bound directly to one table's column |
| Reusable across multiple tables? | ✅ Yes (though unusual in practice) | ❌ No |
| Explicit control (start, increment, cache, cycle) | ✅ Full, fine-grained control | Limited — mostly just start value and increment |
| Gaps possible? | ✅ Yes — a rolled-back transaction, a `CACHE`d-but-unused value, or `NEXTVAL` called without an insert all leave gaps | ✅ Yes — same underlying gap-causing reasons apply |

> ⚠️ **Common Misconception:** Sequences (and `AUTO_INCREMENT`) are **NOT guaranteed to be gap-free**. A rolled-back transaction still "consumes" the sequence value it obtained — it is **not returned**. Never rely on sequence values being perfectly consecutive; if you need a strictly gap-free audit-style numbering, you need additional application-level or table-level locking logic (with a real performance/concurrency cost).

#### Interview Importance: ⭐⭐⭐ — Frequently comes up in "design a unique ID generator" / distributed systems-adjacent interview discussions.

---

## 8.5 Synonyms

**Definition:** A **Synonym** is an **alternative name (alias)** for a database object (table, view, sequence, stored procedure, etc.) — most prominent in **Oracle** and **SQL Server**; PostgreSQL and MySQL have no direct equivalent keyword (workarounds exist, see 8.9).

```sql
-- Oracle:
CREATE SYNONYM Emp FOR HR.Employees;
SELECT * FROM Emp;   -- equivalent to SELECT * FROM HR.Employees;
```

### Why Synonyms Exist

| Problem Solved | Explanation |
|---|---|
| **Location transparency** | Reference `HR.Employees` (possibly in a different schema, or even a different database via a database link) using a short, stable local name — if the underlying object moves, only the synonym definition needs updating, not every query referencing it. |
| **Simplify qualified names** | Avoid repeatedly typing verbose, fully-qualified schema/database-linked names throughout application code. |
| **Public vs Private Synonyms** | A `PUBLIC SYNONYM` is available to all users in the database; a private synonym is scoped to the user/schema that created it — useful for controlled, organization-wide naming standards. |

#### Interview Importance: ⭐⭐ — Mostly relevant for Oracle/SQL Server-focused roles; lower priority for PostgreSQL/MySQL-centric interviews, but good to know conceptually for "why does this exist" theory questions.

---

## 8.6 Temporary Tables

**Definition:** A **Temporary Table** is a table whose data (and, depending on type, sometimes structure) exists only for the **duration of a session or transaction** — automatically dropped afterward, and (typically) **invisible to other sessions**.

```sql
-- PostgreSQL / most engines:
CREATE TEMPORARY TABLE TempHighEarners AS
SELECT emp_id, name, salary FROM Employees WHERE salary > 60000;

SELECT * FROM TempHighEarners;
-- Automatically dropped at the end of the session (or ON COMMIT DROP for a transaction-scoped version)
```

### Why Temporary Tables Exist

| Problem Solved | Explanation |
|---|---|
| **Complex multi-step ETL/batch processing** | Break a large, complex transformation into stages, materializing intermediate results as real (indexable!) tables — something a CTE cannot do (recall Module 4, Section 4.8: CTEs aren't indexable). |
| **Reducing recomputation** | If an expensive intermediate result is needed multiple times within a session/transaction, computing it once into a temp table avoids repeated recomputation (unlike a CTE, whose materialization behavior is engine/version-dependent, Module 4 Section 4.9). |
| **Session isolation** | Different sessions/users can each have their own `TempHighEarners` table simultaneously without colliding — the database manages this isolation automatically. |

### `ON COMMIT` Behavior (Transaction-Scoped Temp Tables)

```sql
CREATE TEMPORARY TABLE TempCalc (
    id INT,
    value NUMERIC
) ON COMMIT DROP;      -- table is dropped automatically when the transaction commits

CREATE TEMPORARY TABLE TempCalc2 (
    id INT,
    value NUMERIC
) ON COMMIT DELETE ROWS;   -- table structure persists for the session, but rows are cleared each commit
```

#### Interview Importance: ⭐⭐⭐ — Common in "how would you structure a complex batch/ETL job" system-design-adjacent questions, and in contrasting with CTEs/derived tables.

---

## 8.7 Global Temporary Tables

**Definition:** A **Global Temporary Table (GTT)** has a **permanent, shared table DEFINITION** visible to all sessions, but each session sees **only its own private data** within that shared structure — the definition is global/persistent; the data is session-local and temporary.

```sql
-- Oracle syntax:
CREATE GLOBAL TEMPORARY TABLE GTT_Calc (
    id INT,
    value NUMERIC
) ON COMMIT PRESERVE ROWS;   -- or ON COMMIT DELETE ROWS

-- SQL Server uses ## prefix for global temp tables (different concept — see note below):
CREATE TABLE ##GlobalTempCalc (id INT, value NUMERIC);
```

> ⚠️ **Naming Confusion Alert:** SQL Server's "global temporary table" (`##table_name`) is **conceptually different** from Oracle's Global Temporary Table — SQL Server's `##table` is actually **shared data visible to ALL sessions** (dropped when the creating session ends AND no other session references it), whereas Oracle's GTT is a **shared definition with private, per-session data**. **This exact terminology overlap-but-different-meaning is a genuine, frequently-confused interview trivia point** — always clarify which engine's semantics are being discussed.

| Engine | "Global Temp Table" Meaning |
|---|---|
| Oracle | Shared/persistent **definition**; **private, per-session data** |
| SQL Server (`##table`) | Shared **definition AND data**, visible to all sessions |
| PostgreSQL | No direct native equivalent — regular `TEMPORARY` tables are already session-private; achieving Oracle-style "shared definition" typically requires `UNLOGGED` tables or extensions |
| MySQL | No direct native equivalent to either concept |

### Why Global Temporary Tables Exist (Oracle-style)
Avoids the overhead of **repeatedly creating and dropping** the same temp table structure in every session (as a plain session-scoped `CREATE TEMPORARY TABLE` would require) — the structure is defined **once**, permanently, and reused efficiently across many sessions, each with isolated data.

#### Interview Importance: ⭐⭐ — More Oracle-specific trivia; useful to know the terminology trap exists, lower priority for PostgreSQL/MySQL roles.

---

## 8.8 Full Comparison: CTE vs Derived Table vs View vs Materialized View vs Temp Table

| Feature | CTE (Mod. 4) | Derived Table (Mod. 3) | View | Materialized View | Temp Table |
|---|---|---|---|---|---|
| Scope | One statement | One statement | Persistent, cross-query | Persistent, cross-query | Session/transaction |
| Stores data physically? | Depends (optimizer) | Depends (optimizer) | ❌ Never | ✅ Always | ✅ Always |
| Reusable across separate queries? | ❌ No | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes (within session) |
| Supports recursion? | ✅ Yes | ❌ No | ❌ No | ❌ No | N/A |
| Indexable? | ❌ No | ❌ No | ❌ No (only base tables) | ✅ Yes | ✅ Yes |
| Always up-to-date? | ✅ Yes (computed fresh) | ✅ Yes | ✅ Yes | ❌ No (until refreshed) | ✅ Yes (as of creation time) |
| Best for | Readable multi-step logic, recursion | Quick one-off inline computation | Reusable abstraction, security layering | Expensive, frequently-read aggregates | Multi-step batch/ETL work needing indexes |

---

## 8.9 PostgreSQL vs MySQL Notes

| Feature | PostgreSQL | MySQL |
|---|---|---|
| Views | ✅ Full support, including `WITH CHECK OPTION` | ✅ Full support, including `WITH CHECK OPTION` |
| Materialized Views | ✅ Native support (`CREATE MATERIALIZED VIEW`, `REFRESH ... [CONCURRENTLY]`) | ❌ **Not natively supported** — must simulate using a regular table + scheduled `INSERT ... SELECT` / `TRUNCATE` + reload (often via an `EVENT` scheduler or external cron job) |
| Sequences | ✅ Native, full-featured `CREATE SEQUENCE` | ❌ No native standalone sequence object — uses `AUTO_INCREMENT` column attribute instead |
| Synonyms | ❌ No native support | ❌ No native support (views are commonly used as a workaround for simple aliasing) |
| Temporary Tables | ✅ `CREATE TEMPORARY TABLE`, with `ON COMMIT` options | ✅ `CREATE TEMPORARY TABLE` (session-scoped; no `ON COMMIT` clause — MySQL temp tables are always session-scoped, not transaction-scoped) |
| Global Temporary Tables (Oracle-style) | ❌ No direct native equivalent | ❌ No direct native equivalent |

> 🎯 **The Materialized View gap in MySQL is a very practically significant, frequently-tested cross-platform difference** — a common interview follow-up is: *"How would you implement materialized-view-like behavior in MySQL?"* Answer: a real table populated via a scheduled `INSERT ... SELECT` (often wrapped in a stored procedure triggered by MySQL's `EVENT` scheduler), effectively hand-rolling the "refresh" mechanism that PostgreSQL provides natively.

---

## 8.10 Best Practices

- Use **Views** for abstraction, security, and simplifying frequently-repeated moderate-cost query logic.
- Use **Materialized Views** (or their manual MySQL equivalent) for expensive, read-heavy aggregate reports where some staleness is acceptable — always define and document a clear refresh cadence.
- Use **Sequences** for high-concurrency unique ID generation; never assume gap-free numbering.
- Use **Temporary Tables** (not CTEs) when an intermediate result is large, reused multiple times within a session, and would benefit from **indexing** — CTEs cannot be indexed (Module 4, Section 4.8).
- Document staleness/refresh expectations for materialized views clearly for downstream consumers (dashboards, reports) to avoid "why is this data wrong" confusion.

---

## 8.11 Common Mistakes & Edge Cases

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Assuming a view caches/speeds up an expensive query | Views re-execute their underlying query every time — no caching at all | Use a Materialized View if performance/caching is the goal |
| Assuming a Materialized View is always fresh | It's a snapshot, stale until explicitly refreshed | Set up an appropriate refresh schedule and document it |
| Relying on sequence values being perfectly gap-free | Rollbacks and caching cause legitimate gaps | Never build business logic (e.g., invoice numbering compliance) assuming no gaps, without additional safeguards |
| Trying to `UPDATE` through a multi-table or aggregated view | Not updatable — ambiguous mapping back to base tables | Update the base table(s) directly, or use `INSTEAD OF` triggers (Module 10) for controlled updatable-view behavior |
| Confusing Oracle's and SQL Server's "Global Temporary Table" semantics | They mean different things (private-data-shared-definition vs shared-data) | Always clarify which engine's specific semantics apply |
| Using a CTE when a real indexed Temp Table would perform far better | CTEs cannot be indexed and materialization behavior is inconsistent | Use a Temp Table for large, reused, performance-critical intermediate data |

---

## 8.12 Interview Questions & MCQs

### Conceptual
1. What is the fundamental difference between a View and a Materialized View?
2. Why are aggregated/multi-table views generally not updatable?
3. What does `WITH CHECK OPTION` do, and what problem does it prevent?
4. Why are sequence values not guaranteed to be gap-free?
5. What is the difference between a session-scoped Temporary Table and Oracle's Global Temporary Table?

### Applied
6. Design a Materialized-View-based solution for a dashboard showing daily revenue by region, including your chosen refresh strategy and its tradeoffs.
7. Since MySQL lacks native Materialized Views, describe how you would implement equivalent functionality.
8. Write a `CREATE VIEW ... WITH CHECK OPTION` example, and demonstrate (in words) what INSERT would fail and why.
9. Explain when you would choose a Temporary Table over a CTE for a multi-step batch data transformation, with a concrete example involving reuse and indexing needs.

### MCQs

**Q1.** Which object physically stores query results on disk?
 a) View  b) Materialized View  c) CTE  d) Derived Table

**Q2.** Which of these is generally NOT updatable?
 a) A single-table view with a simple WHERE filter  b) A view with GROUP BY and aggregate functions  c) A base table directly  d) An updatable view with WITH CHECK OPTION

**Q3.** MySQL's native equivalent to a standalone Sequence object is:
 a) SERIAL  b) AUTO_INCREMENT  c) IDENTITY  d) ROWID

**Q4.** Which database does NOT natively support Materialized Views?
 a) PostgreSQL  b) Oracle  c) MySQL  d) SQL Server (with indexed views as an analog)

**Q5.** A CTE differs from a Temporary Table primarily because:
 a) CTEs are always faster  b) CTEs cannot be indexed and are scoped to a single statement  c) Temp Tables cannot store data  d) CTEs persist across sessions

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) Materialized View**
2. **b) A view with GROUP BY and aggregate functions**
3. **b) AUTO_INCREMENT**
4. **c) MySQL**
5. **b) CTEs cannot be indexed and are scoped to a single statement**

</details>

---

## 8.13 Cheat Sheet — Module 8

```
┌───────────────────────────────────────────────────────────────────┐
│               ADVANCED DATABASE OBJECTS — QUICK REFERENCE              │
├───────────────────────────┬───────────────────────────────────────┤
│ View                        │ Stored query, no data, always live      │
│ Materialized View             │ Stored query RESULT, can go stale,      │
│                                 │ must be REFRESHed, indexable            │
│ Sequence                        │ Standalone unique-number generator;     │
│                                   │ gaps are normal/expected                │
│ Synonym                            │ Alias for another object (Oracle/SQLServer)│
│ Temporary Table                     │ Session/transaction-scoped, real,     │
│                                       │ indexable table                       │
│ Global Temp Table (Oracle)             │ Shared DEFINITION, private per-session │
│                                          │ DATA                                  │
│ Global Temp Table (SQL Server ##)         │ Shared definition AND shared data     │
│ MySQL Materialized View equivalent          │ Manual: real table + scheduled       │
│                                               │ INSERT...SELECT refresh via EVENT    │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 8.14 Exercises

### Beginner
1. Create a simple, single-table View filtering active employees, and demonstrate that querying it always reflects live data.
2. Create a Sequence and use it to generate 3 unique IDs; explain why a rollback could cause a gap.

### Intermediate
3. Create a Materialized View summarizing total sales per region, then write the `REFRESH` statement, and explain the staleness window between data changes and refresh.
4. Write an example of a non-updatable view (using aggregation) and explain precisely why an `UPDATE` through it fails.

### Advanced
5. Design a MySQL-based "manual materialized view" solution (a real table + a scheduled refresh mechanism) equivalent to a PostgreSQL materialized view, and outline the refresh logic in pseudocode/SQL.
6. Compare, with a concrete scenario, when you'd choose a Temporary Table vs a CTE vs a Materialized View for a complex multi-stage reporting pipeline.

### Output Prediction
7. Given a view with `WITH CHECK OPTION` filtering `salary > 60000`, predict what happens when you attempt to `UPDATE` a row through the view to set `salary = 50000`.

### Debugging
8. A dashboard built on a Materialized View shows outdated numbers after a large batch of new orders was inserted. Diagnose and propose a fix (including discussion of refresh strategy options).

### Scenario-Based
9. You're designing a reporting system where a very expensive daily aggregation is queried hundreds of times per hour by different dashboards. Justify, in interview-answer style, why a Materialized View (or its MySQL equivalent) is the right architectural choice over a plain View or a CTE recomputed on every dashboard load.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)

---

## Chapter Summary

- A **View** is a stored, live-executing query with no data storage of its own — great for abstraction and security, but pays the underlying query's full cost every time.
- A **Materialized View** physically stores its result, trading **freshness** for **read performance** — must be explicitly refreshed, and (in PostgreSQL) can even be indexed.
- **Sequences** generate unique, high-concurrency-safe numeric values — gaps are normal and expected, never assume gap-free numbering.
- **Synonyms** (Oracle/SQL Server) provide location-transparent aliasing for other database objects.
- **Temporary Tables** are real, indexable, session/transaction-scoped tables — the right tool when a CTE's lack of indexing or inconsistent materialization becomes a performance problem.
- **Global Temporary Tables** mean genuinely different things in Oracle (shared definition, private data) vs SQL Server (`##table`, shared definition AND data) — a notable interview trivia trap.
- **MySQL lacks native Materialized Views** — the standard workaround is a real table refreshed on a schedule.

---

## Cross-References for Later Parts / Modules
- **Stored Procedures/Functions for encapsulating the refresh logic of a manual materialized view** → Module 9 (next)
- **Triggers for `INSTEAD OF` updatable-view patterns** → Module 10
- **Indexing strategy for Materialized Views and Temporary Tables** → Part 8 (Query Execution Internals)
- **Access control / permissions design using Views** → later Parts (Security & Administration)

---

> ✅ **Module 8 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 9"** to proceed to **Module 9: Stored Procedures & Functions** — Procedures, User-Defined Functions, parameters, return types, variables, control flow, error handling, performance, and best practices.
