# The Complete DBMS Handbook — Part 4

## Relational Algebra, Relational Calculus, Advanced SQL & Query Optimization Fundamentals

> **Prerequisite:** This part assumes you have completed Parts 1–3 (ER Modeling, Basic Relational Model, Basic SQL, Keys & Constraints). If any of those terms are unfamiliar, please review Parts 1–3 first.

> **How this handbook is structured:** Per the plan, each module is delivered one at a time. This document contains **Module 1: Relational Algebra** in full depth. Reply "next" or "continue" and I will generate **Module 2: Relational Calculus**.

---

# 📘 MODULE 1: RELATIONAL ALGEBRA

---

## 1.0 Chapter Roadmap

```
Module 1: Relational Algebra
│
├── 1.1 What is Relational Algebra?
├── 1.2 Why does it exist?
├── 1.3 Mathematical Foundation
├── 1.4 Unary vs Binary Operations
├── 1.5 Unary Operators
│     ├── Selection (σ)
│     └── Projection (π)
├── 1.6 Set-Theoretic Binary Operators
│     ├── Union (∪)
│     ├── Intersection (∩)
│     ├── Difference (−)
│     └── Cartesian Product (×)
├── 1.7 Rename (ρ)
├── 1.8 Join Family
│     ├── Theta Join
│     ├── Equi Join
│     ├── Natural Join
│     ├── Outer Join (Left/Right/Full)
│     ├── Semi Join
│     └── Anti Join
├── 1.9 Division (÷)
├── 1.10 Assignment Operator (←)
├── 1.11 SQL ↔ Relational Algebra Conversion Guide
├── 1.12 PostgreSQL vs MySQL Notes
├── 1.13 Common Mistakes & Edge Cases
├── 1.14 Interview Questions & MCQs
├── 1.15 Cheat Sheet
└── 1.16 Exercises
```

---

## 1.1 What is Relational Algebra?

**Definition:** Relational Algebra (RA) is a **procedural query language**. It is a set of mathematical operators that take one or two relations (tables) as input and produce a **new relation** as output. Every operator's output is again a relation — this property is called **closure**.

> **Closure, in plain English:** If you cut a piece of wood with a saw, you get another piece of wood — not a piece of metal. Similarly, every RA operator takes tables in and gives a table out. This lets you **chain operators together**, feeding the output of one directly into another, exactly like nesting function calls: `f(g(h(x)))`.

Relational Algebra is **not** something you type into PostgreSQL or MySQL directly. Instead, it is the **theoretical language that SQL is built on top of**, and — more importantly for engineers — it is the internal language that the **query optimizer** uses to represent, rewrite, and reason about your SQL query before executing it.

```
   Your SQL Query
        │
        ▼
   Parser converts SQL → Relational Algebra Expression Tree
        │
        ▼
   Optimizer rewrites the RA tree (predicate pushdown, join
   reordering, etc.) into an equivalent but cheaper RA tree
        │
        ▼
   Execution Engine walks the optimized RA tree and
   produces the final result set
```

This is why RA matters even though you'll never "write" it in production: **understanding RA lets you understand what the query optimizer is doing to your SQL, and therefore lets you write SQL that the optimizer can optimize well.**

### Real-life Analogy

Think of Relational Algebra like the **recipe steps** behind a restaurant dish. SQL is the menu item you order ("Chicken Biryani"). Relational Algebra is the actual sequence of kitchen operations (chop, marinate, layer, cook) that produces it. You don't need to know the recipe to eat the dish, but a chef (query optimizer) absolutely needs to know it to make the dish efficiently, substitute steps, or reorder them for speed.

---

## 1.2 Why Does Relational Algebra Exist?

| Problem it Solves | Explanation |
|---|---|
| **Formal foundation** | Before RA (proposed by Edgar F. Codd in 1970), there was no mathematically rigorous way to describe "what a query does." RA gave databases a provable, unambiguous semantics. |
| **Query optimization** | Databases need to prove that two different query plans return the *same* result before choosing the cheaper one. RA's mathematical laws (commutativity, associativity, distributivity) let optimizers safely rewrite queries. |
| **Language design benchmark** | Codd proved that any query language is "relationally complete" if it can express everything RA can express. SQL, QUEL, and other languages are judged against this benchmark. |
| **Teaching & reasoning** | RA gives students and engineers a step-by-step procedural way to reason about *how* a result is produced, rather than only *what* is wanted (which is what SQL, a declarative language, expresses). |

### Declarative vs Procedural — Critical Distinction

```
SQL (Declarative)                 Relational Algebra (Procedural)
─────────────────                 ────────────────────────────────
"WHAT do I want?"                 "HOW do I get it, step by step?"

SELECT name FROM employees        π(name) ( σ(salary > 50000) (Employees) )
WHERE salary > 50000;

You don't specify the order       You explicitly specify: FIRST filter,
of operations.                    THEN project.
```

This is one of the most commonly asked **conceptual interview questions**: *"Is SQL a Relational Algebra engine or a Relational Calculus engine?"* Answer: **SQL is primarily based on Tuple Relational Calculus in its declarative syntax, but is executed internally by converting it into Relational Algebra**, because RA is procedural and directly executable step-by-step.

---

## 1.3 Mathematical Foundation

A **relation** in RA is formally a subset of the Cartesian product of a set of domains, i.e., a **set of tuples**. Two mathematical facts define everything about RA:

1. **A relation is a set** → therefore **RA operations are set operations**, and (in pure theory) duplicate rows do not exist. (SQL relaxes this — see the "Bag vs Set" note below.)
2. **RA is closed** → the output of every operator is itself a valid relation, so operators can be composed/nested arbitrarily.

