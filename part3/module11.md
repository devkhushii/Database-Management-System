# Part 3 — The Complete SQL Handbook
## Module 11: Aggregate Functions

> **Goal of this module:** Move from per-row functions (Module 10) to functions that **collapse many rows into one summary value** — `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `STDDEV`, `VARIANCE` — plus `GROUP BY`, `HAVING`, and multi-dimensional grouping with `ROLLUP`, `CUBE`, and `GROUPING SETS`. This module is the foundation of virtually all SQL-based reporting and analytics.

---

### 📍 Where This Module Fits

```
Module 10: Per-row transformations  →  Module 11: Many-rows-to-one summaries (You are here)  →  Module 12: Joins
```

---

## 11.1 What Is an Aggregate Function?

An **aggregate function** takes **multiple rows** as input and produces a **single value** as output — fundamentally different from the scalar functions in Module 10, which operate independently on each row.

```sql
SELECT COUNT(*) FROM employees;         -- one number, summarizing the whole table
SELECT AVG(salary) FROM employees;      -- one number, the average across all rows
```

> 🧠 **Real-life analogy:** If a scalar function (Module 10) is like asking "what's this one person's age?", an aggregate function is like asking "what's the *average* age of everyone in this room?" — it necessarily requires looking at the whole group at once, not just one individual.

---

## 11.2 COUNT, SUM, AVG, MIN, MAX

```sql
SELECT
    COUNT(*)          AS total_employees,
    COUNT(department)  AS employees_with_department,
    SUM(salary)        AS total_payroll,
    AVG(salary)        AS average_salary,
    MIN(salary)        AS lowest_salary,
    MAX(salary)        AS highest_salary
FROM employees;
```

### COUNT(*) vs COUNT(column) vs COUNT(DISTINCT column) — A Critical, Frequently-Tested Distinction

| Form | Counts |
|---|---|
| `COUNT(*)` | **Every row**, regardless of `NULL`s in any column |
| `COUNT(column)` | Only rows where that **specific column is non-`NULL`** |
| `COUNT(DISTINCT column)` | Only **distinct non-`NULL`** values of that column |

```sql
-- Table: 5 employees total, 2 have department = NULL, and among the 3 with a department,
-- 2 share the same department name

