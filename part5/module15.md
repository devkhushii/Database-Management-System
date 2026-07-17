# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 15: Schema Design Best Practices

> **Where we are:** Modules 7–14 gave us the complete *theory* of normalization and denormalization. This module is the **practical craftsmanship layer** — the conventions, patterns, and habits that separate a schema that's merely "theoretically correct" from one that's genuinely pleasant, safe, and maintainable to work with in a real engineering team, over years of evolution.

---

### 15.1 Naming Conventions

**Why does it exist?**
A schema is read far more often than it's written. Inconsistent naming (`userId` in one table, `user_id` in another, `UserID` in a third) creates constant, low-grade friction — every query, every ORM mapping, every new engineer's onboarding is slowed down by having to remember or look up which convention applies where.

**Best practices:**

| Rule | Example | Why |
|---|---|---|
| **Pick ONE case convention and enforce it everywhere** | `snake_case` (most common in SQL: `user_id`, `created_at`) | SQL is case-insensitive for identifiers by default in most DBMSs, and snake_case avoids quoting issues that `camelCase` sometimes requires |
| **Table names: plural or singular — pick one, stay consistent** | `users` vs `user` — either is fine, but never mix | Consistency lets developers predict a name without checking |
| **Primary key column: consistent pattern** | `id` (simple, if table name gives context) or `user_id` (explicit) | Explicit naming (`user_id`) makes joins self-documenting: `orders.user_id` vs a joined `users.id` |
| **Foreign key columns: name after the referenced entity + `_id`** | `orders.customer_id` referencing `customers.id` | Immediately signals the relationship without needing to check the schema diagram |
| **Boolean columns: prefix with `is_`, `has_`, `can_`** | `is_active`, `has_verified_email` | Removes ambiguity about what `active` (a column with no prefix) actually means as a value |
| **Avoid reserved SQL keywords as identifiers** | Avoid `order`, `group`, `user` (reserved in some DBMSs) as bare table/column names | Prevents constant need for quoting/escaping, and confusing error messages |
| **Junction/association tables: combine both entity names** | `student_course` for a many-to-many between `student` and `course` | Immediately communicates the table's purpose |

**Industry note:** Some organizations adopt more elaborate prefixing conventions (e.g., Hungarian-notation-style `tbl_`, `fk_`) — these are generally considered **outdated** in modern schema design; clean, consistent `snake_case` without decorative prefixes is the dominant modern convention across PostgreSQL, MySQL, and most ORM-based ecosystems.

---

### 15.2 Data Type Selection

**Why does it matter?**
Choosing the right data type affects storage efficiency, query performance, data integrity, and even correctness (using the wrong type can silently truncate or misrepresent data).

**Best practices by category:**

| Category | Guidance |
|---|---|
| **Integers** | Choose the smallest type that comfortably fits the expected range with headroom (`SMALLINT` vs `INT` vs `BIGINT`) — but don't over-optimize prematurely; `BIGINT` for primary keys is a common, safe default in systems expecting significant growth, since the storage difference is often negligible compared to the risk of overflow requiring a painful migration later. |
| **Strings** | Use `VARCHAR(n)` with a sensible, business-justified length limit rather than an unbounded `TEXT` for genuinely bounded fields (e.g., a country code, a status enum-like string) — this documents intent and can catch bad data early. Use `TEXT` for genuinely unbounded content (e.g., a blog post body). |
| **Money/currency** | **Never use floating-point types (`FLOAT`, `DOUBLE`) for money** — floating-point rounding errors can cause real, cumulative financial discrepancies. Use a fixed-point decimal type (`DECIMAL(p,s)` / `NUMERIC(p,s)`) instead. |
| **Dates and times** | Use dedicated `DATE`, `TIME`, `TIMESTAMP` (or `TIMESTAMPTZ` in PostgreSQL) types rather than storing dates as strings — this enables correct sorting, range queries, and date arithmetic, and avoids format-inconsistency bugs. **Always store timestamps in UTC**, converting to local time only at the display/application layer. |
: |
| **Enums / fixed categorical values** | Use a DBMS-native `ENUM` type (where well-supported, e.g., PostgreSQL, MySQL) or a `CHECK` constraint against a fixed set of allowed strings/integers, rather than a free-text column that could contain any typo'd value. |
| **Booleans** | Use a native `BOOLEAN` type where available, rather than encoding true/false as `0/1` integers or `'Y'/'N'` strings, for clarity and to leverage the DBMS's own validation. |

