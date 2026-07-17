# Part 3 — The Complete SQL Handbook
## Module 13: Subqueries

> **Goal of this module:** Master queries nested inside other queries — scalar, row, table, correlated, and nested subqueries — and revisit `EXISTS`/`NOT EXISTS` in full query-writing context. Subqueries let you express genuinely complex, multi-step logical questions as a single SQL statement.

---

### 📍 Where This Module Fits

```
Module 12: Combining tables horizontally (joins)  →  Module 13: Nesting queries (You are here)  →  Module 14: Views
```

---

## 13.1 What Is a Subquery?

A **subquery** (or **inner query**) is a `SELECT` statement nested inside another SQL statement — appearing anywhere an expression, table, or value is expected: inside `SELECT`'s column list, inside `WHERE`/`HAVING`, inside `FROM`, or even inside `INSERT`/`UPDATE`/`DELETE` (as seen already in **Module 7**'s `INSERT ... SELECT`).

```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);   -- the parenthesized part is the subquery
```

> 🧠 **Real-life analogy:** A subquery is like a **nested question inside a bigger question** — "who earns more than [the average salary]?" first requires answering the bracketed sub-question ("what *is* the average salary?") before the outer question can even be evaluated. SQL handles this nesting exactly the way you'd naturally reason through it in English.

### Why Subqueries Exist

They let you express **multi-step logical dependencies** — "compare against a computed value," "check against a dynamically-determined set," "filter based on another table's aggregate" — that would otherwise require multiple separate round-trip queries orchestrated by application code, losing both performance (extra round-trips) and atomicity (the data could change between the separate queries).

---

## 13.2 Scalar Subqueries

**What it is:** A subquery that returns **exactly one row and one column** — a single value — usable anywhere a literal value would be valid.

```sql
SELECT name, salary,
       (SELECT AVG(salary) FROM employees) AS company_avg_salary
FROM employees;

SELECT name FROM employees
WHERE department_id = (SELECT id FROM departments WHERE name = 'Engineering');
```

> ⚠️ **Critical, frequently-tested runtime error:** If a scalar subquery unexpectedly returns **more than one row**, the engine raises a runtime error (`more than one row returned by a subquery used as an expression`, in PostgreSQL) — it does **not** silently pick one arbitrarily or concatenate results. This is a genuinely common real bug: a subquery written assuming uniqueness (e.g., "the department named 'Engineering'") breaks the moment that assumption is violated (e.g., two departments accidentally share that name).

```sql
-- If TWO departments are named 'Engineering', this query FAILS at runtime:
SELECT name FROM employees
WHERE department_id = (SELECT id FROM departments WHERE name = 'Engineering');
-- ERROR: more than one row returned by a subquery used as an expression
```

---

## 13.3 Row Subqueries

**What it is:** A subquery returning a **single row with multiple columns** — compared against a matching row-constructor expression using standard comparison operators.

```sql
SELECT * FROM employees
WHERE (department_id, salary) = (SELECT id, max_dept_salary FROM department_salary_caps WHERE id = 10);
```

**PostgreSQL and MySQL both support row-value comparisons** using the `(col1, col2) = (val1, val2)` row-constructor syntax — a genuinely elegant way to compare multiple columns simultaneously without a verbose chain of `AND`ed equality checks.

```sql
-- Equivalent, but more verbose:
WHERE department_id = <val1> AND salary = <val2>
```

---

## 13.4 Table Subqueries (Derived Tables)

**What it is:** A subquery used in the `FROM` clause, producing an entire **result set treated as if it were a table** — commonly called a **derived table**.

```sql
SELECT dept_summary.department, dept_summary.avg_salary
FROM (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) AS dept_summary
WHERE dept_summary.avg_salary > 70000;
```

> ⚠️ **Both PostgreSQL and MySQL require an alias for a derived table** (`AS dept_summary` above) — omitting it is a syntax error in both engines, a small but real, commonly-hit gotcha for beginners copying subquery examples without the alias.

### Why Use a Derived Table Instead of HAVING?

This specific example could arguably use **Module 11's `HAVING`** instead:

```sql
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 70000;
```

Both produce identical results here — but derived tables become genuinely necessary (not just stylistic) once you need to **further join, filter, or aggregate** an already-aggregated result in ways `HAVING` alone can't express, or when you need to reuse a complex computed result set multiple times within one larger query.

