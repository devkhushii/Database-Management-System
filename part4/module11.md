# The Complete DBMS Handbook — Part 4

## 📘 MODULE 11: SQL EXECUTION INTERNALS (INTRODUCTION)

> **Continuing from Module 10 (Triggers).** This module is a **high-level overview only** — it exists to give you a mental map of what happens between typing a SQL query and getting a result back, tying together everything from Modules 1–10 (RA trees, the optimizer rewriting correlated subqueries into joins, CTE materialization decisions, trigger firing). **The full deep-dive treatment of each stage — cost estimation, join algorithms, access paths, statistics, index selection — is reserved for Part 8.**

---

## 11.0 Chapter Roadmap

```
Module 11: SQL Execution Internals (Introduction)
│
├── 11.1 The Big Picture: From SQL Text to Result Set
├── 11.2 Stage 1: SQL Parser
├── 11.3 Stage 2: Query Rewrite
├── 11.4 Stage 3: Optimizer
├── 11.5 Stage 4: Execution Plan
├── 11.6 Stage 5: Execution Engine
├── 11.7 Stage 6: Result Generation
├── 11.8 Reading a Basic EXPLAIN Output
├── 11.9 PostgreSQL vs MySQL Notes
├── 11.10 Interview Questions & MCQs
├── 11.11 Cheat Sheet
└── 11.12 Exercises
```

---

## 11.1 The Big Picture: From SQL Text to Result Set

```
┌──────────────┐   ┌───────────────┐   ┌───────────┐   ┌────────────────┐   ┌─────────────────┐   ┌─────────────┐
│  SQL Query    │──▶│  SQL Parser    │──▶│  Query    │──▶│  Optimizer      │──▶│  Execution Plan   │──▶│  Execution   │
│  (raw text)   │   │  (syntax +     │   │  Rewrite   │   │  (cost-based     │   │  (chosen physical  │   │  Engine       │
│               │   │  semantic      │   │  (logical  │   │  search over     │   │  operator tree,    │   │  (runs the    │
│               │   │  check → parse │   │  transform-│   │  many equivalent │   │  e.g. Hash Join,   │   │  plan, row by │
│               │   │  tree)         │   │  -ations)  │   │  RA trees)        │   │  Index Scan)        │   │  row / batch) │
└──────────────┘   └───────────────┘   └───────────┘   └────────────────┘   └─────────────────┘   └──────┬──────┘
                                                                                                            │
                                                                                                            ▼
                                                                                                   ┌─────────────────┐
                                                                                                   │ Result Generation │
                                                                                                   │ (rows streamed/   │
                                                                                                   │ returned to client)│
                                                                                                   └─────────────────┘
```

This pipeline is the **practical, physical implementation** of the theoretical journey you traced in Module 1 and Module 2: **SQL (calculus-flavored) → Relational Algebra (procedural) → an actual, chosen sequence of physical operations.** Every stage below maps onto a specific part of that journey.

---

## 11.2 Stage 1: SQL Parser

**What it does:** Takes the **raw SQL text** you typed and:
1. **Lexes/tokenizes** it (breaks the string into keywords, identifiers, literals, operators).
2. **Parses** it against SQL's formal grammar, producing a **parse tree** (a syntactic representation of the query's structure).
3. **Performs semantic validation** — checks that referenced tables/columns actually exist, that data types are compatible, that you have permission to access the referenced objects, and that aggregate functions aren't misused (recall Module 6, Section 6.7's `WHERE`-can't-use-aggregates rule — this is exactly where that gets caught).

```sql
SELECT name FROM Employees WHERE salary > 50000;
```

**Conceptual Parse Tree:**
```
              SELECT
             /       \
     SELECT-LIST     FROM-CLAUSE
          │                │
        name          Employees
                             │
                       WHERE-CLAUSE
                             │
                       salary > 50000
```

**Errors caught here:** syntax errors (`SELECT FORM Employees` — typo), semantic errors (referencing a non-existent column, using `SUM()` in `WHERE` — recall Module 6, using ambiguous column names in a multi-table join without qualification).

> This parse tree is the very first step toward the **Relational Algebra expression tree** you learned to build by hand in Module 1, Section 1.11 — the parser produces the initial, unoptimized structural representation that later stages will transform.

---

## 11.3 Stage 2: Query Rewrite

**What it does:** Applies **logical, semantics-preserving transformations** to the parsed query — converting it into an **initial Relational Algebra tree**, and then applying rule-based rewrites that are **always beneficial (or at least never harmful)**, independent of actual data statistics.

### Examples of Rewrite Rules (Direct Callbacks to Earlier Modules)

