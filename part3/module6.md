# Part 3 — The Complete SQL Handbook
## Module 6: Constraints

> **Goal of this module:** Learn every SQL constraint type — `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `CHECK`, `NOT NULL`, `DEFAULT`, `GENERATED ALWAYS` — and the referential actions (`CASCADE`, `RESTRICT`, `SET NULL`, `SET DEFAULT`, `NO ACTION`) that govern what happens when related data changes.

---

### 📍 Where This Module Fits

```
Module 5: What data CAN be  →  Module 6: What data MUST/MUST NOT be (You are here)  →  Module 7: DML
```

---

## 6.1 What Are Constraints and Why Do They Exist?

A **constraint** is a rule the database engine **enforces automatically**, rejecting any `INSERT`/`UPDATE` that would violate it — data integrity guaranteed at the *storage layer itself*, not left to application code discipline.

> 🧠 **Real-life analogy:** Constraints are like the physical slots in a vending machine — you cannot physically insert a coin of the wrong shape/size, no matter how the buttons (application logic) are pressed. Even if your application code has a bug that tries to insert bad data, a well-designed constraint layer is the database's last line of defense, refusing the invalid write outright.

### Why Enforce Rules at the Database Level (Not Just in Application Code)?

This is a classic, important interview discussion point:

| Enforce in application code only | Enforce as a database constraint |
|---|---|
| Every application/service touching the database must remember to implement the check | The rule is enforced **once, centrally**, regardless of how many applications write to the table |
| Bugs, bypassed validation, or direct database access (scripts, admin tools, other services) can silently violate the rule | **Impossible to bypass** — even a raw `INSERT` from a database console is checked |
| Race conditions between concurrent requests can slip invalid data through | The database enforces constraints **atomically**, even under concurrent writes |

> 💡 **Best practice consensus:** Validate early in application code for good user experience (immediate, friendly error messages) — but **always** also enforce genuinely critical integrity rules as database constraints, since application-level validation alone is never a complete guarantee.

---

## 6.2 PRIMARY KEY

### What It Does

Designates one (or more) columns as the **unique, non-null identifier** for every row in a table.

```sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

