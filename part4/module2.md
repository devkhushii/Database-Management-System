# The Complete DBMS Handbook — Part 4

## 📘 MODULE 2: RELATIONAL CALCULUS

> **Continuing from Module 1 (Relational Algebra).** If you haven't read Module 1, please do so first — this module constantly contrasts Calculus against Algebra, and assumes you know σ, π, ⋈, and the RA execution-tree mental model.

---

## 2.0 Chapter Roadmap

```
Module 2: Relational Calculus
│
├── 2.1 What is Relational Calculus?
├── 2.2 Why does it exist?
├── 2.3 Predicate Logic Primer (prerequisite)
├── 2.4 Tuple Relational Calculus (TRC)
├── 2.5 Domain Relational Calculus (DRC)
├── 2.6 Safe vs Unsafe Expressions
├── 2.7 Algebra vs Calculus — Full Comparison
├── 2.8 Relationship to SQL
├── 2.9 Common Mistakes & Edge Cases
├── 2.10 Interview Questions & MCQs
├── 2.11 Cheat Sheet
└── 2.12 Exercises
```

---

## 2.1 What is Relational Calculus?

**Definition:** Relational Calculus (RC) is a **declarative**, non-procedural query language. Instead of specifying *how* to compute the answer step-by-step (as Algebra does), you specify *what* the answer must look like, using **formal logic (predicate calculus)** — you describe the *properties* the required tuples must satisfy, and let the system figure out how to actually retrieve them.

There are two equivalent flavors:
- **Tuple Relational Calculus (TRC)** — variables range over **tuples** (whole rows).
- **Domain Relational Calculus (DRC)** — variables range over **domain values** (individual column values).

> **Real-life analogy:** Relational Algebra is like giving someone turn-by-turn driving directions ("go straight 2km, turn left, turn right"). Relational Calculus is like giving someone a destination address and letting their GPS/brain figure out the route ("go to 221B Baker Street"). Both get you there — but Calculus only describes *where*, not *how*.

### Why This Matters for SQL Engineers

**SQL's `SELECT ... WHERE` syntax is directly modeled on Tuple Relational Calculus.** When you write:
```sql
SELECT name FROM Employees WHERE salary > 50000;
```
you are essentially saying, in TRC-style thinking: *"Give me the `name` of every tuple `t` in Employees such that `t.salary > 50000`."* You are describing **what** you want (a property), not **how** to scan/filter/loop — that procedural translation is the job of the query optimizer, which converts your declarative SQL into a procedural Relational Algebra execution plan (recall the pipeline diagram from Module 1, Section 1.1).

```
   You write (declarative, Calculus-flavored):
        SELECT name FROM Employees WHERE salary > 50000

   The database internally converts this to
   (procedural, Algebra-flavored) for execution:
        π_(name) ( σ_(salary > 50000) (Employees) )
```

This is one of the most important "aha" realizations in a DBMS course: **SQL LOOKS like Tuple Relational Calculus on the surface, but EXECUTES like Relational Algebra underneath.**

---

## 2.2 Why Does Relational Calculus Exist?

