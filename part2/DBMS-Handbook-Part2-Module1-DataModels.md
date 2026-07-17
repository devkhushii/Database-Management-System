# DBMS Handbook — Part 2
## Module 1: Data Models

> **Prerequisite:** Part 1 (Introduction to DBMS, File Systems vs DBMS, DBMS Architecture, Three-Schema Architecture, Data Independence, DBMS Users, Advantages of DBMS).
>
> **This module builds the conceptual foundation for everything that follows in Part 2** — the Relational Model, ER Modeling, Keys, Constraints, and Schema Design all sit on top of the ideas introduced here.

---

## Table of Contents

1. [What is a Data Model?](#1-what-is-a-data-model)
2. [Why Data Models Exist](#2-why-data-models-exist)
3. [Levels of Abstraction](#3-levels-of-abstraction)
4. [Types of Data Models](#4-types-of-data-models)
5. [Object-Based Data Models](#5-object-based-data-models)
6. [Record-Based Data Models](#6-record-based-data-models)
7. [Physical Data Models](#7-physical-data-models)
8. [Conceptual, Logical, and Physical Models — Deep Comparison](#8-conceptual-logical-and-physical-models--deep-comparison)
9. [Comparison of All Data Models](#9-comparison-of-all-data-models)
10. [Chapter Summary](#10-chapter-summary)
11. [Cheat Sheet](#11-cheat-sheet)
12. [Exercises](#12-exercises)

---

## 1. What is a Data Model?

### Definition

A **Data Model** is a collection of conceptual tools — concepts, notations, and rules — used to describe the **structure of a database**: what data it holds, how pieces of data relate to one another, and what rules (constraints) that data must obey.

Think of a data model as the **blueprint language** used before you build a database, in the same way an architect uses blueprint notation before pouring concrete. The blueprint isn't the building — it's the *plan* for the building. Similarly, a data model isn't the database — it's the *plan* for the database.

Formally, a data model provides three things:

| Component | What it describes | Example |
|---|---|---|
| **Structure** | What kinds of data exist and how they're organized | "A `Student` has a roll number, name, and department" |
| **Operations** | What actions can be performed on the data | Insert a student, update a grade, delete an enrollment |
| **Constraints** | Rules the data must always satisfy | "Roll number must be unique", "Age must be ≥ 17" |

### Real-Life Analogy

Imagine you're designing a school. Before any bricks are laid, you need:

- A **floor plan** (structure) — where classrooms, labs, and the office go.
- **Usage rules** (operations) — how students move between classes, how staff access the office.
- **Safety codes** (constraints) — fire exits, maximum occupancy per room.

A data model is the same idea applied to data: it tells you *what exists*, *how it can be used*, and *what rules it must follow* — all before a single row is ever inserted into a real database.

### How It Works Internally

When a database designer builds a system, they don't write raw storage code directly. Instead, they:

1. Talk to stakeholders and gather requirements.
2. Express those requirements using a **data model** (usually the ER Model, covered in later modules).
3. Translate that data model into a **schema** (a concrete structural definition, like relational tables).
4. Let the DBMS translate the schema into physical storage (files, pages, blocks — covered in Part 1's architecture discussion and revisited in later Parts on storage/indexing).

```
Requirements  --->  Data Model  --->  Schema  --->  Physical Storage
(business needs)   (ER Model,       (CREATE TABLE   (B+ Trees, heap
                    Relational        statements)     files, pages)
                    Model, etc.)
```

### Advantages

- Provides a **common language** between business stakeholders, analysts, and engineers.
- Enables **early error detection** — modeling mistakes are far cheaper to fix on paper than after implementation.
- Supports **data independence** (recall from Part 1): the model separates *what* data means from *how* it's stored.
- Makes systems **easier to document, maintain, and extend**.

### Disadvantages / Limitations

- An overly abstract data model can miss real-world performance constraints.
- Poorly chosen data models lead to painful, expensive migrations later.
- Some data models (e.g., pure conceptual models) don't map 1:1 to how the DBMS physically stores data, so translation errors are possible.

### Industry Usage

Every major database system — relational (PostgreSQL, MySQL, Oracle), NoSQL (MongoDB, Cassandra), NewSQL (CockroachDB, Spanner) — starts life as a data model on a whiteboard before it becomes SQL/DDL or JSON schema. FAANG-scale systems maintain formal data models in tools like ER/Studio, dbdiagram.io, or internal schema registries (e.g., Protobuf/Avro schemas for streaming data).

### Interview Importance

**Very High.** Almost every DB interview begins with "How would you model X?" — this is a direct test of your understanding of data models, even if the interviewer never says the word "data model" explicitly.

> 💡 **Interview Tip:** When asked "design a database for X," always narrate your thought process as: *conceptual model → logical model → physical model*. This signals structured thinking to the interviewer.

---

## 2. Why Data Models Exist

### The Problem They Solve

Before formal data models, early computer systems stored data in **flat files** managed by application code directly (this was covered in Part 1 under "File System vs DBMS"). This caused:

- **Data redundancy** — the same fact stored in multiple places.
- **Data inconsistency** — updates in one place but not another.
- **Poor data integrity** — no formal way to express rules like "age cannot be negative."
- **Difficulty in ad-hoc querying** — every question required custom code.
- **No shared understanding** — different programmers interpreted the same file structure differently.

A data model exists to solve **the communication and structure problem**: it gives everyone — business analysts, DBAs, developers — a shared, rigorous vocabulary to describe data *before* it is built, and a way to reason about it *after* it is built.

### Why Not Just Write Code Directly?

| Without a Data Model | With a Data Model |
|---|---|
| Structure is implicit, buried in code | Structure is explicit, documented |
| Hard to detect design flaws early | Flaws caught on paper/diagram stage |
| Difficult to communicate with non-engineers | Diagrams are understandable to business stakeholders |
| Painful schema evolution | Controlled, versioned schema evolution |
| Ambiguous relationships between data | Relationships are formally declared |

### Analogy

Writing a database directly in code without a data model is like writing a novel without an outline — you *can* do it, but plot holes (design flaws) creep in, and revising structure later is far more expensive than fixing an outline.

---

## 3. Levels of Abstraction

This directly extends the **Three-Schema Architecture** from Part 1 (Internal, Conceptual, External schemas). Here we look at it through the lens of *data modeling* specifically — i.e., how the same real-world information is represented at increasingly concrete levels.

### The Three Levels

```
┌─────────────────────────────────────────────────────────┐
│  CONCEPTUAL LEVEL (Highest abstraction)                  │
│  "A Student takes a Course"                               │
│  — Business meaning, no implementation detail             │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  LOGICAL LEVEL (Mid abstraction)                          │
│  Student(roll_no, name, dept_id)                          │
│  Course(course_id, title)                                 │
│  Enrollment(roll_no, course_id, grade)                    │
│  — Structure defined, DBMS-independent                    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  PHYSICAL LEVEL (Lowest abstraction, most concrete)        │
│  B+ Tree index on roll_no, page size 8KB,                  │
│  row stored on disk block #4521                            │
│  — DBMS-specific, hardware-specific                        │
└─────────────────────────────────────────────────────────┘
```

### Why These Levels Exist

Each level answers a **different question, for a different audience**:

| Level | Question Answered | Audience | Concerned With |
|---|---|---|---|
| Conceptual | *What* data exists and *how* it relates | Business analysts, stakeholders | Meaning, entities, relationships |
| Logical | *How* is data structured (tables, columns, keys) | DB designers, developers | Schema, normalization, constraints |
| Physical | *Where and how* is data stored on disk | DBAs, storage engineers | Indexes, partitioning, file organization |

### Connection to Data Independence (Part 1 recap)

Recall **Logical Data Independence** (ability to change the logical schema without affecting external views) and **Physical Data Independence** (ability to change physical storage without affecting the logical schema). These abstraction levels are precisely *why* such independence is achievable — because each level only depends on the interface exposed by the level below it, not its internal details.

### Real-World Analogy

Think of a company org chart:

- **Conceptual**: "There are Employees, and Employees belong to Departments."
- **Logical**: The literal spreadsheet with Employee ID, Name, Department ID columns.
- **Physical**: The actual folder on a file server, its permissions, backup schedule, and disk sector it occupies.

The org chart's *meaning* stays the same regardless of which spreadsheet software or storage server is used underneath — that's the abstraction at work.

### Common Mistakes

- Conflating logical and physical design too early (e.g., choosing indexes before the logical schema is even validated with stakeholders).
- Presenting physical-level details (indexes, partitions) to business stakeholders — creates confusion, wrong level of detail.
- Skipping the conceptual level entirely and jumping straight to `CREATE TABLE` statements — leads to missed business rules.

### Interview Importance

**High.** Interviewers frequently test whether you understand *why* schema design is layered. A common trick question: *"Why not just design the physical schema directly?"* — good answer: layering enables changing storage strategy without rewriting the business logic, and enables communicating with non-technical stakeholders at the conceptual level.

---

## 4. Types of Data Models

Data models are broadly classified by **how directly they mirror the real-world structure** vs. **how directly they mirror physical storage**. Here is the full classification tree used throughout industry and academia (including GATE syllabus):

```
                         Data Models
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
 Object-Based           Record-Based            Physical
 Data Models            Data Models             Data Models
        │                     │                     │
   - ER Model            - Relational Model    - Unifying Model
   - EER Model           - Network Model       - Frame Memory Model
   - Object-Oriented     - Hierarchical Model
     Model
```

### High-Level Definitions (expanded in later sections)

| Category | Description | Examples |
|---|---|---|
| **Object-based models** | Describe data using entities, attributes, and relationships with rich structural constraints; closer to how humans conceptualize the world | ER Model, EER Model, Object-Oriented Model |
| **Record-based models** | Describe data as fixed-format records of a fixed number of fields; database consists of a fixed number of record types | Relational Model, Network Model, Hierarchical Model |
| **Physical data models** | Describe how data is actually stored — file formats, indexes, access paths | Physical schema, storage-engine-specific formats |

### Why This Classification Matters

Every DBMS you've heard of — MySQL, PostgreSQL, Oracle, MongoDB, Neo4j — implements *one or more* of these underlying models:

| System | Underlying Model |
|---|---|
| MySQL, PostgreSQL, Oracle, SQL Server | Relational Model (record-based) |
| MongoDB | Document Model (object-based, semi-structured) |
| Neo4j | Graph Model (object-based, relationship-centric) |
| IBM IMS (legacy) | Hierarchical Model (record-based) |
| IDMS (legacy) | Network Model (record-based) |

Understanding this taxonomy lets you reason about *any* database system you encounter, even ones you've never used, by asking: "What data model does this implement, and what trade-offs does that imply?"

---

## 5. Object-Based Data Models

### Definition

Object-based data models describe data using concepts close to real-world objects: **entities** (things), **attributes** (properties of things), and **relationships** (associations between things), along with rich constraints such as cardinality and generalization/specialization.

### Why They Exist

Humans naturally think in terms of "things" and "how things relate," not in terms of rows and columns. Object-based models exist to let designers capture requirements in a form that mirrors human/business thinking, *before* being forced into a rigid record structure.

### Members of This Family

#### a) Entity-Relationship (ER) Model
The most widely used object-based model, used at the **conceptual** design stage. (Covered in full depth in **Module 6** of this Part.)

- Entities: e.g., `Student`, `Course`
- Attributes: e.g., `name`, `roll_no`
- Relationships: e.g., `Enrolls`

#### b) Enhanced ER (EER) Model
Extends ER with object-oriented concepts: **generalization, specialization, aggregation, categorization** (Covered in full depth in **Module 12**).

#### c) Object-Oriented Data Model
Goes further than ER by allowing entities to have **behavior** (methods), not just structure (attributes) — mirroring object-oriented programming (classes, inheritance, encapsulation, polymorphism) directly inside the database.

```
class Student {
    attributes: roll_no, name, dept
    methods: enroll(course), calculateGPA()
}
```

Used in **Object-Oriented Database Systems (OODBMS)** like `db4o`, `ObjectDB`. Rare in mainstream industry today but conceptually important — modern **ORMs** (Object-Relational Mappers) like Hibernate, SQLAlchemy, and Django ORM exist precisely to bridge object-oriented application code with the (non-object-oriented) relational model.

### Advantages of Object-Based Models

- Naturally expressive; closely matches business requirements.
- Excellent communication tool with non-technical stakeholders.
- EER supports modeling real-world hierarchies (e.g., "a Manager IS-A Employee").

### Disadvantages

- Not directly implementable — must be translated into a record-based or physical model before use in most production RDBMSs.
- Full object-oriented databases (OODBMS) never achieved mainstream relational-level adoption due to lack of standardization and weaker query optimization tooling.

### Industry Usage

- ER/EER diagrams are used **universally** at design time, regardless of what DBMS is ultimately used.
- True OODBMS usage is niche (some CAD/CAM, telecom systems).
- **Graph databases** (Neo4j, Amazon Neptune) are a modern evolution of object-based, relationship-first thinking.

### Interview Importance

**Very High for ER Model specifically** — nearly every "design a database for X" interview question expects you to sketch or describe an ER diagram first.

---

## 6. Record-Based Data Models

### Definition

Record-based data models represent the database as a collection of **fixed-format records**. Each record type defines a fixed number of fields (attributes), each of a fixed length (or well-defined type). Unlike object-based models, record-based models are directly implementable by a DBMS — they describe data at the **logical level**.

### Why They Exist

Object-based models like ER are great for *thinking*, but a computer needs a **concrete, storable, queryable structure**. Record-based models exist as the bridge between conceptual design and an actual, working database engine.

### The Three Classical Record-Based Models

#### a) Relational Model
Data represented as **relations** (tables) — a set of tuples (rows), each conforming to a schema of attributes (columns). Proposed by **E. F. Codd in 1970**. (Full deep-dive in **Module 2** of this Part — this is the dominant model in the industry today.)

```
Student
+---------+----------+------+
| roll_no | name     | dept |
+---------+----------+------+
| 101     | Aditi     | CSE  |
| 102     | Rohan     | ECE  |
+---------+----------+------+
```

#### b) Hierarchical Model
Data organized as a **tree**: each record (child) has exactly **one parent**, forming parent-child (1:N) relationships only.

```
                 Company
                    │
        ┌───────────┴───────────┐
    Department               Department
        │                        │
   ┌────┴────┐              ┌────┴────┐
Employee  Employee       Employee  Employee
```

- **Advantage:** Very fast traversal for naturally hierarchical data (e.g., file systems, org charts).
- **Disadvantage:** Cannot naturally represent many-to-many relationships; a child having two parents requires data duplication.
- **Real-world example:** IBM's **IMS** (Information Management System, 1966) — still used in some legacy banking/insurance mainframes today. Also, XML documents are inherently hierarchical.

#### c) Network Model
Generalizes the hierarchical model: a record can have **multiple parents**, represented as a **graph** rather than a strict tree, using explicit "set" constructs to model relationships.

```
   Supplier ───┐         ┌─── Warehouse
       │       │         │        │
       └──> Shipment <────┘        
```

- **Advantage:** Can represent many-to-many relationships more naturally than the hierarchical model.
- **Disadvantage:** Navigation requires the programmer to know the exact access path (pointer-chasing) — very low-level, error-prone, and hard to maintain.
- **Real-world example:** **IDMS** (Integrated Database Management System), CODASYL-standard databases from the 1970s–80s.

### Why the Relational Model Won

| Criterion | Hierarchical | Network | Relational |
|---|---|---|---|
| Ease of use | Low | Low | High (declarative SQL) |
| Flexibility for new queries | Poor (fixed access paths) | Poor (fixed access paths) | Excellent (ad-hoc SQL queries) |
| Data independence | Weak | Weak | Strong |
| Mathematical foundation | None | None | Set theory + predicate logic |
| Handling M:N relationships | Very poor | Awkward (via sets) | Natural (via junction tables) |

This comparison itself is a **very common interview and GATE question**: *"Why did the relational model replace hierarchical/network models?"*

### Common Mistakes

- Assuming hierarchical/network models are "obsolete and irrelevant" — they still power legacy mainframe systems handling trillions of dollars in transactions (banking, airlines).
- Confusing "hierarchical model" (a 1960s DBMS architecture) with "hierarchical *data*" (e.g., an org chart, which can still be stored perfectly well in a relational table using a self-referencing foreign key — covered later in recursive relationships).

### Interview Importance

**Medium-High** — mostly asked as a *comparison* question ("why relational over hierarchical/network?") rather than requiring hands-on hierarchical/network modeling skills. GATE exams test this history/comparison directly.

---

## 7. Physical Data Models

### Definition

A **physical data model** describes exactly **how data is stored on physical storage media** — file organization, block/page structure, indexing structures, compression, partitioning, and access paths. It is the lowest, most concrete level of abstraction.

### Why It Exists

Ultimately, all data lives on physical disks (SSD/HDD) or in memory as bytes. The physical data model exists to answer: *"Given this logical schema, how do we lay it out on actual storage for correctness, speed, and durability?"*

### What a Physical Data Model Specifies

| Aspect | Example |
|---|---|
| File organization | Heap file, sorted file, hashed file |
| Record layout | Fixed-length vs. variable-length rows |
| Indexing structure | B+ Tree, Hash Index, Bitmap Index |
| Storage engine | InnoDB (MySQL), Heap/TOAST (PostgreSQL) |
| Partitioning strategy | Range partitioning, hash partitioning |
| Block/page size | Typically 4KB, 8KB, or 16KB |
| Compression | Row-level or column-level compression |

> **Note:** File organization, indexing structures (B+ Trees, hashing), and storage engines are covered in exhaustive depth in the dedicated **Storage & Indexing** module later in this handbook series — this section only introduces the *concept* of the physical model as a level of abstraction.

### Real-World Analogy

If the conceptual model is "a library has books," and the logical model is "each book has a title, author, and ISBN in a table," the physical model is "books are stored on shelf 14B, sorted by ISBN, with a card catalog (index) at the front for fast lookup."

### Advantages

- Enables performance tuning (query speed, storage cost) independent of business logic.
- Enables DBAs to optimize without needing to modify application code (physical data independence, from Part 1).

### Disadvantages

- Highly DBMS-specific and even version-specific — physical models don't transfer well between different database engines.
- Requires deep expertise; mistakes here (e.g., wrong index choice) can silently degrade performance at scale without being visible in the logical schema.

### Interview Importance

**High for Senior/Staff-level roles.** Junior interviews rarely probe here directly, but SDE-3+/Staff Engineer interviews frequently ask about index choice, partitioning strategy, and storage engine trade-offs — this is where "database design" interviews graduate into "database systems" interviews.

---

## 8. Conceptual, Logical, and Physical Models — Deep Comparison

This section consolidates conceptual/logical/physical (introduced in §3) with concrete modeling examples, since this exact three-way comparison is one of the **most frequently asked** GATE and interview questions.

### Side-by-Side Example: Modeling a "Student enrolls in Course"

| Aspect | Conceptual Model | Logical Model | Physical Model |
|---|---|---|---|
| **Representation** | ER diagram | Relational schema (tables + keys) | Storage engine DDL with indexes |
| **Example** | `Student` —Enrolls→ `Course` | `Student(roll_no PK, name)`<br>`Course(course_id PK, title)`<br>`Enrollment(roll_no FK, course_id FK, grade)` | `CREATE TABLE Enrollment (...) ENGINE=InnoDB PARTITION BY HASH(roll_no);` with a B+ Tree index on `(roll_no, course_id)` |
| **Audience** | Business stakeholders, analysts | DB designers, backend developers | DBAs, storage/performance engineers |
| **Concerned with** | Meaning and relationships | Structure, keys, normalization | Speed, storage cost, durability |
| **DBMS-dependent?** | No | Mostly no (some dialect differences) | Yes, fully |
| **Changes how often?** | Rarely (business meaning is stable) | Occasionally (as requirements evolve) | Frequently (as scale/performance needs evolve) |

### Full Design Pipeline Flowchart

```
 [Requirements Gathering]
          │
          ▼
 [Conceptual Model: ER/EER Diagram]  ◄── Module 6, 12 of this Part
          │  (mapped using rules in Module 13)
          ▼
 [Logical Model: Relational Schema, Normalized]  ◄── Modules 2-5, and
          │                                            Normalization (Part 3)
          ▼
 [Physical Model: Indexes, Partitions, Storage Engine]  ◄── Later Part
          │
          ▼
 [Running Database System]
```

### Common Mistakes

- **Skipping the conceptual stage** and designing tables directly — leads to missing business rules and poor stakeholder alignment.
- **Over-indexing at the physical stage before the logical model is validated** — wasted engineering effort if the schema later changes.
- Treating **normalization** (a logical-level concern) as though it were a physical-level concern — normalization is about eliminating redundancy/anomalies in the *logical* schema; physical denormalization for performance is a *separate*, later decision (covered in the Normalization module of Part 3).

### Interview Importance

**Very High.** This three-tier comparison is a go-to opening question for both academic exams (GATE) and system design interviews, since it demonstrates whether a candidate thinks in layers or jumps straight to implementation.

---

## 9. Comparison of All Data Models

A consolidated, exam-and-interview-ready comparison table across every model discussed in this module:

| Model | Category | Abstraction Level | Structure | Relationship Handling | Mathematical Basis | Modern Usage |
|---|---|---|---|---|---|---|
| **ER / EER Model** | Object-based | Conceptual | Entities, attributes, relationships | Explicit, flexible (1:1, 1:N, M:N) | Set theory (informal) | Universal design tool |
| **Object-Oriented Model** | Object-based | Conceptual/Logical | Classes with attributes + methods | Via object references | Object theory | Niche (OODBMS), influences ORMs |
| **Relational Model** | Record-based | Logical | Tables (relations) of tuples | Via foreign keys | Set theory + predicate logic (relational algebra/calculus) | Dominant (MySQL, PostgreSQL, Oracle, SQL Server) |
| **Hierarchical Model** | Record-based | Logical | Tree of records | Parent-child only (1:N) | Graph theory (tree) | Legacy mainframes (IMS), XML |
| **Network Model** | Record-based | Logical | Graph of records ("sets") | Multiple parents allowed | Graph theory | Legacy (IDMS), largely obsolete |
| **Physical/Storage Model** | Physical | Physical | Files, blocks, pages, indexes | Pointers, offsets | Data structures (B+ Trees, hash tables) | Universal (underlies every DBMS) |

### Quick Decision Guide (for interviews)

> *"If asked to justify a model choice, use this reasoning: object-based models (ER/EER) are best for **communication and design**; record-based models (Relational) are best for **implementation with strong data independence and ad-hoc querying**; physical models are best for **performance tuning after the logical design is stable**."*

---

## 10. Chapter Summary

- A **data model** is a set of concepts + notation used to describe database structure, operations, and constraints — it is the blueprint before the building.
- Data models exist to solve problems of **redundancy, inconsistency, ambiguity, and poor communication** that plagued raw file-based systems.
- Data is designed across **three levels of abstraction**: **Conceptual** (business meaning) → **Logical** (structure/schema) → **Physical** (storage/performance) — this mirrors and extends the Three-Schema Architecture from Part 1.
- Data models fall into three families: **Object-based** (ER, EER, Object-Oriented — for conceptual design), **Record-based** (Relational, Hierarchical, Network — for logical implementation), and **Physical** (storage engines, indexes — for physical implementation).
- The **Relational Model** won industry dominance because of its strong mathematical foundation, superior data independence, and support for flexible, declarative (SQL-based) ad-hoc querying — a theme that will be explored in full in Module 2.
- Every well-designed database follows the pipeline: **Requirements → Conceptual Model (ER/EER) → Logical Model (Relational Schema, Normalized) → Physical Model (Indexes/Partitions)**.

---

## 11. Cheat Sheet

```
DATA MODEL = Structure + Operations + Constraints

ABSTRACTION LEVELS:
  Conceptual  → What data exists & how it relates   (ER/EER)
  Logical     → How it's structured                 (Relational Schema)
  Physical    → How it's stored                      (Indexes, Files)

MODEL FAMILIES:
  Object-Based   → ER, EER, Object-Oriented   → Conceptual design
  Record-Based   → Relational, Hierarchical, Network → Logical implementation
  Physical       → Storage engines, indexes    → Physical implementation

WHY RELATIONAL WON:
  ✓ Strong math foundation (set theory, predicate logic)
  ✓ Declarative querying (SQL) — no fixed access paths
  ✓ Strong data independence
  ✓ Natural handling of M:N relationships (via junction tables)

DESIGN PIPELINE:
  Requirements → ER/EER (Conceptual) → Relational Schema (Logical) → Storage/Indexes (Physical)
```

---

## 12. Exercises

### Beginner

1. In your own words, define "data model" using a non-computing analogy other than the ones used in this module.
2. List the three levels of abstraction in data modeling and give one real-world example for each.
3. True/False: The relational model is an object-based data model.
4. Fill in the blank: The __________ model represents data as a tree where each record has exactly one parent.

### Intermediate

5. Explain why the network model, despite solving the hierarchical model's many-to-many problem, still lost out to the relational model.
6. Give two examples of modern systems that are NOT relational, and identify which data model family they belong to.
7. A junior developer says: "I'll just start writing `CREATE TABLE` statements directly — we don't need ER diagrams." Explain, using concepts from this module, two risks in this approach.

### Advanced / Conceptual

8. Compare and contrast **object-oriented data models** with modern **document-oriented NoSQL databases** (e.g., MongoDB). Are they the same idea? Why or why not?
9. Explain how **physical data independence** (Part 1) is only possible *because* of the layered abstraction (conceptual/logical/physical) discussed in this module.
10. Design scenario: You're asked to store an organization's employee hierarchy (who reports to whom). Discuss how you'd model this in (a) a hierarchical model, (b) a relational model. Which is more natural, and why?

### MCQs

**Q1.** Which of the following is an object-based data model?
a) Relational Model  b) Hierarchical Model  c) ER Model  d) Network Model

**Q2.** The physical data model is primarily concerned with:
a) Business meaning  b) Storage and access paths  c) Entity relationships  d) Stakeholder communication

**Q3.** Who proposed the Relational Model?
a) Charles Bachman  b) E. F. Codd  c) Peter Chen  d) Edgar Dijkstra

**Q4.** In the hierarchical model, each child record can have:
a) Multiple parents  b) Exactly one parent  c) Zero or more parents  d) No parent restriction

**Q5.** Which level of abstraction is closest to how a business stakeholder thinks about data?
a) Physical  b) Logical  c) Conceptual  d) Storage

*(Answer key: 1-c, 2-b, 3-b, 4-b, 5-c)*

### Diagram-Based Exercise

11. Draw (in ASCII or on paper) a simple hierarchical-model tree for a **University → Department → Professor** structure, and then redraw the same information as a relational schema (tables + foreign keys). Compare the two.

### Practice Links

- [SQLBolt — Introduction to SQL concepts](https://sqlbolt.com/)
- [HackerRank SQL — for later relational practice](https://www.hackerrank.com/domains/sql)
- *(Modeling topics are conceptual — practice via diagram exercises above rather than coding platforms; coding practice becomes directly relevant starting Module 2 onward.)*

---

**Next: Module 2 — The Relational Model (History, Mathematical Foundation, Relations, Tuples, Attributes, Domains, and more).**

> Say "continue" or "next module" to proceed to Module 2.
