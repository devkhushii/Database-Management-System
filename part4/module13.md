# The Complete DBMS Handbook — Part 4

## 📘 MODULE 13: COMMON SQL PITFALLS

> **Continuing from Module 12 (Best Practices).** Where Module 12 taught the *positive* discipline, this module is a focused, standalone **debugging catalog** — each pitfall is presented as **Symptom → Root Cause → Fix**, in the exact format you'd use to triage a real production bug or answer a "what's wrong with this query" interview question.

---

## 13.0 Chapter Roadmap

```
Module 13: Common SQL Pitfalls
│
├── 13.1 Duplicate Rows
├── 13.2 Incorrect Joins
├── 13.3 Wrong Aggregations
├── 13.4 Recursive Query Mistakes
├── 13.5 Window Function Misuse
├── 13.6 Cartesian Explosions
├── 13.7 Correlated Query Inefficiencies
├── 13.8 Master Debugging Checklist
├── 13.9 Interview Questions & MCQs
├── 13.10 Cheat Sheet
└── 13.11 Exercises
```

---

## 13.1 Duplicate Rows

### Pitfall A: Unexpected Duplicates from a Join

**Symptom:** A report that should show one row per customer shows the same customer multiple times.

**Root Cause:** Classic **join fan-out** (Module 12, Section 12.5) — joining a "one" table to a "many" table without aggregating the "many" side first.
```sql
-- Each customer with 3 orders appears 3 times
SELECT c.name, c.email
FROM Customers c
JOIN Orders o ON c.customer_id = o.customer_id;
```

**Fix:** Aggregate/deduplicate the "many" side before joining, or use `EXISTS` (Module 3) if you only need "has at least one order," not order-level detail.
```sql
SELECT c.name, c.email
FROM Customers c
WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.customer_id = c.customer_id);
```

### Pitfall B: Forgetting DISTINCT Isn't Free / Isn't Always the Right Fix

**Symptom:** Someone "fixes" Pitfall A by slapping `DISTINCT` on the whole `SELECT *` — it works, but is slow and masks the real structural issue.

**Root Cause:** `DISTINCT` requires a full sort/hash deduplication pass (Module 7, Section 7.6) over **every selected column** — treating a symptom instead of the actual join-grain mismatch.

**Fix:** Address the root cause (restructure the join/aggregation, per Pitfall A) rather than papering over it with `DISTINCT`.

---

## 13.2 Incorrect Joins

### Pitfall A: INNER JOIN Where LEFT JOIN Was Intended

**Symptom:** "Customers with zero orders" silently vanish from a report that was supposed to include them.

**Root Cause:** `INNER JOIN` only keeps rows with a match on **both** sides (Module 1, Section 1.8.2) — a customer with no matching `Orders` row is dropped entirely.
```sql
-- Customers with no orders disappear
SELECT c.name, COUNT(o.order_id) AS order_count
FROM Customers c
INNER JOIN Orders o ON c.customer_id = o.customer_id
GROUP BY c.name;
```

**Fix:** Use `LEFT JOIN` to preserve unmatched rows from the "preserved" side (Module 1, Section 1.8.4).
```sql
SELECT c.name, COUNT(o.order_id) AS order_count
FROM Customers c
LEFT JOIN Orders o ON c.customer_id = o.customer_id
GROUP BY c.name;
```

### Pitfall B: Filtering a LEFT JOIN's Right-Side Column in WHERE (Accidentally Converts It Back to an INNER JOIN)

**Symptom:** A `LEFT JOIN` was used specifically to preserve unmatched rows, but the unmatched rows still disappear.

**Root Cause:** Placing a condition on the **right-side (nullable) table's column** in `WHERE` filters out the very `NULL`-filled unmatched rows the `LEFT JOIN` was meant to keep — because `WHERE` runs *after* the join (Module 1, Section 1.11), and `col = value` on a `NULL` evaluates to `UNKNOWN`, excluding the row.
```sql
-- Silently behaves like an INNER JOIN!
SELECT c.name, o.order_date
FROM Customers c
LEFT JOIN Orders o ON c.customer_id = o.customer_id
WHERE o.status = 'COMPLETED';   -- ◀ excludes customers with NO orders (o.status is NULL for them)
```

