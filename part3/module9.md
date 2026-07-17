# Part 3 — The Complete SQL Handbook
## Module 9: SQL Operators

> **Goal of this module:** Fill out `WHERE`'s complete vocabulary — arithmetic, comparison, and logical operators, `BETWEEN`, `IN`/`NOT IN`, `EXISTS`, `ANY`/`ALL`, pattern matching (`LIKE`/`ILIKE`/`REGEXP`), and — critically — how `NULL` actually behaves in three-valued logic, a topic responsible for more subtle SQL bugs than almost anything else in this handbook.

---

### 📍 Where This Module Fits

```
Module 8: WHERE exists  →  Module 9: WHERE's full toolkit (You are here)  →  Module 10: Functions
```

---

## 9.1 Arithmetic Operators

| Operator | Meaning |
|---|---|
| `+` | Addition |
| `-` | Subtraction |
| `*` | Multiplication |
| `/` | Division |
| `%` (or `MOD`) | Modulo (remainder) |
| `^` | Exponentiation (PostgreSQL only, as an operator; MySQL uses `POWER()`) |

```sql
SELECT salary * 12 AS annual_salary FROM employees;
SELECT 17 % 5;    -- 2
```

### Integer Division — A Genuine, Frequently-Tested Gotcha

```sql
SELECT 7 / 2;     -- What does this return?
```

> ⚠️ **Critical, extremely commonly-tested behavior:** When **both operands are integers**, `/` performs **integer division** (truncating toward zero) in both PostgreSQL and MySQL — `7 / 2` returns `3`, **not** `3.5`. To get a fractional result, **at least one operand must be a floating-point/numeric type**:

```sql
SELECT 7 / 2;         -- 3   (integer division)
SELECT 7.0 / 2;        -- 3.5 (one operand is decimal)
SELECT 7 / 2.0;        -- 3.5
SELECT 7::NUMERIC / 2; -- 3.5 (explicit cast, PostgreSQL syntax)
SELECT CAST(7 AS DECIMAL) / 2;  -- 3.5 (ANSI standard cast syntax, works in both)
```

> 🧠 **Real-life analogy:** Integer division is like dividing a group of 7 people into teams of 2 and asking "how many *complete* teams can I form?" — the answer is 3 complete teams, with 1 person left over (the remainder, retrievable via `%`), not "3.5 teams." SQL's integer division mirrors this whole-number-only counting instinct unless you explicitly ask for a fractional answer.

---

## 9.2 Comparison Operators

| Operator | Meaning |
|---|---|
| `=` | Equal to |
| `<>` or `!=` | Not equal to (`<>` is the ANSI standard; `!=` is a widely-supported common extension) |
| `<`, `>`, `<=`, `>=` | Less than, greater than, and their inclusive variants |

```sql
SELECT * FROM employees WHERE salary >= 50000;
SELECT * FROM employees WHERE department <> 'Sales';
```

> 💡 **Best practice:** Prefer `<>` over `!=` in code intended to be maximally standards-portable, though in practice `!=` is universally supported by both PostgreSQL and MySQL and used constantly in the real world — this is a minor stylistic point, not a functional risk with these two engines specifically.

---

## 9.3 Logical Operators

| Operator | Meaning |
|---|---|
| `AND` | Both conditions must be true |
| `OR` | At least one condition must be true |
| `NOT` | Negates a condition |

```sql
SELECT * FROM employees
WHERE department = 'Engineering' AND salary > 80000;

SELECT * FROM employees
WHERE department = 'Sales' OR department = 'Marketing';

SELECT * FROM employees
WHERE NOT (status = 'terminated');
```

### Operator Precedence — A Real, Commonly-Hit Trap

```sql
-- Ambiguous intent without parentheses:
SELECT * FROM employees
WHERE department = 'Engineering' OR department = 'Sales' AND salary > 100000;
```

`AND` has **higher precedence** than `OR` (much like `*` binds tighter than `+` in arithmetic) — so the above is actually evaluated as:

```sql
WHERE department = 'Engineering' OR (department = 'Sales' AND salary > 100000)
```

...which likely is **not** what someone intended if they wanted "either department, but only if salary > 100000." The safe, unambiguous fix:

```sql
WHERE (department = 'Engineering' OR department = 'Sales') AND salary > 100000;
```

> ⚠️ **Best practice, genuinely important:** Always use **explicit parentheses** when mixing `AND` and `OR` in the same condition, regardless of whether you've memorized the precedence rules correctly — this is purely defensive, readability-first practice that prevents an entire class of logic bugs that are notoriously easy to introduce and hard to spot in code review.

---

## 9.4 BETWEEN

```sql
SELECT * FROM employees WHERE salary BETWEEN 50000 AND 80000;
-- Exactly equivalent to:
SELECT * FROM employees WHERE salary >= 50000 AND salary <= 80000;
```

> ⚠️ **Critical, frequently-tested fact: `BETWEEN` is always inclusive on both ends** (per the ANSI SQL standard, consistently implemented this way in both PostgreSQL and MySQL). There's no built-in exclusive variant — if you need an exclusive range, you must write it out manually with `>` / `<`.

`BETWEEN` also works with dates and other ordered types:

```sql
SELECT * FROM orders WHERE order_date BETWEEN '2026-01-01' AND '2026-01-31';
```

> ⚠️ **Genuine, real date-range gotcha:** Because `BETWEEN` is inclusive, `order_date BETWEEN '2026-01-01' AND '2026-01-31'` on a `TIMESTAMP`/`TIMESTAMPTZ` column will **miss** any rows timestamped later on January 31st (e.g., `2026-01-31 14:30:00`), since `'2026-01-31'` is implicitly interpreted as midnight (`2026-01-31 00:00:00`) at the start of that day. The safer, unambiguous pattern for date-range filtering on timestamp columns:

```sql
SELECT * FROM orders
WHERE order_date >= '2026-01-01' AND order_date < '2026-02-01';   -- half-open interval, no gaps
```

---

## 9.5 IN and NOT IN

```sql
SELECT * FROM employees WHERE department IN ('Engineering', 'Sales', 'Marketing');
-- Equivalent to, but far more readable than:
SELECT * FROM employees WHERE department = 'Engineering' OR department = 'Sales' OR department = 'Marketing';

SELECT * FROM employees WHERE department NOT IN ('Sales', 'Marketing');
```

`IN` can also be paired with a **subquery** (fully explored in **Module 13**):

```sql
SELECT * FROM employees WHERE department_id IN (SELECT id FROM departments WHERE region = 'APAC');
```

### The `NOT IN` + `NULL` Trap — One of SQL's Most Famous Gotchas

> ⚠️ **This is possibly THE single most infamous, most frequently-tested `NULL`-related gotcha in all of SQL — memorize this cold:**

```sql
SELECT * FROM employees WHERE department_id NOT IN (1, 2, NULL);
-- Returns ZERO rows. Always. No matter what department_id actually is.
```

**Why this happens (three-valued logic):** `NOT IN (1, 2, NULL)` is internally expanded to `<> 1 AND <> 2 AND <> NULL`. Any comparison against `NULL` (including `<>`) evaluates to **`UNKNOWN`**, not `TRUE` or `FALSE` (formalized fully in Section 9.9 below). Since the overall `AND` chain includes an `UNKNOWN` term, the **entire expression** becomes `UNKNOWN` for every row — and `WHERE` only keeps rows where the condition is definitively `TRUE`, so **every row is silently excluded**, with no error or warning at all.

```
department_id NOT IN (1, 2, NULL)
   = (department_id <> 1) AND (department_id <> 2) AND (department_id <> NULL)
                                                            └─────┬─────┘
                                                          always UNKNOWN
   → entire AND chain becomes UNKNOWN for every row → WHERE excludes all rows
```

This exact trap becomes especially dangerous when the `NOT IN` list comes from a **subquery** that might silently include a `NULL`:

```sql
-- DANGEROUS: if any department has a NULL region, this returns ZERO rows unexpectedly
SELECT * FROM employees
WHERE department_id NOT IN (SELECT id FROM departments WHERE region IS NULL OR region = 'APAC');
```

> 💡 **The safe fix — always prefer `NOT EXISTS` over `NOT IN` when a subquery result might contain `NULL`s** (Section 9.7 covers `EXISTS` in full):

