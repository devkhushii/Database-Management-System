# Part 3 — The Complete SQL Handbook
## Module 16: SQL Best Practices

> **Goal of this module:** Consolidate the scattered best-practice threads from Modules 1–15 into one deliberate, comprehensive discipline — naming, formatting, readability, `NULL`-handling, and defensive query-writing habits that distinguish professional, production-grade SQL from merely "working" SQL.

---

### 📍 Where This Module Fits

```
Module 15: How queries execute  →  Module 16: How to write them WELL (You are here)  →  Module 17: Common Mistakes
```

---

## 16.1 Why a Dedicated Best-Practices Module?

Every module so far has included scattered "best practice" call-outs — this module exists to **consolidate them into one coherent discipline**, framed around a single guiding principle: **SQL is read far more often than it's written.** A query you write once will be read, debugged, modified, and reasoned about by other engineers (and your future self) dozens of times over its lifetime — optimizing for that reality is what separates professional SQL from merely functional SQL.

---

## 16.2 Naming Conventions

| Object | Convention | Example |
|---|---|---|
| Tables | Plural, `snake_case`, nouns | `employees`, `order_items` |
| Columns | Singular, `snake_case`, descriptive | `first_name`, `created_at` |
| Primary keys | Consistently named (`id` is the most common convention) | `id` |
| Foreign keys | `<singular_referenced_table>_id` | `department_id`, `customer_id` |
| Booleans | Prefixed with `is_`/`has_`/`can_` | `is_active`, `has_paid` |
| Junction/association tables | Combine both related table names | `student_courses`, `order_items` |

> ⚠️ **Common mistake:** Using **reserved SQL keywords** as identifiers (`order`, `user`, `group`, `select` are all real, commonly-hit collisions) — this forces every query referencing them to use engine-specific quoting (`"order"` in PostgreSQL, `` `order` `` in MySQL) forever, a genuinely persistent annoyance. **Best practice: avoid naming tables/columns after SQL reserved words entirely** — prefer `orders` (plural, already sidesteps the singular `order` keyword collision) or `user_accounts` instead of the bare `user`.

> 💡 **Consistency matters more than any specific convention chosen:** Whether your team prefers `snake_case` or `camelCase`, singular or plural table names, the single most important rule is **picking one convention and applying it consistently across the entire schema** — a schema that mixes `Employee` and `department_list` and `orderItems` inconsistently is measurably harder to work with than one that's uniformly, if imperfectly, consistent.

---

## 16.3 Formatting and Readability

```sql
-- Poor formatting — hard to scan, hard to debug
SELECT e.name,d.name,e.salary FROM employees e JOIN departments d ON e.department_id=d.id WHERE e.status='active' AND e.salary>50000 ORDER BY e.salary DESC;

-- Well-formatted — one clause per line, consistent indentation, clear keyword casing
SELECT
    e.name,
    d.name AS department_name,
    e.salary
FROM employees e
JOIN departments d
    ON e.department_id = d.id
WHERE e.status = 'active'
    AND e.salary > 50000
ORDER BY e.salary DESC;
```

### Formatting Conventions Worth Standardizing On

- **One major clause per line** (`SELECT`, `FROM`, `WHERE`, `GROUP BY`, etc.) — makes the query's overall structure scannable at a glance, directly mirroring **Module 15's logical execution order**.
- **Consistent keyword casing** — most teams standardize on either `UPPERCASE` keywords (`SELECT`, `WHERE`) with `lowercase` identifiers, or fully lowercase everything; either is fine, but pick one and apply it uniformly.
- **Indent conditions under multi-line `AND`/`OR` chains** for readability, directly reinforcing **Module 9, Section 9.3**'s parenthesization discipline.
- **Alias tables consistently and meaningfully** (`e` for `employees`, `d` for `departments`) rather than arbitrary single letters that don't hint at their source.

> 💡 **Industry usage note:** Many real engineering teams enforce SQL formatting automatically via linters/formatters (e.g., `sqlfluff`, `pgFormatter`) integrated into CI pipelines — exactly analogous to how `prettier`/`black` enforce consistent formatting in application code. Consistent SQL formatting is genuinely treated as a professional standard, not merely a stylistic preference, at serious engineering organizations.

