# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 1: Database Design Fundamentals

> **Prerequisite check:** This module assumes you've completed Parts 1–4 (SQL fundamentals, relational model, ER modeling basics, and query writing). If any of those feel shaky, it's worth a quick revisit before diving in — everything here builds on "you can already write and reason about a schema."

---

### 1.1 What Is Database Design?

**Definition:**
Database design is the process of deciding **how data will be structured, stored, and related** inside a database so that it is correct, non-redundant, efficient to query, and easy to maintain over the system's lifetime.

More formally: database design is the translation of **real-world information requirements** into a **formal schema** — a blueprint made of tables (relations), columns (attributes), data types, keys, and constraints — that a DBMS (PostgreSQL, MySQL, Oracle, SQL Server, etc.) can enforce and operate on.

**Analogy — The Architect's Blueprint:**
Think of a database as a building and the database designer as an architect.

- You wouldn't start construction (writing `CREATE TABLE` statements) without a blueprint.
- A bad blueprint means walls in the wrong place, plumbing that leaks, rooms that don't fit their purpose — and by the time you notice, tearing it down and rebuilding is enormously expensive.
- A bad database design means duplicated data, inconsistent updates, slow queries, and painful migrations — and by the time you notice (usually in production, under load), fixing it is enormously expensive.

Database design is **the blueprint phase**. Everything downstream — application code, APIs, reports, analytics — is built on top of it. Mistakes here ripple through the entire system.

---

### 1.2 Why Database Design Matters

If you skip proper design and just "start creating tables," you tend to get:

| Problem | What Happens | Real-World Consequence |
|---|---|---|
| **Redundancy** | Same fact stored in multiple places | Wasted storage, higher chance of inconsistency |
| **Update Anomalies** | Updating one copy but not others | Customer's address is "correct" in one table, stale in another |
| **Insert Anomalies** | Can't add a fact without adding unrelated facts | Can't add a new course unless a student enrolls in it |
| **Delete Anomalies** | Deleting one fact accidentally deletes another | Deleting the last order of a customer deletes the customer's contact info entirely |
| **Poor Performance** | Badly structured tables force expensive joins or scans | Dashboards time out, APIs become slow under load |
| **Poor Scalability** | Schema can't absorb new requirements | Every new feature requires a risky migration |
| **Data Integrity Loss** | No constraints to prevent invalid states | Orders reference customers that don't exist |

We will formally define and dissect **anomalies** in Module 7 (Normalization), but it's important to see upfront that database design isn't academic box-ticking — it directly determines whether a system stays maintainable at scale or collapses under its own inconsistency 2–3 years into production.

> **Industry note:** At most FAANG-scale companies, a poorly designed core schema (e.g., the "users" or "orders" table) becomes near-impossible to fix later because thousands of queries, reports, and services depend on it. This is why senior engineers spend disproportionate time reviewing schema design *before* code is written — the cost of a design mistake compounds with usage.

---

### 1.3 Design Goals

A well-designed database aims to satisfy the following goals simultaneously. They sometimes pull in different directions, which is why design involves **trade-offs** (Section 1.11).

1. **Correctness** — the schema should be able to represent all valid real-world states and reject invalid ones (via keys, constraints, data types).
2. **Non-redundancy** — each fact should ideally be stored in exactly one place (this is the driving idea behind normalization, Modules 7–13).
3. **Consistency** — the database should never allow contradictory data to persist.
4. **Efficiency** — queries the application actually runs should be fast; storage should not be wasted.
5. **Simplicity & Clarity** — a new engineer should be able to understand the schema without a two-hour briefing.
6. **Scalability** — the schema should tolerate growth in data volume and evolving requirements without a full rewrite.
7. **Maintainability** — schema changes (adding a column, a table) should be low-risk and incremental.
8. **Security & Integrity** — sensitive data should be identifiable and protectable; referential integrity should be enforced wherever the DBMS allows.

**Mnemonic:** *C.O.N.C.E.S.S. — Correctness, Others'-clarity, Non-redundancy, Consistency, Efficiency, Scalability, Security.* (Not a standard industry acronym — just a memory aid.)

---

### 1.4 The Database Design Process — Overview