SELECT COUNT(*) FROM employees;                       -- 5
SELECT COUNT(department) FROM employees;               -- 3 (excludes the 2 NULLs)
SELECT COUNT(DISTINCT department) FROM employees;      -- 2 (2 of the 3 share a department)
```

> ⚠️ **This exact distinction is one of the most frequently-asked SQL interview questions industry-wide** — "what's the difference between `COUNT(*)` and `COUNT(column_name)`?" Being able to explain it precisely, with `NULL`-awareness, is a strong, easy signal in an interview.

### Aggregate Functions Ignore NULLs (Directly Connecting to Module 9, Section 9.9)

```sql
SELECT AVG(salary) FROM employees;
-- If 2 out of 10 rows have salary = NULL, the average is computed
-- over the remaining 8 non-NULL values, NOT treating NULL as 0
```

> ⚠️ **Extremely commonly misunderstood behavior:** `AVG`, `SUM`, `MIN`, `MAX`, and `COUNT(column)` all **silently exclude `NULL` values** from their computation entirely — a `NULL` salary is treated as "this row simply doesn't contribute a data point," not as zero. This is mathematically consistent with `NULL`'s "unknown" meaning (Module 9) but is a genuinely common source of confusion for beginners who expect `NULL` to behave like `0` in an average.

```sql
-- If you actually WANT NULLs treated as zero for averaging purposes:
SELECT AVG(COALESCE(salary, 0)) FROM employees;   -- explicitly using Module 10's COALESCE
```

### An Important Edge Case: Aggregates Over Zero Rows

```sql
SELECT COUNT(*) FROM employees WHERE department = 'NonexistentDept';   -- returns 0
SELECT SUM(salary) FROM employees WHERE department = 'NonexistentDept'; -- returns NULL, NOT 0!
SELECT AVG(salary) FROM employees WHERE department = 'NonexistentDept'; -- returns NULL
```

> ⚠️ **A genuinely surprising, real, frequently-tested edge case:** `COUNT(*)` over zero matching rows correctly returns `0` (you *can* count "zero things"). But `SUM`, `AVG`, `MIN`, and `MAX` over zero rows all return **`NULL`**, not `0` — because there's genuinely no data to sum/average/compare, and "the sum of nothing" is treated as unknown/inapplicable rather than defined as zero. Application code that assumes `SUM(...)` always returns a number (and, say, tries to do arithmetic on it directly without `COALESCE`) can crash or misbehave when the filtered set happens to be empty.

---

## 11.3 STDDEV and VARIANCE

```sql
SELECT STDDEV(salary) AS salary_stddev, VARIANCE(salary) AS salary_variance FROM employees;
```

**What they measure:** **Variance** quantifies how spread out a set of values is from their average (the average of the squared differences from the mean); **standard deviation** is simply the square root of variance, expressed in the same units as the original data (making it more directly interpretable — "salaries typically vary by about ₹12,000 from the average," for instance).

### PostgreSQL vs MySQL — Naming Nuances

| Concept | PostgreSQL | MySQL |
|---|---|---|
| Population standard deviation | `STDDEV_POP()` | `STDDEV_POP()` / `STD()` |
| Sample standard deviation | `STDDEV_SAMP()` / `STDDEV()` (alias for sample) | `STDDEV_SAMP()` |
| Population variance | `VAR_POP()` | `VAR_POP()` |
| Sample variance | `VAR_SAMP()` / `VARIANCE()` (alias for sample) | `VAR_SAMP()` / `VARIANCE()` |

> 💡 **Population vs. sample — a genuine statistical distinction worth knowing:** *Population* statistics assume your data represents the **entire** group of interest; *sample* statistics apply a correction (dividing by `n-1` instead of `n`) appropriate when your data is a **subset** used to estimate a larger population's true characteristics. In everyday reporting queries, this distinction is often glossed over, but it's a legitimate, occasionally-tested point of statistical rigor in senior/data-focused interviews.

> 💡 **Industry usage:** These statistical aggregates are used far more heavily in analytics/data-science-adjacent SQL work (e.g., identifying outliers, understanding data distribution for A/B test analysis) than in typical day-to-day transactional application queries — still worth knowing, but genuinely less universally used than `COUNT`/`SUM`/`AVG`.

---

## 11.4 GROUP BY

### The Problem It Solves

Plain aggregate functions (Section 11.2) summarize the **entire table** into one row. `GROUP BY` lets you compute **separate aggregates for each distinct group** of values — "average salary **per department**," not just "average salary overall."

```sql
SELECT department, AVG(salary) AS avg_salary, COUNT(*) AS headcount
FROM employees
GROUP BY department;
```

```
Before GROUP BY (raw rows):              After GROUP BY department:
┌────┬─────────────┬────────┐            ┌─────────────┬────────────┬───────────┐
│ id │ department  │ salary │            │ department  │ avg_salary │ headcount │
├────┼─────────────┼────────┤            ├─────────────┼────────────┼───────────┤
│ 1  │ Engineering │ 90000  │            │ Engineering │   95000    │     2     │
│ 2  │ Engineering │ 100000 │    ──►     │ Sales       │   55000    │     1     │
│ 3  │ Sales       │ 55000  │            └─────────────┴────────────┴───────────┘
└────┴─────────────┴────────┘
```

### The Golden Rule of GROUP BY

> ⚠️ **The single most important, most frequently-violated rule in this entire module:** Every column in the `SELECT` list must be **either** (a) listed in `GROUP BY`, **or** (b) wrapped inside an aggregate function. Any column that's neither will cause an error (or, dangerously, silently ambiguous/arbitrary results in certain non-strict configurations — see the MySQL note below).

```sql
-- ❌ INVALID in PostgreSQL (and in MySQL under strict/ONLY_FULL_GROUP_BY mode):
SELECT department, name, AVG(salary) FROM employees GROUP BY department;
-- ERROR: column "employees.name" must appear in the GROUP BY clause or be used in an aggregate function
-- (WHY: for each department group, there are potentially MANY different "name" values —
--  which single name should represent the whole group? The engine refuses to guess.)

