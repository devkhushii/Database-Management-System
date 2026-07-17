# Part 3 — The Complete SQL Handbook
## Module 10: SQL Functions

> **Goal of this module:** Master the built-in function library every SQL engine provides — numeric, string, date/time, type-conversion, and conditional functions (`CASE`, `COALESCE`, `NULLIF`, and engine-specific `NULL`-handling shorthand). Functions are what turn raw stored values into the shaped, computed output real applications actually need.

---

### 📍 Where This Module Fits

```
Module 9: Filtering rows  →  Module 10: Transforming values (You are here)  →  Module 11: Aggregating across rows
```

---

## 10.1 What Is a SQL Function?

A **function** takes zero or more input values and returns a single computed value, evaluated **per row** (as opposed to aggregate functions, Module 11, which collapse many rows into one value). Functions are called using standard `function_name(arguments)` syntax, usable almost anywhere an expression is valid: `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, and more.

> 🧠 **Real-life analogy:** A function is like a kitchen appliance — a blender takes raw ingredients (arguments) and produces one specific output (a smoothie) every time, following the exact same process regardless of which ingredients you feed it, as long as they're valid inputs.

---

## 10.2 Numeric Functions

| Function | Purpose | Example |
|---|---|---|
| `ROUND(x, n)` | Round to `n` decimal places | `ROUND(19.567, 2)` → `19.57` |
| `CEIL(x)` / `CEILING(x)` | Round up to nearest integer | `CEIL(4.1)` → `5` |
| `FLOOR(x)` | Round down to nearest integer | `FLOOR(4.9)` → `4` |
| `ABS(x)` | Absolute value | `ABS(-15)` → `15` |
| `POWER(x, y)` | `x` raised to power `y` | `POWER(2, 10)` → `1024` |
| `SQRT(x)` | Square root | `SQRT(144)` → `12` |
| `MOD(x, y)` | Remainder (equivalent to `%`) | `MOD(17, 5)` → `2` |
| `SIGN(x)` | Returns -1, 0, or 1 | `SIGN(-42)` → `-1` |
| `RANDOM()` (PostgreSQL) / `RAND()` (MySQL) | Random float between 0 and 1 | — |
| `TRUNC(x, n)` | Truncate (not round) to `n` decimals | `TRUNC(19.567, 1)` → `19.5` |

```sql
SELECT ROUND(salary / 12, 2) AS monthly_salary FROM employees;
```

> ⚠️ **`ROUND` vs `TRUNC` — a genuinely common point of confusion:** `ROUND` rounds to the nearest value (standard "round half up/away from zero" behavior in most contexts); `TRUNC` simply **chops off** digits beyond the specified precision, without any rounding logic at all. `ROUND(19.567, 1)` → `19.6`, but `TRUNC(19.567, 1)` → `19.5` — these are genuinely different operations, not stylistic variants of each other.

---

## 10.3 String Functions

| Function | Purpose | Example |
|---|---|---|
| `LENGTH(s)` | Character count | `LENGTH('hello')` → `5` |
| `UPPER(s)` / `LOWER(s)` | Case conversion | `UPPER('sql')` → `'SQL'` |
| `TRIM(s)` | Remove leading/trailing whitespace | `TRIM('  hi  ')` → `'hi'` |
| `LTRIM(s)` / `RTRIM(s)` | Trim only left/right | — |
| `SUBSTRING(s, start, len)` | Extract a substring | `SUBSTRING('database', 1, 4)` → `'data'` |
| `CONCAT(s1, s2, ...)` | Concatenate strings | `CONCAT('SQL', ' ', 'Handbook')` → `'SQL Handbook'` |
| `REPLACE(s, from, to)` | Replace all occurrences | `REPLACE('foobar', 'bar', 'baz')` → `'foobaz'` |
| `POSITION(sub IN s)` / `INSTR(s, sub)` | Find substring position (1-indexed) | — |
| `LPAD(s, len, pad)` / `RPAD(s, len, pad)` | Pad to a fixed length | `LPAD('7', 3, '0')` → `'007'` |
| `SPLIT_PART(s, delim, n)` (PostgreSQL) | Split and extract the Nth piece | `SPLIT_PART('a,b,c', ',', 2)` → `'b'` |
| `STRING_AGG` (PostgreSQL) / `GROUP_CONCAT` (MySQL) | Aggregate strings across rows (technically an aggregate function — previewed here, fully covered in Module 11) | — |

### String Concatenation — A Genuine Syntax Divergence

```sql
-- PostgreSQL — supports both the ANSI standard || operator AND the function
SELECT first_name || ' ' || last_name AS full_name FROM employees;
SELECT CONCAT(first_name, ' ', last_name) FROM employees;

