# Part 3 — The Complete SQL Handbook
## Module 17: Common SQL Mistakes

> **Goal of this module:** A dedicated, exhaustive catalog of the SQL mistakes that appear most often in real production code and technical interviews — `NULL` mistakes, join mistakes, aggregate mistakes, duplicate-row causes, Cartesian products, incorrect `GROUP BY` usage, wrong filtering logic, and a preview of SQL injection. Most of these have already appeared scattered across Modules 1–16; this module gathers them into one focused, debugging-oriented reference.

---

### 📍 Where This Module Fits

```
Module 16: How to write SQL well  →  Module 17: What goes wrong when you don't (You are here)  →  Module 18: Interview Questions
```

---

## 17.1 NULL Mistakes

### Mistake 1: Using `= NULL` Instead of `IS NULL`

```sql
-- WRONG — always returns zero rows, no error
SELECT * FROM employees WHERE department_id = NULL;

-- CORRECT
SELECT * FROM employees WHERE department_id IS NULL;
```

**Root cause:** Three-valued logic (**Module 9, Section 9.9**) — any comparison involving `NULL` evaluates to `UNKNOWN`, never `TRUE`.

### Mistake 2: The `NOT IN` + `NULL` Trap

```sql
-- WRONG — silently returns ZERO rows if the subquery/list contains any NULL
SELECT * FROM employees WHERE department_id NOT IN (SELECT id FROM departments WHERE region IS NULL);

-- CORRECT
SELECT * FROM employees e WHERE NOT EXISTS (
    SELECT 1 FROM departments d WHERE d.id = e.department_id AND d.region IS NULL
);
```

**Root cause:** Fully explained in **Module 9, Section 9.5** — arguably the single most important mistake in this entire handbook to have internalized.

### Mistake 3: Assuming Aggregates Treat NULL as Zero

```sql
-- If some salaries are NULL, this average excludes them entirely — it does NOT treat them as 0
SELECT AVG(salary) FROM employees;

-- If you genuinely want NULLs treated as 0:
SELECT AVG(COALESCE(salary, 0)) FROM employees;
```

**Root cause:** **Module 11, Section 11.2** — aggregate functions silently exclude `NULL` inputs.

### Mistake 4: Forgetting SUM/AVG Return NULL (Not 0) Over Zero Rows

```sql
SELECT SUM(salary) FROM employees WHERE department = 'NonexistentDept';   -- returns NULL, not 0
```

**Fix:** Wrap with `COALESCE(SUM(...), 0)` whenever downstream code expects a numeric default.

---

## 17.2 JOIN Mistakes

### Mistake 1: Missing Join Condition (Accidental Cartesian Product)

```sql
-- WRONG — old-style comma join with NO WHERE condition = full Cartesian product
SELECT e.name, d.name FROM employees e, departments d;

-- CORRECT
SELECT e.name, d.name FROM employees e JOIN departments d ON e.department_id = d.id;
```

**Root cause:** **Module 12, Section 12.6** — without an explicit join condition, every row from one table is paired with every row from the other, producing `|A| × |B|` rows.

### Mistake 2: A LEFT JOIN That Accidentally Becomes an INNER JOIN

```sql
-- WRONG — intended to keep ALL employees, but the WHERE clause silently
-- filters out every employee with no matching department
SELECT e.name, d.region
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.region = 'APAC';

-- CORRECT — move the filter into the ON clause to preserve unmatched left rows
SELECT e.name, d.region
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id AND d.region = 'APAC';
```

**Root cause:** Directly connects to **Module 12, Section 12.15** and **Module 9's three-valued logic** — for unmatched left rows, every right-side column is `NULL`, and `d.region = 'APAC'` evaluates to `UNKNOWN` for those rows, causing `WHERE` to silently exclude them — turning what was meant to be a `LEFT JOIN` into behavior indistinguishable from an `INNER JOIN`.

### Mistake 3: Self-Join Producing Reversed Duplicate Pairs

```sql
-- WRONG — produces both (Asha, Ravi) AND (Ravi, Asha) as separate rows
SELECT e1.name, e2.name FROM employees e1 JOIN employees e2
    ON e1.department_id = e2.department_id AND e1.id <> e2.id;

-- CORRECT — use < instead of <> to keep each pair exactly once
SELECT e1.name, e2.name FROM employees e1 JOIN employees e2
    ON e1.department_id = e2.department_id AND e1.id < e2.id;
```

**Root cause:** **Module 12, Section 12.7**.

### Mistake 4: Assuming NATURAL JOIN Is Safe

```sql
-- RISKY — silently changes behavior if a new same-named column is added to either table later
SELECT * FROM employees NATURAL JOIN departments;

-- SAFER
SELECT * FROM employees e JOIN departments d ON e.department_id = d.id;
```

