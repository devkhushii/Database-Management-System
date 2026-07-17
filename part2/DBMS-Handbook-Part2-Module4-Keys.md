# DBMS Handbook — Part 2
## Module 4: Keys (Complete)

> **Builds on:** Module 2 (Relational Model), Module 3 (Terminology — especially "candidate value" vs. "candidate key" distinction).
>
> **Feeds into:** Module 5 (Integrity Constraints — Key Constraint, Entity Integrity, Referential Integrity all build directly on keys), Module 13 (ER-to-Relational Mapping, where keys are chosen during conversion).

---

## Table of Contents

1. [Why Keys Exist — The Big Picture](#1-why-keys-exist--the-big-picture)
2. [The Key Hierarchy (Overview Diagram)](#2-the-key-hierarchy-overview-diagram)
3. [Super Key](#3-super-key)
4. [Candidate Key](#4-candidate-key)
5. [Primary Key](#5-primary-key)
6. [Alternate Key](#6-alternate-key)
7. [Composite Key](#7-composite-key)
8. [Compound Key](#8-compound-key)
9. [Foreign Key](#9-foreign-key)
10. [Natural Key](#10-natural-key)
11. [Surrogate Key](#11-surrogate-key)
12. [Secondary Key](#12-secondary-key)
13. [Unique Key](#13-unique-key)
14. [Partial Key](#14-partial-key)
15. [Business Key](#15-business-key)
16. [Natural vs. Surrogate Keys — The Great Debate](#16-natural-vs-surrogate-keys--the-great-debate)
17. [Composite vs. Compound Key — Resolving the Ambiguity](#17-composite-vs-compound-key--resolving-the-ambiguity)
18. [Master Comparison Table](#18-master-comparison-table)
19. [Chapter Summary](#19-chapter-summary)
20. [Cheat Sheet](#20-cheat-sheet)
21. [Exercises](#21-exercises)

---

## 1. Why Keys Exist — The Big Picture

### The Core Problem

Recall from Module 2, §16 that **SQL tables allow duplicate rows by default** — nothing inherently stops two identical rows from existing. But real-world data almost always needs a way to answer: *"Give me the one and only record for **this specific** entity."*

**Keys exist to solve the identification problem**: they are attribute(s) that let you uniquely, reliably pick out **exactly one tuple** from a relation, every single time.

### Why This Matters Practically

Without keys:
- You cannot reliably `UPDATE` or `DELETE` a single specific row (a "delete WHERE name = 'Rahul'" could delete multiple people named Rahul).
- You cannot establish reliable **relationships between tables** (foreign keys, §9, depend entirely on the referenced table having a reliable key to point to).
- You cannot enforce **entity integrity** (Module 5) — the rule that "every entity must be individually identifiable."

### Real-World Analogy

A key is like a **national ID number, passport number, or vehicle license plate** — a value (or combination of values) guaranteed to pick out exactly one real-world entity, even when names, addresses, or other details might be shared by multiple people/things.

---

## 2. The Key Hierarchy (Overview Diagram)

Before diving into each key type individually, see how they all relate as a **nested hierarchy of increasingly specific concepts**:

```
                         SUPER KEY
                  (any attribute set that
                   uniquely identifies a tuple —
                   including redundant ones)
                            │
                            │  (remove redundant attributes)
                            ▼
                       CANDIDATE KEY
              (minimal super key — no attribute
               can be removed without losing uniqueness)
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
        PRIMARY KEY                 ALTERNATE KEY(S)
    (the ONE candidate key      (every other candidate key
     chosen to officially        NOT chosen as primary)
     identify each row)
              │
   ┌──────────┼──────────┬─────────────────┐
   ▼          ▼          ▼                 ▼
NATURAL   SURROGATE   COMPOSITE         (any of the above
  KEY        KEY      / COMPOUND         can ALSO be a
(real-world (artificial,    KEY          "Business Key" if
 attribute)  meaningless   (made of 2+   it has business
              ID)          attributes)   meaning — see §15)

  Independently:
  FOREIGN KEY   → a key from another table, used to link relations (§9)
  UNIQUE KEY    → enforces uniqueness, but NOT chosen as the primary key (§13)
  SECONDARY KEY → any attribute(s) commonly used for lookups/searches (§12)
  PARTIAL KEY   → used in ER modeling, for weak entities (§14, revisited in Module 8)
```

> 💡 This diagram alone answers a huge fraction of "explain the difference between X key and Y key" interview questions — study it until you can redraw it from memory.

---

## 3. Super Key

### Definition

A **super key** is **any set of one or more attributes** whose values, taken together, **uniquely identify a tuple** in a relation. A super key may contain **extra, unnecessary attributes** beyond what's strictly needed for uniqueness.

### Why It Exists

Super key is the **broadest, most permissive concept** in the key hierarchy — it exists as the starting point from which we later narrow down to more useful, minimal keys (candidate keys, §4).

### Example

```
Student(roll_no, email, name, dept)
```

All of the following are valid super keys (assuming `roll_no` and `email` are each individually unique per student):

- `{roll_no}`
- `{email}`
- `{roll_no, name}` ← still a super key (uniqueness holds), even though `name` is unnecessary
- `{roll_no, email, name, dept}` ← the entire attribute set is always trivially a super key

### Key Insight

> **Every relation always has at least one trivial super key: the set of ALL its attributes.** (If two tuples agreed on every single attribute, they'd be an illegal duplicate tuple per relational theory, Module 2 §16.)

### Advantages / Why We Care

- Super key is the conceptual **starting point** for deriving candidate keys — understanding it makes candidate keys make sense (§4).

### Common Mistakes

- Thinking a super key must be *minimal* — it explicitly does **not** have to be; that's what distinguishes it from a candidate key.

### Interview Importance

**Medium** — mostly tested as a stepping stone toward understanding candidate keys, rarely the main focus of a question by itself.

---

## 4. Candidate Key

### Definition

A **candidate key** is a **minimal super key** — a super key with **no redundant attributes**. Removing even one attribute from a candidate key would cause it to lose its uniqueness property.

### Why It Exists

Candidate keys represent all the **genuinely viable options** for identifying a row — every candidate key is a *candidate* for being chosen as the table's official Primary Key (§5).

### Example

```
Student(roll_no, email, name, dept)
```

If both `roll_no` and `email` are independently guaranteed unique:

- `{roll_no}` → Candidate Key (minimal, unique)
- `{email}` → Candidate Key (minimal, unique)
- `{roll_no, name}` → **NOT** a candidate key — it's a super key, but not minimal, since `{roll_no}` alone already guarantees uniqueness, making `name` redundant

### Formal Test for "Minimal"

A super key `K` is a candidate key **if and only if** no proper subset of `K` is also a super key.

```
Is {roll_no, name} minimal?
  → Remove "name": is {roll_no} still unique? YES.
  → Therefore {roll_no, name} is NOT minimal → NOT a candidate key.

Is {roll_no} minimal?
  → Remove "roll_no": is {} (empty set) unique? NO (can't identify anyone with nothing).
  → Therefore {roll_no} IS minimal → IS a candidate key.
```

### A Relation Can Have Multiple Candidate Keys

This is extremely common and important: `Student` above has **two** candidate keys — `{roll_no}` and `{email}`. Only one will be chosen as the Primary Key (§5); the other becomes an Alternate Key (§6).

### Advantages

- Identifies **all** legitimate options for uniquely identifying a row — giving the designer real choice in picking the best Primary Key.

### Common Mistakes

- Assuming a table can have only one candidate key — false; multiple are common.
- Confusing "candidate key" with "candidate value" (Module 3, §7) — completely unrelated concepts sharing only the word "candidate."

### Interview Importance

**Very High.** "What's the difference between a super key and a candidate key?" is one of the single most frequently asked DBMS interview questions at every level.

---

## 5. Primary Key

### Definition

A **primary key (PK)** is the **one candidate key** formally **chosen by the database designer** to be the official, permanent unique identifier for every tuple in a relation.

### Why It Exists

While a relation may have several candidate keys, a DBMS needs **exactly one designated identifier** to:
- Serve as the default target for foreign key references from other tables (§9).
- Be automatically indexed for fast lookups (covered in the Indexing module of a later Part).
- Enforce **Entity Integrity** (Module 5) — a formal rule stating a primary key can **never be NULL**.

### Example

```sql
CREATE TABLE Student (
    roll_no INT PRIMARY KEY,
    email   VARCHAR(100) UNIQUE,   -- alternate key (§6)
    name    VARCHAR(50),
    dept    VARCHAR(10)
);
```

Here, `roll_no` was chosen as Primary Key; `email`, though also a valid candidate key, becomes an **Alternate Key** (§6), enforced merely with a `UNIQUE` constraint.

### Properties of a Primary Key (Critical, Frequently Tested)

| Property | Explanation |
|---|---|
| **Uniqueness** | No two tuples may share the same primary key value |
| **Not Null (Entity Integrity)** | A primary key value can **never** be NULL — an entity without an identifier is meaningless |
| **Immutability (best practice, not a hard rule)** | A primary key value *should* rarely or never change after creation, since other tables may reference it via foreign keys |
| **Minimality** | Inherited from being a candidate key — no wasted/redundant attributes |
| **Uniquely one per table** | A table can have only ONE primary key (though it may consist of multiple attributes — a composite primary key, §7) |

### How to Choose a Good Primary Key

| Criterion | Good Primary Key | Bad Primary Key |
|---|---|---|
| Stability | Never changes (e.g., surrogate ID) | Changes often (e.g., email, phone number) |
| Simplicity | Single column, simple type (e.g., INT) | Composite, complex, or long strings |
| Meaninglessness (often preferred) | No business meaning (surrogate) — avoids future conflicts | Business-meaningful (natural key) — risk if business rules change |

*(This table foreshadows the full Natural vs. Surrogate Key debate in §16.)*

### Advantages

- Guarantees every row is uniquely and permanently addressable.
- Automatically indexed by virtually all RDBMSs, enabling fast lookups and joins.
- Provides the anchor point for referential integrity (foreign keys, Module 5).

### Disadvantages / Trade-offs

- Choosing a poor primary key (e.g., a mutable natural key like email) can cause cascading update problems across every table that references it via foreign key.
- Composite primary keys (§7) can make foreign key relationships and joins more verbose in child tables.

### Common Mistakes

- Using a mutable, real-world attribute (like email or phone number) as a primary key without considering what happens when that value legitimately needs to change.
- Forgetting that a primary key **automatically implies both `UNIQUE` and `NOT NULL`** — students sometimes redundantly add `NOT NULL UNIQUE` to a column already marked `PRIMARY KEY`.

### Interview Importance

**Extremely High.** Primary key design (natural vs. surrogate, single vs. composite) is one of the most common practical schema-design interview discussion points at every seniority level.

---

## 6. Alternate Key

### Definition

An **alternate key** is any **candidate key that was NOT chosen** as the primary key. It still uniquely identifies each tuple, but plays a secondary/backup identification role.

### Why It Exists

Alternate keys exist to formally acknowledge: *"This attribute is also perfectly capable of uniquely identifying a row — we just picked a different one as the 'official' identifier."* They are typically still enforced with a `UNIQUE` constraint to preserve their uniqueness guarantee.

### Example

Continuing the `Student` example:
```sql
roll_no  → chosen as PRIMARY KEY
email    → ALTERNATE KEY, enforced via UNIQUE constraint
```

### Real-World Analogy

If your Aadhaar number (India) or SSN (US) is used as your "official" system identifier (primary key), your passport number might still uniquely identify you too — it's an alternate key: valid, unique, just not the one chosen as "official."

### Advantages

- Provides a backup uniqueness guarantee and an additional fast lookup path (since `UNIQUE` constraints are typically indexed too).
- Useful for user-facing lookups (e.g., "log in by email" while internally the system uses `roll_no`/`user_id` for joins).

### Common Mistakes

- Forgetting to actually enforce alternate keys with a `UNIQUE` constraint — leaving them as "candidate keys in theory" but unenforced in practice, allowing accidental duplicates to creep in.

### Interview Importance

**Medium-High.** Often tested as: "Table X has two columns that could each serve as an ID. You can only have one Primary Key — what do you call the other, and how do you still protect its uniqueness?"

---

## 7. Composite Key

### Definition

A **composite key** is a **primary key (or candidate key) made up of two or more attributes**, where **no single attribute alone** is sufficient to guarantee uniqueness — only the *combination* is unique.

### Why It Exists

Some entities are only meaningfully unique when described by multiple facts together — especially common in **associative/junction tables** representing many-to-many relationships (a concept formally introduced in Module 10, and mapped in Module 13).

### Example

```sql
CREATE TABLE Enrollment (
    roll_no    INT,
    course_id  INT,
    grade      CHAR(2),
    PRIMARY KEY (roll_no, course_id)   -- composite primary key
);
```

Neither `roll_no` alone (a student takes many courses) nor `course_id` alone (a course has many students) is unique by itself — but the **pair** `(roll_no, course_id)` uniquely identifies one specific enrollment record.

```
Enrollment
+----------+------------+-------+
| roll_no  | course_id  | grade |
+----------+------------+-------+
| 101      | CS101      | A     |
| 101      | CS102      | B     |   ← roll_no repeats (fine)
| 102      | CS101      | A     |   ← course_id repeats (fine)
+----------+------------+-------+
        ↑ but (roll_no, course_id) pair never repeats
```

### Advantages

- Naturally and precisely models many-to-many relationship records without needing an artificial extra ID column.
- Directly enforces the real business rule ("a student can enroll in a specific course only once") at the schema level.

### Disadvantages

- Foreign keys from other tables referencing a composite-keyed table must reference **all** component columns — increases verbosity and join complexity.
- Slightly larger index size compared to a single surrogate integer key.

### Best Practices

- Very common and often preferred for pure junction/associative tables (Module 13) where the combination genuinely *is* the natural identity of the record.
- Consider a surrogate key instead (§11) if the junction table itself needs to be referenced by yet other tables, or needs its own additional attributes with independent lifecycle.

### Interview Importance

**High.** Composite keys are central to correctly designing many-to-many relationship tables — a near-guaranteed topic in schema design interviews.

---

## 8. Compound Key

### Definition, and the Terminology Nuance

A **compound key** is closely related to (and often used interchangeably with) "composite key" — both refer to a **multi-attribute key**. However, many textbooks draw a subtle distinction:

> **Compound Key (stricter definition):** A multi-attribute key where **each individual component attribute is *also* independently a primary/foreign key in its own originating table** — i.e., the compound key is built by combining two (or more) *other tables' primary keys*.
>
> **Composite Key (broader definition):** ANY multi-attribute key, regardless of whether the individual components are independently meaningful keys elsewhere.

### Worked Example Showing the Distinction

```sql
-- roll_no is independently the PK of Student
-- course_id is independently the PK of Course
CREATE TABLE Enrollment (
    roll_no    INT REFERENCES Student(roll_no),
    course_id  INT REFERENCES Course(course_id),
    grade      CHAR(2),
    PRIMARY KEY (roll_no, course_id)
);
```

Under the stricter definition, `(roll_no, course_id)` here is technically a **Compound Key**, because both `roll_no` and `course_id` are *each independently* primary keys of their own source tables (`Student` and `Course`).

Contrast with:

```sql
-- neither part_no nor warehouse_zone is independently a PK anywhere else
CREATE TABLE Inventory (
    part_no         VARCHAR(10),
    warehouse_zone  VARCHAR(5),
    quantity        INT,
    PRIMARY KEY (part_no, warehouse_zone)
);
```

This is a **Composite Key** but arguably *not* a strict "compound key," since `warehouse_zone` isn't independently a primary key elsewhere by itself (it might just be a business-defined code).

### Practical Guidance for Interviews

> ⚠️ **Important honesty note:** In real-world practice and in the *vast majority* of textbooks, "composite key" and "compound key" are used **interchangeably** and this strict distinction is not universally agreed upon or consistently applied. If an interviewer asks you the difference, the **safest, most broadly correct answer** is: *"They're generally treated as synonyms — a key composed of two or more attributes. Some sources reserve 'compound key' specifically for when each component is independently a key from another table, but this distinction isn't consistently used across the industry."* This shows both knowledge of the nuance AND appropriate epistemic humility.

### Interview Importance

**Low-Medium.** Mostly a definitional/vocabulary question; being aware the terms overlap (and why) is more valuable than memorizing a rigid distinction not everyone agrees on.

---

## 9. Foreign Key

### Definition

A **foreign key (FK)** is an attribute (or set of attributes) in one relation that **refers to the primary key (or a unique/candidate key) of another (or the same) relation**. It is the mechanism that formally establishes **relationships between tables** in the relational model.

### Why It Exists

Recall Module 2, §2: the relational model represents *everything* — including relationships — as relations linked via shared attribute values. The foreign key is the concrete, enforced mechanism for this linkage, and is the formal basis of **Referential Integrity** (Module 5).

### Example

```sql
CREATE TABLE Department (
    dept_id    INT PRIMARY KEY,
    dept_name  VARCHAR(50)
);

CREATE TABLE Student (
    roll_no  INT PRIMARY KEY,
    name     VARCHAR(50),
    dept_id  INT,
    FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
);
```

```
Department                    Student
+---------+-----------+       +---------+--------+---------+
| dept_id | dept_name |       | roll_no | name   | dept_id |
+---------+-----------+       +---------+--------+---------+
| D1      | CSE       | ◄───┐ | 101     | Aditi  | D1      |
| D2      | ECE       | ◄─┐ └─| 102     | Rohan  | D1      |
+---------+-----------+   └───| 103     | Kavya  | D2      |
                               +---------+--------+---------+
```

### Key Rules Governing Foreign Keys

| Rule | Explanation |
|---|---|
| **Referenced column must be a key** | A foreign key must reference a `PRIMARY KEY` or `UNIQUE` column in the target table — never a non-key column |
| **Domain compatibility** | The foreign key's data type must match the referenced column's data type |
| **NULL is allowed (usually)** | Unlike a primary key, a foreign key **can be NULL** — meaning "this relationship is currently unassigned" (e.g., an employee with no manager yet) |
| **Referential Integrity** | Every non-NULL foreign key value **must** match an existing value in the referenced table — enforced automatically by the DBMS (Module 5) |
| **Self-referencing FKs are valid** | A table can have a foreign key referencing its *own* primary key (e.g., `Employee.manager_id → Employee.emp_id`) — this models recursive relationships (previewed in Module 9) |

### What Happens on Delete/Update of the Referenced Row?

This is a **very frequently tested practical topic**:

| Referential Action | Behavior |
|---|---|
| `ON DELETE CASCADE` | Deleting the parent row automatically deletes all referencing child rows |
| `ON DELETE SET NULL` | Deleting the parent row sets the child's foreign key to NULL |
| `ON DELETE RESTRICT` / `NO ACTION` | Deleting the parent row is **blocked** if any child rows still reference it |
| `ON DELETE SET DEFAULT` | Sets the child's foreign key to a predefined default value |

```sql
FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
    ON DELETE CASCADE
    ON UPDATE CASCADE;
```

> ⚠️ **Common Interview Trap:** `ON DELETE CASCADE` is powerful but dangerous — accidentally deleting one "root" row can trigger a chain reaction deleting thousands of dependent rows. Senior interviews often probe whether a candidate understands this risk and would default to `RESTRICT` in ambiguous, high-stakes production scenarios.

### Advantages

- Enforces data consistency **automatically**, without relying on application code to remember the rule.
- Makes relationships between tables explicit, discoverable, and self-documenting directly in the schema.
- Enables the DBMS's query optimizer to understand and leverage relationships for smarter join execution plans.

### Disadvantages

- Adds overhead on every insert/update/delete (the DBMS must check referential integrity).
- Can complicate bulk data loading (e.g., must insert parent rows before child rows, or temporarily disable constraint checks).
- In distributed/microservice architectures, cross-service foreign keys aren't directly enforceable by a single DBMS — a major real-world architectural trade-off (see "Industry Usage" below).

### Industry Usage

Foreign keys are standard within a single relational database. However, in **microservice architectures**, where each service often owns its own database, foreign-key-style relationships across service boundaries are typically enforced **at the application level**, not the database level — a well-known trade-off discussed frequently in system design interviews (denormalized data + eventual consistency vs. strict DB-level referential integrity).

### Common Mistakes

- Forgetting to index foreign key columns — most RDBMSs do NOT automatically index foreign keys the way they index primary keys, leading to slow joins at scale (a real, frequently-tested performance gotcha).
- Using `ON DELETE CASCADE` carelessly in systems with deep, multi-level relationships.

### Interview Importance

**Extremely High.** Foreign keys, referential integrity, and cascade behavior are core to nearly every schema design and SQL interview.

---

## 10. Natural Key

### Definition

A **natural key** is a key formed from an attribute (or attributes) that **already exists in the real world / business domain** and has inherent business meaning — as opposed to being artificially generated purely for database purposes.

### Examples

- `email` (uniquely identifies a user in many systems)
- `ISBN` (uniquely identifies a book)
- `SSN` / `Aadhaar number` (uniquely identifies a person, government-issued)
- `vehicle_registration_number` (uniquely identifies a vehicle)

### Why It Exists / Why You Might Use It

Natural keys carry inherent meaning, so they can sometimes eliminate an extra lookup — e.g., if you already have a user's email, you don't need to first look up their internal surrogate ID.

### Disadvantages (Why Natural Keys Are Often Avoided as Primary Keys)

- **Real-world values can change:** a person's legal name, email, or even (rarely) SSN can change or be reissued — but if hundreds of foreign keys across the database reference that value, changing it becomes a painful, error-prone, cascading update.
- **Real-world values can have data quality issues:** typos, inconsistent formatting (e.g., phone numbers with/without country codes), or even outright fraud (fake SSNs).
- **Not always guaranteed unique in practice**, despite theoretical uniqueness (e.g., two companies briefly issuing overlapping ISBNs due to an administrative error).

### Interview Importance

**High**, specifically as the counterpart in the Natural vs. Surrogate debate — see the full dedicated comparison in **§16**.

---

## 11. Surrogate Key

### Definition

A **surrogate key** is an **artificially generated identifier** (typically an auto-incrementing integer or a UUID) that has **no inherent business meaning whatsoever** — it exists purely to uniquely and stably identify a row within the database.

### Examples

```sql
CREATE TABLE Student (
    student_id  INT AUTO_INCREMENT PRIMARY KEY,  -- surrogate key
    roll_no     VARCHAR(10) UNIQUE,               -- natural/business key, alternate key
    name        VARCHAR(50)
);
```

Common surrogate key formats:
- Auto-incrementing integers (`SERIAL` in PostgreSQL, `AUTO_INCREMENT` in MySQL, `IDENTITY` in SQL Server)
- **UUIDs** (Universally Unique Identifiers) — widely used in distributed systems since they can be generated independently on different machines without coordination, avoiding collision (a major reason UUIDs are popular in microservice/distributed architectures)

### Why It Exists

Surrogate keys exist to provide **stability and simplicity**: since they carry no business meaning, they never *need* to change due to business rule changes, and they're typically small, fixed-size, and fast to index/compare (especially integers).

### Advantages

- **Stability:** Never needs to change, regardless of business rule changes.
- **Simplicity/Performance:** Small, fixed-size integers are extremely fast for indexing, joining, and comparison — much faster than comparing long strings like emails.
- **Decoupling:** The database's internal identification scheme is fully decoupled from real-world business data, so business data can be corrected/updated freely without touching relationships.

### Disadvantages

- Carries **no inherent meaning** — you can't tell anything about the row just by looking at its ID (a support engineer can't glance at `student_id = 48213` and know anything useful without a lookup).
- Auto-incrementing integer surrogate keys can leak information (e.g., sequential order reveals approximately how many rows/customers/orders exist — a minor but real security/privacy consideration, sometimes mitigated by using UUIDs instead).
- Requires an extra `UNIQUE` constraint on the natural key anyway if you still want to prevent real-world duplicates (e.g., you'd still want `email UNIQUE` even if `user_id` is the surrogate primary key).

### Industry Usage

**Very common** in modern application development — most ORMs (Hibernate, Django ORM, ActiveRecord, Sequelize) default to auto-generating a surrogate `id` primary key for every table. UUIDs are especially favored in distributed/microservice systems (Module on Distributed Databases, later Part) because they can be generated without a central coordinating authority.

### Interview Importance

**Extremely High** — see the full head-to-head comparison next.

---

## 12. Secondary Key

### Definition

A **secondary key** is any attribute (or set of attributes) that is **not** the primary key but is **frequently used for searching, filtering, or sorting** data — typically indexed separately to speed up those common queries.

### Example

```sql
CREATE TABLE Employee (
    emp_id      INT PRIMARY KEY,      -- primary key
    last_name   VARCHAR(50),          -- secondary key candidate
    department  VARCHAR(30)           -- secondary key candidate
);

CREATE INDEX idx_lastname ON Employee(last_name);  -- speeds up frequent lookups by last name
```

### Why It Exists

Not every frequently-searched column is unique (unlike candidate/alternate keys) — `last_name` might be searched constantly ("find all employees named Sharma") but many employees can share it. A secondary key formalizes: *"this isn't necessarily unique, but it's important enough for lookups that we index it."*

### Secondary Key vs. Candidate/Alternate Key

| | Candidate/Alternate Key | Secondary Key |
|---|---|---|
| Uniqueness required? | Yes | No |
| Purpose | Identification | Fast search/lookup/filtering |
| Typically indexed? | Yes (usually automatically) | Yes (usually manually added) |

### Interview Importance

**Medium.** Comes up mostly in performance/indexing-focused interview discussions rather than pure schema-design questions.

---

## 13. Unique Key

### Definition

A **unique key** is any attribute (or set of attributes) constrained via a `UNIQUE` constraint to **never contain duplicate values** across the table — but, unlike a primary key, it **may allow NULL values** (and typically allows multiple NULLs, since NULL is never considered equal to another NULL).

### Example

```sql
CREATE TABLE Student (
    student_id  INT PRIMARY KEY,
    email       VARCHAR(100) UNIQUE,   -- unique key
    phone       VARCHAR(15) UNIQUE     -- unique key
);
```

### Unique Key vs. Primary Key — The Critical Distinction

| Aspect | Primary Key | Unique Key |
|---|---|---|
| NULL allowed? | **Never** | **Yes** (usually, and often multiple NULLs allowed) |
| How many per table? | Exactly **one** | **Multiple** allowed |
| Used as default FK target? | Yes, typically | Can be, but less common |
| Implies candidate key status? | Yes, and it's THE chosen one | Yes, it's an unchosen candidate/alternate key |

> ⚠️ **Frequently Tested Nuance:** Whether multiple NULLs are allowed in a `UNIQUE` column, and how many, is technically **implementation-defined** across different RDBMSs (most, including PostgreSQL and MySQL, treat each NULL as distinct from every other NULL, so multiple NULLs are allowed in a UNIQUE column — but always verify against the specific system in a real project).

### Interview Importance

**High.** "Difference between Primary Key and Unique Key" is one of the most classic, frequently repeated DBMS interview questions across all experience levels.

---

## 14. Partial Key

### Definition

A **partial key** (also called a **discriminator**) is a set of attributes that can uniquely identify tuples of a **weak entity** *only when combined with the primary key of its owning (strong) entity*. On its own, a partial key is **not sufficient** for unique identification.

> **Note:** Partial keys are primarily an **ER modeling** concept (weak entities are formally covered in **Module 8**). They are introduced briefly here for completeness of the key taxonomy, and revisited in full context in Module 8.

### Example (Preview)

```
Dependent (a weak entity, existing only in relation to an Employee)
  Partial Key: dependent_name  (NOT unique alone — multiple employees
                                 could each have a dependent named "Rahul")

  Full identification requires: (employee_id, dependent_name)
                                  ↑ owning entity's PK  ↑ partial key
```

### Why It Exists

Some entities (like a dependent/family member in an HR system, or an order line-item in an order) genuinely **cannot exist independently** and have **no globally unique identifier of their own** — they're only ever meaningfully identified *in the context of* their parent/owner entity. The partial key captures "what distinguishes this entity **among siblings under the same parent**," while the owning entity's key provides the rest.

### Interview Importance

**Medium**, and mostly discussed together with weak entities — see the full treatment in **Module 8**.

---

## 15. Business Key

### Definition

A **business key** is any key (natural or otherwise) that has **direct, real-world business significance** — it's a value the business itself uses, recognizes, and cares about, independent of the database's internal implementation.

### Relationship to Other Key Types

> **Important clarification:** "Business Key" is **not a separate structural category** like "primary" or "foreign" — it's a **descriptive label** for any key (usually a natural key, §10) that has meaning *to the business*, as opposed to a surrogate key (§11), which has meaning only *to the database*.

```
Business Key examples:
  - invoice_number (the business literally prints/references this)
  - employee_badge_number
  - order_confirmation_code (shown to customers)

NOT typically business keys:
  - auto-incrementing internal `id` columns (surrogate — meaningless to the business)
```

### Why It Matters in Practice

In real systems, it's extremely common to have **both**: a surrogate key for internal database efficiency and stability, AND a business key for user-facing, audit, and cross-system-integration purposes.

```sql
CREATE TABLE Invoice (
    invoice_id      INT AUTO_INCREMENT PRIMARY KEY,     -- surrogate key (internal)
    invoice_number  VARCHAR(20) UNIQUE NOT NULL,        -- business key (external, printed on documents)
    amount          DECIMAL(10,2)
);
```

### Interview Importance

**Medium-High**, particularly in Staff+/architecture-level interviews discussing system integration, auditability, and data warehousing (business keys are especially critical in data warehouse "slowly changing dimension" design, a topic in later, more advanced database modules).

---

## 16. Natural vs. Surrogate Keys — The Great Debate

This is one of the most substantive, opinion-eliciting schema-design discussions in real-world database engineering, and a favorite topic in Senior/Staff-level interviews.

### Side-by-Side Comparison

| Criterion | Natural Key | Surrogate Key |
|---|---|---|
| **Meaning** | Has real-world business meaning | No inherent meaning |
| **Stability** | Can change (emails, names, codes get reissued/corrected) | Never needs to change |
| **Performance** | Often larger/slower to index (strings) | Typically small integers — fast |
| **Readability for debugging** | Immediately meaningful (`email = 'x@y.com'`) | Meaningless without a lookup (`user_id = 48213`) |
| **Risk of duplication across systems** | Lower (already globally meaningful, e.g., ISBN) | Higher, unless UUIDs are used |
| **Merging data from multiple sources** | Can cause natural collisions (two source systems both have "customer #1001" meaning different people) | Cleaner (regenerate/reassign surrogate IDs during merge) |
| **Best for** | Reference/lookup tables with stable, well-governed codes (e.g., ISO country codes) | Nearly everything else, especially high-churn transactional tables |

### The Modern Industry Consensus (with appropriate nuance)

> Most modern production systems **default to surrogate keys** for primary keys (auto-increment or UUID), while still enforcing natural/business keys as **unique alternate keys** — giving you the stability benefits of surrogates *and* the data-quality enforcement of natural key uniqueness. This "best of both worlds" pattern (seen in the Business Key example above) is widely considered current best practice, though reasonable engineers disagree on edge cases (e.g., small, stable reference tables like `Country(country_code CHAR(2) PRIMARY KEY)` often DO use natural keys directly, since ISO country codes are extremely stable and human-readable).

### Interview Importance

**Extremely High** for mid-to-senior interviews — expect to be asked to justify a real choice, not just recite the comparison table.

---

## 17. Composite vs. Compound Key — Resolving the Ambiguity

*(Cross-reference: this was already covered in depth in §8 — repeated here briefly for the master comparison table's completeness.)*

The safest interview-ready summary: **treat them as synonyms in casual usage** (a key of 2+ attributes), while being aware some sources reserve "compound key" for the specific case where each component is independently a primary key of another table (§8's Enrollment example).

---

## 18. Master Comparison Table

A single, revision-ready table covering all thirteen key types from this module:

| Key Type | Unique? | Can be NULL? | # per table | Core Purpose |
|---|---|---|---|---|
| Super Key | Yes | Depends on attributes | Many | Any uniquely-identifying attribute set (possibly redundant) |
| Candidate Key | Yes | Depends on attributes | Usually several | Minimal super key — a genuine option for PK |
| Primary Key | Yes | **Never** | Exactly 1 | The chosen official identifier |
| Alternate Key | Yes | Rare (usually NOT NULL enforced too) | 0 or more | Unchosen candidate key(s), still enforced |
| Composite Key | Yes (as a combination) | Individual parts may vary | N/A (a shape, not a count) | Multi-attribute uniqueness |
| Compound Key | Yes (as a combination) | Individual parts may vary | N/A | Composite key where parts are independently keys elsewhere |
| Foreign Key | No (not required) | **Yes**, usually allowed | 0 or more | Links to another table's key; enforces relationships |
| Natural Key | Yes (if chosen as PK/candidate) | Depends | N/A | Real-world, business-meaningful identifier |
| Surrogate Key | Yes | **Never** (if PK) | Usually 1 (if used as PK) | Artificial, stable, meaningless identifier |
| Secondary Key | **Not required** | Yes, can be | Many | Fast lookup/search, not necessarily unique |
| Unique Key | Yes | **Yes** (usually multiple NULLs allowed) | Many | Enforces uniqueness without being THE primary key |
| Partial Key | No (alone) | N/A | Per weak entity | Distinguishes weak-entity rows *within* an owner |
| Business Key | Depends | Depends | Depends | Descriptive label: "a key the business cares about" |

---

## 19. Chapter Summary

- **Keys exist to solve the identification problem**: reliably and uniquely picking out one tuple, since SQL tables otherwise permit duplicate rows.
- The key hierarchy flows: **Super Key** (any uniquely-identifying set, possibly redundant) → **Candidate Key** (minimal super key) → **Primary Key** (the one candidate key chosen) + **Alternate Key(s)** (the rest).
- **Composite/Compound Keys** use multiple attributes together for uniqueness — essential for correctly modeling many-to-many junction tables (previewed for Module 10/13).
- **Foreign Keys** are the formal mechanism linking relations together, enforcing **Referential Integrity** (fully covered in Module 5), with configurable `ON DELETE`/`ON UPDATE` behavior (`CASCADE`, `SET NULL`, `RESTRICT`).
- **Natural Keys** (real-world meaning) vs. **Surrogate Keys** (artificial, meaningless, stable) represent one of the most important, opinion-informed design trade-offs in real-world schema design — modern best practice often uses **both**, together.
- **Secondary Keys** aid search performance without requiring uniqueness; **Unique Keys** enforce uniqueness without being the chosen primary key (and, unlike primary keys, typically allow NULLs); **Partial Keys** support weak-entity identification (fully explored in Module 8); **Business Keys** are a descriptive label for keys with real business significance, not a separate structural category.

---

## 20. Cheat Sheet

```
KEY HIERARCHY:
  Super Key → Candidate Key → { Primary Key, Alternate Key(s) }

PRIMARY KEY:
  ✓ Unique   ✓ NOT NULL (always)   ✓ Exactly ONE per table   ✓ Usually auto-indexed

UNIQUE KEY:
  ✓ Unique   ✓ CAN be NULL (often multiple NULLs allowed)   ✓ MANY per table

FOREIGN KEY:
  → References another table's PRIMARY/UNIQUE key
  → CAN be NULL (means "not yet assigned")
  → ON DELETE: CASCADE | SET NULL | RESTRICT/NO ACTION | SET DEFAULT

NATURAL vs SURROGATE:
  Natural  = real-world meaning, CAN change, human-readable
  Surrogate = artificial (auto-increment/UUID), NEVER changes, fast, meaningless
  Best practice: surrogate as PK + natural as UNIQUE alternate key

COMPOSITE KEY = multi-attribute key (common in junction/associative tables)
COMPOUND KEY  = often synonym for composite; stricter def = parts are independently keys elsewhere

SECONDARY KEY = for fast lookup, NOT required to be unique
PARTIAL KEY   = weak entity's local discriminator (needs owner's PK to fully identify — Module 8)
BUSINESS KEY  = descriptive label for "a key the business cares about" (not a separate structural type)
```

---

## 21. Exercises

### Beginner

1. Define, in one sentence each: super key, candidate key, primary key.
2. True/False: A primary key can be NULL if the value is simply not known yet.
3. True/False: A table can have multiple candidate keys but only one primary key.
4. Fill in the blank: A key made up of two or more attributes, where no single attribute alone guarantees uniqueness, is called a __________ key.

### Intermediate

5. `User(user_id, email, phone, ssn)` — assume `email`, `phone`, and `ssn` are each independently guaranteed unique. List all candidate keys, and explain why `{user_id, email}` is a super key but NOT a candidate key.
6. Design the `Enrollment` junction table (Student ↔ Course, many-to-many) with an appropriate composite primary key, including the foreign key declarations.
7. Explain the difference between a Primary Key and a Unique Key using a concrete schema example involving a nullable column.
8. A senior engineer proposes: "Let's use the customer's email as the primary key of the `Customer` table — it's simpler." List two concrete risks of this decision.

### Advanced / Conceptual

9. Design a schema for an `Order` and `OrderItem` relationship where `Order` uses a surrogate primary key (`order_id`) but also has a business key (`order_confirmation_code`) shown to customers. Explain why both are useful.
10. Explain, with an example, what `ON DELETE CASCADE` would do if applied to a foreign key from `Enrollment.roll_no` referencing `Student.roll_no`, and discuss one risk of this configuration in a production system.
11. A weak entity `Dependent` (children of an `Employee`) has no independent unique identifier of its own. Explain how a partial key combines with the owning entity's primary key to fully identify a `Dependent` row. (Preview answer — full treatment in Module 8.)

### MCQs

**Q1.** Which of the following is always true of a primary key?
a) Can be NULL  b) Can repeat  c) Must be unique and NOT NULL  d) Must be a string

**Q2.** A candidate key is best described as:
a) Any attribute set that is unique  b) A minimal super key  c) A foreign key  d) A key with business meaning

**Q3.** Which key type is most likely to REMAIN STABLE even if a customer changes their email?
a) Natural key based on email  b) Surrogate key (auto-increment ID)  c) Business key printed on invoices  d) Secondary key

**Q4.** A foreign key can:
a) Never be NULL  b) Be NULL, meaning the relationship isn't yet assigned  c) Only reference a UNIQUE column, never a PRIMARY KEY  d) Only exist within the same table

**Q5.** In a Student-Course many-to-many relationship modeled via an Enrollment junction table, the most natural choice of primary key is typically:
a) A single new surrogate ID column only  b) roll_no alone  c) course_id alone  d) A composite key of (roll_no, course_id)

*(Answer key: 1-c, 2-b, 3-b, 4-b, 5-d)*

### Schema Design Exercise

12. Design a complete schema (in SQL DDL) for a `Library` system with `Book`, `Member`, and `Loan` tables. Explicitly identify and label, in comments, the primary key, any foreign keys, at least one natural key, and at least one surrogate key used in your design.

### Practice Links

- [LeetCode Database — problems involving PRIMARY KEY/FOREIGN KEY constraints](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL — Basic Join (practice foreign key relationships)](https://www.hackerrank.com/domains/sql)
- [DataLemur — SQL interview questions from real companies, several on key design](https://datalemur.com/questions)

---

**Next: Module 5 — Integrity Constraints (Domain, Key, Entity Integrity, Referential Integrity, NOT NULL, UNIQUE, PRIMARY KEY, FOREIGN KEY, CHECK, DEFAULT, Assertions, Business Rules).**

> Say "continue" or "next module" to proceed to Module 5.
