# The Complete DBMS Handbook — Part 4

## 📘 MODULE 12: ADVANCED SQL BEST PRACTICES

> **Continuing from Module 11 (SQL Execution Internals).** This module is deliberately synthesis-focused — it consolidates the best-practice callouts scattered across Modules 1–11 into one coherent, actionable engineering discipline: how a senior engineer actually writes, reviews, and reasons about production SQL.

---

## 12.0 Chapter Roadmap

```
Module 12: Advanced SQL Best Practices
│
├── 12.1 Writing Readable SQL
├── 12.2 Query Decomposition
├── 12.3 Reusing CTEs Effectively
├── 12.4 Avoiding Unnecessary Subqueries
├── 12.5 Choosing Joins Wisely
├── 12.6 NULL Handling — The Complete Discipline
├── 12.7 Defensive SQL
├── 12.8 Interview Questions & MCQs
├── 12.9 Cheat Sheet
└── 12.10 Exercises
```

---

## 12.1 Writing Readable SQL

**Why it matters:** SQL is read far more often than it is written — production queries are debugged, extended, and reviewed by engineers who didn't write the original version, often months or years later. Readability is not cosmetic; it's a direct driver of correctness and maintenance velocity.

### Concrete Formatting Discipline

```sql
-- Poor readability:
select e.name,d.dept_name,e.salary from employees e join departments d on e.dept_id=d.dept_id where e.salary>50000 order by e.salary desc;

-- Good readability:
SELECT
    e.name,
    d.dept_name,
    e.salary
FROM Employees e
JOIN Departments d
    ON e.dept_id = d.dept_id
WHERE e.salary > 50000
ORDER BY e.salary DESC;
```

