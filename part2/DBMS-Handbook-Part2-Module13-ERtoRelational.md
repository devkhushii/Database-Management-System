# DBMS Handbook — Part 2
## Module 13: Converting ER to Relational Model

> **Builds on:** Every previous module in this Part — this is where Modules 6-12's conceptual (ER/EER) vocabulary gets mechanically, systematically converted into the relational (Modules 2-5) vocabulary.
>
> **Feeds into:** Module 14 (Schema Design Basics — refining the raw output of this conversion process) and Normalization (Part 3).

---

## Table of Contents

1. [Why This Conversion Process Matters](#1-why-this-conversion-process-matters)
2. [Rule 1: Strong Entity Conversion](#2-rule-1-strong-entity-conversion)
3. [Rule 2: Weak Entity Conversion](#3-rule-2-weak-entity-conversion)
4. [Rule 3: Relationship Mapping (1:1, 1:N, M:N)](#4-rule-3-relationship-mapping-11-1n-mn)
5. [Rule 4: Multivalued Attribute Mapping](#5-rule-4-multivalued-attribute-mapping)
6. [Rule 5: Composite Attribute Mapping](#6-rule-5-composite-attribute-mapping)
7. [Rule 6: Derived Attribute Handling](#7-rule-6-derived-attribute-handling)
8. [Rule 7: Generalization/Specialization Mapping — Three Strategies](#8-rule-7-generalizationspecialization-mapping--three-strategies)
9. [Rule 8: Aggregation Mapping](#9-rule-8-aggregation-mapping)
10. [Rule 9: N-ary Relationship Mapping](#10-rule-9-n-ary-relationship-mapping)
11. [Complete Step-by-Step Worked Example: University System](#11-complete-step-by-step-worked-example-university-system)
12. [Master Mapping Rules Table](#12-master-mapping-rules-table)
13. [Chapter Summary](#13-chapter-summary)
14. [Cheat Sheet](#14-cheat-sheet)
15. [Exercises](#15-exercises)

---

## 1. Why This Conversion Process Matters

### The Big Picture

Recall the design pipeline established across this entire Part (Module 1, §8; Module 6, §3):

```
Requirements → ER/EER Model (Modules 6-12) → Relational Schema (THIS MODULE) → Normalization (Part 3)
```

This module is the **mechanical bridge** — a set of **precise, learnable, largely deterministic rules** that transform any ER/EER diagram into a correct, working set of relational tables. Because these rules are systematic, this is one of the most "practice makes perfect" topics in the entire handbook — and one of the most reliably, heavily tested skills in both GATE exams and system design interviews.

### Why It's Not "Just Common Sense"

While experienced designers often do this conversion intuitively, having the **explicit rule set** is what lets you handle unusual/edge cases correctly and confidently justify your design decisions in an interview, rather than guessing.

---

## 2. Rule 1: Strong Entity Conversion

### The Rule

> **Every strong entity becomes its own table.** Each of its simple/single-valued attributes becomes a column. Its key attribute becomes the table's `PRIMARY KEY`.

### Example

```
ER: Strong Entity "Student"
    Attributes: roll_no (key), name, dept

SQL:
CREATE TABLE Student (
    roll_no INT PRIMARY KEY,
    name    VARCHAR(50),
    dept    VARCHAR(10)
);
```

This is the simplest, most direct rule — a near 1:1 translation.

### Interview Importance

**High** (as the foundational baseline for everything else in this module).

---

## 3. Rule 2: Weak Entity Conversion

### The Rule (Fully Derived From Module 8's Formula)

> **Every weak entity becomes its own table.** Its primary key is the **composite** of: (a) the owner entity's primary key (as a foreign key), and (b) the weak entity's own partial key. The foreign key to the owner is typically declared `ON DELETE CASCADE` (reflecting the weak entity's mandatory total participation, Module 8 §3 / Module 11 §2).

### Example (Direct Continuation of Module 8, §6)

```
ER: Weak Entity "Dependent", owned by "Employee"
    Partial Key: dependent_name

SQL:
CREATE TABLE Dependent (
    emp_id           INT,
    dependent_name   VARCHAR(50),
    relationship_type VARCHAR(20),
    PRIMARY KEY (emp_id, dependent_name),
    FOREIGN KEY (emp_id) REFERENCES Employee(emp_id) ON DELETE CASCADE
);
```

> 💡 This rule is exactly what was already fully derived and demonstrated across Module 8's four real-system examples (§6-§9) — this section formalizes it as an explicit, numbered mapping rule for quick reference.

### Interview Importance

**Extremely High** — see Module 8 for the full depth; this module simply packages it as Rule 2 of a numbered system.

---

## 4. Rule 3: Relationship Mapping (1:1, 1:N, M:N)

### The Rule (Directly Formalizing Module 10's "Golden Rule")

> **For 1:1 relationships:** add a foreign key on either side (typically the side with total participation, if one exists — Module 11), with a `UNIQUE` constraint.
>
> **For 1:N relationships:** add a foreign key on the "N" (many) side, referencing the "1" side's primary key. Make it `NOT NULL` if participation is total (Module 11, §7), nullable if partial.
>
> **For M:N relationships:** create a **new junction table** with a composite primary key made of both sides' foreign keys (plus any relationship attributes, Module 9, §10, as extra columns).

### Worked Examples (One Per Cardinality Type, Consolidating Modules 10-11)

```sql
-- 1:1  (Employee ↔ ParkingSpot, Module 10 §2)
CREATE TABLE Employee (
    emp_id  INT PRIMARY KEY,
    spot_id INT UNIQUE,
    FOREIGN KEY (spot_id) REFERENCES ParkingSpot(spot_id)
);

-- 1:N  (Department ↔ Employee, Module 10 §3)
CREATE TABLE Employee (
    emp_id  INT PRIMARY KEY,
    dept_id INT NOT NULL,        -- NOT NULL if Employee's participation is TOTAL
    FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
);

-- M:N  (Student ↔ Course, Module 10 §5)
CREATE TABLE Enrollment (
    roll_no    INT,
    course_id  INT,
    grade      CHAR(2),          -- relationship attribute (Module 9 §10)
    PRIMARY KEY (roll_no, course_id),
    FOREIGN KEY (roll_no) REFERENCES Student(roll_no),
    FOREIGN KEY (course_id) REFERENCES Course(course_id)
);
```

### Interview Importance

**Extremely High** — this rule set was already deeply covered in Modules 10-11; this section is its formal packaging as "Rule 3."

---

## 5. Rule 4: Multivalued Attribute Mapping

### The Rule (Directly Formalizing Module 7, §6)

> **Every multi-valued attribute becomes its own new table**, containing: (a) a foreign key referencing the owning entity's primary key, and (b) a column for the attribute's value. The primary key of this new table is typically the **composite** of both columns (since the combination is what's naturally unique).

### Example

```sql
CREATE TABLE StudentPhone (
    roll_no       INT,
    phone_number  VARCHAR(15),
    PRIMARY KEY (roll_no, phone_number),
    FOREIGN KEY (roll_no) REFERENCES Student(roll_no)
);
```

### Interview Importance

**Very High** — see Module 7, §6 for the full depth; formalized here as Rule 4.

---

## 6. Rule 5: Composite Attribute Mapping

### The Rule (Directly Formalizing Module 7, §4)

> **Only the leaf-level (lowest-level) sub-attributes of a composite attribute become actual columns** — the top-level grouping name itself (e.g., "name") typically does **not** become a column; it's purely a conceptual/organizational label in the ER diagram.

### Example

```sql
-- name (composite: first_name, middle_name, last_name) → 3 separate columns:
CREATE TABLE Student (
    roll_no      INT PRIMARY KEY,
    first_name   VARCHAR(30),
    middle_name  VARCHAR(30),
    last_name    VARCHAR(30)
);
```

### Interview Importance

**High** — see Module 7, §4 for the full depth and design-decision nuance; formalized here as Rule 5.

---

## 7. Rule 6: Derived Attribute Handling

### The Rule (Directly Formalizing Module 7, §8)

> **Most derived attributes are NOT stored as columns at all** — they are computed on read (via a `SELECT` expression or a database VIEW), or, if performance genuinely demands it, implemented as a `GENERATED ALWAYS AS (...)` computed column (available in modern RDBMSs) to keep them automatically, reliably in sync with their source data.

### Example

```sql
-- Option A (most common): compute on read, don't store at all
SELECT roll_no, name, TIMESTAMPDIFF(YEAR, dob, CURDATE()) AS age FROM Student;

-- Option B (modern generated column, if frequently needed and indexed):
ALTER TABLE Student ADD COLUMN age INT
    GENERATED ALWAYS AS (TIMESTAMPDIFF(YEAR, dob, CURDATE())) STORED;
```

### Interview Importance

**High** — see Module 7, §8 for the full store-vs-compute trade-off discussion; formalized here as Rule 6.

---

## 8. Rule 7: Generalization/Specialization Mapping — Three Strategies

### Why This Rule Is More Complex Than the Others

Unlike the previous rules (which each have one clear, standard approach), **mapping an ISA hierarchy (Module 12) to relational tables has THREE legitimate, commonly-used strategies**, each with real trade-offs. Choosing between them is a genuine, frequently-interviewed **design decision**, not a mechanical lookup.

### Strategy 1: One Table Per Subclass Only (No Superclass Table)

Push all superclass attributes **down** into each subclass table; there is **no** separate `Employee`-style table at all.

```sql
CREATE TABLE Manager (
    emp_id         INT PRIMARY KEY,
    name           VARCHAR(50),      -- duplicated superclass attribute
    salary         DECIMAL(10,2),    -- duplicated superclass attribute
    dept_managed   VARCHAR(50)
);
CREATE TABLE Engineer (
    emp_id           INT PRIMARY KEY,
    name             VARCHAR(50),    -- duplicated
    salary           DECIMAL(10,2),  -- duplicated
    specialization   VARCHAR(50)
);
```

- **Best used when:** the specialization is **Disjoint AND Total** (Module 12, §9) — since every instance belongs to exactly one subclass, there's no ambiguity about "which table does a generic Employee go in" (there IS no generic table).
- **Disadvantage:** Attribute duplication across tables (`name`, `salary` repeated); querying "all employees regardless of subtype" requires a `UNION` across every subclass table — awkward and error-prone if a new subclass is added later.

### Strategy 2: One Table Per Subclass, Plus a Superclass Table (Class/Subclass Tables)

Keep the superclass table `Employee`, AND create a table per subclass containing **only its own unique attributes** plus a **foreign key (which is also its primary key)** back to the superclass table.

```sql
CREATE TABLE Employee (
    emp_id  INT PRIMARY KEY,
    name    VARCHAR(50),
    salary  DECIMAL(10,2)
);
CREATE TABLE Manager (
    emp_id        INT PRIMARY KEY,
    dept_managed  VARCHAR(50),
    FOREIGN KEY (emp_id) REFERENCES Employee(emp_id)
);
CREATE TABLE Engineer (
    emp_id          INT PRIMARY KEY,
    specialization  VARCHAR(50),
    FOREIGN KEY (emp_id) REFERENCES Employee(emp_id)
);
```

- **Best used when:** the specialization is **Overlapping and/or Partial** (Module 12, §9) — this strategy elegantly handles instances belonging to zero, one, or multiple subclasses (an `Employee` row simply has, or doesn't have, a matching row in `Manager` and/or `Engineer`), and it also handles "generic" employees (belonging to no subclass) gracefully.
- **Advantage:** No attribute duplication; querying "all employees" is trivial (`SELECT * FROM Employee`); this is generally considered the **most flexible, most commonly recommended default strategy** in real-world practice.
- **Disadvantage:** Requires a `JOIN` to retrieve an employee's full subtype-specific details; slightly more tables to manage.

### Strategy 3: Single Table with a Type Discriminator Column (Single Table Inheritance)

Merge **everything** — superclass and all subclasses — into **one single table**, with an extra "type" column indicating which subclass each row represents, and nullable columns for every subclass-specific attribute.

```sql
CREATE TABLE Employee (
    emp_id          INT PRIMARY KEY,
    name            VARCHAR(50),
    salary          DECIMAL(10,2),
    employee_type   VARCHAR(20),        -- discriminator: 'Manager', 'Engineer', etc.
    dept_managed    VARCHAR(50) NULL,   -- only relevant if employee_type = 'Manager'
    specialization  VARCHAR(50) NULL    -- only relevant if employee_type = 'Engineer'
);
```

- **Best used when:** the number of subclasses is small, subclass-specific attributes are few, and query simplicity (no joins ever needed) is highly valued.
- **Disadvantage:** Many columns will be `NULL` for any given row (sparse table) — wastes some storage and makes the schema less self-documenting (a `CHECK` constraint can help enforce "these columns are only non-null for this type," but adds complexity); doesn't scale gracefully if subclasses grow numerous or attribute-rich.

### Decision Table — Which Strategy to Choose

| Scenario | Recommended Strategy |
|---|---|
| Disjoint + Total, few queries need "all superclass instances together" | Strategy 1 (subclass tables only) |
| Overlapping and/or Partial, need flexibility, general real-world default | **Strategy 2 (superclass + subclass tables)** — most commonly recommended |
| Few subclasses, few extra attributes, want to avoid joins entirely | Strategy 3 (single table with discriminator) |

### Interview Importance

**Extremely High.** "How would you map this class hierarchy to a relational schema, and why?" — expecting you to name and compare these three strategies, and justify a specific choice based on the hierarchy's disjoint/overlapping and total/partial constraints (Module 12, §9) — is one of the most sophisticated, senior-level-signaling questions in this entire handbook.

---

## 9. Rule 8: Aggregation Mapping

### The Rule (Referencing Module 12, §10)

> Since most real-world designs resolve an EER "Aggregation" into an ordinary **associative entity** (Module 6, §9) rather than using formal aggregation notation, the mapping is simply: **treat the aggregated relationship as its own table (with a surrogate or composite key), and give it a further foreign key to whatever entity it needs to relate to.**

### Example (Continuing Module 12, §10's Employee-Project-Equipment Scenario)

```sql
CREATE TABLE Assignment (              -- the "aggregated" Works-On relationship,
    assignment_id  INT AUTO_INCREMENT PRIMARY KEY,   -- promoted to a full entity
    emp_id         INT,
    project_id     INT,
    FOREIGN KEY (emp_id) REFERENCES Employee(emp_id),
    FOREIGN KEY (project_id) REFERENCES Project(project_id)
);
CREATE TABLE AssignmentEquipment (     -- the further "Uses" relationship
    assignment_id  INT,
    equipment_id   INT,
    PRIMARY KEY (assignment_id, equipment_id),
    FOREIGN KEY (assignment_id) REFERENCES Assignment(assignment_id),
    FOREIGN KEY (equipment_id) REFERENCES Equipment(equipment_id)
);
```

### Interview Importance

**Medium** — mostly valuable for recognizing that aggregation and associative-entity promotion converge on the same practical relational outcome.

---

## 10. Rule 9: N-ary Relationship Mapping

### The Rule (Directly Formalizing Module 9, §7-§8)

> **An n-ary relationship (n ≥ 3) becomes a new table** with a foreign key to **each** of the n participating entities, plus columns for any relationship attributes. The primary key is typically the composite of all n foreign keys (unless business rules require a different, possibly surrogate, key).

### Example (Continuing Module 9, §7's Ternary Prescribes Relationship)

```sql
CREATE TABLE Prescribes (
    doctor_id       INT,
    drug_id         INT,
    patient_id      INT,
    prescribed_date DATE,
    PRIMARY KEY (doctor_id, drug_id, patient_id),
    FOREIGN KEY (doctor_id) REFERENCES Doctor(doctor_id),
    FOREIGN KEY (drug_id) REFERENCES Drug(drug_id),
    FOREIGN KEY (patient_id) REFERENCES Patient(patient_id)
);
```

### Interview Importance

**High** — see Module 9, §7-§9 for the full depth; formalized here as Rule 9.

---

## 11. Complete Step-by-Step Worked Example: University System

Let's apply **every single rule from this module, in order**, to design a complete, realistic University System from scratch — this is the capstone exercise tying together the entire Part.

### Step 1: Requirements (Informal)

- The university has `Student`s and `Professor`s — both are kinds of `Person` (they share `name`, `dob`; overlapping is possible, since a person could be a student AND, unusually, also employed as a teaching assistant/adjunct — but let's assume for simplicity, **Disjoint** for this example).
- Each `Student` has a `roll_no`, multiple `phone_numbers` (multi-valued), and a `full_name` (composite: first/last).
- A `Department` `offers` many `Course`s (1:N); a `Department` must have exactly one `Professor` as its `head` (1:1, total on Department side).
- `Student`s `enroll` in `Course`s (M:N), with a `grade` recorded per enrollment.
- Each `Course` has many `CourseSection`s — a `CourseSection` has no independent meaning outside its `Course` (weak entity, partial key = `section_number`).

### Step 2: Full ER/EER Summary (Referencing Every Relevant Prior Module)

```
Person (Superclass, Module 12) — Disjoint, Partial (some Persons might be neither)
  ├── Student (Module 6 §7 style strong-ish, but here a subclass)
  └── Professor

Department (Strong Entity)
  ── Offers (1:N) ──> Course (Strong Entity)
  ── Headed By (1:1, Total on Department) ──> Professor

Course
  ── Has (Identifying Relationship) ──> CourseSection (Weak Entity, Module 8)

Student ── Enrolls (M:N, Module 9/10) ──> Course  [grade: relationship attribute]
```

### Step 3: Apply Every Mapping Rule

```sql
-- Rule 7 (Generalization/Specialization) — using Strategy 2 (superclass + subclass tables),
-- appropriate here since Person's specialization is Partial (Module 12 §8):
CREATE TABLE Person (
    person_id  INT PRIMARY KEY,
    first_name VARCHAR(30),          -- Rule 5: composite attribute → leaf columns
    last_name  VARCHAR(30),
    dob        DATE
);
CREATE TABLE Student (
    person_id  INT PRIMARY KEY,       -- Rule 7, Strategy 2: shares Person's key
    roll_no    VARCHAR(15) UNIQUE,    -- natural/business key (Module 4 §10, §15)
    FOREIGN KEY (person_id) REFERENCES Person(person_id)
);
CREATE TABLE StudentPhone (            -- Rule 4: multivalued attribute
    person_id     INT,
    phone_number  VARCHAR(15),
    PRIMARY KEY (person_id, phone_number),
    FOREIGN KEY (person_id) REFERENCES Student(person_id)
);
CREATE TABLE Professor (
    person_id     INT PRIMARY KEY,    -- Rule 7, Strategy 2
    hire_date     DATE,
    FOREIGN KEY (person_id) REFERENCES Person(person_id)
);

-- Rule 1: Strong Entity
CREATE TABLE Department (
    dept_id     INT PRIMARY KEY,
    dept_name   VARCHAR(50),
    head_id     INT NOT NULL UNIQUE,   -- Rule 3: 1:1, TOTAL on Department → NOT NULL + UNIQUE
    FOREIGN KEY (head_id) REFERENCES Professor(person_id)
);
CREATE TABLE Course (
    course_id   INT PRIMARY KEY,
    title       VARCHAR(50),
    dept_id     INT NOT NULL,          -- Rule 3: 1:N, FK on the "many" side
    FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
);

-- Rule 2: Weak Entity
CREATE TABLE CourseSection (
    course_id      INT,
    section_number INT,               -- partial key
    room           VARCHAR(20),
    PRIMARY KEY (course_id, section_number),
    FOREIGN KEY (course_id) REFERENCES Course(course_id) ON DELETE CASCADE
);

-- Rule 3: M:N Relationship
CREATE TABLE Enrollment (
    person_id   INT,
    course_id   INT,
    grade       CHAR(2),
    PRIMARY KEY (person_id, course_id),
    FOREIGN KEY (person_id) REFERENCES Student(person_id),
    FOREIGN KEY (course_id) REFERENCES Course(course_id)
);
```

### Step 4: Reflection

Notice how **every single design decision** in this final schema traces back to a specific, named rule from an earlier module: the `Student`/`Professor` split used **Rule 7, Strategy 2** (justified by Module 12's Partial specialization); `Department.head_id` got `NOT NULL UNIQUE` because of **Rule 3's** 1:1-Total mapping (Module 10 §2 + Module 11 §7); `CourseSection` used **Rule 2's** weak-entity composite-key-plus-cascade pattern (Module 8); and `Enrollment` used **Rule 3's** M:N junction-table pattern (Module 10 §5). This traceability — being able to justify *every* line of SQL by pointing back to a specific conceptual decision — is exactly the skill senior database design interviews are testing for.

---

## 12. Master Mapping Rules Table

A single, consolidated, revision-ready reference for every rule covered in this module:

| ER/EER Construct | Relational Mapping Rule | Full Detail |
|---|---|---|
| Strong Entity | Own table; key attribute → PRIMARY KEY | §2 |
| Weak Entity | Own table; composite PK = (owner's PK + partial key); FK typically `ON DELETE CASCADE` | §3, Module 8 |
| 1:1 Relationship | FK on either side (often the total-participation side) + `UNIQUE` | §4, Module 10 §2 |
| 1:N Relationship | FK on the "many" side; `NOT NULL` if total, nullable if partial | §4, Module 10 §3, Module 11 §7 |
| M:N Relationship | New junction table; composite PK of both FKs; relationship attributes as extra columns | §4, Module 10 §5 |
| Multivalued Attribute | New table with FK to owner + value column; composite PK | §5, Module 7 §6 |
| Composite Attribute | Only leaf sub-attributes become columns | §6, Module 7 §4 |
| Derived Attribute | Usually NOT stored — computed on read, or a `GENERATED` column | §7, Module 7 §8 |
| Generalization/Specialization | THREE strategies: subclass-only tables, superclass+subclass tables (most flexible/common), or single table + discriminator | §8, Module 12 |
| Aggregation | Promote to an associative entity table, with a further FK as needed | §9, Module 12 §10 |
| N-ary (Ternary+) Relationship | New table with FK to each participant; composite PK of all FKs | §10, Module 9 §7-§9 |

---

## 13. Chapter Summary

- This module is the **mechanical bridge** from the conceptual ER/EER Model (Modules 6-12) to the logical Relational Model (Modules 2-5) — nine explicit, learnable rules, most of which directly formalize patterns already deeply explored in earlier modules.
- **Strong entities** and **weak entities** map almost mechanically (Rules 1-2); **relationships** map according to the Golden Rule and participation constraints already established (Rule 3, Modules 10-11).
- **Attribute-level rules** (multivalued → new table, composite → leaf columns only, derived → usually not stored) directly resolve the attribute-classification work from Module 7.
- **Generalization/Specialization mapping (Rule 7) is the single most consequential design DECISION** in this module — the choice between three strategies (subclass-only tables, superclass+subclass tables, or single table with a discriminator) should be justified by the hierarchy's Disjoint/Overlapping and Total/Partial classification (Module 12, §9), with **Strategy 2 (superclass + subclass tables) generally the most flexible, commonly recommended real-world default**.
- The full University System worked example (§11) demonstrates applying **every rule together** in one realistic design, with every SQL decision explicitly traceable back to a specific conceptual justification — this traceability is the core skill senior interviews test for.

---

## 14. Cheat Sheet

```
RULE 1 — Strong Entity:        → own table, key attr = PRIMARY KEY
RULE 2 — Weak Entity:          → own table, PK = (owner PK + partial key),
                                   FK usually ON DELETE CASCADE
RULE 3 — Relationships:        1:1 → FK + UNIQUE (either side)
                                1:N → FK on "many" side (NOT NULL if total)
                                M:N → junction table, composite PK
RULE 4 — Multivalued Attr:     → new table, FK + value column, composite PK
RULE 5 — Composite Attr:       → only LEAF sub-attributes become columns
RULE 6 — Derived Attr:         → usually NOT stored; compute on read or
                                   use a GENERATED column
RULE 7 — Generalization/
         Specialization:       → 3 strategies:
   (1) Subclass tables only         — best for Disjoint + Total
   (2) Superclass + subclass tables — best for Overlapping/Partial,
                                        MOST FLEXIBLE, common default
   (3) Single table + discriminator — best for few, simple subclasses
RULE 8 — Aggregation:          → promote to associative entity table
RULE 9 — N-ary Relationship:   → new table, FK to EACH participant,
                                   composite PK of all FKs

REMEMBER: every SQL decision should be traceable back to a specific
          ER/EER concept and a specific mapping rule — that traceability
          IS the skill being tested.
```

---

## 15. Exercises

### Beginner

1. State, in one sentence each, the mapping rule for a Strong Entity and a Weak Entity.
2. True/False: A derived attribute is always stored as a physical column.
3. Fill in the blank: A composite attribute maps to relational columns only at its __________ level.

### Intermediate

4. Map the `Vehicle → Car/Truck/Motorcycle` Disjoint-Total hierarchy from Module 12, §14 using Strategy 1 (subclass tables only), and explain why this strategy is appropriate here.
5. Map the same hierarchy using Strategy 2 instead, and discuss what would change if `Vehicle`'s specialization were actually Partial rather than Total.
6. Design the full SQL mapping for a ternary relationship `Books(Passenger, Flight, SeatClass)` with attributes `booking_date` and `fare_paid` (from Module 9's exercises), applying Rule 9.

### Advanced / Conceptual

7. A hierarchy is Overlapping + Partial (e.g., `Person → Student/Employee`, Module 12 §9). Explain specifically why Strategy 1 (subclass tables only, no superclass table) would be a poor choice here, referencing how it would handle a `Person` who is neither a `Student` nor an `Employee`.
8. Walk through the full University System worked example (§11) and, for each table created, name the specific mapping rule (1-9) and the specific earlier module/section that justifies the decision.
9. Design a complete schema for a `Shape → Circle/Rectangle/Triangle` Disjoint+Total hierarchy AND a `Person → Student/Employee` Overlapping+Partial hierarchy in the SAME database, using the most appropriate strategy for each, and explain why you chose differently for the two.

### MCQs

**Q1.** The most flexible generalization/specialization mapping strategy, generally recommended as a default, is:
a) Subclass tables only  b) Superclass + subclass tables  c) Single table with discriminator  d) None of these are ever recommended

**Q2.** A multivalued attribute maps to:
a) A single nullable column  b) A new table with a foreign key to the owner  c) A CHECK constraint  d) It cannot be mapped to a relational schema

**Q3.** For a 1:N relationship with TOTAL participation on the "many" side, the foreign key column should be:
a) Nullable  b) NOT NULL  c) UNIQUE  d) Removed entirely

**Q4.** Rule 1 (Strong Entity Conversion) states that the entity's key attribute becomes:
a) A CHECK constraint  b) The table's PRIMARY KEY  c) A derived column  d) A multivalued attribute

*(Answer key: 1-b, 2-b, 3-b, 4-b)*

### Full Schema Design Exercise

10. Starting completely from scratch, design a full ER/EER model (informally described) AND its complete relational mapping (applying every relevant rule from this module) for a **Car Rental System**: `Customer`s rent `Vehicle`s (which specialize into `Car`/`SUV`/`Van`, Disjoint+Total); each `Rental` (M:N-with-attributes between Customer and Vehicle) tracks `start_date`, `end_date`, and `total_cost`; each `Vehicle` belongs to exactly one `RentalLocation` (1:N, total on Vehicle's side); a `RentalLocation` has multiple `contact_phone_numbers` (multivalued).

### Practice Links

- [LeetCode Database — Top SQL 50 (excellent for practicing queries against exactly this kind of mapped schema)](https://leetcode.com/studyplan/top-sql-50/)
- [dbdiagram.io](https://dbdiagram.io) — practice drawing the relational schema output of your ER-to-relational mappings.
- [DataLemur](https://datalemur.com/questions) — real company interview questions, several involving schema design and mapping judgment calls.

---

**Next: Module 14 — Schema Design Basics (Naming Conventions, Avoiding Redundancy, Future-Proof Schemas, Scalability Considerations).**

> Say "continue" or "next module" to proceed to Module 14.