| Problem it Solves | Explanation |
|---|---|
| **Declarative query specification** | Lets users state *what* they want without knowing internal storage/execution details — this is the foundation of all modern declarative query languages, including SQL. |
| **Expressiveness benchmark** | Codd used RC to define **"relational completeness"** — a language is relationally complete if it can express anything TRC/DRC can express. RA was later proven equivalent in power (Codd's Theorem, Section 2.7). |
| **Formal correctness / logic-based reasoning** | RC connects databases to **first-order predicate logic**, letting database theorists prove query properties (equivalence, safety, satisfiability) using well-established logic tools. |
| **User-friendliness** | Business users/analysts think in terms of "what conditions must the data satisfy," not "which operator sequence produces it" — RC (and by extension SQL) matches this natural thought pattern. |

---

## 2.3 Predicate Logic Primer (Prerequisite)

Before TRC/DRC make sense, you need basic familiarity with **First-Order Predicate Logic**:

| Symbol | Name | Meaning |
|---|---|---|
| ∧ | AND (conjunction) | Both conditions must be true |
| ∨ | OR (disjunction) | At least one condition must be true |
| ¬ | NOT (negation) | Condition must be false |
| → | Implication | "if...then" (`P → Q` is equivalent to `¬P ∨ Q`) |
| ∃ | Existential quantifier | "there EXISTS at least one..." |
| ∀ | Universal quantifier | "for ALL..." |
| t ∈ R | Set membership | tuple `t` belongs to relation `R` |

### Quantifiers — The Most Important Concept in This Module

**Existential (∃):** `∃ t ∈ R (P(t))` means *"there is at least one tuple t in R for which P(t) is true."* This directly maps to SQL's `EXISTS`.

**Universal (∀):** `∀ t ∈ R (P(t))` means *"P(t) is true for every single tuple t in R."* This directly maps to the "Division" pattern from Module 1 and to SQL's `NOT EXISTS (SELECT ... WHERE NOT ...)` double-negation pattern.

> **Critical logic identity (very frequently tested):** Universal quantification can always be rewritten using existential quantification and negation:
> ```
> ∀ t (P(t))   ≡   ¬ ∃ t (¬P(t))
> ```
> **In plain English:** *"P is true for ALL t"* is logically identical to *"there does NOT EXIST a t for which P is false."* This is **exactly** why SQL has no native `FOR ALL` keyword — every "for all" query must be rewritten as a double `NOT EXISTS`, which is precisely what you did for the Division operator in Module 1!

```
"All employees earn more than 30000"
    ∀ e ∈ Employees (e.salary > 30000)
        ≡
    ¬ ∃ e ∈ Employees (¬(e.salary > 30000))
        ≡
    ¬ ∃ e ∈ Employees (e.salary ≤ 30000)

SQL:
SELECT CASE WHEN NOT EXISTS (
    SELECT 1 FROM Employees WHERE salary <= 30000
) THEN 'TRUE' ELSE 'FALSE' END;
```

### Free vs Bound Variables

- A variable is **bound** if it is governed by a quantifier (∃ or ∀).
- A variable is **free** if it is not — free variables are the ones whose values actually appear in the final query result.

```
{ t | t ∈ Employees ∧ ∃ d ∈ Departments (t.dept_id = d.dept_id ∧ d.name = 'IT') }
      ▲                                  ▲
      free (this defines the output)     bound (governed by ∃, just used internally to check a condition)
```

---

## 2.4 Tuple Relational Calculus (TRC)

**Definition:** In TRC, a query has the general form:
```
{ t | P(t) }
```
Read as: *"the set of all tuples `t` such that predicate `P(t)` is true."* Here, `t` is a **tuple variable** — it represents an entire row, and `t.attribute` accesses a specific column of that row.

**Formal General Syntax:**
```
{ t | P(t) }
```
where `P` is a formula built from:
- Atoms of the form `t ∈ R` (tuple membership in a named relation)
- Atoms of the form `t.A θ t'.A'` or `t.A θ constant` (comparisons, θ ∈ {=, ≠, <, ≤, >, ≥})
- Logical connectives ∧, ∨, ¬
- Quantifiers ∃, ∀ over other tuple variables

### Worked Example 1 — Simple Selection

**Sample Table `Employees`:**

| emp_id | name | dept | salary |
|---|---|---|---|
| 1 | Aditi | Sales | 55000 |
| 2 | Rohan | IT | 72000 |
| 3 | Kavya | IT | 68000 |

**"Find all employees in the IT department"**
```
{ t | t ∈ Employees ∧ t.dept = 'IT' }
```
**Equivalent RA:** `σ_(dept='IT')(Employees)`
**Equivalent SQL:**
```sql
SELECT * FROM Employees WHERE dept = 'IT';
```

### Worked Example 2 — Projection in TRC (Using a Restricted Tuple)

TRC expresses projection by declaring the output tuple `t` to range over **only some attributes**, then constraining it to match some row in the base relation.

**"Find the names of employees earning more than 60000"**
```
{ t.name | t ∈ Employees ∧ t.salary > 60000 }
```
**Equivalent RA:** `π_name(σ_(salary>60000)(Employees))`
**SQL:**
```sql
SELECT name FROM Employees WHERE salary > 60000;
```

### Worked Example 3 — Existential Quantification (Join-like Logic)

**Sample Table `Departments`:**

| dept_id | dept_name | location |
|---|---|---|
| D1 | Sales | Bengaluru |
| D2 | IT | Pune |

**"Find names of employees who work in a department located in Pune"**
```
{ t.name | t ∈ Employees ∧
           ∃ d ∈ Departments ( d.dept_name = t.dept ∧ d.location = 'Pune' ) }
```
**Reading it aloud:** *"Give me the name of every employee tuple t such that there exists some department tuple d where d's name matches t's department and d's location is Pune."*

**Equivalent SQL:**
```sql
SELECT e.name
FROM Employees e
WHERE EXISTS (
    SELECT 1 FROM Departments d
    WHERE d.dept_name = e.dept AND d.location = 'Pune'
);
```

> 🎯 **Key Insight:** Notice how closely the TRC expression mirrors the SQL `WHERE EXISTS (...)` correlated subquery syntax. This is not a coincidence — **SQL's correlated subqueries are essentially TRC written with SQL keywords.** This connects directly forward to **Module 3: Advanced SQL (Correlated Subqueries)**.

### Worked Example 4 — Universal Quantification (The "Division" Pattern in TRC)

**"Find students enrolled in every course in `RequiredCourses`"** (same data as Module 1, Section 1.9):

```
{ t | t ∈ Students ∧
      ∀ c ∈ RequiredCourses
        ( ∃ e ∈ Enrolled ( e.student = t.name ∧ e.course = c.course ) ) }
```

Since pure TRC often disallows ∀ directly (for safety reasons — see Section 2.6), it's commonly rewritten using the ¬∃¬ identity:
```
{ t | t ∈ Students ∧
      ¬ ∃ c ∈ RequiredCourses
        ( ¬ ∃ e ∈ Enrolled ( e.student = t.name ∧ e.course = c.course ) ) }
```

This is **exactly** the logical skeleton behind the double-`NOT EXISTS` SQL pattern you saw for Division in Module 1 — TRC is the formal logic explaining *why* that SQL pattern works.

---

## 2.5 Domain Relational Calculus (DRC)

**Definition:** DRC is like TRC, but variables range over **individual domain values (column values)** instead of whole tuples. A DRC query has the form:
```
{ <x1, x2, ..., xn> | P(x1, x2, ..., xn) }
```
where each `xi` is a **domain variable** representing a single attribute value, and the result is a set of tuples `<x1,...,xn>` satisfying `P`.

### Worked Example — Same Query, DRC Style

**"Find names of employees earning more than 60000"**

TRC version (recap):
```
{ t.name | t ∈ Employees ∧ t.salary > 60000 }
```

DRC version:
```
{ <n> | ∃ id, d, s ( <id, n, d, s> ∈ Employees ∧ s > 60000 ) }
```

**Reading it aloud:** *"Give me all values `n` such that there exist some `id`, `d`, `s` where the tuple `<id, n, d, s>` is in Employees and `s > 60000`."* Here every single column is named as its own variable (`id` for emp_id, `n` for name, `d` for dept, `s` for salary), and unwanted columns (`id`, `d`, `s`) are existentially quantified away because we don't care about their specific values — we only report `n`.

### DRC Join Example

**"Find employee names and their department's location"**
```
{ <n, loc> | ∃ id, dept, s ( <id, n, dept, s> ∈ Employees ∧
             ∃ dname ( <dept, dname, loc> ∈ Departments ∧ dname = dept ) ) }
```

> **QBE (Query By Example)** — a lesser-known but exam-relevant historical query language — is directly based on Domain Relational Calculus, using a grid/table-based visual notation instead of mathematical symbols. It's occasionally asked about in GATE-style exams as *"which query language is based on DRC?"* Answer: **QBE**.

### TRC vs DRC — Side-by-Side Comparison

| Aspect | TRC | DRC |
|---|---|---|
| Variable ranges over | Whole tuples (rows) | Individual domain values (column values) |
| Notation | `{ t | P(t) }` | `{ <x1,...,xn> | P(x1,...,xn) }` |
| Attribute access | `t.attribute` | Each attribute is its own variable |
| Verbosity | More compact | More verbose (one variable per column) |
| Expressive power | Equivalent to DRC | Equivalent to TRC |
| Real-world example | SQL (`WHERE EXISTS`, correlated subqueries) | QBE (Query By Example) |

**Key takeaway for interviews:** TRC and DRC are **equally expressive** — anything you can write in one, you can rewrite in the other. They differ only in **notation/style**, not in **power**.

---

## 2.6 Safe vs Unsafe Expressions

This is one of the most conceptually important — and most exam-tested — ideas in Relational Calculus.

### The Problem: Infinite Relations

Because RC lets you use **negation (¬)** freely, it's mathematically possible to write a syntactically valid expression whose result is **infinite** or **undefined**, because it implicitly refers to "everything that is NOT in a relation" — and the universe of "everything" is unbounded (e.g., "all possible strings," "all possible numbers").

**Example of an Unsafe Expression:**
```
{ t | ¬ ( t ∈ Employees ) }
```
**Reading it:** *"Give me every tuple `t` that is NOT in Employees."* This is **unsafe** because it would need to enumerate literally every tuple in the (infinite) universe of possible tuples, minus the finite ones in Employees — an infinite, undefined result.

Another subtle unsafe example:
```
{ t | t.salary > 50000 }
```
This has **no relation membership constraint at all** (`t ∈ Employees` is missing) — `t` isn't grounded in any actual finite relation, so it's unsafe (undefined domain to range over).

### The Definition of "Safe"

**An expression is safe if:**
1. Every value that appears in the result comes from the **active domain** of the query (i.e., from actual constants in the query or actual values appearing in the referenced base relations) — not from the theoretical infinite universe of all possible values.
2. Every free variable is properly bound/restricted by a finite relation membership condition (like `t ∈ Employees`), not left dangling.
3. Universal quantifiers, when used, are appropriately restricted to a finite domain (typically converted internally to the safe ¬∃¬ form scoped to actual relations, as shown in Section 2.4's Division example).

### Comparison Table

| | Safe Expression | Unsafe Expression |
|---|---|---|
| Result | Always finite, well-defined | Can be infinite/undefined |
| Example | `{ t | t ∈ Employees ∧ t.salary > 50000 }` | `{ t | ¬(t ∈ Employees) }` |
| Can a real DBMS execute it? | ✅ Yes | ❌ No — rejected before execution |
| Practical relevance | This is what SQL always produces | SQL syntax structurally prevents you from writing this (you cannot say "NOT IN literally every possible row in the universe") |

> 🎯 **Why SQL engineers rarely worry about this in practice:** SQL's syntax is **structurally designed so that unsafe expressions cannot even be written** — every `SELECT` must specify a `FROM` (a finite, real, named relation), so you can never "select from the infinite universe of all tuples." **This safety guarantee is baked into SQL's grammar**, which is precisely why relational calculus theory (safety) matters for language designers even though working engineers never hit this issue directly.

---

## 2.7 Algebra vs Calculus — Full Comparison

### Codd's Theorem (High-Value Theory Fact)

**Codd's Theorem** states that **Relational Algebra and (safe) Relational Calculus are equivalent in expressive power** — anything expressible in one is expressible in the other. This equivalence is what makes SQL (calculus-flavored syntax) implementable via an algebra-based execution engine, without any loss of expressiveness. A language is called **"relationally complete"** if it is at least as powerful as Relational Algebra/(safe) Calculus.

```
                    Codd's Theorem (1972)
    ┌─────────────────┐         equivalent          ┌─────────────────┐
    │ Relational       │ ◀────────power───────────▶ │ Relational       │
    │ Algebra (RA)     │                              │ Calculus (RC)    │
    │ "HOW" - procedural│                             │ "WHAT" - declarative│
    └─────────────────┘                              └─────────────────┘
            │                                                  │
            └──────────────────────┬───────────────────────────┘
                                    ▼
                        Both are the theoretical basis for
                                    SQL
                (syntax ≈ Calculus, execution ≈ Algebra)
```

### Detailed Comparison Table

| Aspect | Relational Algebra | Relational Calculus |
|---|---|---|
| **Nature** | Procedural (HOW) | Declarative (WHAT) |
| **Specifies** | Sequence of operations | Properties/conditions the result must satisfy |
| **Core building blocks** | Operators (σ, π, ⋈, ∪, etc.) | Logical formulas (∧, ∨, ¬, ∃, ∀) |
| **Execution order** | Explicit (tree structure dictates order) | Implicit / not specified by the user |
| **Used directly by** | Query optimizers & execution engines internally | Query language designers (as SQL's semantic basis) |
| **Ease of use for end users** | Requires thinking in steps | Matches natural "what I want" thinking |
| **Expressive power** | Equivalent (Codd's Theorem) | Equivalent (Codd's Theorem) |
| **Real-world direct usage** | Practically never typed by hand; used internally by optimizer | Practically never typed by hand either; but SQL syntax mirrors TRC closely |
| **Two variants?** | No (one unified algebra) | Yes — TRC and DRC (equally powerful, different notation) |
| **Safety concern?** | Not applicable (RA is inherently safe/finite by construction) | Yes — must explicitly avoid unsafe/infinite expressions |

---

## 2.8 Relationship to SQL

```
┌─────────────────────────────────────────────────────────────────┐
│                      SQL (What You Actually Write)                │
│                                                                    │
│  Surface syntax "feels like" Tuple Relational Calculus:            │
│    SELECT <free variable / output columns>                        │
│    FROM   <relation the tuple variable ranges over>                │
│    WHERE  <predicate P(t), can include EXISTS/NOT EXISTS = ∃/¬∃>   │
│                                                                    │
│  Internally compiled into a Relational Algebra expression tree     │
│  (π, σ, ⋈, ∪, ...) for the optimizer & execution engine.            │
└─────────────────────────────────────────────────────────────────┘
```

| SQL Feature | Calculus Concept |
|---|---|
| `SELECT col FROM t WHERE cond` | `{ t.col | t ∈ Table ∧ cond }` (TRC) |
| `WHERE EXISTS (subquery)` | ∃ (existential quantifier) |
| `WHERE NOT EXISTS (subquery)` | ¬∃ (negated existential) |
| `NOT EXISTS ( ... NOT EXISTS ... )` (double negation) | ∀ via ¬∃¬ identity (universal quantification) |
| Every `SELECT` requiring a `FROM` | Enforces **safety** — you cannot query the infinite universe |
| Correlated subquery (`WHERE t.x = outer.y`) | Free variable reference across nested predicates |

---

## 2.9 Common Mistakes & Edge Cases

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Writing a "for all" query directly with ∀ in an interview without knowing the ¬∃¬ rewrite | Most practical systems/derivations expect the safe rewritten form | Always know how to convert `∀ x (P(x))` to `¬∃ x (¬P(x))` |
| Forgetting relation membership (`t ∈ R`) in a TRC formula | Produces an unsafe/undefined expression | Every free tuple variable must be grounded in an actual relation |
| Confusing TRC's `t.attribute` access with DRC's flat variable-per-column style | Leads to malformed hybrid expressions | Keep the two notations separate; know which one a question is asking for |
| Assuming Calculus is "weaker" than Algebra because it "doesn't specify order" | Codd's Theorem proves them equal in *power*, only different in *style* | Learn Codd's Theorem statement precisely for exams |
| Treating `NOT IN` and `¬∃` as always interchangeable | `NOT IN` breaks with `NULL`s (Module 1, Section 1.8.6) even though logically ¬∃ should be NULL-safe when translated correctly | Prefer `NOT EXISTS` in SQL, which correctly implements ¬∃ semantics |

---

## 2.10 Interview Questions & MCQs

### Conceptual
1. State Codd's Theorem in your own words. Why does it matter for database language design?
2. Explain the difference between Tuple Relational Calculus and Domain Relational Calculus with an example.
3. What makes an expression "unsafe" in Relational Calculus? Give an example.
4. Convert `∀ x ∈ R (P(x))` into its safe, ¬∃¬-based equivalent, and explain why this conversion is necessary.
5. Why is SQL considered closer to Tuple Relational Calculus in surface syntax, but executed via Relational Algebra internally?

### Applied
6. Write a TRC expression for: "Find the names of customers who have placed an order for every product in the 'Electronics' category." Then convert it to SQL.
7. Given `Employees` and `Departments`, write a DRC expression to find employee names along with their department's location, without using TRC-style tuple notation (i.e., use only individual domain variables).
8. Explain, using ∃, why `WHERE EXISTS (subquery)` in SQL is a direct implementation of Tuple Relational Calculus's existential quantifier.

### MCQs

**Q1.** Relational Calculus is best described as:
 a) Procedural  b) Declarative  c) Object-oriented  d) Imperative

