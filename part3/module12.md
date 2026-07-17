# Part 3 — The Complete SQL Handbook
## Module 12: Joins (Complete)

> **Goal of this module:** Master every join type — `INNER`, `LEFT`, `RIGHT`, `FULL OUTER`, `CROSS`, `SELF`, `NATURAL` — plus the `USING` clause, with execution diagrams and a clear mental model for how the engine actually combines rows across tables. Joins are the mechanism that makes the relational model (Part 2) and normalization actually *usable* in practice.

---

### 📍 Where This Module Fits

```
Module 11: Summarizing one table  →  Module 12: Combining multiple tables (You are here)  →  Module 13: Subqueries
```

---

## 12.1 Why Joins Exist

Recall **Part 2's normalization**: a well-designed relational schema deliberately **splits related data across multiple tables** to eliminate redundancy (e.g., `employees` and `departments` are separate tables, rather than repeating each department's full details on every employee row). But applications almost always need that data **recombined** for meaningful queries — "show me each employee **along with** their department's name." A **join** is the SQL mechanism that reconstitutes this related data back into a single result set, on demand, at query time.

> 🧠 **Real-life analogy:** Think of two separate spreadsheets — one listing employees with just a `department_id` number, another listing departments with their `id` and full `name`. A join is like using a **VLOOKUP**-style matching operation to combine them side-by-side into one combined view, based on matching IDs — except SQL joins are far more general and powerful than a simple lookup.

### Sample Tables Used Throughout This Module

```sql
employees                          departments
┌────┬────────┬───────────────┐    ┌────┬─────────────┐
│ id │ name   │ department_id │    │ id │ name        │
├────┼────────┼───────────────┤    ├────┼─────────────┤
│ 1  │ Asha   │ 10            │    │ 10 │ Engineering │
│ 2  │ Ravi   │ 10            │    │ 20 │ Sales       │
│ 3  │ Meera  │ 20            │    │ 30 │ HR          │
│ 4  │ Kabir  │ NULL          │    └────┴─────────────┘
└────┴────────┴───────────────┘
```

Note deliberately: `Kabir` (id=4) has **no department** (`NULL`), and `HR` (id=30) has **no employees** — these two "unmatched" rows will be crucial for illustrating how each join type behaves differently.

---

## 12.2 INNER JOIN

```sql
SELECT e.name, d.name AS department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
```

**What it does:** Returns only rows where a match is found **on both sides** — any employee with no matching department (or `NULL` department_id), and any department with no matching employees, are **excluded entirely**.

```
Result:
┌────────┬──────────────────┐
│ name   │ department_name  │
├────────┼──────────────────┤
│ Asha   │ Engineering      │
│ Ravi   │ Engineering      │
│ Meera  │ Sales            │
└────────┴──────────────────┘
(Kabir excluded — no matching department. HR excluded — no matching employees.)
```

```
Venn diagram intuition:

  employees          departments
  ┌─────────┐        ┌─────────┐
  │  Kabir  │████████│   HR    │
  │         │█ MATCH █│         │
  └─────────┘████████└─────────┘
              ▲
       INNER JOIN returns ONLY this overlapping region
```

`INNER` is technically optional — `JOIN` alone defaults to `INNER JOIN` in both PostgreSQL and MySQL — but writing `INNER JOIN` explicitly is widely considered better practice for readability, making the join type unambiguous at a glance.

---

## 12.3 LEFT JOIN (LEFT OUTER JOIN)

```sql
SELECT e.name, d.name AS department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
```

**What it does:** Returns **every row from the left table** (`employees`), regardless of whether a match exists on the right — unmatched left rows get `NULL` for every right-side column.

```
Result:
┌────────┬──────────────────┐
│ name   │ department_name  │
├────────┼──────────────────┤
│ Asha   │ Engineering      │
│ Ravi   │ Engineering      │
│ Meera  │ Sales            │
│ Kabir  │ NULL             │  ← kept, with NULL department (no match)
└────────┴──────────────────┘
(HR still excluded — LEFT JOIN doesn't care about unmatched RIGHT rows.)
```

```
Venn diagram intuition:

  employees          departments
  ┌─────────┐        ┌─────────┐
  │█████████│████████│         │
  │█ ALL of ██ MATCH █│         │
  │█ employ.█████████│         │
  └─────────┘        └─────────┘
       ▲
LEFT JOIN returns ALL of the left circle (matched + unmatched), plus the overlap
```