**Root cause:** **Module 12, Section 12.8**.

---

## 17.3 Aggregate Mistakes

### Mistake 1: Selecting Non-Aggregated, Non-Grouped Columns

```sql
-- INVALID in PostgreSQL and modern MySQL (strict mode)
SELECT department, name, AVG(salary) FROM employees GROUP BY department;

-- CORRECT — every non-aggregated column must be in GROUP BY
SELECT department, AVG(salary) FROM employees GROUP BY department;
```

**Root cause:** **Module 11, Section 11.4** — and, per **Module 15**, this is a direct consequence of `GROUP BY` executing *before* `SELECT` in the logical execution order.

### Mistake 2: Confusing COUNT(*), COUNT(column), and COUNT(DISTINCT column)

```sql
SELECT COUNT(*) FROM employees;                    -- all rows
SELECT COUNT(department) FROM employees;             -- non-NULL department values only
SELECT COUNT(DISTINCT department) FROM employees;     -- distinct non-NULL department values
```

**Root cause:** **Module 11, Section 11.2** — three genuinely different numbers, frequently confused.

### Mistake 3: Filtering Aggregates with WHERE Instead of HAVING

```sql
-- INVALID
SELECT department, AVG(salary) FROM employees WHERE AVG(salary) > 70000 GROUP BY department;

-- CORRECT
SELECT department, AVG(salary) FROM employees GROUP BY department HAVING AVG(salary) > 70000;
```

**Root cause:** **Module 11, Section 11.5**, formally explained by **Module 15's execution order**.

---

## 17.4 Duplicate Rows

### Common Cause 1: A "Fan-Out" Join

```sql
-- If a customer has 3 orders, this query returns the customer's name THREE times
-- (once per matching order row) — not necessarily a "bug," but often unexpected
-- if the goal was one row per customer
SELECT c.name, o.id AS order_id
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

**Root cause:** This is often **not actually a mistake** — it's the natural, correct result of a one-to-many join. The mistake happens when someone **expects one row per customer** but forgot the join naturally multiplies rows for each match. **The fix depends on intent**: if you want one row per customer with aggregated order info, use `GROUP BY` (Module 11); if you genuinely want one row per order, the "duplication" is correct behavior, not a bug at all.

### Common Cause 2: Slapping DISTINCT on Without Understanding Why Duplicates Appeared

```sql
-- Papers over the symptom without addressing the root cause
SELECT DISTINCT c.name FROM customers c JOIN orders o ON o.customer_id = c.id;
```

**Root cause:** **Module 8, Section 8.4** explicitly warns against this — `DISTINCT` is expensive and often masks an underlying join-fanout issue that's better solved by restructuring the query (e.g., using `EXISTS`, **Module 13**, if you just need "customers who have any order," not the order data itself).

---

## 17.5 Cartesian Products (Revisited)

Already covered as a **join mistake** (Section 17.2) and in **Module 12, Section 12.6** — worth reiterating here as its own category because of its severity: an accidental Cartesian product between two moderately-sized tables (say, 10,000 rows each) produces **100,000,000 rows**, capable of exhausting memory, locking up a production database, or silently corrupting downstream aggregate calculations with wildly inflated counts.

> ⚠️ **Debugging tip:** If a query's row count seems **implausibly large** relative to what you'd intuitively expect, the very first thing to check is whether every table in the `FROM`/`JOIN` clause has a **proper, explicit join condition** — a missing condition is by far the most common cause of unexpectedly massive result sets.

---

## 17.6 Incorrect GROUP BY Usage

### Mistake: Grouping by the Wrong Granularity

```sql
-- If you want "average salary per department PER YEAR", but only group by department,
-- you'll get one row per department, silently averaging across ALL years together
SELECT department, AVG(salary) FROM employees GROUP BY department;

-- CORRECT — group by BOTH dimensions if that's genuinely the intent
SELECT department, EXTRACT(YEAR FROM hired_on) AS hire_year, AVG(salary)
FROM employees
GROUP BY department, EXTRACT(YEAR FROM hired_on);
```

**Root cause:** Not a syntax error at all — a **logical** mistake, where the query runs successfully and returns *plausible-looking* results that don't actually answer the intended business question. This category of mistake is arguably **more dangerous** than a syntax error precisely because nothing alerts you that anything is wrong.

---

## 17.7 Wrong Filtering Logic

### Mistake 1: Operator Precedence in Mixed AND/OR

```sql
-- Likely NOT what was intended (AND binds tighter than OR)
WHERE department = 'Engineering' OR department = 'Sales' AND salary > 100000;