### Best Practices
- **One clause per line**, keywords capitalized, consistent indentation — matches the logical structure (Module 1, Section 1.11) rather than a single unbroken line.
- **Always alias multi-table queries**, and **always qualify column names** (`e.name`, not just `name`) once more than one table is involved — prevents ambiguity errors and silent misattribution if both tables happen to share a column name.
- **Name CTEs and subqueries descriptively** (Module 4, Section 4.10) — `ActiveCustomersLast30Days`, never `cte1`, `t1`, `x`.
- **Avoid `SELECT *` in production code** (Module 1, Section 1.5.2) — explicit column lists are self-documenting, more index-friendly, and resilient to upstream schema changes (an added column doesn't silently break downstream assumptions).
- **Comment non-obvious business logic**, not obvious mechanics — a comment explaining *why* a `WHERE status != 'CANCELLED'` filter exists is valuable; a comment saying `-- select the name column` is noise.

---

## 12.2 Query Decomposition

**Definition:** Breaking a large, complex query into a sequence of clearly-named, single-purpose logical steps — using CTEs (Module 4) as the primary tool — rather than writing one enormous, deeply-nested statement.

```sql
-- Hard to read/maintain: everything nested inline
SELECT dept_name, avg_sal FROM (
    SELECT d.dept_name, AVG(e.salary) AS avg_sal
    FROM Employees e
    JOIN Departments d ON e.dept_id = d.dept_id
    WHERE e.emp_id IN (
        SELECT emp_id FROM Employees WHERE hire_date > '2020-01-01'
    )
    GROUP BY d.dept_name
) x WHERE avg_sal > 60000;

-- Decomposed: each stage is named and independently understandable
WITH RecentHires AS (
    SELECT emp_id FROM Employees WHERE hire_date > '2020-01-01'
),
DeptAverages AS (
    SELECT d.dept_name, AVG(e.salary) AS avg_sal
    FROM Employees e
    JOIN Departments d ON e.dept_id = d.dept_id
    WHERE e.emp_id IN (SELECT emp_id FROM RecentHires)
    GROUP BY d.dept_name
)
SELECT dept_name, avg_sal
FROM DeptAverages
WHERE avg_sal > 60000;
```

### Why This Matters (Beyond Aesthetics)
- **Debuggability**: you can run `SELECT * FROM RecentHires;` in isolation to verify an intermediate stage is correct, something impossible with a single monolithic nested query.
- **Testability**: each named stage is a natural unit for a teammate (or yourself, months later) to reason about independently.
- **Matches the RA Assignment Operator mental model** (Module 1, Section 1.10) directly — decomposition is *literally* using SQL's implementation of `X ← expr` chaining.

---

## 12.3 Reusing CTEs Effectively

Recall Module 4, Section 4.11: whether a CTE referenced multiple times is **materialized once** or **recomputed on each reference** is engine/version-dependent. This has direct best-practice implications:

- **For cheap CTEs** (simple filters), don't worry about reference count — inlining/recomputation is cheap either way.
- **For expensive CTEs** (heavy aggregation, multiple joins) referenced more than once, **explicitly verify materialization behavior** with `EXPLAIN` — in PostgreSQL 12+, use `AS MATERIALIZED` explicitly if you need to **guarantee** single computation and reuse (Module 4, Section 4.9).
```sql
WITH ExpensiveAgg AS MATERIALIZED (
    SELECT dept_id, complex_multi_join_aggregate(...) AS metric
    FROM ...
)
SELECT * FROM ExpensiveAgg WHERE metric > 100
UNION ALL
SELECT * FROM ExpensiveAgg WHERE metric < 10;
-- Without MATERIALIZED, PostgreSQL 12+ might compute ExpensiveAgg TWICE here
```
- **Don't over-fragment**: excessive CTE chaining for genuinely trivial, tightly-coupled logic can *hurt* readability just as much as one giant nested query — use judgment (Module 4, Section 4.10).

---

## 12.4 Avoiding Unnecessary Subqueries

Recall Module 3's central lesson: many subquery patterns have a **more efficient, more idiomatic equivalent**:

| Instead of... | Prefer... | Why (Module Reference) |
|---|---|---|
| Correlated scalar subquery in `SELECT` for a per-group aggregate | Window function (`AVG(...) OVER (PARTITION BY ...)`) | Module 5 — avoids conceptual per-row re-execution |
| `WHERE col NOT IN (subquery that might contain NULL)` | `WHERE NOT EXISTS (correlated subquery)` | Module 1 §1.8.6, Module 3 §3.3 — NULL-safety |
| Nested correlated subquery for "top-N per group" | Window function (`ROW_NUMBER() OVER (PARTITION BY ...)`) or `LATERAL` | Module 3 §3.6, Module 5 |
| Multiple, deeply-nested nested subqueries for multi-stage logic | Named, chained CTEs | Module 4 |
| A subquery that just checks row existence and selects real columns (`SELECT *`) | `SELECT 1` inside `EXISTS` | Module 3 §3.3 — signals "existence only" intent clearly |

> **The unifying principle:** every SQL feature you've learned across Modules 1–11 exists because it solves a specific problem more clearly, more safely, or more efficiently than an older/more general alternative. Advanced SQL best practice is fundamentally about **recognizing which tool matches which problem** — not about using the "fanciest" construct available.

---

## 12.5 Choosing Joins Wisely

- **Prefer explicit `JOIN ... ON`** over comma-joins (`FROM A, B WHERE ...`) — the join condition is visually inseparable from the tables it relates, dramatically reducing the risk of an accidental Cartesian Product (Module 1, Section 1.6.4).
- **Avoid `NATURAL JOIN` in production code** (Module 1, Section 1.8.3) — implicit column-name matching is fragile against future schema changes (a newly-added, identically-named but unrelated column silently joins on).
- **Choose the narrowest join type that expresses your actual intent**: if you only need "does at least one match exist" (not the matched row's data), use `EXISTS`/Semi-Join logic (Module 1 §1.8.5, Module 3), not a full `JOIN` followed by `DISTINCT`.
- **Be deliberate about `INNER` vs `LEFT`/`RIGHT`/`FULL OUTER`** — an accidental `INNER JOIN` where a `LEFT JOIN` was intended silently **drops rows with no match**, a very common, hard-to-notice class of bug (e.g., "customers with zero orders disappeared from my report").
- **Watch for join fan-out**: joining a "one" table to a "many" table and then aggregating without being careful can silently inflate/duplicate aggregate values (e.g., joining `Orders` to `OrderItems` before summing `Orders.total` — the total gets counted once per item row). Aggregate at the correct grain, or aggregate the "many" side into a derived table/CTE *before* joining.

```sql
-- ⚠️ Fan-out bug: SUM(o.total) is inflated because each order is repeated once per item
SELECT c.customer_id, SUM(o.total) AS total_spent
FROM Customers c
JOIN Orders o ON c.customer_id = o.customer_id
JOIN OrderItems oi ON o.order_id = oi.order_id     -- ◀ fans out one order into many rows
GROUP BY c.customer_id;

-- ✅ Fix: aggregate OrderItems separately first, or use SUM(DISTINCT) cautiously,
-- or restructure to avoid the fan-out before aggregating the "one" side's value
SELECT c.customer_id, SUM(DISTINCT o.order_id, o.total) -- (illustrative; exact syntax varies)
...
-- Cleanest fix: pre-aggregate before joining
WITH OrderTotals AS (
    SELECT customer_id, SUM(total) AS total_spent
    FROM Orders
    GROUP BY customer_id
)
SELECT * FROM OrderTotals;
```

---

## 12.6 NULL Handling — The Complete Discipline

This section consolidates every `NULL`-related rule from Modules 1–7 into one reference:

| Context | Rule | Module |
|---|---|---|
| `WHERE col = NULL` | Always `UNKNOWN` (never `TRUE`) — use `IS NULL` instead | Module 1 §1.5.1 |
| `NOT IN (subquery containing NULL)` | Silently returns **zero rows** for the entire outer query | Module 1 §1.8.6 |
| `NOT EXISTS` | NULL-safe — always the preferred anti-join pattern | Module 1 §1.8.6, Module 3 §3.3 |
| `DISTINCT` / `GROUP BY` / `UNION` deduplication | Two `NULL`s ARE treated as equal (special case!) | Module 7 §7.7 |
| `salary > 50000` where `salary IS NULL` | Row is excluded (comparison is `UNKNOWN`, not `TRUE`) | Module 1 §1.5.1 |
| `x > ANY (empty subquery)` | `FALSE` | Module 3 §3.4 |
| `x > ALL (empty subquery)` | `TRUE` (vacuous truth) | Module 3 §3.4 |
| `COUNT(*)` vs `COUNT(column)` | `COUNT(*)` counts all rows; `COUNT(column)` skips `NULL`s in that column | (general SQL fact, referenced throughout) |
| Aggregate functions (`SUM`, `AVG`, etc.) | Silently **ignore** `NULL` values (not treated as 0) | (general SQL fact) |

### Defensive NULL Patterns
```sql
-- COALESCE: return the first non-NULL value in a list
SELECT COALESCE(phone_mobile, phone_home, 'No phone on file') AS contact_number FROM Customers;

-- NULLIF: return NULL if two values are equal (useful for avoiding division-by-zero)
SELECT revenue / NULLIF(unit_count, 0) AS revenue_per_unit FROM Sales;
```

> 🎯 **The `NULLIF(x, 0)` division-by-zero guard is an extremely common, high-value defensive SQL idiom** — worth memorizing verbatim for interviews and production code alike.

---

## 12.7 Defensive SQL

**Definition:** Writing SQL that behaves predictably and safely even under unexpected data conditions (unexpected NULLs, empty result sets, duplicate rows, concurrent modifications) — a discipline of anticipating edge cases rather than only handling the "happy path."

### Core Defensive Techniques

1. **Guard against division by zero**: `x / NULLIF(y, 0)` (Section 12.6).
2. **Guard against empty subqueries changing behavior unexpectedly**: know the `ANY`/`ALL` vacuous-truth rules (Section 12.6, Module 3 §3.4).
3. **Always specify explicit column lists** in `INSERT` statements — never rely on positional ordering matching the table's current column order:
```sql
-- Fragile: breaks silently if the table's column order ever changes
INSERT INTO Employees VALUES (10, 'Karan', 20, 55000);

-- Defensive: explicit, resilient to schema changes
INSERT INTO Employees (emp_id, name, dept_id, salary) VALUES (10, 'Karan', 20, 55000);
```
4. **Use transactions for multi-statement operations with side effects** (previewing Parts 6–7) — never leave related `INSERT`/`UPDATE` operations unprotected by a transaction boundary, exactly as demonstrated in Module 9 and Module 10's audit/business-rule examples.
5. **Validate assumptions about uniqueness** before relying on them — a `JOIN` on a column you assume is unique but isn't will silently fan out (Section 12.5); verify with a `GROUP BY ... HAVING COUNT(*) > 1` check during development.
6. **Be explicit about frame clauses in window functions** (Module 5, Section 5.7) rather than relying on default behavior — the `LAST_VALUE()` default-frame gotcha is a textbook example of an implicit, easy-to-miss assumption causing silent incorrect results.
7. **Prefer `NOT EXISTS` over `NOT IN`** whenever the subquery's NULL-ability isn't 100% guaranteed (Module 1 §1.8.6) — defensive by default, rather than defensive only after being burned once.
8. **Use `LIMIT`/`TOP` cautiously with `ORDER BY` on non-unique sort keys** — without a fully deterministic sort order (add a tie-breaking column), which rows appear in a `LIMIT N` result can be non-deterministic across runs, causing confusing, hard-to-reproduce "different results each time" bugs.

---

## 12.8 Interview Questions & MCQs

### Conceptual
1. Why is `SELECT *` generally discouraged in production SQL, beyond just "it's less clear"?
2. Explain "join fan-out" with an example, and describe how to avoid it.
3. Why is `NOT EXISTS` generally preferred over `NOT IN` as a defensive default?
4. What is the purpose of `NULLIF(x, 0)`, and what real-world bug does it prevent?
5. Why can `LIMIT N` without a fully deterministic `ORDER BY` produce different results across runs?

### Applied
6. Refactor a deeply nested subquery (of your own construction) into a readable, decomposed CTE chain.
7. Identify and fix a join fan-out bug in a query joining `Orders` to `OrderItems` before summing order totals.
8. Write a defensive `INSERT` statement with explicit column names, and explain the risk of omitting them.

### MCQs

**Q1.** Which is the primary risk of relying on positional (column-list-omitted) `INSERT` statements?
 a) Slower performance  b) Breaks silently if the table's column order changes  c) Not supported in most engines  d) Requires more permissions

