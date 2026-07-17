# The Complete DBMS Handbook — Part 4

## 📘 MODULE 3: ADVANCED SQL QUERIES

> **Continuing from Module 2 (Relational Calculus).** This module is where theory meets daily engineering practice — nested/correlated subqueries, `EXISTS`/`ANY`/`ALL`, derived tables, `LATERAL` joins, and an introduction to recursion. Recall from Module 2 that `EXISTS` = ∃ and `NOT EXISTS` = ¬∃ — keep that logical lens on throughout this module.

---

## 3.0 Chapter Roadmap

```
Module 3: Advanced SQL Queries
│
├── 3.1 Nested Queries (Subqueries) — Overview
├── 3.2 Non-Correlated vs Correlated Subqueries
├── 3.3 EXISTS / NOT EXISTS
├── 3.4 ANY / SOME / ALL
├── 3.5 Derived Tables & Inline Views
├── 3.6 LATERAL Joins
├── 3.7 Recursive Queries (Introduction)
├── 3.8 Subquery Placement: WHERE vs SELECT vs FROM vs HAVING
├── 3.9 PostgreSQL vs MySQL Notes
├── 3.10 Common Mistakes & Edge Cases
├── 3.11 Performance Notes
├── 3.12 Interview Questions & MCQs
├── 3.13 Cheat Sheet
└── 3.14 Exercises
```

---

## 3.1 Nested Queries (Subqueries) — Overview

**Definition:** A **nested query** (or **subquery**) is a `SELECT` statement embedded inside another SQL statement — inside a `WHERE`, `HAVING`, `SELECT`, or `FROM` clause. The embedded query is called the **inner query**; the query containing it is the **outer query**.

### Why It Exists

SQL is built on **Tuple Relational Calculus**, which naturally allows nested logical conditions (recall `∃`, `∀` referencing other relations from Module 2). Nested queries give SQL a way to express *"a condition that itself depends on the result of another query"* — something a single flat `SELECT` cannot do alone.

### Real-life Analogy

A nested query is like answering *"Who is taller than the average person in this room?"* — you first have to compute "the average height" (inner query) before you can answer "who is taller than that" (outer query). You can't do both in one single pass without first knowing the intermediate answer.

### Sample Tables

`Employees`

| emp_id | name | dept_id | salary |
|---|---|---|---|
| 1 | Aditi | 10 | 55000 |
| 2 | Rohan | 20 | 72000 |
| 3 | Kavya | 20 | 68000 |
| 4 | Suresh | 30 | 40000 |
| 5 | Meera | 10 | 61000 |

`Departments`

| dept_id | dept_name |
|---|---|
| 10 | Sales |
| 20 | IT |
| 30 | HR |

### Basic Example — Subquery in WHERE

**"Find employees earning more than the company's average salary"**
```sql
SELECT name, salary
FROM Employees
WHERE salary > (SELECT AVG(salary) FROM Employees);
```

**Execution Order (conceptual):**
```
Step 1: Inner query runs FIRST (independently)
        SELECT AVG(salary) FROM Employees  →  returns a single scalar, say 59200

Step 2: Outer query runs using that scalar value
        SELECT name, salary FROM Employees WHERE salary > 59200
```

**Result:**

| name | salary |
|---|---|
| Rohan | 72000 |
| Kavya | 68000 |
| Meera | 61000 |

### Classifying Subqueries

| Classification | Meaning | Example |
|---|---|---|
| **Scalar subquery** | Returns exactly one row, one column | `(SELECT AVG(salary) FROM Employees)` |
| **Row subquery** | Returns one row, multiple columns | `(SELECT MAX(salary), dept_id FROM ... GROUP BY dept_id LIMIT 1)` |
| **Table/multi-row subquery** | Returns multiple rows | `(SELECT dept_id FROM Departments WHERE dept_name = 'IT')` — could be 1+ rows |
| **Non-correlated** | Executes independently of the outer query | The AVG example above |
| **Correlated** | References a column from the outer query; re-evaluated per outer row | Covered in 3.2 |

---

## 3.2 Non-Correlated vs Correlated Subqueries

### Non-Correlated Subquery