-- Composite primary key (multiple columns together)
CREATE TABLE enrollments (
    student_id INT,
    course_id  INT,
    PRIMARY KEY (student_id, course_id)
);
```

### Internal Working

A `PRIMARY KEY` constraint is really **two constraints bundled together**:
1. **`UNIQUE`** — no two rows may share the same value(s)
2. **`NOT NULL`** — the value can never be missing

Both PostgreSQL and MySQL implement `PRIMARY KEY` by automatically creating a **unique index** (Part 4 covers indexes fully) on the constrained column(s) — this is precisely *why* primary key lookups are fast: you're not just getting a logical guarantee, you're getting a physical, indexed fast-path to find that row.

> ⚠️ **A real, commonly-tested rule:** A table can have **only one** `PRIMARY KEY` (though it can span multiple columns as a composite key), but it can have **multiple** `UNIQUE` constraints. Interviewers sometimes probe "why can't you have two primary keys?" — the honest answer is that "primary key" is definitionally the *one* designated main identifier; having two would defeat the purpose of "the" primary identifying key, even though nothing stops you from having several independently-unique columns via separate `UNIQUE` constraints.

### PostgreSQL vs MySQL

| Aspect | PostgreSQL | MySQL (InnoDB) |
|---|---|---|
| Physical row storage | Rows generally stored independent of primary key order (heap-organized table) | InnoDB is a **clustered index** engine — rows are **physically stored in primary key order** on disk |

> ⚠️ **Genuinely important internals difference, frequently tested at senior level:** Because InnoDB **clusters** data by primary key, primary key choice in MySQL has a *much bigger* direct impact on physical storage layout and insert performance than in PostgreSQL. This is precisely why the "random UUID hurts insert performance" concern from **Module 5, Section 5.7** is measurably more severe in MySQL/InnoDB than in PostgreSQL — random-order inserts into a clustered index cause page splits and fragmentation throughout the *entire row storage*, not just a separate index structure. This is a genuine, real "why" behind a commonly-cited but often superficially-understood best practice.

---

## 6.3 FOREIGN KEY

### What It Does

Enforces that a column's value must match an existing value in another table's (typically primary key) column — the mechanism that implements **relationships** between tables, directly realizing Part 2's ER modeling concepts in actual enforced SQL.

```sql
CREATE TABLE departments (
    id   INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE employees (
    id            INT PRIMARY KEY,
    name          VARCHAR(100),
    department_id INT REFERENCES departments(id)
);

-- Equivalent, more explicit syntax:
CREATE TABLE employees (
    id            INT PRIMARY KEY,
    name          VARCHAR(100),
    department_id INT,
    CONSTRAINT fk_department FOREIGN KEY (department_id) REFERENCES departments(id)
);
```

### Why It Exists

Without a foreign key constraint, nothing stops an `employees.department_id` value from pointing to a department that doesn't exist — an **orphaned reference**, silently corrupting the logical relationships your schema is supposed to represent. The foreign key constraint makes this **physically impossible** at the database level.

```
Attempted: INSERT INTO employees (id, name, department_id) VALUES (5, 'Kiran', 999);

If department id=999 doesn't exist in `departments`:
ERROR: insert or update on table "employees" violates foreign key
       constraint "fk_department" — Key (department_id)=(999) is not
       present in table "departments".
```

### Internal Working

On every `INSERT`/`UPDATE` to the referencing (child) table's foreign key column, the engine performs a **lookup** against the referenced (parent) table's key to confirm the value exists. This lookup is why **foreign key columns should almost always be indexed** — without an index on the parent's referenced column (typically automatic, since it's usually the parent's primary key) and ideally on the child's FK column too, this validation check (and any subsequent cascading operations) can become a real performance bottleneck on large tables.

> ⚠️ **Common mistake:** Forgetting to index the **child** table's foreign key column itself (the parent side is auto-indexed via its primary key, but the child side often isn't automatic). Without it, operations like "find all employees in department X" or cascading deletes (Section 6.4) require a full table scan of the child table — a genuinely common, real performance issue in production schemas.

---

## 6.4 Referential Actions: CASCADE, RESTRICT, SET NULL, SET DEFAULT, NO ACTION

These clauses define **what happens to child rows when a referenced parent row is deleted or updated** — a frequently-tested, practically crucial part of foreign key design.

```sql
CREATE TABLE employees (
    id            INT PRIMARY KEY,
    department_id INT REFERENCES departments(id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

| Action | Behavior on `DELETE`/`UPDATE` of the parent row |
|---|---|
| **CASCADE** | Automatically **delete (or update) all matching child rows too** — the deletion "cascades" down |
| **RESTRICT** | **Immediately reject** the parent delete/update if any child rows reference it — the strictest option |
| **NO ACTION** | Similar to `RESTRICT`, but the check can be **deferred** until the end of the transaction (a subtle, standards-defined distinction — see edge cases below); this is the **ANSI SQL default** if no action is specified |
| **SET NULL** | Set the child's foreign key column to `NULL` when the parent is deleted/updated |
| **SET DEFAULT** | Set the child's foreign key column to its declared `DEFAULT` value when the parent is deleted/updated |

### Visual: Effect of Each Action

```
Parent table: departments (id=5, 'Engineering')
Child table: employees (id=1, department_id=5), (id=2, department_id=5)

DELETE FROM departments WHERE id = 5;

ON DELETE CASCADE:      → employees rows with department_id=5 are ALSO deleted
ON DELETE RESTRICT:     → the DELETE on departments is REJECTED with an error
ON DELETE NO ACTION:    → same practical effect as RESTRICT in most simple cases
                           (differs mainly in deferred-constraint-checking scenarios)
ON DELETE SET NULL:     → employees rows KEPT, department_id set to NULL
ON DELETE SET DEFAULT:  → employees rows KEPT, department_id set to its DEFAULT value
```

> 🧠 **Real-life analogy:** Think of a company reorganizing. If a department is dissolved:
> - **CASCADE** = every employee in that department is also let go (harsh, but sometimes correct — e.g., deleting a `shopping_cart` should cascade-delete its `cart_items`, since items can't meaningfully exist without a cart).
> - **RESTRICT/NO ACTION** = HR refuses to dissolve the department until every employee is reassigned first (protective — e.g., you shouldn't be able to delete a `customer` who still has unpaid `orders` without deliberately handling those orders first).
> - **SET NULL** = employees keep their jobs, but their "department" field is now blank, to be reassigned later (e.g., an `orders.assigned_employee_id` should probably go `NULL`, not vanish, if that employee record is deleted).
> - **SET DEFAULT** = employees are automatically reassigned to some designated fallback/default department.

### Choosing the Right Action — A Real Design Decision

> ⚠️ **This is genuinely one of the most consequential, frequently-interviewed schema design decisions.** Choosing `CASCADE` carelessly can lead to catastrophic, silent data loss (deleting one row unexpectedly wipes out large amounts of related data down a chain of cascades). Choosing `RESTRICT` everywhere can make legitimate cleanup operations painfully manual. The right choice genuinely depends on the **real-world semantics** of the relationship — "does the child row have any meaning without the parent?" A `cart_item` without a `cart` is meaningless (→ `CASCADE` is reasonable); an `order` without a `customer` might still need to be retained for accounting/audit purposes (→ `RESTRICT` or `SET NULL` is safer).

### PostgreSQL vs MySQL

| Aspect | PostgreSQL | MySQL (InnoDB) |
|---|---|---|
| Supported actions | `CASCADE`, `RESTRICT`, `NO ACTION`, `SET NULL`, `SET DEFAULT` — full standard support | `CASCADE`, `RESTRICT`, `NO ACTION`, `SET NULL` — **`SET DEFAULT` is parsed but not actually enforced** in InnoDB (a real, documented limitation) |
| Storage engine requirement | N/A (built into the core engine) | Foreign keys are **only enforced by the InnoDB storage engine** — the older `MyISAM` storage engine **silently ignores** foreign key constraints entirely (parses the syntax without error, but never enforces it) |

> ⚠️ **Real, occasionally-costly MySQL gotcha:** If a MySQL table is accidentally created using the `MyISAM` engine (or migrated/exported in a way that changes engines) instead of `InnoDB`, any `FOREIGN KEY` clauses are **silently accepted syntactically but never actually enforced** — no error, just quietly-unenforced referential integrity. Always verify `ENGINE=InnoDB` when foreign key integrity matters in MySQL.

---

## 6.5 UNIQUE

```sql
CREATE TABLE users (
    id    INT PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);

-- Composite uniqueness (the combination must be unique, individual columns need not be)
CREATE TABLE course_schedule (
    room       VARCHAR(20),
    time_slot  TIME,
    UNIQUE (room, time_slot)
);
```

**What it does:** Ensures no two rows share the same value (or combination of values, for a composite `UNIQUE` constraint) — but, unlike `PRIMARY KEY`, **`NULL` values are allowed**, and most engines treat multiple `NULL`s as **not violating uniqueness** (since `NULL` represents "unknown," and two unknowns aren't considered provably equal to each other — a concept we'll fully formalize in Module 9's `NULL` handling).

> 💡 **PostgreSQL vs MySQL nuance:** Both allow multiple `NULL`s in a `UNIQUE` column by default, consistent with the ANSI standard's treatment of `NULL` as "unknown, not comparably equal." This is occasionally surprising to beginners who assume `UNIQUE` should treat `NULL` like any other value requiring exact-match uniqueness.

---

## 6.6 CHECK

```sql
CREATE TABLE products (
    id    INT PRIMARY KEY,
    price NUMERIC(10,2) CHECK (price >= 0),
    discount_pct INT CHECK (discount_pct BETWEEN 0 AND 100)
);

-- Table-level CHECK referencing multiple columns
CREATE TABLE bookings (
    check_in  DATE,
    check_out DATE,
    CONSTRAINT valid_dates CHECK (check_out > check_in)
);
```

**What it does:** Enforces an arbitrary boolean expression that every row must satisfy — the most **general-purpose** constraint type, letting you encode business rules directly at the schema level (recall **Module 5, Section 5.2**'s note that PostgreSQL's lack of `UNSIGNED` integers is idiomatically solved with `CHECK (col >= 0)`).

### MySQL History Note

Prior to **MySQL 8.0.16**, `CHECK` constraints were **parsed but silently ignored** — a long-standing, widely-known limitation (similar in spirit to the `MyISAM` foreign-key gotcha above). Since 8.0.16, MySQL genuinely enforces `CHECK` constraints. This version-sensitivity is worth knowing, since plenty of production systems still run older MySQL versions where this trap is very real.

---

## 6.7 NOT NULL

```sql
name VARCHAR(100) NOT NULL;
```

**What it does:** Simply forbids `NULL` as a value for that column — the simplest constraint, but foundational, since `NULL` (representing "unknown"/"missing") has genuinely special, non-intuitive behavior throughout SQL (fully explored in Module 9 and revisited in Module 17's common mistakes).

> 💡 **Best practice:** Default to `NOT NULL` for any column where "missing" genuinely shouldn't be a valid state (e.g., a user's `email` in a system requiring one) — over-permissive nullability is a very common source of downstream bugs, forcing every query and every piece of application code to defensively handle a "missing" case that arguably should never have been allowed to occur.

---

## 6.8 DEFAULT

```sql
status     VARCHAR(20) DEFAULT 'pending',
created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
is_active  BOOLEAN DEFAULT TRUE
```

**What it does:** Supplies an automatic value when `INSERT` omits that column entirely (or explicitly specifies the `DEFAULT` keyword). Not strictly an "integrity constraint" in the same sense as the others (it doesn't *reject* invalid data), but conventionally grouped alongside constraints since it's declared in the same `CREATE TABLE`/`ALTER TABLE` syntax and directly shapes what values end up stored.

> ⚠️ **Common mistake:** Confusing `DEFAULT` with `NOT NULL`. A column can be `NOT NULL` **without** a `DEFAULT` — meaning every `INSERT` *must* explicitly supply a value, or it fails. A column can also have a `DEFAULT` but still allow explicit `NULL` to be inserted (unless also marked `NOT NULL`). These are two independent, composable properties, not variations of the same rule.

---

## 6.9 GENERATED ALWAYS

Covered in depth already for **generated/computed columns** (**Module 4, Section 4.10**). Worth reinforcing here as a constraint-adjacent concept: `GENERATED ALWAYS AS (...)` (computed columns) and `GENERATED ALWAYS AS IDENTITY` (auto-incrementing keys, **Module 4, Section 4.9**) both fall under this same SQL-standard keyword, distinguished by whether they compute a **value from other columns** or **generate a new sequential identity value**.

```sql
total   NUMERIC GENERATED ALWAYS AS (quantity * unit_price) STORED;   -- computed column
id      INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY;                  -- identity column
```

---

## 6.10 Full Worked Example — All Constraints Together

```sql
CREATE TABLE departments (
    id   INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE employees (
    id            INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(255) NOT NULL UNIQUE,
    salary        NUMERIC(10,2) NOT NULL CHECK (salary >= 0),
    department_id INT REFERENCES departments(id)
                       ON DELETE SET NULL
                       ON UPDATE CASCADE,
    status        VARCHAR(20) NOT NULL DEFAULT 'active'
                       CHECK (status IN ('active', 'on_leave', 'terminated')),
    hired_on      DATE NOT NULL DEFAULT CURRENT_DATE
);
```

```
Constraint Summary for `employees`:
┌───────────────┬─────────────────────────────────────────────┐
│ id             │ PRIMARY KEY (implies UNIQUE + NOT NULL)      │
│ name           │ NOT NULL                                     │
│ email          │ NOT NULL + UNIQUE                            │
│ salary         │ NOT NULL + CHECK (>= 0)                      │
│ department_id  │ FOREIGN KEY → departments(id), SET NULL/CASCADE │
│ status         │ NOT NULL + DEFAULT + CHECK (allowed values)   │
│ hired_on       │ NOT NULL + DEFAULT (current date)             │
└───────────────┴─────────────────────────────────────────────┘
```

---

## 6.11 Advantages of Enforcing Constraints

| Advantage | Explanation |
|---|---|
| Guaranteed integrity | Impossible to bypass, regardless of which application/service writes to the table |
| Self-documenting schema | Constraints act as living documentation of business rules — a new engineer reading the `CREATE TABLE` statement understands the rules immediately |
| Query optimizer awareness | The optimizer (Module 1) can sometimes use constraint information (e.g., `NOT NULL`, foreign key relationships) to make smarter execution plan choices |
| Prevents entire classes of bugs | Orphaned records, negative prices, duplicate emails — all become structurally impossible rather than "hopefully caught by application code" |

## 6.12 Disadvantages / Costs

| Disadvantage | Explanation |
|---|---|
| Write performance overhead | Every constraint check adds (typically small, but nonzero) overhead to `INSERT`/`UPDATE` operations |
| Schema rigidity | Overly strict constraints can make legitimate edge-case data awkward to represent, requiring careful upfront design |
| Migration friction | Adding a new `NOT NULL` or `CHECK` constraint to an existing large table with pre-existing violating data requires a cleanup pass first (ties back to **Module 4, Section 4.4**'s discussion of costly `ALTER TABLE` operations) |

## 6.13 Best Practices

- Always enforce genuinely critical business rules (uniqueness, referential integrity, non-negativity) as real database constraints — don't rely solely on application-level validation.
- Index foreign key columns on the **child** table side explicitly (Section 6.3) — it's not automatic in most engines.
- Choose `ON DELETE`/`ON UPDATE` actions deliberately based on real-world relationship semantics (Section 6.4), not by default/habit.
- Verify `ENGINE=InnoDB` in MySQL whenever foreign key enforcement matters (Section 6.4).
- Name your constraints explicitly (`CONSTRAINT fk_department FOREIGN KEY ...`) rather than relying on auto-generated names — makes future `ALTER TABLE ... DROP CONSTRAINT` operations and error messages far more readable.

## 6.14 Common Mistakes

- Assuming `CHECK` constraints are enforced on any MySQL version without verifying (pre-8.0.16 silently ignores them, Section 6.6).
- Using `ON DELETE CASCADE` without fully thinking through the "blast radius" of a single deletion cascading through multiple related tables.
- Forgetting `NOT NULL` on columns that should never logically be empty, leading to defensive `NULL`-handling code scattered everywhere downstream.
- Believing MyISAM tables enforce foreign keys the same way InnoDB does (Section 6.4) — they don't, silently.

## 6.15 Edge Cases

- `NO ACTION` vs `RESTRICT` differ subtly: `NO ACTION` constraints can, per the ANSI standard, be declared **`DEFERRABLE`** (checked only at transaction commit rather than immediately) — genuinely useful for certain multi-step operations that temporarily violate referential integrity mid-transaction before resolving it by commit time. PostgreSQL supports deferrable constraints; MySQL does not.
- Multiple `NULL`s are allowed in `UNIQUE` columns (Section 6.5) — a frequently-surprising, standards-compliant behavior.
- Composite foreign keys (referencing multiple columns together) must reference a matching composite `UNIQUE`/`PRIMARY KEY` constraint on the parent table — you cannot have a multi-column foreign key referencing columns that aren't themselves jointly constrained as unique on the parent side.

## 6.16 Performance Implications

- Every additional constraint adds some write-time validation overhead — generally negligible for `NOT NULL`/simple `CHECK`, more meaningful for `FOREIGN KEY` (requires a lookup in the parent table) and `UNIQUE`/`PRIMARY KEY` (requires an index lookup/insert).
- Un-indexed foreign key columns on the child table (Section 6.3) can turn cascading deletes into expensive full-table scans on large tables — a genuine, measurable production performance issue.
- `CASCADE` operations on deeply nested relationship chains can trigger large, unexpectedly expensive cascading writes across many tables from a single `DELETE` statement.

## 6.17 Industry Usage

Constraint design is core, everyday schema design work — every production relational schema at every serious engineering organization relies on primary keys, foreign keys, and targeted `CHECK`/`UNIQUE`/`NOT NULL` constraints to guarantee the data integrity that application logic alone cannot reliably provide, especially in systems with multiple services/teams writing to shared tables.

## 6.18 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | Definitions of each constraint type; primary key vs foreign key |
| Placement | "What's the difference between `UNIQUE` and `PRIMARY KEY`?" |
| Mid-level | "Explain `ON DELETE CASCADE` vs `SET NULL` vs `RESTRICT`, with real examples." |
| Senior SDE-2/SDE-3 | "How does MySQL/InnoDB's clustered index affect primary key design decisions?" / "Walk through a schema where a careless `CASCADE` chain could cause unintended mass data loss, and how you'd redesign it." |

---

## 6.19 Chapter Summary

- Constraints enforce data integrity **at the database level**, immune to application-layer bugs or bypasses — a `PRIMARY KEY` combines `UNIQUE` + `NOT NULL` and is physically backed by an index.
- `FOREIGN KEY` enforces relationships between tables, and its `ON DELETE`/`ON UPDATE` referential actions (`CASCADE`, `RESTRICT`, `NO ACTION`, `SET NULL`, `SET DEFAULT`) must be chosen deliberately based on real relationship semantics.
- MySQL's InnoDB **clusters** table rows by primary key order (unlike PostgreSQL's heap storage), making primary key choice more performance-consequential in MySQL.
- `MyISAM` silently ignores foreign keys; pre-8.0.16 MySQL silently ignored `CHECK` constraints — both genuine, real-world gotchas worth verifying against your actual engine/version.
- `UNIQUE` allows multiple `NULL`s; `CHECK` enables arbitrary business-rule enforcement; `DEFAULT` and `NOT NULL` are independent, composable properties.

---

## 6.20 Revision Notes / Cheat Sheet

```sql
PRIMARY KEY               -- UNIQUE + NOT NULL, one per table, physically indexed
FOREIGN KEY ... REFERENCES parent(col)
    ON DELETE CASCADE | RESTRICT | NO ACTION | SET NULL | SET DEFAULT
    ON UPDATE CASCADE | RESTRICT | NO ACTION | SET NULL | SET DEFAULT
UNIQUE                     -- no duplicate values; multiple NULLs allowed
CHECK (expression)         -- arbitrary boolean business rule
NOT NULL                   -- forbids missing values
DEFAULT value               -- auto-fill when omitted on INSERT
GENERATED ALWAYS AS (...) STORED       -- computed column
GENERATED ALWAYS AS IDENTITY           -- auto-incrementing identity

⚠ MySQL: FK enforcement requires ENGINE=InnoDB (MyISAM silently ignores FKs)
⚠ MySQL: CHECK silently ignored before version 8.0.16
⚠ MySQL InnoDB: clustered index — PK order = physical row order
⚠ PostgreSQL: heap storage — PK order ≠ physical row order
```

---

## 6.21 Multiple Choice Questions (MCQs)

1. A `PRIMARY KEY` constraint is functionally equivalent to:
   a) `CHECK` alone b) **`UNIQUE` + `NOT NULL`** c) `DEFAULT` + `UNIQUE` d) `FOREIGN KEY`

2. Which referential action deletes child rows automatically when the parent is deleted?
   a) `RESTRICT` b) `SET NULL` c) **`CASCADE`** d) `NO ACTION`