| Rewrite | Example | Module Reference |
|---|---|---|
| **View/CTE expansion (inlining)** | A query against a `View` or non-materialized `CTE` has the view/CTE's definition substituted directly into the main query tree | Module 4 (CTE materialization decisions), Module 8 (Views) |
| **Subquery flattening / unnesting** | A correlated subquery (`WHERE EXISTS (...)`) is rewritten into an equivalent semi-join | Module 3, Section 3.2 |
| **Predicate simplification** | `WHERE 1=1 AND salary > 50000` simplifies to `WHERE salary > 50000` | — |
| **Constant folding** | `WHERE salary > 40000 + 10000` becomes `WHERE salary > 50000`, computed once rather than per-row | — |
| **`NOT IN` → Anti-Join consideration** | Where provably NULL-safe, `NOT IN` may be rewritten toward an anti-join-style plan; otherwise the NULL-check semantics (Module 1, Section 1.8.6) must be preserved exactly | Module 1, Module 3 |

```
Original (post-parse) query tree:              After Query Rewrite:

    π_(name)                                        π_(name)
       │                                                │
    σ_(salary > 50000)                              σ_(salary > 50000)
       │                                                │
    "View: HighEarners"  ◀── expanded/inlined ──▶   σ_(dept='IT')  ◀── HighEarners' own
       │                                                │              definition, substituted in
    Employees                                        Employees
```

> This stage is where Module 1's RA rewrite laws (commutativity of selections, the join-as-product-plus-selection identity) are put into actual, automated practice by the engine.

---

## 11.4 Stage 3: Optimizer

**What it does:** Unlike Query Rewrite (rule-based, always safe), the **Optimizer** makes **cost-based decisions** — it considers **multiple equivalent physical execution strategies** for the (already logically rewritten) query, **estimates the cost of each**, and **picks the cheapest one it can find**, based on:

- **Table statistics**: row counts, column value distributions/histograms, index selectivity estimates.
- **Available indexes**: whether an index scan is possible/cheaper than a sequential scan for a given predicate (Module 1, Section 1.5.1's "sargable predicates" concept becomes directly actionable here).
- **Join order and join algorithm choice**: for a query joining 4 tables, there are many possible join orders and, for each pair, multiple possible join algorithms (Nested Loop, Hash Join, Merge Join — full detail in Part 8) — the optimizer searches this space (often using dynamic programming for small numbers of tables, or heuristics/genetic algorithms for very large joins) to find a low-cost plan.

```
Example: The SAME logical query can be executed via multiple different PHYSICAL strategies:

Logical: Employees ⋈_(dept_id) Departments, filtered WHERE salary > 50000

Physical Option A: Sequential scan Employees, filter, THEN Nested Loop Join with Departments
Physical Option B: Index scan Employees using an index on salary, THEN Hash Join with Departments
Physical Option C: Index scan Departments first (if very small), THEN Nested Loop into Employees

The optimizer ESTIMATES the cost of each (using statistics: how selective is salary>50000?
how many rows in each table? are there useful indexes?) and picks the cheapest ESTIMATE.
```

> 🎯 **Critical Fact for Interviews:** The optimizer's choice is based on **estimated cost, not actual, guaranteed-optimal cost** — outdated or missing statistics (e.g., a table that grew 100x since the last `ANALYZE`/statistics update) can cause the optimizer to pick a genuinely poor plan, even though the logic is 100% correct. This is *the* most common real-world root cause of "this query used to be fast and now it's slow" incidents, and a very common senior-level interview/debugging scenario. **Full treatment of cost models, statistics, and index selection is in Part 8.**

---

## 11.5 Stage 4: Execution Plan

**What it is:** The optimizer's final chosen output — a concrete, **physical operator tree** specifying exactly **which algorithm** will be used for each operation (e.g., "Hash Join," not just "Join"; "Index Scan using idx_salary," not just "Selection").

```
Execution Plan (conceptual, PostgreSQL-style):

  Nested Loop
    -> Index Scan using idx_dept_id on Departments
         Filter: dept_name = 'IT'
    -> Index Scan using idx_emp_dept_id on Employees
         Index Cond: (dept_id = departments.dept_id)
         Filter: salary > 50000
```

This is precisely what `EXPLAIN` shows you (Section 11.8) — the **human-readable rendering of the execution plan** the optimizer decided on.

---

## 11.6 Stage 5: Execution Engine

**What it does:** Actually **runs** the chosen execution plan against the real, live data — walking the physical operator tree, typically using a model where each operator pulls rows from its children **on demand** (a classic pattern called the **iterator/Volcano model**: each node exposes an `open()`/`next()`/`close()` interface, and the top of the tree pulls rows one at a time, cascading `next()` calls down through the tree as needed) — though many modern engines use **vectorized/batch execution** (processing chunks of rows at once) for better CPU cache efficiency rather than pure row-at-a-time processing.

```
Data flows UP through the tree; control (next() calls) flows DOWN:

        [ Nested Loop ]  ◀── caller asks: "give me the next result row"
         /            \
   [Index Scan]   [Index Scan]
   Departments     Employees
   (produces rows   (produces rows
    one at a time,   one at a time,
    on request)      on request)
```

This is where **triggers fire** (Module 10 — a row-level `AFTER INSERT` trigger executes as each row is actually written during this stage), **transactions** are managed (full detail in Parts 6–7), and where the actual **I/O** (reading data pages from disk/buffer cache) happens.

---

## 11.7 Stage 6: Result Generation

**What it does:** Formats and streams the final rows produced by the Execution Engine back to the client (your SQL tool, application driver, etc.) — applying any final `ORDER BY`/`LIMIT` not already handled earlier in the plan, and converting internal row representations into the client-facing wire protocol format.

---

## 11.8 Reading a Basic EXPLAIN Output

**PostgreSQL:**
```sql
EXPLAIN SELECT name FROM Employees WHERE salary > 50000;
```
```
Seq Scan on employees  (cost=0.00..18.50 rows=250 width=32)
  Filter: (salary > 50000)
```

**Key elements to recognize (full depth in Part 8):**

| Element | Meaning |
|---|---|
| `Seq Scan` / `Index Scan` | The **access method** chosen — sequential (full table) scan vs index-based scan |
| `cost=0.00..18.50` | Estimated **startup cost .. total cost**, in arbitrary optimizer cost units (not milliseconds) |
| `rows=250` | The optimizer's **estimated** row count this step will produce (NOT necessarily the actual count — statistics can be wrong!) |
| `width=32` | Estimated average row width in bytes |
| `Filter:` | A condition applied **after** rows are fetched (not used to narrow the scan itself, unlike an `Index Cond`) |

```sql
EXPLAIN ANALYZE SELECT name FROM Employees WHERE salary > 50000;
```
`EXPLAIN ANALYZE` actually **executes** the query and shows **real, measured** row counts and timing alongside the estimates — the single most important tool for diagnosing "the optimizer's estimate was wrong" scenarios (Section 11.4's key interview fact).

> This section is intentionally a light preview — **query plan reading, join algorithm identification, and cost-model interpretation in full depth are covered in Part 8.**

---

## 11.9 PostgreSQL vs MySQL Notes

| Feature | PostgreSQL | MySQL |
|---|---|---|
| Plan inspection command | `EXPLAIN`, `EXPLAIN ANALYZE`, `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` for rich detail | `EXPLAIN`, `EXPLAIN ANALYZE` (MySQL 8.0.18+), `EXPLAIN FORMAT=JSON` |
| Optimizer statistics refresh | `ANALYZE tablename;` (also runs automatically via autovacuum's analyze component) | `ANALYZE TABLE tablename;` |
| Execution model | Primarily iterator/Volcano-style, with some batch optimizations | Historically iterator-style; MySQL 8.0+ introduced a hash join executor and other improvements |
| Query rewrite visibility | `EXPLAIN (COSTS OFF)` / plan output reflects rewrites | Plan output similarly reflects internal rewrites (e.g., derived table merging) |

---

## 11.10 Interview Questions & MCQs

### Conceptual
1. Describe, at a high level, the journey a SQL query takes from raw text to a returned result set.
2. What is the difference between the Query Rewrite stage and the Optimizer stage?
3. Why can two logically equivalent queries have very different actual execution plans?
4. What does it mean that the optimizer's decisions are "cost-based" rather than "correctness-based"? What's the risk of outdated statistics?
5. What is the Volcano/iterator execution model, and how does it differ from vectorized/batch execution?

### Applied
6. Given an `EXPLAIN` output showing a `Seq Scan` on a large, frequently-filtered table, what would you investigate first, and why?
7. Explain why `EXPLAIN ANALYZE`'s actual row counts diverging significantly from the estimated row counts is a red flag, and what it typically indicates.
8. Trace how a query against a non-materialized CTE (Module 4) or a View (Module 8) is transformed during the Query Rewrite stage.

### MCQs

**Q1.** Which stage is responsible for catching a reference to a non-existent column?
 a) Optimizer  b) Execution Engine  c) SQL Parser  d) Result Generation

**Q2.** Which stage decides whether to use a sequential scan vs an index scan, based on cost estimates?
 a) Query Rewrite  b) Optimizer  c) Parser  d) Result Generation

**Q3.** `EXPLAIN ANALYZE` differs from plain `EXPLAIN` because it:
 a) Only shows estimated costs  b) Actually executes the query and shows real measured stats  c) Skips the optimizer entirely  d) Only works on SELECT statements