**Fix:** Move the condition into the `ON` clause (so it's applied *during* the join, before unmatched rows are generated with NULLs) if you want to preserve all customers while still filtering which orders qualify:
```sql
SELECT c.name, o.order_date
FROM Customers c
LEFT JOIN Orders o ON c.customer_id = o.customer_id AND o.status = 'COMPLETED';
```

> 🎯 **This `ON` vs `WHERE` placement distinction on an outer join is one of the single most common, most subtle real-world SQL bugs — and a favorite "spot the bug" interview question.**

### Pitfall C: NATURAL JOIN Silently Joining on an Unintended Column

**Symptom:** A query using `NATURAL JOIN` returns fewer/different rows after an unrelated schema change (e.g., both tables gained an `updated_at` column).

**Root Cause:** `NATURAL JOIN` automatically joins on **every** identically-named column (Module 1, Section 1.8.3) — including columns never intended as part of the join key.

**Fix:** Never use `NATURAL JOIN` in production; always use explicit `JOIN ... ON`.

---

## 13.3 Wrong Aggregations

### Pitfall A: Aggregating Before Filtering (or Vice Versa) Incorrectly

**Symptom:** A "total revenue this quarter" figure includes data from all time.

**Root Cause:** Placing a date filter in `HAVING` instead of `WHERE` (or omitting it from `WHERE` and relying on incorrect logic elsewhere) — recall the logical execution order (Module 1, Section 1.11): `WHERE` filters rows *before* grouping; `HAVING` filters *groups* after.

**Fix:** Row-level filters (like a date range) belong in `WHERE`; only group-level conditions (like `COUNT(*) > 5`) belong in `HAVING`.

### Pitfall B: Confusing COUNT(*) with COUNT(column)

**Symptom:** A "how many customers have a phone number on file" count is wrong.

**Root Cause:** `COUNT(*)` counts **all rows**, regardless of `NULL`s; `COUNT(column)` counts only **non-NULL** values in that specific column (Module 12, Section 12.6) — using the wrong one silently produces an inflated or deflated count.
```sql
-- WRONG: counts ALL customers, not just those with a phone number
SELECT COUNT(*) FROM Customers;

-- CORRECT: counts only customers where phone IS NOT NULL
SELECT COUNT(phone) FROM Customers;
```

### Pitfall C: Averaging Already-Aggregated Data (Average of Averages)

**Symptom:** A "company-wide average order value" computed by averaging per-region averages doesn't match the true overall average.

**Root Cause:** `AVG(AVG(...))` (an "average of averages") is only correct if every group has the **exact same number of underlying rows** — otherwise it's mathematically wrong (it implicitly weights every region equally, regardless of how many orders each region actually had).
```sql
-- WRONG if regions have different order counts (classic mathematically-invalid pattern):
SELECT AVG(region_avg) FROM (
    SELECT region, AVG(order_value) AS region_avg FROM Orders GROUP BY region
) t;

-- CORRECT: compute the true overall average directly over all rows
SELECT AVG(order_value) FROM Orders;
```

> 🎯 **This "average of averages" fallacy is a genuinely important, easy-to-miss statistical/SQL correctness bug** — frequently tested in data-analyst-adjacent interviews.

---

## 13.4 Recursive Query Mistakes

### Pitfall A: Missing Cycle Protection on Graph Data

**Symptom:** A recursive CTE traversing a routes/social-graph table either hangs indefinitely or errors out with a recursion-depth-limit exceeded message.

**Root Cause:** No explicit path-tracking cycle detection (Module 4, Section 4.7) on data that isn't guaranteed to be a strict, cycle-free hierarchy.

**Fix:** Add explicit visited-path tracking and exclude already-visited nodes in the recursive member.

### Pitfall B: Using UNION Instead of UNION ALL, Unnecessarily

**Symptom:** A recursive CTE on a large hierarchy is unexpectedly slow.

**Root Cause:** `UNION` forces a full duplicate-elimination pass at **every single iteration** (Module 4, Section 4.12) — often unnecessary overhead if genuine row-level duplicates were never actually possible given the data/cycle-protection already in place.

**Fix:** Use `UNION ALL` unless true deduplication is specifically required.

### Pitfall C: Imprecise Anchor Member

**Symptom:** A recursive CTE meant to start from a single root (e.g., the CEO) produces far more rows than expected, at every level.

**Root Cause:** The anchor member's `WHERE` clause isn't precise enough (e.g., `WHERE manager_id IS NULL OR manager_id = 0` accidentally matches multiple unintended "root" rows due to a data quality issue).

**Fix:** Verify the anchor member returns exactly the intended starting row(s) by running it standalone first, before adding the recursive member.

---

## 13.5 Window Function Misuse

### Pitfall A: LAST_VALUE() Without an Explicit Frame

**Symptom:** `LAST_VALUE()` returns the *current row's own value* instead of the true last value in the partition.

**Root Cause:** The default frame with `ORDER BY` is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (Module 5, Section 5.7), not the full partition.

**Fix:** Explicitly extend the frame: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

### Pitfall B: Using ROW_NUMBER() When DENSE_RANK() Was Needed (or Vice Versa)

**Symptom:** A "2nd highest salary" query returns a value that doesn't match business expectations when there's a tie at the top.

**Root Cause:** `ROW_NUMBER()` assigns strictly sequential numbers even to tied rows, so "2nd" might just be an arbitrary one of several tied top earners, not the "next distinct value" (Module 5, Section 5.4.4).

**Fix:** Use `DENSE_RANK()` when the business meaning is "Nth distinct value"; use `ROW_NUMBER()` only when strictly "the Nth row regardless of ties" is genuinely intended (e.g., pagination).

### Pitfall C: Window Function in WHERE

**Symptom:** A query using `ROW_NUMBER() OVER (...)` directly in `WHERE` fails with a syntax/semantic error.

**Root Cause:** Window functions are logically evaluated **after** `WHERE` (Module 1 §1.11, Module 5 §5.11) — they cannot be referenced there.

**Fix:** Wrap the windowed query in a subquery/CTE and filter in the outer query.

---

## 13.6 Cartesian Explosions

### Pitfall A: Forgotten Join Condition

**Symptom:** A query returns a wildly larger number of rows than expected — potentially millions instead of a few thousand.

**Root Cause:** A comma-join (`FROM A, B`) with a missing or incorrect `WHERE` join condition, or an explicit `CROSS JOIN` used unintentionally (Module 1, Section 1.6.4) — every row of A is paired with every row of B.
```sql
-- ⚠️ Cartesian explosion: 10,000 employees × 500 departments = 5,000,000 rows
SELECT * FROM Employees, Departments WHERE Employees.name = 'Rohan';
-- The WHERE clause filters Employees, but does NOTHING to relate Employees to Departments!
```

**Fix:** Always pair every table in `FROM` with an explicit `JOIN ... ON` condition relating it to the others; treat an unintended `CROSS JOIN` (or comma-join with a missing relational condition) as a critical bug, not a minor inefficiency.

### Pitfall B: Fan-Out Compounding Across Multiple Joins

**Symptom:** A query joining 3+ "many" tables together (e.g., Orders → OrderItems → Shipments) produces an explosively large, duplicated result.

**Root Cause:** Each successive one-to-many join **multiplies** the row count of the previous join's result (a "double fan-out") — 1 order with 5 items and 3 shipment updates could produce 15 rows for that single order.

**Fix:** Pre-aggregate each "many" side independently (in separate CTEs) before joining them together, rather than joining all the raw many-tables together directly.

---

## 13.7 Correlated Query Inefficiencies

### Pitfall A: Correlated Scalar Subquery in SELECT on a Large Table

**Symptom:** A report showing "order count per customer" as a `SELECT`-clause subquery becomes unacceptably slow as the customer table grows.

**Root Cause:** The subquery is conceptually re-executed once per outer row (Module 3, Section 3.2) — O(n × m) in the worst case if the optimizer can't rewrite it efficiently.
```sql
-- Potentially slow at scale:
SELECT c.name,
       (SELECT COUNT(*) FROM Orders o WHERE o.customer_id = c.customer_id) AS order_count
FROM Customers c;
```

**Fix:** Rewrite as a `LEFT JOIN` + `GROUP BY` (Module 3, Section 3.8), which is generally far more optimizer-friendly.
```sql
SELECT c.name, COUNT(o.order_id) AS order_count
FROM Customers c
LEFT JOIN Orders o ON o.customer_id = c.customer_id
GROUP BY c.name;
```

### Pitfall B: Assuming a Correlated Subquery Is Always Slow (Premature "Optimization")

**Symptom:** An engineer manually rewrites every correlated subquery into a join "for performance," even in cases where it made the query less readable and the optimizer was already handling the original efficiently.

**Root Cause:** Modern optimizers (PostgreSQL, MySQL 8+) frequently rewrite correlated subqueries into semi-joins automatically (Module 3, Section 3.2) — the "slow" mental model is a **worst-case theoretical default**, not a guarantee.

**Fix:** Always verify actual behavior with `EXPLAIN`/`EXPLAIN ANALYZE` (Module 11, Section 11.8) before rewriting purely on the assumption that correlated = slow; prioritize readability when performance is genuinely equivalent.

---

## 13.8 Master Debugging Checklist

When a query returns unexpected results, work through this checklist in order:

```
1. Row count too HIGH?
   → Check for: missing join condition (Cartesian explosion, 13.6),
     join fan-out (13.1, 13.2), UNION vs UNION ALL confusion (13.4/Module 7)

2. Row count too LOW / rows missing?
   → Check for: INNER JOIN where LEFT JOIN was needed (13.2A),
     WHERE filtering a LEFT JOIN's right-side column (13.2B),
     NOT IN with a NULL-containing subquery (Module 1 §1.8.6)

3. Aggregate values look WRONG (too high/low)?
   → Check for: fan-out before aggregation (13.1, 13.6B),
     COUNT(*) vs COUNT(column) confusion (13.3B),
     average-of-averages fallacy (13.3C),
     WHERE vs HAVING misplacement (13.3A)

4. Window function output looks WRONG?
   → Check for: missing explicit frame clause (13.5A),
     ROW_NUMBER vs DENSE_RANK confusion (13.5B)

5. Recursive query hangs or errors?
   → Check for: missing cycle protection (13.4A),
     imprecise anchor member (13.4C)

6. Query is CORRECT but SLOW?
   → Check EXPLAIN/EXPLAIN ANALYZE (Module 11) for:
     sequential scans on large filtered tables,
     correlated subquery re-execution (13.7A),
     unnecessary UNION/DISTINCT deduplication passes (13.1B, Module 7 §7.13)
```

---

## 13.9 Interview Questions & MCQs

### Applied ("Spot the Bug" — Very High-Frequency Interview Format)
1. A `LEFT JOIN` query with a filter on the right table's column in `WHERE` is silently behaving like an `INNER JOIN`. Explain why and fix it.
2. A report shows inflated revenue because `Orders` was joined to `OrderItems` before summing `Orders.total`. Explain and fix.
3. A recursive CTE on a flight-routes graph never terminates. Explain and fix.
4. A "2nd highest salary" query using `ROW_NUMBER()` gives a different answer than expected when there's a tie for 1st. Explain and fix.
5. A `COUNT(*)` used to answer "how many customers have a phone number" gives a suspiciously high number. Explain and fix.
6. A query returns 5 million rows instead of an expected few thousand after a colleague "simplified" a multi-table `JOIN` into a comma-separated `FROM` list. Explain and fix.
7. `LAST_VALUE(salary) OVER (PARTITION BY dept ORDER BY salary)` returns each row's own salary instead of the department's max. Explain and fix.

### MCQs

**Q1.** A `LEFT JOIN` behaving like an `INNER JOIN` is most commonly caused by:
 a) A missing index  b) Filtering the right table's column in `WHERE` instead of `ON`  c) Using `UNION` instead of `UNION ALL`  d) A missing `GROUP BY`

