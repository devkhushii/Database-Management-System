# The Complete DBMS Handbook — Part 4

## 📘 MODULE 5: WINDOW FUNCTIONS

> **Continuing from Module 4 (CTEs).** Window functions are, alongside CTEs, one of the two most heavily interview-tested "advanced SQL" topics. They also directly replace many of Module 3's correlated-subquery and `LATERAL`-join patterns with cleaner, often faster syntax — pay close attention to those callbacks.

---

## 5.0 Chapter Roadmap

```
Module 5: Window Functions
│
├── 5.1 What is a Window Function? The OVER Clause
├── 5.2 Why Window Functions Exist
├── 5.3 PARTITION BY and ORDER BY within OVER
├── 5.4 Ranking Functions (ROW_NUMBER, RANK, DENSE_RANK, NTILE)
├── 5.5 Aggregate Window Functions (SUM, AVG, COUNT, MIN, MAX)
├── 5.6 Value Functions (LEAD, LAG, FIRST_VALUE, LAST_VALUE, NTH_VALUE)
├── 5.7 Window Frames (ROWS vs RANGE vs GROUPS)
├── 5.8 Window Functions vs GROUP BY vs Correlated Subqueries
├── 5.9 PostgreSQL vs MySQL Notes
├── 5.10 Best Practices
├── 5.11 Common Mistakes & Edge Cases
├── 5.12 Performance Notes
├── 5.13 Interview Questions & MCQs
├── 5.14 Cheat Sheet
└── 5.15 Exercises
```

---

## 5.1 What is a Window Function? The OVER Clause

**Definition:** A **window function** performs a calculation **across a set of rows that are related to the current row** — called its **"window"** — **without collapsing those rows into a single output row**, unlike `GROUP BY`. Every input row still produces exactly one output row, but that row now also carries a computed value based on its "neighborhood."

**General Syntax:**
```sql
function_name(expression) OVER (
    [PARTITION BY partition_column(s)]
    [ORDER BY sort_column(s)]
    [frame_clause]
)
```

### The Single Most Important Distinction: Window Functions vs GROUP BY

```
GROUP BY:  Many rows  ──▶  ONE row per group   (rows are COLLAPSED)
Window Fn: Many rows  ──▶  Many rows, each ANNOTATED with a computed value (rows are PRESERVED)
```

**Side-by-Side Example:**

`Employees(name, dept, salary)`:

| name | dept | salary |
|---|---|---|
| Aditi | Sales | 55000 |
| Meera | Sales | 61000 |
| Rohan | IT | 72000 |
| Kavya | IT | 68000 |

```sql
-- GROUP BY: collapses to 2 rows (one per dept)
SELECT dept, AVG(salary) AS avg_salary
FROM Employees
GROUP BY dept;
```
Result: 2 rows (`Sales, 58000`), (`IT, 70000`)

```sql
-- Window function: keeps all 4 rows, each annotated with its dept average
SELECT name, dept, salary,
       AVG(salary) OVER (PARTITION BY dept) AS dept_avg_salary
FROM Employees;
```
Result:

| name | dept | salary | dept_avg_salary |
|---|---|---|---|
| Aditi | Sales | 55000 | 58000 |
| Meera | Sales | 61000 | 58000 |
| Rohan | IT | 72000 | 70000 |
| Kavya | IT | 68000 | 70000 |

> 🎯 **This is THE defining fact to remember about window functions, and it is asked in almost every SQL interview**: *window functions compute aggregates/rankings "alongside" each row, without reducing the row count — `GROUP BY` reduces the row count.*

### Real-life Analogy
Imagine a race with runners grouped by age category. `GROUP BY` is like publishing **only the winning time per age category** (one row per category — the individual runners disappear from the output). A window function is like publishing the **full list of every runner**, but with an extra column next to each name showing **their rank within their own age category** — nobody disappears from the list.

---

## 5.2 Why Window Functions Exist