### Finding Unmatched Rows — The "Anti-Join" Pattern

```sql
-- Find employees with NO department assigned
SELECT e.name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.id IS NULL;
```

> 💡 **This `LEFT JOIN ... WHERE right_table.key IS NULL` pattern is a genuinely important, extremely common idiom** — sometimes called an **anti-join** — used to find rows in one table that have **no corresponding match** in another (e.g., "customers who have never placed an order," "products that have never been sold"). It's a real, frequently-tested interview pattern worth memorizing directly.

---

## 12.4 RIGHT JOIN (RIGHT OUTER JOIN)

```sql
SELECT e.name, d.name AS department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;
```

**What it does:** The mirror image of `LEFT JOIN` — returns **every row from the right table** (`departments`), regardless of whether a match exists on the left.

```
Result:
┌────────┬──────────────────┐
│ name   │ department_name  │
├────────┼──────────────────┤
│ Asha   │ Engineering      │
│ Ravi   │ Engineering      │
│ Meera  │ Sales            │
│ NULL   │ HR               │  ← kept, with NULL employee (no match)
└────────┴──────────────────┘
(Kabir now excluded — RIGHT JOIN doesn't care about unmatched LEFT rows.)
```

> 💡 **Genuine, widely-shared industry style convention (not a hard rule):** `RIGHT JOIN` is used far less often in practice than `LEFT JOIN`, because **any `RIGHT JOIN` can be trivially rewritten as a `LEFT JOIN`** by simply swapping the order the tables are listed in `FROM`:

```sql
-- These two queries are logically identical:
SELECT e.name, d.name FROM employees e RIGHT JOIN departments d ON e.department_id = d.id;
SELECT e.name, d.name FROM departments d LEFT JOIN employees e ON e.department_id = d.id;
```

Many real style guides explicitly recommend **standardizing on `LEFT JOIN` only**, reordering tables as needed, purely for consistency and to reduce the mental overhead of tracking which "side" is which across a large codebase.

---

## 12.5 FULL OUTER JOIN

```sql
SELECT e.name, d.name AS department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.id;
```

**What it does:** Returns **everything** — all matched rows, **plus** all unmatched rows from **both** the left and right tables, with `NULL`s filling in wherever no match exists on the opposite side.

```
Result:
┌────────┬──────────────────┐
│ name   │ department_name  │
├────────┼──────────────────┤
│ Asha   │ Engineering      │
│ Ravi   │ Engineering      │
│ Meera  │ Sales            │
│ Kabir  │ NULL             │  ← unmatched left row kept
│ NULL   │ HR               │  ← unmatched right row kept
└────────┴──────────────────┘
```

```
Venn diagram intuition:

  employees          departments
  ┌─────────┐        ┌─────────┐
  │█████████│████████│█████████│
  │█ ALL of ██ MATCH ██ ALL of █│
  │█ employ.██████████  depts  █│
  └─────────┘        └─────────┘
       ▲                   ▲
FULL OUTER JOIN returns BOTH entire circles, overlapping region included once
```

> ⚠️ **Critical, genuinely important PostgreSQL vs MySQL feature gap:** **MySQL has no native `FULL OUTER JOIN` support at all** (a real, long-standing limitation). The standard, widely-used MySQL workaround is to **simulate it** using a `UNION` of a `LEFT JOIN` and a `RIGHT JOIN`:

```sql
-- MySQL workaround for FULL OUTER JOIN
SELECT e.name, d.name AS department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
UNION
SELECT e.name, d.name AS department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;
```

`UNION` (without `ALL`) automatically deduplicates, ensuring the doubly-matched rows (present in both the `LEFT JOIN` and `RIGHT JOIN` results) don't appear twice in the final combined output — this is a genuinely common, real, frequently-tested workaround pattern worth memorizing given how significant this feature gap is.

---

## 12.6 CROSS JOIN

```sql
SELECT e.name, d.name AS department_name
FROM employees e
CROSS JOIN departments d;
```

**What it does:** Returns the **Cartesian product** — every possible combination of rows from both tables, with **no matching condition at all**. If `employees` has 4 rows and `departments` has 3 rows, the result has exactly **4 × 3 = 12 rows**.

