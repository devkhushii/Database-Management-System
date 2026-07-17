# Part 3 — The Complete SQL Handbook
## Module 8: Basic Queries

> **Goal of this module:** Master the `SELECT` statement's foundational clauses — `SELECT`, `FROM`, `DISTINCT`, `WHERE`, `ORDER BY`, `LIMIT`/`OFFSET`/`FETCH`/`TOP`, and aliasing. This is the beating heart of SQL — the clause you'll write more than any other, for the rest of your career.

---

### 📍 Where This Module Fits

```
Module 7: Data now exists  →  Module 8: Reading it back (You are here)  →  Module 9: Operators to filter/compute with
```

---

## 8.1 The SELECT Statement — Overview

```sql
SELECT column1, column2
FROM table_name
WHERE condition
ORDER BY column1
LIMIT 10;
```

This is the statement you will write **more than any other SQL statement combined** throughout your career — every dashboard, every API response, every report ultimately boils down to `SELECT`.

> ⚠️ **Foreshadowing Module 15:** You will notice the clauses are *written* in the order `SELECT → FROM → WHERE → ORDER BY → LIMIT`, but they do **not** *execute* in that order internally. We'll fully formalize this in **Module 15 (SQL Execution Order)** — for now, treat this module's clause-by-clause walkthrough as "the order you write them," and hold the question "but what order do they actually run in?" in the back of your mind.

---

## 8.2 SELECT and Column Projection

```sql
SELECT name, salary FROM employees;    -- specific columns
SELECT * FROM employees;               -- all columns
```

### Why `SELECT *` is Considered Bad Practice in Production Code

> ⚠️ **Frequently-tested best practice, worth understanding deeply, not just memorizing:**