**Q4.** The most common real-world cause of "a query used to be fast and is now slow" is:
 a) A trigger was added  b) Outdated/stale table statistics leading to poor cost estimates  c) The SQL parser has a bug  d) Result generation is slow

**Q5.** Query Rewrite rules are best described as:
 a) Cost-based, data-dependent choices  b) Rule-based, always-safe logical transformations  c) Random plan selection  d) Only applicable to views

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **c) SQL Parser**
2. **b) Optimizer**
3. **b) Actually executes the query and shows real measured stats**
4. **b) Outdated/stale table statistics leading to poor cost estimates**
5. **b) Rule-based, always-safe logical transformations**

</details>

---

## 11.11 Cheat Sheet — Module 11

```
┌───────────────────────────────────────────────────────────────────┐
│           SQL EXECUTION INTERNALS (INTRO) — QUICK REFERENCE            │
├───────────────────────────┬───────────────────────────────────────┤
│ 1. Parser                    │ Syntax + semantic check → parse tree    │
│ 2. Query Rewrite               │ Rule-based, always-safe RA transforms   │
│                                  │ (view/CTE inlining, subquery flattening)│
│ 3. Optimizer                      │ Cost-based search over physical plans;  │
│                                     │ uses statistics, indexes, join algos   │
│ 4. Execution Plan                    │ The chosen physical operator tree      │
│                                        │ (what EXPLAIN shows you)               │
│ 5. Execution Engine                     │ Runs the plan (iterator/Volcano or     │
│                                           │ vectorized model); fires triggers here │
│ 6. Result Generation                       │ Streams/formats final rows to client   │
│ EXPLAIN                                      │ Shows the estimated plan               │
│ EXPLAIN ANALYZE                                │ Actually runs it; shows real stats too │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 11.12 Exercises

### Beginner
1. Run `EXPLAIN` on a simple `SELECT ... WHERE` query and identify the access method shown.
2. Explain, in your own words, why the Parser stage would reject `SELECT nam FROM Employees;` (typo in column name).

### Intermediate
3. Compare `EXPLAIN` and `EXPLAIN ANALYZE` output for the same query and explain what the differences tell you.
4. Describe how a query against a View (Module 8) is transformed during Query Rewrite before reaching the Optimizer.

### Advanced
5. Explain, with an example, how outdated statistics could cause the optimizer to choose a Nested Loop Join when a Hash Join would actually be far cheaper (full join-algorithm detail is in Part 8, but reason about it at a conceptual level here).
6. Describe the Volcano/iterator execution model's `open()`/`next()`/`close()` interface conceptually, and explain why this model naturally supports lazily streaming results (e.g., allowing a `LIMIT 10` query to potentially avoid scanning an entire large table).

### Scenario-Based
7. A production query has been running fine for months but suddenly became slow after a large data migration. Using this module's concepts, describe your first diagnostic steps.

### Practice Links
- [PostgreSQL EXPLAIN Documentation](https://www.postgresql.org/docs/current/using-explain.html)
- [Use The Index, Luke — SQL Indexing & Tuning](https://use-the-index-luke.com/)

---

## Chapter Summary

- SQL execution proceeds through six conceptual stages: **Parser → Query Rewrite → Optimizer → Execution Plan → Execution Engine → Result Generation**.
- The **Parser** builds a parse tree and validates syntax/semantics — catching the kinds of errors flagged throughout Modules 1–10 (bad aggregate placement, unknown columns).
- **Query Rewrite** applies safe, rule-based logical transformations (view/CTE inlining, subquery flattening) — directly implementing Module 1's RA rewrite laws.
- The **Optimizer** makes **cost-based**, statistics-dependent decisions among many equivalent physical plans — and its estimates can be **wrong** if statistics are stale, a frequent real-world performance root cause.
- The **Execution Plan** is the concrete, chosen physical operator tree — what `EXPLAIN` shows you.
- The **Execution Engine** actually runs the plan (commonly via an iterator/Volcano model), and is where triggers fire and I/O happens.
- **This module is intentionally an introduction** — cost models, join algorithms, access paths, and index-driven optimization are covered in full depth in **Part 8**.

---

## Cross-References for Later Parts / Modules
- **Full query optimizer internals: cost models, statistics, join algorithms (Nested Loop, Hash, Merge), access paths** → Part 8
- **Indexing strategy and design** → Part 8
- **Transaction management and how it interacts with the Execution Engine** → Parts 6–7
- **Storage engines and how they affect Execution Engine I/O** → Part 8/9

---

> ✅ **Module 11 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 12"** to proceed to **Module 12: Advanced SQL Best Practices** — writing readable SQL, query decomposition, reusing CTEs, avoiding unnecessary subqueries, choosing joins wisely, NULL handling, and defensive SQL.
