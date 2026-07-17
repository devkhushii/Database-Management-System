# DBMS Handbook — Part 2
## Module 10: Cardinality Constraints

> **Builds on:** Module 9 (Relationships — especially Binary Relationships, §6, which is the primary context for cardinality).
>
> **Feeds into:** Module 11 (Participation Constraints — cardinality and participation are always specified together in a complete ER diagram), Module 13 (ER-to-Relational Mapping — cardinality directly determines WHERE the foreign key goes).

---

## Table of Contents

1. [What Is a Cardinality Constraint?](#1-what-is-a-cardinality-constraint)
2. [One-to-One (1:1)](#2-one-to-one-11)
3. [One-to-Many (1:N)](#3-one-to-many-1n)
4. [Many-to-One (N:1)](#4-many-to-one-n1)
5. [Many-to-Many (M:N)](#5-many-to-many-mn)
6. [The Golden Rule: Where Does the Foreign Key Go?](#6-the-golden-rule-where-does-the-foreign-key-go)
7. [Cardinality Notation Compared: Chen's vs. Crow's Foot vs. (min, max)](#7-cardinality-notation-compared-chens-vs-crows-foot-vs-min-max)
8. [Real-World Worked Examples (All Four Types)](#8-real-world-worked-examples-all-four-types)
9. [Changing Cardinality Over Time — A Design Trap](#9-changing-cardinality-over-time--a-design-trap)
10. [Chapter Summary](#10-chapter-summary)
11. [Cheat Sheet](#11-cheat-sheet)
12. [Exercises](#12-exercises)

---

## 1. What Is a Cardinality Constraint?

### Definition

A **cardinality constraint** specifies **how many instances of one entity can be associated with how many instances of another entity**, through a given relationship.

> ⚠️ **Reminder (from Module 2, §11 and Module 9, §3):** This is a **different concept** from "cardinality of a relation" (number of tuples/rows) — here, "cardinality" describes the **shape of a relationship**, not a row count. Same word, third distinct meaning across this handbook (recall "degree" had this same double-meaning problem too) — always use context to disambiguate.

### Why It Exists

Knowing *that* two entities relate isn't enough — the business needs to know the **shape** of that relationship to design correct constraints, correct foreign key placement, and correct application logic. "A student enrolls in courses" is incomplete without knowing: can a student enroll in only one course, or many? Can a course have only one student, or many?

### The Four Classical Types (Binary Relationships)

Since binary relationships (Module 9, §6) are by far the most common, cardinality is almost always discussed in terms of **two entities**:

```
              CARDINALITY TYPES (Entity A ↔ Entity B)
                            │
        ┌───────────┬────────────┬────────────┐
        ▼           ▼            ▼            ▼
      1 : 1        1 : N        N : 1        M : N
   (One-to-One) (One-to-Many)(Many-to-One)(Many-to-Many)
```

> 💡 **Note:** 1:N and N:1 describe the **exact same relationship, viewed from opposite directions** — they are not fundamentally different types, just different perspectives on the same asymmetric relationship. This is clarified fully in §3 and §4.

---

## 2. One-to-One (1:1)

### Definition

A **One-to-One** relationship means: **one instance of Entity A relates to at most one instance of Entity B, and vice versa.**

### Real Example

```
Employee  ──has──  ParkingSpot

One employee has at most one assigned parking spot.
One parking spot is assigned to at most one employee.
```

```
┌───────────┐     1        1     ┌──────────────┐
│ Employee  │─────────────────── │  ParkingSpot  │
└───────────┘                     └──────────────┘
```

### Other Classic 1:1 Examples

- `Country` ↔ `Capital` (each country has exactly one capital city; each capital belongs to exactly one country)
- `User` ↔ `UserProfile` (sometimes split for performance/security reasons — e.g., keeping sensitive profile data in a separate, more tightly access-controlled table)
- `Person` ↔ `Passport` (in many countries, one active passport per person)

### Mapping to Relational Model (Preview of Module 13)

For a 1:1 relationship, the foreign key can technically go on **either side** — the choice depends on **which side is optional** (fully explained using participation constraints in Module 11):

```sql
-- Option A: FK on Employee side
CREATE TABLE ParkingSpot (
    spot_id  INT PRIMARY KEY,
    location VARCHAR(30)
);
CREATE TABLE Employee (
    emp_id   INT PRIMARY KEY,
    name     VARCHAR(50),
    spot_id  INT UNIQUE,   -- UNIQUE enforces the "1:1" (not 1:N) constraint!
    FOREIGN KEY (spot_id) REFERENCES ParkingSpot(spot_id)
);
```

> ⚠️ **Critical, frequently-missed detail:** Simply adding a foreign key is **not enough** to enforce 1:1 — a bare foreign key column, by default, allows the *same* `ParkingSpot` to be referenced by multiple `Employee` rows (which would actually be 1:N, not 1:1!). The `UNIQUE` constraint (Module 4, §13; Module 5, §8) on the foreign key column is **exactly what converts a 1:N-shaped foreign key into a true 1:1 relationship.**

### Interview Importance

**High** — the "you need UNIQUE on the FK, not just a plain FK" detail above is a very common, specific gotcha interviewers test.

---

## 3. One-to-Many (1:N)

### Definition

A **One-to-Many** relationship means: **one instance of Entity A can relate to many instances of Entity B, but each instance of Entity B relates to at most one instance of Entity A.**

### Real Example

```
Department  ──has──  Employee

One department has many employees.
Each employee belongs to at most one department.
```

```
┌──────────────┐     1        N     ┌───────────┐
│  Department  │─────────────────── │ Employee  │
└──────────────┘                     └───────────┘
```

### This Is the MOST Common Relationship Shape

The overwhelming majority of real-world business relationships are 1:N — nearly every "owns," "contains," "belongs to," or "has many" business statement is a 1:N relationship.

### Mapping to Relational Model

The foreign key **always goes on the "Many" side** — this is explored fully as "The Golden Rule" in §6, but the intuition is: each `Employee` needs exactly one `dept_id` value to point to its one department; a `Department` cannot hold a single column listing "all its employees" without violating atomicity (Module 2, §18).

```sql
CREATE TABLE Department (
    dept_id    INT PRIMARY KEY,
    dept_name  VARCHAR(50)
);
CREATE TABLE Employee (
    emp_id    INT PRIMARY KEY,
    name      VARCHAR(50),
    dept_id   INT,                              -- FK on the "many" side, NO unique constraint
    FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
);
```

### Interview Importance

**Extremely High** — this is the default, most-practiced relationship shape in every schema design interview.

---

## 4. Many-to-One (N:1)

### Definition

**Many-to-One is not a distinct structural type — it is exactly the same relationship as One-to-Many, described from the opposite entity's perspective.**

```
"Many Employees belong to One Department"   (N:1, viewed from Employee's side)
        is the exact same fact as
"One Department has Many Employees"          (1:N, viewed from Department's side)
```

### Why This Distinction Sometimes Confuses Students

Textbooks and diagrams sometimes label the relationship arrow/notation differently depending on which entity is listed first in the sentence — but the underlying **mapping to a relational schema (foreign key on the "many" side) is identical** regardless of which direction you describe it in.

### Interview Importance

**Low** as a distinct concept — but **high** for demonstrating you understand it's the *same* relationship as 1:N, not a fourth separate category (a quick, easy point to earn by stating this clearly and confidently).

---

## 5. Many-to-Many (M:N)

### Definition

A **Many-to-Many** relationship means: **an instance of Entity A can relate to many instances of Entity B, AND an instance of Entity B can relate to many instances of Entity A, simultaneously.**

### Real Example

```
Student  ──enrolls in──  Course

One student can enroll in many courses.
One course can have many students enrolled.
```

```
┌───────────┐     M        N     ┌───────────┐
│  Student  │─────────────────── │  Course   │
└───────────┘                     └───────────┘
```

### Why M:N CANNOT Be Directly Mapped to a Simple Foreign Key (Critical Insight)

Unlike 1:N (where the "many" side gets a single FK column), M:N has **no single "many" side to place a foreign key on** — both sides are "many." Attempting to add `course_id` directly to `Student` would violate atomicity (a student needs to hold *multiple* course IDs, an illegal multi-valued attribute — recall Module 2, §18 and Module 7, §6's identical problem with multi-valued attributes!).

### The Solution: A Junction/Associative Table (Full Circle Back to Module 4, §7 and Module 6, §9)

```sql
CREATE TABLE Student (
    roll_no INT PRIMARY KEY,
    name    VARCHAR(50)
);
CREATE TABLE Course (
    course_id INT PRIMARY KEY,
    title     VARCHAR(50)
);
CREATE TABLE Enrollment (              -- Junction / Associative Entity
    roll_no    INT,
    course_id  INT,
    grade      CHAR(2),
    PRIMARY KEY (roll_no, course_id),   -- composite key (Module 4 §7)
    FOREIGN KEY (roll_no) REFERENCES Student(roll_no),
    FOREIGN KEY (course_id) REFERENCES Course(course_id)
);
```

```
              M : N                          converted to:
Student ═══════════════ Course      Student ──1:N──> Enrollment <──N:1── Course
```

> 💡 **Deep Insight:** Every M:N relationship, once mapped to the relational model, is **secretly decomposed into two 1:N relationships**, both pointing into a new junction table. This is one of the most important structural insights in all of relational schema design — M:N doesn't really exist as a *direct* structure in the relational model; it's always an *illusion* created by two 1:N foreign keys meeting in a junction table.

### Interview Importance

**Extremely High.** "How do you model a many-to-many relationship in a relational database?" is possibly the single most universally asked ER/schema design interview question at every level — the answer (junction table with composite key, decomposing M:N into two 1:N relationships) must be automatic.

---

## 6. The Golden Rule: Where Does the Foreign Key Go?

This section consolidates §2–§5 into one unified, memorable rule that resolves *any* binary relationship's mapping question instantly:

```
┌──────────────────────────────────────────────────────────────────┐
│  GOLDEN RULE: The foreign key ALWAYS goes on the "MANY" side       │
│  (or, for 1:1, on either side — with a UNIQUE constraint added).   │
│  For M:N, BOTH sides are "many," so NEITHER gets a direct FK —      │
│  instead, a NEW junction table is created, holding a FK to EACH    │
│  side (each individually 1:N into the junction table).             │
└──────────────────────────────────────────────────────────────────┘
```

| Cardinality | Where Does the FK Go? |
|---|---|
| 1 : 1 | Either side — add `UNIQUE` to prevent it becoming 1:N |
| 1 : N | On the **"N" (many)** side, pointing back to the "1" side |
| N : 1 | Same as 1:N — just described from the opposite direction |
| M : N | **Nowhere directly** — create a **junction table** with FKs to both sides |

### Why This Rule Works (The Underlying Logic)

A foreign key column can only hold **one value per row** (atomicity — Module 2, §18). So the side that can only ever be associated with **at most one** of the other side (the "1" in a 1:N, or either side in 1:1) is exactly the side capable of holding a single-valued foreign key pointing to the other. The "many" side needs a place to point to just one parent — hence the FK naturally belongs there.

### Interview Importance

**Extremely High** — this single rule, stated clearly and confidently, answers the vast majority of "how would you design the tables for this relationship" interview questions instantly and correctly.

---

## 7. Cardinality Notation Compared: Chen's vs. Crow's Foot vs. (min, max)

### Chen's Notation

Simple numbers/letters (`1`, `N`, `M`) written directly on the relationship lines, as shown throughout this module.

### Crow's Foot Notation (Recap from Module 6, §6)

```
──╫──   = exactly one         ──○──  = zero (optional)
──<     = many                ──○<   = zero or many
──╫<    = one or many
```

### (min, max) Notation — A Third, More Precise System

Some textbooks (and this is worth recognizing even if not your primary drawing style) use **(min, max)** pairs on each side of a relationship line, which elegantly combines **cardinality AND participation** (Module 11) into a single, more precise pair of numbers:

```
Department (1,1) ──has── (0,N) Employee

  Department side (1,1): every Department has AT LEAST 1 and AT MOST 1... wait,
    this reads FROM the Employee's perspective: each Employee relates to
    exactly 1 Department (min 1, max 1).
  Employee side (0,N): each Department relates to 0 to N Employees
    (min 0 — a brand new department might have no employees yet;
     max N — unlimited).
```

> ⚠️ **Notation Reading Direction Gotcha:** The (min, max) pair written *next to* an entity describes the **constraint on THAT entity's participation**, but conceptually you read it as "how many of the OTHER entity can ONE instance of THIS entity relate to." This directionality is a common source of diagram-reading errors — always double check by asking "from ONE instance of [this entity], how many of [the other entity] can it connect to?"

### Interview Importance

**Medium** — Chen's/Crow's Foot are far more commonly *drawn*; (min,max) is valuable to *recognize* since it's popular in some textbooks and formally elegant (it's actually the notation that most cleanly foreshadows Module 11's participation constraints, since it encodes both cardinality and participation together).

---

## 8. Real-World Worked Examples (All Four Types)

A single consolidated reference table, revisiting familiar domains from Module 3, with the cardinality type explicitly labeled:

| Relationship | Cardinality | Reasoning |
|---|---|---|
| `Country` ↔ `Capital` | **1:1** | Each country has exactly one capital; each capital belongs to exactly one country |
| `Department` ↔ `Employee` | **1:N** | One department has many employees; each employee has one department |
| `Author` ↔ `Book` (assuming single-author books) | **1:N** | One author writes many books; each book has one author |
| `Author` ↔ `Book` (allowing co-authored books) | **M:N** | A book can have multiple authors; an author writes multiple books — needs a junction table `BookAuthor` |
| `Student` ↔ `Course` | **M:N** | Standard enrollment scenario |
| `Patient` ↔ `Doctor` (via appointments) | **M:N** | A patient sees multiple doctors over time; a doctor sees multiple patients — junction table: `Appointment` |
| `Employee` ↔ `ParkingSpot` | **1:1** | Discussed fully in §2 |
| `Order` ↔ `OrderLineItem` | **1:N** | One order has many line items (recall this is also a *weak entity* relationship — Module 8, §8) |

> 💡 **Notice:** the `Author`–`Book` example demonstrates that **cardinality is a business-rule decision, not a fixed fact** — the *same* two entity types can be 1:N or M:N depending on the *actual business rule* ("can a book have multiple authors?"). Always confirm the true cardinality with real requirements — never assume.

---

## 9. Changing Cardinality Over Time — A Design Trap

### The Problem

Business requirements change. A relationship correctly modeled as 1:N today might genuinely need to become M:N tomorrow (e.g., "we assumed one author per book, but now we're adding co-authoring support").

### Why This Is Painful

If `Book` originally stored `author_id` as a direct foreign key column (correct for 1:N), supporting M:N later requires:
1. Creating a new `BookAuthor` junction table.
2. Migrating all existing `author_id` values into it.
3. Removing the `author_id` column from `Book`.
4. Updating **every single query** in the application that previously joined directly via `Book.author_id`.

### Best Practices to Reduce Future Pain

- During initial ER modeling (this very module!), explicitly **ask stakeholders** whether a seemingly-1:N relationship could plausibly become M:N in the foreseeable future — it's far cheaper to build the junction table now than to migrate later, *if* there's genuine reason to expect the change.
- However, avoid the opposite mistake of over-engineering **every** relationship as M:N "just in case" — this adds unnecessary join complexity for relationships that will realistically always remain 1:N. This is a judgment call, and asking the right requirements questions upfront (a core ER-modeling skill, Module 6 §3) is the real solution — not a purely technical hedge.

### Interview Importance

**High for Senior/Staff-level roles** — discussing this exact trade-off (over-engineering M:N "just in case" vs. under-engineering and facing a painful migration) is a strong signal of real-world design maturity beyond textbook cardinality memorization.

---

## 10. Chapter Summary

- A **cardinality constraint** specifies the "shape" of a relationship — **not** the same concept as "cardinality of a relation" (row count, Module 2 §11) despite sharing the word.
- **1:1** relationships require a `UNIQUE` constraint on the foreign key (not just a bare FK) to prevent it from silently becoming 1:N.
- **1:N** (equivalently, **N:1**, just viewed from the opposite side) is the most common relationship shape — the foreign key goes on the "many" side.
- **M:N** relationships cannot be directly represented with a simple foreign key (would violate atomicity, Module 2 §18) — they require a **junction/associative table** with a composite key, which secretly decomposes the single M:N relationship into **two 1:N relationships**.
- The **Golden Rule** ("FK goes on the many side; M:N needs a junction table with FKs on both sides") resolves virtually any binary-relationship mapping question instantly.
- Cardinality can be notated via Chen's simple numbers, Crow's Foot symbols, or the more precise **(min, max)** pair notation (which elegantly foreshadows Module 11's participation constraints).
- **Cardinality is a business-rule decision** (e.g., single-author vs. co-authored books changes 1:N to M:N) — always confirm with real requirements, and consider (without over-engineering) whether today's 1:N might realistically need to become M:N in the future.

---

## 11. Cheat Sheet

```
CARDINALITY TYPES (binary relationships):
  1:1   → FK on either side + UNIQUE constraint (critical! bare FK ≠ 1:1)
  1:N   → FK on the "N" (many) side, pointing to the "1" side
  N:1   → same as 1:N, just described from the opposite direction
  M:N   → NO direct FK possible — requires a JUNCTION TABLE
           (composite PK, FK to BOTH sides — secretly = two 1:N relationships)

GOLDEN RULE:
  "The foreign key always goes on the MANY side."
  "M:N = two 1:N relationships meeting at a junction table."

NOTATION SYSTEMS:
  Chen's:        plain numbers/letters (1, N, M) on relationship lines
  Crow's Foot:   ──╫── one, ──○── zero, ──< many
  (min, max):    e.g., (0,N) — combines cardinality AND participation (→ Module 11)

COMMON REAL EXAMPLES:
  1:1  → Country↔Capital, Employee↔ParkingSpot
  1:N  → Department↔Employee, Order↔OrderLineItem
  M:N  → Student↔Course, Patient↔Doctor (via Appointment), co-authored Book↔Author

REMEMBER: Cardinality is a BUSINESS RULE, not a fixed technical fact —
          always confirm with stakeholders, and don't assume.
```

---

## 12. Exercises

### Beginner

1. Define 1:1, 1:N, and M:N cardinality in one sentence each.
2. True/False: N:1 is a fundamentally different relationship type from 1:N.
3. Fill in the blank: In a many-to-many relationship, the relational model requires a __________ table with a __________ primary key.

### Intermediate

4. A `User` can have exactly one `Profile`, and each `Profile` belongs to exactly one `User`. Write the SQL schema, including the constraint that specifically prevents this from silently becoming a 1:N relationship.
5. Using the Golden Rule (§6), determine where the foreign key belongs for a `Warehouse` ↔ `InventoryItem` relationship where one warehouse stores many inventory items, and each inventory item is stored in exactly one warehouse. Write the SQL.
6. `Patient` ↔ `Doctor` is M:N via `Appointment`. Design the full SQL schema, including reasonable additional attributes on the `Appointment` junction table (e.g., appointment_date, status).

### Advanced / Conceptual

7. Explain, using atomicity (Module 2 §18), exactly why a many-to-many relationship cannot be represented with a single foreign key column on either side.
8. A schema originally modeled `Book.author_id` as a 1:N relationship (one author per book). The business now wants to support co-authored books. Describe the full migration needed to convert this into a proper M:N relationship.
9. Discuss the trade-off from §9: when is it worth proactively modeling a relationship as M:N "just in case," versus keeping it simple as 1:N until requirements clearly demand otherwise? Give one concrete example of each choice being the right call.

### MCQs

**Q1.** In a 1:N relationship, the foreign key is placed:
a) On the "1" side  b) On the "N" (many) side  c) On neither side  d) On both sides equally

**Q2.** A many-to-many relationship is implemented in a relational database using:
a) A single foreign key on one side  b) Two foreign keys on the "1" side only  c) A junction table with a composite key  d) It cannot be implemented in a relational database

**Q3.** To properly enforce a 1:1 relationship (not accidentally allow 1:N), a foreign key column should also have:
a) A CHECK constraint  b) A UNIQUE constraint  c) A DEFAULT value  d) No additional constraint is needed

**Q4.** Which pair of terms describes the exact same relationship from opposite perspectives?
a) 1:1 and M:N  b) 1:N and N:1  c) 1:N and M:N  d) 1:1 and 1:N

*(Answer key: 1-b, 2-c, 3-b, 4-b)*

### Schema Design Exercise

10. Design the complete SQL schema (all tables, keys, and cardinality-appropriate constraints) for a `Movie` ↔ `Actor` relationship where a movie has many actors, and an actor appears in many movies (a classic M:N case), including a `role_name` attribute on the relationship.

### Practice Links

- [LeetCode Database — "Combine Two Tables" and join-based problems (1:N practice)](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL — Advanced Join (M:N junction table practice)](https://www.hackerrank.com/domains/sql)
- [DataLemur — real interview SQL questions involving relationship modeling](https://datalemur.com/questions)

---

**Next: Module 11 — Participation Constraints (Total Participation, Partial Participation, Mandatory Participation, Optional Participation).**

> Say "continue" or "next module" to proceed to Module 11.