-- ✅ VALID — every non-aggregated column is in GROUP BY:
SELECT department, AVG(salary) FROM employees GROUP BY department;
```

> ⚠️ **A genuinely dangerous, real MySQL-specific historical gotcha:** Older MySQL versions (and MySQL today, if the `ONLY_FULL_GROUP_BY` SQL mode is explicitly disabled — it is **enabled by default** since MySQL 5.7.5) used to **silently allow** invalid `GROUP BY` queries like the one above, arbitrarily picking **some** value for the ungrouped, non-aggregated column (typically from whichever row happened to be processed first internally) — producing a query that runs without error but returns **logically meaningless, non-deterministic-feeling results**. This caused real, subtle production bugs for years. **Modern MySQL (5.7.5+) enforces the same strict rule as PostgreSQL by default** — but if you ever encounter an older or non-default-configured MySQL instance behaving strangely with `GROUP BY`, this is very likely why, and it's a genuinely strong, real interview/debugging story to know.

### Grouping by Multiple Columns

```sql
SELECT department, status, COUNT(*) AS headcount
FROM employees
GROUP BY department, status;
```

Groups are formed by the **unique combination** of all `GROUP BY` columns together — one output row per distinct `(department, status)` pair that actually occurs in the data.

---

## 11.5 HAVING

### The Problem It Solves

`WHERE` filters **individual rows** *before* grouping happens; `HAVING` filters **groups** *after* aggregation has already been computed — you cannot use `WHERE` to filter on an aggregate result, because at the point `WHERE` logically executes, the aggregates don't exist yet (a concept fully formalized in **Module 15**).

```sql
-- ❌ INVALID — cannot reference an aggregate in WHERE:
SELECT department, AVG(salary) AS avg_salary
FROM employees
WHERE AVG(salary) > 70000    -- ERROR: aggregate functions are not allowed in WHERE
GROUP BY department;

-- ✅ VALID — use HAVING to filter on the aggregated result:
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 70000;
```

```
Full pipeline:
raw rows → WHERE (filters individual rows) → GROUP BY (forms groups, computes aggregates)
        → HAVING (filters GROUPS based on aggregate results) → SELECT (final output)
```

> 🧠 **Real-life analogy:** Imagine sorting students into classrooms (`GROUP BY`), then computing each classroom's average test score (aggregate). `WHERE` is like disqualifying individual students *before* they're even placed into a classroom (e.g., "ignore absent students"); `HAVING` is like disqualifying entire *classrooms* afterward based on their calculated average (e.g., "only show classrooms whose average score exceeds 80").

### Combining WHERE and HAVING in One Query

```sql
SELECT department, AVG(salary) AS avg_salary
FROM employees
WHERE status = 'active'              -- filter rows first (before grouping)
GROUP BY department
HAVING AVG(salary) > 70000            -- then filter the resulting groups
ORDER BY avg_salary DESC;
```

> ⚠️ **Common mistake:** Using `HAVING` for a condition that could (and should, for performance) be expressed in `WHERE` instead. `HAVING department = 'Engineering'` technically works, but it wastefully computes aggregates for **every** department first, then discards all but one group — whereas `WHERE department = 'Engineering'` filters rows down **before** any aggregation work happens, which is strictly more efficient. **Rule of thumb: use `WHERE` for anything not involving an aggregate function; reserve `HAVING` exclusively for conditions that genuinely require an already-computed aggregate value.**

---

## 11.6 ROLLUP

```sql
-- PostgreSQL
SELECT department, status, COUNT(*) AS headcount
FROM employees
GROUP BY ROLLUP (department, status);

