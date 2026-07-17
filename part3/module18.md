# Part 3 — The Complete SQL Handbook
## Module 18: SQL Interview Questions

> **Goal of this module:** A dedicated, curated question bank spanning beginner to advanced levels — conceptual questions, query-writing challenges, output-prediction puzzles, and scenario-based system-design-adjacent questions — drawing on everything covered in Modules 1–17. Each question includes a full, reasoned answer, not just the final result.

---

### 📍 Where This Module Fits

```
Module 17: What goes wrong  →  Module 18: Interview-format practice (You are here)  →  Module 19: Industry Case Studies
```

---

## 18.1 Beginner-Level Conceptual Questions

**Q1. What is the difference between SQL and MySQL?**
> SQL is a declarative query *language* for interacting with relational databases; MySQL is a specific *database management system product* that implements SQL (with its own dialect and extensions). (Module 1, Section 1.6)

**Q2. What are the five categories of SQL commands?**
> DDL (structure: `CREATE`, `ALTER`, `DROP`), DML (data: `INSERT`, `UPDATE`, `DELETE`), DQL (`SELECT`), DCL (`GRANT`, `REVOKE`), TCL (`COMMIT`, `ROLLBACK`). (Module 1, Section 1.8)

**Q3. What's the difference between `DELETE`, `TRUNCATE`, and `DROP`?**
> `DELETE` (DML) removes specific rows, is slower, fires triggers, is filterable, and is rollback-able. `TRUNCATE` (DDL) removes all rows quickly, generally skips triggers, isn't filterable, and isn't rollback-able in MySQL. `DROP` (DDL) removes the entire table structure permanently. (Modules 3–4, 7)

**Q4. What is a primary key, and how does it differ from a unique key?**
> A primary key uniquely identifies each row, is implicitly `NOT NULL`, and only one is allowed per table. A `UNIQUE` constraint also enforces uniqueness but allows `NULL` values and multiple `UNIQUE` constraints per table. (Module 6, Sections 6.2, 6.5)

**Q5. What's the difference between `WHERE` and `HAVING`?**
> `WHERE` filters individual rows *before* grouping/aggregation; `HAVING` filters groups *after* aggregation. `WHERE` cannot reference aggregate functions; `HAVING` can. (Module 11, Section 11.5; Module 15)

**Q6. What is a foreign key?**
> A constraint ensuring a column's value matches an existing value in another table's referenced column, enforcing referential integrity between tables. (Module 6, Section 6.3)

**Q7. What does `NULL` represent in SQL?**
> "Unknown" or "not applicable" — it is not zero, not an empty string, and not `FALSE`. SQL uses three-valued logic (`TRUE`/`FALSE`/`UNKNOWN`) specifically to handle `NULL` comparisons. (Module 9, Section 9.9)

**Q8. What is a JOIN, and name the main types.**
> A JOIN combines rows from two or more tables based on a related column. Main types: `INNER`, `LEFT`, `RIGHT`, `FULL OUTER`, `CROSS`, `SELF`. (Module 12)

---

## 18.2 Intermediate-Level Conceptual Questions

**Q9. Explain `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT column)`.**
> `COUNT(*)` counts every row regardless of `NULL`s. `COUNT(column)` counts only non-`NULL` values in that column. `COUNT(DISTINCT column)` counts distinct non-`NULL` values. (Module 11, Section 11.2)

**Q10. Why does `NOT IN` sometimes return zero rows unexpectedly?**
> If the list/subquery passed to `NOT IN` contains a `NULL`, the entire `NOT IN` expression evaluates to `UNKNOWN` for every row (since it expands to a chain of `AND`ed `<>` comparisons, one of which compares against `NULL`), causing `WHERE` to exclude all rows. The fix is to use `NOT EXISTS` instead. (Module 9, Section 9.5)

