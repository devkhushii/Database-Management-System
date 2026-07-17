# DBMS Handbook — Part 2
## Module 7: Attributes

> **Builds on:** Module 6 (ER Model — attributes are one of the core ER building blocks introduced there).
>
> **Feeds into:** Module 13 (ER-to-Relational Mapping — each attribute type has its own specific mapping rule), Normalization (Part 3, especially 1NF's atomicity requirement, first previewed in Module 2, §18).

---

## Table of Contents

1. [What Is an Attribute (ER Perspective)?](#1-what-is-an-attribute-er-perspective)
2. [The Full Attribute Classification Tree](#2-the-full-attribute-classification-tree)
3. [Simple (Atomic) Attribute](#3-simple-atomic-attribute)
4. [Composite Attribute](#4-composite-attribute)
5. [Single-Valued Attribute](#5-single-valued-attribute)
6. [Multi-Valued Attribute](#6-multi-valued-attribute)
7. [Stored Attribute](#7-stored-attribute)
8. [Derived Attribute](#8-derived-attribute)
9. [Optional Attribute](#9-optional-attribute)
10. [Mandatory Attribute](#10-mandatory-attribute)
11. [Key Attribute](#11-key-attribute)
12. [Complex Attribute](#12-complex-attribute)
13. [Full ER Notation Reference Table](#13-full-er-notation-reference-table)
14. [Worked Example: Combining All Attribute Types](#14-worked-example-combining-all-attribute-types)
15. [Chapter Summary](#15-chapter-summary)
16. [Cheat Sheet](#16-cheat-sheet)
17. [Exercises](#17-exercises)

---

## 1. What Is an Attribute (ER Perspective)?

Recall from Module 6, §5 that an **attribute** is a property/characteristic of an entity. In Module 2 (Relational Model), "attribute" simply meant "a column." In ER modeling, attributes are **richer** — they come in several distinct flavors that all eventually get flattened down into simple relational columns during the mapping process (Module 13). Understanding these flavors *now*, at the conceptual stage, is what allows you to make **correct** mapping decisions later, rather than losing important structure.

### Why This Richness Exists

Real-world properties aren't all the same "shape." A person's `name` might naturally split into first/last; a person might have *multiple* phone numbers; a person's `age` might be something we *calculate* rather than store. The ER model gives us vocabulary to capture these nuances **before** they get squashed into flat columns.

---

## 2. The Full Attribute Classification Tree

Attributes can be classified along **four independent dimensions simultaneously** — a single real attribute can be described using one label from *each* dimension at once. This is a critical, often-missed insight: these are not mutually exclusive categories, but independent axes.

```
                          ATTRIBUTE
                              │
     ┌────────────┬───────────┼───────────┬─────────────┐
     ▼            ▼           ▼           ▼             ▼
 STRUCTURE    MULTIPLICITY  ORIGIN    OBLIGATION    IDENTIFICATION
     │            │           │           │             │
 Simple/     Single-valued/ Stored/    Mandatory/    Key Attribute
 Composite    Multi-valued  Derived     Optional      (yes/no)
```

**Example of combining axes:** A person's `date_of_birth` is: **Simple** (structure) + **Single-valued** (multiplicity) + **Stored** (origin) + **Mandatory** (obligation) + **Not a key** (identification) — all five labels apply *simultaneously* to describe this one attribute fully.

---

## 3. Simple (Atomic) Attribute

### Definition

A **simple attribute** (also called an **atomic attribute**) is one that **cannot be meaningfully subdivided further** — it is a single, indivisible unit of data.

### Example

```
Student
  roll_no  → Simple (a single ID number, no natural sub-parts)
  gender   → Simple
```

### Direct Connection to Module 2

This is the **exact same "atomicity" concept** introduced in Module 2, §18 — the ER model's "simple attribute" is precisely what eventually satisfies the **1NF** atomicity requirement (fully explored in Part 3's Normalization module) once mapped to a relational column.

### Interview Importance

**Medium** — mostly important as the baseline contrast against composite attributes (§4).

---

## 4. Composite Attribute

### Definition

A **composite attribute** is one that **can be broken down into smaller, meaningful sub-attributes**, each of which carries independent significance.

### Example

```
Student.name  (Composite)
     │
     ├── first_name
     ├── middle_name
     └── last_name

Student.address  (Composite)
     │
     ├── street
     ├── city
     ├── state
     └── postal_code
```

### ER Notation (Chen's)

Composite attributes are drawn as an oval, with smaller ovals branching off it for each component:

```
                    ┌──────┐
                    │ name │
                    └──┬───┘
              ┌─────────┼─────────┐
              ▼         ▼         ▼
       ┌───────────┐┌──────────┐┌──────────┐
       │first_name ││middle_nm ││last_name │
       └───────────┘└──────────┘└──────────┘
```

### Why It Matters (Design Decision, Not Just Vocabulary)

Whether to model something as composite or simple is a genuine **design decision** with real consequences:

| Decision | When to Choose It |
|---|---|
| Keep as **composite** (split into sub-columns) | You need to **query, sort, or filter** by the sub-parts independently (e.g., "find all customers in city = 'Mumbai'") |
| Treat as **simple** (single column) | The sub-parts are never queried independently, and splitting would add unnecessary complexity (e.g., a `full_name` field for a simple contact list app that never searches by first/last name separately) |

### Mapping to Relational Model (Preview of Module 13)

A composite attribute typically becomes **multiple separate columns** in the relational schema — one per leaf-level sub-attribute (the top-level composite grouping itself, e.g. "name," usually doesn't become a column at all).

```sql
-- name (composite) → flattened into 3 columns:
CREATE TABLE Student (
    first_name  VARCHAR(30),
    middle_name VARCHAR(30),
    last_name   VARCHAR(30)
);
```

### Common Mistakes

- Splitting every possible attribute into composites "just in case" — leads to over-engineered schemas with unnecessary columns for data that's never queried at that granularity.
- The opposite mistake: cramming naturally composite data (like a full address) into one giant string column when the application genuinely needs city/state-level filtering later — a classic, costly redesign trigger.

### Interview Importance

**High.** "Should `name` be one column or split into first/last?" is an extremely common, practical schema-design interview question, and a good answer explicitly invokes the query-pattern reasoning above.

---

## 5. Single-Valued Attribute

### Definition

A **single-valued attribute** holds **exactly one value** for each entity instance.

### Example

```
Student.roll_no  → each student has exactly one roll number
Student.dob      → each student has exactly one date of birth
```

### Interview Importance

**Low** standalone — mostly relevant as the direct contrast to multi-valued attributes (§6).

---

## 6. Multi-Valued Attribute

### Definition

A **multi-valued attribute** can hold **more than one value simultaneously** for a single entity instance.

### Example

```
Student.phone_numbers   → a student might have both a personal and a home number
Employee.skills         → an employee might have multiple skills
Book.authors            → a book might have multiple co-authors
```

### ER Notation (Chen's)

Drawn as a **double-outlined oval**.

```
        ╔═══════════════╗
        ║ phone_numbers ║
        ╚═══════════════╝
```

### Why It's a Modeling Challenge

Multi-valued attributes **cannot be directly stored as a single relational column** without violating atomicity (Module 2, §18) — recall the "non-atomic" bad example from that section:

```
❌ BAD: Student(roll_no, name, phone_numbers)
   101 | Aditi | "9876543210, 9998887776"
```

### Correct Mapping Rule (Preview of Module 13, Rule Covered in Full There)

A multi-valued attribute must be extracted into its **own separate table**, with a foreign key back to the owning entity:

```sql
CREATE TABLE Student (
    roll_no INT PRIMARY KEY,
    name    VARCHAR(50)
);

CREATE TABLE StudentPhone (
    roll_no       INT,
    phone_number  VARCHAR(15),
    PRIMARY KEY (roll_no, phone_number),   -- composite key (Module 4, §7)
    FOREIGN KEY (roll_no) REFERENCES Student(roll_no)
);
```

This is the exact same fix demonstrated in Module 2, §18 — now you understand *why* it's needed from the ER-modeling side, not just the normalization side.

### Advantages of Correctly Identifying Multi-Valued Attributes Early

- Prevents the classic "comma-separated list in a column" anti-pattern from ever being designed in the first place.
- Makes the eventual relational mapping mechanical and predictable (Module 13).

### Common Mistakes

- Storing multi-valued data as a delimited string (`"tag1,tag2,tag3"`) — breaks atomicity, makes querying/indexing individual values very difficult, and is one of the most common real-world schema smells found in legacy systems.
- Confusing a multi-valued attribute with a genuine one-to-many **relationship** to another full entity — if the "many values" have enough independent significance/attributes of their own (e.g., each phone number needs a `type: mobile/home/work` and a `verified: boolean`), it likely deserves to be modeled as its own **weak entity** (Module 6, §8) rather than a bare multi-valued attribute.

### Interview Importance

**Very High.** Multi-valued attribute handling is one of the most frequently tested *practical* schema design questions (e.g., "how would you store a user's multiple email addresses?").

---

## 7. Stored Attribute

### Definition

A **stored attribute** holds a value that is **directly recorded/input** into the database — it isn't computed from other data.

### Example

```
Student.dob   → directly entered when the student registers
```

### Interview Importance

**Low** standalone — its importance comes entirely from being the contrast to derived attributes (§8).

---

## 8. Derived Attribute

### Definition

A **derived attribute** is one whose value can be **calculated/computed from other attributes** (often another stored attribute), rather than being independently stored.

### Example

```
Student.dob   (Stored)
Student.age   (Derived — calculated as: current_date - dob)
```

### ER Notation (Chen's)

Drawn as a **dashed oval**.

```
        ┌ ─ ─ ─ ┐
          age
        └ ─ ─ ─ ┘
```

### The Central Design Question: Should You Actually Store a Derived Attribute?

This is a genuinely important, frequently debated real-world trade-off:

| Approach | Advantages | Disadvantages |
|---|---|---|
| **Don't store it — compute on every query** (e.g., `SELECT DATEDIFF(CURDATE(), dob) AS age`) | Always accurate; no risk of becoming stale/inconsistent with its source | Adds computation cost to every read; can't be indexed directly (though computed/generated columns, below, solve this in modern systems) |
| **Store it, and update it whenever source data changes** | Fast reads (no computation needed); can be indexed directly | Risk of becoming **stale/inconsistent** if the update logic is ever missed or buggy; violates a strict reading of normalization principles (redundant, derivable data) |
| **Modern middle ground: Generated/Computed Columns** (`GENERATED ALWAYS AS`) | DBMS automatically keeps it in sync — best of both worlds | Not supported identically across all RDBMSs; still adds some storage/write overhead |

```sql
-- Modern SQL generated column syntax (e.g., MySQL 8+, PostgreSQL 12+)
ALTER TABLE Student ADD COLUMN age INT
    GENERATED ALWAYS AS (TIMESTAMPDIFF(YEAR, dob, CURDATE())) STORED;
```

### Real-World Examples of Derived Attributes

- `age` from `date_of_birth`
- `total_price` from `quantity × unit_price`
- `full_name` from `first_name + ' ' + last_name`
- Aggregate values like `total_orders_placed` for a customer (often intentionally **denormalized and cached** for performance in high-read systems — a very common real-world trade-off discussed further in Part 3's Normalization/Denormalization module)

### Best Practices

- Prefer **computing on read** by default, for simplicity and guaranteed accuracy.
- Only **store/cache** a derived value when performance profiling genuinely justifies it (e.g., an expensive aggregate queried extremely frequently) — and if you do, ensure a reliable mechanism (triggers, generated columns, or careful application logic) keeps it in sync.

### Interview Importance

**High.** "Would you store `age` or compute it?" and "how do you keep a derived/cached value in sync?" are both very standard, practical interview questions testing real engineering judgment, not just definitions.

---

## 9. Optional Attribute

### Definition

An **optional attribute** is one that **may legitimately be absent (NULL)** for some entity instances — recall this connects directly to Module 2, §17's discussion of NULL semantics.

### Example

```
Employee.middle_name    → not everyone has one
Employee.manager_id     → the CEO has no manager
```

### ER Notation

Optional attributes are sometimes shown with a dashed connecting line to the entity, or explicitly annotated `(O)` for optional in some notational conventions — this varies more between textbooks/tools than most other ER notation, so always check the legend of whatever specific diagram you're reading.

### Interview Importance

**Medium** — mostly discussed together with mandatory attributes (§10) and NULL semantics (Module 2, §17).

---

## 10. Mandatory Attribute

### Definition

A **mandatory attribute** must have a value for **every single entity instance** — it can never be NULL.

### Example

```
Student.roll_no  → mandatory (also a key attribute — every student must have one)
Employee.name    → mandatory (the business always requires a name to be recorded)
```

### Direct Mapping to SQL (Preview of Module 13 / Module 5 connection)

Mandatory attributes map directly to `NOT NULL` constraints (Module 5, §7) when the ER diagram is converted to a relational schema.

### Best Practices

- Decide mandatory vs. optional status **during ER modeling**, based on genuine business requirements — not arbitrarily during implementation. This ensures the eventual `NOT NULL` constraints reflect real business rules, not implementation guesswork.

### Interview Importance

**Medium-High** — mostly valuable as a demonstration that ER-level decisions (mandatory/optional) directly and predictably determine later SQL constraints (`NOT NULL`), reinforcing the design pipeline concept (Module 1, §8; Module 6, §3).

---

## 11. Key Attribute

### Definition

A **key attribute** is an attribute (or attribute combination) that **uniquely identifies each entity instance** within its entity set — this is the ER-level equivalent of the relational model's Primary/Candidate Key (Module 4).

### ER Notation (Chen's)

Key attributes are shown with their name **underlined** inside the oval.

```
        ┌──────────┐
        │ roll_no  │   ← underlined = key attribute
        └──────────┘
```

For a **weak entity's partial key** (Module 6, §8; Module 4, §14), the underline is **dashed** instead of solid, visually distinguishing "fully identifying key" from "partial key, needs the owner's key too."

### Interview Importance

**High** — directly reinforces Module 4 and Module 6, §8's weak entity discussion; frequently tested by asking a student to correctly underline (or dashed-underline) the right attribute(s) on a diagram.

---

## 12. Complex Attribute

### Definition

A **complex attribute** is an attribute that is **both composite AND multi-valued simultaneously** — i.e., it has sub-parts (composite), and there can be more than one instance of that whole composite structure per entity (multi-valued).

### Example

```
Employee.addresses   (Complex: multiple addresses, each itself composite)

  addresses (multi-valued) — an employee can have a home address AND a work address
      │
      └── each individual address (composite) breaks down into:
              ├── street
              ├── city
              ├── state
              └── postal_code
```

```
                    ╔═══════════╗
                    ║ addresses ║   ← double oval = multi-valued
                    ╚═════╦═════╝
                          │
                    ┌─────┴─────┐
                    ▼           (this whole composite structure
              ┌──────────┐       repeats for each address)
              │  street   │
              └──────────┘
              ┌──────────┐
              │   city    │
              └──────────┘
              ┌──────────┐
              │  state    │
              └──────────┘
```

### Mapping to Relational Model (Preview)

Just like a plain multi-valued attribute (§6), a complex attribute requires its **own separate table** — but this new table itself has multiple columns (one per composite sub-part), rather than a single value column:

```sql
CREATE TABLE Employee (
    emp_id INT PRIMARY KEY,
    name   VARCHAR(50)
);

CREATE TABLE EmployeeAddress (
    emp_id       INT,
    address_type VARCHAR(10),   -- e.g., 'Home', 'Work' -- discriminates multiple addresses
    street       VARCHAR(100),
    city         VARCHAR(50),
    state        VARCHAR(50),
    postal_code  VARCHAR(10),
    PRIMARY KEY (emp_id, address_type),
    FOREIGN KEY (emp_id) REFERENCES Employee(emp_id)
);
```

### Interview Importance

**Medium** — less commonly named explicitly by this term in interviews, but the underlying *pattern* (multiple composite sub-records per entity) is extremely common in real schema design (e.g., multiple shipping addresses per customer in an e-commerce system).

---

## 13. Full ER Notation Reference Table

A single, consolidated, revision-ready reference for every attribute type's Chen's-notation symbol:

| Attribute Type | Chen's Notation Symbol |
|---|---|
| Simple / Atomic | Single oval |
| Composite | Oval with branching sub-ovals |
| Single-valued | Single oval (default; no special marking) |
| Multi-valued | **Double**-outlined oval |
| Stored | Single oval (default; no special marking) |
| Derived | **Dashed**-outlined oval |
| Key Attribute | Oval with **underlined** name |
| Partial Key (weak entity) | Oval with **dashed-underlined** name |
| Complex | Double oval (multi-valued) containing branching sub-ovals (composite) |

---

## 14. Worked Example: Combining All Attribute Types

Let's fully specify the `Employee` entity from earlier modules, correctly labeling every attribute using everything learned in this module:

```
Employee
├── emp_id           → Simple, Single-valued, Stored, Mandatory, KEY attribute
├── name              → Composite (first_name, last_name), Single-valued, Stored, Mandatory
├── dob                → Simple, Single-valued, Stored, Mandatory
├── age                → Simple, Single-valued, DERIVED (from dob), Mandatory (always computable)
├── phone_numbers     → Simple, MULTI-VALUED, Stored, Optional
├── addresses          → COMPLEX (composite + multi-valued), Stored, Optional
├── manager_id         → Simple, Single-valued, Stored, OPTIONAL (CEO has none)
```

### Full ASCII ER Diagram

```
                              ┌──────┐
                              │ name │  (composite)
                              └──┬───┘
                        ┌─────────┴─────────┐
                        ▼                   ▼
                 ┌────────────┐      ┌────────────┐
                 │ first_name │      │ last_name  │
                 └────────────┘      └────────────┘
                        │
┌ ─ ─ ─ ┐        ┌──────┴───────┐        ╔══════════════╗
  age    ── ─ ──│   Employee    │──────║ phone_numbers ║
└ ─ ─ ─ ┘        └──────┬───────┘        ╚══════════════╝
                        │
             ┌──────────┼───────────┐
             ▼          ▼           ▼
       ┌──────────┐┌─────────┐╔═══════════╗
       │ emp_id   ││   dob   │║ addresses ║ (complex:
       │(underlined)         │╚═════╦═════╝  multi-valued
       └──────────┘└─────────┘      │        + composite)
                                     ▼
                          (street, city, state, postal_code)
```

---

## 15. Chapter Summary

- Attributes are classified along **four independent axes simultaneously**: **Structure** (Simple vs. Composite), **Multiplicity** (Single-valued vs. Multi-valued), **Origin** (Stored vs. Derived), and **Obligation** (Mandatory vs. Optional) — plus a separate **Identification** label (Key Attribute or not).
- **Composite attributes** split into meaningful sub-parts (e.g., `name` → first/middle/last) — the decision to split depends on whether sub-parts need independent querying.
- **Multi-valued attributes** (e.g., `phone_numbers`) can never be stored as a single relational column — they require extraction into a separate table with a foreign key back to the owner, directly connecting to Module 2, §18's atomicity discussion.
- **Derived attributes** (e.g., `age` from `dob`) raise a genuine engineering trade-off: compute-on-read (simple, always accurate) vs. store-and-sync (fast, but risk of staleness) vs. modern generated/computed columns (best of both, where supported).
- **Mandatory/Optional** attributes map directly and predictably to `NOT NULL` constraints (Module 5) once converted to a relational schema.
- **Key attributes** are underlined in Chen's notation (dashed-underline for a weak entity's partial key, connecting directly to Module 4, §14 and Module 6, §8).
- **Complex attributes** combine composite + multi-valued simultaneously, and map to their own table with multiple columns (not just a single value column).

---

## 16. Cheat Sheet

```
FOUR INDEPENDENT AXES (a single attribute gets ONE label from EACH):
  Structure:      Simple  |  Composite
  Multiplicity:   Single-valued  |  Multi-valued
  Origin:         Stored  |  Derived
  Obligation:     Mandatory  |  Optional
  (+ separately)  Key Attribute?  Yes / No

CHEN'S NOTATION QUICK REFERENCE:
  Single oval             → Simple / Single-valued / Stored (default, no marking)
  Oval + branching ovals  → Composite
  DOUBLE oval              → Multi-valued
  DASHED oval               → Derived
  UNDERLINED name           → Key Attribute
  DASHED-UNDERLINED name     → Partial Key (weak entity, Module 6 §8 / Module 4 §14)
  Double oval + branching   → Complex (multi-valued AND composite)

MAPPING PREVIEW (fully covered in Module 13):
  Simple/Composite/Single-valued/Stored → becomes ordinary column(s)
  Multi-valued / Complex                → becomes a SEPARATE table + foreign key
  Derived                                → often NOT stored at all (computed on read),
                                            or implemented as a GENERATED column
  Mandatory / Optional                   → becomes NOT NULL / nullable
```

---

## 17. Exercises

### Beginner

1. Classify `Student.dob` along all four attribute axes (structure, multiplicity, origin, obligation).
2. True/False: A multi-valued attribute can be safely stored as a comma-separated string in a single column.
3. Fill in the blank: An attribute whose value is calculated from other attributes, such as `age` from `dob`, is called a __________ attribute.

### Intermediate

4. `Customer.shipping_addresses` — a customer may have multiple shipping addresses, each with street/city/state/postal_code. Classify this attribute completely, and sketch its ER notation.
5. Explain, using the storage vs. computation trade-off from §8, when you would choose to physically store a derived attribute rather than compute it on every query.
6. A schema currently stores `full_name` as a single column. The business now needs to sort customers by last name. Explain what ER-level change is needed and why.

### Advanced / Conceptual

7. Design the full attribute list (with complete classification along all axes) for a `Product` entity in an e-commerce system, including at least one composite, one multi-valued, one derived, and one optional attribute.
8. A multi-valued attribute `Employee.skills` is being considered for promotion into a full weak or associative entity instead. What additional business requirement (give a concrete example) would justify that promotion? (Hint: revisit Module 6, §8 and §9.)
9. Explain why "mandatory vs. optional" should ideally be decided during ER modeling rather than left for a developer to guess while writing `CREATE TABLE` statements.

### MCQs

**Q1.** In Chen's notation, a multi-valued attribute is shown as a:
a) Dashed oval  b) Double oval  c) Single oval with underline  d) Diamond

**Q2.** Which of the following is a derived attribute?
a) date_of_birth  b) age  c) roll_no  d) email

**Q3.** A composite attribute is one that:
a) Can take multiple values  b) Can be broken into meaningful sub-parts  c) Is always a key  d) Is always optional

**Q4.** Storing a comma-separated list of phone numbers in a single column violates:
a) Referential Integrity  b) Entity Integrity  c) Atomicity (1NF)  d) Domain Constraint

*(Answer key: 1-b, 2-b, 3-b, 4-c)*

### Diagram Exercise

10. Draw a complete Chen's-notation attribute diagram for a `Book` entity with: `isbn` (key), `title` (simple), `authors` (multi-valued), `price` (simple), `discounted_price` (derived from price and a discount rule), and `publication_address` (composite: publisher_name, city, country).

### Practice Links

- *(Attribute classification is a conceptual/diagram skill — practice via the exercises above.)*
- [LeetCode Database — problems involving derived/computed values (e.g., calculating age, totals)](https://leetcode.com/studyplan/top-sql-50/)

---

**Next: Module 8 — Entity Sets (Strong Entity, Weak Entity, Owner Entity, Identifying Relationship, Partial Key — full deep dive with real-system examples).**

> Say "continue" or "next module" to proceed to Module 8.