| Problem Solved | Explanation |
|---|---|
| **"Row + aggregate together"** | Before window functions, getting a row's value AND a group-level aggregate together required a self-join or correlated subquery (Module 3) — clunky and often slow. |
| **Ranking within groups** | "Top N per group," "Nth highest salary per department" — previously required correlated subqueries or `LATERAL` (Module 3, Section 3.6) — window functions do this natively and (usually) faster. |
| **Running totals / moving averages** | Financial reports, time-series analysis need cumulative sums, moving averages — mathematically impossible to express with plain aggregate functions alone (which always collapse rows). |
| **Row-to-row comparison** | "Compare this row to the previous/next row" (`LEAD`/`LAG`) — previously required self-joins on a manually computed row offset. |
| **Standard-compliant, optimizer-friendly** | Being a first-class SQL construct (not a workaround), window functions are recognized and optimized directly by the query planner, often outperforming equivalent correlated-subquery rewrites. |

---

## 5.3 PARTITION BY and ORDER BY within OVER

### PARTITION BY — "Which rows are in my window?"

`PARTITION BY` divides the rows into groups (**partitions**) — conceptually similar to `GROUP BY`, but **without collapsing rows**. The window function is computed **independently within each partition**. If `PARTITION BY` is omitted, the **entire table is treated as one single partition**.

### ORDER BY (inside OVER) — "In what order should rows be processed within the window?"

This `ORDER BY` is completely independent of any `ORDER BY` at the end of the whole query — it defines the **row sequence used to compute the function** (essential for ranking functions, running totals, `LEAD`/`LAG`). Without it, ranking functions have no defined sequence, and aggregate window functions default to treating the whole partition as one frame (Section 5.7 explains this default precisely).

```
Execution Model (conceptual):

Step 1: PARTITION BY splits rows into independent groups
Step 2: Within each partition, ORDER BY (inside OVER) establishes row sequence
Step 3: The function is computed for each row using its window
        (either the whole partition, or a FRAME within it — Section 5.7)
Step 4: Every original row is preserved, now with the computed value attached
```

```
        Employees Table
┌──────┬──────┬────────┐
│ name │ dept │ salary │
├──────┼──────┼────────┤
│ Aditi│Sales │ 55000  │──┐
│ Meera│Sales │ 61000  │──┤ PARTITION: Sales
├──────┼──────┼────────┤
│ Rohan│  IT  │ 72000  │──┐
│ Kavya│  IT  │ 68000  │──┤ PARTITION: IT
└──────┴──────┴────────┘
      Each partition computed independently
```

---

## 5.4 Ranking Functions

All ranking functions **require an `ORDER BY` inside `OVER()`** — ranking is meaningless without a defined sequence.

**Sample Data — `Employees` (with a tie deliberately included):**

| name | dept | salary |
|---|---|---|
| Rohan | IT | 72000 |
| Kavya | IT | 68000 |
| Aditi | IT | 68000 |
| Meera | IT | 55000 |

### 5.4.1 ROW_NUMBER()

**Definition:** Assigns a **unique, sequential integer** (1, 2, 3, ...) to each row within its partition, based on the `ORDER BY` — **ties get different, arbitrary-but-deterministic-per-run numbers** (the exact tie-breaking order depends on the engine unless you add a tie-breaking column).

```sql
SELECT name, salary,
       ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
FROM Employees;
```

| name | salary | rn |
|---|---|---|
| Rohan | 72000 | 1 |
| Kavya | 68000 | 2 |
| Aditi | 68000 | 3 |
| Meera | 55000 | 4 |

> Notice Kavya and Aditi are **tied** at 68000, but `ROW_NUMBER()` still forces distinct numbers (2 and 3) — the assignment between them is **not guaranteed deterministic** unless you add a secondary `ORDER BY` tie-breaker (e.g., `ORDER BY salary DESC, name ASC`).

### 5.4.2 RANK()

**Definition:** Assigns rank with **gaps after ties** — tied rows get the **same rank**, and the next rank **skips** to account for the number of tied rows.

```sql
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS rnk
FROM Employees;
```

| name | salary | rnk |
|---|---|---|
| Rohan | 72000 | 1 |
| Kavya | 68000 | 2 |
| Aditi | 68000 | 2 |
| Meera | 55000 | **4** ← notice the gap (skips rank 3) |

### 5.4.3 DENSE_RANK()

**Definition:** Like `RANK()`, tied rows get the same rank — but **no gap** is left in the ranking sequence afterward.

