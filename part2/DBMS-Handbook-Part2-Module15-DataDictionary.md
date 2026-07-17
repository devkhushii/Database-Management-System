# DBMS Handbook — Part 2
## Module 15: Data Dictionary

> **Builds on:** Module 2 (Database Schema, §14), Module 5 (Integrity Constraints), Module 14 (Schema Design — documentation practices).
>
> **Feeds into:** Module 16 (Common Modeling Mistakes — many are caught by consulting the data dictionary), and query optimization topics in later Parts (the query optimizer itself relies on catalog metadata).

---

## Table of Contents

1. [What Is Metadata?](#1-what-is-metadata)
2. [The System Catalog](#2-the-system-catalog)
3. [Catalog Tables](#3-catalog-tables)
4. [INFORMATION_SCHEMA — The Standardized View](#4-informationschema--the-standardized-view)
5. [Why the Data Dictionary Is Important](#5-why-the-data-dictionary-is-important)
6. [Worked Example: Exploring a Real Database's Metadata](#6-worked-example-exploring-a-real-databases-metadata)
7. [Chapter Summary](#7-chapter-summary)
8. [Cheat Sheet](#8-cheat-sheet)
9. [Exercises](#9-exercises)

---

## 1. What Is Metadata?

### Definition

**Metadata** is literally **"data about data"** — information describing the structure, constraints, and characteristics of the actual data stored in a database, rather than the data itself.

### Example

```
DATA:      (101, "Aditi", "CSE")     ← an actual row in Student

METADATA:  Table name: Student
           Columns: roll_no (INT, PK), name (VARCHAR(50)), dept (VARCHAR(10))
           Constraints: PRIMARY KEY (roll_no)
           Row count: 4,502
           Created: 2024-03-15
```

### Where You've Already Encountered Metadata (Full-Circle Connections)

Every concept from **Module 2** — relation schema (§12), database schema (§14), degree (§10) — is, precisely, a piece of metadata. This module simply formalizes **where and how the DBMS itself stores and exposes** that metadata for querying.

### Interview Importance

**Medium** — foundational vocabulary, rarely the focus of a deep question but useful for framing the rest of this module.

---

## 2. The System Catalog

### Definition

The **system catalog** (also called the **data dictionary**, **system tables**, or **metastore**) is a special, DBMS-managed collection of tables that stores **all metadata about the database itself** — every table, column, constraint, index, user, and permission is recorded here.

### Why It Exists

> **The system catalog exists because a DBMS needs to know about its own structure to function at all** — before it can execute `SELECT name FROM Student`, the query optimizer and execution engine must first look up: "Does a table called `Student` exist? Does it have a column called `name`? What's its data type? Is there an index on it?" All of this lookup happens against the system catalog.

### The Catalog Is Itself... a Database

> 💡 **A genuinely elegant, frequently-appreciated design insight:** The system catalog is, itself, typically implemented as a set of **ordinary relational tables**, managed by the very same DBMS engine it describes. This means you can query metadata using the **exact same SQL `SELECT` syntax** you use for regular data — a form of beautiful self-consistency in relational database design.

```sql
-- Querying actual data:
SELECT name FROM Student WHERE dept = 'CSE';

-- Querying METADATA (about the Student table itself) using the SAME SQL syntax:
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'Student';
```

### Interview Importance

**Medium-High** — understanding that "metadata is just more relational data" is a genuinely insightful, senior-signaling point.

---

## 3. Catalog Tables

### What Kinds of Information Live in Catalog Tables

Every major RDBMS exposes its own specific set of catalog tables (names vary by vendor), but they universally cover roughly the same categories:

| Category | Example Information Stored |
|---|---|
| **Tables** | Table names, schemas/namespaces they belong to, row count estimates, storage engine |
| **Columns** | Column names, data types, nullability, default values, ordinal position |
| **Constraints** | Primary keys, foreign keys, unique constraints, check constraints, and which columns they apply to |
| **Indexes** | Index names, which columns they cover, index type (e.g., B+ Tree, Hash) |
| **Views** | View definitions (the underlying `SELECT` statement) |
| **Users/Permissions** | Who can access what, and with what privileges (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, etc.) |
| **Stored Procedures/Triggers** | Procedural code definitions attached to the database |

### Vendor-Specific Catalog Examples

| RDBMS | Catalog Access Method |
|---|---|
| **PostgreSQL** | `pg_catalog` schema (e.g., `pg_class`, `pg_attribute`) |
| **MySQL** | `information_schema` (standard) + MySQL-specific `mysql` system database |
| **Oracle** | `ALL_TABLES`, `USER_TABLES`, `DBA_TABLES`, and the broader `SYS`/`SYSTEM` schemas |
| **SQL Server** | `sys` schema (e.g., `sys.tables`, `sys.columns`) + `INFORMATION_SCHEMA` |

### Interview Importance

**Medium** — knowing that vendor-specific catalogs exist (and roughly what they contain) is useful DBA-adjacent knowledge, more relevant for Senior/Staff and DBA-track interviews than junior ones.

---

## 4. INFORMATION_SCHEMA — The Standardized View

### Definition

`INFORMATION_SCHEMA` is a **SQL-standard**, **vendor-neutral** set of views providing access to catalog metadata — designed so that, in theory, the same metadata queries work across different RDBMS products (PostgreSQL, MySQL, SQL Server all support it, to varying degrees of completeness), unlike each vendor's own proprietary catalog tables (§3).

### Common, Practically Useful Queries

```sql
-- List all tables in the current database:
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public';  -- (schema name varies by system)

-- List all columns of a specific table, with their types:
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'student';

-- List all foreign key constraints in the database:
SELECT constraint_name, table_name, column_name
FROM information_schema.key_column_usage
WHERE referenced_table_name IS NOT NULL;
```

### Why This Matters Practically

- **Database exploration/onboarding:** A new engineer joining a team can query `INFORMATION_SCHEMA` to quickly understand an unfamiliar database's structure, without needing separate documentation (though good documentation, Module 14, is still valuable).
- **Tooling and automation:** ORMs, migration tools, and database GUI clients (like DBeaver, TablePlus, MySQL Workbench) all rely heavily on `INFORMATION_SCHEMA` (or vendor catalogs) under the hood to auto-generate their table browsers, auto-complete suggestions, and migration diffing tools.
- **Auditing and governance:** Security teams can query `INFORMATION_SCHEMA`-adjacent permission views to audit exactly who has access to what data across an entire organization's databases.

### Interview Importance

**Medium-High** — practical, hands-on `INFORMATION_SCHEMA` querying ability is a genuinely useful, sometimes directly-tested DBA/backend-engineer skill (e.g., "write a query to find all tables missing a primary key" — solvable via `INFORMATION_SCHEMA`).

---

## 5. Why the Data Dictionary Is Important

### For the DBMS Itself

- **Query parsing and validation:** Every SQL statement is checked against the catalog before execution — does the table exist? Does the column exist? Is the data type compatible?
- **Query optimization:** The query optimizer (a topic for a later Part) relies heavily on catalog statistics (row counts, index availability, data distribution estimates) to choose an efficient execution plan.
- **Constraint enforcement:** Recall Module 5, §15 — the DBMS checks constraints by consulting the catalog to know *which* constraints apply to a given table/column.

### For Human Engineers and Teams

- **Documentation of record:** Even with good external documentation (wikis, README files — Module 14's naming/consistency practices help here too), the system catalog is the **one source that can never lie or go stale** — it always reflects the *actual*, currently-enforced schema, unlike a wiki page someone forgot to update after a migration.
- **Impact analysis:** Before changing or dropping a column, an engineer can query the catalog (e.g., via `information_schema.key_column_usage`) to find every foreign key or view that depends on it — critical for safe schema migrations at scale.
- **Security auditing:** As mentioned in §4, permission-related catalog views let security/compliance teams verify access control policies are correctly applied.

### Interview Importance

**High** — being able to articulate *both* the DBMS-internal and human/team-facing value of the data dictionary demonstrates a well-rounded understanding beyond just "it's where table info is stored."

---

## 6. Worked Example: Exploring a Real Database's Metadata

Let's use `INFORMATION_SCHEMA` to answer several realistic, practical questions about the `University` database built across Modules 6-13.

```sql
-- Q1: What tables exist in this database?
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'university_db';

-- Q2: What are all the columns of the 'course' table, and are they nullable?
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'course';

-- Q3: Which tables have a foreign key referencing 'department'?
SELECT table_name, column_name
FROM information_schema.key_column_usage
WHERE referenced_table_name = 'department';

-- Q4: Which tables currently have NO primary key defined?
--     (A common, high-value data-quality/governance check!)
SELECT t.table_name
FROM information_schema.tables t
LEFT JOIN information_schema.table_constraints tc
    ON t.table_name = tc.table_name AND tc.constraint_type = 'PRIMARY KEY'
WHERE tc.constraint_name IS NULL AND t.table_schema = 'university_db';
```

### Why Q4 Is a Particularly Valuable, Real-World Query

Finding tables with **no primary key** is one of the most common, genuinely useful "schema health check" queries a real DBA or backend engineer runs when auditing an unfamiliar or legacy database — it directly flags a violation of Entity Integrity (Module 5, §5) waiting to cause problems.

### Interview Importance

**High** — being able to write a query like Q4 live, from memory, is a strong, practical, hands-on interview signal that goes beyond textbook definitions.

---

## 7. Chapter Summary

- **Metadata** is "data about data" — every ER/relational concept from earlier modules (schema, constraints, keys) is, formally, a piece of metadata.
- The **System Catalog** (data dictionary) is the DBMS's own internal record of all its metadata, and — elegantly — is itself typically implemented as ordinary relational tables, queryable with standard SQL.
- **Catalog tables** cover tables, columns, constraints, indexes, views, users/permissions, and stored procedures — every major RDBMS exposes its own vendor-specific catalog (`pg_catalog`, MySQL's system database, Oracle's `ALL_TABLES`/`DBA_TABLES`, SQL Server's `sys` schema).
- **`INFORMATION_SCHEMA`** is the SQL-standard, vendor-neutral way to query this same metadata, making it the most portable and broadly useful tool for real-world database exploration, tooling, auditing, and impact analysis.
- The data dictionary matters both **internally** (query parsing, optimization, constraint enforcement) and **externally**, for human engineers (always-accurate documentation, safe migration impact analysis, security auditing) — and it's the source of truth that, unlike a wiki, can never go stale.

---

## 8. Cheat Sheet

```
METADATA = "data about data" (schema, constraints, indexes — NOT the actual rows)

SYSTEM CATALOG (Data Dictionary):
  → the DBMS's own internal record of ALL its metadata
  → itself implemented as ORDINARY relational tables — query it with normal SQL!

VENDOR-SPECIFIC CATALOGS:
  PostgreSQL → pg_catalog (pg_class, pg_attribute, ...)
  MySQL      → information_schema + mysql system database
  Oracle     → ALL_TABLES / USER_TABLES / DBA_TABLES
  SQL Server → sys schema (sys.tables, sys.columns, ...)

INFORMATION_SCHEMA = SQL-STANDARD, vendor-neutral metadata views
  → information_schema.tables            (list of tables)
  → information_schema.columns           (columns, types, nullability)
  → information_schema.table_constraints (PK/FK/UNIQUE/CHECK constraints)
  → information_schema.key_column_usage  (FK relationships)

WHY IT MATTERS:
  DBMS-internal:  query parsing, optimization, constraint enforcement
  Human-facing:   always-accurate docs, migration impact analysis, security audits

CLASSIC PRACTICAL QUERY: "Find all tables with no primary key"
  → LEFT JOIN information_schema.tables with table_constraints
    WHERE constraint_type = 'PRIMARY KEY', filter for NULL matches
```

---

## 9. Exercises

### Beginner

1. Define "metadata" in one sentence, with an example.
2. True/False: The system catalog is stored in a completely different, non-relational format from ordinary data.
3. Fill in the blank: The SQL-standard, vendor-neutral way to query catalog metadata is called __________.

### Intermediate

4. Write an `INFORMATION_SCHEMA` query to list every column in a table called `employee`, along with its data type and whether it allows NULL.
5. Explain why "the query optimizer relies on the system catalog" — what specific information would it need, and why?
6. A team is about to drop a column from a `customer` table. Write a query approach (using `INFORMATION_SCHEMA`) to check whether any foreign keys reference that column first.

### Advanced / Conceptual

7. Explain the elegant self-consistency of the system catalog being implemented as ordinary relational tables. What practical benefit does this design choice provide to database administrators and tool builders?
8. Discuss why `INFORMATION_SCHEMA` is generally preferred over vendor-specific catalogs (like `pg_catalog` or SQL Server's `sys` schema) for portability, and identify one scenario where you might need to use a vendor-specific catalog instead because `INFORMATION_SCHEMA` doesn't expose enough detail.
9. Design and explain a query (conceptually, using `INFORMATION_SCHEMA`) that would help a security team audit which database users have `DELETE` privileges on any table containing a column named `salary`.

### MCQs

**Q1.** Metadata is best described as:
a) The actual rows of a table  b) Data describing the structure of data  c) A type of index  d) A backup of the database

**Q2.** INFORMATION_SCHEMA is:
a) A vendor-specific PostgreSQL-only feature  b) A SQL-standard, vendor-neutral metadata interface  c) A type of primary key  d) Only used for storing user passwords

**Q3.** The system catalog is typically implemented as:
a) A binary file outside the database  b) Ordinary relational tables, queryable with standard SQL  c) An external XML document  d) A feature only available in NoSQL systems

**Q4.** A key practical use of querying the data dictionary is:
a) Encrypting data  b) Finding tables with no primary key, or finding foreign key dependencies before a migration  c) Generating random test data  d) Compiling application code

*(Answer key: 1-b, 2-b, 3-b, 4-b)*

### Practice Exercise

10. If you have access to any running RDBMS (PostgreSQL, MySQL, or similar), connect to it and run `SELECT table_name FROM information_schema.tables WHERE table_schema = 'your_schema';` followed by a `information_schema.columns` query on one of the returned tables. Compare the output structure to what this module described.

### Practice Links

- [PostgreSQL Documentation — The Information Schema](https://www.postgresql.org/docs/current/information-schema.html)
- [MySQL Documentation — INFORMATION_SCHEMA](https://dev.mysql.com/doc/refman/8.0/en/information-schema.html)

---

**Next: Module 16 — Common Modeling Mistakes (Wrong Primary Keys, Duplicate Entities, Missing Relationships, Poor Naming, Incorrect Cardinality, Circular References, Over/Under-Normalization Preview).**

> Say "continue" or "next module" to proceed to Module 16.