**Q11. What is the difference between a clustered and non-clustered index conceptually (as it relates to MySQL's InnoDB vs PostgreSQL)?**
> InnoDB (MySQL) physically stores table rows in primary key order (a clustered index) — primary key choice directly affects physical storage layout. PostgreSQL uses heap storage, where row physical order is independent of primary key value. This makes primary key design (e.g., avoiding random UUIDs) more performance-consequential in MySQL. (Module 6, Section 6.2; Module 5, Section 5.7)

**Q12. What's the difference between a view and a materialized view?**
> A regular view stores only the query definition and is re-computed live on every access (no meaningful performance overhead beyond the underlying query, due to query rewriting). A materialized view physically stores the computed result set, offering fast reads at the cost of data staleness until explicitly refreshed. MySQL has no native materialized view support. (Module 14, Sections 14.2, 14.5)

**Q13. Explain `UPSERT` and why it's safer than "check-then-insert" application logic.**
> `UPSERT` (`INSERT ... ON CONFLICT` in PostgreSQL, `ON DUPLICATE KEY UPDATE` in MySQL) atomically inserts or updates a row based on a unique constraint conflict, in a single statement. Hand-rolled "check if exists, then insert or update" logic is vulnerable to race conditions under concurrent execution, since two simultaneous checks could both see "doesn't exist" before either has written. (Module 7, Section 7.5)

**Q14. What is a correlated subquery?**
> A subquery that references a column from its enclosing outer query, conceptually re-evaluated per outer row (though the optimizer may rewrite it into a more efficient join/semi-join internally). (Module 13, Section 13.5)

**Q15. Why should you avoid `SELECT *` in production application code?**
> It's fragile to future schema changes (new columns silently included), wastes I/O/network bandwidth, and defeats the optimizer's ability to use index-only scans. (Module 8, Section 8.2)

---

## 18.3 Advanced-Level Conceptual Questions (Senior SDE-2/SDE-3)

**Q16. Explain SQL's logical execution order, and derive why `GROUP BY` requires non-aggregated `SELECT` columns to also appear in `GROUP BY`.**
> Logical order: `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`. Since `GROUP BY` executes before `SELECT`, by the time `SELECT` runs, the engine is only working with *groups* of rows, not individual rows — so any column not in `GROUP BY` and not wrapped in an aggregate has no single, well-defined value to return per group, and the engine refuses to guess. (Module 15)

**Q17. Why might a careless choice of `ON DELETE CASCADE` cause a production incident?**
> `CASCADE` chains can propagate deletions across multiple related tables from a single `DELETE` statement — if the "blast radius" isn't fully understood (e.g., deleting a customer cascades through orders, order_items, invoices, etc.), a single deletion can silently destroy far more data than intended. The choice between `CASCADE`, `RESTRICT`, `SET NULL`, and `SET DEFAULT` should be driven by genuine relationship semantics, not habit. (Module 6, Section 6.4)

**Q18. Why does deep `OFFSET`-based pagination degrade in performance, and what's the standard fix?**
> `OFFSET n` requires the engine to generate and discard the first `n` rows before returning the next batch — it cannot "jump directly" to a position. This makes later pages progressively slower on large tables. The standard fix is **keyset (cursor-based) pagination**, filtering with `WHERE id > last_seen_id` instead of using `OFFSET`. (Module 8, Section 8.7)

**Q19. Explain `TIMESTAMP` vs `TIMESTAMPTZ` in PostgreSQL, and why using naive `TIMESTAMP` for event data is risky in multi-timezone systems.**
> `TIMESTAMPTZ` normalizes incoming values to UTC internally and converts to the session's timezone on display, preserving the absolute instant regardless of input timezone. Naive `TIMESTAMP` stores exactly what's given with no conversion, which can cause inconsistent, confusing behavior (events appearing "in the future," inconsistent day-boundary calculations) in systems spanning multiple timezones. (Module 5, Section 5.5)

**Q20. Why is `EXISTS` generally preferred over `IN`/`NOT IN` with a subquery?**
> `EXISTS` evaluates to a clean `TRUE`/`FALSE` regardless of `NULL`s in the related data, making it immune to the `NOT IN` + `NULL` trap. It also often allows the optimizer to short-circuit (stop at the first match) via a semi-join strategy. (Module 9, Section 9.7; Module 13, Section 13.8)

**Q21. Why does MySQL's InnoDB clustered index design make random UUID primary keys more costly than in PostgreSQL?**
> Because InnoDB physically stores rows in primary key order, non-sequential UUID inserts cause page splits and fragmentation throughout the *entire row storage*, not just a separate index structure — a more severe effect than in PostgreSQL's heap-organized tables, where only the (separate) index structure is affected. (Module 5, Section 5.7; Module 6, Section 6.2)

**Q22. Design a schema decision: `ENUM` vs. a lookup table for a `status` column that may need new values added over time. Which would you choose, and why?**
> A lookup table (a separate table listing valid statuses, referenced via foreign key) is generally more flexible — adding a new valid value is a simple `INSERT`, whereas adding a value to an `ENUM` requires a schema migration (`ALTER TYPE ... ADD VALUE` in PostgreSQL, or a full column redefinition in MySQL), and removing/reordering `ENUM` values is even more disruptive. For statuses expected to evolve, many engineers prefer the lookup-table approach precisely for this flexibility. (Module 5, Section 5.8)

---

## 18.4 Output Prediction Questions

**Q23.** What does the following return?
```sql
SELECT 7 / 2;
```
> `3` — integer division truncates when both operands are integers. (Module 9, Section 9.1)

**Q24.** Given `employees` has 5 rows, 2 with `department = NULL`, what does each return?
```sql
SELECT COUNT(*) FROM employees;           -- ?
SELECT COUNT(department) FROM employees;  -- ?
```
> `COUNT(*)` → `5`. `COUNT(department)` → `3` (excludes the 2 `NULL`s). (Module 11, Section 11.2)

**Q25.** What does this return?
```sql
SELECT NULL = NULL;
```
> `UNKNOWN` (or `NULL` displayed, depending on context) — not `TRUE`. Two unknowns aren't provably equal. (Module 9, Section 9.9)

**Q26.** Given `employees` has zero rows matching a filter, what does each return?
```sql
SELECT COUNT(*) FROM employees WHERE department = 'Ghost';   -- ?
SELECT SUM(salary) FROM employees WHERE department = 'Ghost'; -- ?
```
> `COUNT(*)` → `0`. `SUM(salary)` → `NULL` (not `0`). (Module 11, Section 11.2)

**Q27.** In MySQL's default configuration, what does this return?
```sql
SELECT 'Hello' || 'World';
```
> A boolean-like `0` (logical OR of two non-empty strings, both treated as truthy in a numeric-ish evaluation, typically yielding `1` in MySQL's default non-`PIPES_AS_CONCAT` mode) — **not** `'HelloWorld'`. Use `CONCAT()` instead. (Module 10, Section 10.3)

**Q28.** What does this return, given `department_id` values `(1, 2, NULL)` exist in the subquery result?
```sql
SELECT * FROM employees WHERE department_id NOT IN (SELECT department_id FROM some_table);
```
> Zero rows, always — regardless of what `department_id` actually is for any row. (Module 9, Section 9.5)

---

## 18.5 Query-Writing Challenges

**Q29.** Write a query to find the second-highest salary in the `employees` table (without using `LIMIT`/`OFFSET`).
```sql
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```
*(A classic, extremely common interview question — testing subquery fluency, Module 13.)*

**Q30.** Write a query to find departments with no employees.
```sql
SELECT d.name FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.department_id = d.id);
```
*(The anti-join pattern, Module 12, Section 12.3 / Module 13, Section 13.8.)*

**Q31.** Write a query to find duplicate email addresses in a `users` table.
```sql
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```
*(A classic `GROUP BY`/`HAVING` question, Module 11.)*

**Q32.** Write a query showing each employee alongside their manager's name (self-referencing `manager_id`).
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```
*(Self-join, Module 12, Section 12.7.)*

**Q33.** Write a query to safely compute average order value, avoiding a division-by-zero error if `total_orders` is 0.
```sql
SELECT total_revenue / NULLIF(total_orders, 0) AS avg_order_value FROM sales_summary;
```
*(Module 10, Section 10.7.)*

**Q34.** Write a query using conditional aggregation to show, per department, counts of active and inactive employees in a single row per department.
```sql
SELECT
    department,
    COUNT(CASE WHEN status = 'active' THEN 1 END) AS active_count,
    COUNT(CASE WHEN status = 'inactive' THEN 1 END) AS inactive_count
FROM employees
GROUP BY department;
```
*(Module 10, Section 10.7 combined with Module 11.)*

**Q35.** Write an UPSERT that increments a product's stock count, inserting a new row if the product doesn't already exist.
```sql
-- PostgreSQL
INSERT INTO inventory (product_id, stock)
VALUES (101, 10)
ON CONFLICT (product_id) DO UPDATE SET stock = inventory.stock + EXCLUDED.stock;
```
*(Module 7, Section 7.5.)*

---

## 18.6 Scenario-Based Questions

**Q36.** *"Our production report shows plausible but incorrect total revenue after we added a `product_tags` join. What's your debugging approach?"*
> This strongly suggests a **fan-out/Cartesian-product-style duplication** from a one-to-many join (e.g., each product joined against multiple tags multiplies the row count) inflating the `SUM`. The fix is typically to aggregate tags separately (or use `DISTINCT`/pre-aggregate in a subquery/CTE) before joining, or to restructure the query so the revenue aggregation happens independently of the tag join's row multiplication. (Module 17, Sections 17.4–17.5)

**Q37.** *"You need to design a globally-distributed system where multiple regional services generate order IDs without a central coordinator. Auto-increment or UUID?"*
> UUIDs (ideally UUIDv7 or similar time-ordered variants) allow independent, coordinator-free ID generation across regions, at the cost of some index locality (especially significant in MySQL/InnoDB's clustered index). Auto-increment requires a single authoritative counter, which conflicts with the distributed, coordinator-free requirement. (Module 5, Section 5.7)

**Q38.** *"A junior engineer wants to add a `NOT NULL` column with a default to a 500-million-row production table during peak hours. What do you advise?"*
> Flag the real risk of table locking/rewriting on some engine versions/configurations (Module 4, Section 4.4) — recommend testing on a realistic data-volume staging copy first, scheduling the change during low-traffic hours, and considering an online-schema-change tool (e.g., `gh-ost` for MySQL, `pg_repack`-adjacent techniques for PostgreSQL) if the change is confirmed to be blocking.

**Q39.** *"Design the constraints for an `orders` table where you need to retain historical orders even after a customer is deleted, but still prevent orders from referencing nonexistent customers at insert time."*
> Use a `FOREIGN KEY` with `ON DELETE SET NULL` (or a soft-delete pattern on `customers` instead of a hard delete) — this preserves the order record while nulling the now-invalid customer reference, rather than `CASCADE` (which would destroy the order) or `RESTRICT` (which would block legitimate customer deletion). (Module 6, Section 6.4)

**Q40.** *"How would you build an infinite-scroll feed that stays fast even after a user scrolls through thousands of posts?"*
> Use **keyset (cursor-based) pagination** (`WHERE id/created_at < last_seen_value ORDER BY ... LIMIT n`) instead of `OFFSET`-based pagination, which degrades linearly with scroll depth. (Module 8, Section 8.7)

---

## 18.7 Advantages of Interview-Format Practice

| Benefit | Explanation |
|---|---|
| Simulates real interview pressure | Practicing recall and articulation, not just recognition |
| Reveals genuine gaps | Struggling to answer reveals which modules need review |
| Builds fluent, confident explanation | Interviews reward clear articulation of *why*, not just *what* |

## 18.8 Interview Importance

This entire module *is* interview practice — but worth noting the **meta-pattern**: at every level, interviewers are testing whether you understand **why** something is true (three-valued logic, execution order, clustered vs. heap storage), not just whether you can recite syntax. The strongest answers throughout this question bank consistently trace back to a small set of foundational principles from Modules 1, 6, 9, and 15.

---

## 18.9 Chapter Summary

This module deliberately contains no new SQL concepts — it's a curated rehearsal of everything from Modules 1–17, organized by interview format (conceptual, output-prediction, query-writing, scenario-based) rather than by topic, mirroring how real interviews actually mix and interleave these categories.

---

## 18.10 Exercises

### 🟢 Beginner
1. Without looking back, answer Q1–Q8 from memory, then check your answers against Section 18.1.
2. Write out the query for Q29 (second-highest salary) from scratch, then verify it against Section 18.5.

### 🟡 Intermediate
3. Answer Q9–Q15 from memory, and for each, name which earlier module fully covers it.
4. Attempt Q23–Q28 (output prediction) without running any code — then verify by actually running them against a practice database.

### 🔴 Advanced
5. Answer Q16–Q22 in full, spoken-interview style (record yourself or explain aloud to someone else) — aim for clear, structured 60-90 second answers.
6. Attempt Q36–Q40 (scenario-based) as if in a live interview, explaining your reasoning process, not just your final answer.

### 🏗️ Mini Project
7. Write five *new* interview questions of your own — one per category (conceptual, output-prediction, query-writing, scenario-based, and a "what's wrong with this query" debugging question) — based on a topic from this handbook you find personally trickiest, complete with full answers.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [DataLemur — SQL Interview Questions](https://datalemur.com/)
- [StrataScratch](https://www.stratascratch.com/)

---

## ✅ Module 18 Complete

You've now rehearsed the full range of SQL interview formats — conceptual, output-prediction, query-writing, and scenario-based — all directly traceable back to the foundational principles built across this entire handbook.

**Next up: Module 19 — Industry Case Studies**, where you'll implement complete SQL solutions for realistic Banking, E-commerce, Student Management, Hospital, Inventory, and Food Delivery systems.

---

*Reply "continue" or "next module" when you're ready for Module 19.*
