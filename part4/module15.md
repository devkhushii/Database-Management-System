# The Complete DBMS Handbook — Part 4

## 📘 MODULE 15: INTERVIEW MASTERY

> **Continuing from Module 14 (Industry Case Studies).** This module is a dense, standalone **interview preparation reference** — consolidated question banks by topic, output-prediction drills, scenario-based system-design-adjacent questions, and a timed mock interview round, drawing on every module in this Part.

---

## 15.0 Chapter Roadmap

```
Module 15: Interview Mastery
│
├── 15.1 How Advanced SQL Interviews Are Actually Structured
├── 15.2 Frequently Asked Advanced SQL Questions (Conceptual Bank)
├── 15.3 Window Function Question Bank
├── 15.4 CTE Question Bank
├── 15.5 Ranking Question Bank
├── 15.6 Recursive SQL Question Bank
├── 15.7 Query Optimization Basics — Interview Framing
├── 15.8 Output Prediction Drills
├── 15.9 Scenario-Based Questions
├── 15.10 Real Interview Coding Round (Timed Mock)
└── 15.11 Final Cheat Sheet — All of Part 4
```

---

## 15.1 How Advanced SQL Interviews Are Actually Structured

Most FAANG/Senior Backend/Database Engineer SQL rounds follow a predictable shape:

```
1. WARM-UP (5 min): a simple JOIN/WHERE/GROUP BY question — checks basic fluency.
2. CORE PROBLEM (20-25 min): a "Nth highest," "top-N per group," "running total,"
   or "gap/island" style problem — almost always solvable with window functions
   and/or CTEs. The interviewer wants to see:
     - Can you recognize the PATTERN (Module 14's core skill)?
     - Do you write clean, decomposed SQL (Module 12)?
     - Do you handle edge cases (ties, NULLs, empty results) unprompted (Module 12 §12.6-12.7)?
3. FOLLOW-UP / EXTENSION (5-10 min): "now what if there are ties?", "what if this
   needs to run on 1 billion rows?", "how would you verify this is fast?" —
   tests depth (Modules 5, 9, 11, 13).
4. THEORY / DESIGN QUESTIONS (5-10 min, more senior roles): RA vs Calculus,
   View vs Materialized View, Trigger design tradeoffs, WHERE vs HAVING internals —
   tests fundamentals (Modules 1, 2, 8, 10).
```

**The single highest-leverage interview skill this entire Part has built:** rapidly mapping a business question to the right SQL construct (Module 14) — not memorizing syntax.

---

## 15.2 Frequently Asked Advanced SQL Questions (Conceptual Bank)

*(Rapid-fire format — answer each in under 60 seconds, as you would in an interview.)*

1. What is the difference between `WHERE` and `HAVING`? *(Module 1 §1.11, Module 6 §6.7)*
2. Why is `NOT EXISTS` preferred over `NOT IN`? *(Module 1 §1.8.6)*
3. What is the difference between a correlated and non-correlated subquery? *(Module 3 §3.2)*
4. What does `EXISTS` return if the subquery returns rows with all-NULL columns? *(TRUE — EXISTS only checks row existence, not content, Module 3 §3.3)*
5. What is the difference between `RANK()`, `DENSE_RANK()`, and `ROW_NUMBER()`? *(Module 5 §5.4)*
6. What is the default window frame when `ORDER BY` is used inside `OVER()`? *(Module 5 §5.7)*
7. What is the difference between a View and a Materialized View? *(Module 8 §8.3)*
8. Why can't `CHECK` constraints replace triggers for cross-row business rules? *(Module 10 §10.8)*
9. What is the "mutating table problem"? *(Module 10 §10.9)*
10. What is the difference between `ROLLUP` and `CUBE`? *(Module 6 §6.3-6.4)*
11. Why does `SELECT DISTINCT dept FROM Employees` sometimes surprise people coming from plain SQL to RA theory? *(Bag vs Set semantics, Module 1 §1.3)*
12. What is Codd's Theorem? *(Module 2 §2.7)*
13. What's the difference between a Stored Procedure and a Function? *(Module 9 §9.1)*
14. Why should you avoid row-by-row cursor loops? *(Module 9 §9.7)*
15. What is a fixpoint, in the context of recursive CTEs? *(Module 4 §4.4)*

