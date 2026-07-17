# DBMS Handbook — Part 2
## Module 5: Integrity Constraints

> **Builds on:** Module 4 (Keys — Primary Key, Foreign Key, Unique Key are all reused here as enforcement mechanisms).
>
> **Feeds into:** Module 6 (ER Model, where business rules are first captured conceptually before becoming constraints), and the Normalization module of Part 3.

---

## Table of Contents

1. [What Are Integrity Constraints, and Why Do They Exist?](#1-what-are-integrity-constraints-and-why-do-they-exist)
2. [Classification of Constraints](#2-classification-of-constraints)
3. [Domain Constraint](#3-domain-constraint)
4. [Key Constraint](#4-key-constraint)
5. [Entity Integrity](#5-entity-integrity)
6. [Referential Integrity](#6-referential-integrity)
7. [NOT NULL Constraint](#7-not-null-constraint)
8. [UNIQUE Constraint](#8-unique-constraint)
9. [PRIMARY KEY Constraint](#9-primary-key-constraint)
10. [FOREIGN KEY Constraint](#10-foreign-key-constraint)
11. [CHECK Constraint](#11-check-constraint)
12. [DEFAULT Constraint](#12-default-constraint)
13. [Assertions](#13-assertions)
14. [Business Rules](#14-business-rules)
15. [Internal Enforcement — How the DBMS Actually Checks Constraints](#15-internal-enforcement--how-the-dbms-actually-checks-constraints)
16. [Constraint Timing: Immediate vs. Deferred](#16-constraint-timing-immediate-vs-deferred)
17. [Chapter Summary](#17-chapter-summary)
18. [Cheat Sheet](#18-cheat-sheet)
19. [Exercises](#19-exercises)

---

## 1. What Are Integrity Constraints, and Why Do They Exist?

### Definition

**Integrity constraints** are **rules that the DBMS automatically enforces** to ensure the database always remains in a **valid state** (recall the "database state" concept from Module 2, §15) — protecting data from becoming inconsistent, inaccurate, or logically impossible.

### Why They Exist

Without constraints, nothing stops an application bug (or a careless manual `INSERT`) from putting garbage into the database: a negative account balance, an employee with no name, an order referencing a customer that doesn't exist. Constraints exist to make the DBMS itself the **last line of defense** for data correctness — independent of, and more reliable than, any single application's code.

### Core Principle

> **Constraints should be enforced as close to the data as possible** — i.e., in the database itself — not only in application code. Application-level validation can be bypassed (bugs, multiple applications sharing one DB, direct DB access, batch jobs) but a DBMS-level constraint is checked on **every single write, from every source, always**.

### Real-World Analogy

Integrity constraints are like a building's structural safety codes: a construction company (application code) *should* follow safe practices voluntarily, but a building inspector (the DBMS) provides an independent, non-negotiable check that catches violations regardless of who's building, or how careful they claim to be.

### Interview Importance

**Very High.** "Where should validation logic live — application or database?" is a very common system-design discussion, and the DBMS-level answer (constraints) is the theoretically correct baseline, even though real systems often use both layers for different reasons (performance, user-friendly error messages, etc.).

---

## 2. Classification of Constraints

Integrity constraints are traditionally classified into these categories, which this module covers in order:

```
                    Integrity Constraints
                            │
      ┌──────────┬──────────┼───────────┬─────────────┐
      ▼          ▼          ▼           ▼             ▼
   Domain      Key       Entity     Referential   Semantic/
 Constraint  Constraint  Integrity  Integrity    Business Rules
      │          │          │           │             │
  (legal      (uniqueness (PK can't  (FK must match  (custom, app-
   values      of keys)    be NULL)   existing PK/    specific logic,
   only)                              be NULL)        e.g., CHECK)
```

Additionally, SQL provides concrete **syntactic mechanisms** to enforce these conceptual constraint categories: `NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`, `CHECK`, `DEFAULT`, and (conceptually) `ASSERTIONS`. Sections §7 onward map each conceptual category to its concrete SQL tool.

---

## 3. Domain Constraint

### Definition

A **domain constraint** restricts the values of an attribute to its declared **domain** (recall Module 2, §9) — i.e., its data type and any additional business-defined valid-value restrictions.

### Why It Exists

It's the most basic level of data correctness: *"Is this even the right kind/range of value at all?"* — e.g., an `age` column should never contain the text `"banana"`, and shouldn't be `-5` either.

### How It's Enforced

- **Basic type enforcement**: `INT`, `VARCHAR`, `DATE`, etc. (the DBMS engine rejects type-mismatched inserts outright).
- **Finer-grained enforcement**: `CHECK` constraints (§11) or `ENUM` types for restricting to a specific valid subset beyond the raw data type.

```sql
CREATE TABLE Student (
    age INT CHECK (age BETWEEN 0 AND 120)   -- domain constraint beyond base INT type
);
```

### Common Mistakes

- Relying only on the raw data type (`INT` for age) without a `CHECK`, allowing nonsensical values like `age = -5` or `age = 999`.

### Interview Importance

**Medium-High** — usually tested together with `CHECK` constraints (§11) rather than in isolation.

---

## 4. Key Constraint

### Definition

The **key constraint** states that the **values of a key attribute (or attribute set) must be unique** across all tuples in a relation — this is the formal, conceptual rule that underlies the practical `PRIMARY KEY` and `UNIQUE` mechanisms (§9, §8) covered later in this module.

### Why It Exists

This is the formal integrity-constraint expression of everything covered in **Module 4** — a key constraint is what makes a "key" a key at all: without an *enforced* uniqueness rule, `roll_no` would just be a regular attribute that *happens* to often contain distinct values, with nothing stopping a future duplicate.

### Interview Importance

**High**, mostly as the conceptual bridge between Module 4 (Keys) and this module's concrete SQL enforcement mechanisms.

---

## 5. Entity Integrity

### Definition

**Entity Integrity** is the rule that **no attribute participating in the primary key of a base relation may be NULL**.

### Why It Exists

Recall from Module 4, §5 that a primary key's job is to uniquely identify every real-world entity represented by a tuple. A NULL primary key would mean *"this row represents... some entity, but we don't know which one"* — which is a logical contradiction: **every entity must be identifiable**, or it isn't meaningfully represented in the database at all.

### Example

```sql
CREATE TABLE Student (
    roll_no INT PRIMARY KEY,   -- entity integrity: roll_no can NEVER be NULL
    name    VARCHAR(50)
);

INSERT INTO Student (roll_no, name) VALUES (NULL, 'Aditi');  -- ❌ REJECTED by the DBMS
```

### Entity Integrity and Composite Primary Keys

For a **composite primary key** (Module 4, §7), entity integrity requires that **every single component attribute** be non-NULL — a partially-NULL composite key still violates entity integrity.

```sql
PRIMARY KEY (roll_no, course_id)
-- roll_no = 101, course_id = NULL   →  ❌ violates entity integrity (even though roll_no is provided)
```

### Interview Importance

**High.** "Why can't a primary key be NULL?" is a very standard conceptual interview question, and the correct, precise answer invokes exactly this "entity must be identifiable" reasoning.

---

## 6. Referential Integrity

### Definition

**Referential Integrity** is the rule that a **foreign key value must either be NULL, or must exactly match an existing value of the primary/unique key it references** in the target table. It ensures relationships between tables remain logically consistent — no "dangling references" to non-existent rows.

### Why It Exists

Without referential integrity, you could have an `Enrollment` row pointing to `roll_no = 999`, even though no student with that roll number exists — a logically broken, meaningless relationship (an "orphan" record).

### Example

```sql
CREATE TABLE Student (
    roll_no INT PRIMARY KEY,
    name    VARCHAR(50)
);

CREATE TABLE Enrollment (
    roll_no    INT,
    course_id  INT,
    FOREIGN KEY (roll_no) REFERENCES Student(roll_no)
);

INSERT INTO Enrollment (roll_no, course_id) VALUES (999, 'CS101');
-- ❌ REJECTED: no Student with roll_no = 999 exists (referential integrity violation)
```

### Referential Integrity Across Operations

| Operation | Referential Integrity Check |
|---|---|
| `INSERT` into child table | New foreign key value must already exist in the parent table (or be NULL) |
| `UPDATE` child's foreign key | New value must exist in the parent table (or be NULL) |
| `DELETE` from parent table | Blocked (by default `RESTRICT`), or triggers `CASCADE`/`SET NULL`, per the FK's configured action (Module 4, §9) |
| `UPDATE` parent's key value | Same as delete — blocked unless `ON UPDATE CASCADE` is configured |

### Referential Integrity Diagram

```
Student (Parent)                    Enrollment (Child)
+---------+--------+                +---------+------------+
| roll_no | name   |                | roll_no | course_id  |
+---------+--------+                +---------+------------+
| 101     | Aditi  | ◄───────────── | 101     | CS101      |  ✅ valid reference
| 102     | Rohan  | ◄───────────── | 102     | CS102      |  ✅ valid reference
+---------+--------+                | 999     | CS103      |  ❌ INVALID — no such roll_no
                                     +---------+------------+     (referential integrity violated)
```

### Interview Importance

**Extremely High.** Referential integrity, cascade behavior, and orphan-record prevention are among the most consistently tested DBMS concepts across all levels.

---

## 7. NOT NULL Constraint

### Definition

`NOT NULL` is the SQL syntax used to enforce that a column **must always have a value** — it can never be left as NULL.

### Example

```sql
CREATE TABLE Employee (
    emp_id  INT PRIMARY KEY,       -- implicitly NOT NULL (entity integrity, §5)
    name    VARCHAR(50) NOT NULL,  -- explicitly required
    manager_id INT                 -- nullable — a new CEO has no manager (Module 2, §17)
);
```

### Relationship to Entity Integrity

`NOT NULL` on a primary key column is how **Entity Integrity (§5)** gets concretely enforced in SQL — though most databases apply this automatically the moment you declare a column `PRIMARY KEY`, without needing to write `NOT NULL` explicitly.

### Best Practices

- Apply `NOT NULL` liberally to any attribute the business genuinely always requires — don't leave columns nullable "just in case," since this weakens data quality guarantees and complicates every downstream query (Module 2, §17's three-valued logic issues).

### Interview Importance

**High** — foundational, frequently tested alongside NULL-handling questions (Module 2, §17).

---

## 8. UNIQUE Constraint

### Definition

`UNIQUE` is the SQL syntax used to enforce that **all non-NULL values in a column (or column combination) must be distinct** across the table — the concrete implementation of an Alternate Key or general Unique Key (Module 4, §6, §13).

### Example

```sql
CREATE TABLE Student (
    roll_no INT PRIMARY KEY,
    email   VARCHAR(100) UNIQUE   -- no two students can share an email
);
```

### Composite UNIQUE Constraints

```sql
CREATE TABLE Enrollment (
    roll_no    INT,
    course_id  INT,
    UNIQUE (roll_no, course_id)   -- a student can't enroll in the same course twice
);
```

### Interview Importance

**High** — often directly compared against `PRIMARY KEY` (see Module 4, §13's comparison table).

---

## 9. PRIMARY KEY Constraint

### Definition

`PRIMARY KEY` is the SQL syntax that **simultaneously enforces**:
1. **Uniqueness** (like `UNIQUE`)
2. **Non-nullability** (Entity Integrity, §5 / `NOT NULL`, §7)
3. Marks the column(s) as the table's single official identifier

```sql
CREATE TABLE Student (
    roll_no INT PRIMARY KEY   -- equivalent to: roll_no INT NOT NULL UNIQUE, PLUS "this is THE identifier"
);
```

### Interview Importance

**Extremely High** — see the extensive treatment already given in Module 4, §5; this section exists to explicitly connect it into the constraint taxonomy.

---

## 10. FOREIGN KEY Constraint

### Definition

`FOREIGN KEY` is the SQL syntax that enforces **Referential Integrity (§6)** — linking a column in one table to the primary/unique key of another (or the same) table.

*(Full syntax, cascade options, and detailed examples already covered in Module 4, §9 — this section exists to explicitly connect it into the constraint taxonomy alongside its conceptual counterpart, §6.)*

### Interview Importance

**Extremely High** — see Module 4, §9.

---

## 11. CHECK Constraint

### Definition

`CHECK` is the SQL syntax used to enforce **arbitrary boolean conditions** on column values — the primary concrete tool for implementing fine-grained **Domain Constraints (§3)** and simple **Business Rules (§14)** directly in the schema.

### Examples

```sql
CREATE TABLE Employee (
    emp_id     INT PRIMARY KEY,
    salary     DECIMAL(10,2) CHECK (salary > 0),
    age        INT CHECK (age BETWEEN 18 AND 65),
    email      VARCHAR(100) CHECK (email LIKE '%@%.%')
);

CREATE TABLE Orders (
    order_id     INT PRIMARY KEY,
    order_date   DATE,
    ship_date    DATE,
    CHECK (ship_date >= order_date)   -- multi-column CHECK constraint
);
```

### Advantages

- Enforces business-critical rules directly at the database level, guaranteeing correctness regardless of which application writes the data.
- Multi-column `CHECK` constraints can express relationships between columns within the same row (e.g., `ship_date >= order_date`).

### Disadvantages / Limitations

- `CHECK` constraints **cannot reference other tables** — they can only look at columns within the same row of the same table. Cross-table business rules require **Assertions (§13)** (rarely supported) or application-level/trigger-based enforcement.
- Historically, some older RDBMS versions had limited or buggy `CHECK` constraint support (worth verifying per-system, though modern PostgreSQL, MySQL 8+, SQL Server, and Oracle all support them robustly).

### Common Mistakes

- Assuming `CHECK` can be used for rules spanning multiple tables (e.g., "total order amount must not exceed customer's credit limit") — it cannot; this requires triggers or application logic.

### Interview Importance

**High.** `CHECK` constraints are commonly tested in schema design questions requiring business rule enforcement (e.g., "ensure discount percentage is between 0 and 100").

---

## 12. DEFAULT Constraint

### Definition

`DEFAULT` specifies a **fallback value automatically inserted** into a column when no explicit value is provided during an `INSERT`.

### Example

```sql
CREATE TABLE Orders (
    order_id    INT PRIMARY KEY,
    status      VARCHAR(20) DEFAULT 'PENDING',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO Orders (order_id) VALUES (1);
-- Resulting row: (1, 'PENDING', <current timestamp>)
```

### Why It Exists

Reduces application-side boilerplate (no need for every application/insert statement to remember to set `status = 'PENDING'`) and guarantees consistent defaults regardless of the source of the insert.

### DEFAULT vs. NOT NULL — Important Distinction

`DEFAULT` and `NOT NULL` solve **different problems** and are often combined:

| | Purpose |
|---|---|
| `NOT NULL` alone | Forces the caller to explicitly provide a value — omission causes an error |
| `DEFAULT` alone | Silently provides a fallback value if none is given — omission does NOT cause an error |
| `NOT NULL DEFAULT 'x'` | Caller *may* omit the value (DEFAULT kicks in), but can never explicitly insert NULL |

### Interview Importance

**Medium** — straightforward, occasionally tested in schema design and rarely a major standalone topic.

---

## 13. Assertions (Concept)

### Definition

An **assertion** is a **general, standalone integrity constraint** that can express complex business rules — potentially spanning **multiple tables** — independent of any single table's `CHECK` constraint.

### Example (Conceptual SQL Standard Syntax)

```sql
CREATE ASSERTION total_enrollment_limit
CHECK (
    (SELECT COUNT(*) FROM Enrollment WHERE course_id = 'CS101') <= 60
);
```

This says: "At all times, across the entire database, no more than 60 students may be enrolled in CS101" — a rule that inherently spans the `Enrollment` table's aggregate state, something a single-row `CHECK` constraint cannot express.

### Why This Section Says "(Concept)"

> ⚠️ **Critical, frequently-tested nuance:** `CREATE ASSERTION` is part of the **SQL standard**, but is **NOT actually implemented** by most major production RDBMSs (including PostgreSQL, MySQL, and SQL Server, as of current mainstream versions) due to the significant performance cost of continuously re-checking complex, potentially cross-table/aggregate conditions on every relevant write. In practice, the *effect* of an assertion is achieved instead through:
> - **Triggers** (procedural code that runs on INSERT/UPDATE/DELETE, covered in a later Part)
> - **Application-level validation logic**
> - Careful transaction design

### Interview Importance

**Medium-High for conceptual understanding, but candidates should NOT claim `CREATE ASSERTION` is commonly used in production** — a nuanced, accurate answer here (knowing it's standard-but-rarely-implemented) is a strong signal of genuine depth versus rote textbook memorization.

---

## 14. Business Rules

### Definition

**Business rules** (also called **semantic integrity constraints**) are constraints derived from the **real-world policies and logic of the organization** the database serves — as opposed to the more generic, structural constraints (domain, key, entity, referential integrity) that apply to *any* well-formed relational database regardless of business context.

### Examples

- "An employee's salary cannot exceed their manager's salary." (cross-row, same table — could use a trigger)
- "A customer cannot place an order exceeding their credit limit." (cross-table — needs a trigger, application logic, or an assertion in theory)
- "A book cannot be marked as 'returned' if it was never marked as 'borrowed'." (state-machine style rule)

### Where Business Rules Get Enforced in Practice

```
                     Business Rule
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   Simple, single-   Cross-table/     Complex, workflow-
   row rule?         aggregate rule?  dependent rule?
          │               │               │
          ▼               ▼               ▼
   CHECK constraint    Trigger /       Application-layer
   (§11)               Assertion       business logic
                        (rare in       (most common
                        practice)      approach)
```

### Best Practices

- Push as much validation as reasonably possible into the database (`CHECK`, `NOT NULL`, `FOREIGN KEY`) — but accept that some genuinely complex, multi-step, or workflow-dependent business rules are more practically enforced in application code or triggers, and that's a legitimate, common engineering trade-off, not a failure of database design.

### Interview Importance

**High**, especially in system design interviews discussing where validation logic should live (see §1's "core principle").

---

## 15. Internal Enforcement — How the DBMS Actually Checks Constraints

### The General Mechanism

Every time a `DML` (Data Manipulation Language) statement — `INSERT`, `UPDATE`, `DELETE` — is executed, the DBMS performs constraint checks **as part of the same operation**, before the change is considered "committed":

```
 Application sends: INSERT INTO Student VALUES (101, 'Aditi', NULL)
                              │
                              ▼
              ┌───────────────────────────────┐
              │  DBMS Constraint Checking      │
              │  1. Domain check (types match?)│
              │  2. NOT NULL check              │
              │  3. UNIQUE / PRIMARY KEY check  │
              │  4. FOREIGN KEY check           │
              │  5. CHECK constraint evaluation │
              └───────────────────────────────┘
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
           All pass → COMMIT         Any fail → ROLLBACK
           (row is written)          (entire statement rejected,
                                       error returned to caller)
```

### Indexing and Constraint Enforcement

Most constraint checks (`UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`) are made efficient by the DBMS automatically building an **index** (typically a B+ Tree — covered in depth in the Storage & Indexing module of a later Part) on the constrained column(s). Without an index, checking uniqueness or a foreign key reference would require an expensive full-table scan on every single write.

> ⚠️ **Important, frequently-tested detail:** Most RDBMSs automatically index `PRIMARY KEY` and `UNIQUE` columns, but **do NOT automatically index `FOREIGN KEY` columns** (already flagged as a common mistake in Module 4, §9) — a DBA/engineer typically must add this index manually for good join/delete-cascade performance.

### Interview Importance

**High for Senior/Staff roles** — understanding *how* constraints are enforced (not just *that* they are) demonstrates systems-level depth beyond surface SQL syntax knowledge.

---

## 16. Constraint Timing: Immediate vs. Deferred

### The Problem

Sometimes, enforcing a constraint **immediately** after every single statement within a multi-step transaction is too strict. Consider swapping two employees' `emp_id` values (each protected by a `UNIQUE` constraint) — at some intermediate point *within the transaction*, there would temporarily be a duplicate, even though the *final* state is perfectly valid.

### Two Modes

| Mode | Behavior |
|---|---|
| **Immediate (default in most systems)** | Constraint is checked right after each individual statement |
| **Deferred** | Constraint checking is postponed until the **entire transaction commits** — allowing temporary, intermediate "violations" as long as the final state is valid |

```sql
-- Standard SQL syntax (support varies by RDBMS — e.g., well-supported in PostgreSQL/Oracle)
SET CONSTRAINTS ALL DEFERRED;
```

### Why This Connects Forward

This section is a direct preview of the **ACID Consistency property** and **Transactions module** (a later Part) — constraint checking timing is fundamentally a transaction-management concern, showing how integrity constraints and transaction theory are deeply intertwined.

### Interview Importance

**Medium for general roles, higher for roles heavily involving transaction design.**

---

## 17. Chapter Summary

- **Integrity constraints** are DBMS-enforced rules that keep the database in a valid state at all times — a more reliable, universal enforcement layer than relying solely on application code.
- The conceptual categories are: **Domain Constraint** (legal value ranges), **Key Constraint** (uniqueness of keys), **Entity Integrity** (primary keys can never be NULL), **Referential Integrity** (foreign keys must match an existing parent row, or be NULL), and **Business Rules/Semantic constraints** (organization-specific policies).
- These map to concrete SQL mechanisms: `NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`, `CHECK`, and `DEFAULT` — each covered with syntax and real trade-offs.
- `CHECK` constraints handle single-row, single-table business logic; genuinely cross-table/aggregate rules theoretically belong to `ASSERTIONS`, but assertions are **rarely implemented** in real production RDBMSs — triggers or application logic fill that gap in practice.
- Constraints are enforced internally via checks performed on every `INSERT`/`UPDATE`/`DELETE`, typically accelerated by automatic indexing (though foreign keys often need **manual** indexing).
- Constraint checking can be **immediate** (default) or **deferred** until transaction commit — connecting this module directly to ACID transaction theory (a later Part).

---

## 18. Cheat Sheet

```
CONCEPTUAL CONSTRAINT           →   SQL MECHANISM
─────────────────────────────────────────────────────
Domain Constraint                →  Data type + CHECK
Key Constraint                   →  UNIQUE / PRIMARY KEY
Entity Integrity                 →  PRIMARY KEY (implies NOT NULL)
Referential Integrity            →  FOREIGN KEY
Arbitrary row-level rule         →  CHECK
Fallback value                   →  DEFAULT
Cross-table rule (rare, standard)→  ASSERTION (mostly NOT implemented in practice!)
Cross-table rule (practical)     →  TRIGGER or application logic

KEY FACTS TO NEVER FORGET:
  ✓ Primary Key = UNIQUE + NOT NULL, always, automatically
  ✓ Foreign Key CAN be NULL (means "not yet linked")
  ✓ CHECK constraints CANNOT reference other tables
  ✓ FOREIGN KEY columns are NOT auto-indexed by most RDBMSs (add manually!)
  ✓ ASSERTIONS are standard SQL but rarely implemented in real DBMSs
  ✓ Constraint checks can be IMMEDIATE (default) or DEFERRED (until COMMIT)
```

---

## 19. Exercises

### Beginner

1. Define entity integrity and referential integrity in one sentence each.
2. True/False: A CHECK constraint can reference a column from a different table.
3. Fill in the blank: The __________ constraint ensures a foreign key either matches an existing parent row or is NULL.

### Intermediate

4. Write a `CHECK` constraint ensuring a `discount_pct` column is between 0 and 100.
5. Explain why `FOREIGN KEY` columns often need to be manually indexed, even though `PRIMARY KEY` columns are indexed automatically.
6. A table has `PRIMARY KEY (roll_no, course_id)`. Would a row with `roll_no = 101, course_id = NULL` be accepted? Justify using entity integrity.

### Advanced / Conceptual

7. Explain, using a concrete example, why "no course can have more than 60 enrolled students" cannot be enforced with a simple `CHECK` constraint, and describe two realistic ways to enforce it in a real production system.
8. Discuss: should validation primarily live in the application layer or the database layer? Give one argument for each side, and state which side database-level integrity constraints support most strongly.
9. Explain the difference between immediate and deferred constraint checking, and construct a scenario (like the emp_id-swap example) where deferred checking is necessary.

### MCQs

**Q1.** Which constraint guarantees a primary key is never NULL?
a) Referential Integrity  b) Entity Integrity  c) Domain Constraint  d) DEFAULT

**Q2.** A CHECK constraint can:
a) Reference other tables freely  b) Only reference columns in the same row  c) Never involve more than one column  d) Replace a FOREIGN KEY

**Q3.** Which SQL constraint type is part of the SQL standard but rarely implemented in major production RDBMSs?
a) CHECK  b) DEFAULT  c) ASSERTION  d) UNIQUE

**Q4.** A foreign key column, by default, is:
a) Always NOT NULL  b) Allowed to be NULL, meaning no relationship is currently assigned  c) Always auto-indexed  d) Required to be a composite key

*(Answer key: 1-b, 2-b, 3-c, 4-b)*

### Schema Design Exercise

10. Design a `BankAccount` table incorporating at least: a PRIMARY KEY, a CHECK constraint (balance >= 0), a DEFAULT (account status = 'ACTIVE'), a NOT NULL column, and a FOREIGN KEY to a `Branch` table with an appropriate `ON DELETE` action. Justify your choice of `ON DELETE` behavior.

### Practice Links

- [LeetCode Database — problems requiring understanding of NULL/constraint behavior](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL — Advanced Join and Constraint-related problems](https://www.hackerrank.com/domains/sql)
- [SQLBolt — Lesson on constraints and table creation](https://sqlbolt.com/)

---

**Next: Module 6 — The Entity Relationship (ER) Model (Why ER Modeling, History, Purpose, Conceptual Modeling, ER Components, Notation, Strong/Weak/Associative Entities).**

> Say "continue" or "next module" to proceed to Module 6.