-- MySQL
SELECT department, status, COUNT(*) AS headcount
FROM employees
GROUP BY department, status WITH ROLLUP;
```

**What it does:** Produces the normal `GROUP BY` result **plus** automatic **subtotal rows** at each level of the hierarchy, plus one **grand total** row — extremely useful for reports that need "totals per department, totals per department+status, AND an overall grand total," all in a single query rather than three separate ones unioned together.

```
GROUP BY ROLLUP (department, status) produces:

department    | status  | headcount
--------------+---------+----------
Engineering   | active  |     8      ← normal group
Engineering   | on_leave|     1      ← normal group
Engineering   | NULL    |     9      ← subtotal for ALL of Engineering (status collapsed)
Sales         | active  |     4      ← normal group
Sales         | NULL    |     4      ← subtotal for ALL of Sales
NULL          | NULL    |    13      ← GRAND TOTAL (everything collapsed)
```

> ⚠️ **Important, practical note:** The `NULL` values appearing in `ROLLUP`'s subtotal/grand-total rows represent **"all values collapsed together,"** not an actual missing/unknown data value — this can be genuinely ambiguous if your data *also* has real `NULL`s in the grouped columns, and PostgreSQL provides a `GROUPING()` function (Section 11.8) specifically to disambiguate "is this `NULL` a real data value, or a rollup-generated subtotal marker?"

---

## 11.7 CUBE

```sql
-- PostgreSQL
SELECT department, status, COUNT(*) AS headcount
FROM employees
GROUP BY CUBE (department, status);

-- MySQL: no native CUBE support — must be emulated via UNION ALL of multiple GROUP BY queries
```

**What it does:** Generates subtotals for **every possible combination** of the grouped columns, not just the hierarchical progression `ROLLUP` produces. For two columns `(department, status)`, `CUBE` produces: department+status combinations, department-only subtotals, status-only subtotals, **and** the grand total — a strict superset of what `ROLLUP` generates.

```
GROUP BY CUBE (department, status) additionally includes rows like:

department    | status  | headcount
--------------+---------+----------
NULL          | active  |    12      ← subtotal for ALL "active" status, across every department
                                        (ROLLUP would NOT produce this row — only CUBE does)
```

> ⚠️ **Real, significant PostgreSQL vs MySQL feature gap:** `CUBE` (like `MERGE` in **Module 7**) is **not natively supported in MySQL at all** — MySQL only supports `WITH ROLLUP`. Achieving `CUBE`-equivalent output in MySQL requires manually constructing and `UNION ALL`-ing several separate `GROUP BY` queries covering each desired combination — considerably more verbose and error-prone than PostgreSQL's single-statement `CUBE`.

---

## 11.8 GROUPING SETS (Overview)

```sql
SELECT department, status, COUNT(*) AS headcount
FROM employees
GROUP BY GROUPING SETS (
    (department, status),
    (department),
    (status),
    ()
);
```

**What it does:** The most general, explicit form of multi-level grouping — you directly specify **exactly which combinations** of columns you want grouped/subtotaled, rather than relying on `ROLLUP`'s fixed hierarchical pattern or `CUBE`'s "every possible combination" behavior. In fact, `ROLLUP(department, status)` and `CUBE(department, status)` are both technically expressible as specific, particular `GROUPING SETS` configurations — `GROUPING SETS` is the true general-purpose primitive underlying both.

**The `GROUPING()` function** helps disambiguate real `NULL` data values from rollup/cube/grouping-sets-generated "this dimension was collapsed" markers (mentioned in Section 11.6):

```sql
SELECT
    department,
    status,
    GROUPING(department) AS dept_is_subtotal,   -- 1 if this row is a department-level subtotal, 0 otherwise
    COUNT(*) AS headcount