-- CORRECT, explicit intent
WHERE (department = 'Engineering' OR department = 'Sales') AND salary > 100000;
```

**Root cause:** **Module 9, Section 9.3**.

### Mistake 2: BETWEEN on Timestamps Missing Same-Day Later Times

```sql
-- Misses rows timestamped later on Jan 31st (e.g., 2026-01-31 18:00:00)
WHERE order_date BETWEEN '2026-01-01' AND '2026-01-31';

-- CORRECT — half-open interval
WHERE order_date >= '2026-01-01' AND order_date < '2026-02-01';
```

**Root cause:** **Module 9, Section 9.4**.

### Mistake 3: Wrapping an Indexed Column in a Function

```sql
-- Can prevent standard index usage on `name`
WHERE UPPER(name) = 'ALICE';
```

**Root cause:** **Module 10, Section 10.12** — a performance mistake rather than a correctness one, but genuinely common and worth flagging again here.

---

## 17.8 SQL Injection (Preview)

**What it is:** A security vulnerability where **untrusted user input is concatenated directly into a SQL query string**, allowing an attacker to inject their own SQL logic and manipulate the query's behavior — potentially reading, modifying, or deleting data far beyond what the application intended to expose.

```
-- Conceptual illustration (NOT executable SQL — illustrating the vulnerability pattern):

Application code builds a query like:
   "SELECT * FROM users WHERE username = '" + userInput + "'"

If userInput is the string:  ' OR '1'='1
The resulting query becomes:
   SELECT * FROM users WHERE username = '' OR '1'='1'

Since '1'='1' is always TRUE, this returns EVERY row in the users table,
completely bypassing the intended username filter.
```

> ⚠️ **This is intentionally a brief, conceptual preview, not a deep security module** — full SQL injection prevention (parameterized queries/prepared statements, input validation, least-privilege database accounts, ORM-level protections) is properly a topic for application security and backend engineering practice more broadly, sitting somewhat outside this handbook's core SQL-language scope. The **single most important, universally-agreed-upon defense** is: **never build SQL queries by directly concatenating untrusted user input into query strings — always use parameterized queries/prepared statements**, which separate the query's structure from user-supplied data at the database driver level, making this entire class of attack structurally impossible regardless of what the input contains.

```sql
-- Vulnerable pattern (illustrative, application-layer pseudocode):
query = "SELECT * FROM users WHERE username = '" + user_input + "'"

-- Safe pattern — parameterized query (using placeholder syntax, driver-dependent):
query = "SELECT * FROM users WHERE username = %s"  -- the driver safely binds user_input separately,
                                                      -- never treating it as executable SQL syntax
```

> 💡 **Industry usage / interview relevance:** SQL injection remains one of the most consistently-cited vulnerabilities in real-world security incidents and is an extremely common, near-universal topic in both backend engineering and security-focused interviews — every backend engineer, regardless of specialization, is expected to know the parameterized-query defense at a minimum.

---

## 17.9 Consolidated "Debugging Checklist" for Unexpected Query Results

```
Query returns UNEXPECTEDLY ZERO rows?
  □ Check for `= NULL` instead of `IS NULL` (Section 17.1)
  □ Check for `NOT IN` with a NULL-containing list (Section 17.1)
  □ Check if a LEFT JOIN's WHERE clause accidentally filters out NULLs (Section 17.2)

Query returns UNEXPECTEDLY MANY / duplicate rows?
  □ Check for a missing join condition (Cartesian product, Section 17.5)
  □ Check for a legitimate one-to-many join fan-out being misinterpreted as a bug (Section 17.4)

Query fails with a GROUP BY error?
  □ Check every non-aggregated SELECT column is listed in GROUP BY (Section 17.3)

Query returns PLAUSIBLE BUT WRONG results (no error at all)?
  □ Check GROUP BY granularity matches the actual intended business question (Section 17.6)
  □ Check AND/OR operator precedence (Section 17.7)
  □ Check date range boundaries (BETWEEN inclusivity, Section 17.7)
