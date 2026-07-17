# DBMS Handbook — Part 2
## Module 8: Entity Sets

> **Builds on:** Module 6, §7-§8 (Strong/Weak Entity were first introduced there); Module 4, §14 (Partial Key); Module 7 (Attributes, especially Key Attributes).
>
> **Purpose of this module:** Module 6 introduced strong and weak entities as part of the broader ER Model overview. This module goes **deeper** — formalizing the *Owner Entity* and *Identifying Relationship* concepts precisely, working through multiple real-system examples, and covering edge cases that are frequently tested but often missed.
>
> **Feeds into:** Module 13 (ER-to-Relational Mapping — weak entity conversion has its own dedicated, precise rule).

---

## Table of Contents

1. [Recap: Strong vs. Weak Entities](#1-recap-strong-vs-weak-entities)
2. [Owner Entity (Formal Definition)](#2-owner-entity-formal-definition)
3. [Identifying Relationship (Formal Definition)](#3-identifying-relationship-formal-definition)
4. [Partial Key — Full Deep Dive](#4-partial-key--full-deep-dive)
5. [The Complete Weak Entity Identification Formula](#5-the-complete-weak-entity-identification-formula)
6. [Real-System Example 1: HR System — Dependent](#6-real-system-example-1-hr-system--dependent)
7. [Real-System Example 2: Hotel System — Room](#7-real-system-example-2-hotel-system--room)
8. [Real-System Example 3: E-Commerce — OrderLineItem](#8-real-system-example-3-e-commerce--orderlineitem)
9. [Real-System Example 4: Banking — Transaction](#9-real-system-example-4-banking--transaction)
10. [Chains of Weak Entities (Edge Case)](#10-chains-of-weak-entities-edge-case)
11. [How to Decide: Strong, Weak, or Just an Attribute?](#11-how-to-decide-strong-weak-or-just-an-attribute)
12. [Common Mistakes in Identifying Weak Entities](#12-common-mistakes-in-identifying-weak-entities)
13. [Chapter Summary](#13-chapter-summary)
14. [Cheat Sheet](#14-cheat-sheet)
15. [Exercises](#15-exercises)

---

## 1. Recap: Strong vs. Weak Entities

From Module 6:

| | Strong Entity | Weak Entity |
|---|---|---|
| Has own full primary key? | Yes | No — only a **partial key** |
| Can exist independently? | Yes | **No** — depends on an owner entity |
| Notation | Single rectangle | Double rectangle |
| Example | `Student`, `Course`, `Employee` | `Dependent`, `OrderItem`, `Room` |

This module treats the weak entity's supporting cast — **Owner Entity**, **Identifying Relationship**, and **Partial Key** — as first-class concepts worth their own precise definitions, since interview questions frequently probe each piece individually.

---

## 2. Owner Entity (Formal Definition)

### Definition

The **owner entity** (also called the **identifying entity** or **dominant entity**) is the **strong entity** that a weak entity depends on for its existence and identification. Every weak entity has exactly one (type of) owner entity.

### Key Properties

| Property | Explanation |
|---|---|
| Must be a strong entity | An owner entity always has its own independent primary key |
| One owner entity **type**, many owner entity **instances** | E.g., `Employee` is the owner entity *type* for `Dependent`; each individual employee (`emp_id = 501`, `emp_id = 502`, ...) is one *instance* acting as an owner |
| Provides the "outer" part of the weak entity's full identification | Combined with the weak entity's partial key (§4) to form full identification (§5) |

### Why the Term Matters

Precisely naming "owner entity" (rather than just informally saying "the parent table") signals ER-modeling fluency in interviews and exams, and cleanly connects the conceptual (ER) layer to its eventual relational implementation (a `FOREIGN KEY NOT NULL` reference).

### Interview Importance

**Medium-High** — usually tested as part of a broader weak-entity design question rather than in complete isolation.

---

## 3. Identifying Relationship (Formal Definition)

### Definition

An **identifying relationship** is the specific relationship (Module 9 covers relationships generally) that **connects a weak entity to its owner entity**, and through which the weak entity draws part of its identity.

### Key Properties

| Property | Explanation |
|---|---|
| Notation | **Double-outlined diamond** in Chen's notation (Module 6, §8) |
| Participation | The weak entity **must** have **total participation** (Module 11 covers this fully) in its identifying relationship — a `Dependent` cannot exist in the database without being linked to some `Employee` |
| Cardinality | Almost always **one owner instance to many weak entity instances** (1:N) — e.g., one `Employee` can have many `Dependent`s, but each `Dependent` belongs to exactly one `Employee` (previewing Module 10's cardinality notation) |

### Why "Identifying" Specifically?

The word "identifying" is deliberate and precise: this isn't just *any* relationship the weak entity happens to participate in — it is **the specific one that contributes to resolving its identity**. A weak entity could, in principle, have *other*, non-identifying relationships to different entities too (e.g., a `Dependent` might also have a relationship to an `InsurancePlan`) — only the relationship to its owner is the *identifying* one.

### Interview Importance

**Medium-High** — precise vocabulary here differentiates strong candidates in ER-modeling discussions.

---

## 4. Partial Key — Full Deep Dive

*(First introduced in Module 4, §14 as part of the key taxonomy — this section gives it the full treatment promised there.)*

### Definition (Restated, Precisely)

A **partial key** (or **discriminator**) is the attribute (or attribute combination) of a weak entity that **distinguishes it from other weak entity instances belonging to the SAME owner instance** — but is **not, by itself, globally unique** across the entire weak entity set.

### The Critical Scoping Insight

> **A partial key only needs to be unique *within* one owner's "family" of weak entities — not across the whole system.** This is the single most important, and most frequently misunderstood, fact about partial keys.

```
Employee 501 (Ramesh)          Employee 502 (Suresh)
  Dependents:                    Dependents:
    "Rahul" (Son)                  "Rahul" (Son)     ← SAME partial key value
    "Priya" (Daughter)             "Anjali" (Daughter)  "Rahul" is fine here!
                                                          Different owners.
```

Both `Employee 501` and `Employee 502` can have a dependent named "Rahul" — the partial key `dependent_name = "Rahul"` is **not** globally unique, and that's completely acceptable, because uniqueness is only required **within each owner's scope**.

### Full Identification Requires BOTH Parts

```
Full Identifying Key = (Owner's Primary Key) + (Weak Entity's Partial Key)
                      = (emp_id, dependent_name)
                      = (501, "Rahul")   -- globally unique combination
                      = (502, "Rahul")   -- ALSO globally unique combination
                                             (different because emp_id differs)
```

### Notation

Dashed underline in Chen's notation (as established in Module 6, §8) — visually distinct from a solid underline (full key, Module 7, §11), signaling "this is only PART of the answer."

### Interview Importance

**High** — the scoping insight above ("unique only within the owner, not globally") is a very common, specific point interviewers probe to check genuine understanding versus surface memorization.

---

## 5. The Complete Weak Entity Identification Formula

This section formalizes the pattern that has been building since Module 4, §14 and Module 6, §8 into one clean, memorable formula:

```
┌─────────────────────────────────────────────────────────────────┐
│  Full identifying key of a weak entity =                          │
│      (Primary Key of Owner Entity)  +  (Partial Key of Weak Entity)│
└─────────────────────────────────────────────────────────────────┘
```

This formula directly determines the **composite primary key** (Module 4, §7) that the weak entity's table will receive once mapped to the relational model — a direct, mechanical preview of one of Module 13's specific mapping rules.

```sql
-- Direct mechanical translation of the formula above:
CREATE TABLE Dependent (
    emp_id          INT,               -- from owner's PK
    dependent_name  VARCHAR(50),       -- partial key
    relationship_type VARCHAR(20),
    PRIMARY KEY (emp_id, dependent_name),      -- the FULL identifying key
    FOREIGN KEY (emp_id) REFERENCES Employee(emp_id) ON DELETE CASCADE
);
```

> 💡 Notice the `ON DELETE CASCADE` — this is not arbitrary. It's the direct relational consequence of the weak entity's **total participation** requirement (§3): if the owning `Employee` is deleted, their `Dependent` rows have no valid reason to continue existing, so cascading the delete is the *correct*, meaning-preserving choice (revisited formally in Module 11 and Module 13).

---

## 6. Real-System Example 1: HR System — Dependent

```
Owner Entity:            Employee (Strong)
                          emp_id (PK), name, hire_date

Weak Entity:              Dependent
                          dependent_name (Partial Key)
                          relationship_type, date_of_birth

Identifying Relationship: "Has" (Employee 1 ──── N Dependent)
                          Total participation of Dependent (mandatory)
```

**Why weak?** A dependent has no company-wide, globally meaningful identity of their own — HR only ever cares about "Employee 501's dependent Rahul," never "Rahul the dependent" as a standalone concept.

---

## 7. Real-System Example 2: Hotel System — Room

```
Owner Entity:            Hotel (Strong)
                          hotel_id (PK), name, city

Weak Entity:              Room
                          room_number (Partial Key)
                          room_type, floor, capacity

Identifying Relationship: "Contains" (Hotel 1 ──── N Room)
                          Total participation of Room (mandatory)
```

**Why weak?** "Room 204" is meaningless without knowing *which hotel* — two completely different hotels in the same chain can each have a "Room 204." The partial key `room_number = "204"` repeats across different `hotel_id` owners exactly as the scoping insight in §4 describes — and that's correct, expected behavior, not a bug.

```sql
CREATE TABLE Room (
    hotel_id     INT,
    room_number  VARCHAR(10),
    room_type    VARCHAR(20),
    floor        INT,
    PRIMARY KEY (hotel_id, room_number),
    FOREIGN KEY (hotel_id) REFERENCES Hotel(hotel_id) ON DELETE CASCADE
);
```

---

## 8. Real-System Example 3: E-Commerce — OrderLineItem

```
Owner Entity:            Order (Strong)
                          order_id (PK), order_date, customer_id

Weak Entity:              OrderLineItem
                          line_number (Partial Key — e.g., 1, 2, 3 within the order)
                          product_id, quantity, unit_price

Identifying Relationship: "Contains" (Order 1 ──── N OrderLineItem)
                          Total participation of OrderLineItem (mandatory)
```

**Why weak?** "Line item #1" only makes sense in the context of one specific order — `line_number = 1` will naturally repeat across every single order (nearly every order has a "line 1"), and that repetition is completely fine, since uniqueness is only required within each order.

```sql
CREATE TABLE OrderLineItem (
    order_id     INT,
    line_number  INT,
    product_id   INT,
    quantity     INT CHECK (quantity > 0),
    unit_price   DECIMAL(10,2),
    PRIMARY KEY (order_id, line_number),
    FOREIGN KEY (order_id) REFERENCES `Order`(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Product(product_id)
);
```

> 💡 Notice `OrderLineItem` has **two** foreign keys: one to its **owner** (`Order`, forming part of its identifying relationship) and one to an **unrelated strong entity** (`Product`, a completely ordinary, non-identifying relationship). This demonstrates that a weak entity can absolutely participate in relationships beyond just its identifying one — a nuance worth remembering (echoing §3's `Dependent`-`InsurancePlan` aside).

---

## 9. Real-System Example 4: Banking — Transaction

```
Owner Entity:            Account (Strong)
                          account_no (PK), balance, account_type

Weak Entity:              Transaction
                          transaction_seq (Partial Key — sequential number per account)
                          amount, transaction_date, transaction_type

Identifying Relationship: "Records" (Account 1 ──── N Transaction)
                          Total participation of Transaction (mandatory)
```

**Why weak?** A transaction only has meaning in the context of one specific account — "transaction #47" is meaningless without knowing which account it's #47 *for*.

> ⚠️ **Real-World Nuance:** In practice, many production banking systems actually DO give transactions a **globally unique surrogate transaction ID** (e.g., a UUID) for cross-system reconciliation, auditing, and regulatory reference — even though *conceptually* the transaction is still weak/dependent on its account. This is a great example of the pragmatic trade-off flagged back in Module 6, §8's "Common Mistakes" section: adding a surrogate key for implementation convenience does **not** eliminate the underlying weak-entity dependency (the `NOT NULL, ON DELETE CASCADE`-style foreign key to `Account` should still exist) — it just adds a *second*, independently useful identifier alongside it.

---

## 10. Chains of Weak Entities (Edge Case)

### Can a Weak Entity Own Another Weak Entity?

**Yes.** This is a legitimate, if less common, edge case: a weak entity can itself serve as the "owner" of an even more deeply nested weak entity, forming a **chain of dependency**.

### Example

```
Hotel (Strong)
  └── owns → Room (Weak, partial key: room_number)
                └── owns → RoomInspection (Weak, partial key: inspection_date)

Full identification of a RoomInspection:
  (hotel_id, room_number, inspection_date)
  ↑ owner's owner's PK  ↑ owner's partial key  ↑ this entity's own partial key
```

The full identifying key formula from §5 simply **chains**: each level's full key becomes the "owner's primary key" input for the next level down.

```sql
CREATE TABLE RoomInspection (
    hotel_id         INT,
    room_number      VARCHAR(10),
    inspection_date  DATE,
    inspector_name   VARCHAR(50),
    notes            TEXT,
    PRIMARY KEY (hotel_id, room_number, inspection_date),
    FOREIGN KEY (hotel_id, room_number) REFERENCES Room(hotel_id, room_number)
        ON DELETE CASCADE
);
```

### Interview Importance

**Medium** — a good edge case to mention if asked "can weak entities be nested?" to demonstrate depth beyond the single-level textbook example, but rarely the primary focus of a question.

---

## 11. How to Decide: Strong, Weak, or Just an Attribute?

This is the **most practically valuable** section of this module — a decision framework for real design work and interviews.

```
                    "Should this be its own entity, or just an attribute?"
                                          │
                    ┌─────────────────────┴─────────────────────┐
                    ▼                                           ▼
        Does it have MULTIPLE meaningful           No independent significance,
        attributes of its own, or a                 single value or simple list?
        one-to-many relationship to track?                     │
                    │                                           ▼
                    ▼                                    Model as an ATTRIBUTE
        Model as an ENTITY                               (simple or multi-valued,
                    │                                      Module 7)
                    ▼
        "Does it have a GLOBALLY meaningful,
         independent identity — could it exist
         and be referenced even if its 'parent'
         were deleted?"
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
       YES                     NO
        │                       │
        ▼                       ▼
  STRONG ENTITY            WEAK ENTITY
  (own full PK)            (partial key + owner's PK,
                             total participation,
                             ON DELETE CASCADE typical)
```

### Worked Decision Examples

| Candidate | Decision | Reasoning |
|---|---|---|
| `Employee.phone_number` | Attribute (possibly multi-valued) | No independent significance beyond being a contact detail |
| `Employee.skills` | Attribute (multi-valued) *or* weak/associative entity | Depends: if just a list, multi-valued attribute; if it needs `proficiency_level`, `certified_date`, etc., promote to an entity (Module 6, §9's associative entity logic) |
| `Dependent` | Weak Entity | Has multiple attributes (name, relationship, DOB) AND cannot exist/be referenced independently of the `Employee` |
| `Department` | Strong Entity | Has independent significance — could exist as a concept even if a specific `Employee` were deleted; often referenced from multiple other entities |

### Interview Importance

**Extremely High.** This decision framework is *the* practical skill being tested in nearly every "design a schema for X" interview — memorize and practice narrating this exact reasoning chain.

---

## 12. Common Mistakes in Identifying Weak Entities

- **Forgetting total participation:** Modeling a weak entity's relationship to its owner as optional/partial participation — a weak entity, by definition, cannot exist without its owner, so this relationship must always be total (mandatory) on the weak entity's side. (Fully covered in Module 11.)
- **Using a non-scoped partial key:** Accidentally trying to make the partial key globally unique when it doesn't need to be (e.g., unnecessarily using a UUID for `room_number` instead of the natural, owner-scoped `"204"`) — this misses the actual point of the weak-entity pattern, though it isn't *technically* wrong, just needlessly complex.
- **Missing `ON DELETE CASCADE` (or equivalent) after mapping to SQL:** If the ER model correctly identifies total participation and a weak entity relationship, but the resulting foreign key is declared with `ON DELETE RESTRICT` or no cascade behavior at all, the eventual relational schema no longer faithfully reflects the conceptual design — a common source of "orphaned but blocked" data integrity headaches.
- **Over-promoting simple attributes to weak entities:** Not every "list of things" needs to become a weak entity — revisit §11's decision framework before reaching for a weak entity table.

### Interview Importance

**High** — being able to proactively name these mistakes (rather than just avoiding them silently) is itself a strong signal in a live interview discussion.

---

## 13. Chapter Summary

- The **Owner Entity** is always a strong entity that provides part of a weak entity's identity; the **Identifying Relationship** is the specific relationship connecting them, always requiring the weak entity's **total participation** (Module 11).
- The **Partial Key** only needs to be unique **within its owner's scope**, not globally — this scoping insight (repeated across four real-system examples: `Dependent`, `Room`, `OrderLineItem`, `Transaction`) is the single most important, frequently-tested fact in this module.
- The **full identifying key formula** — `Owner's Primary Key + Weak Entity's Partial Key` — mechanically determines the weak entity's eventual composite primary key and `ON DELETE CASCADE` foreign key once mapped to the relational model (Module 13).
- Weak entities **can chain** (a weak entity can itself own another, even-more-deeply-nested weak entity), and **can participate in additional, non-identifying relationships** beyond their identifying one (e.g., `OrderLineItem → Product`).
- A practical **decision framework** (§11) — asking "multiple attributes/own significance?" then "can it exist independently?" — is the core reusable skill for correctly classifying any candidate entity as an attribute, strong entity, or weak entity during real design work and interviews.
- Real production systems sometimes pragmatically add a surrogate key to a conceptually weak entity (e.g., banking `Transaction`) for cross-system referencing — this is a valid trade-off, as long as the underlying ownership dependency (`NOT NULL` FK, cascade behavior) is preserved.

---

## 14. Cheat Sheet

```
OWNER ENTITY               → always a STRONG entity, provides part of weak entity's identity
IDENTIFYING RELATIONSHIP    → connects weak entity to its owner; weak entity has
                               TOTAL PARTICIPATION (mandatory, Module 11)
PARTIAL KEY                 → unique only WITHIN the owner's scope, NOT globally
                               (dashed underline in Chen's notation)

FULL IDENTIFYING KEY FORMULA:
  Weak Entity's Composite PK = Owner's Primary Key + Weak Entity's Partial Key

MAPPING PREVIEW (Module 13):
  → Composite PK = (owner_fk_column(s), partial_key_column)
  → FOREIGN KEY to owner, typically ON DELETE CASCADE (reflects mandatory dependency)

DECISION FRAMEWORK:
  "Multiple own attributes / own significance?"  NO → just an ATTRIBUTE (Module 7)
                                                  YES ↓
  "Can it exist / be referenced independently of its 'parent'?"
        YES → STRONG ENTITY
        NO  → WEAK ENTITY

CLASSIC WEAK ENTITY EXAMPLES:
  Dependent (of Employee) | Room (of Hotel) | OrderLineItem (of Order) | Transaction (of Account)

WEAK ENTITIES CAN CHAIN: a weak entity can own an even deeper weak entity
  (e.g., Hotel → Room → RoomInspection)
```

---

## 15. Exercises

### Beginner

1. Define "owner entity" and "identifying relationship" in one sentence each.
2. True/False: A partial key must be globally unique across the entire weak entity set.
3. Fill in the blank: The full identifying key of a weak entity is formed by combining the owner's __________ with the weak entity's __________.

### Intermediate

4. Explain why `room_number = "204"` can validly appear in multiple different hotels' `Room` weak entity instances without violating any key constraint.
5. Design the weak entity `Transaction` (§9) fully in SQL, including its composite primary key and foreign key with appropriate `ON DELETE` behavior. Justify your choice of `ON DELETE` action.
6. A designer claims `OrderLineItem` cannot have a relationship to any entity other than `Order`. Correct this misconception using the `Product` foreign key example from §8.

### Advanced / Conceptual

7. Design a three-level chain of weak entities (similar to Hotel → Room → RoomInspection in §10) for a different domain of your choice (e.g., University → Course → CourseSection → ClassSession). Write out the full composite primary key formula at each level.
8. Using the decision framework in §11, walk through whether `Order.shipping_address` should be modeled as a composite attribute, a weak entity, or a strong entity, considering a business requirement where customers can save and reuse multiple named addresses (e.g., "Home", "Office") across different orders.
9. Explain the real-world trade-off (§9) of adding a surrogate key to a conceptually weak entity like `Transaction`, and describe what would still need to be preserved in the schema to avoid losing the weak-entity dependency.

### MCQs

**Q1.** A weak entity's participation in its identifying relationship is:
a) Always optional  b) Always total (mandatory)  c) Irrelevant to the ER model  d) Only relevant after normalization

**Q2.** A partial key must be unique:
a) Across the entire database  b) Only within its owner entity's scope  c) Only within a single row  d) It never needs to be unique

**Q3.** Which of these is the most likely `ON DELETE` behavior for a weak entity's foreign key to its owner?
a) RESTRICT  b) CASCADE  c) SET NULL  d) SET DEFAULT

**Q4.** Which entity is the odd one out (i.e., NOT a typical weak entity)?
a) Dependent (of Employee)  b) Department (of Company)  c) Room (of Hotel)  d) OrderLineItem (of Order)

*(Answer key: 1-b, 2-b, 3-b, 4-b — Department typically has independent significance and is usually modeled as a strong entity, unlike the other three.)*

### Schema/Diagram Exercise

10. Draw a Chen's-notation ER diagram (owner entity, weak entity, identifying relationship, partial key with dashed underline) for a `Library` → `BookCopy` relationship, where each physical copy of a book is identified by a `copy_number` unique only within that specific book title.

### Practice Links

- *(Weak entity modeling is a conceptual/diagram skill — practice via the exercises above.)*
- [LeetCode Database — problems involving composite keys and parent-child table relationships](https://leetcode.com/studyplan/top-sql-50/)

---

**Next: Module 9 — Relationships (Degree of Relationship, Unary, Binary, Ternary, Recursive Relationships, N-ary Relationships).**

> Say "continue" or "next module" to proceed to Module 9.