> ⚠️ **Bag vs Set — a critical, frequently-tested nuance**
> Pure Relational Algebra operates on **sets** (no duplicates ever). SQL tables are actually **bags/multisets** (duplicates allowed unless you explicitly use `DISTINCT`). This is *the* number one gap between textbook RA and real SQL behavior, and interviewers love asking about it. We'll flag every operator below where this matters.

### Notation Primer

| Symbol | Meaning | Read as |
|---|---|---|
| σ | Selection | "sigma" |
| π | Projection | "pi" |
| ρ | Rename | "rho" |
| ∪ | Union | "union" |
| ∩ | Intersection | "intersect" |
| − | Set Difference | "minus" |
| × | Cartesian Product | "cross" |
| ⋈ | Join | "join" |
| ÷ | Division | "divide" |
| ← | Assignment | "assign" |

---

## 1.4 Unary vs Binary Operations

| Type | Definition | Operators |
|---|---|---|
| **Unary** | Takes exactly **one** relation as input | Selection (σ), Projection (π), Rename (ρ) |
| **Binary** | Takes exactly **two** relations as input | Union, Intersection, Difference, Cartesian Product, all Joins, Division |

**Union-Compatibility Rule (critical prerequisite for binary set operators):** Two relations R and S can be combined with ∪, ∩, or − **only if**:
1. Both have the **same number of attributes (degree)**.
2. Corresponding attributes have **compatible (same) domains** (data types).

> If R has columns `(id: int, name: varchar)` and S has `(emp_id: int, emp_name: varchar)`, they ARE union-compatible — **column names don't need to match**, only the number and domain/type of columns.

---

## 1.5 Unary Operators

### 1.5.1 Selection (σ) — "Filter Rows"

**What it is:** Selection picks a **subset of rows (tuples)** from a relation that satisfy a given **predicate (condition)**.

**Formal notation:**
```
σ_condition (Relation)
```

**Sample Table — `Employees`**

| emp_id | name | dept | salary | age |
|---|---|---|---|---|
| 1 | Aditi | Sales | 55000 | 29 |
| 2 | Rohan | IT | 72000 | 34 |
| 3 | Kavya | IT | 68000 | 27 |
| 4 | Suresh | HR | 40000 | 45 |
| 5 | Meera | Sales | 61000 | 31 |

**Example:**
```
σ_(dept = 'IT') (Employees)
```

**Result:**

| emp_id | name | dept | salary | age |
|---|---|---|---|---|
| 2 | Rohan | IT | 72000 | 34 |
| 3 | Kavya | IT | 68000 | 27 |

**Equivalent SQL:**
```sql
SELECT * FROM Employees WHERE dept = 'IT';
```

**Compound predicates** use standard logical connectives ∧ (AND), ∨ (OR), ¬ (NOT):
```
σ_(dept = 'IT' ∧ salary > 70000) (Employees)
```

#### Internal Working

Conceptually, selection performs a **full linear scan**: for each tuple `t` in R, evaluate the boolean predicate; if true, include `t` in the output; else discard it. This is O(n) in the number of tuples **unless an index exists** on the filtered column, in which case the execution engine can use an **index scan** instead of a **sequential scan** — this is where RA theory meets real performance (cross-reference: **Indexing**, covered in Part 8).

```
Execution Tree:

        σ_(dept='IT')
             │
        Employees
```

#### Mathematical Intuition
Selection corresponds to set-builder notation from set theory:
```
σ_p(R) = { t | t ∈ R  ∧  p(t) is true }
```

#### Real-life Analogy
Selection is like a bouncer at a club checking IDs — everyone (every row) walks past, but only those who satisfy the condition ("age ≥ 21") are let through (kept in the output).

#### Advantages
- Simple, cheap conceptually.
- Highly optimizable via indexes.
- Commutative with itself: `σ_p1(σ_p2(R)) = σ_p2(σ_p1(R))` — the optimizer can reorder filters freely.

#### Disadvantages / Limitations
- Cannot change columns — only rows.
- On huge tables without an index, it's a full scan → expensive.

#### Best Practices
- Push selection as early as possible in a query plan (**predicate pushdown**) — filter before joining, not after.
- Use **sargable** predicates (conditions that can use an index, e.g., avoid wrapping indexed columns in functions like `WHERE YEAR(date_col) = 2024`).

#### Common Mistakes
- Forgetting `NULL` handling: `salary > 50000` **excludes** rows where salary is `NULL`, because `NULL > 50000` evaluates to `UNKNOWN`, not `TRUE`.
- Applying selection *after* an expensive join instead of before (bad for performance, though the optimizer often fixes this automatically).

#### Edge Cases
- Selection with a condition that matches zero rows returns an **empty relation with the same schema** — not an error.
- Selection combined with `NULL` comparisons needs `IS NULL` / `IS NOT NULL`, not `= NULL`.

#### Performance Implication
Selectivity (the fraction of rows a predicate keeps) directly drives the optimizer's join-order decisions — highly selective filters (few rows survive) are pushed as early as possible.

#### Interview Importance: ⭐⭐⭐⭐⭐ (Very High)
Almost every "convert this SQL to relational algebra" interview question begins with identifying the σ operator.

---

### 1.5.2 Projection (π) — "Select Columns"

**What it is:** Projection extracts a **subset of columns (attributes)** from a relation, producing a new relation with only those columns.

**Formal notation:**
```
π_(col1, col2, ...) (Relation)
```

**Example:**
```
π_(name, salary) (Employees)
```

**Result:**