3. Which MySQL storage engine silently does NOT enforce foreign keys?
   a) InnoDB b) **MyISAM** c) Both enforce them d) Neither enforces them

4. Which statement about `UNIQUE` constraints is TRUE?
   a) They forbid all `NULL` values b) **They typically allow multiple `NULL` values** c) Only one `UNIQUE` constraint is allowed per table d) They require a `FOREIGN KEY`

5. In MySQL/InnoDB, table rows are physically stored:
   a) In insertion order always b) Randomly, regardless of key c) **Clustered in primary key order** d) Alphabetically by table name

---

## 6.22 Exercises

### 🟢 Beginner
1. Create a `students` table with a primary key, a `NOT NULL` name, and a `UNIQUE` email.
2. Add a `CHECK` constraint ensuring a `products.price` column is never negative.
3. Add a `DEFAULT` value of `CURRENT_TIMESTAMP` to a `created_at` column.

### 🟡 Intermediate
4. Create two related tables (`authors` and `books`) with a foreign key from `books.author_id` to `authors.id`, using `ON DELETE CASCADE`. Insert sample data, delete an author, and confirm their books are also removed.
5. Repeat exercise 4, but using `ON DELETE SET NULL` instead, and observe the different outcome.
6. Attempt to insert a row into a foreign-key-constrained child table referencing a non-existent parent row, and record the exact error message your engine returns.