```sql
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM departments d WHERE d.id = e.department_id AND d.region = 'APAC'
);
```

> ⚠️ **This single gotcha is genuinely one of the highest-value things to internalize in this entire handbook** — it has caused real, silent, hard-to-detect production bugs (queries that "just return nothing" with no error) at companies of every size, and is an extremely common senior-level interview question ("what's wrong with this query?").

---

## 9.6 ANY and ALL

```sql
SELECT * FROM employees WHERE salary > ANY (SELECT salary FROM employees WHERE department = 'Sales');
-- TRUE if salary exceeds AT LEAST ONE Sales employee's salary (effectively: exceeds the MINIMUM)

SELECT * FROM employees WHERE salary > ALL (SELECT salary FROM employees WHERE department = 'Sales');
-- TRUE only if salary exceeds EVERY Sales employee's salary (effectively: exceeds the MAXIMUM)
```

| Operator | Equivalent Intuition |
|---|---|
| `> ANY (...)` | Greater than the **minimum** of the set |
| `> ALL (...)` | Greater than the **maximum** of the set |
| `= ANY (...)` | Equivalent to `IN (...)` |
| `<> ALL (...)` | Equivalent to `NOT IN (...)` (and inherits the exact same `NULL` trap as Section 9.5 if the set contains `NULL`) |

> 💡 **Industry usage note:** `ANY`/`ALL` are part of the ANSI standard and fully supported by both PostgreSQL and MySQL, but in practice, most engineers reach for `IN`/`NOT IN`, `MAX()`/`MIN()` aggregates (Module 11), or `EXISTS` instead, since they're generally more immediately readable for equivalent logic — `ANY`/`ALL` show up more often in academic/exam contexts and in specific cases (like the "exceeds every value" pattern above) where they genuinely express the intent more directly than alternatives.

---

## 9.7 EXISTS and NOT EXISTS

```sql
SELECT * FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.id);
-- Departments that have at least one employee

SELECT * FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.id);
-- Departments with ZERO employees
```

**What it does:** Tests whether a subquery returns **any rows at all** — `TRUE` if at least one row is returned, `FALSE` if the subquery returns nothing. Critically, `EXISTS` **doesn't care what columns or values the subquery returns** (`SELECT 1` is a common convention, signaling "we only care about row existence, not the actual data") — this is why `EXISTS`/`NOT EXISTS` handle `NULL`s **correctly** and safely, unlike `NOT IN`'s trap in Section 9.5: `EXISTS` only ever evaluates to a clean `TRUE`/`FALSE`, never `UNKNOWN`.

### Internal Working — Why EXISTS Can Be Faster Than IN

`EXISTS` typically allows the query optimizer (Module 1) to use a **semi-join** strategy: for each outer row, it can often **stop as soon as it finds the first matching inner row**, rather than necessarily materializing the entire subquery's result set first (which is sometimes, though not always, how `IN` gets executed, depending on the optimizer's chosen plan). In practice, on modern PostgreSQL and MySQL query optimizers, `IN` and `EXISTS` are frequently rewritten into **logically equivalent execution plans** when there's no `NULL` ambiguity — but `EXISTS` remains the **strictly safer default** specifically because it sidesteps Section 9.5's `NULL` trap entirely, regardless of the optimizer's internal choices.

> 💡 **Best practice, worth memorizing as a rule of thumb:** When checking "does a related row exist" (or its negation), default to `EXISTS`/`NOT EXISTS` over `IN`/`NOT IN` with a subquery — it's both commonly at-least-as-fast **and** immune to the `NULL` trap.

---

## 9.8 Pattern Matching: LIKE, ILIKE, REGEXP

### LIKE

```sql
SELECT * FROM employees WHERE name LIKE 'A%';       -- starts with 'A'
SELECT * FROM employees WHERE name LIKE '%son';      -- ends with 'son'
SELECT * FROM employees WHERE name LIKE '%an%';      -- contains 'an' anywhere
SELECT * FROM employees WHERE name LIKE '_ohn';       -- exactly one character, then 'ohn' (matches 'John')
```