```
CROSS JOIN result size = |left table rows| × |right table rows|

employees (4 rows) × departments (3 rows) = 12 combined rows,
regardless of any logical relationship between them
```

> ⚠️ **This connects directly to Module 17's "common mistakes" coverage:** An **accidental** Cartesian product — forgetting the `ON`/`WHERE` join condition entirely, or writing `FROM a, b` (old-style comma-join syntax) without a matching `WHERE` clause — is one of the most classic, damaging real SQL bugs, silently multiplying row counts and producing nonsensical, duplicated results. `CROSS JOIN` written **explicitly and deliberately** is a legitimate, useful tool (e.g., generating every combination of "size" × "color" for a product catalog, or building a calendar of every date × every store location) — the danger is specifically an **unintentional** one from a missing condition, not the `CROSS JOIN` keyword itself.

---

## 12.7 SELF JOIN

**What it is:** Not a distinct join *type* — it's simply a regular `JOIN` (typically `INNER` or `LEFT`) where a table is joined **to itself**, requiring table aliases to distinguish the two "copies" being compared.

```sql
-- Find pairs of employees who work in the same department
SELECT e1.name AS employee1, e2.name AS employee2, e1.department_id
FROM employees e1
JOIN employees e2 ON e1.department_id = e2.department_id AND e1.id < e2.id;
```

> 💡 **Why `e1.id < e2.id`, not just `e1.id <> e2.id`?** Using `<>` would produce **both** `(Asha, Ravi)` and `(Ravi, Asha)` as separate rows — the same pair counted twice, in reverse order. Using `<` (strictly less than) ensures each unique pair appears **exactly once**, a genuinely important, commonly-tested trick worth internalizing for any "compare rows within the same table" query.

### Classic Self-Join Use Case: Hierarchical/Manager Relationships

```sql
CREATE TABLE employees (
    id         INT PRIMARY KEY,
    name       VARCHAR(100),
    manager_id INT REFERENCES employees(id)   -- references the SAME table (Module 6)
);

-- Show each employee alongside their manager's name
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;   -- LEFT JOIN keeps the top-level exec with no manager
```

> 🧠 **Real-life analogy:** A self-join is like comparing entries **within the same phone book** — "find every pair of people who share the same last name" requires scanning the phone book against itself, not against a different book.

---

## 12.8 NATURAL JOIN

```sql
SELECT e.name, d.name AS department_name
FROM employees e
NATURAL JOIN departments d;
```

**What it does:** Automatically joins tables on **all columns that share the same name** in both tables — no explicit `ON` condition needed at all. Supported by both PostgreSQL and MySQL.

> ⚠️ **Extremely strong, near-universal industry consensus: avoid `NATURAL JOIN` in real production code.** The automatic, implicit matching behavior is a genuine, real liability:

| Risk | Explanation |
|---|---|
| **Silent breakage on schema changes** | If someone later adds a new column with a matching name to *both* tables (e.g., both tables gain a `created_at` column), the `NATURAL JOIN`'s matching condition **silently changes** — potentially breaking the query's logic without any syntax error at all |
| **Non-obvious behavior** | A reader of the query cannot tell **which columns** are actually being used for matching without separately inspecting both tables' schemas — a serious readability and maintainability cost |
| **No control** | You cannot exclude a same-named column from the match condition without renaming columns |

> 💡 **Best practice: always use explicit `JOIN ... ON` (or `USING`, Section 12.9) instead of `NATURAL JOIN`** — this is one of the most consistently-agreed-upon SQL style recommendations across the industry, and being able to explain *why* `NATURAL JOIN` is risky is a genuinely good, real interview talking point.

---

## 12.9 The USING Clause

```sql
SELECT e.name, d.name AS department_name
FROM employees e
JOIN departments d USING (department_id);   -- only valid if BOTH tables' matching column has the SAME name
```

**What it does:** A more explicit, safer middle ground than `NATURAL JOIN` — you **explicitly name** which shared column(s) to join on, but avoid repeating the full `table.column = table.column` syntax of a standard `ON` clause. Requires the joining columns to have the **identical name** in both tables (unlike `ON`, which places no such requirement).

### A Genuine Behavioral Difference Between USING and ON