-- MySQL — || means something ENTIRELY different by default!
SELECT CONCAT(first_name, ' ', last_name) FROM employees;   -- ✅ the reliable way
SELECT first_name || ' ' || last_name FROM employees;        -- ⚠ behaves as LOGICAL OR by default!
```

> ⚠️ **A genuinely dangerous, real gotcha:** In MySQL, by default (unless the `PIPES_AS_CONCAT` SQL mode is explicitly enabled), `||` is treated as a **logical OR operator**, not string concatenation — the exact opposite of its ANSI-standard meaning (which PostgreSQL correctly implements). Running a Postgres-style concatenation query unmodified against MySQL doesn't necessarily error — it can silently produce **wrong boolean-like results** (`0` or `1`) instead of concatenated text, a genuinely dangerous, silent-failure category of bug. **Always use `CONCAT()` for string concatenation in MySQL** to avoid this trap entirely; it also works perfectly well in PostgreSQL, making it the safer, portable default across both engines.

### NULL Propagation in String Functions — A Recurring Theme

```sql
SELECT 'Hello' || NULL;         -- PostgreSQL: returns NULL (not 'Hello')!
SELECT CONCAT('Hello', NULL);   -- PostgreSQL: also returns NULL
SELECT CONCAT('Hello', NULL);   -- MySQL: also returns NULL
```

> ⚠️ **Connects directly back to Module 9, Section 9.9:** Almost every function in this module (and in SQL generally) **propagates `NULL`** — if any input argument is `NULL`, the function's result is typically `NULL` too, consistent with `NULL`'s "unknown" semantics ("concatenating a known string with an *unknown* value produces an *unknown* result"). This is precisely why `COALESCE` (Section 10.6) exists — to provide a fallback value specifically for these situations.

---

## 10.4 Date Functions

| Function | Purpose | PostgreSQL | MySQL |
|---|---|---|---|
| Current date | Today's date | `CURRENT_DATE` | `CURDATE()` / `CURRENT_DATE` |
| Current timestamp | Now, with time | `CURRENT_TIMESTAMP` / `NOW()` | `NOW()` / `CURRENT_TIMESTAMP` |
| Extract a part | Year, month, day, etc. | `EXTRACT(YEAR FROM col)` | `YEAR(col)`, `MONTH(col)`, `DAY(col)` |
| Add/subtract an interval | Date arithmetic | `col + INTERVAL '7 days'` | `DATE_ADD(col, INTERVAL 7 DAY)` |
| Difference between dates | Days between two dates | `date2 - date1` (returns integer days directly) | `DATEDIFF(date2, date1)` |
| Format a date as text | Custom display format | `TO_CHAR(col, 'YYYY-MM-DD')` | `DATE_FORMAT(col, '%Y-%m-%d')` |
| Truncate to a unit | Start of month/week/etc. | `DATE_TRUNC('month', col)` | No direct equivalent — commonly emulated |

```sql
-- PostgreSQL
SELECT EXTRACT(YEAR FROM hired_on) AS hire_year FROM employees;
SELECT hired_on + INTERVAL '30 days' AS review_date FROM employees;
SELECT DATE_TRUNC('month', hired_on) AS hire_month FROM employees;

