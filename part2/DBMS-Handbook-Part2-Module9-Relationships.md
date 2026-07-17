# DBMS Handbook — Part 2
## Module 9: Relationships

> **Builds on:** Module 6, §5 (Relationships introduced as an ER building block), Module 8 (Identifying Relationships — a special case covered there).
>
> **Feeds into:** Module 10 (Cardinality Constraints), Module 11 (Participation Constraints), Module 13 (ER-to-Relational Mapping — relationship mapping rules).

---

## Table of Contents

1. [What Is a Relationship (Formal Definition)?](#1-what-is-a-relationship-formal-definition)
2. [Relationship Set](#2-relationship-set)
3. [Degree of Relationship](#3-degree-of-relationship)
4. [Unary (Degree 1) Relationships](#4-unary-degree-1-relationships)
5. [Recursive Relationships — Full Deep Dive](#5-recursive-relationships--full-deep-dive)
6. [Binary (Degree 2) Relationships](#6-binary-degree-2-relationships)
7. [Ternary (Degree 3) Relationships](#7-ternary-degree-3-relationships)
8. [N-ary Relationships (General Case)](#8-n-ary-relationships-general-case)
9. [Why Higher-Degree Relationships Are Avoided When Possible](#9-why-higher-degree-relationships-are-avoided-when-possible)
10. [Relationship Attributes](#10-relationship-attributes)
11. [Chapter Summary](#11-chapter-summary)
12. [Cheat Sheet](#12-cheat-sheet)
13. [Exercises](#13-exercises)

---

## 1. What Is a Relationship (Formal Definition)?

### Definition

A **relationship** is a **meaningful association between two or more entities**. If entities are the "nouns" of a business domain, relationships are the "verbs" connecting them.

```
Student  ──Enrolls In──  Course
Employee  ──Manages──  Employee
Doctor  ──Treats──  Patient  (with a Hospital involved too — ternary, §7)
```

### Why It Exists

Recall Module 2, §2: the relational model represents *everything*, including associations between entities, uniformly. But at the **conceptual** (ER) level (Module 6, §4), we need explicit vocabulary to capture *which* entities are meaningfully connected and *how* — this is exactly what a relationship expresses, before any foreign key ever gets written.

### Interview Importance

**High** — foundational vocabulary for every subsequent ER topic in this Part.

---

## 2. Relationship Set

### Definition

Just as an **Entity Set** is the full collection of entities of one type (Module 6, §5), a **Relationship Set** is the full collection of relationship *instances* of one type.

```
Relationship Set: Enrolls
  Instance 1: (Student 101, Course CS101)
  Instance 2: (Student 101, Course CS102)
  Instance 3: (Student 102, Course CS101)
```

### The Entity/Relationship Parallel (Reinforcing Module 6, §5)

| Entity World | Relationship World |
|---|---|
| Entity (one instance) | Relationship (one instance/link) |
| Entity Set (all instances of a type) | Relationship Set (all links of a type) |
| → becomes a **table** (Module 13) | → becomes a **table** too (often a junction table, Module 4 §7) |

### Interview Importance

**Medium** — mostly valuable for the structural parallel with entities.

---

## 3. Degree of Relationship

### Definition

The **degree of a relationship** is the **number of entity types (entity sets) participating** in that relationship.

> ⚠️ **Critical reminder (already flagged in Module 2, §10):** This is a **completely different concept** from the "degree of a relation" (number of attributes/columns in a table). Same word, two entirely unrelated meanings — one of the most classic exam/interview traps in all of DBMS terminology.

| Degree | Name | Number of Entity Types Involved |
|---|---|---|
| 1 | Unary (or Recursive) | 1 |
| 2 | Binary | 2 |
| 3 | Ternary | 3 |
| n | N-ary | n |

The rest of this module walks through each degree, in increasing order.

---

## 4. Unary (Degree 1) Relationships

### Definition

A **unary relationship** connects entities **within the same entity set** to each other — i.e., an entity type relates to *itself*.

### Example

```
Employee ──Manages──> Employee
   (one employee manages another employee — both are "Employee" entities)
```

### Why This Feels Unusual at First

Beginners often initially expect a relationship to always connect two *different-looking* boxes on a diagram. A unary relationship connects a **single entity box to itself**, typically drawn with the relationship diamond looping back:

```
        ┌───────────────┐
        │   Employee    │
        └───────┬───────┘
                │
         ┌──────┴──────┐
         │   Manages    │◄──┐
         └──────┬───────┘   │
                └───────────┘
          (loops back to the same entity)
```

### Interview Importance

**Medium** — mostly important as the setup for the much more heavily-tested **recursive relationship** pattern, covered next.

---

## 5. Recursive Relationships — Full Deep Dive

> **Note:** "Unary relationship" and "recursive relationship" refer to the same underlying structural pattern — "unary" describes it by *degree* (§3), while "recursive" describes it by *behavior* (the entity type relating to instances of itself). Both terms are used interchangeably in practice, and you should recognize either.

### Why Recursive Relationships Matter So Much (High Interview Value)

Recursive relationships model extremely common real-world hierarchical/network structures **without needing a separate hierarchical DBMS** (recall Module 1, §6 — this is precisely how the relational model naturally subsumes the use cases that once required the now-legacy hierarchical model!).

### Classic Examples

| Domain | Recursive Relationship |
|---|---|
| Org chart | `Employee.manager_id → Employee.emp_id` |
| Social network | `User "follows" User` |
| Category taxonomy | `Category.parent_category_id → Category.category_id` |
| Bill of materials (manufacturing) | `Part "is made of" Part` (a car is made of an engine, which is made of pistons, ...) |
| File system | `Folder "contains" Folder` |

### Mapping to Relational Model (Preview of Module 13, Shown Fully Here Since It's So Common)

A recursive relationship becomes a **self-referencing foreign key** — a foreign key column in a table that references that **same table's** own primary key.

```sql
CREATE TABLE Employee (
    emp_id      INT PRIMARY KEY,
    name        VARCHAR(50),
    manager_id  INT,                          -- self-referencing FK
    FOREIGN KEY (manager_id) REFERENCES Employee(emp_id)
);
```

```
Employee
+---------+---------+-------------+
| emp_id  | name    | manager_id  |
+---------+---------+-------------+
| 1       | Aditi   | NULL        |   ← CEO, no manager (recall Module 2 §17's NULL example!)
| 2       | Rohan   | 1           |   ← reports to Aditi
| 3       | Kavya   | 2           |   ← reports to Rohan
+---------+---------+-------------+
```

### Recursive Relationship Cardinality Types (Preview of Module 10)

Recursive relationships aren't always 1:N like the manager example — the cardinality (Module 10) varies by scenario:

| Scenario | Cardinality | Example |
|---|---|---|
| One manages many | 1:N | `Employee` manager hierarchy |
| Many-to-many self-relation | M:N | `User "follows" User` (Instagram/Twitter-style — needs a junction table, e.g. `Follows(follower_id, followee_id)`) |
| One-to-one self-relation | 1:1 | `Employee "is understudy for" Employee` (rare, but possible) |

### Querying Recursive Data (Brief Preview, Fully Covered in a Later SQL Part)

Traversing a recursive relationship of *arbitrary depth* (e.g., "find all employees under Rohan, at any level") requires a **Recursive Common Table Expression (CTE)** in modern SQL — a technique covered fully in the Advanced SQL module of Part 3, but worth knowing this is *why* recursive relationships matter so much for real query design, not just diagram theory.

```sql
-- Preview only (full treatment in Part 3):
WITH RECURSIVE Subordinates AS (
    SELECT emp_id, name, manager_id FROM Employee WHERE emp_id = 2
    UNION ALL
    SELECT e.emp_id, e.name, e.manager_id
    FROM Employee e
    JOIN Subordinates s ON e.manager_id = s.emp_id
)
SELECT * FROM Subordinates;
```

### Common Mistakes

- Forgetting `manager_id` must be **nullable** (the top of the hierarchy has no parent) — a very common oversight leading to an inability to represent the CEO/root node.
- Attempting to enforce "no cycles" (e.g., A manages B, B manages A) using only basic constraints — standard `FOREIGN KEY`/`CHECK` constraints **cannot** express "no cycles in this graph" on their own; this typically requires application logic or specialized graph constraints/triggers.

### Interview Importance

**Extremely High.** Recursive relationships (self-referencing foreign keys, and the recursive CTE needed to query them) are one of the single most common "gotcha" schema design + SQL questions across all interview levels, from junior to Staff Engineer.

---

## 6. Binary (Degree 2) Relationships

### Definition

A **binary relationship** connects **two distinct entity types** — this is, by far, the **most common** relationship degree in real-world schema design.

### Example

```
Student ──Enrolls In──> Course     (two distinct entity types: Student, Course)
Employee ──Works In──> Department  (two distinct entity types: Employee, Department)
```

### Why Binary Relationships Dominate

Nearly every relationship you'll design in practice is binary — cardinality types (1:1, 1:N, M:N — Module 10) are almost always discussed and taught specifically in the context of binary relationships, since they're both the simplest to reason about and the overwhelming majority of real cases.

### Interview Importance

**Extremely High** — this is the "default," most heavily tested relationship shape, and the entire next module (Module 10, Cardinality) is built almost entirely around binary relationship examples.

---

## 7. Ternary (Degree 3) Relationships

### Definition

A **ternary relationship** connects **three distinct entity types simultaneously**, where the relationship's meaning genuinely requires **all three** to be present together — it is **not** simply three separate binary relationships in disguise.

### Classic Example

```
Doctor ──┐
         ├── Prescribes ── Patient
Drug ────┘

A single "Prescribes" relationship instance means:
  "THIS Doctor prescribed THIS Drug to THIS Patient (on this date)"
```

```
┌──────────┐
│  Doctor  │
└────┬─────┘
     │
     ▼
╔═══════════════╗
║  Prescribes   ║────────┐
╚═══════════════╝        │
     ▲                   ▼
     │              ┌──────────┐
┌──────────┐        │  Patient │
│   Drug   │        └──────────┘
└──────────┘
```

### Why This Cannot Be Simplified to Three Binary Relationships (Critical Insight)

This is the **single most important, most frequently tested conceptual point** about ternary relationships:

> Breaking `Prescribes(Doctor, Drug, Patient)` into three separate binary relationships — `Doctor–Drug`, `Drug–Patient`, `Doctor–Patient` — **loses information**. Specifically, it becomes impossible to know *which doctor* prescribed *which drug* to *which specific patient* as one linked fact; you'd only know that "some doctor prescribes some drug" and "some drug goes to some patient" as separate, disconnected statements, unable to reconstruct the true three-way combinations.

```
❌ WRONG (loses the true 3-way association):
   Doctor-Drug: (Dr. Mehta, Paracetamol)
   Drug-Patient: (Paracetamol, Patient Rahul)
   Doctor-Patient: (Dr. Mehta, Patient Rahul)

   → Cannot determine: did Dr. Mehta actually prescribe Paracetamol
     specifically TO Rahul? Or did Dr. Mehta prescribe something
     else to Rahul, while separately, someone else gave Rahul
     Paracetamol? The three-way fact is GENUINELY AMBIGUOUS once split.

✅ CORRECT (ternary relationship preserves the full 3-way fact):
   Prescribes: (Dr. Mehta, Paracetamol, Rahul, date: 2026-07-10)
```

### Mapping to Relational Model (Preview of Module 13)

A ternary relationship becomes a table with a **composite key made of all three participating entities' keys** (extending the composite key concept from Module 4, §7):

```sql
CREATE TABLE Prescribes (
    doctor_id    INT,
    drug_id      INT,
    patient_id   INT,
    prescribed_date DATE,
    dosage       VARCHAR(20),
    PRIMARY KEY (doctor_id, drug_id, patient_id),
    FOREIGN KEY (doctor_id) REFERENCES Doctor(doctor_id),
    FOREIGN KEY (drug_id) REFERENCES Drug(drug_id),
    FOREIGN KEY (patient_id) REFERENCES Patient(patient_id)
);
```

### Interview Importance

**High.** "Why can't you just split a ternary relationship into three binary ones?" is a classic, high-value GATE and interview conceptual question — the loss-of-information argument above is the exact expected answer.

---

## 8. N-ary Relationships (General Case)

### Definition

An **n-ary relationship** generalizes binary and ternary relationships to **any number (n) of participating entity types**.

### Example (Degree 4)

```
Supplier ── Supplies ── Part ── (to) Project ── (via) Shipment Method
  A single relationship instance: "Supplier X supplied Part Y to Project Z via Shipment Method W"
```

### Practical Note

Beyond ternary, n-ary relationships become **increasingly rare and increasingly hard to reason about** in real design work — see §9 for why practitioners actively try to avoid them when possible.

### Interview Importance

**Low-Medium** — mostly tested as a generalization/definition question, rarely requiring you to design a genuine 4+-ary relationship from scratch.

---

## 9. Why Higher-Degree Relationships Are Avoided When Possible

### The General Design Principle

> **Whenever practical, prefer binary relationships (or, if truly necessary, an associative entity — Module 6, §9) over higher-degree relationships.** Higher-degree relationships are harder to model correctly, harder to query, and more prone to real-world business-rule ambiguity.

### Techniques to Reduce Relationship Degree

1. **Promote the relationship to an associative entity** (Module 6, §9) — this is exactly what was done implicitly with `Prescribes` above; giving it its own identity (e.g., a `prescription_id` surrogate key) makes it easier to reference, extend with more attributes, and query, while still preserving all three (or more) foreign key links.

```sql
-- Same ternary fact, modeled as an associative entity with its own surrogate key:
CREATE TABLE Prescription (
    prescription_id  INT AUTO_INCREMENT PRIMARY KEY,   -- surrogate key
    doctor_id        INT,
    drug_id          INT,
    patient_id       INT,
    prescribed_date  DATE,
    FOREIGN KEY (doctor_id) REFERENCES Doctor(doctor_id),
    FOREIGN KEY (drug_id) REFERENCES Drug(drug_id),
    FOREIGN KEY (patient_id) REFERENCES Patient(patient_id)
);
```

2. **Question whether all participants are truly required simultaneously** — sometimes what looks like a ternary relationship is actually two independent binary relationships that were incorrectly conflated (the reverse mistake of §7 — always verify with real business stakeholders whether the *true* fact genuinely requires all three, or fewer).

### Interview Importance

**High** — recommending "promote to an associative entity with a surrogate key" is considered strong, senior-level design judgment in interviews involving complex multi-entity relationships.

---

## 10. Relationship Attributes

### Definition

Just like entities, **relationships themselves can have attributes** — properties that describe the relationship instance itself, not any single participating entity.

### Example

```
Student ──Enrolls In──> Course
              │
              ▼
        (attribute: enrollment_date, grade)
```

`enrollment_date` and `grade` don't belong to `Student` alone (a student might enroll in different courses on different dates with different grades) or `Course` alone (different students get different grades) — they belong to the **relationship instance itself**.

### Connection to Associative Entities (Module 6, §9)

> This is *precisely* the trigger condition for promoting a relationship into an **associative entity**, first discussed in Module 6, §9: once a relationship accumulates enough of its own meaningful attributes, it's often clearer (and easier to reference/extend) to model it as a full associative entity (like `Enrollment`) rather than a bare relationship diamond with attached attribute ovals.

### Notation

In Chen's notation, relationship attributes are drawn as ordinary ovals connected directly to the relationship diamond (not to either entity):

```
┌───────────┐        ◇──────────◇        ┌───────────┐
│  Student  │──────Enrolls In──────────│   Course  │
└───────────┘        ◇──────────◇        └───────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
              ┌──────────┐  ┌──────────┐
              │  grade   │  │enroll_dt │
              └──────────┘  └──────────┘
```

### Interview Importance

**High** — directly reinforces when/why to use an associative entity, a frequently tested design judgment call.

---

## 11. Chapter Summary

- A **relationship** is a meaningful association between entities; a **relationship set** is the full collection of instances of one relationship type — directly paralleling the entity/entity-set distinction (Module 6, §5).
- **Degree of relationship** (number of participating entity *types*) must never be confused with **degree of relation** (number of attributes/columns, Module 2, §10) — same word, unrelated meanings.
- **Unary/Recursive relationships** connect an entity type to itself — extremely common for hierarchies (org charts, category trees) and social graphs (follows/friends) — mapped to a **self-referencing foreign key**, almost always nullable at the "root" (Module 2, §17's NULL semantics apply directly).
- **Binary relationships** (two entity types) are by far the most common and the foundation for the cardinality discussion in Module 10.
- **Ternary relationships** (three entity types) genuinely cannot be decomposed into three separate binary relationships without **losing information** — this is the single most important conceptual insight in this module, and a classic interview/GATE trap.
- **N-ary relationships** generalize further but are rare and increasingly hard to reason about; the standard practical mitigation is to **promote the relationship into an associative entity** with its own surrogate key.
- **Relationship attributes** (properties of the relationship instance itself, not of any single entity) are the trigger condition for deciding to promote a relationship into a full associative entity (Module 6, §9).

---

## 12. Cheat Sheet

```
DEGREE OF RELATIONSHIP = number of participating ENTITY TYPES
  (NOT the same as "degree of a relation" = number of columns! Module 2 §10)

UNARY / RECURSIVE (degree 1):
  Entity type relates to itself
  → self-referencing FOREIGN KEY (e.g., Employee.manager_id → Employee.emp_id)
  → root/top of hierarchy needs a NULLABLE FK
  → querying arbitrary depth needs a RECURSIVE CTE (Part 3)
  → constraints alone CANNOT prevent cycles

BINARY (degree 2):
  Most common relationship shape — foundation of Module 10 (Cardinality)

TERNARY (degree 3):
  Genuinely connects 3 entity types AT ONCE
  → CANNOT be split into 3 binary relationships without LOSING information
  → maps to a table with a COMPOSITE key of all 3 participants' FKs

N-ARY (degree n):
  Generalization beyond ternary — rare, hard to reason about
  → best practice: PROMOTE to an ASSOCIATIVE ENTITY with its own surrogate key

RELATIONSHIP ATTRIBUTES:
  Properties of the relationship ITSELF (e.g., grade, enrollment_date)
  → accumulating enough of these = signal to promote to an ASSOCIATIVE ENTITY (Module 6 §9)
```

---

## 13. Exercises

### Beginner

1. Define "degree of a relationship" and give one example each of unary, binary, and ternary relationships.
2. True/False: The degree of a relationship and the degree of a relation (table) mean the same thing.
3. Fill in the blank: A relationship that connects an entity type to itself is called a __________ or __________ relationship.

### Intermediate

4. Design the SQL schema for a self-referencing `Category` table (e-commerce product categories, where a category can have a parent category), including appropriate nullability.
5. Explain, using a concrete example other than `Prescribes`, why a ternary relationship cannot be safely decomposed into three separate binary relationships.
6. A `Follows` relationship in a social media app connects `User` to `User` (many-to-many: a user can follow many users, and be followed by many). Design the SQL junction table for this recursive M:N relationship.

### Advanced / Conceptual

7. Discuss: when should a relationship with attributes be promoted into an associative entity rather than kept as a bare relationship? Use the `Enrolls In (grade, enrollment_date)` example to justify your answer.
8. A logistics system needs to track "Supplier X supplied Part Y to Warehouse Z on Date D, via Shipping Method M" — is this ternary or higher-degree? Design an appropriate schema, applying the "promote to associative entity" technique from §9.
9. Explain why a self-referencing foreign key (e.g., `manager_id`) alone cannot prevent a data-integrity cycle (A manages B, B manages A), and briefly suggest one non-constraint-based way this might be enforced in practice.

### MCQs

**Q1.** A relationship connecting an entity type to itself is called:
a) Binary  b) Ternary  c) Unary/Recursive  d) N-ary

**Q2.** The most common relationship degree in real-world schema design is:
a) Unary  b) Binary  c) Ternary  d) N-ary

**Q3.** A ternary relationship, when decomposed into three binary relationships, typically:
a) Preserves all information perfectly  b) Loses the true three-way association  c) Becomes easier to query  d) Is always the preferred approach

**Q4.** The standard mapping for a recursive 1:N relationship (like a manager hierarchy) is:
a) A junction table  b) A composite key across three tables  c) A self-referencing foreign key  d) A multi-valued attribute

*(Answer key: 1-c, 2-b, 3-b, 4-c)*

### Diagram Exercise

10. Draw a Chen's-notation ER diagram for a `Flight` booking system where a ternary relationship `Books` connects `Passenger`, `Flight`, and `SeatClass` (Economy/Business/First), with relationship attributes `booking_date` and `fare_paid`.

### Practice Links

- [LeetCode Database — "Employee" self-join / manager hierarchy problems (classic recursive relationship practice)](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL — Advanced Join (self-joins and hierarchical queries)](https://www.hackerrank.com/domains/sql)

---

**Next: Module 10 — Cardinality Constraints (One-to-One, One-to-Many, Many-to-One, Many-to-Many, with real examples and full ER + relational mapping notation).**

> Say "continue" or "next module" to proceed to Module 10.