**Q2.** The "average of averages" fallacy occurs when:
 a) Using `AVG()` at all  b) Averaging per-group averages when group sizes differ, instead of averaging the raw data directly  c) Using `SUM()` instead of `AVG()`  d) Forgetting `GROUP BY`

**Q3.** A Cartesian explosion is most directly caused by:
 a) Too many indexes  b) A join with no relating condition between the tables  c) Using `DISTINCT`  d) A missing `ORDER BY`

**Q4.** The safest first fix for a slow correlated scalar subquery in `SELECT` is to:
 a) Add `DISTINCT`  b) Rewrite as a `LEFT JOIN` + `GROUP BY` and verify with `EXPLAIN`  c) Switch to `NATURAL JOIN`  d) Add `LIMIT 1`

**Q5.** `COUNT(*)` differs from `COUNT(column)` because:
 a) They are identical in all cases  b) `COUNT(column)` excludes NULL values in that column; `COUNT(*)` counts all rows  c) `COUNT(*)` is always faster  d) `COUNT(column)` requires an index

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) Filtering the right table's column in `WHERE` instead of `ON`**
2. **b) Averaging per-group averages when group sizes differ, instead of averaging the raw data directly**
3. **b) A join with no relating condition between the tables**
4. **b) Rewrite as a `LEFT JOIN` + `GROUP BY` and verify with `EXPLAIN`**
5. **b) `COUNT(column)` excludes NULL values in that column; `COUNT(*)` counts all rows**