---

### 15.3 Nullability

**Why does it matter?**
Every column's nullability (whether `NULL` is a permitted value) is a **data integrity decision**, not a default to leave unconsidered. `NULL` in SQL has notoriously tricky three-valued logic (`NULL = NULL` is not `TRUE`, it's `NULL`) — every nullable column is a small, ongoing source of potential bugs in application logic and queries.

**Best practices:**
- **Default to `NOT NULL` wherever the business logic genuinely requires a value.** Only allow `NULL` where "no value" is a real, meaningful, distinct state (e.g., `middle_name` being `NULL` for someone with no middle name is meaningful; `email` being `NULL` on a system that requires email to register is usually a bug waiting to happen).
- **Avoid using `NULL` to mean multiple different things in the same column** (e.g., `NULL` sometimes meaning "not yet set" and other times meaning "explicitly opted out") — if these are genuinely different states, model them explicitly (e.g., a separate boolean flag, or a proper enum with a "not set" value).
- **Be deliberate about `NULL`s interaction with `UNIQUE` constraints:** most DBMSs treat multiple `NULL`s in a `UNIQUE` column as **not** violating uniqueness (since `NULL ≠ NULL`) — this can be surprising and occasionally exploited as a feature (e.g., allowing multiple rows with an "unset" unique value) or can silently cause an unexpected duplicate-looking state if not understood.
- **Document why a column is nullable** when it's not immediately obvious — a stray nullable column with no clear reasoning is a common source of "wait, why can this be empty?" confusion for future engineers.

---

### 15.4 Constraints

**Why does it matter?**
Constraints are the DBMS's own enforcement mechanism for data integrity — pushing correctness rules down to the data layer means they're enforced **regardless of which application, script, or careless direct SQL query touches the data**, rather than relying solely on application-level validation (which can be bypassed, buggy, or simply forgotten in a new code path).

**Core constraint types and best practices:**

| Constraint | Purpose | Best Practice |
|---|---|---|
| `PRIMARY KEY` | Uniquely identifies each row | Every table should have one; prefer a simple, stable key (Module 6, Section 6.7) |
| `FOREIGN KEY` | Enforces referential integrity between tables | Always define these explicitly, rather than relying on "implicit" application-level relationships — this catches orphaned-reference bugs at the database level |
| `UNIQUE` | Enforces alternate-key uniqueness (Module 6) | Use for any business-unique attribute (e.g., `email`, `username`) beyond the primary key |
| `NOT NULL` | Enforces mandatory presence of a value | Default choice (Section 15.3) unless there's a specific reason for nullability |
| `CHECK` | Enforces arbitrary value-level business rules | Use for simple, single-row logic (e.g., `CHECK (price >= 0)`, `CHECK (end_date > start_date)`) — don't overload `CHECK` with complex cross-table logic; that belongs in triggers or application logic |
| `DEFAULT` | Provides a sensible fallback value when none is supplied | Use for genuinely sensible defaults (e.g., `created_at DEFAULT NOW()`, `is_active DEFAULT TRUE`) — avoid defaults that mask a genuinely required input being accidentally omitted |

**Best practice — cascading behavior on foreign keys:** always explicitly decide and declare `ON DELETE` / `ON UPDATE` behavior (`CASCADE`, `RESTRICT`, `SET NULL`, `SET DEFAULT`) rather than relying on the DBMS's default — an unconsidered default can lead to either accidental mass-deletion (`CASCADE` when you meant `RESTRICT`) or blocked, confusing deletion failures (`RESTRICT` when you meant `CASCADE`).

---

### 15.5 Future Scalability

**Why does it matter?**
A schema designed only for today's requirements and data volume will often need painful, risky migrations as the system grows — thinking ahead (without over-engineering) reduces this future pain.