> 💡 **Preview of Part 4:** **Common Table Expressions (CTEs)**, using the `WITH` keyword, are a more readable, often-preferred modern alternative to deeply nested derived-table subqueries — covered in full depth in Part 4, but worth knowing they exist as you encounter increasingly complex subquery nesting in this module.

---

## 13.5 Correlated Subqueries

**What it is:** A subquery that **references a column from the outer query** — meaning it cannot be evaluated independently; it must be re-evaluated **once for every row** the outer query processes.

```sql
SELECT e.name, e.salary, e.department_id
FROM employees e
WHERE e.salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id   -- ← references the OUTER query's e.department_id!
);
```

**What this query answers:** "Show employees who earn more than **their own department's** average salary" — genuinely different from `Module 13.2`'s example, which compared against the **single, company-wide** average for every row.

### Internal Working — Why Correlated Subqueries Can Be Expensive

```
Non-correlated (independent) subquery:          Correlated subquery:
  Executed ONCE, result reused for every            Conceptually re-executed ONCE PER
  row of the outer query                            OUTER ROW, using that row's specific
                                                      correlated value each time

  ┌─────────────┐                                   outer row 1 → run subquery (dept=10)
  │ Run subquery │──► single result                  outer row 2 → run subquery (dept=10)
  │  ONCE        │    reused everywhere               outer row 3 → run subquery (dept=20)
  └─────────────┘                                     outer row 4 → run subquery (dept=30)
                                                        ... (conceptually — the real optimizer
                                                             often rewrites this into a more
                                                             efficient join/aggregate internally)
```

> ⚠️ **Genuinely important performance nuance for interviews:** Naively, a correlated subquery sounds like it must literally re-run once per outer row — which would be catastrophically slow on large tables (an **N+1-style** problem, conceptually similar to a well-known application-layer anti-pattern). In practice, **modern PostgreSQL and MySQL query optimizers are often smart enough to automatically rewrite many correlated subqueries into an equivalent, much more efficient JOIN or semi-join execution plan** internally — but this rewrite isn't guaranteed for every query shape, and understanding the *conceptual* "once per row" cost model is essential both for reasoning about correctness and for recognizing when a query might need to be manually rewritten (often as an explicit `JOIN`, Module 12) for better, more predictable performance. **Always verify with `EXPLAIN`** (Part 4) rather than assuming either "correlated subqueries are always slow" or "the optimizer always saves you."

### Correlated EXISTS — Revisiting Module 9's Safe NULL-Handling Pattern

```sql
SELECT d.name
FROM departments d
WHERE EXISTS (
    SELECT 1 FROM employees e WHERE e.department_id = d.id AND e.salary > 100000
);
```

This is a **correlated** `EXISTS` subquery (referencing the outer `d.id`) — precisely the pattern introduced in **Module 9, Section 9.7**, now shown in its full, natural query-writing context: "departments that have at least one employee earning over 100,000."

---

## 13.6 Nested Subqueries (Subqueries Within Subqueries)

Subqueries can be **nested to multiple levels** — a subquery inside a subquery inside a subquery — though readability degrades quickly beyond two or three levels.

```sql
SELECT name FROM employees
WHERE department_id IN (
    SELECT id FROM departments
    WHERE region_id IN (
        SELECT id FROM regions WHERE country = 'India'
    )
);
```

> ⚠️ **Best practice, genuinely important for real code maintainability:** While SQL permits essentially unlimited nesting depth, **deeply nested subqueries become extremely difficult to read, debug, and reason about**. Once nesting exceeds roughly two or three levels, strongly consider refactoring using **CTEs** (`WITH` clauses, Part 4) — which let you name and sequence each logical step, dramatically improving readability without changing the query's actual result or, typically, its performance characteristics.

---

## 13.7 Subqueries in the SELECT List

```sql
SELECT
    d.name,
    (SELECT COUNT(*) FROM employees e WHERE e.department_id = d.id) AS headcount
FROM departments d;
```

**What it does:** A scalar subquery computed **once per outer row**, embedded directly as a computed column — genuinely useful, but worth comparing against the join-based alternative:

```sql
-- Often a more efficient, more idiomatic alternative to the SELECT-list subquery above:
SELECT d.name, COUNT(e.id) AS headcount
FROM departments d
LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.name;
```