-- MySQL
SELECT YEAR(hired_on) AS hire_year FROM employees;
SELECT DATE_ADD(hired_on, INTERVAL 30 DAY) AS review_date FROM employees;
```

### DATE_TRUNC — A Genuinely Powerful PostgreSQL Feature

```sql
SELECT DATE_TRUNC('month', order_date) AS month, SUM(total) AS monthly_revenue
FROM orders
GROUP BY DATE_TRUNC('month', order_date);   -- foreshadowing Module 11's GROUP BY
```

`DATE_TRUNC` rounds a timestamp **down** to the start of the specified unit (`'month'`, `'week'`, `'day'`, `'hour'`, etc.) — extremely common for reporting/analytics ("group orders by month"). MySQL has no single-function equivalent; the idiomatic workaround is typically `DATE_FORMAT(col, '%Y-%m-01')` (for month-truncation specifically) or combining `YEAR()`/`MONTH()` in a `GROUP BY`.

### CURRENT_DATE vs NOW() — Timing Subtlety

> 💡 **Worth knowing:** Within a single statement (or, in PostgreSQL, within a single transaction), `CURRENT_TIMESTAMP`/`NOW()` returns the **same value every time it's referenced** (the moment the statement/transaction began), not a live, continuously-updating clock — this is a deliberate consistency guarantee, ensuring multiple references to "now" within one query see identical values rather than drifting mid-execution.

---

## 10.5 Time Functions

| Function | Purpose |
|---|---|
| `CURRENT_TIME` | Current time of day (no date) |
| `EXTRACT(HOUR FROM col)` (PostgreSQL) / `HOUR(col)` (MySQL) | Extract hour component |
| `AGE(timestamp)` (PostgreSQL only) | Human-readable interval since a given timestamp |

```sql
SELECT AGE(hired_on) AS tenure FROM employees;   -- PostgreSQL: e.g., "3 years 2 mons 10 days"
```

`AGE()` is a genuinely convenient PostgreSQL-only function for computing human-readable durations — MySQL requires manually computing and formatting the difference using `DATEDIFF`/`TIMESTAMPDIFF` instead.

---

## 10.6 Conversion Functions

```sql
-- PostgreSQL — the :: cast operator (PostgreSQL-specific shorthand)
SELECT '123'::INT;
SELECT salary::TEXT FROM employees;

-- ANSI standard CAST — works in BOTH PostgreSQL and MySQL
SELECT CAST('123' AS INTEGER);
SELECT CAST(salary AS CHAR) FROM employees;

-- Formatting functions (as introduced in Section 10.4)
SELECT TO_CHAR(hired_on, 'DD Mon YYYY');     -- PostgreSQL
SELECT DATE_FORMAT(hired_on, '%d %b %Y');    -- MySQL
```

> 💡 **Best practice:** Use the standard `CAST(expr AS type)` syntax when writing code intended to be portable across engines; PostgreSQL's `::` shorthand is convenient and extremely widely used in Postgres-specific codebases, but it's non-standard and unavailable in MySQL.

---

## 10.7 Conditional Functions

### CASE — The General-Purpose Conditional Expression

```sql
SELECT
    name,
    salary,
    CASE
        WHEN salary >= 100000 THEN 'High'
        WHEN salary >= 60000  THEN 'Medium'
        ELSE 'Entry'
    END AS salary_band
FROM employees;
```

**What it is:** SQL's equivalent of an `if/elif/else` chain — evaluated **per row**, checking each `WHEN` condition in order and returning the first match's `THEN` result, falling back to `ELSE` (or `NULL` if `ELSE` is omitted and no condition matches).

### Two Forms of CASE

```sql
-- "Searched" CASE — arbitrary boolean conditions per branch (most flexible, shown above)
CASE WHEN condition1 THEN result1 WHEN condition2 THEN result2 ELSE default END

-- "Simple" CASE — compares one expression against several exact values
CASE department
    WHEN 'Engineering' THEN 'Tech'
    WHEN 'Sales' THEN 'Revenue'
    ELSE 'Other'