</details>

---

## 13.10 Cheat Sheet — Module 13

```
┌───────────────────────────────────────────────────────────────────┐
│                  COMMON SQL PITFALLS — QUICK REFERENCE                 │
├───────────────────────────┬───────────────────────────────────────┤
│ Duplicates from a join       │ Check join grain / fan-out; use EXISTS  │
│                                │ instead of JOIN if only checking match │
│ LEFT JOIN acting like INNER     │ Move right-table filter from WHERE      │
│                                   │ into ON                                │
│ NATURAL JOIN surprises              │ Never use it — explicit JOIN...ON only│
│ WHERE vs HAVING confusion              │ WHERE=row filter pre-group;           │
│                                          │ HAVING=group filter post-aggregate    │
│ COUNT(*) vs COUNT(col)                    │ COUNT(col) skips NULLs                │
│ Average of averages fallacy                 │ Average raw data directly, not        │
│                                               │ per-group averages (unless equal-sized)│
│ Recursive CTE hangs                            │ Add explicit cycle/path protection     │
│ LAST_VALUE() wrong result                        │ Add explicit full-partition frame      │
│ Cartesian explosion                                │ Every FROM table needs a JOIN...ON     │
│ Slow correlated subquery in SELECT                   │ Rewrite as LEFT JOIN + GROUP BY;      │
│                                                         │ verify with EXPLAIN first             │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 13.11 Exercises

### Debugging (This Entire Module Is Debugging-Focused — All Exercises Are "Spot and Fix")
1. Fix: a `LEFT JOIN Orders` query that silently drops customers with zero orders due to a `WHERE o.status = 'COMPLETED'` clause.
2. Fix: a revenue report inflated due to joining `Orders` to `OrderItems` before summing.
3. Fix: a recursive CTE over a social-graph "friends" table that never terminates.
4. Fix: a "top 3 per department" query using `ROW_NUMBER()` that returns fewer than 3 rows for some departments unexpectedly (hint: consider ties and the underlying data).
5. Fix: a `COUNT(*)` used to answer "how many orders have a discount code applied," which should have used `COUNT(discount_code)`.
6. Fix: a query joining 4 tables via comma-separated `FROM` that returns far more rows than expected.
7. Fix: a `LAST_VALUE()` window function returning the wrong value due to a missing frame clause.
8. Fix: a `NOT IN` anti-join query returning zero rows unexpectedly, given a subquery that can return NULLs.
9. Fix: an "average order value across all regions" computed as `AVG(region_avg)`, giving a mathematically incorrect result given unequal region sizes.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

This module is a **standalone reference catalog** — return to it whenever debugging unexpected SQL results. The seven pitfall categories (duplicate rows, incorrect joins, wrong aggregations, recursive query mistakes, window function misuse, Cartesian explosions, correlated query inefficiencies) cover the overwhelming majority of real-world "why is my query wrong/slow" incidents, and **Section 13.8's Master Debugging Checklist** is designed to be used directly, symptom-first, the next time something looks off.

---

## Cross-References for Later Parts / Modules
- **Industry Case Studies applying these debugging skills to realistic multi-table schemas** → Module 14 (next)
- **Interview Mastery — timed practice applying this exact debugging discipline** → Module 15
- **Full query optimizer behavior for diagnosing performance-category pitfalls** → Part 8

---

> ✅ **Module 13 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 14"** to proceed to **Module 14: Industry Case Studies** — advanced SQL problems across Banking, HR Analytics, Sales Analytics, Inventory, Healthcare, Social Media, Recommendation Systems, and Finance, each with multiple solution approaches.