> 💡 **General guidance, worth internalizing:** When a correlated scalar subquery in the `SELECT` list is essentially computing "an aggregate of related rows," it's very often **rewritable as a `LEFT JOIN` + `GROUP BY`** (Modules 11–12) instead — and the join-based rewrite is frequently more efficient (letting the optimizer use a single combined pass rather than one subquery execution per outer row) and, once you're comfortable with joins, arguably just as readable. This is a genuinely common, real "how would you optimize this query" interview exercise.

---

## 13.8 EXISTS and NOT EXISTS — Full Query-Writing Recap

Building on **Module 9, Section 9.7**'s introduction, here's `EXISTS`/`NOT EXISTS` shown in complete, realistic query patterns:

```sql
-- Customers who HAVE placed at least one order
SELECT c.name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- Customers who have NEVER placed an order
SELECT c.name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

### EXISTS vs IN vs JOIN — When to Use Which

| Approach | Best When |
|---|---|
| `JOIN` (Module 12) | You need columns from **both** tables in the final result |
| `IN` with a subquery | Simple membership test against a **known-safe** (no `NULL`s), typically small/static list |
| `EXISTS`/`NOT EXISTS` | Testing **existence/non-existence** of a related row, especially when `NULL`s might be present in the related data (per **Module 9, Section 9.5**'s `NOT IN` trap) — the generally safest default for "does a match exist" logic |

```sql
-- These three approaches can express similar (though not always identical) logic:

-- JOIN — use when you need columns from BOTH tables
SELECT c.name, o.id FROM customers c JOIN orders o ON o.customer_id = c.id;

-- IN — simple, but risky with NULL-containing subqueries (Module 9)
SELECT c.name FROM customers c WHERE c.id IN (SELECT customer_id FROM orders);

-- EXISTS — the generally safest default for pure existence checks
SELECT c.name FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

---

## 13.9 Subqueries vs Joins — A Deeper Comparison

| Aspect | Subquery | Join |
|---|---|---|
| Result shape | Can collapse to a single value/existence check | Naturally produces combined rows from multiple tables |
| Readability for "does X exist" questions | Often clearer (`EXISTS`) | Requires `DISTINCT`/`GROUP BY` to avoid row duplication if not careful |
| Readability for "show combined data" questions | Awkward — subqueries can't easily project columns from both sides in one flat row | Natural fit |
| Optimizer flexibility | Modern optimizers often rewrite subqueries into join-equivalent plans internally | Direct, explicit |

> 💡 **A genuinely mature, senior-level perspective:** In modern PostgreSQL and MySQL, the **choice between a correlated subquery and an equivalent join is often more about readability and intent than raw performance**, since the optimizer frequently produces similar execution plans either way for logically-equivalent queries — but this isn't universally guaranteed, and profiling with `EXPLAIN` (Part 4) is always the definitive way to verify, rather than relying on a blanket rule of thumb.

---

## 13.10 Full Worked Example — Subqueries Combined

```sql
SELECT
    d.name AS department,
    (SELECT COUNT(*) FROM employees e WHERE e.department_id = d.id) AS headcount,
    (SELECT AVG(salary) FROM employees e WHERE e.department_id = d.id) AS avg_salary
FROM departments d
WHERE EXISTS (
    SELECT 1 FROM employees e WHERE e.department_id = d.id AND e.salary > (
        SELECT AVG(salary) FROM employees
    )
)
ORDER BY avg_salary DESC;
```

This deliberately layers: two **correlated scalar subqueries** in the `SELECT` list, a **correlated `EXISTS`** subquery in `WHERE`, and a **nested, non-correlated scalar subquery** (the company-wide average) inside that `EXISTS` — a realistic example of how these patterns compose in genuine reporting queries.

---

## 13.11 Advantages / Disadvantages Summary

| Aspect | Advantage | Watch-out |
|---|---|---|
| Subqueries generally | Express multi-step logic in one atomic statement | Deep nesting hurts readability — prefer CTEs beyond ~2-3 levels |
| Scalar subqueries | Clean, simple embedding of a single computed value | Runtime error if the subquery unexpectedly returns >1 row |
| Correlated subqueries | Powerful per-row, context-dependent logic | Conceptually "once per row" cost — verify actual performance with `EXPLAIN` |
| `EXISTS`/`NOT EXISTS` | Safe against `NULL`s, often efficient | Slightly less intuitive to read than `IN` for simple cases |