END
```

> 💡 **`CASE` works virtually anywhere an expression is valid** — not just `SELECT`, but also `WHERE`, `ORDER BY`, and even inside aggregate functions (a genuinely powerful, commonly-used pattern called **conditional aggregation**, previewed here and fully explored in Module 11):

```sql
SELECT COUNT(CASE WHEN status = 'active' THEN 1 END) AS active_count FROM employees;
```

### COALESCE — Return the First Non-NULL Value

```sql
SELECT COALESCE(preferred_name, first_name, 'Unknown') AS display_name FROM employees;
```

**What it does:** Evaluates its arguments **left to right** and returns the **first one that isn't `NULL`** — if every argument is `NULL`, the result is `NULL`. `COALESCE` is part of the **ANSI SQL standard** and works identically in both PostgreSQL and MySQL, accepting **any number of arguments** (not just two).

> 🧠 **Real-life analogy:** `COALESCE` is like a chain of backup contacts — "call their cell phone; if no answer, try their home phone; if still no answer, try their emergency contact" — it works through the list in order and stops at the first one that actually "answers" (is non-`NULL`).

### NULLIF — Return NULL If Two Values Are Equal

```sql
SELECT NULLIF(discount_pct, 0) AS effective_discount FROM products;
-- If discount_pct is 0, returns NULL; otherwise returns discount_pct unchanged
```

**What it does:** The logical inverse of `COALESCE` in spirit — `NULLIF(a, b)` returns `NULL` if `a = b`, otherwise returns `a` unchanged. Genuinely useful for **avoiding division-by-zero errors**:

```sql
SELECT total_revenue / NULLIF(total_orders, 0) AS avg_order_value FROM sales_summary;
-- If total_orders is 0, this becomes total_revenue / NULL = NULL (safe), instead of a
-- division-by-zero ERROR that would otherwise abort the entire query
```

> 💡 **This `NULLIF`-for-safe-division pattern is a genuinely common, real, frequently-used idiom** in production reporting queries — well worth memorizing directly, since "how would you avoid a division by zero in SQL" is a realistic, practical interview question.

### IFNULL (MySQL) and NVL (Oracle) — Engine-Specific Two-Argument Shorthand

```sql
SELECT IFNULL(preferred_name, first_name) FROM employees;   -- MySQL only
SELECT NVL(preferred_name, first_name) FROM employees;       -- Oracle only (mentioned for completeness)
```

**What they are:** Simpler, **two-argument-only** versions of the same "fallback if `NULL`" concept as `COALESCE`, but **not** part of the ANSI standard and **not portable**:

| Function | Engine | Standard? | Argument Count |
|---|---|---|---|
| `COALESCE(a, b, c, ...)` | **Both** PostgreSQL and MySQL | ✅ ANSI standard | Any number |
| `IFNULL(a, b)` | MySQL only | ❌ MySQL-proprietary | Exactly 2 |
| `NVL(a, b)` | Oracle only | ❌ Oracle-proprietary | Exactly 2 |
| PostgreSQL equivalent to `IFNULL`/`NVL`? | Just use `COALESCE(a, b)` | — | — |

> ⚠️ **Frequently-tested best practice:** Since `COALESCE` works identically and portably in **both** PostgreSQL and MySQL (and is the true ANSI-standard function), **prefer `COALESCE` over `IFNULL`/`NVL`** in new code, even in MySQL where `IFNULL` is available and slightly more concise — this avoids vendor lock-in and keeps your SQL skills/code maximally transferable, a genuinely good habit this entire handbook has been building toward since Module 1's "standard vs. engine behavior" theme.

---

## 10.8 Full Worked Example — Functions Combined

```sql
SELECT
    UPPER(TRIM(name)) AS clean_name,
    COALESCE(department, 'Unassigned') AS department,
    ROUND(salary / 12, 2) AS monthly_salary,
    CASE
        WHEN salary >= 100000 THEN 'Senior'
        WHEN salary >= 60000  THEN 'Mid'
        ELSE 'Junior'
    END AS level,
    EXTRACT(YEAR FROM hired_on) AS hire_year,
    total_revenue / NULLIF(total_orders, 0) AS avg_order_value
