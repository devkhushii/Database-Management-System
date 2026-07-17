# DBMS Handbook — Part 2
## Module 12: Enhanced ER (EER) Model

> **Builds on:** Module 6 (ER Model basics), Module 8 (Entity Sets), Module 9 (Relationships), Module 10-11 (Cardinality & Participation).
>
> **Feeds into:** Module 13 (ER-to-Relational Mapping — Generalization/Specialization have three distinct, commonly-tested mapping strategies).

---

## Table of Contents

1. [Why the EER Model Exists](#1-why-the-eer-model-exists)
2. [Generalization](#2-generalization)
3. [Specialization](#3-specialization)
4. [Generalization vs. Specialization — Same Diagram, Opposite Direction of Thought](#4-generalization-vs-specialization--same-diagram-opposite-direction-of-thought)
5. [ISA Hierarchy and Inheritance](#5-isa-hierarchy-and-inheritance)
6. [Disjoint Constraint](#6-disjoint-constraint)
7. [Overlapping Constraint](#7-overlapping-constraint)
8. [Completeness Constraint (Total vs. Partial Specialization)](#8-completeness-constraint-total-vs-partial-specialization)
9. [Combining Disjoint/Overlapping with Completeness — All Four Combinations](#9-combining-disjointoverlapping-with-completeness--all-four-combinations)
10. [Aggregation](#10-aggregation)
11. [Composition](#11-composition)
12. [Aggregation vs. Composition — The Critical Distinction](#12-aggregation-vs-composition--the-critical-distinction)
13. [Categorization (Union Types)](#13-categorization-union-types)
14. [Full Worked Example: Vehicle Hierarchy](#14-full-worked-example-vehicle-hierarchy)
15. [Chapter Summary](#15-chapter-summary)
16. [Cheat Sheet](#16-cheat-sheet)
17. [Exercises](#17-exercises)

---

## 1. Why the EER Model Exists

### The Problem It Solves

The basic ER Model (Module 6) treats every entity as a flat "bag of attributes" — it has no way to express **"this entity is a specialized kind of that entity"** (e.g., "a `Manager` IS-A specific kind of `Employee`, with everything an `Employee` has, PLUS extra properties"). Real-world domains are absolutely full of such hierarchical, object-oriented-style relationships.

The **Enhanced/Extended ER (EER) Model** adds exactly this capability, borrowing concepts directly from **object-oriented programming** (classes, inheritance, subclasses/superclasses) and layering them on top of the basic ER Model from Module 6.

### Real-World Analogy

If basic ER modeling is like describing "things and how they connect," EER modeling is like adding **"family trees" of things** — a `Vehicle` is a general concept; `Car`, `Truck`, and `Motorcycle` are all specific *kinds* of `Vehicle`, each inheriting everything a `Vehicle` has, while adding their own unique traits.

---

## 2. Generalization

### Definition

**Generalization** is the **bottom-up** process of identifying **common attributes/relationships shared by several existing entity types** and abstracting them into a single, more general **superclass** entity.

### Example

```
STARTING POINT (before generalization):
  Employee(emp_id, name, salary)
  Customer(cust_id, name, loyalty_points)

REALIZATION: both share "name" — and conceptually, both are "Persons"
             the business deals with.

AFTER GENERALIZATION:
                    ┌──────────┐
                    │  Person  │   ← new SUPERCLASS (generalized)
                    │ person_id│
                    │  name    │
                    └────┬─────┘
                         │ ISA
              ┌──────────┴──────────┐
              ▼                     ▼
       ┌─────────────┐       ┌─────────────┐
       │  Employee   │       │  Customer   │  ← SUBCLASSES
       │  (salary)   │       │(loyalty_pts)│
       └─────────────┘       └─────────────┘
```

### Why It's Called "Bottom-Up"

You start with the **specific** entities already defined (`Employee`, `Customer`) — the "bottom" of the hierarchy — and work **upward**, inferring/creating the more abstract superclass (`Person`) that didn't originally exist as a separate concept.

### Interview Importance

**High** — usually tested together with specialization as a compare/contrast pair (§4).

---

## 3. Specialization

### Definition

**Specialization** is the **top-down** process of taking an **existing entity type** and dividing it into **more specific subclasses**, each capturing distinct attributes/relationships relevant only to that subset.

### Example

```
STARTING POINT (before specialization):
  Employee(emp_id, name, salary, employee_type)

REALIZATION: different kinds of employees need different extra data —
             Managers need a "department managed"; Engineers need a
             "specialization area"; Salespeople need a "commission_rate".

AFTER SPECIALIZATION:
                    ┌──────────┐
                    │ Employee │   ← existing SUPERCLASS
                    │ emp_id   │
                    │ name     │
                    │ salary   │
                    └────┬─────┘
                         │ ISA
          ┌──────────────┼──────────────┐
          ▼               ▼              ▼
   ┌─────────────┐ ┌─────────────┐ ┌──────────────┐
   │   Manager   │ │  Engineer   │ │  Salesperson │  ← SUBCLASSES
   │(dept_managed)│ │(spec_area)  │ │(commission)  │
   └─────────────┘ └─────────────┘ └──────────────┘
```

### Why It's Called "Top-Down"

You start with the **general** entity already defined (`Employee`) — the "top" — and work **downward**, carving out more specific subclasses.

### Interview Importance

**High** — this is the far more common *practical* starting point in real design work (you usually already have a general entity and later realize it needs subtypes), so specialization is often the more heavily discussed direction in interviews.

---

## 4. Generalization vs. Specialization — Same Diagram, Opposite Direction of Thought

### The Key Insight

> **Generalization and specialization produce the exact same final ISA hierarchy diagram** — the only difference is **which direction the designer's thought process traveled** to arrive at it: bottom-up (spotting commonality among existing specifics → generalization) or top-down (splitting an existing general concept into specifics → specialization).

| | Generalization | Specialization |
|---|---|---|
| Direction | Bottom-up | Top-down |
| Starting point | Multiple existing specific entities | One existing general entity |
| Process | Merge shared attributes into a new superclass | Split into new subclasses with distinct attributes |
| Resulting diagram | Identical ISA hierarchy either way | Identical ISA hierarchy either way |

### Interview Importance

**High** — precisely explaining "same result, opposite direction of reasoning" is a very common, specific interview/exam question, and confidently stating this equivalence is a strong, easy-to-earn point.

---

## 5. ISA Hierarchy and Inheritance

### Definition

The relationship between a superclass and its subclasses is called an **ISA relationship** (read as "**is a**") — e.g., "a `Manager` **IS A** `Employee`." The overall structure connecting a superclass to its subclasses is called an **ISA hierarchy**.

### Inheritance

Just like in object-oriented programming, subclasses **inherit** all attributes, relationships, and key attributes from their superclass, in addition to having their own specific ones.

```
Employee (superclass)
   attributes: emp_id (key), name, salary
   relationships: WorksIn → Department

Manager (subclass, ISA Employee)
   INHERITS: emp_id, name, salary, WorksIn → Department  (automatically!)
   OWN attributes: dept_managed
```

### Key Attribute Inheritance Rule

> **A subclass NEVER gets its own separate primary key** — it always inherits the **exact same** primary key attribute from its superclass. `Manager`'s primary key is `emp_id`, inherited unchanged from `Employee` — never a new, independent `manager_id`.

### Notation (Chen's EER Extension)

```
                    ┌──────────┐
                    │ Employee │
                    └────┬─────┘
                         │
                        (○ or ▽ triangle — "ISA" symbol,
                         sometimes drawn as a circle with
                         "ISA" written inside, or a triangle
                         pointing toward the superclass)
              ┌──────────┴──────────┐
              ▼                     ▼
       ┌─────────────┐       ┌─────────────┐
       │   Manager   │       │  Engineer   │
       └─────────────┘       └─────────────┘
```

### Interview Importance

**Extremely High** — the "subclass inherits the SAME primary key, never a new one" rule is one of the most frequently tested, easy-to-get-wrong specifics of EER modeling, and directly determines Module 13's mapping strategies.

---

## 6. Disjoint Constraint

### Definition

A **disjoint constraint** specifies that an entity instance belonging to the superclass can be a member of **at most ONE** subclass — the subclasses do not overlap.

### Example

```
Employee (superclass)
   ├── Manager      (disjoint: an employee is EITHER a Manager...)
   └── Engineer     (...OR an Engineer, never both simultaneously)
```

### Notation

Chen's notation marks this with a **"d"** inside the ISA circle/triangle.

```
                    ┌──────────┐
                    │ Employee │
                    └────┬─────┘
                        (d)
              ┌──────────┴──────────┐
              ▼                     ▼
       ┌─────────────┐       ┌─────────────┐
       │   Manager   │       │  Engineer   │
       └─────────────┘       └─────────────┘
```

### Interview Importance

**High** — always tested together with overlapping (§7) as a direct comparison pair.

---

## 7. Overlapping Constraint

### Definition

An **overlapping constraint** specifies that an entity instance **CAN belong to more than one subclass simultaneously**.

### Example

```
Person (superclass)
   ├── Student    (overlapping: a person CAN be both a Student...)
   └── Employee   (...AND an Employee at the same time — e.g., a TA/graduate assistant)
```

### Notation

Chen's notation marks this with an **"o"** inside the ISA circle/triangle.

```
                    ┌──────────┐
                    │  Person  │
                    └────┬─────┘
                        (o)
              ┌──────────┴──────────┐
              ▼                     ▼
       ┌─────────────┐       ┌─────────────┐
       │   Student   │       │  Employee   │
       └─────────────┘       └─────────────┘
```

### Interview Importance

**High** — the classic "Student who is also an Employee (TA)" example is extremely commonly used in academic/GATE exams specifically to illustrate overlapping.

---

## 8. Completeness Constraint (Total vs. Partial Specialization)

### Definition

The **completeness constraint** specifies whether **every instance of the superclass MUST belong to at least one subclass** (Total Specialization), or whether **some superclass instances can exist without belonging to any subclass at all** (Partial Specialization).

> ⚠️ **Terminology Alert:** This is a **different, independent axis** from disjoint/overlapping (§6, §7) — disjoint/overlapping asks *"can an instance belong to MULTIPLE subclasses?"*, while completeness asks *"must an instance belong to AT LEAST ONE subclass?"* These two questions are answered **independently**, exactly analogous to how cardinality and participation (Modules 10-11) were independent axes for ordinary relationships — the EER model reuses that same conceptual structure for ISA hierarchies.

### Total Specialization

**Every** superclass instance **must** belong to at least one subclass.

```
Vehicle (superclass)
   ├── Car
   └── Truck
   (TOTAL: every Vehicle in the system MUST be either a Car or a Truck —
    there's no such thing as a generic "Vehicle" with no more specific type)
```

### Partial Specialization

**Some** superclass instances **may** exist without belonging to any subclass.

```
Employee (superclass)
   ├── Manager
   └── Engineer
   (PARTIAL: an employee could be neither a Manager nor an Engineer —
    e.g., an administrative assistant, HR staff, etc.)
```

### Notation

Total specialization is shown with a **double line** connecting the superclass to the ISA symbol (echoing the exact same double-line-for-total convention from Module 11's participation constraints!); partial specialization uses a **single line**.

### Interview Importance

**High** — recognizing this notation directly parallels Module 11's total/partial participation notation is a strong signal of connected, structural understanding.

---

## 9. Combining Disjoint/Overlapping with Completeness — All Four Combinations

Since disjoint/overlapping and total/partial are **independent axes** (§8), any ISA hierarchy falls into exactly one of **four combinations**:

| Combination | Meaning | Example |
|---|---|---|
| **Disjoint + Total** | Every instance belongs to exactly ONE subclass, no exceptions | `Vehicle` → `Car` XOR `Truck` (every vehicle is exactly one type) |
| **Disjoint + Partial** | Instances belong to AT MOST one subclass, but some belong to none | `Employee` → `Manager` XOR `Engineer` (some employees are neither) |
| **Overlapping + Total** | Every instance belongs to at least one (possibly more) subclass | `Account` → `SavingsAccount` and/or `CheckingAccount` (every bank account must be at least one of these, and could be both, e.g., a combined product) |
| **Overlapping + Partial** | Instances can belong to multiple subclasses, or none at all | `Person` → `Student` and/or `Employee` (a Person could be neither, one, or both) |

### Full Notation Combining Both Axes

```
Disjoint + Total:           Overlapping + Partial:
┌──────────┐                ┌──────────┐
│  Vehicle │                │  Person  │
└────╦═════┘  (double =     └────┬─────┘  (single line =
     (d)       total)             (o)       partial)
┌────┴────┐                ┌──────┴──────┐
▼         ▼                ▼             ▼
Car     Truck            Student       Employee
```

### Interview Importance

**Extremely High.** Being asked to correctly classify a given hierarchy along BOTH axes simultaneously (e.g., "is this Disjoint-Total, Disjoint-Partial, Overlapping-Total, or Overlapping-Partial?") is one of the single most common EER-specific exam and interview question formats.

---

## 10. Aggregation

### Definition

**Aggregation** is an EER construct that allows a **relationship itself to be treated as a higher-level entity**, so that it can participate in **further relationships** with other entities — solving a structural limitation of the basic ER model, where a relationship (the diamond) normally cannot directly connect to another relationship.

### The Problem It Solves

```
❌ ILLEGAL in basic ER: a relationship diamond cannot directly connect
   to ANOTHER relationship diamond.

Example scenario: "An Employee Works On a Project, using certain Equipment
   assigned specifically to that Employee-Project combination."

   Employee ──Works On── Project        (this is a relationship)
                              │
                    "Uses" ── Equipment  (can this connect directly to
                                          the Works-On relationship itself?
                                          NOT in basic ER!)
```

### The Aggregation Solution

Aggregation lets you draw a **boundary box around the entire `Works On` relationship** (including its two participating entities, `Employee` and `Project`), treating that whole boundary as **one combined, higher-level conceptual unit** — which *can* then participate in a further relationship (`Uses`) with `Equipment`.

```
        ┌─────────────────────────────────────┐
        │   ┌──────────┐         ┌──────────┐  │
        │   │ Employee │─Works On─│ Project │  │   ← AGGREGATED
        │   └──────────┘         └──────────┘  │      (whole boundary
        └──────────────────┬──────────────────┘       treated as one unit)
                            │
                          Uses
                            │
                            ▼
                     ┌───────────┐
                     │ Equipment │
                     └───────────┘
```

### Why Not Just Use an Associative Entity (Module 6, §9) Instead?

> ⚠️ **Important, frequently-debated nuance:** In practice, most modern designers **skip formal "aggregation" notation entirely** and instead simply promote `Works On` into a full **associative entity** (Module 6, §9) — e.g., an `Assignment` entity with its own surrogate key — which can then trivially have an ordinary relationship to `Equipment`, without needing any special aggregation notation at all. **Aggregation is a valid, textbook-recognized EER concept, but many practicing engineers consider the associative-entity approach simpler and just as expressive** — it's worth knowing aggregation exists and what problem it solves (for exams/interviews that ask about it directly), while recognizing that real-world modeling very often reaches for the associative-entity solution instead.

### Interview Importance

**Medium** — important to recognize conceptually and for GATE-style exams, but be ready to also mention the associative-entity alternative as the more common real-world practice.

---

## 11. Composition

### Definition

**Composition** is a **stronger form of aggregation** (borrowed directly from object-oriented/UML terminology) where the "part" entities **cannot exist independently of the "whole"** — if the whole is destroyed, all its parts are destroyed too. This is conceptually very close to the **weak entity** pattern already covered extensively in Module 8.

### Example

```
Car (whole) ── composed of ── Engine (part)

If the Car is scrapped/deleted, its specific Engine instance
(as "this car's engine") ceases to be meaningful too —
strong existence dependency, just like a weak entity.
```

### Composition vs. Weak Entity — Are They the Same Thing?

> **Honest, nuanced answer:** Composition (from UML/OO terminology) and the **weak entity** pattern (Module 8, native to ER modeling) describe essentially the **same real-world existence-dependency idea**, but come from different modeling traditions (UML class diagrams vs. classic ER/EER diagrams). Different courses and textbooks emphasize one term or the other; recognizing that they express the same underlying dependency concept is more valuable than treating them as two unrelated things to separately memorize.

### Interview Importance

**Medium** — mostly valuable for recognizing the UML-terminology overlap with concepts you already deeply understand from Module 8.

---

## 12. Aggregation vs. Composition — The Critical Distinction

This distinction is actually **most famous from UML/OOP**, but appears in EER-adjacent discussions too — worth knowing precisely, since it's frequently asked as a "compare and contrast" question.

| | Aggregation | Composition |
|---|---|---|
| Relationship strength | Weaker — "has-a," parts CAN exist independently | Stronger — "owns," parts CANNOT exist independently |
| Existence dependency | No | Yes (mirrors weak entity, Module 8) |
| Example | `Department` "has" `Employee`s — if the Department is dissolved, the Employees still exist (can be reassigned) | `Car` "has" `Engine` — if the Car is scrapped, that specific Engine instance (as part of THAT car) ceases to be meaningful in that context |
| ER Equivalent | The EER "Aggregation" construct (§10) — a relationship treated as an entity | Very close to a **Weak Entity** (Module 8) |

### Interview Importance

**Medium-High**, more commonly asked in the context of object-oriented design interviews than pure DBMS interviews, but worth having crisp and ready given the significant conceptual overlap with weak entities.

---

## 13. Categorization (Union Types)

### Definition

**Categorization** (also called a **Union Type**) models a subclass/category whose members are drawn from the **union of several DIFFERENT superclasses** — unlike ordinary specialization, where a subclass draws from just **one** superclass.

### Why It's Different From Ordinary Specialization

In ordinary Generalization/Specialization (§2-§3), a subclass has exactly **one** superclass (e.g., `Manager` ISA `Employee` — one parent). In **Categorization**, a single subclass can draw its members from **multiple, different superclass types simultaneously**.

### Example

```
Owner (Categorization / Union subclass)
   can be EITHER a:
     Person   (an individual who owns a vehicle), OR
     Bank     (a bank that holds a lien/loan ownership on a vehicle), OR
     Company  (a corporation that owns a fleet vehicle)

┌──────────┐    ┌──────────┐    ┌──────────┐
│  Person  │    │   Bank   │    │ Company  │   ← THREE DIFFERENT superclasses
└────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │
     └───────────────┼───────────────┘
                      ▼
                (∪  — union symbol)
                      │
                      ▼
               ┌──────────┐
               │  Owner   │   ← Categorization: draws from a UNION of
               └──────────┘      Person, Bank, OR Company — NOT from
                                  just a single superclass
```

### Key Difference From Overlapping Specialization

> ⚠️ Don't confuse Categorization with **Overlapping** specialization (§7): overlapping means one instance can belong to multiple subclasses **of the same superclass**; categorization means one subclass draws its members from **multiple, genuinely different superclasses** (`Person`, `Bank`, `Company` are unrelated entity types, not siblings under one common parent).

### Interview Importance

**Low-Medium** — one of the least commonly tested EER concepts in practice (compared to Generalization/Specialization, Disjoint/Overlapping, and Total/Partial), but worth recognizing by name if it comes up, since it's a genuinely distinct structural pattern.

---

## 14. Full Worked Example: Vehicle Hierarchy

Bringing every EER concept in this module together into one complete, exam-ready example:

```
                         ┌──────────┐
                         │ Vehicle  │   emp_id... err, vehicle_id (PK),
                         │          │   manufacturer, model_year
                         └────╦═════┘
                              (d)         ← DISJOINT
                              ║            ← DOUBLE line = TOTAL specialization
              ┌───────────────┼───────────────┐
              ▼                ▼                ▼
       ┌─────────────┐ ┌─────────────┐  ┌──────────────┐
       │     Car      │ │   Truck     │  │  Motorcycle  │  ← SUBCLASSES,
       │(num_doors)   │ │(cargo_cap)  │  │(engine_cc)    │     each inherits
       └─────────────┘ └─────────────┘  └──────────────┘     vehicle_id, etc.
```

**Full classification:** This is a **Disjoint, Total** specialization — every `Vehicle` must be exactly one of `Car`, `Truck`, or `Motorcycle` (never more than one, never none).

### Mapping Preview (Full Rules in Module 13)

```sql
-- One of THREE possible mapping strategies (fully explained in Module 13):
CREATE TABLE Vehicle (
    vehicle_id    INT PRIMARY KEY,
    manufacturer  VARCHAR(50),
    model_year    INT
);
CREATE TABLE Car (
    vehicle_id  INT PRIMARY KEY,      -- SAME key as Vehicle (inherited, §5's rule!)
    num_doors   INT,
    FOREIGN KEY (vehicle_id) REFERENCES Vehicle(vehicle_id)
);
-- (Truck, Motorcycle tables follow the identical pattern)
```

---

## 15. Chapter Summary

- The **EER Model** extends the basic ER Model (Module 6) with object-oriented-style concepts — **Generalization** (bottom-up: merging specifics into a superclass) and **Specialization** (top-down: splitting a general entity into subclasses) produce the **identical** resulting ISA hierarchy diagram, differing only in the designer's direction of reasoning.
- The **ISA hierarchy** models inheritance: subclasses inherit ALL attributes/relationships from their superclass, and critically, **always share the exact same primary key** — never generating an independent new key.
- **Disjoint** (instance belongs to at most one subclass) vs. **Overlapping** (instance can belong to multiple subclasses) is one independent axis; **Total** (every superclass instance must belong to some subclass) vs. **Partial** (some may belong to none) is a **second, independent** axis — directly paralleling Module 11's cardinality/participation independence. Together they form **four possible combinations**.
- **Aggregation** lets a relationship be treated as a higher-level entity so it can participate in further relationships — though many real-world designers prefer to simply promote such relationships into full associative entities (Module 6, §9) instead.
- **Composition** is aggregation's stronger cousin (parts cannot exist independently of the whole) and conceptually overlaps heavily with the **weak entity** pattern already mastered in Module 8.
- **Categorization (Union Types)** models a subclass drawing its members from **multiple, genuinely different superclasses simultaneously** — distinct from ordinary overlapping specialization (which involves siblings under one shared parent).

---

## 16. Cheat Sheet

```
GENERALIZATION = bottom-up (specific entities → new shared superclass)
SPECIALIZATION = top-down (general entity → new specific subclasses)
  → SAME resulting diagram either way, just opposite reasoning direction

ISA HIERARCHY / INHERITANCE:
  Subclass inherits ALL attributes + relationships from superclass
  Subclass NEVER gets a new/independent primary key — always shares
    the superclass's key

TWO INDEPENDENT AXES (just like Cardinality vs. Participation, Modules 10-11):
  Axis 1: DISJOINT (at most 1 subclass, marked "d") vs.
          OVERLAPPING (can belong to multiple, marked "o")
  Axis 2: TOTAL (every instance MUST belong to a subclass, double line) vs.
          PARTIAL (some instances belong to none, single line)
  → 4 combinations: Disjoint-Total, Disjoint-Partial,
                     Overlapping-Total, Overlapping-Partial

AGGREGATION  = treats a relationship AS an entity, so it can have
               further relationships (alternative: promote to
               associative entity instead — Module 6 §9)
COMPOSITION  = STRONGER aggregation; parts cannot exist without the
               whole — conceptually overlaps with WEAK ENTITY (Module 8)
CATEGORIZATION (Union Type) = subclass draws members from MULTIPLE,
               DIFFERENT superclasses simultaneously (NOT the same as
               overlapping specialization, which is siblings under ONE parent)
```

---

## 17. Exercises

### Beginner

1. Define generalization and specialization, and explain how their resulting diagrams relate.
2. True/False: A subclass in an ISA hierarchy gets its own independent primary key, separate from its superclass.
3. Fill in the blank: The __________ constraint asks whether an instance can belong to multiple subclasses, while the __________ constraint asks whether every superclass instance must belong to at least one subclass.

### Intermediate

4. Classify the `Vehicle → Car/Truck/Motorcycle` hierarchy from §14 along both the disjoint/overlapping and total/partial axes, and justify your answer.
5. Design an EER diagram for `Account → SavingsAccount / CheckingAccount`, where accounts can be BOTH types simultaneously (a combined product) and every account must be at least one of the two. Name the correct combination from §9.
6. Explain, using the `Person/Student/Employee` example, the difference between overlapping specialization and categorization (union types).

### Advanced / Conceptual

7. A designer wants to model "an Employee Works On a Project using specific Equipment." Explain both the formal Aggregation solution and the more commonly-used associative-entity alternative, and discuss the trade-offs of each.
8. Explain why Composition is conceptually so similar to a Weak Entity (Module 8), using a concrete example (e.g., `Car`–`Engine`) mapped into both notations.
9. Design a full EER diagram for a `Payment` categorization where payments can come from a `CreditCard`, `BankAccount`, or `DigitalWallet` (three unrelated superclasses) — correctly using Categorization/Union Type notation rather than ordinary specialization.

### MCQs

**Q1.** Generalization is best described as:
a) Top-down  b) Bottom-up  c) Neither top-down nor bottom-up  d) Only used in relational databases

**Q2.** In an ISA hierarchy, a subclass's primary key is:
a) Always a new, independent key  b) The same key inherited from the superclass  c) Never defined  d) Always a composite key

**Q3.** The "d" and "o" notations in an EER diagram represent:
a) Total and Partial  b) Disjoint and Overlapping  c) Derived and Original  d) Domain and Ordinal

**Q4.** Categorization (Union Type) differs from ordinary overlapping specialization because:
a) It never has a completeness constraint  b) It draws members from multiple, different superclasses rather than siblings under one shared parent  c) It cannot be mapped to a relational schema  d) It is identical to overlapping specialization

*(Answer key: 1-b, 2-b, 3-b, 4-b)*

### Diagram Exercise

10. Draw a full EER diagram for `Person → Student / Employee` using Overlapping + Partial specialization, and separately, `Shape → Circle / Rectangle / Triangle` using Disjoint + Total specialization, clearly marking the "d"/"o" and single/double line notation on each.

### Practice Links

- *(EER modeling is a conceptual/diagram skill — practice via the exercises above.)*
- [dbdiagram.io](https://dbdiagram.io) — practice drawing hierarchy-style schemas (note: most modern ER tools default to the relational MAPPING of an EER hierarchy rather than drawing the EER diagram itself — useful for seeing where Module 13 is headed).

---

**Next: Module 13 — Converting ER to Relational Model (Strong Entity, Weak Entity, Relationship Mapping, Multivalued/Composite/Derived Attribute Mapping, Generalization/Specialization/Aggregation Mapping — step-by-step rules with full examples).**

> Say "continue" or "next module" to proceed to Module 13.