> 💡 **A subtle, real, occasionally-tested distinction:** With a standard `ON` clause, the joined column appears **twice** in a `SELECT *` result (once from each table, e.g., `employees.department_id` and `departments.department_id` as two separate output columns). With `USING`, the joined column is **automatically merged/de-duplicated** into a **single** output column in `SELECT *` — a small but genuine ergonomic difference that occasionally matters for downstream code consuming `SELECT *` results.

```sql
SELECT * FROM employees e JOIN departments d ON e.department_id = d.department_id;
-- Result includes department_id TWICE (once per table)

SELECT * FROM employees e JOIN departments d USING (department_id);
-- Result includes department_id ONCE (merged)
```

---

## 12.10 Comparison Table — All Join Types at a Glance

| Join Type | Returns | Unmatched Left Rows? | Unmatched Right Rows? | MySQL Support |
|---|---|---|---|---|
| `INNER JOIN` | Only matched rows | ❌ Excluded | ❌ Excluded | ✅ Full |
| `LEFT JOIN` | All left + matches | ✅ Kept (NULL right side) | ❌ Excluded | ✅ Full |
| `RIGHT JOIN` | All right + matches | ❌ Excluded | ✅ Kept (NULL left side) | ✅ Full |
| `FULL OUTER JOIN` | Everything | ✅ Kept | ✅ Kept | ❌ **Not supported** — emulate via `UNION` |
| `CROSS JOIN` | Every combination | N/A (no matching logic) | N/A | ✅ Full |
| `SELF JOIN` | Not a type — a table joined to itself | Depends on join type used | Depends on join type used | ✅ Full |
| `NATURAL JOIN` | Auto-matched on same-named columns | Depends on variant | Depends on variant | ✅ Full (but discouraged) |

---

## 12.11 Execution: How the Engine Actually Processes a Join

Connecting back to **Module 1's optimizer discussion**, a join isn't executed by some single universal "join algorithm" — the query optimizer chooses **among several physical join strategies** based on table sizes, available indexes, and cost estimates (a topic given full depth treatment in Part 4). A brief conceptual preview:

```
Nested Loop Join:                  Hash Join:                       Merge Join:
For each row in table A:           Build a hash table from           Sort both tables by the
  scan table B for matches         the smaller table (B);            join key, then merge them
  (efficient with a good index      probe it once per row of A       together in one linear pass
   on B's join column;              (efficient for larger,           (efficient when both inputs
   inefficient without one)         unsorted/unindexed data)          are already sorted, e.g.,
                                                                       from an index)
```

> 💡 This is intentionally a light preview — **Part 4** covers join algorithm selection, indexing strategy, and reading `EXPLAIN` output to see exactly which strategy the optimizer chose for a given query, in full depth.

---

## 12.12 Full Worked Example — Joins Combined

```sql
SELECT
    e.name                        AS employee_name,
    d.name                        AS department_name,
    m.name                        AS manager_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id     -- keep employees even without a department
LEFT JOIN employees m   ON e.manager_id = m.id          -- self-join, keep employees even without a manager
ORDER BY department_name, employee_name;
```

This single query combines a **regular table join** (`employees` to `departments`) and a **self-join** (`employees` to itself, for the manager relationship) — both as `LEFT JOIN`s, deliberately chosen so that employees with no department or no manager are still included in the results, rather than silently dropped.

---

## 12.13 Advantages / Best Practices Recap

| Practice | Why |
|---|---|
| Always use explicit `JOIN ... ON`, never `NATURAL JOIN` | Avoids silent breakage from future schema changes (Section 12.8) |
| Standardize on `LEFT JOIN`, avoid `RIGHT JOIN` | Reduces cognitive overhead; any `RIGHT JOIN` is trivially rewritable (Section 12.4) |
| Use the `LEFT JOIN ... WHERE right.key IS NULL` anti-join pattern for "find unmatched rows" queries | A clean, standard, efficient idiom (Section 12.3) |
| Be deliberate, not accidental, with `CROSS JOIN` | Distinguishes intentional Cartesian products from bugs (Section 12.6) |
| Remember MySQL has no `FULL OUTER JOIN` | Plan to use the `UNION` workaround when needed (Section 12.5) |

## 12.14 Common Mistakes