FROM employees
JOIN sales_summary USING (employee_id);   -- (JOIN syntax previewed here, fully covered Module 12)
```

---

## 10.9 Advantages / Best Practices Recap

| Practice | Why |
|---|---|
| Use `CONCAT()`, not `||`, for MySQL string concatenation | Avoids the silent logical-OR trap (Section 10.3) |
| Prefer `COALESCE` over `IFNULL`/`NVL` | Portable, ANSI-standard, works identically in both engines (Section 10.7) |
| Use `NULLIF` to guard against division by zero | Converts a hard error into a safe `NULL` (Section 10.7) |
| Prefer `CAST(expr AS type)` over `::` for portable code | `::` is PostgreSQL-only shorthand (Section 10.6) |
| Understand `NULL` propagation through functions | Prevents confusing "why is my result `NULL`" debugging sessions (Section 10.3) |

## 10.10 Common Mistakes

- Using `||` for string concatenation in MySQL, silently getting boolean-like results instead of an error (Section 10.3).
- Confusing `ROUND` with `TRUNC` (Section 10.2).
- Forgetting that most functions propagate `NULL` — a single `NULL` argument silently `NULL`-ifies the entire expression.
- Dividing without `NULLIF` protection, causing a hard runtime error when a divisor happens to be zero (Section 10.7).
- Using MySQL's `IFNULL` or Oracle's `NVL` out of habit in code meant to be portable, instead of standard `COALESCE`.

## 10.11 Edge Cases

- `COALESCE` short-circuits — later arguments after the first non-`NULL` one are not evaluated in some engines' implementations, which can matter if a later argument is an expensive subquery or has side effects (rare, but worth knowing).
- `DATE_TRUNC` in PostgreSQL is timezone-aware when used with `TIMESTAMPTZ` — the "start of day/month" boundary shifts depending on the session's timezone setting, directly connecting back to **Module 5, Section 5.5**'s `TIMESTAMPTZ` conversion behavior.
- `CASE` without an `ELSE` clause returns `NULL` for any row matching no condition — an easy, silent source of unexpected `NULL`s downstream if forgotten.

## 10.12 Performance Implications

- Wrapping an indexed column in a function inside `WHERE` (e.g., `WHERE UPPER(name) = 'ALICE'`) generally **prevents the optimizer from using a standard index** on that column, since the stored index values don't match the transformed output — a genuinely important, frequently-tested performance consideration (the fix, **functional/expression indexes**, is previewed here and covered fully in Part 4).
- `CASE`-based conditional aggregation (Section 10.7) is a highly efficient pattern — computing multiple conditional counts/sums in a **single pass** over the data, rather than running several separate filtered queries.

## 10.13 Industry Usage

Functions are used constantly in real reporting and application queries — formatting dates for display, normalizing text for search/comparison, safely handling missing data with `COALESCE`, and building categorical breakdowns with `CASE` are some of the most common, everyday SQL patterns in production analytics and backend code.

## 10.14 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Basic string/numeric/date function syntax |
| Placement | "What does `COALESCE` do?" / "How do you avoid division by zero in SQL?" |
| Mid-level | "Why might `||` behave unexpectedly in MySQL?" / "Write a query using `CASE` for conditional aggregation." |
| Senior SDE-2/SDE-3 | "Why does wrapping a column in a function in `WHERE` hurt performance, and how would you fix it?" |

---

## 10.15 Chapter Summary

- Numeric functions (`ROUND`, `FLOOR`, `CEIL`, `TRUNC`) each behave distinctly — don't conflate rounding and truncation.
- String concatenation should use `CONCAT()`, not `||`, when MySQL compatibility matters — `||` silently means logical OR in default MySQL configuration.
- Date functions diverge significantly in naming between PostgreSQL and MySQL, though the underlying concepts (extracting parts, adding intervals, truncating to a unit) are consistent.
- `CASE` is SQL's general-purpose conditional expression, usable almost anywhere, including for powerful conditional aggregation patterns.
- `COALESCE` (ANSI-standard, portable) should be preferred over engine-specific `IFNULL`/`NVL` for `NULL` fallback logic; `NULLIF` is the standard, elegant guard against division by zero.
- Nearly every function **propagates `NULL`** — a single `NULL` input typically `NULL`-ifies the entire result.

---

## 10.16 Revision Notes / Cheat Sheet

```sql
ROUND(x, n)   -- rounds
TRUNC(x, n)   -- chops, no rounding