| Wildcard | Meaning |
|---|---|
| `%` | Zero or more of any character |
| `_` | Exactly one of any character |

### Case Sensitivity — A Real, Frequently-Tested Engine Difference

> ⚠️ **Genuinely important, commonly-tested difference:** `LIKE` is **case-sensitive by default in PostgreSQL** (`'Alice' LIKE 'alice'` → `FALSE`). In **MySQL, `LIKE`'s case sensitivity depends on the column's collation** — the very common default collations (e.g., `utf8mb4_general_ci`, where `_ci` stands for **case-insensitive**) make `LIKE` **case-insensitive by default** (`'Alice' LIKE 'alice'` → `TRUE`), while a case-sensitive collation (`_cs` or binary) would behave like PostgreSQL.

### ILIKE (PostgreSQL-Only)

```sql
SELECT * FROM employees WHERE name ILIKE 'alice';   -- explicit case-INSENSITIVE match, PostgreSQL only
```

`ILIKE` exists specifically to give PostgreSQL users an explicit, guaranteed case-insensitive match **without needing to change the column's collation** — a genuinely useful, PostgreSQL-specific convenience with no MySQL equivalent (since MySQL's default collations often already provide this behavior implicitly, as noted above). The common cross-engine-portable workaround in PostgreSQL, when `ILIKE` isn't available or for standards portability, is:

```sql
SELECT * FROM employees WHERE LOWER(name) LIKE LOWER('alice');
```

### REGEXP / Regular Expression Matching

```sql
-- PostgreSQL
SELECT * FROM employees WHERE name ~ '^A.*n$';       -- case-sensitive regex match
SELECT * FROM employees WHERE name ~* '^a.*n$';       -- case-INsensitive regex match

-- MySQL
SELECT * FROM employees WHERE name REGEXP '^A.*n$';
```

| Operator/Function | Engine | Meaning |
|---|---|---|
| `~` | PostgreSQL | Case-sensitive regex match |
| `~*` | PostgreSQL | Case-insensitive regex match |
| `!~` / `!~*` | PostgreSQL | Negated versions |
| `REGEXP` / `RLIKE` | MySQL | Regex match (case sensitivity follows column collation, same as `LIKE`) |