**Q2.** Join fan-out most commonly occurs when:
 a) Using `LEFT JOIN` instead of `INNER JOIN`  b) Joining a "one" side to a "many" side before aggregating the "one" side's value  c) Using `NATURAL JOIN`  d) Filtering with `WHERE` instead of `HAVING`

**Q3.** `NULLIF(unit_count, 0)` is primarily used to:
 a) Convert NULL to 0  b) Prevent division-by-zero errors by turning 0 into NULL first  c) Filter out NULL rows  d) Compare two columns for equality

**Q4.** Which is the more defensive choice when a subquery's NULL-ability is uncertain?
 a) NOT IN  b) NOT EXISTS  c) Both are equally safe  d) Neither is safe

**Q5.** Why might `SELECT * FROM T ORDER BY dept_id LIMIT 5` return different rows on different runs?
 a) SQL is inherently random  b) `dept_id` is not a fully unique/deterministic sort key, so ties can be ordered arbitrarily  c) `LIMIT` is non-deterministic by design  d) It never can — this is impossible

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) Breaks silently if the table's column order changes** (for Q1); performance/index-friendliness/schema-resilience combined answer for the original SELECT * question — see explanations above.
2. **b) Joining a "one" side to a "many" side before aggregating the "one" side's value**
3. **b) Prevent division-by-zero errors by turning 0 into NULL first**
4. **b) NOT EXISTS**
5. **b) `dept_id` is not a fully unique/deterministic sort key, so ties can be ordered arbitrarily**