```sql
SELECT name, salary,
       DENSE_RANK() OVER (ORDER BY salary DESC) AS drnk
FROM Employees;
```

| name | salary | drnk |
|---|---|---|
| Rohan | 72000 | 1 |
| Kavya | 68000 | 2 |
| Aditi | 68000 | 2 |
| Meera | 55000 | **3** ← no gap |

### Side-by-Side Comparison (Most Frequently Tested Table in This Module)

| salary | ROW_NUMBER() | RANK() | DENSE_RANK() |
|---|---|---|---|
| 72000 | 1 | 1 | 1 |
| 68000 | 2 | 2 | 2 |
| 68000 | 3 | 2 | 2 |
| 55000 | 4 | 4 | 3 |

> 🎯 **Memorize this table cell-by-cell — it is asked, in some form, in nearly every SQL interview that covers window functions.** The mnemonic: `ROW_NUMBER` never repeats and never skips; `RANK` repeats on ties and skips after; `DENSE_RANK` repeats on ties and never skips.

### 5.4.4 NTILE(n)

**Definition:** Divides the ordered rows within a partition into **`n` roughly equal-sized buckets**, numbered 1 to n. If rows don't divide evenly, **earlier buckets get the extra rows**.

```sql
SELECT name, salary,
       NTILE(2) OVER (ORDER BY salary DESC) AS quartile_bucket
FROM Employees;
```

| name | salary | quartile_bucket |
|---|---|---|
| Rohan | 72000 | 1 |
| Kavya | 68000 | 1 |
| Aditi | 68000 | 2 |
| Meera | 55000 | 2 |

**Real-world use:** Splitting customers into quartiles/deciles by spending (`NTILE(4)`), computing percentile-style buckets for reporting.

### Classic Use-Case: "Find the Nth Highest Salary" (Very High-Frequency Interview Question)

```sql
SELECT name, salary
FROM (
    SELECT name, salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS drnk
    FROM Employees
) ranked
WHERE drnk = 2;   -- 2nd highest DISTINCT salary
```

> **Why `DENSE_RANK` and not `ROW_NUMBER` here?** If two employees are tied for the highest salary, "the 2nd highest salary" should logically mean the *next distinct value*, which `DENSE_RANK` correctly captures. `ROW_NUMBER` would incorrectly treat one of the tied top earners as "2nd," which is usually **not** the intended business meaning. This exact reasoning is a favorite interview follow-up question.

---

## 5.5 Aggregate Window Functions

**Definition:** Standard aggregate functions (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) can be used with `OVER()` to compute a **running or partition-wide aggregate**, without collapsing rows.

### Partition-Wide Aggregate (No ORDER BY inside OVER)

```sql
SELECT name, dept, salary,
       SUM(salary) OVER (PARTITION BY dept) AS dept_total,
       AVG(salary) OVER (PARTITION BY dept) AS dept_avg,
       COUNT(*)    OVER (PARTITION BY dept) AS dept_headcount
FROM Employees;
```
Here, **every row in a partition gets the same aggregate value** — because without an `ORDER BY`, the default frame is "the entire partition" (explained precisely in Section 5.7).

### Running Total (WITH ORDER BY inside OVER)

```sql
SELECT name, salary,
       SUM(salary) OVER (ORDER BY salary) AS running_total
FROM Employees;
```

| name | salary | running_total |
|---|---|---|
| Meera | 55000 | 55000 |
| Aditi | 68000 | 123000 |
| Kavya | 68000 | *191000 or 246000?* |
| Rohan | 72000 | ... |

> ⚠️ **This is exactly where the default window frame behavior with ties becomes critical** — see Section 5.7 (`RANGE` vs `ROWS`) for the precise, correct answer to what happens with tied `ORDER BY` values in a running total. This is a genuinely subtle, high-value interview trap.

### Real-World Use Cases
- **Running total of sales by date** (cumulative revenue).
- **Percentage of total** (`salary / SUM(salary) OVER () * 100`) — note the empty `OVER ()` means "the whole table is one partition."
- **Department headcount attached to every employee row** for reporting dashboards.

---

## 5.6 Value Functions

### 5.6.1 LEAD() and LAG()