---

## 16.4 NULL Handling Discipline

This consolidates **Module 9's** three-valued logic and **Module 10's** `NULL`-propagation lessons into concrete, actionable habits:

| Discipline | Reference |
|---|---|
| Always use `IS NULL`/`IS NOT NULL`, never `= NULL` | Module 9, Section 9.9 |
| Never use `NOT IN` with a subquery/list that might contain `NULL` — prefer `NOT EXISTS` | Module 9, Section 9.5/9.7 |
| Remember `SUM`/`AVG`/`MIN`/`MAX` return `NULL` (not `0`) over zero matching rows — guard with `COALESCE` where a numeric default is needed | Module 11, Section 11.2 |
| Remember most functions propagate `NULL` — a single `NULL` argument silently `NULL`-ifies an entire expression | Module 10, Section 10.3 |
| Declare `NOT NULL` wherever "missing" genuinely shouldn't be a valid state | Module 6, Section 6.7 |

> 💡 **A genuinely important, unifying mental habit:** Before writing any expression or condition involving a column that's nullable, explicitly ask: **"what should happen here if this value is `NULL`?"** — and write the query to handle that case deliberately (via `COALESCE`, `IS NULL`, `IS NOT DISTINCT FROM`, etc.), rather than discovering the answer accidentally in production. This single habit prevents the majority of `NULL`-related bugs covered throughout this handbook.

---

## 16.5 Query-Writing Discipline

### Always Specify Explicit Column Lists

```sql
-- Fragile
INSERT INTO employees VALUES (DEFAULT, 'Ananya', 'Engineering', 95000);
SELECT * FROM employees;

-- Robust
INSERT INTO employees (name, department, salary) VALUES ('Ananya', 'Engineering', 95000);
SELECT name, department, salary FROM employees;
```

Directly reinforces **Module 7, Section 7.2** (`INSERT` column lists) and **Module 8, Section 8.2** (`SELECT *` avoidance) — both stem from the same underlying principle: **explicit is more resilient than implicit**, especially as schemas evolve over time via `ALTER TABLE` (Module 4).

### Verify Before You Modify

```sql
-- Always run this first:
SELECT * FROM employees WHERE department = 'Sales' AND status = 'terminated';
-- THEN, once confirmed, run:
DELETE FROM employees WHERE department = 'Sales' AND status = 'terminated';
```

Directly reinforces **Module 7, Sections 7.7–7.8** — the single most damage-preventing habit in all of DML.

### Prefer Set-Based Thinking Over Row-by-Row Thinking

SQL is fundamentally **declarative and set-based** (**Module 1, Section 1.1**) — a genuinely important best practice is training yourself to think in terms of "what set of rows satisfies this condition" rather than mentally simulating a procedural loop. This mindset shift directly leads to using `JOIN`s, `CASE`-based conditional aggregation (Module 10–11), and window functions (Part 4) instead of writing application-side loops that issue one query per row — a pattern that is almost always both slower and less idiomatic.

### Break Down Complexity with CTEs Over Deep Nesting

As introduced in **Module 13, Section 13.6**, prefer naming and sequencing logical steps with `WITH` clauses (fully covered in Part 4) over deeply nested subqueries — readability compounds in value as queries grow in complexity, and a query that takes a reader thirty seconds to parse is a genuinely different (better) artifact than one that takes five minutes.

---

## 16.6 Defensive SQL

**Defensive SQL** means writing queries that **fail safely and predictably** rather than silently producing wrong results or catastrophic damage.

| Defensive Habit | Prevents |
|---|---|
| Always use `WHERE` with `UPDATE`/`DELETE`, verified via `SELECT` first | Accidental full-table modification (Module 7) |
| Use `IF EXISTS`/`IF NOT EXISTS` in DDL scripts meant to be re-runnable | Errors on re-running idempotent migration scripts (Modules 3–4) |
| Explicitly choose `ON DELETE`/`ON UPDATE` referential actions rather than relying on engine defaults | Unintended cascades or unexpected `RESTRICT` failures (Module 6) |
| Guard divisions with `NULLIF` | Runtime division-by-zero errors (Module 10) |
| Prefer `EXISTS`/`NOT EXISTS` over `NOT IN` with subqueries | The silent zero-rows `NULL` trap (Module 9) |
| Wrap multi-statement DML sequences in explicit transactions (Part 4 formalizes this) | Partial, inconsistent updates if a later statement fails |
| Test destructive operations on a copy/staging environment first | Irreversible production data loss (Modules 3–4, 7) |