**Best practices:**
- **Choose primary key types with headroom.** An `INT` primary key (typically maxing out around 2.1 billion) can be a real constraint for a high-growth table (e.g., an events/log table) — `BIGINT` is a cheap insurance policy against a painful future migration.
- **Design for eventual partitioning/sharding**, even if not implemented immediately — e.g., including a natural partition key (like a `tenant_id` in a multi-tenant system, or a `created_at` date for time-series data) from the start makes a future partitioning migration far less disruptive.
- **Avoid tightly coupling schema to a single, current business rule that's likely to change** — e.g., hardcoding "a user has exactly one address" as a column on the `Users` table (rather than a separate `Addresses` table) can become a costly redesign if the business later needs to support multiple addresses per user (shipping vs billing, for instance).
- **Consider read/write ratio and expected query patterns early** — this shapes whether a more normalized (write-optimized) or somewhat denormalized (read-optimized) initial design is more appropriate (Module 14).

---

### 15.6 Maintainability

**Why does it matter?**
A schema is a **long-lived artifact** maintained by many engineers over years — designs that are easy to understand and safely modify reduce the ongoing cost and risk of the entire system's evolution.

**Best practices:**
- **Prefer explicit, simple foreign-key relationships over "clever" implicit ones** (e.g., a generic `related_id` + `related_type` polymorphic column that requires application-level logic to interpret) — explicit relationships are self-documenting and let the DBMS enforce integrity directly.
- **Avoid deeply nested or excessively normalized structures where a simpler design would serve just as well** — Module 1's trade-off framework applies here: normalize enough to prevent real anomalies, not for its own sake.
- **Keep migrations small and incremental** where possible, rather than large, risky, all-at-once schema rewrites — this reduces the blast radius of any single change and makes rollback easier if something goes wrong.
- **Avoid "magic values"** embedded in application logic that aren't reflected in the schema (e.g., a special `status = -1` meaning "archived," known only through tribal knowledge) — model such states explicitly, ideally via a constrained/enum type (Section 15.2).

---

### 15.7 Documentation

**Why does it matter?**
Schemas accumulate implicit knowledge over time — why a column exists, why it's nullable, what a cryptic abbreviation means — that isn't visible from the DDL alone. Undocumented schemas force every new engineer to reconstruct this knowledge from scratch, often incorrectly.