- Forgetting the join condition entirely (old comma-join syntax with no `WHERE`), accidentally producing a Cartesian product (Section 12.6).
- Using `NATURAL JOIN` in production code, risking silent breakage from schema evolution (Section 12.8).
- Using `<>` instead of `<` in self-joins comparing rows within the same table, producing duplicate reversed pairs (Section 12.7).
- Assuming `FULL OUTER JOIN` works in MySQL without checking (Section 12.5).
- Forgetting that `LEFT JOIN` only guarantees left-side completeness — a subsequent `WHERE` clause on a right-side column can accidentally turn it back into something behaving like an `INNER JOIN` if not written carefully (covered in depth as a common mistake in **Module 17**).

## 12.15 Edge Cases

- A `LEFT JOIN` combined with a `WHERE` condition on the right table's column (other than the specific `IS NULL` anti-join pattern) can accidentally filter out the very unmatched rows the `LEFT JOIN` was meant to preserve — because `WHERE` runs after the join and evaluates to `UNKNOWN`/`FALSE` for `NULL` right-side values (**directly connecting back to Module 9's three-valued logic**), silently discarding them. The fix is typically to move such conditions into the `ON` clause instead of `WHERE`, when the intent is genuinely to keep unmatched rows.
- Joining on columns with **mismatched data types** (e.g., an `INT` joined against a `VARCHAR`) can silently prevent index usage (implicit type coercion, previewed here, expanded in Part 4) — always verify joined columns share compatible, ideally identical, types.

## 12.16 Performance Implications

- Join performance is heavily dependent on whether the joined columns are **indexed** (fully explored in Part 4) — an unindexed join column on a large table forces the optimizer toward the more expensive nested-loop or hash-join strategies with significant scan costs.
- `CROSS JOIN`s and accidental Cartesian products scale **multiplicatively**, not additively — joining two 10,000-row tables without a proper condition produces 100,000,000 rows, a genuinely catastrophic, real performance/memory issue.
- `FULL OUTER JOIN` emulation via `UNION` (MySQL) effectively runs **two separate join operations** plus a deduplication pass — genuinely more expensive than PostgreSQL's native single-pass `FULL OUTER JOIN`.

## 12.17 Industry Usage

Joins are used in essentially **every non-trivial query** in production systems — displaying an order with its customer's name, a blog post with its author's details, a product with its category — any time related data lives in separate normalized tables (which, per Part 2's normalization principles, is the correct, standard way to design a relational schema), joins are the mechanism that brings it back together for real-world use.

## 12.18 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Definitions and syntax of each join type |
| Placement | "What's the difference between `INNER JOIN` and `LEFT JOIN`?" (one of the most universally-asked SQL interview questions) |
| Mid-level | "How would you find all customers who have never placed an order?" (the anti-join pattern) |
| Senior SDE-2/SDE-3 | "Why doesn't MySQL support `FULL OUTER JOIN`, and how would you emulate it?" / "Explain why `NATURAL JOIN` is considered risky in production schemas." |

---

## 12.19 Chapter Summary

- **`INNER JOIN`** returns only matched rows; **`LEFT`/`RIGHT JOIN`** additionally preserve unmatched rows from one side (filled with `NULL`); **`FULL OUTER JOIN`** preserves unmatched rows from both sides.
- **MySQL has no native `FULL OUTER JOIN`** — it must be emulated with a `UNION` of `LEFT JOIN` and `RIGHT JOIN`.
- **`CROSS JOIN`** produces a full Cartesian product — powerful when deliberate, dangerous when accidental.
- **`SELF JOIN`** is a regular join applied to a table and itself (via aliases), essential for hierarchical/comparative relationships.
- **`NATURAL JOIN`** auto-matches same-named columns and is widely discouraged in production due to silent-breakage risk; explicit `ON` (or `USING` for same-named columns) is the safer standard.
- The `LEFT JOIN ... WHERE right.key IS NULL` **anti-join pattern** is the standard idiom for "find rows with no match" queries.

---

## 12.20 Revision Notes / Cheat Sheet

```sql
INNER JOIN     -- only matches (both sides)
LEFT JOIN      -- all of left + matches
RIGHT JOIN     -- all of right + matches (rarely used — prefer LEFT + reorder tables)
FULL OUTER JOIN -- all of both sides (⚠ NOT in MySQL — emulate via UNION of LEFT + RIGHT)
CROSS JOIN     -- Cartesian product (rows_A × rows_B)
SELF JOIN      -- table joined to itself, via aliases
NATURAL JOIN   -- ⚠ avoid in production — implicit same-name-column matching
USING (col)    -- explicit shared-column join; merges duplicate output column (unlike ON)

-- Anti-join idiom: find unmatched rows
SELECT a.* FROM a LEFT JOIN b ON a.id = b.a_id WHERE b.a_id IS NULL;

-- Self-join pair dedup trick
JOIN t t2 ON t.id < t2.id   -- (not <>) avoids reversed-duplicate pairs

-- MySQL FULL OUTER JOIN emulation
SELECT ... FROM a LEFT JOIN b ...
UNION
SELECT ... FROM a RIGHT JOIN b ...;
```

---

## 12.21 Multiple Choice Questions (MCQs)

1. Which join returns only rows with matches on both sides?
   a) `LEFT JOIN` b) **`INNER JOIN`** c) `CROSS JOIN` d) `FULL OUTER JOIN`