**Definition:** The inner query does **not** reference any column from the outer query — it can be run **completely independently**, and its result is computed **exactly once**.

```sql
SELECT name FROM Employees
WHERE dept_id IN (SELECT dept_id FROM Departments WHERE dept_name = 'IT');
```

**Execution model:**
```
Inner query runs ONCE → produces {20}
Outer query runs ONCE, using that fixed set → filters dept_id IN {20}
```

### Correlated Subquery

**Definition:** The inner query **references a column from the outer query**, meaning it **cannot** be evaluated independently — conceptually, it must be **re-evaluated once for every row** processed by the outer query.

**"Find employees who earn more than the average salary of THEIR OWN department"**
```sql
SELECT e1.name, e1.salary, e1.dept_id
FROM Employees e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM Employees e2
    WHERE e2.dept_id = e1.dept_id   -- ◀ references outer row (e1) — CORRELATION
);
```

**Execution Model (Conceptual — Nested Loop Style):**
```
FOR EACH row e1 in Employees:
      compute AVG(salary) for e2 WHERE e2.dept_id = e1.dept_id
      IF e1.salary > that average:
          include e1 in result

┌──────────────────────────────────────────────────────────┐
│ Outer row (e1)      │ Inner subquery re-run with e1.dept_id │
├──────────────────────────────────────────────────────────┤
│ Aditi, dept=10, 55k │ AVG(salary) WHERE dept_id=10 → 58000  │ → 55k < 58k → excluded
│ Rohan, dept=20, 72k │ AVG(salary) WHERE dept_id=20 → 70000  │ → 72k > 70k → included
│ Kavya, dept=20, 68k │ AVG(salary) WHERE dept_id=20 → 70000  │ → 68k < 70k → excluded
│ Suresh,dept=30, 40k │ AVG(salary) WHERE dept_id=30 → 40000  │ → 40k = 40k → excluded
│ Meera, dept=10, 61k │ AVG(salary) WHERE dept_id=10 → 58000  │ → 61k > 58k → included
└──────────────────────────────────────────────────────────┘
```

**Result:**

| name | salary | dept_id |
|---|---|---|
| Rohan | 72000 | 20 |
| Meera | 61000 | 10 |