| name | salary |
|---|---|
| Aditi | 55000 |
| Rohan | 72000 |
| Kavya | 68000 |
| Suresh | 40000 |
| Meera | 61000 |

**Equivalent SQL:**
```sql
SELECT name, salary FROM Employees;
```

#### ⚠️ Critical Rule: Duplicate Elimination
In **pure Relational Algebra**, projection **automatically removes duplicate rows** in its output (because a relation is a *set*). 

Example: `π_(dept)(Employees)` on our table would mathematically yield only `{Sales, IT, HR}` — **3 rows**, not 5, even though 5 employees exist.

```sql
-- SQL equivalent REQUIRES an explicit DISTINCT to match pure RA semantics:
SELECT DISTINCT dept FROM Employees;

-- Plain SQL SELECT does NOT remove duplicates (bag semantics):
SELECT dept FROM Employees;  -- returns 5 rows, RA would return 3
```

> 🎯 **This exact gap is one of the most frequently tested interview/exam trick questions.** Always mention it explicitly when discussing projection.

#### Internal Working
For each tuple, extract only the specified attribute values, then (in pure theory) run a duplicate-elimination pass — typically implemented via **sorting** or **hashing** internally when a real engine needs strict set semantics (e.g., `DISTINCT`, `UNION`).

```
Execution Tree:

     π_(name, salary)
             │
        Employees
```

#### Mathematical Intuition
```
π_(A1,...,An)(R) = { t[A1,...,An] | t ∈ R }
```
where `t[A1,...,An]` denotes the tuple `t` restricted to only those attributes.

#### Real-life Analogy
Projection is like photocopying a spreadsheet but covering up (masking) all columns except the two or three you care about — you get a narrower sheet, and if two rows become identical after masking, pure math treats them as "the same row."

#### Advantages
- Reduces data volume transferred/processed (I/O and network savings) — this is why `SELECT *` is discouraged; selecting only needed columns is a real performance best practice.
- Enables **covering indexes** (index-only scans) when the projected + filtered columns are all present in an index.

#### Disadvantages
- Duplicate elimination (when required) needs sorting/hashing, which is O(n log n) — a real cost when `DISTINCT` is used on large data.

#### Best Practices
- Avoid `SELECT *` in production code — explicit projection lists are more index-friendly, more maintainable, and cheaper.
- Use `DISTINCT` only when duplicates are truly undesired — it's not free.

#### Common Mistakes
- Assuming plain SQL `SELECT` removes duplicates like textbook RA does — it does not.
- Projecting a column not needed downstream, bloating result sets unnecessarily.