---

## 15.3 Window Function Question Bank

1. Write a query to find the 3rd highest salary overall (not per department).
```sql
SELECT DISTINCT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS drnk FROM Employees
) t WHERE drnk = 3;
```
2. Write a query showing, for each order, the number of days since the customer's previous order.
```sql
SELECT customer_id, order_date,
       order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS days_since_last
FROM Orders;
```
3. Write a query to find the longest streak of consecutive days a user was active (classic "gaps and islands" problem).
```sql
WITH Numbered AS (
    SELECT user_id, activity_date,
           activity_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date))::int AS grp
    FROM UserActivity
)
SELECT user_id, MIN(activity_date) AS streak_start, MAX(activity_date) AS streak_end,
       COUNT(*) AS streak_length
FROM Numbered
GROUP BY user_id, grp
ORDER BY streak_length DESC
LIMIT 1;
```
> 🎯 **The "gaps and islands" pattern (subtracting a `ROW_NUMBER()` from a date to find contiguous groups) is one of the single most-tested advanced window-function patterns in FAANG interviews.** The trick: for consecutive dates, `date - row_number()` is **constant** within each streak, giving you a free grouping key.
4. Compute each employee's salary percentile within their department.
```sql
SELECT name, dept_id, salary,
       PERCENT_RANK() OVER (PARTITION BY dept_id ORDER BY salary) AS salary_percentile
FROM Employees;
```
5. Find the second-largest order amount for each customer, returning NULL if they have fewer than 2 orders.
```sql
SELECT customer_id, second_largest FROM (
    SELECT customer_id, amount AS second_largest,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rn
    FROM Orders
) t WHERE rn = 2;
-- Customers with <2 orders simply won't appear — LEFT JOIN back to a full customer list if NULL rows are required
```

---

## 15.4 CTE Question Bank