#### Internal Working & Optimization Note
Conceptually, a correlated subquery looks like an expensive **row-by-row (nested loop)** re-execution. In practice, **modern optimizers (PostgreSQL, MySQL 8+, Oracle, SQL Server) frequently rewrite correlated subqueries into equivalent JOINs or semi-joins internally** (recall Module 1's Semi Join / Anti Join!) to avoid literally re-running the inner query per row. This is why understanding the RA-level equivalence (Module 1, Section 1.8.5–1.8.6) matters: it explains *why* the optimizer can safely do this rewrite without changing the result.

#### Advantages
- Expresses row-dependent logic naturally and readably (e.g., "compare to peers in the same group").
- Maps directly to Tuple Relational Calculus (Module 2) — natural for "exists a related row" logic.

#### Disadvantages
- Naively executed, correlated subqueries are **O(n × m)** — for every outer row, scan/aggregate the inner relation.
- Can be a major performance bottleneck on large tables if the optimizer cannot rewrite it into a join.

#### Best Practices
- Prefer rewriting correlated subqueries as **JOINs** or **window functions** (Module 5) where possible, since they're often more optimizer-friendly and clearer:
```sql
-- Equivalent using window function (introduced fully in Module 5):
SELECT name, salary, dept_id
FROM (
    SELECT name, salary, dept_id,
           AVG(salary) OVER (PARTITION BY dept_id) AS dept_avg
    FROM Employees
) sub
WHERE salary > dept_avg;
```
- Always check the actual execution plan (`EXPLAIN`) rather than assuming a correlated subquery is automatically slow — the optimizer might already have rewritten it efficiently.

#### Common Mistakes
- Assuming a correlated subquery *always* executes literally row-by-row — this is the **conceptual/logical model**, not necessarily the *physical* execution plan.
- Forgetting to correlate the inner query correctly (typo in join condition), producing a non-correlated (accidentally constant) subquery that silently returns wrong results.

#### Interview Importance: ⭐⭐⭐⭐⭐ — Correlated subqueries (department-wise comparisons, "Nth highest salary," "employees earning more than their manager") are among the most frequently asked FAANG/SDE-2 SQL questions.

---

## 3.3 EXISTS / NOT EXISTS

**Definition:** `EXISTS` is a boolean test on a subquery — it returns `TRUE` if the subquery returns **at least one row**, regardless of the row's actual content. `NOT EXISTS` is its negation.

> Recall from Module 2: **`EXISTS` = ∃** (existential quantifier), and **`NOT EXISTS` = ¬∃**.

### Example — EXISTS

**"Find departments that have at least one employee earning above 60000"**
```sql
SELECT d.dept_name
FROM Departments d
WHERE EXISTS (
    SELECT 1 FROM Employees e
    WHERE e.dept_id = d.dept_id AND e.salary > 60000
);
```
**Result:** `IT` (Rohan=72000, Kavya=68000), and Sales via Meera(61000). → `{Sales, IT}`

> **Why `SELECT 1` and not `SELECT *`?** `EXISTS` only checks for **row existence**, not content — the columns selected inside an `EXISTS` subquery are **completely irrelevant** to the result. Using `SELECT 1` is an idiomatic best practice signaling "I only care whether a row exists," and in most engines it makes **zero performance difference** vs `SELECT *` since the optimizer only checks for existence — but `SELECT 1` remains the widely accepted convention for clarity.

### Example — NOT EXISTS (Anti-Join Pattern, recap from Module 1)

**"Find departments with NO employees earning above 60000"**
```sql
SELECT d.dept_name
FROM Departments d
WHERE NOT EXISTS (
    SELECT 1 FROM Employees e
    WHERE e.dept_id = d.dept_id AND e.salary > 60000
);
```
**Result:** `HR` (only Suresh, 40000).

### EXISTS vs IN — Detailed Comparison

| Aspect | `EXISTS` | `IN` |
|---|---|---|
| Works on | Correlated or non-correlated subqueries | Typically non-correlated (a list of values) |
| NULL-safety (positive form) | Safe | Safe (`IN` is fine; only `NOT IN` has the NULL trap) |
| NULL-safety (negative form) | ✅ `NOT EXISTS` is NULL-safe | ❌ `NOT IN` breaks silently if subquery returns any `NULL` (Module 1, Section 1.8.6 — memorize this) |
| Performance on large subqueries | Often better — can short-circuit on first match | Historically could materialize the full list first (modern optimizers usually equalize this) |
| Best for | Correlated existence checks | Simple static/small value lists |

```
⚠️ THE #1 SQL INTERVIEW GOTCHA — worth repeating from Module 1:

SELECT * FROM Customers
WHERE customer_id NOT IN (SELECT customer_id FROM Orders WHERE customer_id IS NULL... );

If the subquery's result set contains even ONE NULL,
`NOT IN` returns ZERO rows for the ENTIRE outer query —
because `x NOT IN (1, 2, NULL)` evaluates to UNKNOWN for every x.

FIX: Always prefer NOT EXISTS for anti-join / "not in" logic,
     or explicitly filter NULLs: NOT IN (SELECT col FROM T WHERE col IS NOT NULL)
```

---

## 3.4 ANY / SOME / ALL

**Definition:** `ANY`, `SOME`, and `ALL` are quantified comparison operators used with a comparison operator (`=, <, >, <=, >=, <>`) against a **multi-row subquery**.

> `ANY` and `SOME` are **exact synonyms** in the SQL standard — `SOME` exists purely for readability/compatibility; they behave identically everywhere.

### Semantics

| Operator | Meaning |
|---|---|
| `x = ANY (subquery)` | TRUE if x equals **at least one** value returned — logically identical to `x IN (subquery)` |
| `x > ANY (subquery)` | TRUE if x is greater than **at least one** (i.e., greater than the **minimum**) value returned |
| `x > ALL (subquery)` | TRUE if x is greater than **every** value returned (i.e., greater than the **maximum**) |
| `x <> ALL (subquery)` | TRUE if x matches **none** of the values — logically identical to `x NOT IN (subquery)` |

### Worked Examples

**"Find employees earning more than at least one IT employee" (`> ANY`)**
```sql
SELECT name, salary FROM Employees
WHERE salary > ANY (SELECT salary FROM Employees WHERE dept_id = 20);
-- IT salaries are {72000, 68000}. > ANY means > MIN(68000).
```
**Equivalent simplification:** `salary > (SELECT MIN(salary) FROM Employees WHERE dept_id = 20)`

**"Find employees earning more than ALL IT employees" (`> ALL`)**
```sql
SELECT name, salary FROM Employees
WHERE salary > ALL (SELECT salary FROM Employees WHERE dept_id = 20);
-- Equivalent to: salary > (SELECT MAX(salary) FROM Employees WHERE dept_id = 20)
```

### ANY/ALL vs MIN/MAX vs EXISTS — Equivalence Table

| Quantified Form | Exact Equivalent |
|---|---|
| `x = ANY (subq)` | `x IN (subq)` |
| `x <> ALL (subq)` | `x NOT IN (subq)` (⚠️ same NULL trap as `NOT IN`) |
| `x > ANY (subq)` | `x > (SELECT MIN(...) FROM subq)` |
| `x > ALL (subq)` | `x > (SELECT MAX(...) FROM subq)` |
| `x < ANY (subq)` | `x < (SELECT MAX(...) FROM subq)` |
| `x < ALL (subq)` | `x < (SELECT MIN(...) FROM subq)` |

> ⚠️ **Common Confusion (top interview trap):** `> ANY` sounds like it should mean "greater than everything" but it actually means **"greater than at least one" (i.e., greater than the minimum)** — the intuitively "weaker" condition. `> ALL` is the "greater than every single one" (i.e., greater than the maximum) — the intuitively "stronger" condition. Many students get this backwards. **Memorize: ANY = easier to satisfy (compare to MIN for `>`), ALL = harder to satisfy (compare to MAX for `>`).**

### Empty Subquery Edge Case
- `x > ANY (empty subquery)` → **FALSE** (there's nothing to be greater than "at least one of").
- `x > ALL (empty subquery)` → **TRUE** (vacuously true — "greater than all zero of them" is trivially satisfied, same logic as `∀` over an empty set being vacuously true).

> This vacuous-truth edge case is directly connected to Module 2's Universal Quantifier (∀) discussion — `∀ x ∈ ∅ (P(x))` is always true because there's no counter-example to violate it.

---

## 3.5 Derived Tables & Inline Views

**Definition:** A **derived table** (also called an **inline view**) is a subquery placed in the `FROM` clause, given a name (alias), and then queried **as if it were a real table**.

```sql
SELECT dept_id, avg_sal
FROM (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM Employees
    GROUP BY dept_id
) AS dept_avg_table         -- ◀ MUST be aliased; this is the "derived table"
WHERE avg_sal > 50000;
```

### Why It Exists
SQL sometimes needs to **filter, join, or aggregate on top of an already-aggregated result** — you cannot put an aggregate function's result directly into `WHERE` (that's what `HAVING` is for, but `HAVING` only filters groups of the *same* query, not a further-processed result). A derived table lets you treat a computed intermediate result as a fresh, queryable "table," enabling multi-stage computation.

### Real-life Analogy
A derived table is like a **rough draft**: you first compute an intermediate summary (e.g., department averages), name that draft, and then write your final query against the draft — rather than trying to do everything in one giant unreadable single query.

### Internal Working
The optimizer treats a derived table as a **subplan**: it may be materialized (computed fully into a temporary result set first) or it may be "inlined"/"unnested" directly into the outer query plan (a technique called **subquery flattening** or **view merging**) if doing so is safe and cheaper. Whether materialization happens is entirely up to the optimizer's cost-based decisions.

### Comparison: Derived Table vs CTE vs View

| Feature | Derived Table | CTE (`WITH`) | View |
|---|---|---|---|
| Defined in | `FROM (...)` inline | `WITH name AS (...)` before main query | Stored in the database schema (persistent) |
| Reusable across queries? | No — scoped to one query | Only within the same query (unless recursive) | Yes — reusable across many queries |
| Must be aliased? | ✅ Yes, mandatory | Optional if given a name in `WITH` | N/A (has a name) |
| Readability | Can get deeply nested/unreadable | Much more readable (named, top-down) | Excellent — abstracts complexity away entirely |
| Persisted in schema? | No | No | Yes |

> This table is a preview bridge into **Module 4 (CTEs)** and **Module 8 (Views/Materialized Views)** — derived tables are the "quick and dirty" ancestor of both.

---

## 3.6 LATERAL Joins

**Definition:** A `LATERAL` join (or a `LATERAL` derived table) is a special kind of derived table that is allowed to **reference columns from tables that appear earlier in the same `FROM` clause** — something an ordinary derived table/subquery in `FROM` **cannot** do.

### The Problem LATERAL Solves

An ordinary subquery in `FROM` is evaluated **once**, independently, with no knowledge of other tables in the same query. But sometimes you need: **"for each row of table A, compute a correlated subquery against table B"** — exactly like a correlated subquery, but one that needs to return **multiple rows and/or multiple columns** (something a scalar correlated subquery in `WHERE`/`SELECT` cannot do).

### Worked Example — "Top 2 highest-paid employees per department"

```sql
SELECT d.dept_name, top_emps.name, top_emps.salary
FROM Departments d
CROSS JOIN LATERAL (
    SELECT e.name, e.salary
    FROM Employees e
    WHERE e.dept_id = d.dept_id      -- ◀ references outer table `d` — only possible with LATERAL
    ORDER BY e.salary DESC
    LIMIT 2
) AS top_emps;
```

**Result:**

| dept_name | name | salary |
|---|---|---|
| Sales | Meera | 61000 |
| Sales | Aditi | 55000 |
| IT | Rohan | 72000 |
| IT | Kavya | 68000 |
| HR | Suresh | 40000 |

**Execution Model:**
```
FOR EACH row d in Departments:
      run the LATERAL subquery WITH d.dept_id bound
      → returns up to 2 rows (top-salary employees in that dept)
      → join those rows with d
```

> 🎯 **Why this needs LATERAL and not a plain correlated subquery in WHERE:** a `WHERE`-clause correlated subquery can only return a **single scalar value** for a comparison. Here, we need **multiple rows AND multiple columns** per outer row ("give me the top 2, with both name and salary") — only `LATERAL` (or a window function alternative, covered in Module 5) supports this "per-group top-N" pattern cleanly.

### Best Practices
- Use `LATERAL` for **"top-N per group"** queries when window functions feel less natural, or when the per-row computation is itself a complex multi-step subquery.
- Prefer `CROSS JOIN LATERAL` (inner-join-like — drops groups with zero matches) vs `LEFT JOIN LATERAL ... ON true` (keeps groups even with zero matches, filling `NULL`s) depending on whether you want unmatched outer rows preserved.

#### Interview Importance: ⭐⭐⭐⭐ — "Top-N per group" is an extremely common interview pattern; showing you know **both** the `LATERAL` approach and the **window function** approach (Module 5) demonstrates strong SQL depth.

---

## 3.7 Recursive Queries (Introduction)

**Definition:** A recursive query is one that **references itself** to process **hierarchical or graph-structured data** (org charts, category trees, bill-of-materials, social graphs) where the depth of nesting is not known in advance.

> **Full deep-dive treatment (syntax, execution model, termination, cycle detection) is in Module 4 (Recursive CTEs)** — this section gives you the conceptual foundation so Module 4 builds on solid footing.

### Why It Exists

Plain SQL, being based on Relational Algebra/Calculus (Modules 1–2), has **no native way to express "repeat this join an unknown number of times."** A fixed SQL query has a fixed number of joins written in its text — but "find all descendants of a manager, at any depth" requires a **variable, data-dependent number of joins**. Recursion solves this by letting a query **repeatedly apply itself** to its own growing result set until no new rows are produced.

### Real-life Analogy

Finding all-org-chart descendants of a CEO is like a **chain letter**: the CEO tells 3 direct reports, each of whom tells their own direct reports, and so on — you don't know in advance how many "rounds" of forwarding will happen; you just keep forwarding until nobody has anyone new to tell.

### Minimal Preview Example

```sql
WITH RECURSIVE OrgChart AS (
    -- Anchor member: the starting point (e.g., the CEO)
    SELECT emp_id, name, manager_id, 1 AS level
    FROM Employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive member: joins the recursive result back to Employees
    SELECT e.emp_id, e.name, e.manager_id, oc.level + 1
    FROM Employees e
    JOIN OrgChart oc ON e.manager_id = oc.emp_id
)
SELECT * FROM OrgChart;
```

We stop here intentionally — **full explanation of the anchor member, recursive member, termination condition, cycle prevention, and hierarchical/graph traversal patterns is Module 4's core content.**

---

## 3.8 Subquery Placement: WHERE vs SELECT vs FROM vs HAVING

| Placement | Purpose | Must return | Example |
|---|---|---|---|
| `WHERE` | Row-level filter using a computed value | Scalar (for `=`,`>`, etc.) or multi-row (for `IN`/`ANY`/`ALL`/`EXISTS`) | `WHERE salary > (SELECT AVG(salary) ...)` |
| `HAVING` | Group-level filter using a computed value | Scalar (typically) | `HAVING COUNT(*) > (SELECT AVG(cnt) FROM ...)` |
| `SELECT` | Compute a per-row scalar value as an output column | Scalar, and must be correlated to the outer row typically | `SELECT name, (SELECT COUNT(*) FROM Orders o WHERE o.cust_id = c.id) AS order_count FROM Customers c` |
| `FROM` | Derived table / inline view / LATERAL | Any shape (rows × columns) | Section 3.5, 3.6 |

**Scalar Subquery in SELECT — Worked Example:**
```sql
SELECT c.name,
       (SELECT COUNT(*) FROM Orders o WHERE o.customer_id = c.customer_id) AS order_count
FROM Customers c;
```
> ⚠️ **Performance Warning:** A scalar subquery in `SELECT` is, in the naive execution model, a **correlated subquery re-run once per outer row** — for large tables this can be far slower than an equivalent `LEFT JOIN ... GROUP BY`. Always compare execution plans; this is a very common "junior vs senior" SQL code-review flag.

**Senior-preferred rewrite using JOIN + GROUP BY:**
```sql
SELECT c.name, COUNT(o.order_id) AS order_count
FROM Customers c
LEFT JOIN Orders o ON o.customer_id = c.customer_id
GROUP BY c.name;
```

---

## 3.9 PostgreSQL vs MySQL Notes

| Feature | PostgreSQL | MySQL |
|---|---|---|
| `LATERAL` joins | ✅ Full support (`CROSS JOIN LATERAL`, `LEFT JOIN LATERAL ... ON true`) | ✅ Supported from MySQL 8.0.14+ |
| Correlated subquery optimization (rewrite to join/semi-join) | ✅ Strong optimizer support | ✅ Improved significantly in MySQL 8.0+ (older 5.x versions had known correlated-subquery performance issues) |
| `ANY`/`SOME`/`ALL` | ✅ Full support | ✅ Full support |
| Derived tables requiring alias | ✅ Mandatory | ✅ Mandatory (`Error: Every derived table must have its own alias`) |
| Recursive CTEs (`WITH RECURSIVE`) | ✅ Supported | ✅ Supported from MySQL 8.0+ (not available in 5.x) |

---

## 3.10 Common Mistakes & Edge Cases

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using `NOT IN` with a subquery that might return `NULL` | Silently returns zero rows | Use `NOT EXISTS`, or filter `WHERE col IS NOT NULL` inside the subquery |
| Forgetting to alias a derived table | Syntax error in most DBs | Always `... ) AS alias_name` |
| Assuming `ANY`/`ALL` compare "in bulk" like an aggregate | `> ANY` = compare to MIN; `> ALL` = compare to MAX — not literal bulk comparison | Memorize the MIN/MAX equivalence table (3.4) |
| Writing a scalar subquery in `SELECT` on a huge table without checking `EXPLAIN` | Can silently be O(n²) | Prefer `JOIN` + `GROUP BY` rewrites; verify with execution plan |
| Using `EXISTS` but selecting real columns inside (`SELECT * ` instead of `SELECT 1`) — not wrong, but non-idiomatic | Slight code-review/readability flag in most teams | Use `SELECT 1` inside `EXISTS` for clarity |
| Assuming empty-subquery `> ALL` is FALSE | Vacuously TRUE (Section 3.4) | Understand vacuous truth for empty sets |

---

## 3.11 Performance Notes

- **Correlated subqueries** can be O(n × m) in the worst case if not rewritten; modern optimizers often auto-rewrite them into semi-joins/joins, but **always verify with `EXPLAIN` / `EXPLAIN ANALYZE`** rather than assuming.
- **Derived tables** may or may not be materialized — the optimizer decides based on cost; a derived table doing significant work should ideally be filtered as early as possible inside itself (predicate pushdown, recall Module 1).
- **LATERAL joins** are typically executed as a nested-loop-style operation (once per outer row) — appropriate for "top-N per group" but can be expensive without good indexing on the join/order columns.
- **`IN` vs `EXISTS`**: in modern optimizers (PostgreSQL, MySQL 8+), performance is usually comparable because both get rewritten into semi-joins internally — but on **large subqueries with many duplicate values**, `EXISTS` sometimes short-circuits faster since it can stop at the first match, while `IN` may need to build/deduplicate the whole candidate list first (implementation-dependent).

---

## 3.12 Interview Questions & MCQs

### Conceptual
1. What is the difference between a correlated and a non-correlated subquery?
2. Why is `NOT IN` dangerous with subqueries that may return `NULL`, while `NOT EXISTS` is safe?
3. Explain the difference between `> ANY` and `> ALL` with an example.
4. What is a LATERAL join, and what problem does it solve that a regular derived table cannot?
5. What is the difference between a derived table, a CTE, and a view?

### Applied (Very High-Frequency FAANG/SDE-2/3 Patterns)
6. Write a query to find the **second-highest salary** in `Employees` using a subquery (without `LIMIT`/`OFFSET`).
7. Write a query to find employees who earn **more than their own manager**, using a self-join or correlated subquery.
8. Write a query using `LATERAL` (or a window function, foreshadowing Module 5) to find the **top 3 highest-paid employees per department**.
9. Write a query to find customers who have placed **zero orders**, using `NOT EXISTS`.
10. Rewrite a correlated scalar subquery in `SELECT` (per-customer order count) into an equivalent `JOIN` + `GROUP BY`, and explain why the rewrite is usually faster.

### MCQs

**Q1.** Which of these is NULL-unsafe?
 a) `NOT EXISTS`  b) `NOT IN`  c) `EXISTS`  d) `IN`