#### Edge Cases
- Projecting zero columns is invalid in SQL (`SELECT ;` is a syntax error) but conceptually would yield a degree-0 relation in RA.
- Projection on a column that has `NULL` values keeps the `NULL`s (they are not treated as duplicates of each other in most systems' `DISTINCT` implementations — actually, `DISTINCT` treats two `NULL`s as duplicates of each other in SQL standard, which is itself a special-cased edge behavior worth remembering).

#### Performance Implication
Column pruning (projecting early) is a top-3 optimizer rewrite rule — reduces memory footprint through the entire remaining query pipeline.

#### Interview Importance: ⭐⭐⭐⭐⭐

---

### Combining Selection + Projection

```
π_(name, salary) ( σ_(dept = 'IT') (Employees) )
```

**Execution Tree:**
```
        π_(name, salary)
               │
        σ_(dept = 'IT')
               │
          Employees
```

**Equivalent SQL:**
```sql
SELECT name, salary FROM Employees WHERE dept = 'IT';
```

**Result:**

| name | salary |
|---|---|
| Rohan | 72000 |
| Kavya | 68000 |

> **Order matters conceptually, not just visually.** σ must be applied *before* π here, because once you project away `dept`, you can no longer filter by it. This is a foundational rule: **always filter (σ) using columns before projecting (π) them away**, unless the filter doesn't need that column at all.

---

## 1.6 Set-Theoretic Binary Operators

All four operators below require **union-compatibility** (Section 1.4).

### 1.6.1 Union (∪)

**Definition:** Combines all tuples from R and S, **removing duplicates**.

```
R ∪ S = { t | t ∈ R  ∨  t ∈ S }
```

**Sample Tables:**

`Managers` (emp_id, name)

| emp_id | name |
|---|---|
| 2 | Rohan |
| 4 | Suresh |

`TeamLeads` (emp_id, name)

| emp_id | name |
|---|---|
| 4 | Suresh |
| 5 | Meera |

**Query:** `Managers ∪ TeamLeads`

**Result:**

| emp_id | name |
|---|---|
| 2 | Rohan |
| 4 | Suresh |
| 5 | Meera |

(Note: (4, Suresh) appears in both source tables but only **once** in the output — duplicate elimination.)

**SQL:**
```sql
SELECT emp_id, name FROM Managers
UNION
SELECT emp_id, name FROM TeamLeads;
```

### 1.6.2 Intersection (∩)

**Definition:** Tuples present in **both** R and S.
```
R ∩ S = { t | t ∈ R  ∧  t ∈ S }
```
**Query:** `Managers ∩ TeamLeads` → **Result:** `{(4, Suresh)}`

**SQL:**
```sql
SELECT emp_id, name FROM Managers
INTERSECT
SELECT emp_id, name FROM TeamLeads;
```

### 1.6.3 Set Difference (−)

**Definition:** Tuples in R but **not** in S. **Not commutative**: `R − S ≠ S − R`.
```
R − S = { t | t ∈ R  ∧  t ∉ S }
```
**Query:** `Managers − TeamLeads` → **Result:** `{(2, Rohan)}`
**Query:** `TeamLeads − Managers` → **Result:** `{(5, Meera)}`

**SQL:**
```sql
SELECT emp_id, name FROM Managers
EXCEPT   -- MINUS in Oracle
SELECT emp_id, name FROM TeamLeads;
```

### 1.6.4 Cartesian Product (×)

**Definition:** Every tuple of R combined with every tuple of S — **all possible combinations**. If R has `m` rows and S has `n` rows, `R × S` has `m × n` rows and `degree(R) + degree(S)` columns.

```
R × S = { (t_r, t_s) | t_r ∈ R  ∧  t_s ∈ S }
```

**Sample:**

`Departments` (dept_id, dept_name)

| dept_id | dept_name |
|---|---|
| D1 | Sales |
| D2 | IT |

`Locations` (loc_id, city)

| loc_id | city |
|---|---|
| L1 | Bengaluru |
| L2 | Pune |

`Departments × Locations` → **2 × 2 = 4 rows:**

| dept_id | dept_name | loc_id | city |
|---|---|---|---|
| D1 | Sales | L1 | Bengaluru |
| D1 | Sales | L2 | Pune |
| D2 | IT | L1 | Bengaluru |
| D2 | IT | L2 | Pune |

**SQL:**
```sql
SELECT * FROM Departments CROSS JOIN Locations;
-- or, equivalently in most engines:
SELECT * FROM Departments, Locations;
```

> ⚠️ **This is the #1 cause of accidental "Cartesian explosions."** If you write `SELECT * FROM A, B WHERE ...` and forget the `WHERE` join condition, you silently get every combination — a table with 1,000 rows joined this way with another 1,000-row table produces **1,000,000 rows**. This is covered in depth in Module 13 (Common SQL Pitfalls).

**Why Cartesian Product exists in the algebra despite being "dangerous":** it is the **mathematical foundation for all Joins** — every join is formally definable as a Cartesian Product followed by a Selection (see Section 1.8).

---

## 1.7 Rename (ρ)

**Definition:** Renames a relation and/or its attributes, **without changing the data**. Necessary because RA (being math) needs every intermediate relation to have unambiguous names — especially when joining a table to itself.

**Notation:**
```
ρ_NewName (R)                          -- rename the relation
ρ_NewName(A1, A2, ...) (R)             -- rename relation AND its attributes
```

**Example — Self-Join Scenario (Employees reporting to Employees):**

`Employees` (emp_id, name, manager_id)

To find each employee's manager's name, we conceptually need **two copies** of `Employees`:
```
ρ_E1(Employees)  and  ρ_E2(Employees)
```
Then:
```
π_(E1.name, E2.name) ( σ_(E1.manager_id = E2.emp_id) (E1 × E2) )
```

**Equivalent SQL:**
```sql
SELECT e1.name AS employee, e2.name AS manager
FROM Employees e1
JOIN Employees e2 ON e1.manager_id = e2.emp_id;
```

#### Why it Exists
Without renaming, `E1.emp_id` and `E2.emp_id` in the Cartesian product would be indistinguishable — the expression would be mathematically ambiguous. Rename disambiguates **exactly** the way SQL table aliases (`AS e1`, `AS e2`) do.

#### Interview Importance: ⭐⭐⭐⭐ — Self-joins and the role of ρ are a classic FAANG whiteboard question ("find the manager of each employee").

---

## 1.8 The Join Family

All joins are **derived operators** — meaning they can all be expressed using the primitive operators above (×, σ, π, ρ), but are given their own symbol/keyword because they are used so often that databases optimize them specially (via hash joins, merge joins, nested-loop joins internally — covered in Part 8: Query Execution Internals).

### 1.8.0 The Universal Join Law

```
R ⋈_θ S   ≡   σ_θ ( R × S )
```

**Every join is "Cartesian Product, then filter."** This single identity is the master key to understanding the entire join family and is a favorite interview proof question: *"Prove that Theta Join can be expressed using only primitive RA operators."*

### 1.8.1 Theta Join (⋈_θ)

**Definition:** A Cartesian Product filtered by an **arbitrary condition θ** (any comparison operator: `=, ≠, <, >, ≤, ≥`).

```
R ⋈_θ S = σ_θ (R × S)
```

**Example:** Join Employees and SalaryBands where an employee's salary falls in the band's range.
```
Employees ⋈_(salary BETWEEN band_min AND band_max) SalaryBands
```
**SQL:**
```sql
SELECT * FROM Employees e
JOIN SalaryBands s ON e.salary BETWEEN s.band_min AND s.band_max;
```

### 1.8.2 Equi Join

**Definition:** A special case of Theta Join where θ uses **only equality (=)**.
```
Employees ⋈_(Employees.dept_id = Departments.dept_id) Departments
```
This is the **most common join in practice** — powers almost every foreign-key-based join.

### 1.8.3 Natural Join (⋈)

**Definition:** An Equi Join performed **automatically on all commonly-named columns**, with the duplicate column appearing **only once** in the output (unlike Equi Join, which keeps both copies).

```
R ⋈ S
```

**Sample:**

`Employees` (emp_id, name, dept_id) ⋈ `Departments` (dept_id, dept_name)

**SQL:**
```sql
-- True Natural Join:
SELECT * FROM Employees NATURAL JOIN Departments;

-- Manually-written equivalent (what most engineers actually write):
SELECT * FROM Employees e JOIN Departments d ON e.dept_id = d.dept_id;
```

> ⚠️ **Danger of `NATURAL JOIN` in real SQL:** it joins on **every** identically-named column automatically. If both tables happen to also share an unrelated column like `updated_at`, the join silently includes it in the condition, corrupting results. **Most style guides ban `NATURAL JOIN` in production code** for exactly this reason — prefer explicit `JOIN ... ON`.

### 1.8.4 Outer Joins

Outer joins preserve **unmatched rows** from one or both sides, filling missing columns with `NULL`.

| Join Type | RA Symbol | Keeps unmatched rows from |
|---|---|---|
| Left Outer Join | ⟕ | Left relation (R) |
| Right Outer Join | ⟖ | Right relation (S) |
| Full Outer Join | ⟗ | Both relations |

```
                LEFT OUTER JOIN                    FULL OUTER JOIN
   R                    S              R                    S
┌─────┐             ┌─────┐         ┌─────┐             ┌─────┐
│ A B │             │ A C │         │ A B │             │ A C │
│ 1 x │──matched───▶│ 1 p │         │ 1 x │──matched───▶│ 1 p │
│ 2 y │  no match    │ 3 q │         │ 2 y │  no match    │ 3 q │◀──no match
└─────┘   (kept,     └─────┘         └─────┘   (both kept,└─────┘  (kept, C-side
          C=NULL)                              NULLs filled          B=NULL)
                                                on the missing side)
```

**SQL:**
```sql
SELECT * FROM Employees e LEFT JOIN  Departments d ON e.dept_id = d.dept_id;
SELECT * FROM Employees e RIGHT JOIN Departments d ON e.dept_id = d.dept_id;
SELECT * FROM Employees e FULL JOIN  Departments d ON e.dept_id = d.dept_id;
```

> 🐬 **MySQL Note:** MySQL (up to and including current versions) does **not** support `FULL OUTER JOIN` natively. The standard workaround is:
> ```sql
> SELECT * FROM Employees e LEFT JOIN Departments d ON e.dept_id = d.dept_id
> UNION
> SELECT * FROM Employees e RIGHT JOIN Departments d ON e.dept_id = d.dept_id;
> ```
> PostgreSQL, SQL Server, and Oracle all support `FULL OUTER JOIN` directly. **This exact gap is a very common interview/practical question.**

### 1.8.5 Semi Join (⋉)

**Definition:** Returns rows from R that **have at least one match** in S, but **only R's columns appear in the output** (S's columns are discarded — used only for filtering).