1. Write a recursive CTE to compute factorial(n) using pure SQL recursion (an unusual but real "test your recursive CTE understanding" question).
```sql
WITH RECURSIVE Factorial AS (
    SELECT 1 AS n, 1 AS fact
    UNION ALL
    SELECT n + 1, fact * (n + 1)
    FROM Factorial
    WHERE n < 10
)
SELECT * FROM Factorial;
```
2. Using multiple chained CTEs, find departments where average salary increased for 3 consecutive years.
```sql
WITH YearlyDeptAvg AS (
    SELECT dept_id, EXTRACT(YEAR FROM salary_date) AS yr, AVG(salary) AS avg_sal
    FROM SalaryHistory
    GROUP BY dept_id, EXTRACT(YEAR FROM salary_date)
),
WithTrend AS (
    SELECT dept_id, yr, avg_sal,
           LAG(avg_sal, 1) OVER (PARTITION BY dept_id ORDER BY yr) AS prev1,
           LAG(avg_sal, 2) OVER (PARTITION BY dept_id ORDER BY yr) AS prev2
    FROM YearlyDeptAvg
)
SELECT dept_id, yr
FROM WithTrend
WHERE avg_sal > prev1 AND prev1 > prev2;
```
3. Explain, without writing SQL, why a recursive CTE is the right tool for "find all sub-categories under Electronics," but a simple `JOIN` is NOT sufficient. *(Unbounded depth — Module 4 §4.2, contrasted with Module 14 §14.6 Problem 3's fixed-depth case.)*
4. Write a recursive CTE-based Fibonacci sequence generator (first 10 terms).
```sql
WITH RECURSIVE Fib AS (
    SELECT 1 AS n, 0 AS a, 1 AS b
    UNION ALL
    SELECT n + 1, b, a + b FROM Fib WHERE n < 10
)
SELECT n, a AS fibonacci_value FROM Fib;
```

---

## 15.5 Ranking Question Bank

1. Write a query to assign each customer a loyalty tier (Gold/Silver/Bronze) based on which spending third (tercile) they fall into.
```sql
SELECT customer_id, total_spent,
       CASE NTILE(3) OVER (ORDER BY total_spent DESC)
           WHEN 1 THEN 'Gold' WHEN 2 THEN 'Silver' ELSE 'Bronze'
       END AS tier
FROM CustomerSpending;
```
2. Find employees who rank in the top 10% of earners company-wide.
```sql
SELECT name, salary FROM (
    SELECT name, salary, PERCENT_RANK() OVER (ORDER BY salary DESC) AS pr FROM Employees
) t WHERE pr <= 0.10;
```
3. Explain the exact output difference between `RANK()` and `ROW_NUMBER()` for the input salaries `[90, 90, 90, 80]` ordered descending.
```
RANK():       1, 1, 1, 4
ROW_NUMBER(): 1, 2, 3, 4
```
4. Write a query ranking products within category by revenue, but only showing rank 1 (the category leader).
```sql
SELECT * FROM (
    SELECT category, product_name, revenue,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS rn
    FROM ProductRevenue
) t WHERE rn = 1;
```

---

## 15.6 Recursive SQL Question Bank

1. Given a `Categories(id, name, parent_id)` table, write a recursive CTE returning each category's full breadcrumb path AND its depth.
```sql
WITH RECURSIVE CatPath AS (
    SELECT id, name, parent_id, name::text AS path, 1 AS depth
    FROM Categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, cp.path || ' > ' || c.name, cp.depth + 1
    FROM Categories c
    JOIN CatPath cp ON c.parent_id = cp.id
)
SELECT * FROM CatPath ORDER BY depth, path;
```
2. Write a recursive CTE to detect if a given `Employees(emp_id, manager_id)` table contains a management cycle (a data integrity bug).
```sql
WITH RECURSIVE ChainCheck AS (
    SELECT emp_id, manager_id, ARRAY[emp_id] AS path, false AS has_cycle
    FROM Employees
    UNION ALL
    SELECT e.emp_id, e.manager_id, cc.path || e.emp_id, e.emp_id = ANY(cc.path)
    FROM Employees e
    JOIN ChainCheck cc ON e.manager_id = cc.emp_id
    WHERE NOT cc.has_cycle
)
SELECT DISTINCT emp_id FROM ChainCheck WHERE has_cycle = true;
```
3. Explain the exact stopping condition of a recursive CTE (in your own words, without looking it up). *(Module 4 §4.4 — fixpoint: iteration produces zero new rows.)*
4. Given a `BillOfMaterials(parent_part, child_part, quantity)` table, write a recursive CTE computing the total quantity of the most granular sub-component needed to build 1 unit of a top-level product.
```sql
WITH RECURSIVE Explosion AS (
    SELECT parent_part, child_part, quantity::numeric AS total_qty
    FROM BillOfMaterials WHERE parent_part = 'FinishedProduct'
    UNION ALL
    SELECT bom.parent_part, bom.child_part, e.total_qty * bom.quantity
    FROM BillOfMaterials bom
    JOIN Explosion e ON bom.parent_part = e.child_part
)
SELECT child_part, SUM(total_qty) AS total_required
FROM Explosion
GROUP BY child_part;
```

---

## 15.7 Query Optimization Basics — Interview Framing

**A common interview prompt:** *"This query is slow. Walk me through how you'd diagnose it."*

**Expected answer structure (a strong, structured response):**
```
1. Run EXPLAIN ANALYZE (Module 11 §11.8) — compare ESTIMATED vs ACTUAL row counts.
   Large divergence → stale statistics (Module 11 §11.4) → run ANALYZE.

2. Check access methods — Seq Scan on a large, highly-filtered table?
   → Is there a missing index? Is the predicate sargable (Module 1 §1.5.1)?

3. Check join strategy — Nested Loop on large tables where a Hash Join
   would be expected? → investigate why the optimizer chose it (stats, join
   order, missing index on join key).

4. Check for classic anti-patterns (Module 13):
   - Correlated subquery in SELECT on a large table → rewrite as JOIN+GROUP BY
   - Cartesian explosion from a missing join condition
   - Unnecessary UNION/DISTINCT where UNION ALL would suffice
   - Non-sargable predicate (function-wrapped indexed column)

5. Check for join fan-out inflating intermediate row counts before aggregation.

6. Consider whether a Materialized View (Module 8) is appropriate if this
   exact expensive query is run frequently and some staleness is tolerable.
```

**This exact structured-diagnosis answer is a very strong signal in senior interviews** — it demonstrates you don't just "know SQL syntax" but reason like an engineer about performance root-causing.

---

## 15.8 Output Prediction Drills

*(Given the query, predict the exact output — the single most common interview question FORMAT across all topics.)*

**Drill 1:**
```sql
SELECT salary, RANK() OVER (ORDER BY salary DESC) AS r
FROM (VALUES (100), (100), (90), (80)) AS t(salary);
```
<details><summary>Answer</summary>

| salary | r |
|---|---|
| 100 | 1 |
| 100 | 1 |
| 90 | 3 |
| 80 | 4 |
</details>

**Drill 2:**
```sql
SELECT COUNT(*) FROM (SELECT NULL AS x UNION SELECT NULL AS x) t;
```
<details><summary>Answer</summary>

**1** — two NULLs are treated as equal/duplicate for `UNION` deduplication purposes (Module 7 §7.7).
</details>

**Drill 3:**
```sql
SELECT * FROM Employees WHERE salary NOT IN (SELECT salary FROM Employees WHERE salary IS NULL);
```
<details><summary>Answer</summary>

**Zero rows** — if the subquery returns at least one `NULL` (guaranteed here, since it filters `WHERE salary IS NULL`, assuming at least one such row exists), `NOT IN` evaluates to `UNKNOWN` for every row, excluding everything (Module 1 §1.8.6). If NO row has `salary IS NULL`, the subquery returns zero rows, and `NOT IN (empty set)` is `TRUE` for everyone — so the answer critically depends on whether such a NULL row actually exists.
</details>

**Drill 4:**
```sql
SELECT dept_id, SUM(salary) FROM Employees GROUP BY ROLLUP(dept_id);
```
<details><summary>Answer</summary>

One row per distinct `dept_id` with that department's salary sum, **PLUS one extra row** with `dept_id = NULL` containing the grand total across all departments (Module 6 §6.3).
</details>

**Drill 5:**
```sql
SELECT LAST_VALUE(salary) OVER (ORDER BY salary) FROM Employees LIMIT 1;
```
<details><summary>Answer</summary>

**The current row's own salary value** (for whichever row is returned) — NOT the true maximum — because the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (Module 5 §5.6.2). This is the classic `LAST_VALUE` gotcha.
</details>

---

## 15.9 Scenario-Based Questions

1. **"Design a query (or set of queries) to detect duplicate customer records that might be the same person with slightly different data entry (e.g., typos in email)."** *(Discuss: exact match on normalized email via a `BEFORE INSERT` trigger normalizing case/whitespace (Module 10 §10.3), fuzzy matching considerations beyond pure SQL scope, self-join with `<` idiom from Module 14.)*

2. **"You need a dashboard showing real-time inventory levels, but the underlying query joins 6 tables and takes 8 seconds. The business says 2-second-stale data is acceptable. What do you build?"** *(Expected answer: Materialized View, Module 8 §8.2, with a documented refresh cadence — directly recall the View vs Materialized View tradeoff.)*

3. **"A junior engineer's stored procedure to apply a bulk 5% raise uses a cursor and takes 40 minutes on 500k employees. How do you fix it?"** *(Expected answer: replace the cursor loop with a single set-based `UPDATE`, Module 9 §9.7 — near-instant by comparison.)*

4. **"How would you enforce 'an order's total must equal the sum of its line items' as a database-level guarantee, not just an application check?"** *(Expected answer: a trigger, since this is a cross-row/cross-table business rule that a `CHECK` constraint cannot express, Module 10 §10.8.)*

5. **"Explain how you'd build a 'friends within 3 hops' feature for a social network with 100 million users, and why naive unbounded recursion might be dangerous here."** *(Expected discussion: recursive CTE with depth cap AND cycle protection, Module 4 §4.7; at very large scale, discuss whether this belongs in the OLTP database at all vs a specialized graph store — a good senior candidate raises this architectural question.)*

---

## 15.10 Real Interview Coding Round (Timed Mock — 30 Minutes)

**Instructions:** Set a 30-minute timer. Attempt all parts before checking the solution sketch below.

**Schema:**
```sql
Employees(emp_id, name, dept_id, manager_id, salary, hire_date)
Departments(dept_id, dept_name)
```

**Part 1 (5 min):** Find the 2nd highest salary in each department.

**Part 2 (10 min):** For each department, find the employee with the longest tenure (earliest `hire_date`), and include the department's total headcount alongside each result row.

**Part 3 (10 min):** Write a recursive CTE showing the full management chain (as a breadcrumb string) for every employee, from the CEO down.

**Part 4 (5 min, discussion only — no SQL required):** Your Part 1 query works but is slow on a 10-million-row `Employees` table with no index on `(dept_id, salary)`. Explain what you'd check and what index you'd propose.

<details>
<summary><strong>Solution Sketch</strong></summary>

**Part 1:**
```sql
SELECT dept_id, name, salary FROM (
    SELECT dept_id, name, salary,
           DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS drnk
    FROM Employees
) t WHERE drnk = 2;
```

**Part 2:**
```sql
WITH DeptHeadcount AS (
    SELECT dept_id, COUNT(*) AS headcount FROM Employees GROUP BY dept_id
),
Ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY hire_date ASC) AS rn
    FROM Employees
)
SELECT r.dept_id, r.name, r.hire_date, dh.headcount
FROM Ranked r
JOIN DeptHeadcount dh ON r.dept_id = dh.dept_id
WHERE r.rn = 1;
```

**Part 3:**
```sql
WITH RECURSIVE Chain AS (
    SELECT emp_id, name, manager_id, name::text AS breadcrumb
    FROM Employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.name, e.manager_id, c.breadcrumb || ' > ' || e.name
    FROM Employees e JOIN Chain c ON e.manager_id = c.emp_id
)
SELECT * FROM Chain;
```

**Part 4 (discussion):** Run `EXPLAIN ANALYZE` first to confirm a `Seq Scan` is indeed occurring and where the time actually goes (Module 11). Propose a **composite index on `(dept_id, salary DESC)`** — this directly supports the `PARTITION BY dept_id ORDER BY salary DESC` window specification, potentially letting the engine avoid a separate sort step entirely. Confirm the improvement by re-running `EXPLAIN ANALYZE` after adding the index (full indexing depth is in Part 8, but recognizing the composite-index-matches-the-window-spec idea is a strong senior-level signal here).

</details>

---

## 15.11 Final Cheat Sheet — All of Part 4

```
┌───────────────────────────────────────────────────────────────────────┐
│                    PART 4 MASTER CHEAT SHEET (Modules 1–15)                │
├───────────────────────────┬───────────────────────────────────────────┤
│ M1  Relational Algebra       │ σ π ρ ∪ ∩ − × ⋈ ÷ ← | Every join = σ(R×S) │
│ M2  Relational Calculus         │ TRC/DRC, ∃/∀, safety, Codd's Theorem     │
│ M3  Advanced SQL                  │ Correlated subq, EXISTS, ANY/ALL,       │
│                                     │ LATERAL, derived tables                  │
│ M4  CTEs                             │ WITH, WITH RECURSIVE, fixpoint,          │
│                                        │ cycle protection                          │
│ M5  Window Functions                    │ OVER, ROW_NUMBER/RANK/DENSE_RANK,        │
│                                           │ LAG/LEAD, ROWS/RANGE/GROUPS frames         │
│ M6  Advanced Aggregation                    │ GROUPING SETS, ROLLUP (N+1),               │
│                                               │ CUBE (2^N), GROUPING(), FILTER              │
│ M7  Set Operations                              │ UNION/UNION ALL/INTERSECT/EXCEPT;           │
│                                                   │ NULL=NULL in dedup context                   │
│ M8  Database Objects                                │ View (live) vs Materialized View            │
│                                                       │ (stale, cached); Sequences; Temp Tables      │
│ M9  Procedures & Functions                              │ Procedure=workflow/DML;                     │
│                                                           │ Function=must RETURN, usable in SELECT       │
│ M10 Triggers                                                │ BEFORE(validate)/AFTER(audit)/               │
│                                                                │ INSTEAD OF(updatable views); mutating table │
│ M11 Execution Internals                                          │ Parser→Rewrite→Optimizer→Plan→Engine       │
│                                                                     │ →Result; EXPLAIN ANALYZE for real stats      │
│ M12 Best Practices                                                    │ Decompose via CTEs; NOT EXISTS>NOT IN;      │
│                                                                          │ NULLIF(x,0); explicit JOIN...ON              │
│ M13 Common Pitfalls                                                        │ Fan-out, LEFT JOIN+WHERE bug,               │
│                                                                               │ Cartesian explosion, avg-of-avgs fallacy     │
│ M14 Case Studies                                                                │ Pattern recognition across 8 domains        │
│ M15 Interview Mastery                                                              │ Structure, question banks, mock round      │
└───────────────────────────┴───────────────────────────────────────────┘
```

---

## Chapter Summary

This module closes out Part 4 by converting everything you've learned into **interview-ready fluency**: rapid conceptual recall (Section 15.2), topic-specific question banks with worked solutions (Sections 15.3–15.6), a structured framework for query-optimization discussions (Section 15.7), output-prediction drills that mirror the most common real interview question format (Section 15.8), scenario-based system-design-adjacent questions (Section 15.9), and a full timed mock round (Section 15.10). **The Final Cheat Sheet (Section 15.11) consolidates all 15 modules of this Part onto a single page** — use it as your last-minute review reference before any Advanced SQL interview.

---

## Preview of Part 5

Part 5 builds directly on this Part's relational foundations to cover:
- **Functional Dependencies** — the formal theory of "which columns determine which other columns," the mathematical basis for everything that follows
- **Normalization** — 1NF through BCNF, 4NF, 5NF, and the precise anomalies each normal form eliminates
- **Schema Design** — translating business requirements into a normalized (or deliberately denormalized) relational schema
- **Database Design** — the full ER-to-relational mapping process, revisited with the advanced query patterns from this Part in mind
- **Dependency Theory** — multivalued dependencies, join dependencies, and the theoretical limits of decomposition

---

> ✅ **Module 15 Complete — Part 4 is now fully finished.**
>
> This concludes **Part 4: Relational Algebra, Relational Calculus, Advanced SQL, CTEs, Window Functions, Recursive Queries, Advanced Query Writing, and SQL Optimization Fundamentals** — 15 modules covering every topic specified, from absolute-beginner explanations through Senior SDE-2/SDE-3 interview depth.
>
> Reply **"Part 5"**, **"next part"**, or **"continue"** to begin **Part 5: Functional Dependencies, Normalization, Schema Design, and Database Design** — following the same module-by-module, wait-for-confirmation format established in this Part.