**Q2.** `salary > ANY (SELECT salary FROM T)` is equivalent to:
 a) `salary > MAX(...)`  b) `salary > MIN(...)`  c) `salary = ANY(...)`  d) `salary < ALL(...)`

**Q3.** A derived table must always have:
 a) An index  b) An alias  c) A primary key  d) A `WHERE` clause

**Q4.** `LATERAL` allows a subquery in `FROM` to:
 a) Run faster automatically  b) Reference columns from preceding tables in the same FROM clause  c) Avoid using JOINs entirely  d) Return only scalar values

**Q5.** `x > ALL (empty subquery)` evaluates to:
 a) FALSE  b) TRUE (vacuous truth)  c) NULL  d) Syntax error

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) `NOT IN`**
2. **b) `salary > MIN(...)`**
3. **b) An alias**
4. **b) Reference columns from preceding tables in the same FROM clause**
5. **b) TRUE (vacuous truth)**

</details>

---

## 3.13 Cheat Sheet — Module 3

```
┌───────────────────────────────────────────────────────────────────┐
│                 ADVANCED SQL QUERIES — QUICK REFERENCE               │
├───────────────────────────┬───────────────────────────────────────┤
│ Non-correlated subquery   │ Runs once, independent of outer query   │
│ Correlated subquery       │ References outer row; conceptually re-run │
│                           │ per row (may be optimized to a join)     │
│ EXISTS                    │ TRUE if subquery returns ≥1 row (= ∃)    │
│ NOT EXISTS                │ TRUE if subquery returns 0 rows (= ¬∃)   │
│ IN                        │ = ANY, safe                              │
│ NOT IN                    │ ⚠️ Unsafe with NULLs — prefer NOT EXISTS │
│ x > ANY (subq)            │ x > MIN(subq)                            │
│ x > ALL (subq)            │ x > MAX(subq)                            │
│ Derived table             │ Subquery in FROM, must be aliased        │
│ LATERAL                   │ Derived table that can reference earlier │
│                           │ tables in the same FROM clause           │
│ Recursive query           │ Self-referencing query for hierarchies/  │
│                           │ graphs — full treatment in Module 4      │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 3.14 Exercises

### Beginner
1. Write a query to find employees earning more than the average salary (non-correlated subquery).
2. Write a query using `EXISTS` to find departments with at least one employee.

### Intermediate
3. Write a correlated subquery to find employees earning more than their department's average.
4. Write a query using `ANY` and another using `ALL` to compare an employee's salary against all IT department salaries; explain the difference in results.
5. Rewrite a `NOT IN` anti-join query into a `NOT EXISTS` query and explain the NULL-safety improvement.

### Advanced
6. Write a `LATERAL` join query to fetch the top 2 highest-paid employees in every department.
7. Write a query to find the **Nth highest salary** using a correlated subquery with `COUNT`.
8. Convert a scalar correlated subquery in `SELECT` into an equivalent `LEFT JOIN + GROUP BY` and compare expected performance characteristics.

### Output Prediction
9. Given `salary > ALL (SELECT salary FROM Employees WHERE dept_id = 999)` where dept_id 999 doesn't exist (empty result), what does this evaluate to for every row, and why?

### Debugging
10. A query using `NOT IN (SELECT dept_id FROM Departments)` returns zero rows unexpectedly even though matching rows clearly exist. Diagnose and fix.

### Scenario-Based
11. In an interview, you're asked to find "the top 3 selling products in each category" — describe two different approaches (LATERAL vs window functions) and the tradeoffs.

### Mini Project
12. Using a 3-table e-commerce schema (`Customers`, `Orders`, `OrderItems`), write: (a) a correlated subquery, (b) a non-correlated subquery, (c) a derived table query, (d) a `LATERAL` top-N query, (e) an `EXISTS`/`NOT EXISTS` pair — one query each.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [SQLBolt — Interactive SQL Lessons](https://sqlbolt.com/)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

- **Nested queries** embed one `SELECT` inside another; they can be **non-correlated** (run once) or **correlated** (conceptually re-run per outer row, often optimizer-rewritten into joins).
- **`EXISTS`/`NOT EXISTS`** implement ∃/¬∃ directly and are the **NULL-safe** choice for existence/anti-existence checks — always prefer `NOT EXISTS` over `NOT IN`.
- **`ANY`/`SOME`** are synonyms; `> ANY` compares against the **MIN**, `> ALL` compares against the **MAX** — a very commonly misremembered rule.
- **Derived tables** are aliased subqueries in `FROM`; **LATERAL** extends this by allowing per-outer-row correlated, multi-row subqueries in `FROM` — ideal for "top-N per group."
- **Recursive queries** solve the "unknown depth" problem for hierarchical/graph data — fully covered next in Module 4.
- Always evaluate whether a correlated subquery should be rewritten as a `JOIN`/window function for performance, and always verify via `EXPLAIN`.

---

## Cross-References for Later Parts / Modules
- **Recursive CTEs — full syntax, anchor/recursive members, cycle detection** → Module 4 (next)
- **Window functions as an alternative to LATERAL for "top-N per group"** → Module 5
- **Views / Materialized Views as reusable derived tables** → Module 8
- **Query optimizer's rewriting of correlated subqueries into semi-joins** → Part 8 (Query Execution Internals)

---

> ✅ **Module 3 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 4"** to proceed to **Module 4: Common Table Expressions (CTEs)** — the `WITH` clause, multiple CTEs, Recursive CTEs, hierarchical & graph traversal queries, best practices, and performance considerations.