```
R ⋉ S = π_(R's attributes) ( R ⋈ S )
```

**Real-world use:** "Find all customers who have placed **at least one** order" — you want customer data, not order data.

**SQL Equivalent (SQL has no direct `SEMI JOIN` keyword; expressed via `EXISTS` or `IN`):**
```sql
SELECT * FROM Customers c
WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.customer_id = c.customer_id);
```

> This is a critical bridge concept to **Module 3 (Advanced SQL)** where `EXISTS`/`IN` subqueries are covered in depth — they are the SQL surface syntax for Semi Join under the hood, and query optimizers frequently rewrite `IN`-subqueries into semi-joins internally.

### 1.8.6 Anti Join (▷, sometimes written ⋉̄)

**Definition:** The exact opposite of Semi Join — returns rows from R that have **no match** in S.

```
R ▷ S = R − (R ⋉ S)
```

**Real-world use:** "Find all customers who have **never** placed an order."

**SQL Equivalent:**
```sql
SELECT * FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.customer_id = c.customer_id);
```

> ⚠️ **Common Trap:** Using `NOT IN (SELECT ...)` instead of `NOT EXISTS` for anti-joins is dangerous — if the subquery returns even **one `NULL`**, `NOT IN` returns an **empty result for the entire query** (because `x NOT IN (1, 2, NULL)` evaluates to `UNKNOWN` for every `x`, not `TRUE`). `NOT EXISTS` does not have this problem. **This is one of the highest-value SQL interview gotchas — memorize it.**

### Join Family Summary Table

| Join | Keeps unmatched left? | Keeps unmatched right? | Removes duplicate join column? | Output columns |
|---|---|---|---|---|
| Theta Join | ❌ | ❌ | ❌ | R + S (all) |
| Equi Join | ❌ | ❌ | ❌ | R + S (all) |
| Natural Join | ❌ | ❌ | ✅ | R + S (deduped) |
| Left Outer | ✅ | ❌ | ❌ | R + S |
| Right Outer | ❌ | ✅ | ❌ | R + S |
| Full Outer | ✅ | ✅ | ❌ | R + S |
| Semi Join | only matched | n/a | n/a | R only |
| Anti Join | only unmatched | n/a | n/a | R only |

---

## 1.9 Division (÷)

**Definition:** Division is the RA operator behind the classic **"find X that relates to ALL of Y"** query pattern. Given `R(A, B)` and `S(B)`, `R ÷ S` returns all values of `A` such that **every** value of `B` in `S` is associated with that `A` in `R`.

```
R ÷ S = { a | ∀ b ∈ S, (a, b) ∈ R }
```

**Sample:**

`Enrolled` (student, course)

| student | course |
|---|---|
| Aditi | DBMS |
| Aditi | OS |
| Aditi | CN |
| Rohan | DBMS |
| Rohan | OS |
| Kavya | DBMS |
| Kavya | OS |
| Kavya | CN |

`RequiredCourses` (course)

| course |
|---|
| DBMS |
| OS |
| CN |

**Query:** `Enrolled ÷ RequiredCourses` → **Which students are enrolled in ALL required courses?**

**Result:**

| student |
|---|
| Aditi |
| Kavya |

(Rohan is excluded — missing `CN`.)