> 🧠 **Real-life analogy:** Defensive SQL is like a pilot's pre-flight checklist — not because pilots are incompetent, but because **systematically checking for known failure modes before committing to an irreversible action** is simply what professional practice looks like, regardless of individual skill level. The best engineers aren't the ones who never make mistakes — they're the ones who've built habits and systems that catch mistakes before they cause damage.

---

## 16.7 Consolidated Best-Practices Checklist

```
□ Schema design
  □ Consistent naming convention (snake_case, singular/plural rule) applied uniformly
  □ Avoid SQL reserved words as identifiers
  □ Appropriate data types chosen deliberately (Module 5), not by default habit
  □ Constraints (PK, FK, CHECK, NOT NULL) enforced at the database level (Module 6)

□ Query writing
  □ Explicit column lists in SELECT and INSERT (never bare SELECT * or positional INSERT)
  □ WHERE used for row-level filters; HAVING reserved only for aggregate-level filters
  □ Mixed AND/OR conditions always parenthesized
  □ NULL handled deliberately: IS NULL/IS NOT NULL, EXISTS over NOT IN, COALESCE where needed
  □ CTEs preferred over deeply nested subqueries for anything beyond 2-3 levels

□ Safety
  □ UPDATE/DELETE always verified with a matching SELECT first
  □ Destructive DDL (DROP, TRUNCATE) access-controlled and deliberate
  □ ON DELETE/ON UPDATE referential actions chosen explicitly, not left to defaults
  □ Idempotent migration scripts use IF EXISTS/IF NOT EXISTS guards

□ Formatting
  □ Consistent keyword casing and indentation
  □ One clause per line for anything beyond a trivial query
  □ Meaningful table aliases
```

---

## 16.8 Advantages of Disciplined SQL Practice

| Advantage | Explanation |
|---|---|
| Reduced production incidents | Defensive habits (Section 16.6) directly prevent the most common, most damaging real-world SQL mistakes |
| Faster onboarding/collaboration | Consistent naming/formatting lets new team members read and trust unfamiliar queries quickly |
| Easier debugging | Explicit, well-structured queries are far faster to diagnose when something goes wrong |
| Stronger interview performance | Demonstrating these habits organically signals genuine production experience, not just textbook knowledge |

## 16.9 Common Mistakes (Meta-Level, Consolidating the Handbook)

- Treating best practices as optional "nice to haves" rather than genuine risk-reduction discipline — nearly every practice in this module exists because it prevents a real, previously-documented category of production bug covered somewhere in Modules 1–15.
- Inconsistency — applying a best practice sometimes but not others, which often confuses collaborators more than not following the practice at all.
- Optimizing for "getting the query to work once" rather than "will this query remain correct and safe as the schema and data evolve."

## 16.10 Industry Usage

Serious engineering organizations formalize much of this module into **SQL style guides**, **code review checklists**, and **automated linting rules** — the difference between a hobbyist's SQL and a production engineer's SQL is rarely raw syntax knowledge; it's near-universally this exact discipline: naming consistency, defensive habits, and readability investment.

## 16.11 Interview Importance

| Level | Typical Questions |
|---|---|
| Mid-level | "What are some SQL best practices you follow?" |
| Senior SDE-2/SDE-3 | Best practices are rarely asked directly as a standalone question at this level — instead, they're **evaluated implicitly** throughout every other SQL question, in whether your example queries and schema designs naturally demonstrate this discipline without being prompted to. |

---

## 16.12 Chapter Summary