**Best practices:**
- **Use DBMS-native comment features** where available (e.g., PostgreSQL's `COMMENT ON COLUMN`) to attach human-readable explanations directly to schema objects, so the documentation lives alongside the structure itself and doesn't drift out of sync with a separate wiki page.
- **Maintain a data dictionary** (even a simple shared document) for genuinely complex or business-critical tables, documenting each column's purpose, valid values, and relationships.
- **Document the FDs and normalization decisions** behind non-obvious schema choices (as flagged back in Module 7, Section 7.13) — this prevents a future engineer from "helpfully" reintroducing redundancy without understanding why the current structure avoids it.
- **Keep an entity-relationship diagram up to date** as a living artifact, not a one-time deliverable frozen at initial design time (connecting back to Module 1's database lifecycle).

---

### 15.8 Auditing Columns

**What are they?**
Standard columns added to (most) tables specifically to track the **history of changes** to each row, supporting debugging, compliance, and analytics.

**Common auditing columns:**

| Column | Purpose |
|---|---|
| `created_at` | Timestamp of row creation — nearly universal, low-cost, high-value |
| `updated_at` | Timestamp of the most recent modification — often auto-maintained via a trigger or ORM hook |
| `created_by` | Identifier (usually a user ID) of who/what created the row |
| `updated_by` | Identifier of who/what most recently modified the row |
| `version` | An incrementing integer, useful for optimistic concurrency control (detecting conflicting concurrent updates) |

**Best practice:** Add `created_at` and `updated_at` to essentially every table by default in most business applications — the storage cost is trivial, and the debugging/audit value ("when did this actually change, and who changed it?") is consistently high. `created_by`/`updated_by` are worth adding specifically for tables where accountability or compliance matters (financial records, user-permission changes, etc.).

---

### 15.9 Soft Delete

**What is it?**
Instead of physically removing a row via `DELETE`, a **soft delete** marks the row as deleted (typically via a `deleted_at` timestamp column, or an `is_deleted` boolean), while keeping the actual data in the table — application queries then filter out soft-deleted rows by default.

**Why does it exist?**
- **Recoverability:** an accidental or user-initiated "delete" can be reversed by simply un-marking the flag, rather than requiring a backup restore.
- **Preserving referential integrity for historical records:** if a `Product` is "deleted" but historical `Orders` still reference it, a soft delete keeps the product record available for those historical orders to still display correctly, rather than leaving a broken foreign-key reference or requiring `ON DELETE CASCADE` to also erase order history.
- **Auditing/compliance:** some regulatory contexts require retaining a record of "deleted" data for a period, rather than genuinely purging it immediately.

**Trade-offs and best practices:**
- **Every query against a soft-deletable table must remember to filter `WHERE deleted_at IS NULL`** (or equivalent) — this is easy to forget, and forgetting it is a common, subtle bug source (a "deleted" row silently reappearing in some report or API response). Using a DB view or ORM-level default scope to enforce this filtering automatically is a common mitigation.
- **Unique constraints become trickier** — if `email` must be unique among *active* users but a soft-deleted user's email should be "freed up" for reuse, a naive `UNIQUE(email)` constraint won't allow this; a partial/filtered unique index (`UNIQUE(email) WHERE deleted_at IS NULL`, supported natively in PostgreSQL) is the typical solution.
- **Soft-deleted data still consumes storage indefinitely** unless a separate, deliberate archival/purge process is established — soft delete alone is not a storage-management strategy.
- **Not always the right choice:** for data with strict legal "right to be forgotten" requirements (e.g., GDPR-driven deletion requests), a genuine hard delete (or cryptographic erasure) may be legally required, and soft delete alone would not satisfy that obligation.

---

### 15.10 Versioning (Overview)

**What is it?**
Schema/data **versioning** refers to strategies for tracking how a row's data (or the schema itself) changes over time, going beyond simple `created_at`/`updated_at` auditing to preserve the actual **historical values**, not just the fact that a change occurred.

**Two related but distinct concepts:**
1. **Schema versioning / migrations:** tracking and managing changes to the *structure* of the database over time (adding columns, changing types) via version-controlled migration scripts (a common practice using tools in the broader engineering ecosystem, generally covered in tooling/DevOps-adjacent material rather than pure DBMS theory).
2. **Data versioning / temporal tables:** tracking how a specific row's *data* has changed over time — e.g., keeping a full history of every price a product has ever had, not just its current price. This connects directly back to **Module 12's brief mention of 6NF**, since fully independent temporal versioning of different attributes is exactly the specialized scenario 6NF-style extreme decomposition is designed to support.

**Common data-versioning patterns (overview only — full temporal database theory is a specialized topic beyond this Part's scope):**
- **Row-level history tables:** a separate `Product_History` table storing a snapshot of a `Product` row every time it changes, alongside a validity date range.
- **Bitemporal modeling:** tracking both "when this was true in the real world" and "when we recorded this fact in the database" as two separate time dimensions — used in finance and other domains requiring precise retroactive-correction auditability.

**Best practice takeaway:** Only introduce full data-versioning complexity when the business genuinely requires historical reconstruction (e.g., "what was this customer's address when this specific order shipped?") — for most tables, simple `created_at`/`updated_at` auditing (Section 15.8) is sufficient, and full temporal modeling should be reserved for the specific tables where the business need is real and explicit.

---

### 15.11 Industry Implementation Notes

- Most modern web frameworks and ORMs (Django, Rails, Laravel) have **built-in conventions and helpers** for auditing columns (`created_at`/`updated_at` auto-timestamps) and soft delete (`SoftDeletes` traits/mixins) — leveraging these framework conventions is usually preferable to hand-rolling equivalent logic.
- Schema migration tools (Flyway, Liquibase, Django migrations, Rails migrations, Alembic) are the standard industry mechanism for managing schema versioning (Section 15.10, point 1) in a version-controlled, repeatable, team-collaborative way.
- Naming convention debates (`snake_case` vs `camelCase`, singular vs plural table names) are common points of team disagreement — the specific choice matters far less than **team-wide consistency**; picking one and documenting it in a shared style guide is more valuable than which specific convention is chosen.

---

### 15.12 Best Practices — Consolidated Checklist

```
✓ Consistent naming convention (snake_case, singular/plural — pick one)
✓ Appropriate, deliberately-chosen data types (never FLOAT for money!)
✓ NOT NULL by default; NULL only where genuinely meaningful
✓ Explicit constraints: PK, FK (with ON DELETE/UPDATE behavior), UNIQUE, CHECK
✓ Primary keys with growth headroom (BIGINT where appropriate)
✓ created_at / updated_at on most tables
✓ Soft delete where recoverability/history matters, with partial unique
  indexes to handle uniqueness-among-active-rows correctly
✓ Documented schema (comments, data dictionary, up-to-date ER diagrams)
✓ Versioning/temporal modeling ONLY where a genuine business need exists
```

---

### 15.13 Common Mistakes

- Using `FLOAT`/`DOUBLE` for monetary values — a classic, serious mistake that causes real financial discrepancies over time.
- Making every column nullable "just in case," rather than deliberately deciding nullability per column.
- Forgetting to specify `ON DELETE`/`ON UPDATE` behavior on foreign keys, leading to either surprise cascading deletes or confusing blocked-deletion errors.
- Implementing soft delete without a partial/filtered unique index strategy, then being surprised that "deleted" users' emails can't be reused by new signups.
- Skipping `created_at`/`updated_at` columns, then struggling to debug a data-integrity incident months later with no timeline to investigate.
- Over-engineering full temporal/bitemporal versioning for tables that will never actually need historical reconstruction — needless complexity for no real business benefit.

---

### 15.14 Edge Cases

- **Soft-deleted rows referenced by foreign keys from other, non-deleted rows** — requires careful application-level handling to display "this product has been discontinued" rather than a broken or missing reference.
- **A `NOT NULL` column that later needs to become nullable** (or vice versa) due to a changed business rule — a real, sometimes non-trivial migration, reinforcing Section 15.5's "design for likely evolution" guidance.
- **Multi-tenant systems needing a `tenant_id` on nearly every table** — an architecture-wide naming/constraint convention decision that's far easier to establish from the start than retrofit later.

---

### 15.15 Performance Implications

- Appropriately-sized data types (Section 15.2) reduce storage and improve cache efficiency (more rows fit in a memory page).
- Soft delete's mandatory `WHERE deleted_at IS NULL` filtering on every query benefits significantly from a partial index specifically covering non-deleted rows, rather than relying on a full-table index that includes soft-deleted rows the application will always filter out anyway.
- Auditing columns (`created_at`, etc.) are typically low-cost to store and index, and frequently valuable for range-based queries ("show me everything created in the last 24 hours") — often worth an index of their own in high-write tables.

---

### 15.16 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | Explain why money should never be stored as FLOAT; explain the purpose of created_at/updated_at. |
| Intermediate | Design a soft-delete strategy for a `Users` table, including how to handle the unique-email constraint. |
| Senior SDE-2/SDE-3 | Discuss schema design trade-offs for a real system, covering naming, constraints, scalability (key sizing, partitioning readiness), and how you'd decide between soft delete and hard delete for a specific compliance scenario. |
| GATE | Schema design best practices are generally not a GATE numerical-question topic, but do appear in some university-level design-oriented exam questions. |

---

### 15.17 Chapter Summary

- **Naming conventions** and consistent **data type selection** (never FLOAT for money!) form the foundation of a maintainable schema.
- **Nullability** and **constraints** (PK, FK with explicit cascade behavior, UNIQUE, CHECK) are deliberate integrity decisions, not defaults to leave unconsidered.
- Designing for **future scalability** (key sizing, partitioning readiness) and **maintainability** (explicit relationships, small incremental migrations) reduces long-term pain.
- **Documentation** keeps implicit schema knowledge from being lost as teams change over time.
- **Auditing columns** (`created_at`/`updated_at`) are a near-universal, low-cost, high-value addition.
- **Soft delete** provides recoverability and historical integrity, at the cost of needing careful unique-constraint and query-filtering handling.
- **Versioning/temporal modeling** should be reserved for genuine business needs, not applied by default.

---

### 15.18 Cheat Sheet

```
NAMING:      snake_case, consistent singular/plural, explicit FK naming (_id)
DATA TYPES:  DECIMAL for money (never FLOAT), proper DATE/TIMESTAMP types, ENUM/CHECK for categories
NULLABILITY: NOT NULL by default; NULL only where genuinely meaningful
CONSTRAINTS: PK + FK (explicit ON DELETE/UPDATE) + UNIQUE + CHECK — enforce at the DB layer
SCALABILITY: BIGINT PKs where growth is expected; plan for partition keys early
AUDITING:    created_at, updated_at (near-universal); created_by/updated_by where accountability matters
SOFT DELETE: deleted_at/is_deleted + partial unique index for "unique among active rows"
VERSIONING:  only when genuine historical-reconstruction business need exists
```

---

### 15.19 MCQs

**Q1.** Why should monetary values never be stored as FLOAT/DOUBLE?
a) They take up more storage
b) Floating-point rounding errors can cause real financial discrepancies
c) FLOAT doesn't support negative numbers
d) SQL doesn't support FLOAT for currency by default