**Definition:** `LAG(expr, offset, default)` looks **backward** `offset` rows (default 1); `LEAD(expr, offset, default)` looks **forward** `offset` rows. Both require `ORDER BY` inside `OVER()` to define "forward"/"backward." If there's no such row (e.g., first row has no previous row), the result is `NULL` unless a `default` value is specified.

**Sample Data — `Sales(sale_date, amount)`:**

| sale_date | amount |
|---|---|
| 2026-01-01 | 1000 |
| 2026-01-02 | 1200 |
| 2026-01-03 | 900 |
| 2026-01-04 | 1500 |

```sql
SELECT sale_date, amount,
       LAG(amount, 1) OVER (ORDER BY sale_date) AS prev_day_amount,
       LEAD(amount, 1) OVER (ORDER BY sale_date) AS next_day_amount,
       amount - LAG(amount, 1) OVER (ORDER BY sale_date) AS day_over_day_change
FROM Sales;
```

| sale_date | amount | prev_day_amount | next_day_amount | day_over_day_change |
|---|---|---|---|---|
| 2026-01-01 | 1000 | NULL | 1200 | NULL |
| 2026-01-02 | 1200 | 1000 | 900 | 200 |
| 2026-01-03 | 900 | 1200 | 1500 | -300 |
| 2026-01-04 | 1500 | 900 | NULL | 600 |

**Real-world use:** day-over-day / month-over-month growth, detecting consecutive login streaks, comparing a row to its neighbor without a self-join.

### 5.6.2 FIRST_VALUE() and LAST_VALUE()