- SQL is read far more than it's written — naming, formatting, and readability investment pay compounding dividends over a query's or schema's lifetime.
- Consistent naming conventions and avoidance of reserved-word identifiers prevent long-term friction.
- `NULL`-handling discipline (Module 9–11's lessons) should become an automatic, deliberate habit, not an afterthought.
- Defensive SQL — verifying before modifying, explicit referential actions, guarded division, idempotent DDL — is what separates production-grade query-writing from merely-functional query-writing.
- This module doesn't introduce new syntax; it **consolidates and operationalizes** the best-practice threads woven throughout Modules 1–15 into one deliberate discipline.

---

## 16.13 Revision Notes / Cheat Sheet

```
Naming:      snake_case, plural tables, singular FK columns (<table>_id), avoid reserved words
Formatting:  one clause per line, consistent keyword casing, meaningful aliases
NULLs:       IS NULL not = NULL; EXISTS not NOT IN; COALESCE for safe defaults
Query habits: explicit columns always; WHERE for rows, HAVING for aggregates; CTEs over deep nesting
Defensive:   SELECT before UPDATE/DELETE; explicit ON DELETE/ON UPDATE; NULLIF for safe division;
             IF EXISTS/IF NOT EXISTS for idempotent DDL
```

---

## 16.14 Multiple Choice Questions (MCQs)

1. Why should you avoid naming a table `order`?
   a) It's invalid syntax b) **It's a SQL reserved word, requiring quoting everywhere it's used** c) Tables can't be singular d) It causes a performance penalty

2. Which is the recommended defensive habit before running `DELETE`/`UPDATE` in production?
   a) Run it directly, it's usually fine b) **Run an equivalent `SELECT` first to verify affected rows** c) Always use `TRUNCATE` instead d) Disable constraints first

3. What's the primary reason to avoid `SELECT *` and unqualified `INSERT` in production code?
   a) They're slower to type b) **They're fragile to future schema changes** c) They're not valid SQL d) They only work with one table

4. Why is `NULLIF(divisor, 0)` considered a best practice?
   a) It improves performance b) **It prevents a division-by-zero runtime error by safely producing NULL instead** c) It's required by the SQL standard d) It only works in PostgreSQL

5. What is the primary reason SQL formatting consistency matters?
   a) Faster query execution b) **Readability and maintainability for anyone who reads the query later, including your future self** c) It's enforced by the database engine d) It reduces storage size

---

## 16.15 Exercises

### 🟢 Beginner
1. Take a poorly-formatted query (write one deliberately dense and hard to read) and reformat it following this module's conventions.
2. List three SQL reserved words you should avoid using as table/column names.
3. Rewrite a `SELECT *` query from an earlier module's exercises with explicit column names.

### 🟡 Intermediate
4. Take a `DELETE` statement from Module 7's exercises and write the "verify first" `SELECT` you'd run before executing it.
5. Design a small schema (3-4 tables) applying every naming convention from Section 16.2, and justify each naming choice in a comment.
6. Identify and fix at least two `NULL`-handling issues in a deliberately flawed query you construct yourself, referencing Section 16.4.

### 🔴 Advanced / Scenario-Based
7. Your team is onboarding a new engineer who inherited a schema with inconsistent naming (`Employee`, `dept_list`, `orderItems`). Write a short, professional explanation of why you'd advocate for standardizing this, and propose a consistent convention.
8. Design a defensive checklist (using Section 16.7 as a template) specifically for a hypothetical "bulk data cleanup" script your team runs quarterly against production.

### 🏗️ Mini Project
9. Take the most complex query you've written across this handbook's exercises so far, and produce a "before and after" — the original, and a refactored version applying every relevant best practice from this module (naming, formatting, `NULL` handling, CTEs if applicable) — with a short write-up of what changed and why.

### 🔗 Practice Links
- [SQLStyle.guide — a widely-referenced SQL style guide](https://www.sqlstyle.guide/)
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [DataLemur](https://datalemur.com/)

---

## ✅ Module 16 Complete

You've now consolidated the scattered best-practice lessons from every prior module into one deliberate, professional discipline — the difference between SQL that merely works and SQL that's genuinely production-ready.

**Next up: Module 17 — Common SQL Mistakes**, a dedicated, exhaustive catalog of NULL mistakes, join mistakes, aggregate mistakes, duplicate-row causes, Cartesian products, incorrect `GROUP BY` usage, wrong filtering, and a preview of SQL injection.

---

*Reply "continue" or "next module" when you're ready for Module 17.*