CONCAT(a, b, c)          -- portable string concat (safe in both engines)
a || b                    -- PostgreSQL concat / ⚠ MySQL logical OR by default

CURRENT_DATE / CURRENT_TIMESTAMP        -- both engines
EXTRACT(YEAR FROM col)                   -- PostgreSQL
YEAR(col)                                -- MySQL
DATE_TRUNC('month', col)                 -- PostgreSQL only

CAST(expr AS type)    -- ANSI standard, portable
expr::type              -- PostgreSQL shorthand only

CASE WHEN cond THEN val ... ELSE default END   -- general conditional
COALESCE(a, b, c, ...)                          -- first non-NULL, ANSI standard, portable
NULLIF(a, b)                                     -- NULL if a=b, else a — safe division guard
IFNULL(a, b)   -- MySQL only, 2 args    NVL(a, b)  -- Oracle only, 2 args
```

---

## 10.17 Multiple Choice Questions (MCQs)

1. In MySQL's default configuration, what does `'a' || 'b'` return?
   a) `'ab'` b) **A boolean-like `0`/`1` result (logical OR)** c) An error d) `NULL`

2. Which function safely prevents a division-by-zero error?
   a) `COALESCE` b) **`NULLIF`** c) `CASE` alone d) `ROUND`

3. Which function is ANSI-standard and portable across both PostgreSQL and MySQL?
   a) `IFNULL` b) `NVL` c) **`COALESCE`** d) None of the above

4. `TRUNC(19.567, 1)` returns:
   a) 19.6 b) **19.5** c) 20 d) 19

5. Wrapping an indexed column in a function inside `WHERE` typically:
   a) Improves performance b) Has no effect c) **Prevents the optimizer from using a standard index on that column** d) Is not allowed

---

## 10.18 Exercises

### 🟢 Beginner
1. Write a query that rounds every employee's salary to the nearest thousand using `ROUND`.
2. Write a query concatenating `first_name` and `last_name` into a `full_name` column, using `CONCAT`.
3. Write a query extracting the year from a `hired_on` date column.

### 🟡 Intermediate
4. Write a query using `CASE` to classify employees into 'High', 'Medium', 'Low' salary bands.
5. Write a query using `COALESCE` to display a `nickname` if present, otherwise fall back to `first_name`.
6. Write a query computing `revenue / orders` for a sales table, using `NULLIF` to safely handle rows where `orders` is 0.

### 🔴 Advanced / Debugging
7. A MySQL query using `first_name || ' ' || last_name` returns `0` for every row instead of a concatenated name. Diagnose and fix it.
8. Explain why `WHERE UPPER(email) = 'ALICE@EXAMPLE.COM'` might perform poorly on a large, indexed `email` column, and describe (conceptually) how a functional index could help.

### 🧩 Scenario-Based
9. You need to compute, per row, the number of days since an order was placed, formatted as a human-readable string. Describe the PostgreSQL functions you'd combine (referencing Sections 10.4–10.5) to accomplish this.

### 🏗️ Mini Project
10. Using your practice schema, write a single query that: cleans/normalizes a text column (trim + case), computes a conditional `CASE`-based category, safely handles a potential division, and formats a date — combining at least five distinct functions from this module.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [PostgreSQL Docs — Functions and Operators](https://www.postgresql.org/docs/current/functions.html)
- [MySQL Docs — Functions](https://dev.mysql.com/doc/refman/8.0/en/functions.html)

---

## ✅ Module 10 Complete

You can now transform, format, and safely handle `NULL`-prone data using SQL's full function library — and you understand several genuinely dangerous, real cross-engine gotchas (`||` in MySQL, `IFNULL` vs `COALESCE`) that trip up even experienced engineers moving between PostgreSQL and MySQL.

**Next up: Module 11 — Aggregate Functions**, where individual-row functions give way to `COUNT`, `SUM`, `AVG`, `GROUP BY`, `HAVING`, and multi-dimensional grouping with `ROLLUP`/`CUBE`.

---

*Reply "continue" or "next module" when you're ready for Module 11.*
