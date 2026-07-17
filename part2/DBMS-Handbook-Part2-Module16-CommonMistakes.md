# DBMS Handbook — Part 2
## Module 16: Common Modeling Mistakes

> **Builds on:** Nearly every prior module in this Part — this module is a **deliberate synthesis**, revisiting mistakes already flagged individually throughout Modules 2-15, and adding several new ones, all consolidated into one practical, review-ready checklist.
>
> **Feeds into:** Module 17 (Industry Case Studies — apply this checklist while reviewing each case study's design), and Normalization (Part 3, which formally addresses over/under-normalization).

---

## Table of Contents

1. [Why a Dedicated "Mistakes" Module?](#1-why-a-dedicated-mistakes-module)
2. [Wrong Primary Keys](#2-wrong-primary-keys)
3. [Duplicate Entities](#3-duplicate-entities)
4. [Missing Relationships](#4-missing-relationships)
5. [Poor Naming](#5-poor-naming)
6. [Incorrect Cardinality](#6-incorrect-cardinality)
7. [Circular References](#7-circular-references)
8. [Over-Normalization (Preview)](#8-over-normalization-preview)
9. [Under-Normalization (Preview)](#9-under-normalization-preview)
10. [The Full Schema Audit Checklist](#10-the-full-schema-audit-checklist)
11. [Chapter Summary](#11-chapter-summary)
12. [Cheat Sheet](#12-cheat-sheet)
13. [Exercises](#13-exercises)

---

## 1. Why a Dedicated "Mistakes" Module?

### The Value of Studying Failure Modes

Throughout this Part, individual "Common Mistakes" callouts were embedded within nearly every module — this is deliberate: recognizing what **not** to do is often just as valuable, and just as heavily interview-tested, as knowing the correct theory. This module exists to **consolidate every one of those scattered warnings into a single, comprehensive, review-ready reference** — exactly the kind of checklist you'd want open next to you during a live schema-design interview or a real production schema review.

---

## 2. Wrong Primary Keys

### The Mistake

Choosing a primary key that is **mutable, non-unique in edge cases, or unnecessarily complex** — most commonly, using a natural/business attribute (email, phone number, government ID) directly as a primary key without considering the consequences (fully explored in Module 4, §10, §16).

### Real Example of the Failure

```
❌ Customer(email PRIMARY KEY, name, ...)

Six months later: a customer legitimately needs to change their email.
Now every table with a foreign key to Customer.email needs a
cascading update — a slow, risky, error-prone operation across
potentially dozens of tables.
```

### The Fix

Prefer a **surrogate key** (Module 4, §11) as the primary key, and enforce the natural key's uniqueness separately via a `UNIQUE` constraint (Module 4, §16's "best of both worlds" pattern).

### Interview Importance

**Extremely High** — "what's wrong with using email as a primary key?" is one of the most common schema-critique interview questions.

---

## 3. Duplicate Entities

### The Mistake

Accidentally modeling the **same real-world concept as two (or more) separate, disconnected entities**, usually because different parts of a team modeled different subsystems independently without cross-checking.

### Example

```
❌ Modeled independently by two different teams:
   Customer(customer_id, name, email, ...)     -- from the Sales team's schema
   Client(client_id, full_name, email_address, ...)  -- from the Billing team's schema

   Both represent the EXACT SAME real-world concept (a paying customer),
   but exist as two unrelated tables with no shared identifier — leading
   to duplicated data entry, inconsistent updates, and confused reporting
   ("why does our customer count differ between Sales and Billing dashboards?").
```

### Why This Happens

Frequently a symptom of **insufficient requirements-gathering and cross-team communication** during the conceptual ER modeling stage (Module 6, §3) — a reminder that ER modeling's *communication* purpose (Module 6, §1) is not just a nice-to-have, but a genuine defense against this exact class of mistake.

### The Fix

- Consolidate into a single, shared `Customer` entity, with team-specific *additional* attributes/tables layered on top via foreign keys (not duplicated core identity data).
- Establish clear, shared entity ownership and vocabulary **before** independent teams begin modeling — a data governance practice, not just a technical one.

### Interview Importance

**Medium-High** — more commonly discussed in the context of larger, multi-team system design interviews than pure single-schema exercises.

---

## 4. Missing Relationships

### The Mistake

Failing to model a relationship that genuinely exists in the business domain — often because it wasn't surfaced during requirements gathering (Module 6, §3), or because it seemed "obviously implied" and was skipped.

### Example

```
❌ Order(order_id, customer_id, ...)
   Coupon(coupon_id, code, discount_pct, ...)

   The business actually needs to know WHICH coupon (if any) was applied
   to WHICH order — but no relationship between Order and Coupon was ever
   modeled, because it wasn't mentioned explicitly during initial
   requirements discussions. This gets discovered late — often during
   development, or worse, after launch — requiring a retrofit migration.
```

### The Fix

- Systematically walk through every entity pair during ER modeling and explicitly ask: "does any real business relationship exist between these two?" — don't rely solely on relationships stakeholders happened to mention unprompted.
- Revisit Module 6, §10's full worked-example process (requirements → entities → relationships) as a disciplined checklist, rather than an informal pass.

### Interview Importance

**High** — interviewers sometimes deliberately under-specify requirements to see if a candidate proactively asks clarifying questions that would surface a missing relationship, rather than silently designing around an incomplete requirement set.

---

## 5. Poor Naming

### The Mistake

*(Extensively covered in Module 14, §2-§4 — summarized here for checklist completeness.)*

Inconsistent case styles, ambiguous abbreviations, redundant table-name repetition in column names, or undocumented magic values — all covered in full depth in Module 14.

### Interview Importance

**Medium** — see Module 14 for the full treatment; included here purely for checklist completeness.

---

## 6. Incorrect Cardinality

### The Mistake

Modeling a relationship with the **wrong cardinality type** (Module 10) — most commonly, modeling something as 1:N when it should genuinely be M:N (or vice versa), because the true business rule wasn't verified.

### Classic Example (Revisiting Module 10, §8's Warning)

```
❌ Book(book_id, title, author_id)   -- assumes 1:N (one author per book)

   Reality: the business needs to support CO-AUTHORED books.
   This is actually M:N, and the schema above cannot represent
   a book with two authors at all without a painful migration
   (already discussed as "Changing Cardinality Over Time," Module 10 §9).
```

### Another Classic Example: Incorrectly Assuming 1:1

```
❌ Employee(emp_id, ..., phone_number)   -- assumes each employee has exactly
                                              one phone number (single-valued)

   Reality: employees can have multiple numbers (home, mobile, work) —
   this isn't even a cardinality mistake between two entities, but a
   MULTIVALUED ATTRIBUTE mistake (Module 7 §6) — a closely related,
   equally common failure mode.
```

### The Fix

- **Always verify cardinality assumptions explicitly with real stakeholders** — never assume based on what "seems typical." (Module 10, §8's key lesson: cardinality is a business-rule decision, not a fixed technical fact.)
- When genuinely uncertain, and the cost of being wrong is high, consider the trade-off discussion from Module 10, §9 (design for likely-future M:N vs. accept a future migration risk).

### Interview Importance

**Extremely High** — verifying cardinality assumptions out loud, rather than silently guessing, is one of the clearest signals of a strong, requirements-driven design interview candidate.

---

## 7. Circular References

### The Mistake

Creating a **circular foreign key dependency** — Table A has a (typically mandatory, `NOT NULL`) foreign key to Table B, and Table B has a (typically mandatory) foreign key back to Table A — making it **impossible to insert either row first** without violating a constraint.

### Example

```
❌ CREATE TABLE Employee (
       emp_id      INT PRIMARY KEY,
       dept_id     INT NOT NULL,
       FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
   );
   CREATE TABLE Department (
       dept_id     INT PRIMARY KEY,
       head_emp_id INT NOT NULL,             -- every department MUST have a head
       FOREIGN KEY (head_emp_id) REFERENCES Employee(emp_id)
   );

   -- DEADLOCK-STYLE INSERT PROBLEM:
   -- To insert an Employee, its dept_id must already exist in Department.
   -- To insert a Department, its head_emp_id must already exist in Employee.
   -- Neither can be inserted FIRST!
```

### Why This Is a Genuinely Tricky, Frequently Tested Edge Case

This scenario directly echoes Module 11, §6's `Department`–`Employee`–`Manages` worked example — and this module now reveals **why that example carefully chose `Employee.dept_id` to be nullable** in one direction, precisely to *avoid* this exact circular deadlock.

### The Fix — Three Legitimate Approaches

| Approach | How It Works |
|---|---|
| **Make one FK nullable** | `Department.head_emp_id` becomes nullable — insert the `Department` first (with `head_emp_id = NULL`), then insert `Employee`s, then `UPDATE Department SET head_emp_id = ...` afterward |
| **Deferred constraint checking** | Recall Module 5, §16 — if the RDBMS supports deferred constraints, both rows can be inserted within a single transaction, with checking postponed until `COMMIT` |
| **Break the cycle with a separate junction/assignment table** | e.g., a `DepartmentHead(dept_id, emp_id, effective_date)` table, removing the direct circular FK entirely — often the cleanest, most auditable long-term solution, since it also naturally supports tracking head-of-department *history* over time |

### Interview Importance

**High** — this exact "chicken-and-egg" insert problem, and the ability to name at least one of the three fixes above, is a classic, specific, well-defined interview question that tests genuine hands-on schema design experience.

---

## 8. Over-Normalization (Preview)

> **Note:** Formal Normalization (1NF through BCNF) is the complete subject of Part 3 — this section previews only the **mistake pattern**, so it can sit in this module's checklist alongside its counterpart, under-normalization.

### The Mistake

Splitting data into **excessively many, overly granular tables**, beyond what any real query pattern or redundancy concern actually justifies — resulting in a schema that requires numerous `JOIN`s even for simple, common queries, hurting both query performance and developer usability.

### Illustrative (Deliberately Exaggerated) Example

```
❌ Overly fragmented:
   PersonFirstName(person_id, first_name)
   PersonLastName(person_id, last_name)
   PersonEmail(person_id, email)

   -- Even fetching one person's basic contact card now requires
   -- THREE separate joins for data that's simple, single-valued,
   -- and virtually always accessed together.
```

### The Fix

Recall Module 7, §4's composite-attribute guidance: split into separate columns/tables **only when there's a genuine, concrete reason** (independent querying needs, or genuine multi-valued/1NF violations) — not reflexively, "just in case," or purely for theoretical purity's sake.

### Interview Importance

**Medium-High** — mentioning over-normalization as a real, opposite-direction risk (not just under-normalization) demonstrates balanced, practical design judgment — full formal treatment awaits Part 3.

---

## 9. Under-Normalization (Preview)

### The Mistake

The **opposite** failure: cramming too much unrelated or redundant data into too few, overly-wide tables — leading directly to the redundancy problems already explored in Module 14, §6, and to update/insert/delete "anomalies" that will be formally named and diagnosed via Normal Forms in Part 3.

### Illustrative Example (Revisiting Module 14, §6)

```
❌ Under-normalized:
   Order(order_id, customer_name, customer_email, product_name, quantity)

   Same redundancy problem already fully explored in Module 14 §6 —
   included here for checklist completeness, since it's the direct,
   necessary counterpart to over-normalization (§8) above.
```

### The Fix

Apply the redundancy-avoidance principles from Module 14, §6 as a practical first pass, with the **formal, rigorous** Normal Form analysis (1NF/2NF/3NF/BCNF) awaiting full treatment in Part 3.

### Interview Importance

**High** — see Module 14, §6; repeated here to sit alongside over-normalization as a complete, balanced pair.

---

## 10. The Full Schema Audit Checklist

A single, consolidated, print-and-use checklist combining every mistake in this module — designed to be run through, item by item, when reviewing any real ER diagram or relational schema (your own, or in a live interview whiteboard session):

```
□ PRIMARY KEYS
   - Is every table's primary key stable (won't need to change)?
   - Are natural/business keys enforced via UNIQUE rather than used directly as PK?

□ ENTITIES
   - Does every real-world concept map to EXACTLY ONE entity (no duplicates
     across different parts of the schema)?
   - Has every entity been validated against real stakeholder requirements,
     not just assumed?

□ RELATIONSHIPS
   - Has every entity PAIR been explicitly checked for a missing relationship?
   - Has cardinality (1:1, 1:N, M:N) been VERIFIED with real business rules,
     not assumed from "typical" patterns?
   - Has participation (total/partial) been specified for EACH side
     independently (Module 11)?

□ NAMING (Module 14)
   - Is naming consistent (case style, table pluralization, suffixes)?
   - Are foreign keys and booleans clearly, unambiguously named?

□ STRUCTURAL INTEGRITY
   - Are there any circular, mutually-mandatory foreign key dependencies?
     If so, has one of the three fixes (§7) been applied?
   - Are weak entities correctly using composite keys with the owner's PK
     (Module 8)?

□ NORMALIZATION BALANCE
   - Is the schema free of obvious, unjustified redundancy (under-normalized)?
   - Is the schema free of excessive, query-pattern-unjustified fragmentation
     (over-normalized)?

□ DATA TYPES & CONSTRAINTS (Module 5, 14)
   - Is currency stored as DECIMAL, never FLOAT?
   - Are NOT NULL / nullable choices correctly derived from participation
     constraints (Module 11 §7)?
   - Are evolving categorical values in lookup tables, not magic numbers?
```

### Interview Importance

**Extremely High** — literally memorizing and mentally running through this checklist during any live schema-design interview is one of the single highest-leverage preparation activities in this entire Part.

---

## 11. Chapter Summary

- This module consolidates mistakes flagged throughout the entire Part into one unified, practical reference: **Wrong Primary Keys** (mutable natural keys, Module 4), **Duplicate Entities** (a governance/communication failure, Module 6), **Missing Relationships** (incomplete requirements gathering), **Poor Naming** (Module 14), **Incorrect Cardinality** (unverified business-rule assumptions, Module 10), **Circular References** (mutually-mandatory FKs causing insert deadlocks), and **Over/Under-Normalization** (previewing Part 3's formal Normal Forms).
- **Circular references** deserve special attention: the classic `Employee`↔`Department` mutual-mandatory-FK deadlock has **three legitimate fixes** — making one FK nullable, deferred constraint checking (Module 5, §16), or breaking the cycle with a separate junction/assignment table.
- **Over-normalization** and **under-normalization** are genuine **opposite-direction** risks — good schema design is a balance, not a maximization of either "no redundancy at all costs" or "no joins at all costs."
- The **Full Schema Audit Checklist** (§10) is the single most practically valuable artifact in this module — a ready-to-use, comprehensive review tool for any real ER diagram or relational schema.

---

## 12. Cheat Sheet

```
WRONG PRIMARY KEYS       → mutable natural keys as PK; fix: surrogate PK + UNIQUE natural key
DUPLICATE ENTITIES        → same real-world concept modeled twice; fix: shared entity, cross-team alignment
MISSING RELATIONSHIPS     → unasked "does X relate to Y?" questions; fix: systematic entity-pair review
POOR NAMING                → inconsistency (Module 14); fix: one convention, enforced everywhere
INCORRECT CARDINALITY      → unverified 1:N vs M:N assumptions; fix: confirm with real stakeholders
CIRCULAR REFERENCES         → mutual NOT NULL FKs = insert deadlock; fix: nullable FK, deferred
                              constraints, OR a separate junction table
OVER-NORMALIZATION (preview) → excessive fragmentation, unjustified joins
UNDER-NORMALIZATION (preview)→ redundancy, update/insert/delete anomalies (full: Part 3)

RUN THE FULL AUDIT CHECKLIST (§10) ON EVERY SCHEMA YOU DESIGN OR REVIEW.
```

---

## 13. Exercises

### Beginner

1. List the seven mistake categories covered in this module.
2. True/False: A circular foreign key dependency where both sides are NOT NULL can always be inserted in some order without issue.
3. Fill in the blank: Splitting a schema into excessively many, overly granular tables beyond what queries or redundancy concerns justify is called __________.

### Intermediate

4. Design a fix for the circular `Employee`↔`Department` deadlock (§7) using the "separate junction/assignment table" approach, and explain one advantage this approach has over simply making one FK nullable.
5. A schema has `Customer` and `Client` tables representing the same real-world entity (§3). Propose a consolidation plan, including how to handle any team-specific attributes each currently has.
6. Identify which mistake category best explains this scenario: a schema assumed `Employee.phone_number` was single-valued, but the business needs multiple numbers per employee. (Hint: revisit §6's second example.)

### Advanced / Conceptual

7. Run the Full Schema Audit Checklist (§10) against the final University System schema from Module 13, §11. Identify any potential issues (even minor or debatable ones) and justify your findings.
8. Explain why "incorrect cardinality" and "missing relationships" are both, at their root, symptoms of the same underlying failure — inadequate requirements verification during ER modeling (Module 6, §3). What process change would address both simultaneously?
9. Discuss a real or hypothetical scenario where over-normalization caused a genuine performance problem, and explain how you would identify and fix it without swinging too far into under-normalization.

### MCQs

**Q1.** Which of the following is a legitimate fix for a circular, mutually-mandatory foreign key dependency?
a) Making both foreign keys NOT NULL  b) Making one foreign key nullable, or using deferred constraints, or a separate junction table  c) Deleting one of the two tables  d) There is no fix; circular references are always impossible

**Q2.** Using a customer's email address directly as a primary key is risky primarily because:
a) Emails are always too short  b) Emails can change, requiring painful cascading updates  c) Emails cannot be indexed  d) Primary keys must always be integers

**Q3.** Over-normalization primarily causes which problem?
a) Data redundancy  b) Update anomalies  c) Excessive joins for common queries  d) Missing foreign keys

**Q4.** The best defense against "duplicate entities" (the same concept modeled twice) is:
a) Using surrogate keys everywhere  b) Cross-team alignment and shared vocabulary during ER modeling  c) Avoiding all foreign keys  d) Over-normalizing every table

*(Answer key: 1-b, 2-b, 3-c, 4-b)*

### Schema Audit Exercise

10. Take any schema you designed in a previous module's exercises (e.g., the Hotel Booking System from Module 6, §13's exercises, or the Car Rental System from Module 13, §15's exercises) and run the Full Schema Audit Checklist (§10) against it. Document at least two findings (even if minor) and propose fixes.

### Practice Links

- *(This module is a synthesis/review skill — the richest practice is re-auditing every schema you've designed throughout this Part using the checklist in §10.)*
- [LeetCode Database — Top SQL 50 (many "given schema" problems implicitly reward recognizing well- vs. poorly-designed structures)](https://leetcode.com/studyplan/top-sql-50/)

---

**Next: Module 17 — Industry Case Studies (Complete ER Models for Banking, Hospital Management, E-commerce, University Management, Railway Reservation, Social Media, Food Delivery, Inventory Management).**

> Say "continue" or "next module" to proceed to Module 17.