**Q2.** Which quantifier does `NOT EXISTS (SELECT ... WHERE NOT ...)` (double negation) typically represent?
 a) ∃  b) ¬∃  c) ∀  d) ∧

**Q3.** According to Codd's Theorem:
 a) RA is strictly more powerful than RC
 b) RC is strictly more powerful than RA
 c) RA and safe RC are equally expressive
 d) Neither is comparable

**Q4.** Which real query language is historically based on Domain Relational Calculus?
 a) SQL  b) QBE (Query By Example)  c) Datalog  d) XQuery

**Q5.** An expression like `{ t | ¬(t ∈ Employees) }` is:
 a) Safe  b) Unsafe  c) Equivalent to a Selection  d) A valid SQL query

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) Declarative**
2. **c) ∀** — the double-negation `NOT EXISTS(... NOT ...)` pattern implements universal quantification via the ¬∃¬ identity.
3. **c) RA and safe RC are equally expressive** — this is precisely Codd's Theorem.
4. **b) QBE (Query By Example)**
5. **b) Unsafe** — it implicitly references the infinite universe of all tuples not in Employees.

</details>

---

## 2.11 Cheat Sheet — Module 2

```
┌───────────────────────────────────────────────────────────────────┐
│                RELATIONAL CALCULUS — QUICK REFERENCE                │
├───────────────────────────┬───────────────────────────────────────┤
│ { t | P(t) }              │ TRC — t ranges over whole tuples        │
│ { <x1,...,xn> | P(...) }  │ DRC — each xi ranges over one domain    │
│ ∃ x (P(x))                │ "there exists an x such that P(x)"      │
│ ∀ x (P(x))                │ "for all x, P(x) holds"                 │
│ ∀ x (P(x)) ≡ ¬∃x(¬P(x))   │ Universal → safe existential rewrite    │
│ Safe expression           │ Result is finite, grounded in real data │
│ Unsafe expression         │ Result could be infinite/undefined      │
│ Codd's Theorem            │ RA ≡ safe RC in expressive power        │
│ SQL EXISTS                │ = ∃                                     │
│ SQL NOT EXISTS             │ = ¬∃                                    │
│ SQL double NOT EXISTS      │ = ∀ (via ¬∃¬)                          │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 2.12 Exercises

### Beginner
1. Write a TRC expression for "find all employees with salary greater than 40000."
2. Convert your TRC expression from Q1 into equivalent SQL.

### Intermediate
3. Write a TRC expression using ∃ to find "employees who belong to a department located in Bengaluru," given `Employees(name, dept)` and `Departments(dept_name, location)`.
4. Rewrite the following into its safe ¬∃¬ form: `∀ c ∈ Courses (student has taken c)`.

### Advanced
5. Write both a TRC and a DRC expression for: "Find product names that have been ordered by every customer." Then write the corresponding SQL using the `COUNT`/`HAVING` pattern from Module 1.
6. Explain, using Codd's Theorem, why it is *theoretically guaranteed* that any query you can express in SQL's `WHERE EXISTS` correlated-subquery style could also be rewritten as a pure join+filter (Algebra-style) query — and why an optimizer relies on this guarantee.

### Output Prediction
7. Is this expression safe or unsafe? Justify your answer: `{ t | t ∈ Employees ∨ t.salary > 100000 }` (note the `∨`, not `∧`).

### Debugging
8. A student writes: `{ t | t.salary > 50000 }` intending to find high earners, but their professor marks it wrong. What's missing, and why does it matter?

### Scenario-Based
9. In an interview, you're asked: *"Without writing SQL, describe in predicate-logic form how you'd find customers who have never placed an order."* Give your answer using ¬∃.

### Practice Links
- [GATE Overflow — Relational Calculus PYQs](https://gateoverflow.in/)
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

- Relational Calculus is a **declarative** language: you describe **what** you want via **predicate logic**, not **how** to compute it.
- **TRC** variables range over tuples (`t.attribute`); **DRC** variables range over individual domain values — they are **equally expressive**, differing only in notation.
- **Safety** is essential: an expression is safe only if its result is guaranteed finite and grounded in real, named relations — SQL's grammar structurally enforces this by requiring `FROM`.
- **Universal quantification (∀)** is always rewritten as **¬∃¬** for safe evaluation — this is the exact logical basis of the "Division" operator (Module 1) and the double-`NOT EXISTS` SQL idiom.
- **Codd's Theorem**: Relational Algebra and safe Relational Calculus have **equal expressive power** — this equivalence is why SQL can *look* declarative (calculus-like) while *executing* procedurally (algebra-like).
- SQL's `EXISTS`/`NOT EXISTS` are the direct real-world implementations of ∃ and ¬∃.

---

## Cross-References for Later Parts / Modules
- **Correlated Subqueries, EXISTS/NOT EXISTS in depth** → Module 3 (this Part)
- **Division operator & double-NOT EXISTS pattern (procedural view)** → Module 1, Section 1.9 (this Part)
- **Query optimizer's algebra-based rewriting of calculus-style SQL** → Part 8 (Query Execution Internals)
- **Functional Dependencies / Normalization** → Part 5

---

> ✅ **Module 2 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 3"** to proceed to **Module 3: Advanced SQL Queries** (Nested Queries, Correlated Queries, EXISTS/NOT EXISTS, ANY/SOME/ALL, Derived Tables, LATERAL Joins, Recursive Queries).