#### How to Express Division Using Primitive Operators (classic interview derivation)
```
R ÷ S  =  π_A(R)  −  π_A( (π_A(R) × S)  −  R )
```

**Equivalent SQL (the "double NOT EXISTS" pattern — extremely common interview question):**
```sql
SELECT DISTINCT e1.student
FROM Enrolled e1
WHERE NOT EXISTS (
    SELECT rc.course
    FROM RequiredCourses rc
    WHERE NOT EXISTS (
        SELECT 1 FROM Enrolled e2
        WHERE e2.student = e1.student AND e2.course = rc.course
    )
);
```
**Alternative, more common SQL pattern using `COUNT`/`HAVING` (also frequently expected in interviews):**
```sql
SELECT student
FROM Enrolled
WHERE course IN (SELECT course FROM RequiredCourses)
GROUP BY student
HAVING COUNT(DISTINCT course) = (SELECT COUNT(*) FROM RequiredCourses);
```

#### Real-life Analogy
Division answers: **"Which students have taken every course on this checklist?"** — like finding "which vendors stock every item on my shopping list."

#### Interview Importance: ⭐⭐⭐⭐⭐ — "Find X related to ALL Y" (students enrolled in all courses, employees who worked on all projects, customers who bought all products in a category) is an extremely common Senior SDE-2/SDE-3 SQL interview pattern (frequently seen on LeetCode Database, DataLemur, and in FAANG onsites).

---

## 1.10 Assignment Operator (←)

**Definition:** Assigns the result of an RA expression to a temporary relation variable, allowing complex expressions to be broken into readable steps — conceptually identical to a **CTE (`WITH` clause)** in SQL, which we cover in full in **Module 4**.

**Notation:**
```
TempResult ← σ_(dept = 'IT') (Employees)
FinalResult ← π_(name, salary) (TempResult)
```

**SQL Equivalent:**
```sql
WITH TempResult AS (
    SELECT * FROM Employees WHERE dept = 'IT'
)
SELECT name, salary FROM TempResult;
```

> This is the conceptual bridge to Module 4 — CTEs are literally SQL's syntax for the RA Assignment operator, making complex nested expressions readable by giving intermediate results names.

#### Best Practices
- Break down deeply nested RA expressions into assignment steps for readability — exactly as you'd break a nested subquery into a CTE.
- In interviews, using assignment notation for multi-step derivations (like Division above) shows structured thinking and is well-regarded.

---

## 1.11 SQL ↔ Relational Algebra Conversion Guide

### General Template

```
SQL:
SELECT  <columns>              →  π (projection, applied LAST)
FROM    <tables>                →  × or ⋈ (product/join, applied FIRST)
WHERE   <condition>             →  σ (selection, applied AFTER join)
GROUP BY / aggregate            →  𝔊 (grouping — extended RA, see Module 6)
ORDER BY                        →  τ (tau, sort — extended RA)
```

### Logical Order of Evaluation (Extremely High-Yield Interview Topic)

SQL's **written** order and **logical execution** order are famously different. This is one of the most important things to internalize:

```
Written Order:        Logical (Conceptual) Execution Order:

SELECT                 1. FROM        (+ JOINs → Cartesian Product/Join)
FROM                    2. WHERE       (σ - row filter, pre-aggregation)
WHERE                   3. GROUP BY    (grouping)
GROUP BY                4. HAVING      (σ - filter on groups)
HAVING                   5. SELECT      (π - column projection)
ORDER BY                6. DISTINCT    (duplicate elimination)
LIMIT                    7. ORDER BY    (τ - sort)
                          8. LIMIT       (row cap)
```

> 🎯 **Why this matters practically:** it explains WHY you cannot reference a `SELECT`-defined column alias inside a `WHERE` clause (WHERE executes before SELECT), but CAN reference it in `ORDER BY` (which executes after SELECT). This single diagram resolves a huge fraction of "why doesn't my SQL work" confusion and is a guaranteed interview/exam topic.

### Worked Conversion Example

**SQL:**
```sql
SELECT d.dept_name, COUNT(*) AS emp_count
FROM Employees e
JOIN Departments d ON e.dept_id = d.dept_id
WHERE e.salary > 50000
GROUP BY d.dept_name
HAVING COUNT(*) > 2
ORDER BY emp_count DESC;
```

**Relational Algebra Expression Tree:**
```
                    τ_(emp_count DESC)                    <- ORDER BY
                            │
                    σ_(COUNT > 2)                          <- HAVING
                            │
                    𝔊_(dept_name; COUNT(*)→emp_count)       <- GROUP BY + aggregate
                            │
                π_(dept_name, ...)                          <- (implicit, columns needed)
                            │
                    σ_(salary > 50000)                       <- WHERE
                            │
            ⋈_(e.dept_id = d.dept_id)                        <- JOIN
              /                        \
        Employees                  Departments
```

**Reading the tree bottom-up = the actual logical execution order.**

### Quick-Reference Conversion Table

| SQL Clause | RA Operator | Notes |
|---|---|---|
| `SELECT` | π (Projection) | Applied near the end |
| `SELECT DISTINCT` | π with dedup (pure RA default) | Plain SQL SELECT does NOT dedupe |
| `FROM t1, t2` | × (Cartesian Product) | Rare/dangerous if unfiltered |
| `FROM t1 JOIN t2 ON ...` | ⋈_θ or natural ⋈ | Preferred over comma-join |
| `WHERE` | σ (Selection) | Row-level filter, before grouping |
| `HAVING` | σ (Selection) | Group-level filter, after grouping |
| `GROUP BY` | 𝔊 (Grouping, extended RA) | Covered in Module 6 |
| `ORDER BY` | τ (Tau, extended RA) | Sort, not part of Codd's original 8 |
| `UNION` | ∪ | Dedupes |
| `UNION ALL` | ∪ (bag union) | No dedupe — matches SQL bag semantics best |
| `INTERSECT` | ∩ | |
| `EXCEPT`/`MINUS` | − | Not commutative |
| `IN (subquery)` / `EXISTS` | ⋉ (Semi Join) | |
| `NOT IN` / `NOT EXISTS` | ▷ (Anti Join) | Prefer `NOT EXISTS` (NULL-safe) |