Database design is typically carried out in **three progressive stages**, each narrowing from "what the business needs" to "what the DBMS actually stores":

```
┌─────────────────────┐        ┌─────────────────────┐        ┌─────────────────────┐
│   CONCEPTUAL DESIGN   │  ───▶  │    LOGICAL DESIGN     │  ───▶  │   PHYSICAL DESIGN     │
│  (What data exists?)  │        │ (How is it structured? │        │ (How is it stored?)   │
│                       │        │  relations, keys, FDs) │        │  indexes, partitions, │
│   ER Diagrams         │        │   Normalized Schema     │        │  storage engine tuning│
└─────────────────────┘        └─────────────────────┘        └─────────────────────┘
      Business analyst              Database designer              DBA / Backend engineer
       & domain expert                 (this Part!)                  (later Parts)
```

Each stage answers a different question and produces a different artifact. Let's go through them one at a time.

---

### 1.5 Conceptual Design

**What is it?**
Conceptual design captures **what information the business/domain needs to track**, independent of any specific DBMS or even the relational model. It answers: *"What are the real-world entities, and how do they relate to each other?"*

**Why does it exist?**
Because jumping straight to tables tends to bake in premature technical decisions before you've even agreed on *what* you're modeling. Conceptual design forces you to talk to domain experts (e.g., a hospital administrator, a warehouse manager) and nail down the entities and relationships in plain language first.

**What problem does it solve?**
It prevents "solutioning too early" — designing tables before understanding the business rules, which almost always leads to rework.

**How does it work?**
Typically expressed using an **Entity-Relationship (ER) Diagram**:
- **Entities** — real-world "things" (Student, Course, Order, Patient)
- **Attributes** — properties of an entity (Student has Name, Roll Number, DOB)
- **Relationships** — how entities interact (Student *enrolls in* Course)
- **Cardinality** — how many of one entity relate to how many of another (one-to-many, many-to-many)

Example (informal, conceptual level — no data types, no keys decided yet):

```
[Student] ---enrolls in---< [Course]
    |                           |
  has                         has
    |                           |
[Contact Info]            [Instructor]
```

**Output artifact:** An ER diagram + a glossary of entities/relationships, reviewed with business stakeholders — *not* SQL.