<details><summary>Answer</summary>b) Floating-point rounding errors cause real discrepancies; use DECIMAL/NUMERIC</details>

**Q2.** What is the main purpose of a soft delete?
a) To save storage space
b) To allow recoverability and preserve historical referential integrity instead of permanently removing data
c) To speed up all queries automatically
d) To avoid needing a primary key

<details><summary>Answer</summary>b) Recoverability and preserving historical integrity</details>

**Q3.** When implementing soft delete with a unique `email` constraint that should allow reuse after deletion, the best solution is:
a) Remove the UNIQUE constraint entirely
b) Use a partial/filtered unique index scoped to non-deleted rows
c) Hard delete instead
d) Store email in a separate unindexed table

<details><summary>Answer</summary>b) A partial/filtered unique index</details>

**Q4.** Why is `BIGINT` often preferred over `INT` for primary keys in high-growth tables?
a) BIGINT is always faster
b) INT has a much smaller maximum range and could realistically be exhausted, requiring a painful future migration
c) BIGINT uses less storage
d) SQL requires BIGINT for primary keys

<details><summary>Answer</summary>b) INT's smaller range risk of exhaustion in high-growth tables</details>

**Q5.** What should you always explicitly decide when defining a FOREIGN KEY constraint?
a) The column's data type only
b) The ON DELETE / ON UPDATE cascading behavior
c) Whether to use snake_case or camelCase
d) Whether the column should be a primary key too