```

---

## 17.10 Advantages of Studying Common Mistakes Directly

| Benefit | Explanation |
|---|---|
| Faster real-world debugging | Recognizing these patterns immediately, rather than re-deriving them from scratch every time |
| Stronger interview performance | "What's wrong with this query?" is one of the most common SQL interview formats, and this module directly prepares for it |
| Prevention over cure | Internalizing *why* each mistake happens (not just memorizing the fix) builds the instinct to avoid them proactively |

## 17.11 Interview Importance

| Level | Typical Questions |
|---|---|
| Placement | "What's wrong with this query?" (given a flawed example from any category above) |
| Mid-level | "Why would a `LEFT JOIN` behave like an `INNER JOIN`?" |
| Senior SDE-2/SDE-3 | "A report's numbers look plausible but are subtly wrong — walk me through your debugging process." (Often intentionally tests the Section 17.6-style "no error, wrong answer" category, which is the hardest to catch) |

---

## 17.12 Chapter Summary

This module is intentionally a **synthesis**, not new material — it re-organizes mistakes already explained in depth across Modules 6–13 into one mistake-first, debugging-oriented reference, plus a new preview of SQL injection as a foundational security awareness topic. The recurring theme across nearly every mistake: **`NULL`'s three-valued logic (Module 9) and the logical execution order (Module 15) are the root cause of the majority of "silent," no-error SQL bugs** — internalizing those two modules deeply is the highest-leverage investment for avoiding this entire category of problems.

---

## 17.13 Revision Notes / Cheat Sheet

```
Zero rows unexpectedly?      → = NULL, NOT IN + NULL, LEFT JOIN + WHERE-on-right-column
Too many/duplicate rows?     → missing join condition, misunderstood one-to-many fan-out
GROUP BY error?              → non-aggregated column missing from GROUP BY
Wrong (but plausible) result?→ GROUP BY granularity, AND/OR precedence, BETWEEN date boundaries
Security?                    → NEVER concatenate user input into SQL — use parameterized queries
```

---

## 17.14 Multiple Choice Questions (MCQs)

1. A `LEFT JOIN` silently behaves like an `INNER JOIN` most commonly because:
   a) `LEFT JOIN` is deprecated b) **A `WHERE` condition on the right table filters out `NULL` unmatched rows** c) `LEFT JOIN` requires an index d) MySQL doesn't support `LEFT JOIN`

2. What is the most common root cause of an unexpectedly massive result set?
   a) Too many columns selected b) **A missing or incorrect join condition (Cartesian product)** c) Using `ORDER BY` d) Using `LIMIT`

3. What's the recommended defense against SQL injection?
   a) Escaping quotes manually b) **Parameterized queries / prepared statements** c) Using `TRUNCATE` instead of `DELETE` d) Disabling `WHERE` clauses

4. A query with a subtly wrong `GROUP BY` granularity typically:
   a) Throws a syntax error b) **Runs successfully but answers the wrong business question** c) Always returns zero rows d) Cannot happen in PostgreSQL

5. Which is the safest way to handle "customers with no orders"?
   a) `NOT IN` with a subquery b) **`NOT EXISTS` or a `LEFT JOIN ... WHERE ... IS NULL` anti-join** c) `CROSS JOIN` d) `NATURAL JOIN`

---

## 17.15 Exercises

### 🟢 Beginner
1. Identify what's wrong with: `SELECT * FROM employees WHERE department_id = NULL;` and fix it.
2. Identify what's wrong with: `SELECT department, name FROM employees GROUP BY department;` and fix it.
3. Identify what's wrong with: `SELECT * FROM employees, departments;` and fix it.

### 🟡 Intermediate
4. A `LEFT JOIN` query intended to list all customers (including those with no orders) is silently excluding customers with no orders. Diagnose and fix, referencing Section 17.2.
5. A report shows implausibly high total revenue after joining `orders` with `order_items`. Explain the likely root cause (Section 17.4/17.5) and how you'd verify it.
6. Rewrite `WHERE department = 'Sales' OR department = 'HR' AND salary > 80000` to reflect the intended logic of "either department, but only above 80000 salary."

### 🔴 Advanced / Debugging
7. A monthly report's numbers are technically valid SQL output but don't match the business's expectations — no errors are thrown. Walk through the Section 17.9 debugging checklist to identify likely categories of issue to investigate first.
8. Explain, in an interview-answer style, why SQL injection is possible in the first place, and why parameterized queries structurally prevent it (not just "escape" the problem).

### 🧩 Scenario-Based
9. You're doing code review and see: `SELECT customer_id FROM orders WHERE customer_id NOT IN (SELECT customer_id FROM vip_customers);` Flag the risk here and provide the safer rewrite.

### 🏗️ Mini Project
10. Compile your own "top 10 SQL mistakes" reference card, in your own words, drawing from this module and citing which earlier module each mistake was first fully explained in — this becomes a genuinely useful, personalized interview-prep artifact.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [DataLemur](https://datalemur.com/)
- [StrataScratch](https://www.stratascratch.com/)

---

## ✅ Module 17 Complete

You now have a consolidated, debugging-oriented catalog of the most common real SQL mistakes — and, critically, an understanding of the *root causes* (three-valued logic, execution order, join semantics) that unify nearly all of them, rather than a disconnected list of trivia to memorize.

**Next up: Module 18 — SQL Interview Questions**, a dedicated bank of beginner, intermediate, and advanced questions, including output-prediction and scenario-based formats, spanning everything covered in Modules 1–17.

---

*Reply "continue" or "next module" when you're ready for Module 18.*