| Reason | Explanation |
|---|---|
| **Fragility** | If a new column is added later (Module 4's `ALTER TABLE`), every `SELECT *`-based query/application silently starts returning extra data — potentially breaking assumptions in application code, exposing sensitive columns unintentionally, or bloating response payloads |
| **Performance** | Fetching columns you don't need wastes I/O and network bandwidth — especially costly for wide tables or large `TEXT`/`JSON` columns |
| **Index-only scan prevention** | The query optimizer (**Module 1**) can sometimes answer a query **entirely from an index**, never touching the underlying table row at all (a fast "index-only scan," fully explained in Part 4) — but only if the query doesn't request columns outside what the index covers. `SELECT *` defeats this optimization by definition, since it demands every column. |
| **Readability** | Explicit column lists document exactly what a query depends on, making code review and refactoring safer |

```
SELECT *                          SELECT name, salary
FROM employees;                   FROM employees;
   │                                  │
   ▼                                  ▼
Must read EVERY column          Can potentially satisfy this
from disk for every row,        query using ONLY an index on
even ones you'll never use      (name, salary) — never touching
                                 the full table row at all
```

> 💡 **When `SELECT *` is genuinely fine:** Quick, ad-hoc, interactive exploration during development ("what does this table even contain?") — the concern is specifically about `SELECT *` embedded in **production application code or reports** that run repeatedly and are expected to be stable over time.

---

## 8.3 FROM

```sql
SELECT * FROM employees;
```

`FROM` specifies the **source** of the rows being queried — a table, a view (Module 14), a subquery (Module 13), or (from Module 12 onward) the result of a `JOIN` between multiple tables.

### Table Aliasing (Preview — Full Coverage in Section 8.8)

```sql
SELECT e.name FROM employees AS e;
SELECT e.name FROM employees e;      -- AS is optional in both PostgreSQL and MySQL
```

---

## 8.4 DISTINCT

```sql
SELECT DISTINCT department FROM employees;
```

**What it does:** Removes duplicate rows from the result set, based on the **entire selected row** (all listed columns considered together, not just the first one).

```sql
SELECT DISTINCT department, status FROM employees;
-- Removes rows only where BOTH department AND status match another row exactly
```

### Internal Working

To eliminate duplicates, the engine must effectively **compare every row against every other row** — typically implemented via either a **sort** (sort rows, then scan for adjacent duplicates — cheap to detect once sorted) or a **hash-based** approach (build a hash set of seen rows, check membership as you go). Either way, `DISTINCT` inherently requires examining the **entire result set** before it can return anything — it cannot stream results back incrementally the way an unfiltered `SELECT` can.

> ⚠️ **Performance implication, frequently tested:** `DISTINCT` is **not free** — on large result sets, it can be a genuinely expensive operation (sorting or hashing potentially millions of rows). A common, real anti-pattern is slapping `DISTINCT` onto a query to "fix" unexpectedly duplicated results **without understanding *why* the duplicates appeared in the first place** — usually the true root cause is a join producing a Cartesian-product-like fan-out (a mistake formally covered in **Module 17**), and `DISTINCT` merely papers over the symptom while potentially hiding a real logical bug and adding unnecessary performance overhead.

### DISTINCT ON (PostgreSQL-Only Extension)

```sql
-- Get the highest-paid employee per department
SELECT DISTINCT ON (department) department, name, salary
FROM employees
ORDER BY department, salary DESC;
```

`DISTINCT ON (expression)` keeps only the **first row per group** of the specified expression(s), based on the `ORDER BY` — an extremely useful, genuinely powerful PostgreSQL-specific feature with **no direct MySQL equivalent** (MySQL users typically achieve the same result using window functions, covered in Part 4, or correlated subqueries, covered in Module 13).

---

## 8.5 WHERE

```sql
SELECT * FROM employees WHERE department = 'Engineering';
SELECT * FROM employees WHERE salary > 80000 AND status = 'active';
```

**What it does:** Filters rows **before** they're included in the result — the primary mechanism for narrowing down which rows a query considers.

### WHERE vs Filtering in Application Code

> 💡 **A foundational best practice, worth internalizing early:** Always filter data **in SQL** (`WHERE`) rather than fetching everything and filtering in application code afterward. Filtering in the database lets the query optimizer potentially use an **index** to jump directly to matching rows (Part 4) rather than transferring every row over the network only to discard most of them in your application — a difference that can be the gap between a query taking milliseconds versus seconds (or worse) at scale.

We'll cover every operator usable inside `WHERE` (`=`, `<`, `BETWEEN`, `IN`, `LIKE`, `IS NULL`, etc.) exhaustively in **Module 9** — this module focuses on `WHERE`'s *role* in the query structure, not its full operator vocabulary.

---

## 8.6 ORDER BY

```sql
SELECT name, salary FROM employees ORDER BY salary DESC;
SELECT name, salary FROM employees ORDER BY department ASC, salary DESC;   -- multi-column sort
```

- `ASC` (ascending — the default if omitted) or `DESC` (descending).
- Multiple columns: sorts by the first column, then breaks ties using the second column, and so on — exactly like sorting a spreadsheet by multiple columns.

### Sorting by Column Position (Legacy, Discouraged)

```sql
SELECT name, salary FROM employees ORDER BY 2 DESC;   -- sorts by the 2nd selected column (salary)
```

> ⚠️ **Best practice:** Avoid ordinal (position-based) `ORDER BY` in production code — like unqualified `INSERT` (**Module 7, Section 7.2**), it silently breaks if the `SELECT` list is ever reordered. Always order by explicit column name.

### NULLS Ordering

```sql
SELECT * FROM employees ORDER BY department NULLS LAST;    -- PostgreSQL — explicit control
-- MySQL has no NULLS FIRST/LAST syntax; NULLs sort as the lowest value by default
--   (i.e., they appear first in ASC order), and the common workaround is:
SELECT * FROM employees ORDER BY (department IS NULL), department;
```

> ⚠️ **Real, commonly-tested PostgreSQL vs MySQL difference:** By ANSI SQL standard, where `NULL` sorts is **implementation-defined** — engines are free to choose. PostgreSQL defaults to sorting `NULL` as **larger than any value** (appearing last in `ASC` order) and provides explicit `NULLS FIRST`/`NULLS LAST` syntax to override this. MySQL sorts `NULL` as **smaller than any value** (appearing first in `ASC` order) and has no direct `NULLS FIRST/LAST` syntax, requiring the boolean-expression workaround shown above. This is a genuine, real source of subtly different query results when porting queries between engines.

---

## 8.7 LIMIT, OFFSET, FETCH, TOP

These clauses restrict **how many rows** are returned — essential for pagination, "top N" reports, and preventing accidentally returning millions of rows to a client.

### PostgreSQL & MySQL: LIMIT / OFFSET

```sql
SELECT * FROM employees ORDER BY salary DESC LIMIT 10;              -- top 10 highest earners
SELECT * FROM employees ORDER BY salary DESC LIMIT 10 OFFSET 20;    -- rows 21–30 ("page 3" of size 10)
```

### The ANSI Standard Way: FETCH (SQL:2008)

```sql
SELECT * FROM employees
ORDER BY salary DESC
OFFSET 20 ROWS
FETCH NEXT 10 ROWS ONLY;
```

**Both PostgreSQL and MySQL (8.0+) support the standard `FETCH` syntax**, though `LIMIT`/`OFFSET` remains far more commonly used in practice in both ecosystems due to its brevity and long-standing familiarity.

### SQL Server's TOP (Mentioned for Completeness)

```sql
SELECT TOP 10 * FROM employees ORDER BY salary DESC;   -- SQL Server (T-SQL) only
```

`TOP` is a **SQL Server-specific** (T-SQL) construct, functionally similar to `LIMIT` — included here so you recognize it if you encounter SQL Server code, even though it's outside this handbook's PostgreSQL/MySQL primary focus.

### Comparison Table

| Syntax | PostgreSQL | MySQL | SQL Server | Standard? |
|---|---|---|---|---|
| `LIMIT n OFFSET m` | ✅ | ✅ | ❌ | Not ANSI standard (widely-adopted convention) |
| `OFFSET m ROWS FETCH NEXT n ROWS ONLY` | ✅ | ✅ (8.0+) | ✅ | **ANSI SQL:2008 standard** |
| `TOP n` | ❌ | ❌ | ✅ | Not ANSI standard |

### Internal Working / Performance Implications

`LIMIT`/`FETCH` alone is generally cheap — the engine can often stop processing as soon as it has enough rows (especially when combined with an index that already provides the required `ORDER BY` order, letting it stop scanning early rather than sorting the entire table first).

**`OFFSET`, however, is a genuine, frequently-tested performance trap:**

> ⚠️ **Critical performance point for pagination at scale:** `OFFSET m` does **not** let the engine "jump directly" to row `m` — it must still **generate and then discard** the first `m` rows before returning the next batch. `LIMIT 10 OFFSET 1000000` requires computing (or at least counting through) over a million rows internally before returning your 10 — meaning **later pages get progressively, linearly slower**, a real, commonly-hit production issue with naive "page number" based pagination on large tables.

```
LIMIT 10 OFFSET 0        → fast (return first 10, stop early)
LIMIT 10 OFFSET 100000   → slow (must traverse/discard 100,000 rows first)
```

**The standard industry fix: keyset (cursor-based) pagination**, which uses a `WHERE` condition on the last-seen sort key instead of `OFFSET`:

```sql
-- Instead of: LIMIT 10 OFFSET 100000 (slow at scale)
-- Use keyset pagination (fast at any depth):
SELECT * FROM employees
WHERE id > 458291    -- the last ID seen on the previous page
ORDER BY id
LIMIT 10;
```

> 💡 **This is a genuinely strong senior-level talking point:** Being able to explain *why* `OFFSET`-based pagination degrades at scale, and proposing keyset/cursor pagination as the fix, is a common, real system design discussion point in backend interviews — not just a database trivia fact.

---

## 8.8 Aliases

### Column Aliases

```sql
SELECT name AS employee_name, salary AS annual_salary FROM employees;
SELECT name employee_name, salary annual_salary FROM employees;   -- AS is optional
```

**Why they exist:** Rename output columns for clarity in results, especially important for **computed/expression columns** which otherwise get an ugly, engine-generated default name:

```sql
SELECT salary * 12 AS monthly_salary_equivalent FROM employees;
-- Without the alias: the column would be named something unhelpful
-- like "?column?" (PostgreSQL) or "salary * 12" (MySQL)
```

### Table Aliases

```sql
SELECT e.name, e.salary
FROM employees AS e
WHERE e.salary > 50000;
```

**Why they exist:** Shorten verbose table names for readability, and become **essential** (not just convenient) once you start joining multiple tables (**Module 12**) — without aliases, disambiguating which table a column comes from when two tables share a column name becomes impossible.

### A Genuinely Important Caveat: Column Aliases Cannot Be Used in WHERE

```sql
-- This FAILS in both PostgreSQL and MySQL:
SELECT salary * 12 AS annual_total
FROM employees
WHERE annual_total > 1000000;    -- ERROR: column "annual_total" does not exist

-- Must repeat the expression instead:
SELECT salary * 12 AS annual_total
FROM employees
WHERE salary * 12 > 1000000;
```

**Why this happens:** This connects directly to **Module 15's execution order** — `WHERE` is logically evaluated *before* `SELECT` (and therefore before its aliases even exist yet) in the standard logical processing sequence, even though `SELECT` is *written* first. We'll fully explain this apparent contradiction in Module 15; for now, just know it as a very real, very commonly-hit practical rule: **aliases defined in `SELECT` are not visible to `WHERE`** (though, as a genuine and frequently-surprising exception, most engines — including both PostgreSQL and MySQL — **do** allow `SELECT`-defined aliases to be reused in `ORDER BY` and `GROUP BY`, since those clauses execute *after* `SELECT` logically).

```sql
-- This WORKS (ORDER BY can use SELECT aliases):
SELECT salary * 12 AS annual_total
FROM employees
ORDER BY annual_total DESC;
```

---

## 8.9 Full Worked Example

```sql
SELECT
    e.name AS employee_name,
    e.department,
    e.salary * 12 AS annual_salary
FROM employees AS e
WHERE e.status = 'active'
  AND e.salary > 50000
ORDER BY annual_salary DESC
LIMIT 5;
```

**Reading this query aloud (the "written order" narrative, before Module 15 reveals the real execution order):**
1. Look at the `employees` table, calling it `e` for short.
2. Keep only active employees earning over 50,000.
3. For each remaining row, compute their name, department, and annual salary (monthly × 12).
4. Sort those results by annual salary, highest first.
5. Return only the top 5.

---

## 8.10 Advantages / Best Practices Recap

| Practice | Why |
|---|---|
| Avoid `SELECT *` in production code | Fragility, performance, defeats index-only scans (Section 8.2) |
| Filter with `WHERE`, not application code | Lets the optimizer use indexes; reduces network transfer (Section 8.5) |
| Use explicit column names in `ORDER BY`, not ordinal positions | Resilient to `SELECT` list reordering (Section 8.6) |
| Prefer keyset pagination over deep `OFFSET` for large datasets | Avoids linear slowdown on later pages (Section 8.7) |
| Alias computed expressions | Produces meaningful, readable column names (Section 8.8) |

## 8.11 Common Mistakes

- Using `SELECT *` habitually even in application code that will run repeatedly in production.
- Assuming a `SELECT`-defined alias is usable in the same query's `WHERE` clause (Section 8.8).
- Using deep `OFFSET`-based pagination on large, growing tables without realizing the performance cliff (Section 8.7).
- Reaching for `DISTINCT` to "fix" duplicate rows without diagnosing the actual root cause, often a join issue (Section 8.4, fully explored in Module 17).
- Assuming `NULL` sorting behavior is identical across PostgreSQL and MySQL (Section 8.6).

## 8.12 Edge Cases

- `LIMIT 0` is valid and simply returns zero rows — sometimes deliberately used to cheaply check a query's column structure/validity without transferring any actual data.
- `DISTINCT` combined with `ORDER BY` on a column *not* in the `SELECT` list has engine-specific rules about whether it's even allowed — PostgreSQL requires `ORDER BY` expressions to appear in the `SELECT` list when `DISTINCT` is used; this is a genuine, sometimes-surprising restriction worth testing directly.
- `OFFSET` with a value larger than the total row count simply returns an empty result set — not an error, in both engines.

## 8.13 Performance Implications

This entire module is deeply performance-relevant: `SELECT *` vs explicit columns, `WHERE`-side filtering vs application-side filtering, and `OFFSET` vs keyset pagination are all genuinely significant, real-world performance decisions that show up constantly in production systems and in interview system-design discussions — not merely academic syntax choices.

## 8.14 Industry Usage

Every API endpoint that returns a list, every dashboard, every report — virtually all of them are built on exactly the clauses in this module: filtered (`WHERE`), sorted (`ORDER BY`), and paginated (`LIMIT`/`OFFSET` or keyset) `SELECT` queries. Mastery of this module alone already covers a substantial fraction of day-to-day, real production SQL usage.

## 8.15 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Basic `SELECT`/`WHERE`/`ORDER BY` syntax |
| Placement | "Why is `SELECT *` considered bad practice?" |
| Mid-level | "How would you implement pagination for a large table?" |
| Senior SDE-2/SDE-3 | "Explain why `OFFSET`-based pagination degrades at scale and how you'd redesign it with keyset pagination." / "Why can't you reference a `SELECT` alias in the same query's `WHERE` clause?" |

---

## 8.16 Chapter Summary

- `SELECT`/`FROM`/`WHERE`/`ORDER BY`/`LIMIT` form the backbone of virtually all data retrieval — mastery here underlies everything in the rest of the handbook.
- `SELECT *` is convenient for exploration but a genuine anti-pattern in production code (fragility, performance, defeats index-only scans).
- `DISTINCT` isn't free — it requires sorting/hashing the entire result set, and is often misused to mask an underlying join-fanout bug.
- `NULL` sorting order genuinely differs between PostgreSQL (`NULL` last by default) and MySQL (`NULL` first by default).
- `OFFSET`-based pagination degrades linearly with depth — **keyset pagination** is the standard, senior-level-expected fix for large tables.
- `SELECT`-defined column aliases are invisible to `WHERE` (evaluated earlier, per Module 15) but visible to `ORDER BY`/`GROUP BY` (evaluated later).

---

## 8.17 Revision Notes / Cheat Sheet

```sql
SELECT col1, col2 AS alias1 FROM table_name AS t
WHERE condition
ORDER BY col1 [ASC|DESC], col2 [ASC|DESC]
LIMIT n [OFFSET m];

-- ANSI standard alternative to LIMIT/OFFSET (both engines support it):
OFFSET m ROWS FETCH NEXT n ROWS ONLY;

-- PostgreSQL-only:
SELECT DISTINCT ON (col) ... ORDER BY col, tiebreaker;
ORDER BY col NULLS FIRST | NULLS LAST;

⚠ SELECT * → avoid in production code
⚠ Deep OFFSET → slow at scale; prefer keyset pagination (WHERE id > last_seen_id)
⚠ SELECT aliases NOT usable in WHERE (but ARE usable in ORDER BY / GROUP BY)
```

---

## 8.18 Multiple Choice Questions (MCQs)

1. Why is `SELECT *` discouraged in production application code?
   a) It's slower to type b) **Fragility to schema changes, wasted I/O, and defeats index-only scans** c) It's not valid SQL d) It only works with one table