> Note: Full ER modeling notation (crow's foot, Chen notation, weak entities, ISA hierarchies) was covered in earlier parts of this series as a prerequisite. Here we focus on where it fits in the overall design pipeline.

---

### 1.6 Logical Design

**What is it?**
Logical design translates the conceptual model into a **formal relational schema**: tables, columns, primary keys, foreign keys, and functional dependencies — but *still independent of the specific DBMS product* (this schema should work equivalently on PostgreSQL, MySQL, or Oracle).

**Why does it exist?**
Because a conceptual ER diagram doesn't yet tell you exactly how many tables you need, what the primary keys are, or whether a many-to-many relationship needs a junction table. Logical design formalizes those decisions using relational theory.

**What problem does it solve?**
It's the stage where **normalization happens** (Modules 7–13 of this Part). This is where redundancy and anomalies are identified and eliminated using functional dependency analysis.

**How does it work?**
1. Convert entities → tables.
2. Convert attributes → columns, with appropriate data types.
3. Convert relationships → foreign keys (or junction tables, for many-to-many).
4. Identify candidate keys and choose a primary key.
5. Derive functional dependencies from business rules.
6. Normalize the schema (typically to 3NF or BCNF) to eliminate redundancy.
7. Document constraints (uniqueness, not-null, check constraints).

Example — Student/Course conceptual model becomes:

```sql
Student(student_id PK, name, dob, email)
Course(course_id PK, title, credits)
Enrollment(student_id FK, course_id FK, enrollment_date, grade)
   PRIMARY KEY (student_id, course_id)
```

This is the **core focus of this entire Part 5** — Modules 2 through 13 go deep into the theory (functional dependencies, Armstrong's Axioms, normal forms, decomposition) that makes logical design rigorous rather than guesswork.

---

### 1.7 Physical Design

**What is it?**
Physical design decides **how the logical schema is actually stored and accessed** by a specific DBMS: indexing strategy, partitioning, storage engine choice, file organization, denormalization for performance, and hardware/resource considerations.

**Why does it exist?**
Because two databases with the *identical* logical schema can perform completely differently depending on physical choices — which columns are indexed, whether the table is partitioned by date, whether you've denormalized a hot-path table to avoid a join.

**What problem does it solve?**
It bridges the gap between "theoretically correct schema" and "schema that performs well under real production workloads."

**How does it work?** (previewed here, covered in depth in later Parts and Module 14 for denormalization)
- Choosing indexes (B-Tree, Hash, GIN, etc. — covered in the Indexing part of this series)
- Partitioning large tables (by range, hash, or list)
- Choosing storage engines (e.g., InnoDB vs MyISAM in MySQL)
- Denormalizing specific tables for read-heavy paths (Module 14)
- Choosing data types precisely for storage efficiency (`SMALLINT` vs `INT`, `VARCHAR(50)` vs `TEXT`)
- Setting up replication/sharding considerations (previewed, not this Part's focus)

Example: the logical schema above might, at the physical level, get an index on `Enrollment(course_id)` because reporting queries frequently ask "who is enrolled in this course," even though `(student_id, course_id)` is already the primary key.

> **This Part (Part 5) focuses almost entirely on Logical Design** — functional dependencies and normalization. Physical design (indexing, storage engines, query optimization) is covered in depth in upcoming parts, as flagged in Module 19.

---

### 1.8 Conceptual vs Logical vs Physical Design — Comparison Table

| Aspect | Conceptual Design | Logical Design | Physical Design |
|---|---|---|---|
| **Question answered** | What data exists? | How is it structured relationally? | How is it stored and accessed? |
| **Audience** | Business stakeholders, domain experts | Database designers, backend engineers | DBAs, backend/infra engineers |
| **Artifact** | ER Diagram | Normalized relational schema (tables, keys, FDs) | DDL with indexes, partitions, storage settings |
| **DBMS-specific?** | No | No (mostly) | Yes, highly |
| **Focus** | Entities & relationships | Eliminating redundancy & anomalies | Performance & scalability |
| **Typical tools** | ER modeling tools, whiteboards | Normalization theory, dependency diagrams | `EXPLAIN`, index advisors, DBMS-specific tuning |
| **Covered in** | Prior parts (ER modeling) | **This Part (Part 5)** | Later parts (Indexing, Query Optimization) |

---

### 1.9 Bottom-Up Design

**What is it?**
Bottom-up design starts from **individual attributes** (fields you know you need — e.g., `student_name`, `course_title`, `enrollment_date`), groups them into relations based on their functional dependencies, and builds up to the full schema using normalization theory.

**Why does it exist?**
It's useful when you already have a large, messy set of data attributes (e.g., a legacy spreadsheet, a flat CSV export, an existing badly-designed table) and need to reverse-engineer a clean, normalized structure from them.

**How does it work?**
1. List every attribute in the system.
2. Determine functional dependencies among them (Module 2).
3. Group attributes into relations using normalization algorithms (Modules 7–13).
4. Verify the result is lossless and dependency-preserving (Module 13).

**Analogy:** Bottom-up design is like sorting a huge pile of mixed LEGO bricks (attributes) into labeled bins (tables) based on which bricks "belong together" — determined mathematically by functional dependencies, not by guesswork.

**When it's used:** Migrating legacy systems, consolidating spreadsheets into a real database, academic/GATE-style problems where you're given a flat relation and asked to normalize it.

---

### 1.10 Top-Down Design

**What is it?**
Top-down design starts from **high-level entities and relationships** (the ER model) and progressively refines them into detailed tables with attributes, keys, and constraints.

**Why does it exist?**
It mirrors how real-world systems are usually designed from scratch: you first understand *what* the business is (a university, a hospital, an e-commerce platform), identify major entities (Student, Doctor, Product), and only then flesh out the attribute-level details.

**How does it work?**
1. Identify major entities and relationships (ER diagram).
2. Refine each entity with its attributes.
3. Resolve relationships into foreign keys or junction tables.
4. Apply normalization as a *validation* step (rather than a construction step) to confirm no redundancy slipped in.

**Analogy:** Top-down is like designing a building starting from "this is a hospital, it needs wards, an OPD, a pharmacy, and a billing department" and then filling in room-level details — versus bottom-up's "here's a pile of bricks, sort them into rooms."

---

### 1.11 Bottom-Up vs Top-Down — Comparison Table

| Aspect | Bottom-Up Design | Top-Down Design |
|---|---|---|
| **Starting point** | Individual attributes | High-level entities/relationships |
| **Primary tool** | Functional dependency analysis, normalization | ER modeling |
| **Best suited for** | Reverse-engineering legacy/flat data | Greenfield system design |
| **Risk** | Can miss the "big picture" business meaning | Can miss subtle attribute-level redundancy without a normalization check |
| **Typical use case** | Data warehouse consolidation, legacy migration, GATE-style problems | New product/system design |
| **Relationship to normalization** | Normalization *drives* the design | Normalization *validates* the design |

> **In practice:** Most real-world designs use a **hybrid approach** — top-down to establish the entities and relationships quickly, then bottom-up (functional dependency + normalization analysis) to rigorously validate and refine the resulting tables. This Part gives you the bottom-up toolkit (Modules 2–13) so you can validate *any* schema, regardless of how it was initially conceived.

---

### 1.12 The Database Lifecycle

Database design doesn't happen once and then get forgotten — it's part of a lifecycle that continues for as long as the system is in production.

```
 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ Requirements  │──▶│ Conceptual &  │──▶│ Implementation│──▶│  Operation &  │──▶│ Maintenance & │
 │  Analysis     │   │ Logical Design│   │ (DDL, load    │   │  Monitoring   │   │  Evolution    │
 │               │   │ (+ Physical)  │   │  data)        │   │               │   │ (schema       │
 │               │   │               │   │               │   │               │   │  migrations)  │
 └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
        ▲                                                                              │
        └──────────────────────────────────────────────────────────────────────────────┘
                              (requirements evolve → cycle repeats)
```

1. **Requirements Analysis** — talk to stakeholders, gather functional and non-functional requirements (expected data volume, query patterns, consistency needs).
2. **Conceptual & Logical Design** — build the ER model, derive functional dependencies, normalize (this Part).
3. **Physical Design & Implementation** — write DDL, choose indexes/partitions, load initial data.
4. **Operation & Monitoring** — the system runs in production; query performance and data growth are observed.
5. **Maintenance & Evolution** — requirements change, so the schema must evolve — new columns, new tables, sometimes controlled denormalization (Module 14) for performance.

The cycle repeats because business requirements are never static. A senior engineer's job includes not just designing the *initial* schema well, but designing it so that **step 5 (evolution) is cheap and low-risk** — this is a recurring theme across Modules 14 and 15.

---

### 1.13 Design Trade-offs

Good database design is rarely about finding a single "correct" answer — it's about consciously choosing between competing goals. Here are the core trade-offs you will face repeatedly throughout this Part and in real-world design work:

| Trade-off | One Side | Other Side | Where It's Covered |
|---|---|---|---|
| **Normalization vs Performance** | Higher normal forms (3NF/BCNF) reduce redundancy | Denormalization can reduce expensive joins | Modules 7–14 |
| **Flexibility vs Simplicity** | Generic, highly flexible schemas (e.g., EAV models) | Simple, rigid schemas that are easy to reason about | Module 15 |
| **Storage vs Speed** | Storing computed/duplicated values saves recomputation | Storing only raw facts saves space, avoids staleness | Module 14 |
| **Strict Integrity vs Write Throughput** | More constraints/FKs = stronger guarantees | Fewer constraints = faster writes, riskier data | Module 15, later Parts |
| **OLTP vs OLAP shape** | Normalized schema, optimized for transactional writes | Star/snowflake schema, optimized for analytical reads | Module 17 |

**Why this matters for interviews:** Senior-level (SDE-2/SDE-3) interview questions rarely ask "normalize this to 3NF" in isolation — they ask **"design a schema for X, and justify your normalization/denormalization decisions given the expected read/write pattern."** Understanding trade-offs, not just the mechanical algorithms, is what separates a mid-level answer from a senior one.

---

### 1.14 Common Mistakes in Database Design

- **Skipping conceptual design entirely** — jumping straight to `CREATE TABLE` without understanding entities/relationships, leading to ad-hoc, inconsistent schemas.
- **Over-normalizing prematurely** — normalizing to BCNF/4NF for tables that will never see update anomalies, adding unnecessary join complexity.
- **Under-normalizing out of laziness** — leaving obvious redundancy (e.g., storing a customer's full address in every order row) "because it's easier for now."
- **Ignoring non-functional requirements** — designing a schema that's theoretically clean but ignores expected query volume, leading to a redesign under load.
- **No naming conventions** — mixing `snake_case`, `camelCase`, and abbreviations inconsistently across tables (Module 15).
- **Designing without talking to domain experts** — assuming business rules instead of confirming them, leading to a schema that can't represent real cases (e.g., assuming a student can only enroll in one course at a time).
- **No plan for schema evolution** — hardcoding assumptions (e.g., "a person has exactly one phone number") that break the moment requirements change.

---

### 1.15 Edge Cases to Keep in Mind

- **Optional relationships:** Not every entity relationship is mandatory — e.g., an `Order` might not yet have a `Shipment`. Design must accommodate `NULL`-able foreign keys or separate tables appropriately.
- **Historical/temporal data:** Business rules change over time (e.g., a product's price). A naive schema captures only the *current* state; real systems often need to preserve history (covered conceptually in Module 15 under versioning).
- **Many-to-many relationships with attributes:** When a relationship itself has attributes (e.g., `Enrollment.grade`), it *must* become its own table (junction/associative entity), not just a foreign key pair.
- **Self-referencing relationships:** An entity relating to itself (e.g., `Employee.manager_id` referencing `Employee.employee_id`) needs careful handling of nullability (top-level employees have no manager) and cycle prevention.
- **Polymorphic associations:** A table that could logically relate to *multiple* different entity types (e.g., a `Comment` that could belong to a `Post` or a `Photo`) — a genuinely hard design problem, previewed here and revisited in schema design best practices (Module 15).

---

### 1.16 Interview Importance

| Interview Level | What's Expected |
|---|---|
| **Beginner (0–1 YOE)** | Define conceptual/logical/physical design; explain why design matters (anomalies, redundancy) |
| **Intermediate (1–3 YOE)** | Given a scenario, sketch an ER diagram and derive a logical schema; explain bottom-up vs top-down |
| **Senior (SDE-2/SDE-3)** | Design a schema for a real system (e.g., "design a ride-sharing database") end-to-end, explicitly reasoning about normalization vs performance trade-offs, evolution, and edge cases like polymorphic associations or historical data |
| **GATE** | Definitions, matching conceptual/logical/physical stages to their correct descriptions, identifying anomalies in a given flat relation |

---

### 1.17 Chapter Summary

- Database design translates real-world requirements into a formal, efficient, maintainable schema.
- It proceeds in three stages: **Conceptual** (ER model) → **Logical** (normalized relational schema — the focus of this Part) → **Physical** (indexing, storage, performance tuning).
- Skipping proper design leads to redundancy and anomalies (formalized in Module 7).
- Schemas can be built **bottom-up** (from attributes, using functional dependencies) or **top-down** (from entities, using ER modeling) — real-world practice usually blends both.
- The **database lifecycle** doesn't end at initial design; schemas must evolve, and good design makes that evolution cheap.
- Every design decision involves **trade-offs** — normalization vs performance being the most recurring one in this Part.

---

### 1.18 Cheat Sheet

```
CONCEPTUAL   → ER Diagram → entities, relationships, cardinality → business language
LOGICAL      → Tables, keys, FDs → normalized schema → DBMS-independent  ★ THIS PART ★
PHYSICAL     → Indexes, partitions, storage engine → DBMS-specific → performance

BOTTOM-UP    → attributes → FDs → normalize → tables
TOP-DOWN     → entities → relationships → refine → tables → validate via normalization

ANOMALIES (preview, full detail in Module 7):
  Update Anomaly  → same fact stored twice, updated inconsistently
  Insert Anomaly  → can't add a fact without an unrelated one
  Delete Anomaly  → deleting one fact wipes out another unintentionally

DESIGN GOALS: Correctness, Non-redundancy, Consistency, Efficiency,
              Simplicity, Scalability, Maintainability, Security
```

---

### 1.19 MCQs

**Q1.** Which stage of database design produces a DBMS-independent, normalized relational schema?
a) Conceptual Design
b) Logical Design
c) Physical Design
d) Requirements Analysis

<details><summary>Answer</summary>b) Logical Design</details>