2. Which join type does MySQL NOT natively support?
   a) `LEFT JOIN` b) `RIGHT JOIN` c) **`FULL OUTER JOIN`** d) `CROSS JOIN`

3. What's the size of the result of `CROSS JOIN` between a 5-row table and a 4-row table?
   a) 9 b) **20** c) 5 d) Depends on matching values

4. Why is `NATURAL JOIN` generally discouraged in production code?
   a) It's slower than other joins b) It's not supported in PostgreSQL c) **It silently changes behavior if matching-named columns are added later** d) It cannot be used with `WHERE`

5. In a self-join deduplicating pairs, why use `e1.id < e2.id` instead of `e1.id <> e2.id`?
   a) Performance only b) **Prevents each pair from appearing twice in reversed order** c) `<>` is invalid syntax in joins d) No real difference

---

## 12.22 Exercises

### 🟢 Beginner
1. Write an `INNER JOIN` between `employees` and `departments` showing each employee's department name.
2. Write a `LEFT JOIN` version of the same query, and identify which extra row(s) appear compared to the `INNER JOIN` version.
3. Write a `CROSS JOIN` between a small `sizes` table (`S`, `M`, `L`) and a `colors` table (`Red`, `Blue`) to generate every size/color combination.

### 🟡 Intermediate
4. Write the anti-join pattern to find all departments with zero employees.
5. Write a self-join query finding all pairs of employees in the same department, ensuring no reversed duplicate pairs appear.
6. Simulate a `FULL OUTER JOIN` in MySQL using `UNION` of a `LEFT JOIN` and a `RIGHT JOIN`, and verify it matches PostgreSQL's native `FULL OUTER JOIN` result on the same data.

### 🔴 Advanced / Debugging
7. A query uses `LEFT JOIN departments d ON e.department_id = d.id WHERE d.region = 'APAC'` intending to keep all employees but only show APAC department names when available — but instead it silently excludes every employee with no department. Explain why (referencing Section 12.15) and fix it.
8. Rewrite a `NATURAL JOIN` query as an explicit `JOIN ... ON` query, and explain a realistic future schema change that could have silently broken the `NATURAL JOIN` version.

### 🧩 Scenario-Based
9. You need a report listing every product, and for products that have never been ordered, show `0` instead of omitting them entirely. Which join type and which additional function (from Module 10/11) would you combine to achieve this?

### 🏗️ Mini Project
10. Using your practice schema (Sakila/Pagila) or your own mini-schema, write six queries — one for each core join type covered in this module (`INNER`, `LEFT`, `RIGHT`, `FULL OUTER`/emulated, `CROSS`, `SELF`) — and write a one-sentence explanation of what real-world question each query answers.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [SQLBolt — JOINs](https://sqlbolt.com/)
- [DataLemur](https://datalemur.com/)
- [StrataScratch](https://www.stratascratch.com/)

---

## ✅ Module 12 Complete

You can now confidently combine data across any number of related tables — including the anti-join pattern, self-joins for hierarchical data, and the genuinely important MySQL `FULL OUTER JOIN` gap — all backed by a clear mental (Venn-diagram) model of exactly which rows each join type preserves or discards.

**Next up: Module 13 — Subqueries**, covering scalar, row, table, correlated, and nested subqueries, plus `EXISTS`/`NOT EXISTS` revisited in full query-writing context.

---

*Reply "continue" or "next module" when you're ready for Module 13.*