**Definition:** Returns the value of the expression evaluated at the **first**/**last** row of the current **window frame** (not necessarily the whole partition — this distinction is critical, see the warning below).

```sql
SELECT name, dept, salary,
       FIRST_VALUE(name) OVER (PARTITION BY dept ORDER BY salary DESC) AS top_earner_in_dept
FROM Employees;
```

| name | dept | salary | top_earner_in_dept |
|---|---|---|---|
| Rohan | IT | 72000 | Rohan |
| Kavya | IT | 68000 | Rohan |
| Aditi | IT | 68000 | Rohan |
| Meera | IT | 55000 | Rohan |

> ⚠️ **The #1 `LAST_VALUE()` Gotcha (extremely commonly asked/miscoded in interviews):**
> ```sql
> SELECT name, salary,
>        LAST_VALUE(name) OVER (PARTITION BY dept ORDER BY salary DESC) AS bottom_earner
> FROM Employees;
> ```
> This does **NOT** return the lowest earner as you might expect! By default, the window frame for `ORDER BY`-based functions is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (explained fully in 5.7) — meaning `LAST_VALUE` only sees rows **up to the current row**, so it just returns **the current row's own value** at every row, not the true last row of the partition!
>
> **Fix:** explicitly extend the frame to the full partition:
> ```sql
> LAST_VALUE(name) OVER (
>     PARTITION BY dept ORDER BY salary DESC
>     ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
> ) AS bottom_earner
> ```

### 5.6.3 NTH_VALUE()

**Definition:** Returns the value of the expression at the **Nth row** of the window frame.

```sql
SELECT name, salary,
       NTH_VALUE(name, 2) OVER (
           PARTITION BY dept ORDER BY salary DESC
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS second_highest_earner_name
FROM Employees;
```
Returns the name of whoever is **2nd** in the ordered frame (per partition) — again requiring the full-frame extension as shown above, or it will only "see" the 2nd row **if the current row's frame has reached that far**.

---

## 5.7 Window Frames (ROWS vs RANGE vs GROUPS)

This is the **most conceptually advanced, and most commonly misunderstood**, part of window functions — it explains exactly *why* the `LAST_VALUE` gotcha above happens.

### The Frame Clause Syntax

```sql
function() OVER (
    [PARTITION BY ...]
    [ORDER BY ...]
    { ROWS | RANGE | GROUPS } BETWEEN <start> AND <end>
)
```

**Frame boundary options:**
- `UNBOUNDED PRECEDING` — the first row of the partition
- `N PRECEDING` — N rows/values before the current row
- `CURRENT ROW`
- `N FOLLOWING` — N rows/values after the current row
- `UNBOUNDED FOLLOWING` — the last row of the partition

### ROWS — Physical Row Count

`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` means **exactly the physical rows**: the current row plus the 2 rows immediately before it (by position), **regardless of whether their values are tied**.

```
Row Position:    1     2     3     4     5
Frame for row 4 (ROWS BETWEEN 2 PRECEDING AND CURRENT ROW):
                       [ 2 ][ 3 ][ 4 ]
                       └─────────────┘
                    exactly 3 physical rows
```

### RANGE — Logical Value Range

`RANGE BETWEEN ... ` groups by **value equality** according to the `ORDER BY` expression — **all rows with the same `ORDER BY` value are treated as part of the same frame position**, not counted individually.

```
ORDER BY salary:   55000   68000   68000   72000
                            (tied!)
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW,
evaluated at either of the two 68000 rows:
   → frame includes BOTH 68000 rows (since RANGE treats ties as the same "peer group"),
     not just up to the "current physical row"
```

### GROUPS — Peer-Group Count

`GROUPS BETWEEN 1 PRECEDING AND CURRENT ROW` counts **distinct peer groups** (sets of tied rows under `ORDER BY`) rather than individual physical rows — a middle ground between `ROWS` (individual rows) and `RANGE` (all-or-nothing on ties).

### Comparison Table

| Frame Type | Unit of Counting | Handles Ties By |
|---|---|---|
| `ROWS` | Physical row count | Ignores ties — purely positional |
| `RANGE` | Logical value equality | ALL tied rows included/excluded together as one "peer group" |
| `GROUPS` | Count of distinct peer groups | Counts tied-value clusters as a single "group" unit for the N in "N preceding/following" |

### The Critical Default Behavior (Root Cause of the LAST_VALUE Gotcha)

```
IF ORDER BY is present inside OVER() and no frame clause is explicitly written,
the DEFAULT frame is:

    RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

This means: "everything from the start of the partition up to
(and including all rows TIED WITH) the current row" — NOT the whole partition!

IF ORDER BY is ABSENT inside OVER(), the DEFAULT frame is:

    RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    (i.e., the entire partition — this is why Section 5.5's dept-wide
    SUM/AVG examples worked correctly without any explicit frame clause)
```

> 🎯 **This single fact resolves nearly every "why did my running total / LAST_VALUE not work as expected" bug**, and is one of the highest-value, most senior-level facts in this entire module. **Always be explicit about your frame clause** when using `ORDER BY` + an aggregate/value function, unless you specifically want the "up-to-current-row" running behavior.

### Frame Visualization — Running Total with Ties (RANGE vs ROWS)

```
Data (ordered by salary): 55000, 68000, 68000, 72000

Using RANGE (default with ORDER BY) — ties share the SAME cumulative value:
  55000 → running_total = 55000
  68000 → running_total = 55000 + 68000 + 68000 = 191000   (both tied rows included together)
  68000 → running_total = 191000                             (same as above — tied peer)
  72000 → running_total = 191000 + 72000 = 263000

Using ROWS (explicit, purely positional) — ties are NOT merged:
  55000 → running_total = 55000
  68000 → running_total = 55000 + 68000 = 123000
  68000 → running_total = 123000 + 68000 = 191000
  72000 → running_total = 191000 + 72000 = 263000
```

> This exact RANGE-vs-ROWS discrepancy on tied values is a genuinely advanced, senior-level ("SDE-3") interview/exam question — most engineers only ever use the default and are surprised the first time ties break their running-total logic.

---

## 5.8 Window Functions vs GROUP BY vs Correlated Subqueries

| Need | GROUP BY | Correlated Subquery (Module 3) | Window Function |
|---|---|---|---|
| One aggregate row per group | ✅ Best fit | Possible but awkward | Possible but overkill |
| Aggregate value **alongside** every original row | ❌ Impossible directly (needs a self-join back) | ✅ Works, but O(n×m) risk | ✅ Best fit, usually fastest |
| Ranking within groups (Top-N per group) | ❌ Not directly possible | ✅ Works via `LATERAL`/correlated subquery | ✅ Best fit — `RANK()`/`ROW_NUMBER()` |
| Row-to-row comparison (previous/next row) | ❌ Not possible | Possible via self-join on computed offset (clunky) | ✅ Best fit — `LEAD()`/`LAG()` |
| Running totals / moving averages | ❌ Not possible | Very awkward | ✅ Best fit |

**"Top-N per group" — Window Function version of Module 3's LATERAL example:**
```sql
-- Recall Module 3, Section 3.6's LATERAL solution for "top 2 per department"
-- Here is the window-function equivalent, often preferred for readability/performance:
SELECT name, dept, salary
FROM (
    SELECT name, dept, salary,
           ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
    FROM Employees
) ranked
WHERE rn <= 2;
```

---

## 5.9 PostgreSQL vs MySQL Notes

| Feature | PostgreSQL | MySQL |
|---|---|---|
| Window functions overall | ✅ Full support since PG 9.4 | ✅ Full support from MySQL 8.0+ (**not available at all in MySQL 5.x**) |
| `GROUPS` frame mode | ✅ Supported (PG 11+) | ❌ Not supported — only `ROWS` and `RANGE` |
| `FILTER` clause combined with window aggregates | ✅ Supported (`SUM(x) FILTER (WHERE ...) OVER (...)`) | ❌ Not directly supported the same way |
| Named windows (`WINDOW w AS (...)`, reused via `OVER w`) | ✅ Supported | ✅ Supported |
| `IGNORE NULLS` / `RESPECT NULLS` for LAG/LEAD/FIRST_VALUE | ❌ Not natively supported (needs workaround) | ❌ Not supported either (unlike Oracle/SQL Server, which do support this) |

> **Oracle/SQL Server note (relevant for cross-platform interview prep):** Oracle and SQL Server support `IGNORE NULLS` (e.g., `LAST_VALUE(col IGNORE NULLS) OVER (...)`), letting you skip `NULL`s when looking for the last non-null value — PostgreSQL and MySQL require manual workarounds (e.g., combining with `COALESCE` and a secondary window function) to achieve the same effect.

---

## 5.10 Best Practices

- **Always specify an explicit frame clause** for aggregate/value window functions when using `ORDER BY`, to avoid the default `RANGE ... CURRENT ROW` surprise (Section 5.7).
- **Add a tie-breaking column to ORDER BY** inside ranking functions whenever exact, reproducible ordering matters (e.g., `ORDER BY salary DESC, emp_id ASC`), since `ROW_NUMBER()` on ties is otherwise non-deterministic.
- **Use `DENSE_RANK()` for "Nth distinct value" business logic**, and `ROW_NUMBER()` for "exactly N rows" logic (e.g., pagination, strict Top-N regardless of ties).
- **Name windows** (`WINDOW w AS (PARTITION BY dept ORDER BY salary)`) when the same window definition is reused across multiple functions in one query, for readability and to avoid repetition errors.
- **Filter on window function results using an outer query or CTE**, never directly in `WHERE` (window functions cannot be used in `WHERE`/`HAVING` directly — see Section 5.11).

---

## 5.11 Common Mistakes & Edge Cases

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using a window function directly in `WHERE`/`HAVING` | Window functions are evaluated **after** `WHERE`/`GROUP BY`/`HAVING` in logical SQL execution order (recall Module 1, Section 1.11) — they literally cannot be referenced there | Wrap in a subquery/CTE and filter in the outer query |
| Assuming `LAST_VALUE()` gives the true last row without an explicit frame | Default frame is `RANGE ... CURRENT ROW`, so it returns the current row itself | Add `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| Forgetting `ORDER BY` inside `OVER()` for ranking functions | Undefined/arbitrary rank assignment, or an error in strict engines | Always include a deterministic `ORDER BY` for ranking functions |
| Using `ROW_NUMBER()` when business logic actually means "distinct rank" | Ties get artificially split into different numbers, causing incorrect "Nth highest" results | Use `DENSE_RANK()` for "Nth distinct value" semantics |
| Confusing the outer query's `ORDER BY` with the one inside `OVER()` | They serve entirely different purposes (final display order vs window computation order) | Keep them conceptually and explicitly separate; they can differ |

---

## 5.12 Performance Notes

- Window functions typically require the engine to **sort the data by `PARTITION BY` + `ORDER BY`** (unless an index already provides that order) — this sort is often the dominant cost, similar in nature to a `GROUP BY`'s sort/hash step.
- Multiple window functions in the same query that share the **same `PARTITION BY`/`ORDER BY`** are often computed in a **single sort pass** by the optimizer — grouping compatible window specifications together (or using named windows) can help the planner recognize this opportunity.
- Window functions are generally **more efficient than the equivalent correlated subquery or self-join** for "value alongside aggregate" and "top-N per group" patterns, because they avoid the O(n×m) repeated-subquery execution model (Module 3) — but this should still be verified with `EXPLAIN` on your specific engine/dataset.
- Very large partitions with wide frames (e.g., `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) may require significant memory for the sort/frame buffer — watch for spill-to-disk behavior on huge datasets.

---

## 5.13 Interview Questions & MCQs

### Conceptual
1. What is the fundamental difference between a window function and `GROUP BY`?
2. Explain the difference between `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()` with an example involving ties.
3. Why can't window functions be used directly in a `WHERE` clause?
4. What is the default window frame when `ORDER BY` is present inside `OVER()`, and why does this default cause `LAST_VALUE()` to behave unexpectedly?
5. Explain the difference between `ROWS`, `RANGE`, and `GROUPS` frame modes.

### Applied (Very High-Frequency FAANG/SDE-2/3 Patterns)
6. Write a query to find the **2nd highest salary** in each department using a window function.
7. Write a query to compute a **running total of daily sales**, and explain what would go wrong if two days accidentally had identical dates (a data quality scenario) with the default `RANGE` frame.
8. Write a query using `LAG()` to compute the **day-over-day percentage change** in sales.
9. Write a query using `LAST_VALUE()` correctly (with the proper frame clause) to find, for each employee, the **lowest salary in their department**.
10. Rewrite Module 3's `LATERAL` "top-3-per-department" query using a window function instead, and discuss when you would prefer one approach over the other.

### MCQs

**Q1.** Which ranking function leaves NO gaps in the ranking sequence after ties?
 a) ROW_NUMBER()  b) RANK()  c) DENSE_RANK()  d) NTILE()

**Q2.** The default window frame when `ORDER BY` is specified inside `OVER()` is:
 a) `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`
 b) `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
 c) `GROUPS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`
 d) No default; frame clause is mandatory

**Q3.** Window functions can be referenced directly in:
 a) `WHERE`  b) `HAVING`  c) `SELECT` and `ORDER BY`  d) `GROUP BY`

**Q4.** `LAG(amount, 1)` looks:
 a) Forward 1 row  b) Backward 1 row  c) At the current row  d) At the first row of the partition

**Q5.** Which frame mode treats all rows tied on the `ORDER BY` value as a single unit?
 a) ROWS  b) RANGE  c) GROUPS (counts them, but as one unit per count) — both b and c relate to ties, but which is "all tied rows in or out together, with no partial split"?  d) None

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **c) DENSE_RANK()**
2. **b) `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`**
3. **c) `SELECT` and `ORDER BY`** (also technically usable inside a derived table/CTE that's later filtered, but never directly in `WHERE`/`HAVING`/`GROUP BY` of the same query level)
4. **b) Backward 1 row**
5. **b) RANGE** — RANGE always includes/excludes an entire tied peer group together as one indivisible unit relative to the current row's value.

</details>

---

## 5.14 Cheat Sheet — Module 5

```
┌───────────────────────────────────────────────────────────────────┐
│                  WINDOW FUNCTIONS — QUICK REFERENCE                   │
├───────────────────────────┬───────────────────────────────────────┤
│ OVER (PARTITION BY ... ORDER BY ...)  │ Defines the window            │
│ Rows preserved?                        │ YES — unlike GROUP BY (collapses)│
│ ROW_NUMBER()                            │ Unique, sequential — no gaps, no ties│
│ RANK()                                   │ Ties share rank; gap after tie   │
│ DENSE_RANK()                              │ Ties share rank; NO gap after   │
│ NTILE(n)                                   │ Split into n roughly-equal buckets│
│ SUM/AVG/COUNT/MIN/MAX OVER(...)             │ Aggregate without collapsing rows│
│ LAG(expr, n)                                 │ Look backward n rows           │
│ LEAD(expr, n)                                 │ Look forward n rows           │
│ FIRST_VALUE(expr)                              │ First row's value in the frame│
│ LAST_VALUE(expr)                                │ ⚠️ Needs explicit full-frame   │
│ NTH_VALUE(expr, n)                               │ Nth row's value in the frame  │
│ Default frame (w/ ORDER BY)                       │ RANGE UNBOUNDED PRECEDING    │
│                                                     │ TO CURRENT ROW (NOT full!)  │
│ Default frame (no ORDER BY)                          │ Entire partition            │
│ ROWS                                                    │ Physical row count          │
│ RANGE                                                     │ Logical value equality (ties merge)│
│ GROUPS                                                      │ Counts distinct tied peer groups│
│ Window fn in WHERE/HAVING?                                    │ ❌ Not allowed — filter outside│
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 5.15 Exercises

### Beginner
1. Write a query using `ROW_NUMBER()` to number employees within each department by salary descending.
2. Write a query using an aggregate window function to show each employee's salary alongside their department's total salary.

### Intermediate
3. Write a query demonstrating the difference between `RANK()` and `DENSE_RANK()` on data with at least one tie.
4. Write a query using `LAG()` and `LEAD()` to show each day's sales alongside the previous and next day's sales.

### Advanced
5. Write a query to find the **3rd highest distinct salary** per department using `DENSE_RANK()`.
6. Write a query using `LAST_VALUE()` correctly (with an explicit frame) to find the lowest-paid employee's name in each department, and explain why the frame clause is necessary.
7. Demonstrate, with a concrete example including tied `ORDER BY` values, the difference between a running total computed with `ROWS` vs the default `RANGE` frame.

### Output Prediction
8. Given 4 rows with salaries `[70000, 70000, 65000, 60000]` ordered descending, predict the exact `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()` output for all 4 rows.

### Debugging
9. A query uses `LAST_VALUE(salary) OVER (ORDER BY salary)` expecting the maximum salary in every row, but gets each row's own salary instead. Diagnose and fix.

### Scenario-Based
10. You're asked to compute each customer's **percentage of total company revenue**. Write the query using a window function, and explain why `GROUP BY` alone cannot achieve this in a single query without an additional join.

### Mini Project
11. Using a `Transactions(customer_id, transaction_date, amount)` table, write: (a) a running total of each customer's spending over time, (b) each customer's rank by total spending, (c) month-over-month percentage change using `LAG()`, (d) a 3-transaction moving average using an explicit `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` frame.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [SQLBolt — Interactive SQL Lessons](https://sqlbolt.com/)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

- **Window functions** compute a value "alongside" each row without collapsing rows — the single defining contrast with `GROUP BY`.
- **Ranking functions** (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`) require `ORDER BY` inside `OVER()`; memorize the tie-handling differences between the first three.
- **Aggregate window functions** (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) with `OVER()` compute running or partition-wide aggregates.
- **Value functions** (`LEAD`, `LAG`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`) enable row-to-row comparisons and frame-relative lookups — beware `LAST_VALUE`'s default-frame gotcha.
- **Window frames** (`ROWS`/`RANGE`/`GROUPS`) define exactly which rows participate in the calculation; the **default frame with `ORDER BY` is `RANGE ... CURRENT ROW`**, not the whole partition — this single fact resolves most window-function bugs.
- Window functions **cannot** be used in `WHERE`/`HAVING`/`GROUP BY` directly (recall the logical SQL execution order from Module 1) — filter their results in an outer query or CTE.

---

## Cross-References for Later Parts / Modules
- **GROUPING SETS, ROLLUP, CUBE (advanced aggregation beyond simple GROUP BY)** → Module 6 (next)
- **Materialized Views for precomputing expensive window-function reports** → Module 8
- **Query optimizer's sort-reuse across multiple window specifications** → Part 8 (Query Execution Internals)
- **Indexing to support ORDER BY / PARTITION BY sort avoidance** → Part 8

---

> ✅ **Module 5 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 6"** to proceed to **Module 6: Advanced Aggregation** — `GROUPING SETS`, `ROLLUP`, `CUBE`, the `GROUPING()` function, the `FILTER` clause, and `HAVING` internals.