</details>

---

## 12.9 Cheat Sheet — Module 12

```
┌───────────────────────────────────────────────────────────────────┐
│              ADVANCED SQL BEST PRACTICES — QUICK REFERENCE             │
├───────────────────────────┬───────────────────────────────────────┤
│ Readability                  │ One clause per line; alias + qualify all │
│                                │ columns in multi-table queries          │
│ Decomposition                   │ Use named, chained CTEs over deep nesting│
│ CTE reuse                         │ Verify materialization w/ EXPLAIN;      │
│                                     │ use AS MATERIALIZED (PG12+) if needed  │
│ Subquery avoidance                   │ Window fns > correlated scalar subq;   │
│                                        │ NOT EXISTS > NOT IN                    │
│ Joins                                    │ Explicit JOIN...ON; avoid NATURAL JOIN;│
│                                            │ watch for fan-out before aggregating   │
│ NULL discipline                             │ IS NULL not =NULL; NOT EXISTS safe;    │
│                                               │ NULLIF(x,0) for div-by-zero guard      │
│ Defensive SQL                                  │ Explicit INSERT columns; transactions │
│                                                  │ around multi-step writes; deterministic│
│                                                   │ ORDER BY before LIMIT                 │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 12.10 Exercises

### Beginner
1. Rewrite a query using `SELECT *` to use an explicit column list, and explain the resilience benefit.
2. Rewrite a comma-join (`FROM A, B WHERE ...`) into an explicit `JOIN ... ON`.

### Intermediate
3. Identify a correlated scalar subquery in a query of your choice and rewrite it as a window function.
4. Write a defensive query using `NULLIF` to safely compute an average order value per unit without risking division by zero.

### Advanced
5. Construct a query with a join fan-out bug (joining Orders to OrderItems before summing order totals), demonstrate the incorrect inflated result, and fix it using a pre-aggregated CTE.
6. Take a deeply nested 3-level subquery of your own design and decompose it into a readable, well-named CTE chain.

### Scenario-Based
7. In a code review, you spot a colleague's query using `NOT IN (SELECT customer_id FROM Orders)` where `customer_id` could be NULL. Explain the bug, its potential silent impact, and the fix — as you would explain it in an actual review comment.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

- **Readable SQL** (formatting, aliasing, explicit column lists, descriptive naming) is a correctness and maintainability tool, not just style.
- **Query decomposition** via named CTEs makes complex logic debuggable and testable stage-by-stage.
- **CTE reuse** requires knowing your engine's materialization behavior (Module 4) to avoid silent recomputation of expensive logic.
- **Prefer the more targeted, purpose-built construct** (window functions over correlated subqueries, `NOT EXISTS` over `NOT IN`, explicit joins over comma-joins/`NATURAL JOIN`) — this is the unifying theme across Modules 1–11.
- **Join fan-out** is a subtle, common bug — always be conscious of grain when joining "one" to "many" before aggregating.
- **NULL handling** discipline (Section 12.6's consolidated table) should be internalized as muscle memory, not looked up each time.
- **Defensive SQL** anticipates edge cases (division by zero, non-deterministic `LIMIT`, schema drift on `INSERT`) rather than only handling the happy path.

---

## Cross-References for Later Parts / Modules
- **Common SQL Pitfalls (a focused catalog of exactly the mistakes this module teaches you to avoid)** → Module 13 (next)
- **Transactions and atomicity for defensive multi-statement operations** → Parts 6–7
- **Full query optimizer behavior underlying "why prefer X over Y" performance claims** → Part 8

---

> ✅ **Module 12 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 13"** to proceed to **Module 13: Common SQL Pitfalls** — duplicate rows, incorrect joins, wrong aggregations, recursive query mistakes, window function misuse, Cartesian explosions, and correlated query inefficiencies, presented as a focused, standalone debugging catalog.