**Q2.** Bottom-up database design primarily relies on:
a) ER diagrams
b) Functional dependency analysis
c) Index selection
d) Storage engine configuration

<details><summary>Answer</summary>b) Functional dependency analysis</details>

**Q3.** Which of the following is a physical design decision, not a logical one?
a) Choosing a primary key
b) Deciding to add an index on a foreign key column
c) Identifying a functional dependency
d) Normalizing a relation to 3NF

<details><summary>Answer</summary>b) Deciding to add an index on a foreign key column</details>

**Q4.** A many-to-many relationship with its own attributes (e.g., `Enrollment.grade`) should be modeled as:
a) A foreign key in one of the two entity tables
b) A separate junction/associative table
c) A denormalized column in both entity tables
d) An index

<details><summary>Answer</summary>b) A separate junction/associative table</details>

**Q5.** Which trade-off is most directly addressed by denormalization (Module 14)?
a) Conceptual vs Logical design
b) Normalization vs Performance
c) Bottom-up vs Top-down design
d) Reflexivity vs Augmentation

<details><summary>Answer</summary>b) Normalization vs Performance</details>

---

### 1.20 Exercises

**Beginner:**
1. In your own words, explain the difference between conceptual, logical, and physical design.
2. List three consequences of skipping proper database design.
3. Give a real-world analogy (other than the architect one used here) for the three design stages.