2. What does `DISTINCT` require internally?
   a) Nothing extra b) **Sorting or hashing the entire result set to detect duplicates** c) An index always d) A `JOIN`

3. Why does deep `OFFSET`-based pagination get slower on later pages?
   a) The database caches earlier pages b) **The engine must generate and discard all preceding rows first** c) It's a bug in most engines d) It doesn't get slower

4. Can a `SELECT`-defined column alias be used in that same query's `WHERE` clause?
   a) Yes, always b) **No — `WHERE` is logically evaluated before `SELECT`** c) Only in MySQL d) Only in PostgreSQL

5. By default, how does PostgreSQL sort `NULL` values in `ASC` order?
   a) First b) **Last** c) Randomly d) It errors

---

## 8.19 Exercises

### 🟢 Beginner
1. Write a query selecting `name` and `salary` from `employees`, sorted by salary descending.
2. Write a query returning only the distinct `department` values from `employees`.
3. Write a query returning the 5 highest-paid employees using `LIMIT`.

### 🟡 Intermediate
4. Write a query using `LIMIT`/`OFFSET` to return "page 3" of a 10-row-per-page employee listing, sorted by `id`.
5. Rewrite the same pagination using the ANSI-standard `OFFSET ... FETCH NEXT ... ROWS ONLY` syntax.
6. Write a query aliasing a computed column (e.g., `salary * 12 AS annual_salary`) and sort the results by that alias.