FROM employees
GROUP BY ROLLUP (department, status);
```

> 💡 **Industry usage note:** `GROUPING SETS`/`ROLLUP`/`CUBE` are genuinely powerful for **data warehousing and BI reporting** use cases (generating multi-level pivot-table-style summaries in one query), but are used far less frequently in typical day-to-day transactional application backend code than plain `GROUP BY`/`HAVING` — worth knowing conceptually and recognizing when you see it, but not something you'll write as often as the fundamentals earlier in this module. This is intentionally covered here as an overview; a deeper treatment fits naturally alongside window functions in Part 4.

---

## 11.9 Full Worked Example — Aggregate Functions Combined

```sql
SELECT
    department,
    COUNT(*)                                    AS headcount,
    COUNT(DISTINCT status)                       AS distinct_statuses,
    ROUND(AVG(salary), 2)                        AS avg_salary,
    MIN(salary)                                  AS min_salary,
    MAX(salary)                                  AS max_salary,
    ROUND(STDDEV(salary), 2)                     AS salary_stddev
FROM employees
WHERE hired_on >= '2020-01-01'
GROUP BY department
HAVING COUNT(*) >= 3
ORDER BY avg_salary DESC;
```

**Reading this query:** Filter to employees hired since 2020 (`WHERE`, row-level) → group the remaining rows by department (`GROUP BY`) → compute several aggregates per group → keep only departments with at least 3 qualifying employees (`HAVING`, group-level) → sort the final summarized results by average salary.

---

## 11.10 Advantages / Best Practices Recap

| Practice | Why |
|---|---|
| Use `WHERE` for row-level filters, `HAVING` only for aggregate-level filters | More efficient — filters rows before the (more expensive) grouping/aggregation step (Section 11.5) |
| Always know whether you want `COUNT(*)`, `COUNT(col)`, or `COUNT(DISTINCT col)` | These produce genuinely different, easily-confused numbers (Section 11.2) |
| Guard against `SUM`/`AVG` returning `NULL` on empty result sets | Wrap with `COALESCE(SUM(...), 0)` when a numeric default is needed downstream (Section 11.2) |
| Verify `ONLY_FULL_GROUP_BY` behavior when working with legacy/misconfigured MySQL instances | Avoids silently non-deterministic `GROUP BY` results (Section 11.4) |
| Reach for `ROLLUP`/`CUBE`/`GROUPING SETS` for multi-level reporting instead of multiple separate queries | Single-statement, atomic, often more efficient than manually `UNION`-ing several `GROUP BY` queries (Sections 11.6–11.8) |

## 11.11 Common Mistakes

- Confusing `COUNT(*)`, `COUNT(column)`, and `COUNT(DISTINCT column)` (Section 11.2).
- Assuming `SUM`/`AVG` return `0` (not `NULL`) when no rows match (Section 11.2).
- Selecting a non-aggregated, non-grouped column alongside `GROUP BY` — invalid in strict-mode engines, dangerously silent in legacy/misconfigured MySQL (Section 11.4).
- Trying to filter on an aggregate using `WHERE` instead of `HAVING` (Section 11.5).
- Using `HAVING` for simple row-level filters that belong in `WHERE`, wasting aggregation effort (Section 11.5).

## 11.12 Edge Cases

- `COUNT(*)` genuinely differs from `COUNT(1)` **only in theory** — both are optimized identically by every modern PostgreSQL and MySQL query planner; the old folk belief that `COUNT(1)` is faster than `COUNT(*)` is a persistent, widely-repeated **myth** in modern engines, worth explicitly debunking if it comes up.
- `GROUP BY` on a column containing `NULL` values treats all `NULL`s as belonging to **one single group** together (unlike the "two unknowns aren't equal" logic from **Module 9** that applies to `=` comparisons) — a deliberate, standards-defined special case specifically for grouping purposes.
- Aggregate functions can be combined with `CASE` (**Module 10, Section 10.7**) for **conditional aggregation** — computing multiple different conditional summaries in a single pass, a genuinely powerful, frequently-used real pattern:

```sql
SELECT
    department,
    COUNT(CASE WHEN status = 'active' THEN 1 END) AS active_count,
    COUNT(CASE WHEN status = 'on_leave' THEN 1 END) AS on_leave_count
