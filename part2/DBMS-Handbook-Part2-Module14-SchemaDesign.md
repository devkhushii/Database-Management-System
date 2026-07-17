# DBMS Handbook — Part 2
## Module 14: Schema Design Basics

> **Builds on:** Module 13 (ER-to-Relational Mapping — this module refines the raw output of that mechanical process into a clean, professional, maintainable schema).
>
> **Feeds into:** Normalization (Part 3, which further refines schema structure for redundancy/anomaly elimination) and real-world production schema reviews.

---

## Table of Contents

1. [Why Schema Design "Basics" Still Matter After Mastering ER Mapping](#1-why-schema-design-basics-still-matter-after-mastering-er-mapping)
2. [Naming Conventions](#2-naming-conventions)
3. [Table Naming](#3-table-naming)
4. [Column Naming](#4-column-naming)
5. [Data Consistency](#5-data-consistency)
6. [Avoiding Redundancy](#6-avoiding-redundancy)
7. [Future-Proof Schemas](#7-future-proof-schemas)
8. [Scalability Considerations](#8-scalability-considerations)
9. [Full Before/After Schema Review Example](#9-full-beforeafter-schema-review-example)
10. [Chapter Summary](#10-chapter-summary)
11. [Cheat Sheet](#11-cheat-sheet)
12. [Exercises](#12-exercises)

---

## 1. Why Schema Design "Basics" Still Matter After Mastering ER Mapping

### The Gap Between "Correct" and "Good"

Module 13 taught you how to produce a **structurally correct** relational schema from an ER diagram. But structural correctness alone doesn't guarantee a schema is **pleasant, safe, and efficient to work with** at scale, across a large team, over many years. This module covers the practical, often underrated craftsmanship layer — the difference between a schema that merely *works* and one that a team can *confidently build on for a decade*.

### Real-World Analogy

Two houses can both be structurally sound (pass inspection, Module 13's equivalent) — but one has clearly labeled circuit breakers, logically grouped rooms, and wiring an electrician could safely extend in 15 years, while the other is a maze only the original builder understands. This module is about building the *first* kind of house.

---

## 2. Naming Conventions

### Why They Matter

Inconsistent naming is one of the **most common, most avoidable sources of confusion and bugs** in real production schemas — a query author has to guess (or look up) whether a column is `emailAddress`, `email_address`, or `Email`, every single time, across potentially hundreds of tables.

### Core Principles

| Principle | Explanation |
|---|---|
| **Pick ONE case style and never deviate** | `snake_case` (most common in SQL: `first_name`) or `camelCase` — but never mix both within the same database |
| **Be consistent across the ENTIRE schema** | If one table uses `created_at`, every table with a creation timestamp should use `created_at` too — not `date_created` in one table and `created_on` in another |
| **Avoid SQL reserved words** | Don't name a column/table `order`, `group`, `user` (in some systems) without quoting — causes constant, avoidable friction |
| **Prefer clarity over brevity, but avoid excessive verbosity** | `customer_id` is better than the ambiguous `cid`; but `customer_identification_number_primary` is excessive |

### Interview Importance

**Medium** — rarely the *focus* of an interview, but sloppy naming in a live schema-design exercise is a subtle, real signal that gets noticed.

---

## 3. Table Naming

### Singular vs. Plural — The Classic Debate

```
Singular style:  Student, Course, Enrollment
Plural style:    Students, Courses, Enrollments
```

> **Honest guidance:** Both styles are widely used in industry, and reasonable engineers disagree — **the only truly wrong choice is inconsistency** (mixing `Student` and `Courses` in the same schema). Many ORMs (e.g., Ruby on Rails' ActiveRecord) default to **plural** table names by convention; many hand-written academic/textbook schemas (and this handbook, for clarity when referring to a single conceptual entity) favor **singular**. Pick one, document the decision, and enforce it consistently across the whole team/codebase.

### Junction/Associative Table Naming

A common, readable convention is combining both participating table names:

```
Student + Course  → StudentCourse or Enrollment (a meaningful business name,
                     when one clearly exists — Module 6, §9's insight that
                     associative entities often deserve their own real name,
                     not just a mechanical concatenation)
```

> 💡 **Best Practice:** When an associative entity has a clear real-world name (like `Enrollment`, `Appointment`, `Assignment` — all seen throughout this Part), **use that name** rather than a mechanically concatenated one (`StudentCourse`) — it's more readable and self-documenting.

### Prefixing/Namespacing (Large Systems)

In large multi-domain databases, some teams prefix tables by module/domain (`billing_invoice`, `billing_payment`, `hr_employee`, `hr_department`) to make ownership and grouping clear at a glance — a practical convention worth knowing for large-scale system design discussions.

### Interview Importance

**Medium** — the "consistency matters more than the specific choice" framing is the key, mature takeaway to communicate if asked.

---

## 4. Column Naming

### Core Guidelines

| Guideline | Good Example | Poor Example |
|---|---|---|
| Be specific about what a foreign key references | `manager_id` (clearly an FK to a person) | `mgr` |
| Use consistent suffixes for common patterns | `_id` for keys, `_at` for timestamps, `_date` for dates without time, `is_`/`has_` prefix for booleans | Mixing `flag`, `is_active`, `active_yn` inconsistently across tables |
| Don't repeat the table name unnecessarily in the column name | `Student.name` (not `Student.student_name`) — the table context already tells you it's a student's name | `Student.student_name` (redundant, verbose) |
| **Exception to the above:** foreign keys and truly ambiguous columns SHOULD repeat context | `Employee.manager_id` (not just `Employee.id_2`) | Overly terse, unclear FK naming |

### Boolean Column Naming

```
✅ is_active, has_paid, is_deleted
❌ active (ambiguous — active WHAT? a flag? a count?), status_flag (unclear what values it holds)
```

### Interview Importance

**Medium** — column naming quality is one of those "death by a thousand cuts" issues that doesn't usually get a dedicated interview question, but consistently good naming throughout a live design exercise builds real credibility with an interviewer.

---

## 5. Data Consistency

### What This Means at the Schema Design Level

Beyond naming, **data consistency** here refers to ensuring that the **same real-world concept is always represented the same way** across the entire schema — same data type, same units, same encoding, same nullability rules.

### Common Consistency Pitfalls

| Pitfall | Example |
|---|---|
| **Inconsistent units** | `Product.weight` stored in kilograms in one table, but `Shipment.weight` stored in pounds in another — a silent, dangerous bug waiting to happen |
| **Inconsistent date/time handling** | Some tables storing timestamps in UTC, others in local server time, with no clear documentation of which is which |
| **Inconsistent representation of the same concept** | `status` stored as an integer code (`1`, `2`, `3`) in one table, but as a string (`'Active'`, `'Inactive'`) in another, for conceptually the same kind of status field |
| **Inconsistent precision** | `price` stored as `FLOAT` in one table (imprecise for currency!) and `DECIMAL(10,2)` in another (correct for currency) |

> ⚠️ **Critical, frequently-tested point:** **Never use floating-point types (`FLOAT`, `DOUBLE`) for currency/monetary values** — floating-point representation is inherently imprecise for base-10 decimal arithmetic (a classic computer science gotcha), and can cause real, cumulative rounding errors in financial calculations. Always use a fixed-precision type like `DECIMAL(precision, scale)` for money.

### Best Practices

- Document units and encoding conventions explicitly (e.g., in a data dictionary — Module 15) for anything not self-evident.
- Standardize on **UTC** for all stored timestamps, converting to local time only at the presentation/application layer — a very common, strongly recommended industry practice.
- Use `ENUM` types or a dedicated lookup/reference table (rather than free-text strings) for any attribute with a fixed, known set of valid values — connects directly back to Module 2, §9's Domain concept.

### Interview Importance

**High** — the "never use FLOAT for money" rule specifically is an extremely common, sharp interview/code-review gotcha question.

---

## 6. Avoiding Redundancy

### What This Means (A Preview of Normalization, Part 3)

> **Note:** This section introduces redundancy avoidance as a **schema design principle**; the *formal, rigorous* toolkit for eliminating redundancy (Normal Forms — 1NF, 2NF, 3NF, BCNF) is the entire subject of the **Normalization module in Part 3**. This section covers the intuitive, practical version of the idea.

### The Core Problem

Storing the **same fact in more than one place** creates the risk that the copies will eventually **disagree** — one gets updated, the other doesn't, and now the database contains contradictory "truths."

### Example

```
❌ BAD (redundant):
Order(order_id, customer_id, customer_name, customer_email, product_id, product_name, quantity)

  If a customer changes their email, EVERY order row for that customer
  must be updated too — miss even one, and the database now has
  conflicting email addresses for the same customer.

✅ GOOD (redundancy eliminated via foreign keys):
Customer(customer_id, customer_name, customer_email)
Product(product_id, product_name)
Order(order_id, customer_id, product_id, quantity)

  Customer's email is stored EXACTLY ONCE, in Customer.
  Order simply references it via customer_id.
```

### When SOME Redundancy Is a Deliberate, Acceptable Trade-off

> ⚠️ **Important nuance:** Avoiding redundancy is a strong default, but it is **not an absolute, unconditional rule** — real, high-performance production systems sometimes deliberately introduce **controlled, well-understood redundancy** (a technique called **denormalization**, covered fully in Part 3) for significant read-performance gains, especially in read-heavy analytics/reporting systems. The key difference between "sloppy redundancy" (a design mistake) and "deliberate denormalization" (a considered trade-off) is: the latter is **intentional, documented, and paired with a reliable mechanism** (triggers, batch jobs, application logic) to keep the redundant copies in sync — not an accident.

### Interview Importance

**High** — recognizing both "avoid redundancy by default" AND "but denormalization is sometimes a valid, deliberate choice" demonstrates real engineering maturity valued at senior interview levels.

---

## 7. Future-Proof Schemas

### The Core Challenge

Requirements change. A schema designed only for *today's* exact requirements often needs painful, risky migrations when the business evolves. Future-proofing means making **reasonable, low-cost design choices now** that reduce the pain of *likely* future changes — without wastefully over-engineering for changes that will probably never happen.

### Practical Techniques

| Technique | Why It Helps |
|---|---|
| **Prefer surrogate keys over natural keys for primary keys** (Module 4, §16) | Natural keys can change or later prove non-unique; surrogate keys insulate the schema from that risk |
| **Use lookup/reference tables instead of hardcoded ENUM-like string columns for evolving categories** | Adding a new "status" value becomes an `INSERT` into a reference table, not a risky `ALTER TABLE ... MODIFY COLUMN` schema change |
| **Add `created_at`/`updated_at` timestamp columns to every table by default** | Nearly universally useful later for auditing, debugging, and analytics — very cheap to add now, painful to retrofit onto years of historical data later |
| **Avoid overly rigid, hardcoded structures for genuinely variable data** | e.g., don't hardcode `phone_1`, `phone_2`, `phone_3` columns for a "multiple phone numbers" need — model it properly as a multivalued attribute (Module 7 §6 / Module 13 §5) so it scales to any number of phones |
| **Consider soft-deletes (`is_deleted` / `deleted_at`) instead of hard `DELETE`s for business-critical data** | Preserves historical/audit data; easily reversible; but adds query complexity (must always filter `WHERE deleted_at IS NULL`) — a genuine trade-off, not a free win |

### The Balance: Avoiding BOTH Under- and Over-Engineering

> **Guidance:** Future-proofing is a **judgment call**, not a checklist to maximize. Adding `created_at`/`updated_at` to every table is near-universally cheap and valuable — do it by default. But speculatively modeling every relationship as many-to-many "just in case" (already flagged as a specific trap in Module 10, §9), or building an elaborate, generic "entity-attribute-value" schema to handle hypothetical future flexibility, usually adds far more complexity and performance cost than it saves — **only future-proof against changes you have real, concrete reason to expect**, informed by genuine conversations with business stakeholders (echoing Module 6, §3's requirements-gathering emphasis).

### Interview Importance

**High for Senior/Staff roles** — discussing this exact "don't over-engineer, don't under-engineer" balance, with concrete reasoning, is a strong senior-level signal.

---

## 8. Scalability Considerations

### What "Scalability" Means at the Schema Design Level (Preview — Full Depth in Later Parts)

> **Note:** Deep scalability topics — indexing strategy, partitioning, sharding, replication, caching — belong to later Parts of this handbook (Storage & Indexing, Distributed Databases). This section covers only the **schema-design-level** considerations that influence scalability, as a bridge/preview.

### Schema-Level Scalability Principles

| Principle | Why It Matters |
|---|---|
| **Prefer small, fixed-size primary keys (e.g., `INT`/`BIGINT`) over large text-based keys** | Smaller keys mean smaller indexes, faster joins, and less storage overhead — compounds significantly at large row counts (millions/billions of rows) |
| **Avoid excessively wide tables (too many columns) when a table serves very different access patterns** | A table mixing frequently-updated "hot" columns with rarely-touched "cold" columns can benefit from being split — a design consideration that connects toward normalization/denormalization trade-offs (Part 3) |
| **Design with eventual partitioning/sharding in mind for very high-growth tables** | Choosing a good partition key (e.g., `created_at` for time-series data, or `customer_id` for multi-tenant data) is far easier to plan for at initial design time than to retrofit after a table has grown to hundreds of millions of rows |
| **Be deliberate about foreign key usage in distributed/microservice contexts** | As flagged in Module 4, §9's "Industry Usage" section, foreign keys spanning separate databases/services aren't enforceable by a single DBMS — this is a genuine architectural constraint that shapes schema boundaries in large distributed systems |

### Interview Importance

**Medium at this stage** (deep scalability mechanics come in later Parts), but **High** for demonstrating awareness that schema design decisions made *today* have real consequences at scale *later* — a hallmark of senior system design thinking.

---

## 9. Full Before/After Schema Review Example

Let's apply every principle in this module to review and improve a poorly-designed schema.

### BEFORE (Poorly Designed)

```sql
CREATE TABLE orders (
    OrderID       INT PRIMARY KEY,
    custName      VARCHAR(100),
    custEmail     VARCHAR(100),
    cust_phone1   VARCHAR(15),
    cust_phone2   VARCHAR(15),
    productName   VARCHAR(100),
    Price         FLOAT,               -- ❌ FLOAT for money!
    order_status  INT,                 -- ❌ magic number, undocumented meaning
    orderDate     VARCHAR(20)          -- ❌ date stored as free-text string!
);
```

### Issues Identified (Mapped to This Module's Sections)

| Issue | Section |
|---|---|
| Inconsistent naming (`OrderID` vs `custName` vs `cust_phone1`) | §2, §4 |
| `cust_phone1`/`cust_phone2` — rigid, doesn't scale to N phone numbers | §7 |
| Customer and Product data duplicated directly into `orders` (redundancy) | §6 |
| `Price` as `FLOAT` — imprecise for currency | §5 |
| `order_status` as an undocumented magic integer | §5 |
| `orderDate` stored as `VARCHAR` instead of a proper `DATE`/`TIMESTAMP` type — loses type safety, sorting, and date-arithmetic capability | §5 |
| No `created_at`/`updated_at` audit columns | §7 |

### AFTER (Redesigned Using This Module's Principles)

```sql
CREATE TABLE customer (
    customer_id  INT PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    email        VARCHAR(100) UNIQUE,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE customer_phone (              -- multivalued attribute, properly scaled (§7, Module 7 §6)
    customer_id   INT,
    phone_number  VARCHAR(15),
    PRIMARY KEY (customer_id, phone_number),
    FOREIGN KEY (customer_id) REFERENCES customer(customer_id)
);

CREATE TABLE product (
    product_id  INT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_status (                -- lookup/reference table instead of magic numbers (§7)
    status_id    INT PRIMARY KEY,
    status_name  VARCHAR(20) UNIQUE NOT NULL   -- e.g., 'Pending', 'Shipped', 'Delivered'
);

CREATE TABLE `order` (
    order_id     INT PRIMARY KEY,
    customer_id  INT NOT NULL,
    product_id   INT NOT NULL,
    price        DECIMAL(10,2) NOT NULL,        -- ✅ DECIMAL for currency (§5)
    status_id    INT NOT NULL,
    order_date   DATE NOT NULL,                  -- ✅ proper DATE type (§5)
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customer(customer_id),
    FOREIGN KEY (product_id) REFERENCES product(product_id),
    FOREIGN KEY (status_id) REFERENCES order_status(status_id)
);
```

### Interview Importance

**Extremely High.** "Review this schema and tell me what's wrong with it" is an extremely common, practical interview format at every level — being able to rapidly spot and articulate exactly this kind of issue list is a directly transferable, high-value skill.

---

## 10. Chapter Summary

- **Naming conventions** should be picked once and applied with total consistency across an entire schema — the *specific* choice (snake_case vs. camelCase, singular vs. plural table names) matters far less than the consistency itself.
- **Column naming** should avoid redundant repetition of the table name, use consistent suffixes (`_id`, `_at`, `is_`), and be unambiguous, especially for foreign keys.
- **Data consistency** means representing the same real-world concept the same way everywhere — same units, same date/time handling (prefer UTC), same precision (never `FLOAT` for currency — always `DECIMAL`), and using lookup tables/`ENUM`s instead of undocumented magic values.
- **Avoiding redundancy** (a practical preview of Part 3's formal Normalization) means storing each fact exactly once by default, via foreign keys — though **deliberate, documented denormalization** is a legitimate, considered trade-off for performance in specific scenarios, distinct from accidental redundancy.
- **Future-proofing** means making cheap, high-value defaults now (surrogate keys, `created_at`/`updated_at`, lookup tables for evolving categories, proper multivalued-attribute modeling) — while explicitly avoiding speculative over-engineering for changes with no real evidence of ever happening.
- **Scalability considerations** at the schema level include preferring small fixed-size keys, being mindful of wide/hot-vs-cold-column tables, and planning ahead for eventual partitioning — full depth deferred to later Parts on Storage/Indexing and Distributed Databases.
- The full before/after `orders` table review (§9) demonstrates every principle applied together — a directly interview-transferable "spot the schema smells" exercise.

---

## 11. Cheat Sheet

```
NAMING:
  ✓ ONE consistent case style (snake_case OR camelCase) — never mixed
  ✓ Consistent suffixes: _id (keys), _at (timestamps), is_/has_ (booleans)
  ✓ Don't repeat table name in column name (Student.name, not Student.student_name)
  ✓ Singular vs plural table names: EITHER is fine — just be CONSISTENT

DATA CONSISTENCY:
  ✓ NEVER use FLOAT/DOUBLE for currency — always DECIMAL(precision, scale)
  ✓ Store timestamps in UTC; convert to local time only at the display layer
  ✓ Use ENUM or a lookup/reference table for fixed value sets — never magic numbers

AVOIDING REDUNDANCY:
  ✓ Store each fact ONCE; use foreign keys to reference it elsewhere
  ✓ Deliberate DENORMALIZATION (documented, synced) ≠ accidental redundancy (a bug)

FUTURE-PROOFING:
  ✓ Prefer surrogate keys (Module 4 §11) as primary keys
  ✓ Add created_at / updated_at to every table by default — cheap now, painful later
  ✓ Model evolving categories as lookup tables, not hardcoded columns/ENUMs
  ✓ Model "multiple X" as a proper multivalued-attribute table, never phone_1/phone_2/phone_3
  ✓ Don't over-engineer for hypothetical changes with no real business signal

SCALABILITY (schema-level preview):
  ✓ Small, fixed-size PKs (INT/BIGINT) over large text keys
  ✓ Plan partition keys for high-growth tables early
  ✓ Cross-service FKs aren't DB-enforceable in microservice architectures
```

---

## 12. Exercises

### Beginner

1. Why is consistency more important than the specific naming convention chosen?
2. True/False: FLOAT is an acceptable data type for storing currency values.
3. Fill in the blank: Storing the same fact in multiple places, risking disagreement between copies, is called __________.

### Intermediate

4. Redesign this column: `Employee.emp_status_flag INT` (values: 1, 2, 3, undocumented) using this module's principles.
5. A table has columns `phone1`, `phone2`, `phone3`. Explain why this is a future-proofing anti-pattern and redesign it properly.
6. Explain the difference between "accidental redundancy" (a design flaw) and "deliberate denormalization" (a valid trade-off), with an example of each.

### Advanced / Conceptual

7. Take the full BEFORE schema from §9 and, without looking at the AFTER version, redesign it yourself, explaining each change you make and which principle from this module justifies it.
8. Discuss: when is adding `created_at`/`updated_at` to every table worth the (small) overhead, and is there ever a case where you'd deliberately skip it?
9. A team debates whether to use `INT` surrogate keys or `UUID` surrogate keys for a new high-growth, multi-region distributed system. Using both Module 4, §11 and this module's scalability section (§8), discuss the trade-offs of each choice.

### MCQs

**Q1.** Which data type should be used for storing currency values?
a) FLOAT  b) DOUBLE  c) DECIMAL  d) VARCHAR

**Q2.** The most important principle in choosing a naming convention is:
a) Using camelCase specifically  b) Using snake_case specifically  c) Being consistent throughout the schema  d) Using the shortest possible names

**Q3.** Deliberate denormalization differs from accidental redundancy because it is:
a) Always faster  b) Intentional, documented, and kept in sync via a reliable mechanism  c) Never used in production  d) The same thing, just a different name

**Q4.** A future-proofing best practice cheap enough to apply by default to nearly every table is:
a) Making every relationship many-to-many  b) Adding created_at/updated_at timestamp columns  c) Using an entity-attribute-value generic schema  d) Avoiding all foreign keys

*(Answer key: 1-c, 2-c, 3-b, 4-b)*

### Schema Review Exercise

10. Review this schema and list every issue, referencing the specific section of this module that explains why it's a problem:
```sql
CREATE TABLE tbl_ord (
    id INT,
    cName VARCHAR(50),
    cPhone VARCHAR(15),
    amt FLOAT,
    st INT,
    dt VARCHAR(30)
);
```

### Practice Links

- *(Schema design review is best practiced by critiquing real, messy schemas — search for "database schema design mistakes" case studies, or review any personal/side-project schema you've built against this module's checklist.)*
- [LeetCode Database — Top SQL 50 (many problems reveal schema design quality issues in their given table structures)](https://leetcode.com/studyplan/top-sql-50/)

---

**Next: Module 15 — Data Dictionary (Metadata, System Catalog, Catalog Tables, Information Schema).**

> Say "continue" or "next module" to proceed to Module 15.