> 💡 **Performance note, previewed here, fully covered in Part 4:** Unlike a `LIKE` pattern that starts with a fixed prefix (`'A%'`), which a standard B-tree index **can** use efficiently, **regex matches (and `LIKE` patterns starting with `%`, like `'%son'`) generally cannot use a normal index** and force a full table scan in most default configurations — a genuinely important performance consideration for any search feature built on these operators at scale. Specialized indexes (like PostgreSQL's trigram `pg_trgm` extension) exist specifically to address this, a topic for Part 4.

---

## 9.9 IS NULL and IS NOT NULL — Understanding NULL Properly

This section is the **conceptual heart of this module** and arguably one of the most important sections in the entire handbook.

### What NULL Actually Represents

`NULL` represents **"unknown" or "not applicable"** — it is emphatically **not** the same as zero, an empty string, or "false." It's a genuinely distinct marker meaning "we don't know this value" or "this value doesn't apply here."

### Three-Valued Logic (3VL)

Unlike most general-purpose programming languages (which use **two-valued** boolean logic: `TRUE`/`FALSE`), SQL's `WHERE` clause operates under **three-valued logic**: every condition evaluates to `TRUE`, `FALSE`, or **`UNKNOWN`** — and `WHERE` only retains rows where the result is definitively `TRUE` (both `FALSE` and `UNKNOWN` rows are excluded).

```
Any comparison involving NULL (using standard operators like =, <>, <, >) evaluates to UNKNOWN:

5 = 5        → TRUE
5 = 6        → FALSE
5 = NULL     → UNKNOWN   (NOT false — genuinely "we don't know")
NULL = NULL  → UNKNOWN   (⚠ NOT true! Two unknowns aren't provably equal to each other)
```

> 🧠 **Real-life analogy:** Imagine asking "is the mystery box's weight equal to 5kg?" when nobody has ever weighed the box (its weight is `NULL`/unknown). The honest answer isn't "yes" or "no" — it's "we don't know." And if you ask "are two *different* unweighed mystery boxes the same weight as each other?" the honest answer is *still* "we don't know" — even though intuitively you might think "two unknowns" feels like it should be "equal," logically you cannot claim they match without evidence.

### Why You Cannot Use `= NULL`

```sql
SELECT * FROM employees WHERE department_id = NULL;    -- ⚠ ALWAYS returns zero rows, no error!
SELECT * FROM employees WHERE department_id IS NULL;   -- ✅ Correct way to test for NULL
```

Because `department_id = NULL` always evaluates to `UNKNOWN` (per the three-valued logic above, never `TRUE`), this condition **can never match any row**, no matter what — a silent, no-error logic bug that's genuinely common among SQL beginners (and occasionally trips up experienced engineers who momentarily forget this rule). `IS NULL` / `IS NOT NULL` are **special-purpose operators** specifically designed to correctly test for `NULL`-ness, bypassing the standard comparison operators' `UNKNOWN`-producing behavior entirely.

```sql
SELECT * FROM employees WHERE department_id IS NULL;
SELECT * FROM employees WHERE department_id IS NOT NULL;
```

### NULL in Arithmetic

```sql
SELECT 5 + NULL;    -- NULL (any arithmetic involving NULL produces NULL — "unknown + 5 = still unknown")
SELECT NULL * 0;    -- NULL, NOT 0! (a frequently-surprising result)
```

### NULL in Aggregate Functions (Preview — Fully Covered in Module 11)

```sql
SELECT AVG(salary) FROM employees;   -- NULL values are SILENTLY EXCLUDED from the average calculation
SELECT COUNT(salary) FROM employees; -- counts only NON-NULL salary values
SELECT COUNT(*) FROM employees;      -- counts ALL rows, regardless of NULLs in any column
```

> ⚠️ **A genuinely important, frequently-tested distinction (fully explored in Module 11):** `COUNT(*)` counts rows; `COUNT(column_name)` counts only the **non-`NULL`** values in that specific column — these can produce meaningfully different numbers on the same table, and confusing the two is a real, common mistake.

### Truth Table — AND / OR / NOT with UNKNOWN

| A | B | A AND B | A OR B |
|---|---|---|---|
| TRUE | UNKNOWN | UNKNOWN | TRUE |
| FALSE | UNKNOWN | FALSE | UNKNOWN |
| UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN |

```
NOT UNKNOWN = UNKNOWN   (crucially, NOT UNKNOWN is NOT "known"/TRUE — it stays UNKNOWN)
```

> 💡 **Interview-favorite pattern to internalize:** `TRUE AND UNKNOWN = UNKNOWN` (still excluded by `WHERE`), but `TRUE OR UNKNOWN = TRUE` (included!) — meaning `NULL`'s effect on a compound condition genuinely depends on *how* it's combined with other conditions, not just on `NULL`'s presence alone. This truth table is a legitimate, real thing to be able to reconstruct from memory in a rigorous interview.

---

## 9.10 Full Worked Example — Combining Operators

```sql
SELECT name, department, salary
FROM employees
WHERE (department IN ('Engineering', 'Sales'))
  AND salary BETWEEN 50000 AND 120000
  AND name ILIKE 'a%'
  AND department_id IS NOT NULL
  AND EXISTS (
      SELECT 1 FROM departments d
      WHERE d.id = employees.department_id AND d.active = TRUE
  )
ORDER BY salary DESC;
```

This deliberately layers nearly every operator from this module into one realistic query: set membership (`IN`), an inclusive range (`BETWEEN`), case-insensitive pattern matching (`ILIKE`), explicit `NULL` handling (`IS NOT NULL`), and a safe relational existence check (`EXISTS`) — combined with parenthesized, unambiguous logical grouping.

---

## 9.11 Advantages / Best Practices Recap

| Practice | Why |
|---|---|
| Always parenthesize mixed `AND`/`OR` | Prevents precedence-related logic bugs (Section 9.3) |
| Use half-open date ranges (`>=` / `<`) instead of `BETWEEN` for timestamps | Avoids silently missing same-day, later-time rows (Section 9.4) |
| Prefer `EXISTS`/`NOT EXISTS` over `IN`/`NOT IN` with subqueries | Immune to the `NOT IN` + `NULL` trap (Sections 9.5, 9.7) |
| Always use `IS NULL`/`IS NOT NULL`, never `= NULL` | `= NULL` silently never matches (Section 9.9) |
| Cast explicitly for fractional division | Avoids silent integer-division truncation (Section 9.1) |

## 9.12 Common Mistakes

- The `NOT IN (... NULL ...)` trap — silently returns zero rows (Section 9.5), arguably the single most important mistake in this entire module.
- Writing `column = NULL` instead of `column IS NULL` (Section 9.9).
- Forgetting `AND` binds tighter than `OR`, producing unintended logic (Section 9.3).
- Assuming `LIKE` case-sensitivity is consistent across PostgreSQL and MySQL without checking collation (Section 9.8).
- Using `BETWEEN` on timestamp ranges, silently excluding same-day later times (Section 9.4).

## 9.13 Edge Cases

- `NULL = NULL` evaluates to `UNKNOWN`, not `TRUE` — but the special construct `IS NOT DISTINCT FROM` (PostgreSQL; MySQL's equivalent is `<=>`, the "null-safe equal" operator) **does** treat two `NULL`s as equal, for the specific cases where that comparison semantics is genuinely needed.

```sql
SELECT NULL IS NOT DISTINCT FROM NULL;   -- PostgreSQL: TRUE
SELECT NULL <=> NULL;                     -- MySQL: 1 (TRUE)
```

- Empty string (`''`) is **not** the same as `NULL` in either engine — `'' IS NULL` is `FALSE` in both PostgreSQL and MySQL (a distinction some other database systems, notably Oracle, notoriously blur, treating empty strings as `NULL` — a genuine, real portability trap if you've worked with Oracle before).

## 9.14 Performance Implications

- Leading-wildcard `LIKE '%text'` and regex patterns generally cannot use standard B-tree indexes, forcing full scans (Section 9.8) — a real, common performance bottleneck in naive search implementations.
- `EXISTS` is the generally safer and often equally-or-more performant choice versus `IN`/`NOT IN` with subqueries, particularly once `NULL`s are possible in the subquery's result (Section 9.7).

## 9.15 Industry Usage

Every one of these operators appears constantly in real production `WHERE` clauses — filtering by category (`IN`), searching text (`LIKE`/`ILIKE`), checking relationships (`EXISTS`), and handling optional/missing data (`IS NULL`) are some of the most common query patterns written by backend engineers daily.

## 9.16 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Operator syntax and basic truth tables |
| Placement | "What's the difference between `LIKE` and `ILIKE`?" |
| Mid-level | "Why does `NOT IN` sometimes return zero rows unexpectedly?" (one of the most classic SQL interview questions) |
| Senior SDE-2/SDE-3 | "Explain three-valued logic and construct the `AND`/`OR` truth tables involving `UNKNOWN`." / "Why is `EXISTS` generally preferred over `NOT IN` with subqueries?" |

---

## 9.17 Chapter Summary

- Integer division truncates; cast to a decimal type for fractional results (Section 9.1).
- `AND` binds tighter than `OR` — always parenthesize mixed conditions (Section 9.3).
- `BETWEEN` is always inclusive on both ends — a real trap for timestamp ranges (Section 9.4).
- **`NOT IN` combined with a `NULL`-containing list silently returns zero rows** — the single most important gotcha in this module; prefer `NOT EXISTS` (Sections 9.5, 9.7).
- `LIKE` case-sensitivity genuinely differs by default between PostgreSQL (sensitive) and MySQL (collation-dependent, often insensitive); `ILIKE` is PostgreSQL-only.
- SQL uses **three-valued logic** (`TRUE`/`FALSE`/`UNKNOWN`) — `NULL` comparisons always yield `UNKNOWN`, never `TRUE` or `FALSE`, which is why `= NULL` never matches and why `IS NULL`/`IS NOT NULL` exist as dedicated operators.

---

## 9.18 Revision Notes / Cheat Sheet

```sql
-- Arithmetic: integer / integer = integer (truncated); cast for fractions
SELECT 7::NUMERIC / 2;         -- 3.5 (PostgreSQL cast)

-- Always parenthesize mixed AND/OR
WHERE (a OR b) AND c

-- BETWEEN is inclusive both ends — use half-open ranges for timestamps
WHERE ts >= '2026-01-01' AND ts < '2026-02-01'

-- NOT IN + NULL = always zero rows. Use NOT EXISTS instead.
WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)

-- NULL testing
WHERE col IS NULL / IS NOT NULL     -- ✅ correct
WHERE col = NULL                     -- ❌ never matches, no error

-- Pattern matching
LIKE 'A%'      -- prefix match, case rules vary by engine
ILIKE 'a%'     -- PostgreSQL only, explicit case-insensitive
~ / ~*         -- PostgreSQL regex (case-sensitive / insensitive)
REGEXP         -- MySQL regex

-- Null-safe equality
IS NOT DISTINCT FROM   -- PostgreSQL
<=>                     -- MySQL
```

---

## 9.19 Multiple Choice Questions (MCQs)

1. What does `SELECT 7 / 2;` return in both PostgreSQL and MySQL (integer operands)?
   a) 3.5 b) **3** c) 4 d) An error