### 🔴 Advanced / Debugging
7. Attempt to filter using a `SELECT`-defined alias inside `WHERE`, observe the error, and rewrite the query correctly.
8. Explain why `LIMIT 20 OFFSET 500000` on a 10-million-row table can be significantly slower than `LIMIT 20 OFFSET 0`, and rewrite it using keyset pagination assuming rows are ordered by `id`.

### 🧩 Scenario-Based
9. You're building an infinite-scroll social media feed that must remain fast even after a user scrolls through thousands of posts. Would you use `OFFSET`-based or keyset-based pagination? Justify your answer using Section 8.7.

### 🏗️ Mini Project
10. Using your practice schema (Sakila/Pagila from Module 2, or your own mini-schema), write five distinct `SELECT` queries demonstrating: explicit column selection (no `SELECT *`), a `WHERE` filter, a multi-column `ORDER BY`, `DISTINCT`, and keyset-style pagination.

### 🔗 Practice Links
- [SQLBolt — SELECT queries](https://sqlbolt.com/)
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [DataLemur](https://datalemur.com/)

---

## ✅ Module 8 Complete

You now have a firm command of `SELECT`'s foundational clauses, along with the performance and correctness nuances (index-only scans, `OFFSET` degradation, alias visibility) that separate a merely-working query from a genuinely well-engineered one.

**Next up: Module 9 — SQL Operators**, where we fill out `WHERE`'s full vocabulary: comparison, logical, `BETWEEN`, `IN`, `EXISTS`, `LIKE`/`ILIKE`, `REGEXP`, and — critically — how `NULL` really behaves.

---

*Reply "continue" or "next module" when you're ready for Module 9.*