---

## 1.12 PostgreSQL vs MySQL — Relational Algebra Related Differences

| Feature | PostgreSQL | MySQL |
|---|---|---|
| `FULL OUTER JOIN` | ✅ Supported natively | ❌ Not supported — must simulate via `UNION` of `LEFT` + `RIGHT` JOIN |
| `INTERSECT` | ✅ Supported | ✅ Supported (MySQL 8.0.31+); not in older versions |
| `EXCEPT` | ✅ Supported | ✅ Supported as `EXCEPT` (MySQL 8.0.31+); older versions need `LEFT JOIN ... WHERE NULL` workaround |
| `NATURAL JOIN` | ✅ Supported | ✅ Supported (but discouraged in both) |
| `LATERAL` joins (for correlated derived tables) | ✅ Full support | ✅ Supported from MySQL 8.0.14+ |
| Duplicate handling in `UNION` | Standard SQL (dedupes) | Standard SQL (dedupes) |
| `CROSS JOIN` syntax | ✅ | ✅ |

> **Note:** Always verify current version-specific behavior against official docs before relying on this table for a specific production version — vendor SQL dialects evolve.

---

## 1.13 Common Mistakes & Edge Cases (Relational Algebra)

| Mistake | Why it's Wrong | Fix |
|---|---|---|
| Forgetting union-compatibility before ∪/∩/− | Mismatched degree/domain → undefined operation | Ensure same number & type of columns before combining |
| Treating SQL `SELECT` as auto-deduplicating like π | Produces unexpected duplicate rows | Use `DISTINCT` explicitly when set semantics are required |
| Writing `A, B` in `FROM` without a join condition | Silent Cartesian explosion | Always pair comma-joins with an explicit `WHERE` join predicate, or better, use explicit `JOIN ... ON` |
| Using `NATURAL JOIN` on tables with coincidentally-matching column names | Joins on unintended columns | Use explicit `JOIN ... ON` with named columns |
| Using `NOT IN` for anti-join logic | Breaks silently if subquery has `NULL` | Use `NOT EXISTS` |
| Assuming R − S = S − R | Set difference is not commutative | Always check direction of subtraction |
| Confusing Theta Join with Equi Join in an interview | Equi Join is a *subset* of Theta Join (only `=`) | State the definition precisely |

---

## 1.14 Interview Questions & MCQs

### Conceptual (Beginner → Intermediate)

1. What is the difference between Relational Algebra and Relational Calculus? *(Full answer in Module 2.)*
2. Why is Relational Algebra called "procedural" while SQL is called "declarative"?
3. What does "closure" mean in Relational Algebra, and why is it important?
4. What is union-compatibility? Give an example of two tables that are NOT union-compatible.
5. Prove that `R ⋈_θ S = σ_θ(R × S)`.

### Applied (Intermediate → Advanced, Senior SDE-2/SDE-3)

6. Convert this SQL to a Relational Algebra expression tree: *(a 3-table join with WHERE, GROUP BY, HAVING — practice using the worked example in 1.11 as a template)*.
7. Write the RA expression, then the SQL, to find employees who have worked on **every** project in the company (Division operator pattern).
8. Explain why `NOT IN` can silently return zero rows in the presence of `NULL`s, using Anti Join semantics.
9. Why does MySQL not need a native `FULL OUTER JOIN` implementation to still support full-outer-join *semantics*? Write the workaround query.
10. Given `Employees(emp_id, name, manager_id)`, write the RA expression (using ρ) to list every employee alongside their manager's name.

### MCQs (Practice — answers below)

**Q1.** Which RA operator is used to eliminate columns from a relation?
 a) σ  b) π  c) ρ  d) ×

**Q2.** `R − S` is:
 a) Commutative  b) Not commutative  c) Same as S − R always  d) Undefined

**Q3.** Which of these is NOT a primitive (fundamental) RA operator?
 a) Selection  b) Natural Join  c) Union  d) Cartesian Product

**Q4.** In pure Relational Algebra, does Projection remove duplicate rows automatically?
 a) Yes, always  b) No, never  c) Only with DISTINCT keyword  d) Only in PostgreSQL

**Q5.** Which join type is used to find "customers with no orders"?
 a) Semi Join  b) Natural Join  c) Anti Join  d) Theta Join

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) π** — Projection selects columns; Selection (σ) selects rows.
2. **b) Not commutative** — `R − S ≠ S − R` in general.
3. **b) Natural Join** — it's a *derived* operator (built from ⋈_θ + π), not one of Codd's original primitives (σ, π, ∪, −, ×, ρ).
4. **a) Yes, always** — pure RA relations are sets; SQL's plain `SELECT` differs (bag semantics).
5. **c) Anti Join** — rows in R with no match in S.

</details>

---

## 1.15 Cheat Sheet — Module 1