## 13.12 Best Practices

- Prefer `EXISTS`/`NOT EXISTS` over `IN`/`NOT IN` when the related data might contain `NULL`s (Module 9, reinforced here).
- Always alias derived tables (`FROM (subquery) AS alias`) — required syntax in both engines (Section 13.4).
- Consider rewriting correlated `SELECT`-list subqueries as `LEFT JOIN` + `GROUP BY` when performance matters (Section 13.7).
- Refactor deeply nested subqueries into CTEs for readability once nesting exceeds two or three levels (Section 13.6).
- Verify scalar subqueries are logically guaranteed to return at most one row, or wrap uniqueness with appropriate constraints/aggregates.

## 13.13 Common Mistakes

- Assuming a scalar subquery will always return exactly one row, then hitting a runtime error when that assumption breaks (Section 13.2).
- Forgetting to alias a derived table in `FROM` (Section 13.4).
- Confusing a correlated subquery's outer-reference with an unrelated, similarly-named column from a different table.
- Using `IN` with a subquery that could return `NULL`s, reintroducing **Module 9's** infamous `NOT IN` trap.

## 13.14 Edge Cases

- A correlated subquery can reference **any** column from any enclosing outer query, even across multiple levels of nesting — a powerful but easily-confusing capability that demands careful aliasing discipline.
- An `EXISTS` subquery's `SELECT` list is conventionally `SELECT 1` (or `SELECT *`) precisely **because it's ignored entirely** — only row *existence* matters, not the projected columns; some engines optimize this internally regardless of what's selected, but the `SELECT 1` convention signals intent clearly to human readers.
- Subqueries in `FROM` (derived tables) are, in some circumstances, subject to different optimizer treatment than logically-equivalent CTEs — PostgreSQL historically treated CTEs as strict "optimization fences" (opaque, always materialized) in older versions, though this changed in PostgreSQL 12+ to allow inlining — a nuance worth being aware of but covered in full only alongside CTEs in Part 4.

## 13.15 Performance Implications

- Scalar/correlated subqueries in the `SELECT` list execute conceptually once per outer row — for large result sets, this can be meaningfully slower than an equivalent join-based rewrite, though modern optimizers often narrow this gap significantly.
- `EXISTS` allows the optimizer to often **short-circuit** (stop at the first match) rather than computing a full result set, making it frequently efficient for existence checks even on large related tables.
- Deeply nested subqueries can sometimes defeat the optimizer's ability to reorder/simplify the overall query plan effectively — flatter, join-based, or CTE-based rewrites are often both more readable **and** more optimizer-friendly.

## 13.16 Industry Usage

Subqueries are used constantly in real reporting, analytics, and even everyday application queries — anywhere a filter or computed value depends on "some other, related piece of data" rather than a fixed literal. The `EXISTS`-based "does a related row exist" pattern specifically is one of the most common, genuinely universal query patterns in production backend systems.

## 13.17 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Definitions: scalar, table, correlated subqueries |
| Placement | "Write a query to find employees earning more than the average salary." |
| Mid-level | "What's the difference between a correlated and non-correlated subquery?" |
| Senior SDE-2/SDE-3 | "When would you rewrite a correlated subquery as a join, and why?" / "Explain why `EXISTS` is generally safer than `IN` with a subquery." |

---

## 13.18 Chapter Summary

- **Scalar subqueries** return one value; **row subqueries** return one multi-column row; **table subqueries** (derived tables) return an entire result set usable in `FROM`.
- **Correlated subqueries** reference the outer query and are conceptually evaluated once per outer row — modern optimizers often rewrite them efficiently, but this isn't guaranteed, so verify with `EXPLAIN`.
- **Nested subqueries** can go arbitrarily deep, but readability degrades quickly — prefer CTEs beyond a couple of levels.
- **`EXISTS`/`NOT EXISTS`** remain the safest default for existence checks, immune to `NULL`-related traps that affect `IN`/`NOT IN`.
- Correlated `SELECT`-list subqueries computing per-row aggregates are frequently rewritable (and often more efficient) as `LEFT JOIN` + `GROUP BY`.

---