FROM employees
GROUP BY department;
```

## 11.13 Performance Implications

- `GROUP BY` typically requires either a **sort** or a **hash-based grouping** strategy internally (conceptually similar to `DISTINCT`'s internal mechanics from **Module 8, Section 8.4**) — genuinely expensive on very large, unindexed datasets; an index on the `GROUP BY` column(s) can allow the optimizer to avoid a full sort in some execution plans (fully covered in Part 4).
- Filtering with `WHERE` before `GROUP BY` (rather than `HAVING` after) reduces the number of rows that must be sorted/hashed/aggregated in the first place — a genuinely significant, real performance difference on large tables (Section 11.5).
- `COUNT(DISTINCT column)` is meaningfully more expensive than plain `COUNT(column)`, since it requires deduplication work similar to `DISTINCT` itself.

## 11.14 Industry Usage

`GROUP BY`/aggregate queries are the literal backbone of virtually all business intelligence, analytics dashboards, and reporting systems — "revenue by month," "signups by region," "average order value by customer segment" are all direct, real-world instances of exactly the patterns covered in this module, executed constantly across every industry that uses relational databases.

## 11.15 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | `GROUP BY`/`HAVING` syntax; aggregate function definitions |
| Placement | "What's the difference between `WHERE` and `HAVING`?" (one of the most classic SQL interview questions) |
| Mid-level | "What's the difference between `COUNT(*)` and `COUNT(column)`?" / "Why might `SUM()` return `NULL` instead of `0`?" |
| Senior SDE-2/SDE-3 | "Explain the legacy `ONLY_FULL_GROUP_BY` MySQL gotcha and why it mattered." / "Design a single query producing subtotals and a grand total using `ROLLUP`." |

---

## 11.16 Chapter Summary

- Aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `STDDEV`, `VARIANCE`) collapse multiple rows into one summary value, and **silently ignore `NULL`s** in their computation.
- `COUNT(*)` (all rows), `COUNT(column)` (non-`NULL` values), and `COUNT(DISTINCT column)` (distinct non-`NULL` values) are three genuinely different, commonly-confused counts.
- `SUM`/`AVG`/`MIN`/`MAX` return `NULL` (not `0`) over an empty/zero-row result set — a real, frequently-tested edge case.
- `GROUP BY` requires every non-aggregated `SELECT` column to also appear in the `GROUP BY` list — strictly enforced in PostgreSQL and modern MySQL (`ONLY_FULL_GROUP_BY`), though legacy MySQL configurations dangerously allowed silent violations.
- `WHERE` filters rows before grouping; `HAVING` filters groups after aggregation — using the wrong one for the wrong purpose is both a correctness and a performance mistake.
- `ROLLUP`, `CUBE`, and `GROUPING SETS` generate multi-level subtotals/grand totals in a single query — `CUBE` and `GROUPING SETS` are PostgreSQL-only among the two engines this handbook focuses on; MySQL supports only `ROLLUP`.

---

## 11.17 Revision Notes / Cheat Sheet

```sql
COUNT(*)              -- all rows
COUNT(col)             -- non-NULL values of col
COUNT(DISTINCT col)    -- distinct non-NULL values of col
SUM/AVG/MIN/MAX(col)   -- ignore NULLs; return NULL (not 0) over zero matching rows

SELECT grp_col, AGG(col) FROM t
WHERE row_level_condition          -- filters BEFORE grouping
GROUP BY grp_col
HAVING agg_level_condition         -- filters AFTER grouping/aggregation
ORDER BY ...;