```
┌───────────────────────────────────────────────────────────────────┐
│                  RELATIONAL ALGEBRA — QUICK REFERENCE               │
├───────────────┬───────────────────────────────────────────────────┤
│ σ_p(R)        │ Rows of R where predicate p is true    (WHERE)     │
│ π_A(R)        │ Columns A of R, duplicates removed      (SELECT)   │
│ ρ_X(R)        │ Rename R to X                            (alias)   │
│ R ∪ S         │ All rows in R or S, deduped              (UNION)   │
│ R ∩ S         │ Rows in both R and S                 (INTERSECT)   │
│ R − S         │ Rows in R but not S                    (EXCEPT)    │
│ R × S         │ All combinations of R and S rows    (CROSS JOIN)   │
│ R ⋈_θ S       │ σ_θ(R × S)                          (JOIN ... ON)  │
│ R ⋈ S         │ Natural join (auto-match common cols) (NATURAL JOIN)│
│ R ⟕ S         │ Left outer join                       (LEFT JOIN) │
│ R ⟖ S         │ Right outer join                     (RIGHT JOIN) │
│ R ⟗ S         │ Full outer join                       (FULL JOIN) │
│ R ⋉ S         │ Semi join — R rows w/ a match in S      (EXISTS)   │
│ R ▷ S         │ Anti join — R rows w/ NO match in S  (NOT EXISTS)  │
│ R ÷ S         │ A-values related to ALL B-values in S  (GROUP BY/  │
│               │                                          HAVING COUNT)│
│ X ← expr      │ Assignment, name an intermediate result   (WITH)   │
└───────────────┴───────────────────────────────────────────────────┘

REMEMBER:
• Every join = Cartesian Product + Selection.
• Pure RA = SET semantics (no duplicates). SQL default = BAG semantics.
• Logical SQL execution order: FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
• NOT EXISTS is NULL-safe; NOT IN is NOT NULL-safe. Prefer NOT EXISTS for anti-joins.
```

---

## 1.16 Exercises

### Beginner
1. Given `Students(id, name, marks)`, write RA and SQL to find students with `marks > 90`.
2. Write RA and SQL to project only `name` and `marks` from the same table, with duplicates removed.

### Intermediate
3. Given `Orders(order_id, customer_id, amount)` and `Customers(customer_id, name)`, write the RA expression for an equi-join followed by a filter for `amount > 1000`, then convert to SQL.
4. Explain, with an example, why `A NATURAL JOIN B` can silently produce wrong results if both tables have an unrelated column with the same name.

### Advanced / Window into Later Modules
5. Given `Enrolled(student, course)` and `AllCourses(course)`, write both the RA Division expression and the two SQL equivalents (double-`NOT EXISTS` and `COUNT`/`HAVING`) for "students enrolled in every course."
6. Given `Employees(emp_id, name, manager_id)`, write the full RA expression (using ρ, ×, σ, π) to list `(employee_name, manager_name)` pairs, then the equivalent self-join SQL.

### Output Prediction
7. Given `R = {(1,'a'), (2,'b')}` and `S = {(2,'b'), (3,'c')}` (union-compatible), what is the exact output (as a set of tuples) of: `R ∪ S`, `R ∩ S`, `R − S`, `S − R`?

### Debugging
8. A junior engineer wrote:
   ```sql
   SELECT * FROM Employees, Departments WHERE Employees.name = 'Rohan';
   ```
   intending to get Rohan's department info. What's wrong, and what will actually happen? Fix it.

### Scenario-Based (Interview Style)
9. You're asked in an interview: *"Find all products that have been ordered by every customer in the 'VIP' segment."* Identify which RA operator this maps to, and write the SQL.

### Mini Project
10. Take any 3-table schema of your choice (e.g., `Students`, `Courses`, `Enrollments`). Write five different queries, each using a different RA operator family (selection, projection, join, set operation, division), and draw the RA expression tree for each by hand before writing the SQL.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [SQLBolt — Interactive SQL Lessons](https://sqlbolt.com/)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

- Relational Algebra is a **procedural**, mathematically **closed** query language built from a small set of primitive operators: σ, π, ρ, ∪, ∩, −, ×, plus derived operators (joins, division).
- Every **join** is fundamentally a **Cartesian Product followed by a Selection** — this single identity underlies query optimization.
- **Pure RA uses set semantics** (no duplicates); **SQL defaults to bag semantics** — this gap causes some of the most common real-world bugs and is a top interview topic.
- **Division (÷)** is the operator behind "find X related to ALL Y" queries — a very high-frequency interview pattern.
- SQL's **written syntax order** and **logical execution order** differ; understanding `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY` is essential for writing correct SQL and for reasoning about aliases and performance.
- `NOT EXISTS` (Anti Join) is the NULL-safe way to express "has no match," in contrast to the NULL-unsafe `NOT IN`.

---

## Cross-References for Later Parts
- **Indexing & Access Paths** → Part 8 (how σ/π are physically executed: sequential scan vs index scan vs index-only scan)
- **Join Algorithms** (Nested Loop, Hash Join, Merge Join) → Part 8 (how ⋈ is physically executed)
- **Query Optimizer internals** (cost-based optimization, join reordering using RA's algebraic laws) → Part 8
- **Functional Dependencies & Normalization** → Module 16 preview in this Part, full treatment in **Part 5**
- **Transactions & Isolation** → Part 6/7
- **CTEs as the SQL form of the Assignment Operator** → Module 4 (this Part)
- **GROUP BY / Aggregation as extended RA (𝔊)** → Module 6 (this Part)

---

> ✅ **Module 1 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 2"** to proceed to **Module 2: Relational Calculus** (Predicate Logic, Tuple Relational Calculus, Domain Relational Calculus, Safe/Unsafe Expressions).