2. `department_id NOT IN (1, 2, NULL)` returns:
   a) All rows except department 1 and 2 b) **Zero rows, always** c) All rows d) An error

3. What is the result of `5 = NULL`?
   a) TRUE b) FALSE c) **UNKNOWN** d) NULL is not comparable, causing an error

4. Which operator is PostgreSQL-specific for explicit case-insensitive pattern matching?
   a) `LIKE` b) **`ILIKE`** c) `REGEXP` d) `SIMILAR TO`

5. `BETWEEN 50000 AND 80000` is:
   a) Exclusive on both ends b) **Inclusive on both ends** c) Inclusive only on the lower end d) Engine-dependent

---

## 9.20 Exercises

### 🟢 Beginner
1. Write a query finding all employees with a salary `BETWEEN` 60,000 and 90,000.
2. Write a query finding all employees whose name starts with 'S' using `LIKE`.
3. Write a query finding all employees with a `NULL` department.

### 🟡 Intermediate
4. Demonstrate the `NOT IN` + `NULL` trap: create a small table with a `NULL` in a list, run a `NOT IN` query, observe the zero-row result, then rewrite it correctly using `NOT EXISTS`.
5. Write a query combining `AND`/`OR` with explicit parentheses to find employees who are either in Engineering with salary > 100,000, or in Sales with salary > 70,000.
6. Compare `LIKE` vs `ILIKE` (or MySQL's collation-based case sensitivity) by querying for a name with mismatched letter casing.

### 🔴 Advanced / Output Prediction
7. Predict the output of `SELECT NULL = NULL, NULL IS NOT DISTINCT FROM NULL;` and explain the difference between the two results.
8. Given a `WHERE department = 'Engineering' OR department = 'Sales' AND salary > 100000` clause, explain (without running it) exactly which rows will be returned, referencing operator precedence.

### 🧩 Scenario-Based
9. A production report using `WHERE customer_id NOT IN (SELECT customer_id FROM blocked_customers)` started returning zero rows after a new row with a `NULL` customer_id was added to `blocked_customers`. Diagnose the cause and provide the fix.

### 🏗️ Mini Project
10. Write a comprehensive query against your practice schema (Sakila/Pagila) using at least six different operators from this module (e.g., `IN`, `BETWEEN`, `LIKE`, `IS NOT NULL`, `EXISTS`, and a logical combination with parentheses), and explain in a comment above each clause what it does.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [SQLBolt — WHERE clauses](https://sqlbolt.com/)
- [DataLemur](https://datalemur.com/)

---

## ✅ Module 9 Complete

You now have a complete, rigorous command of SQL's operator vocabulary — including the `NULL`/three-valued-logic subtleties that quietly cause more real production bugs than almost any other SQL topic, and are a near-guaranteed interview question at every level.

**Next up: Module 10 — SQL Functions**, covering numeric, string, date/time, conversion, and conditional functions (`CASE`, `COALESCE`, `NULLIF`, and more).

---

*Reply "continue" or "next module" when you're ready for Module 10.*