GROUP BY ROLLUP (a, b)             -- hierarchical subtotals + grand total (both engines, MySQL: "WITH ROLLUP")
GROUP BY CUBE (a, b)               -- every combination of subtotals (PostgreSQL only)
GROUP BY GROUPING SETS ((a,b),(a),())  -- explicit, general form (PostgreSQL only)

GROUPING(col)   -- disambiguates real NULL vs rollup/cube-generated subtotal marker

⚠ Every non-aggregated SELECT column MUST be in GROUP BY
⚠ WHERE cannot reference aggregate results — use HAVING
⚠ COUNT(1) is NOT faster than COUNT(*) — this is a myth in modern engines
```

---

## 11.18 Multiple Choice Questions (MCQs)

1. What does `SUM(salary)` return when zero rows match the `WHERE` clause?
   a) 0 b) **NULL** c) An error d) -1

2. Which clause filters rows BEFORE grouping occurs?
   a) `HAVING` b) **`WHERE`** c) `GROUP BY` itself d) `ORDER BY`

3. `COUNT(DISTINCT department)` counts:
   a) All rows b) Non-NULL department values, including duplicates c) **Distinct, non-NULL department values** d) Only NULL values

4. Which of these is NOT natively supported in MySQL?
   a) `GROUP BY ... WITH ROLLUP` b) **`CUBE`** c) `HAVING` d) `COUNT(DISTINCT ...)`

5. Historically, what did legacy (or non-strict) MySQL do with an invalid `GROUP BY` query selecting a non-aggregated, non-grouped column?
   a) Always threw an error b) **Silently returned an arbitrary value for that column** c) Returned NULL for that column always d) Refused to execute entirely

---

## 11.19 Exercises

### 🟢 Beginner
1. Write a query counting the total number of employees.
2. Write a query showing average salary per department using `GROUP BY`.
3. Write a query showing only departments with more than 5 employees, using `HAVING`.

### 🟡 Intermediate
4. Write a query demonstrating the difference between `COUNT(*)`, `COUNT(department)`, and `COUNT(DISTINCT department)` on the same table (include at least one row with a `NULL` department to make the difference visible).
5. Write a query combining `WHERE` (row-level filter) and `HAVING` (group-level filter) in the same statement.
6. Write a conditional aggregation query using `CASE` inside `COUNT()` to show active vs. inactive employee counts per department in a single query.

### 🔴 Advanced / Debugging
7. A query `SELECT department, name, AVG(salary) FROM employees GROUP BY department;` fails in PostgreSQL but "worked" on an old MySQL server. Explain both behaviors and correct the query properly.
8. Explain, with a concrete example, why `SUM(salary)` returning `NULL` (instead of `0`) for an empty filtered result set could cause a bug in application code that does further arithmetic on the result — and show the `COALESCE`-based fix.

### 🧩 Scenario-Based
9. You need a single report showing headcount by department, headcount by status, and total headcount — all in one query. Which SQL feature from this module solves this most elegantly, and how would MySQL's more limited support change your approach?

### 🏗️ Mini Project
10. Using your practice schema (Sakila/Pagila), write a multi-part report query producing: total row count, a `GROUP BY` breakdown by a relevant category column, a `HAVING`-filtered subset of that breakdown, and (in PostgreSQL) a `ROLLUP` version showing subtotals and a grand total.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [DataLemur — Aggregate Function Questions](https://datalemur.com/)
- [StrataScratch](https://www.stratascratch.com/)

---

## ✅ Module 11 Complete

You can now summarize and report on data at any level of granularity — from simple totals to multi-dimensional `ROLLUP`/`CUBE`-based breakdowns — and you understand the `WHERE`-vs-`HAVING` and `NULL`-handling subtleties that are among the most frequently tested topics in SQL interviews at every level.

**Next up: Module 12 — Joins (Complete)**, where you'll finally combine data across multiple tables: `INNER`, `LEFT`, `RIGHT`, `FULL OUTER`, `CROSS`, `SELF` joins, and the `USING` clause.

---

*Reply "continue" or "next module" when you're ready for Module 12.*