<details><summary>Answer</summary>b) ON DELETE/ON UPDATE behavior</details>

---

### 15.20 Exercises

**Beginner:**
1. List five schema design best practices from this module and briefly justify each.
2. Explain why `NOT NULL` should generally be the default choice for columns, with an example of a column where `NULL` genuinely makes sense.

**Intermediate — Applied Design:**
3. Design the columns (with data types) for an `Orders` table for an e-commerce system, applying naming conventions, appropriate data types (especially for money), nullability decisions, and auditing columns.
4. A `Users` table needs a soft-delete strategy. Design the relevant columns and the unique-constraint approach needed to allow a deleted user's email to be reused by a new signup.

**Advanced — Trade-off Discussions:**
5. Discuss when you would choose hard delete over soft delete, considering a scenario involving GDPR-style "right to be forgotten" legal requirements.
6. A high-growth events-logging table is being designed. Discuss your primary key type choice, and what other future-scalability considerations (Section 15.5) you'd apply.

**Case Study:**
7. Review this (deliberately flawed) column list for a `Product` table: `id (INT), Name (VARCHAR(255)), price (FLOAT), Created (VARCHAR(50)), deleted (INT)`. Identify every schema design mistake present, and propose a corrected version, explaining each fix.

**Whiteboard Design:**
8. Design a full schema (tables + key columns only, no need for every attribute) for a multi-tenant SaaS application, applying the `tenant_id` convention discussed in Section 15.14, plus appropriate auditing and soft-delete columns.

**Interview Questions:**
- *Beginner:* "What data type would you use for storing money, and why?"
- *Intermediate:* "How would you implement soft delete while still enforcing uniqueness on an email column?"
- *Senior:* "Walk me through the schema design decisions (naming, types, constraints, scalability) you'd make when designing a new core table for a system expected to scale significantly over the next few years."

**Practice links:**
- GeeksforGeeks — Database Design Best Practices: https://www.geeksforgeeks.org/dbms/database-design-in-dbms/
- (This topic is primarily industry/system-design focused rather than GATE-numerical; practicing via real schema design case studies and system design interview resources is recommended.)

---

**End of Module 15.**

*Next up — Module 16: Real-World Database Design — complete schema designs for Banking, E-commerce, Hospital Management, University Management, Library Management, Inventory, Food Delivery, Ride Sharing, and Social Media systems, applying everything from Modules 1–15 end to end. Reply "next" / "continue" when ready.*