## 13.19 Revision Notes / Cheat Sheet

```sql
-- Scalar subquery (must return exactly 1 row, 1 column)
WHERE salary > (SELECT AVG(salary) FROM employees)

-- Row subquery
WHERE (col1, col2) = (SELECT c1, c2 FROM t WHERE ...)

-- Table subquery / derived table (MUST be aliased)
FROM (SELECT department, AVG(salary) AS avg_sal FROM employees GROUP BY department) AS d

-- Correlated subquery (references outer query's columns)
WHERE salary > (SELECT AVG(salary) FROM employees e2 WHERE e2.dept_id = e.dept_id)

-- EXISTS / NOT EXISTS — safe existence checks, immune to NULL traps
WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)
WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)

⚠ Scalar subquery returning >1 row → runtime ERROR
⚠ Derived tables MUST be aliased
⚠ Prefer EXISTS over IN when subquery data may contain NULLs
⚠ Deep nesting → refactor to CTEs (Part 4) for readability
```

---

## 13.20 Multiple Choice Questions (MCQs)

1. What happens if a scalar subquery returns more than one row?
   a) Only the first row is used b) **A runtime error occurs** c) All rows are concatenated d) `NULL` is returned

2. A subquery that references a column from the outer query is called:
   a) A scalar subquery b) A row subquery c) **A correlated subquery** d) A derived table

3. Which is required when using a subquery in the `FROM` clause?
   a) `WHERE` clause b) **An alias for the derived table** c) `GROUP BY` d) `DISTINCT`

4. Why is `EXISTS` often preferred over `IN` with a subquery?
   a) `EXISTS` is always faster b) **`EXISTS` is immune to the `NULL`-related trap that affects `NOT IN`** c) `IN` cannot use subqueries d) `EXISTS` returns actual data, `IN` doesn't

5. A correlated scalar subquery computing per-row aggregates in the `SELECT` list is often better rewritten as:
   a) A `CROSS JOIN` b) **A `LEFT JOIN` + `GROUP BY`** c) A `NATURAL JOIN` d) `TRUNCATE`

---

## 13.21 Exercises

### 🟢 Beginner
1. Write a query finding all employees earning more than the company-wide average salary, using a scalar subquery.
2. Write a query using a derived table to compute average salary per department, then filter to departments above 70,000.
3. Write a correlated `EXISTS` subquery to find departments that have at least one employee.

### 🟡 Intermediate
4. Rewrite exercise 3 using `NOT EXISTS` to find departments with zero employees, and compare the result with **Module 12's** anti-join `LEFT JOIN ... WHERE ... IS NULL` pattern — confirm they produce the same result.
5. Write a correlated subquery finding employees who earn more than their own department's average salary (not the company-wide average).
6. Rewrite the correlated `SELECT`-list headcount-per-department subquery from Section 13.7 as an equivalent `LEFT JOIN` + `GROUP BY` query, and confirm both produce identical results.

### 🔴 Advanced / Debugging
7. A query `WHERE department_id = (SELECT id FROM departments WHERE name = 'Sales')` suddenly starts failing with "more than one row returned by a subquery." Diagnose the likely cause and propose two different fixes.
8. Explain, in your own words, why a correlated subquery's "once per outer row" execution model is a useful mental model for reasoning about correctness, even though the actual optimizer may execute it far more efficiently in practice.

### 🧩 Scenario-Based
9. You need to build a report showing, for every customer, their name and total number of orders — including customers with zero orders showing `0`, not being omitted. Would you use a correlated subquery, a `JOIN`, or both? Justify your choice.

### 🏗️ Mini Project
10. Using your practice schema, write a single comprehensive query combining: a scalar subquery, a correlated `EXISTS` subquery, and a derived table — solving a genuine, realistic reporting question you define yourself, with a one-sentence explanation of what the query answers.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [DataLemur](https://datalemur.com/)
- [StrataScratch](https://www.stratascratch.com/)

---

## ✅ Module 13 Complete

You can now express genuinely complex, multi-step logical questions as single SQL statements — scalar, row, table, correlated, and nested subqueries — and you understand when to prefer them over joins, and when the reverse is true.

**Next up: Module 14 — Views**, covering regular views, materialized views, updatable views, and view-based security patterns.

---

*Reply "continue" or "next module" when you're ready for Module 14.*