**Intermediate:**
4. You are given a flat spreadsheet with columns: `student_name, student_email, course_title, instructor_name, instructor_office, grade`. Identify which design approach (bottom-up or top-down) is more natural here, and explain why.
5. Sketch a simple ER diagram (entities + relationships only, no attributes) for a **Library Management System** (Book, Member, Loan).

**Advanced / Whiteboard Design:**
6. You're asked in an interview to "design the database for a food delivery app." Before writing any tables, list the conceptual-level entities and relationships you'd confirm with a domain expert, and explain what non-functional requirements (read/write ratio, expected scale) would influence your logical-vs-physical trade-offs.
7. Describe a scenario where a **polymorphic association** (Section 1.15) would arise in a real system you've used (e.g., a social media app), and propose two different ways to model it at the logical design stage. What are the trade-offs of each?

**Case Study:**
8. A startup's engineering team built their entire product on a single denormalized `orders` table containing customer name, address, product name, price, and quantity duplicated on every row — skipping conceptual and logical design entirely to "move fast." Eighteen months later, they're struggling with inconsistent customer addresses across old orders. Diagnose which anomaly(ies) this represents (you'll formally learn these in Module 7) and outline, at a high level, what a proper logical design process would have caught early.

**GATE-style:**
9. Match the following:
   - (i) Conceptual Design — (a) DDL with index definitions
   - (ii) Logical Design — (b) ER Diagram
   - (iii) Physical Design — (c) Normalized relational schema

**Interview Questions (Beginner → Senior):**
- *Beginner:* "What is database design, and why is it needed?"
- *Intermediate:* "Walk me through how you'd go from a business requirement to a working schema."
- *Senior SDE-2/SDE-3:* "Design a schema for a ride-sharing platform (riders, drivers, rides, payments). Justify your choices around normalization, and identify at least one place you'd consider denormalizing for performance, explaining why."

**Practice links for schema-design style thinking (conceptual, not SQL-syntax practice):**
- GeeksforGeeks DBMS Practice: https://www.geeksforgeeks.org/dbms/
- GateOverflow (Database Design/ER tagged questions): https://gateoverflow.in/
- (Functional dependency and normalization-specific problem sets will be linked in Modules 2–13, where they're directly applicable.)

---

**End of Module 1.**

*Next up — Module 2: Functional Dependencies (FD): the mathematical foundation that makes normalization rigorous rather than guesswork. Reply "next" / "continue" whenever you're ready, and we'll pick up from there.*