### 🔴 Advanced / Debugging
7. Explain why, in MySQL, a table accidentally created with `ENGINE=MyISAM` might appear to allow orphaned foreign key references even though the `FOREIGN KEY` syntax was written correctly.
8. A production incident occurs: deleting a single `customer` row unexpectedly deleted rows across five other tables. Using Section 6.4, explain how a chain of `ON DELETE CASCADE` constraints could cause this, and propose a safer alternative design.

### 🧩 Scenario-Based
9. Design the constraints for an `orders` table referencing a `customers` table, where you want to **retain** historical order records even after a customer account is deleted (for accounting/audit purposes), but still want to prevent orders from referencing nonexistent customers at insert time. Which `ON DELETE` action fits, and why?

### 🏗️ Mini Project
10. Take your mini-schema from Module 4/5's exercises and add a full, deliberate set of constraints to every table: primary keys, foreign keys with explicitly chosen referential actions, at least one `CHECK`, at least one `UNIQUE`, and appropriate `NOT NULL`/`DEFAULT` usage — documenting your `ON DELETE`/`ON UPDATE` choices with a one-sentence justification each.

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [PostgreSQL Docs — Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
- [MySQL Docs — Foreign Keys](https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html)

---

## ✅ Module 6 Complete

You can now design schemas that enforce their own correctness — primary and foreign keys, uniqueness, business-rule checks, and deliberately-chosen cascading behaviors — turning the ER diagrams from Part 2 into genuinely enforced, production-grade table definitions.

**Next up: Module 7 — DML (Data Manipulation)**, where you'll finally start inserting, updating, and deleting real data: `INSERT`, `UPSERT`, `MERGE`, `UPDATE`, `DELETE`, and the `RETURNING` clause.

---

*Reply "continue" or "next module" when you're ready for Module 7.*
