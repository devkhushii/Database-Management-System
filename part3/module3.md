# Part 3 — The Complete SQL Handbook
## Module 3: Database Operations (DDL)

> **Goal of this module:** Write your first real SQL. Learn how to create, alter, drop, and inspect **databases themselves** (not yet tables — that's Module 4). Understand the `information_schema`, the standardized "map" every relational database exposes describing its own structure.

---

### 📍 Where This Module Fits

```
Module 2: Environment ready  →  Module 3: Database-level DDL (You are here)  →  Module 4: Table-level DDL
```

---

## 3.1 What is DDL, Really?

Recall from **Module 1, Section 1.8**: **DDL (Data Definition Language)** is the SQL category responsible for defining and modifying the **structure** of database objects — not the data inside them, but the containers themselves: databases, tables, schemas, indexes, views.

### Why Does DDL Exist as a Separate Category?

This connects to Part 2's relational model: before you can *store or query* data, the database needs a **schema** — a formal description of what tables exist, what columns they have, what types those columns hold, and what rules (constraints) govern them. DDL is the language you use to declare that schema to the engine.

> 🧠 **Real-life analogy:** If DML (Module 7) is like *filling in* forms with information, DDL is like *designing the form itself* — deciding how many fields it has, what each field is called, and what type of answer each field expects (a date, a number, free text).

### DDL Statements Are Auto-Committing (Mostly)

A subtlety worth flagging early, expanded fully once we reach transactions in Part 4: in **MySQL**, DDL statements cause an **implicit commit** — they cannot be rolled back as part of a transaction. In **PostgreSQL**, most DDL statements are **transactional** — they *can* be wrapped in a transaction and rolled back if something goes wrong. This is a genuine, frequently-tested **PostgreSQL vs MySQL** difference.

| | PostgreSQL | MySQL |
|---|---|---|
| DDL inside a transaction | Generally rollback-able | Causes implicit commit (not rollback-able in most cases) |

---

## 3.2 CREATE DATABASE

### What It Does

Creates a new, empty **database** — a top-level container that will eventually hold schemas, tables, views, indexes, and all other objects.

### Syntax (ANSI Standard Core)

```sql
CREATE DATABASE database_name;
```

### PostgreSQL Example

```sql
CREATE DATABASE company_db
    WITH
    OWNER = postgres
    ENCODING = 'UTF8'
    CONNECTION LIMIT = -1;
```

### MySQL Example

```sql
CREATE DATABASE company_db
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

### Internal Working

When `CREATE DATABASE` runs:

1. The engine validates the name doesn't already exist (unless `IF NOT EXISTS` is used).
2. It allocates the necessary **catalog entries** — internal system tables that track metadata about this new database (its owner, encoding, collation, permissions).
3. **PostgreSQL** physically creates a new directory on disk (under its data directory) to hold this database's files — PostgreSQL databases are strongly isolated from each other at the storage level.
4. **MySQL** (with InnoDB) creates the database as a **schema/namespace**; depending on configuration (`innodb_file_per_table`), it may create a dedicated directory too, but the isolation model is looser — cross-database queries are much more common and natural in MySQL than in PostgreSQL.

> ⚠️ **This is a big, frequently-misunderstood PostgreSQL vs MySQL distinction:** In MySQL, "database" and "schema" are **synonyms** — `CREATE DATABASE` and `CREATE SCHEMA` do the exact same thing. In PostgreSQL, a **database** is the top-level isolation boundary, and a **schema** is a *namespace inside* a database (you'll meet schemas properly when we discuss namespacing in Module 4). You cannot query across two different PostgreSQL databases in a single query without special extensions (like `dblink` or `postgres_fdw`), but you *can* easily query across multiple MySQL "databases" in one query, because they're just namespaces within the same server. This single distinction resolves a huge amount of beginner confusion when moving between the two engines.

```
PostgreSQL:                              MySQL:
┌─────────────┐  ┌─────────────┐        ┌───────────────────────────┐
│  Database A │  │  Database B │        │        MySQL Server        │
│ (isolated)  │  │ (isolated)  │        │  ┌──────────┐ ┌──────────┐ │
│  schema1    │  │  schema1    │        │  │ database1│ │ database2│ │
│  schema2    │  │             │        │  │(=schema) │ │(=schema) │ │
└─────────────┘  └─────────────┘        │  └──────────┘ └──────────┘ │
   Cannot JOIN across databases          │   CAN JOIN across databases │
   directly in one query                  │   in one query easily       │
                                          └───────────────────────────┘
```

### IF NOT EXISTS — Defensive Creation

```sql
CREATE DATABASE IF NOT EXISTS company_db;   -- MySQL syntax
```

```sql
-- PostgreSQL does NOT support IF NOT EXISTS for CREATE DATABASE directly
-- (a documented, real quirk) — you typically check programmatically instead,
-- e.g., via a script, or by catching the "database already exists" error.
```

> ⚠️ **Common mistake / real gotcha:** Assuming `IF NOT EXISTS` works identically everywhere. It doesn't — PostgreSQL notably lacks it for `CREATE DATABASE` (though it supports it for `CREATE TABLE` and several other objects, as we'll see in Module 4). Always verify per-engine, per-statement support rather than assuming uniformity.

---

## 3.3 DROP DATABASE

### What It Does

Permanently deletes a database **and everything inside it** — every table, every row, every index, irreversibly (barring backups).

```sql
DROP DATABASE company_db;
```

**Defensive version:**

```sql
DROP DATABASE IF EXISTS company_db;   -- Both PostgreSQL and MySQL support this
```

### Internal Working

- The engine first checks no active connections are using the database (PostgreSQL will **refuse** to drop a database with active connections, throwing an error — MySQL is more permissive here).
- All associated catalog metadata is removed.
- All physical files/directories associated with that database are deleted from disk.

> ⚠️ **Critical safety note:** `DROP DATABASE` has **no undo**, and unlike `DELETE` (Module 7), it isn't something you can wrap in a transaction and roll back in MySQL (implicit commit, per Section 3.1), and even in PostgreSQL, dropping a database is a `DROP DATABASE` command that **cannot run inside a transaction block at all** — it's always immediate. In production environments, this command is almost always **restricted to DBAs/senior engineers** via permission controls (DCL, covered later), precisely because of how catastrophic an accidental run can be.

### Real-World Best Practice

```sql
-- Always verify you're connected to the RIGHT server/environment
-- before running DROP DATABASE. A shockingly common real incident:
-- running a "cleanup" DROP DATABASE against production because a
-- terminal tab was still connected to the wrong host.
```

---

## 3.4 ALTER DATABASE

### What It Does

Modifies properties of an **existing** database — not its data or tables, but database-level settings like owner, connection limits, or default configuration parameters.

### PostgreSQL Examples

```sql
-- Change the owner
ALTER DATABASE company_db OWNER TO new_admin;

-- Rename a database
ALTER DATABASE company_db RENAME TO company_database;

-- Set a connection limit
ALTER DATABASE company_db WITH CONNECTION LIMIT = 100;
```

### MySQL Examples

```sql
-- MySQL's ALTER DATABASE is much more limited — mainly character set/collation
ALTER DATABASE company_db
    CHARACTER SET = utf8mb4
    COLLATE = utf8mb4_unicode_ci;
```

> 💡 **PostgreSQL vs MySQL:** PostgreSQL's `ALTER DATABASE` supports a richer set of database-level configuration (ownership, connection limits, per-database config parameter overrides via `SET`). MySQL's is intentionally narrower, mostly limited to character set/collation — this reflects the deeper architectural difference from Section 3.2: PostgreSQL treats each database as a much more independently configurable unit.

---

## 3.5 USE DATABASE (MySQL) — and Its Absence in PostgreSQL

### MySQL

```sql
USE company_db;
```

This tells the session: "all subsequent unqualified table references refer to `company_db` unless otherwise specified." It's a **session-level context switch**, not a persistent change — it only affects your current connection.

### PostgreSQL — No `USE` Statement

This surprises almost every beginner coming from MySQL. In PostgreSQL, you **cannot switch databases within an active session** at all. To work with a different database, you must **reconnect** to it:

```bash
psql -d another_database
```

or, inside `psql`:

```
\c another_database
```

(`\c` is a `psql` **meta-command**, not SQL — it tells the *client* to open a new connection, not the server to switch context.)

> ⚠️ **This is one of the single most common "gotcha" differences tested in interviews and hit in real practice:** Because PostgreSQL databases are fully isolated (Section 3.2), there's no server-side concept of "switching" — a *connection* is bound to exactly one database for its entire lifetime. MySQL's looser database-as-namespace model is precisely what makes `USE` meaningful there.

```
MySQL:  one connection --- USE db1 ---> USE db2 ---> (same connection, different context)

Postgres: one connection ---> permanently bound to db1
          (need a NEW connection entirely to reach db2)
```

---

## 3.6 SHOW DATABASES / Listing Databases

### MySQL

```sql
SHOW DATABASES;
```

### PostgreSQL

PostgreSQL has no `SHOW DATABASES` (it's not part of the ANSI standard, and MySQL's `SHOW` family is a proprietary MySQL extension throughout). Instead:

```sql
-- Standard SQL-ish approach: query the catalog
SELECT datname FROM pg_database;
```

```
-- Or, using the psql meta-command (client-side, not real SQL):
\l
```

> 💡 **Meta-commands vs SQL:** It's worth explicitly noting the difference between a **SQL statement** (`SELECT`, `CREATE`, etc. — understood by the *server*, portable across any client) and a **client meta-command** (`\l`, `\c`, `\dt` in `psql`; these are shortcuts interpreted by the *psql client itself* and don't exist as "real SQL" you could send from, say, a Python script using a database driver). MySQL's `SHOW` commands are actually a middle ground: they *are* real SQL statements (the server understands them directly), just proprietary, non-ANSI-standard ones.

---

## 3.7 Information Schema — The Standardized System Catalog

### What Is It?

The **`information_schema`** is a special, **ANSI SQL-standard schema** that every standards-compliant relational database exposes, containing metadata (data *about* data) — descriptions of every table, column, constraint, and other object in the database, represented as regular queryable tables/views.

> 🧠 **Real-life analogy:** If your database is a library, `information_schema` is the library's own **catalog system** — not the books themselves, but a structured index describing what books exist, which shelf they're on, how many pages each has, and so on. Crucially, this catalog is itself something you can "read" using the exact same tools (SQL queries) you'd use to read the books.

### Why It Exists

Before `information_schema` was standardized (introduced in SQL-92), every vendor had completely proprietary, incompatible ways to introspect database structure. Standardizing it means the same query can retrieve schema metadata across **any** ANSI-compliant engine — a genuinely portable piece of SQL, unlike most DDL syntax itself.

### Key Views Inside `information_schema`

| View | Contains |
|---|---|
| `information_schema.tables` | Every table (and view) in the database, with its schema, name, and type |
| `information_schema.columns` | Every column of every table: name, data type, nullability, default value, position |
| `information_schema.table_constraints` | Every constraint (PK, FK, UNIQUE, CHECK) attached to tables |
| `information_schema.key_column_usage` | Which columns participate in which keys/constraints |
| `information_schema.referential_constraints` | Foreign key relationships specifically |
| `information_schema.schemata` | Every schema in the database |

### Practical Examples

**List all tables in the current database:**
```sql
SELECT table_name, table_type
FROM information_schema.tables
WHERE table_schema = 'public';   -- 'public' is PostgreSQL's default schema
```

**List all columns of a specific table, with their types:**
```sql
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'employees'
ORDER BY ordinal_position;
```

**Sample output (ASCII table):**
```
 column_name |     data_type      | is_nullable | column_default
--------------+--------------------+-------------+----------------
 id           | integer            | NO          | nextval(...)
 name         | character varying  | NO          | NULL
 department   | character varying  | YES         | NULL
 salary       | numeric            | YES         | 0
```

### PostgreSQL vs MySQL — Information Schema Support

| Aspect | PostgreSQL | MySQL |
|---|---|---|
| `information_schema` support | Full, standards-compliant | Full, standards-compliant |
| Proprietary alternative | `pg_catalog` (PostgreSQL's own, richer, non-standard system catalog — often faster and more detailed than `information_schema`) | `SHOW` command family (`SHOW TABLES`, `SHOW COLUMNS`, etc. — proprietary, quicker to type, less portable) |
| Recommended for portability | `information_schema` | `information_schema` |
| Recommended for deep internals | `pg_catalog` | `SHOW` / `performance_schema` |

> 💡 **Best practice:** For portable scripts/tools meant to work across multiple database engines (e.g., a generic schema-migration tool), always prefer `information_schema`. For quick, interactive, engine-specific exploration, the native shortcuts (`\dt`, `\d table_name` in psql; `SHOW TABLES`, `DESCRIBE table_name` in MySQL) are faster to type and often show more engine-specific detail.

### Quick Reference — Native Inspection Shortcuts

| Task | psql (PostgreSQL) | MySQL CLI |
|---|---|---|
| List databases | `\l` | `SHOW DATABASES;` |
| Switch database | `\c dbname` (new connection) | `USE dbname;` |
| List tables | `\dt` | `SHOW TABLES;` |
| Describe a table | `\d tablename` | `DESCRIBE tablename;` or `SHOW COLUMNS FROM tablename;` |
| List schemas | `\dn` | *(not applicable — schema = database in MySQL)* |

---

## 3.8 Full Worked Example — Database Lifecycle End to End

```sql
-- 1. Create a database (PostgreSQL)
CREATE DATABASE bookstore;

-- 2. Connect to it (psql meta-command)
\c bookstore

-- 3. Inspect what exists (should be empty except system schemas)
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public';
-- (empty result — no user tables yet; that's Module 4's job)

-- 4. Rename it
ALTER DATABASE bookstore RENAME TO bookstore_v2;

-- 5. Eventually, clean up
DROP DATABASE IF EXISTS bookstore_v2;
```

```
Execution Timeline:
CREATE DATABASE ──► \c (reconnect) ──► query information_schema ──► ALTER (rename) ──► DROP DATABASE
     │                     │                       │                       │                 │
  catalog entry      session rebinds         reads catalog            catalog updated   catalog + files
  + disk directory      to new DB             metadata                                    removed
  created (PG)
```

---

## 3.9 Advantages of Database-Level DDL Organization

| Advantage | Explanation |
|---|---|
| **Logical isolation** | Separate databases let you cleanly separate unrelated applications/environments (e.g., `app_dev`, `app_test`, `app_prod` as distinct databases) |
| **Independent configuration** | Each database can have its own encoding, collation, and (in Postgres) connection limits |
| **Security boundary** | Permissions can be scoped per database, limiting blast radius of compromised credentials |
| **Standardized introspection** | `information_schema` gives a portable way to programmatically inspect structure — powers tools like ORMs, migration frameworks, and database GUIs themselves |

## 3.10 Disadvantages / Limitations

| Limitation | Explanation |
|---|---|
| Cross-database queries are hard in PostgreSQL | Requires `dblink`/`postgres_fdw` extensions; not a native single-query JOIN |
| `DROP DATABASE` is irreversible | No native "trash bin" — recovery depends entirely on backups |
| MySQL's database=schema conflation | Can lead to accidental cross-database queries/joins that wouldn't be possible (and thus wouldn't be accidentally written) in PostgreSQL |

## 3.11 Best Practices

- Use consistent, environment-suffixed naming for non-production databases (`app_dev`, `app_staging`) to reduce the risk of running destructive commands against the wrong one.
- Restrict `CREATE DATABASE` / `DROP DATABASE` privileges to DBA/admin roles in any shared environment (foreshadowing DCL).
- Always query `information_schema` (not hardcoded assumptions) when building tools/scripts that need to introspect schema — assumptions about column order or existence break silently otherwise.
- Prefer `IF NOT EXISTS` / `IF EXISTS` guards in scripts that may be re-run (idempotent migrations), where the engine supports it.

## 3.12 Common Mistakes

- Running `DROP DATABASE` against the wrong host/environment because of an open terminal tab — always double-check your connection target first.
- Assuming `USE` works in PostgreSQL (it doesn't — Section 3.5).
- Assuming `information_schema` and a vendor's native catalog (`pg_catalog`, MySQL's `SHOW`) always return identical information — the standard schema is a common *subset*, not a perfect mirror, of what each engine actually tracks internally.

## 3.13 Edge Cases

- PostgreSQL will refuse `DROP DATABASE` if there are active connections to it (you'll need to terminate them first, e.g., via `pg_terminate_backend()`, a function we'll return to when discussing administration).
- Renaming a database that active applications are currently connected to can break those connections mid-session, since the old name becomes instantly invalid.
- MySQL's `CHARACTER SET`/`COLLATE` choices at the database level are only *defaults* — individual tables/columns can override them (previewed now, covered fully with data types in Module 5).

## 3.14 Performance Implications

Database-level DDL operations (`CREATE`, `DROP`, `ALTER DATABASE`) are relatively rare, administrative operations — not something executed in application hot paths — so their direct query performance is less of a concern than table-level DDL (Module 4) or DML/DQL performance (Modules 7+). Their main "performance" consideration is operational: `DROP DATABASE` on a very large database can take meaningful time as it deallocates large amounts of disk space.

## 3.15 Industry Usage

In real production systems, engineers rarely run these commands manually and interactively — they're typically embedded in **infrastructure-as-code** or **migration tooling** (Terraform, Flyway, Liquibase, Django/Rails migrations) that programmatically creates databases as part of environment provisioning, precisely because of the destructive risk discussed in Section 3.3.

## 3.16 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Syntax recall: `CREATE`, `DROP`, `ALTER DATABASE` |
| Placement | "What's the difference between a database and a schema in Postgres vs MySQL?" |
| Mid-level | "Why doesn't PostgreSQL have `USE`?" / "What's `information_schema` used for?" |
| Senior SDE-2/SDE-3 | "How would you build a database-agnostic schema introspection tool?" / "Why is `DROP DATABASE` dangerous in production, and how do real systems guard against it?" |

---

## 3.17 Chapter Summary

- **DDL** defines structure, not data — and in MySQL, DDL auto-commits, while much of PostgreSQL's DDL is transactional.
- `CREATE DATABASE`, `DROP DATABASE`, and `ALTER DATABASE` manage databases as top-level containers — but PostgreSQL treats databases as **strongly isolated**, while MySQL treats "database" as a **synonym for schema**, a namespace within one server.
- PostgreSQL has **no `USE` statement** — connections are permanently bound to one database; MySQL's `USE` freely switches context within a session.
- **`information_schema`** is the ANSI-standard, portable way to introspect database structure across any compliant engine; `pg_catalog` (Postgres) and `SHOW` (MySQL) are faster but proprietary alternatives.
- `DROP DATABASE` is irreversible and should be tightly access-controlled in any real environment.

---

## 3.18 Revision Notes / Cheat Sheet

```sql
CREATE DATABASE db_name;                 -- create
DROP DATABASE IF EXISTS db_name;         -- delete (irreversible!)
ALTER DATABASE db_name RENAME TO new_name;  -- rename (Postgres)

-- MySQL only:
USE db_name;
SHOW DATABASES;

-- PostgreSQL only:
\c db_name        -- reconnect (psql meta-command)
\l                 -- list databases (psql meta-command)
SELECT datname FROM pg_database;  -- portable-ish SQL equivalent

-- ANSI Standard, works everywhere:
SELECT table_name FROM information_schema.tables WHERE table_schema = '...';
SELECT column_name, data_type FROM information_schema.columns WHERE table_name = '...';
```

---

## 3.19 Multiple Choice Questions (MCQs)

1. In MySQL, "database" and "schema" are:
   a) Completely different concepts b) **Synonyms** c) Only related in Postgres d) Deprecated terms

2. Which statement does PostgreSQL **NOT** support?
   a) `CREATE DATABASE` b) `ALTER DATABASE ... RENAME TO` c) **`USE database_name;`** d) `DROP DATABASE IF EXISTS`

3. `information_schema` is:
   a) A MySQL-only proprietary feature b) **An ANSI SQL standard schema for metadata** c) A GUI tool d) A type of index

4. Which of these is a `psql` **meta-command**, not real SQL?
   a) `SELECT * FROM pg_database;` b) **`\c mydb`** c) `CREATE DATABASE mydb;` d) `information_schema.tables`

5. Why does PostgreSQL refuse to drop a database with active connections?
   a) It's a licensing restriction b) **To prevent data corruption from concurrent access during deletion** c) It's a bug d) It never refuses

---

## 3.20 Exercises

### 🟢 Beginner
1. Create a database called `practice_db` in both PostgreSQL and MySQL.
2. In MySQL, switch into `practice_db` using `USE`, then list all databases on the server.
3. In PostgreSQL, attempt to `\c` into `practice_db` from a different database connection, and observe what changes.

### 🟡 Intermediate
4. Query `information_schema.tables` in both PostgreSQL and MySQL against your sample databases (Pagila/Sakila from Module 2) and compare the output structure.
5. Rename `practice_db` in PostgreSQL using `ALTER DATABASE ... RENAME TO`, then verify the rename by listing databases again.
6. Write a query against `information_schema.columns` that lists every column and data type for one table of your choice from Sakila or Pagila.

### 🔴 Advanced / Debugging
7. Explain, using this module's content, why the following MySQL query has no direct PostgreSQL equivalent without extensions: `SELECT * FROM db1.customers JOIN db2.orders ON ...`
8. You need to drop a PostgreSQL database but get an error about active connections. Research (or recall from this module) the function used to terminate those connections first, and write out the full command sequence you'd use.

### 🧩 Scenario-Based
9. Your team maintains `app_dev`, `app_staging`, and `app_prod` as separate PostgreSQL databases on the same server. Explain, referencing Section 3.9–3.10, both the safety benefits and the practical limitations (e.g., cross-environment reporting) of this setup.

### 🏗️ Mini Project
10. Write a single script (a `.sql` file) that: creates a new database, connects to it (documenting the correct approach for both PostgreSQL and MySQL separately, since the syntax diverges), queries `information_schema.tables` to confirm it's empty, and then safely drops it using an `IF EXISTS` guard.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [PostgreSQL Official Docs — CREATE DATABASE](https://www.postgresql.org/docs/current/sql-createdatabase.html)
- [MySQL Official Docs — CREATE DATABASE](https://dev.mysql.com/doc/refman/8.0/en/create-database.html)

---

## ✅ Module 3 Complete

You can now create, modify, inspect, and safely tear down databases themselves, and — critically — you understand the deep architectural difference between how PostgreSQL and MySQL model "databases" vs. "schemas," a distinction that will keep paying off throughout the rest of this handbook.

**Next up: Module 4 — Table Operations**, where you'll create your first real tables, and encounter auto-increment columns, generated columns, and temporary tables.

---

*Reply "continue" or "next module" when you're ready for Module 4.*
