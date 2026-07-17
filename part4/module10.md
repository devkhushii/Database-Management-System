# The Complete DBMS Handbook — Part 4

## 📘 MODULE 10: TRIGGERS

> **Continuing from Module 9 (Stored Procedures & Functions).** Where a Procedure must be **explicitly called**, a Trigger is procedural logic that the database **calls automatically** in response to a data-modification event — the final piece connecting declarative SQL, procedural logic, and event-driven automation.

---

## 10.0 Chapter Roadmap

```
Module 10: Triggers
│
├── 10.1 What is a Trigger?
├── 10.2 Why Triggers Exist
├── 10.3 BEFORE Triggers
├── 10.4 AFTER Triggers
├── 10.5 INSTEAD OF Triggers
├── 10.6 Row-Level vs Statement-Level Triggers
├── 10.7 Audit Triggers (Pattern)
├── 10.8 Business Rule Triggers (Pattern)
├── 10.9 Recursive Triggers & Mutating Table Problem
├── 10.10 PostgreSQL vs MySQL Notes
├── 10.11 Best Practices
├── 10.12 Common Mistakes & Edge Cases
├── 10.13 Performance Notes
├── 10.14 Interview Questions & MCQs
├── 10.15 Cheat Sheet
└── 10.16 Exercises
```

---

## 10.1 What is a Trigger?

**Definition:** A **Trigger** is a named block of procedural code, stored in the database, that is **automatically executed ("fired") in response to a specified DML event** (`INSERT`, `UPDATE`, or `DELETE`) on a specified table or view — the person/application performing the DML **never explicitly calls** the trigger; the database invokes it automatically.

**Basic Syntax Pattern (PostgreSQL requires a separate trigger FUNCTION, then a TRIGGER binding it to a table/event):**
```sql
-- Step 1: Define the trigger's logic as a function
CREATE FUNCTION log_salary_change() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO SalaryChangeLog (emp_id, old_salary, new_salary, changed_at)
    VALUES (OLD.emp_id, OLD.salary, NEW.salary, now());
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Step 2: Bind the function to a table + event
CREATE TRIGGER trg_log_salary_change
AFTER UPDATE OF salary ON Employees
FOR EACH ROW
WHEN (OLD.salary IS DISTINCT FROM NEW.salary)
EXECUTE FUNCTION log_salary_change();
```

**MySQL Syntax (self-contained, no separate function needed):**
```sql
DELIMITER //
CREATE TRIGGER trg_log_salary_change
AFTER UPDATE ON Employees
FOR EACH ROW
BEGIN
    IF OLD.salary <> NEW.salary THEN
        INSERT INTO SalaryChangeLog (emp_id, old_salary, new_salary, changed_at)
        VALUES (OLD.emp_id, OLD.salary, NEW.salary, NOW());
    END IF;
END //
DELIMITER ;
```

### OLD and NEW — The Two Special Row References

| Reference | Available in | Meaning |
|---|---|---|
| `OLD` | `UPDATE`, `DELETE` triggers | The row's values **before** the change |
| `NEW` | `INSERT`, `UPDATE` triggers | The row's values **after** the change |

```
INSERT trigger:  only NEW exists       (there was no "before" row)
UPDATE trigger:  both OLD and NEW exist (compare before vs after)
DELETE trigger:  only OLD exists        (there is no "after" row)
```

### Real-life Analogy
A trigger is like a **smoke alarm**: nobody explicitly "calls" the smoke alarm when there's a fire — it's wired to automatically activate the moment smoke (the triggering event) is detected. Similarly, a database trigger automatically "goes off" the moment a matching `INSERT`/`UPDATE`/`DELETE` event occurs, with zero explicit invocation needed by the application.

---

## 10.2 Why Triggers Exist

| Problem Solved | Explanation |
|---|---|
| **Automatic, unbypassable enforcement** | Unlike application-layer validation (which can be bypassed by a different application, a direct SQL client, or a bug), a trigger fires **no matter how** the data change happens — guaranteeing consistent enforcement at the database layer itself. |
| **Auditing / change history** | Automatically log every change to sensitive data (salary changes, permission changes) without relying on every application to remember to log it manually. |
| **Complex business rule enforcement beyond simple constraints** | Constraints (`CHECK`, `FOREIGN KEY`) can only express relatively simple, single-row or single-table rules — triggers can implement arbitrary procedural logic, including cross-table validation. |
| **Derived/denormalized data maintenance** | Automatically keep a denormalized summary column (e.g., `order_count` on a `Customers` table) in sync whenever related rows in another table (`Orders`) change. |
| **Making non-updatable Views updatable (`INSTEAD OF`)** | Recall Module 8, Section 8.1 — complex Views (joins, aggregates) aren't directly updatable; an `INSTEAD OF` trigger lets you define exactly how an `INSERT`/`UPDATE`/`DELETE` against the view should translate into changes on the underlying base tables. |

---

## 10.3 BEFORE Triggers

**Definition:** Fires **before** the triggering DML operation is actually applied to the table — commonly used for **validation** and **data modification/normalization** of the incoming row, since a `BEFORE` trigger can still **change `NEW`'s values** before they're written.

```sql
-- PostgreSQL:
CREATE FUNCTION validate_and_normalize_employee() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.salary < 0 THEN
        RAISE EXCEPTION 'Salary cannot be negative';
    END IF;

    NEW.email := LOWER(TRIM(NEW.email));   -- normalize before insert
    RETURN NEW;   -- MUST return NEW (or the modified NEW) for the operation to proceed
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_validate_employee
BEFORE INSERT OR UPDATE ON Employees
FOR EACH ROW
EXECUTE FUNCTION validate_and_normalize_employee();
```

> 🎯 **Key Fact:** A `BEFORE` trigger can **modify `NEW`** and can **abort the operation entirely** (via `RAISE EXCEPTION`, or by returning `NULL` in some engines' row-level trigger semantics) — this is precisely why `BEFORE` is the natural place for **validation** and **normalization** logic: the "bad" or "unnormalized" data is intercepted **before** it's ever actually written to the table.

---

## 10.4 AFTER Triggers

**Definition:** Fires **after** the triggering DML operation has already been applied — the change is already committed to the table (within the current transaction) by the time the trigger runs. Commonly used for **auditing**, **cascading side effects**, and **derived data maintenance**, since the "final" state of the row is already settled.

```sql
CREATE FUNCTION update_customer_order_count() RETURNS TRIGGER AS $$
BEGIN
    UPDATE Customers
    SET order_count = order_count + 1
    WHERE customer_id = NEW.customer_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_order_count
AFTER INSERT ON Orders
FOR EACH ROW
EXECUTE FUNCTION update_customer_order_count();
```

> **Key Fact:** An `AFTER` trigger **cannot modify `NEW`'s values to change what gets stored** (the row is already written) — its return value is generally ignored for this purpose. It CAN still abort the overall transaction by raising an exception, which would roll back the original DML too (since it's all within the same transaction).

### BEFORE vs AFTER — Decision Table

| Need | Use |
|---|---|
| Validate/reject the incoming data | `BEFORE` |
| Normalize/transform the incoming data before it's stored | `BEFORE` |
| Log/audit what changed, after it's confirmed to have happened | `AFTER` |
| Cascade the change to other tables (update a related summary/count) | `AFTER` |
| Enforce a cross-table business rule that depends on the row already being committed | `AFTER` |

---

## 10.5 INSTEAD OF Triggers

**Definition:** Fires **in place of** the triggering DML operation — the original `INSERT`/`UPDATE`/`DELETE` **never actually executes** against the object the trigger is defined on; instead, **only the trigger's own logic runs**. Primarily used to make **non-updatable Views updatable** (recall Module 8, Section 8.1).

```sql
CREATE VIEW EmployeeDeptView AS
SELECT e.emp_id, e.name, e.salary, d.dept_name
FROM Employees e
JOIN Departments d ON e.dept_id = d.dept_id;
-- This view is a JOIN — NOT directly updatable (Module 8, Section 8.1)

CREATE FUNCTION handle_employee_dept_update() RETURNS TRIGGER AS $$
BEGIN
    -- Manually define what an UPDATE through the view should actually do:
    UPDATE Employees SET name = NEW.name, salary = NEW.salary WHERE emp_id = NEW.emp_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_instead_of_update
INSTEAD OF UPDATE ON EmployeeDeptView
FOR EACH ROW
EXECUTE FUNCTION handle_employee_dept_update();

-- Now this works, even though EmployeeDeptView is a join-based view:
UPDATE EmployeeDeptView SET salary = 75000 WHERE emp_id = 2;
-- The actual UPDATE against Employees is performed by the trigger, not directly.
```

> 🎯 **`INSTEAD OF` is only usable on Views (not base tables) in most engines** — it exists specifically to solve the "how do I make a complex view updatable" problem, giving you full manual control over how each DML operation against the view should be translated into changes on the real underlying base tables.

#### Interview Importance: ⭐⭐⭐⭐ — Directly connects Module 8's view-updatability discussion with a concrete solution; a strong signal of depth when brought up unprompted in an interview about complex reporting views.

---

## 10.6 Row-Level vs Statement-Level Triggers

| Aspect | Row-Level (`FOR EACH ROW`) | Statement-Level (default if `FOR EACH ROW` omitted, in engines that support both) |
|---|---|---|
| Fires how many times? | **Once per affected row** | **Once per statement**, regardless of how many rows it affects |
| Access to `OLD`/`NEW`? | ✅ Yes, per-row values | ❌ No individual row values (in standard SQL statement-level triggers) — though some engines (SQL Server) provide pseudo-tables (`inserted`/`deleted`) containing ALL affected rows for statement-level access |
| Typical use case | Per-row validation, per-row audit logging with row-specific old/new values | Bulk summary logging ("an UPDATE affecting N rows occurred at time T"), enforcing statement-level invariants |
| Performance on bulk operations | Can be a bottleneck — fires N times for an N-row `UPDATE` | Fires exactly once, regardless of N — much cheaper for bulk operations |

```
UPDATE Employees SET salary = salary * 1.05 WHERE dept_id = 20;  -- affects 50 rows

ROW-LEVEL trigger:        fires 50 TIMES (once per row)
STATEMENT-LEVEL trigger:  fires 1 TIME (once for the whole statement)
```

> ⚠️ **Performance-Critical Interview Point:** A row-level `AFTER` trigger that itself performs an `UPDATE`/`INSERT` on **every fire** can turn a single bulk `UPDATE` affecting thousands of rows into thousands of **additional** individual trigger-driven statements — a very common, very real production performance pitfall. Always consider whether the trigger's logic can be expressed as a **statement-level** trigger (or restructured as a single set-based follow-up operation) when bulk DML is expected.

---

## 10.7 Audit Triggers (Pattern)

**Definition:** A trigger whose purpose is purely to **record a history of changes** to a table, typically inserting a row into a separate "audit log" or "history" table capturing what changed, when, and by whom.

```sql
CREATE TABLE EmployeeAuditLog (
    audit_id SERIAL PRIMARY KEY,
    emp_id INT,
    operation VARCHAR(10),   -- 'INSERT', 'UPDATE', 'DELETE'
    old_data JSONB,
    new_data JSONB,
    changed_by TEXT,
    changed_at TIMESTAMP DEFAULT now()
);

CREATE FUNCTION audit_employee_changes() RETURNS TRIGGER AS $$
BEGIN
    IF (TG_OP = 'DELETE') THEN
        INSERT INTO EmployeeAuditLog (emp_id, operation, old_data, changed_by)
        VALUES (OLD.emp_id, 'DELETE', row_to_json(OLD), current_user);
        RETURN OLD;
    ELSIF (TG_OP = 'UPDATE') THEN
        INSERT INTO EmployeeAuditLog (emp_id, operation, old_data, new_data, changed_by)
        VALUES (NEW.emp_id, 'UPDATE', row_to_json(OLD), row_to_json(NEW), current_user);
        RETURN NEW;
    ELSIF (TG_OP = 'INSERT') THEN
        INSERT INTO EmployeeAuditLog (emp_id, operation, new_data, changed_by)
        VALUES (NEW.emp_id, 'INSERT', row_to_json(NEW), current_user);
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_employees
AFTER INSERT OR UPDATE OR DELETE ON Employees
FOR EACH ROW
EXECUTE FUNCTION audit_employee_changes();
```

**Industry Usage:** Extremely common in **compliance-heavy domains** — banking (regulatory audit trails), healthcare (HIPAA-related change tracking), and any system requiring a legally defensible "who changed what, when" history.

---

## 10.8 Business Rule Triggers (Pattern)

**Definition:** A trigger enforcing a business rule that is **too complex for a simple `CHECK` constraint** — typically because it requires checking data across **multiple rows or multiple tables**.

```sql
-- Business rule: "An employee's salary cannot exceed their manager's salary"
CREATE FUNCTION enforce_salary_hierarchy() RETURNS TRIGGER AS $$
DECLARE
    manager_salary NUMERIC;
BEGIN
    SELECT salary INTO manager_salary FROM Employees WHERE emp_id = NEW.manager_id;

    IF manager_salary IS NOT NULL AND NEW.salary > manager_salary THEN
        RAISE EXCEPTION 'Employee salary (%) cannot exceed manager salary (%)', NEW.salary, manager_salary;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_enforce_salary_hierarchy
BEFORE INSERT OR UPDATE OF salary, manager_id ON Employees
FOR EACH ROW
EXECUTE FUNCTION enforce_salary_hierarchy();
```

> **Why not a `CHECK` constraint here?** A `CHECK` constraint can only reference **columns within the same row** being inserted/updated (in most engines) — it **cannot query another row** (like the manager's salary in a different row of the same table). This exact limitation is precisely why business rules requiring **cross-row or cross-table validation** need a trigger instead of a simple constraint.

---

## 10.9 Recursive Triggers & Mutating Table Problem

### Recursive Triggers

**Definition:** A trigger whose own action **causes the same trigger (or another trigger that eventually leads back to it) to fire again** — either **directly** (a trigger on `Employees` that itself updates `Employees`) or **indirectly** (Trigger A on Table X updates Table Y, which has Trigger B that updates Table X again).

```
Direct recursion:
  UPDATE Employees → fires Trigger T → T itself does UPDATE Employees → fires Trigger T again → ...

Indirect recursion:
  UPDATE TableX → fires Trigger A → A does UPDATE TableY
       → fires Trigger B (on TableY) → B does UPDATE TableX
            → fires Trigger A again → ... (potential infinite loop)
```

> ⚠️ **Danger:** Without care, this can cause **infinite recursion**, exhausting stack/resource limits and crashing the operation. Most engines have a **default recursion depth limit** (analogous to Module 4's recursive CTE safety limits) that will eventually raise an error — but relying on that as your only safeguard is poor practice.

**Best Practice — Guard Conditions:**
```sql
CREATE FUNCTION prevent_infinite_update() RETURNS TRIGGER AS $$
BEGIN
    -- Only proceed if something ACTUALLY changed, breaking potential infinite loops
    IF NEW.salary IS DISTINCT FROM OLD.salary THEN
        -- ... trigger logic that might cause further updates ...
        NULL;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### The Mutating Table Problem (Classic Oracle Terminology, Concept Applies Broadly)

**Definition:** A row-level trigger attempting to **query or modify the very same table that is currently being modified** (the table that fired the trigger) can lead to **inconsistent reads** — because, mid-statement, some rows have already been changed and others haven't yet, so a query against that table from within its own trigger sees a **partially-updated, inconsistent snapshot**.

```sql
-- ❌ PROBLEMATIC PATTERN (classic mutating-table issue):
CREATE FUNCTION enforce_dept_budget() RETURNS TRIGGER AS $$
DECLARE
    dept_total NUMERIC;
BEGIN
    -- Querying Employees FROM a trigger fired BY a change to Employees itself —
    -- mid-statement, this SELECT may see a partially-updated table
    SELECT SUM(salary) INTO dept_total FROM Employees WHERE dept_id = NEW.dept_id;

    IF dept_total > 1000000 THEN
        RAISE EXCEPTION 'Department budget exceeded';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Standard Solutions:**
- Use a **statement-level `AFTER` trigger** instead of row-level, so the query runs **once, after the entire statement has fully completed** — by then, the table is in a fully consistent state.
- Use **`FOR EACH STATEMENT`** with a **transition table** (PostgreSQL/Oracle support `REFERENCING NEW TABLE AS ...` to see all rows affected by the statement as a consistent set) rather than querying the live base table row-by-row.

> 🎯 **The "mutating table problem" is a genuinely advanced, senior-level ("SDE-3"/Database Engineer) interview topic** — being able to explain *why* querying a table from its own row-level trigger is dangerous, and *how* to fix it (statement-level trigger + transition table), is a strong signal of deep practical trigger experience.

---

## 10.10 PostgreSQL vs MySQL Notes

| Feature | PostgreSQL | MySQL |
|---|---|---|
| Trigger definition style | Two-step: separate trigger **function**, then `CREATE TRIGGER` binding it | Single-step: `CREATE TRIGGER` with inline body |
| `INSTEAD OF` triggers | ✅ Supported (on Views) | ❌ **Not supported** — MySQL has no `INSTEAD OF` trigger equivalent |
| Statement-level triggers | ✅ Supported (`FOR EACH STATEMENT`) | ❌ **Not supported** — MySQL only supports row-level (`FOR EACH ROW`) triggers |
| Transition tables (`REFERENCING NEW TABLE AS ...`) | ✅ Supported (PG 10+) — solves the mutating table problem cleanly | ❌ Not supported |
| Multiple triggers on the same event | ✅ Supported, with explicit ordering (`BEFORE`/`AFTER` combined with creation order or explicit `FOLLOWS`/`PRECEDES`, PG uses alphabetical trigger name order by default unless specified) | ✅ Supported, with `FOLLOWS`/`PRECEDES` clause (MySQL 5.7.2+) for explicit ordering |
| Conditional trigger firing (`WHEN` clause) | ✅ Supported (`CREATE TRIGGER ... WHEN (condition) ...`) | ❌ Not supported directly — condition must be checked with an `IF` inside the trigger body instead |

> 🎯 **MySQL lacking both `INSTEAD OF` and statement-level triggers is a very practically significant, frequently-tested cross-platform difference** — if asked "how would you make a complex view updatable in MySQL," the honest answer is: **MySQL has no `INSTEAD OF` trigger mechanism at all**; the workaround typically involves restructuring the application logic to update base tables directly, or using stored procedures that encapsulate the "virtual update" logic instead of relying on trigger-based view updatability.

---

## 10.11 Best Practices

- Keep trigger logic **fast and simple** — triggers fire synchronously, inside the same transaction as the triggering statement, so slow trigger logic directly slows down every `INSERT`/`UPDATE`/`DELETE` on that table.
- Prefer **statement-level triggers** over row-level triggers for bulk-operation-heavy tables, when the logic doesn't genuinely need per-row `OLD`/`NEW` values (MySQL users: this isn't an option — be extra cautious with row-level trigger cost on bulk operations).
- Always guard against **recursive trigger loops** with explicit "did anything actually change" conditions.
- Document triggers clearly and centrally — because they fire **invisibly** (not called explicitly in application code), they are a common source of "mystery" behavior/bugs for engineers unfamiliar with a schema; maintain a clear inventory of what triggers exist and why.
- Avoid using triggers for logic that's more naturally and transparently expressed as **application-layer code** or a **stored procedure explicitly called** — reserve triggers for genuinely cross-cutting, unbypassable concerns (auditing, referential integrity beyond FKs, cross-table business rules).

---

## 10.12 Common Mistakes & Edge Cases

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Forgetting `RETURN NEW;` in a PostgreSQL `BEFORE` row-level trigger | The row insert/update can be silently suppressed (returning `NULL` cancels the operation in PostgreSQL row-level triggers) | Always explicitly `RETURN NEW;` (or `RETURN OLD;` for DELETE) unless intentionally cancelling |
| Row-level trigger performing expensive work on every fire during bulk operations | Can turn one bulk `UPDATE` into thousands of expensive individual trigger executions | Use statement-level triggers (PostgreSQL) or restructure logic to avoid per-row expense |
| Querying the same table from its own row-level trigger | Mutating table problem — inconsistent, partial reads | Use statement-level trigger + transition table, or restructure the check |
| Unbounded recursive trigger chains | Risk of infinite loops / stack exhaustion | Add explicit "did anything change" guard conditions |
| Relying on triggers as the ONLY layer of business rule enforcement, invisible to the dev team | Triggers are easy to forget exist, causing confusing "hidden" behavior | Document triggers thoroughly; consider whether application-layer enforcement is more maintainable for non-critical rules |
| Assuming MySQL supports `INSTEAD OF` triggers | It does not | Use application-layer or stored-procedure-based workarounds for updatable-view-like behavior in MySQL |

---

## 10.13 Performance Notes

- Triggers execute **synchronously**, within the same transaction as the triggering DML — their cost is **directly added** to the cost of every `INSERT`/`UPDATE`/`DELETE` they fire on.
- **Row-level triggers on bulk DML are the most common trigger-related performance bottleneck** in production systems — always consider the expected DML volume (single-row app inserts vs nightly bulk-loading batch jobs) when designing trigger logic.
- Triggers that themselves perform additional `INSERT`/`UPDATE`/`DELETE` (e.g., audit logging, cascading updates) effectively **multiply the total amount of work** done per original statement — profile and monitor accordingly, especially under high write throughput.
- Consider whether **denormalized/derived data** genuinely needs to be updated **synchronously via a trigger**, or whether an **asynchronous** approach (a scheduled job, an event/message-queue-driven update) would be more scalable for very high-write-volume systems — this is a common senior-level system-design tradeoff discussion.

---

## 10.14 Interview Questions & MCQs

### Conceptual
1. What is a trigger, and how does it differ from a stored procedure in terms of invocation?
2. Explain the difference between `BEFORE`, `AFTER`, and `INSTEAD OF` triggers, with a use case for each.
3. What is the difference between row-level and statement-level triggers, and which engines support statement-level triggers?
4. What is the "mutating table problem," and how would you resolve it?
5. Why can't a `CHECK` constraint replace a trigger for a rule like "employee salary cannot exceed manager's salary"?

### Applied
6. Design an audit-trigger system for a `Salaries` table that records every change with old value, new value, and timestamp.
7. Write an `INSTEAD OF` trigger making a join-based view updatable, and explain exactly what the trigger must do to correctly propagate the change to base tables.
8. Explain how a row-level `AFTER` trigger performing an expensive operation could severely degrade the performance of a bulk `UPDATE` affecting 100,000 rows, and propose a mitigation.
9. Design a guard condition to prevent infinite recursion in a trigger that updates the same table it's defined on.

### MCQs

**Q1.** Which trigger type can modify the `NEW` row's values before they're written to the table?
 a) AFTER  b) BEFORE  c) INSTEAD OF (on a base table)  d) None

**Q2.** `INSTEAD OF` triggers are primarily used to:
 a) Improve query performance  b) Make complex, non-updatable Views updatable  c) Replace all AFTER triggers  d) Enforce foreign keys

**Q3.** Which engine does NOT support statement-level triggers?
 a) PostgreSQL  b) Oracle  c) MySQL  d) SQL Server

**Q4.** The "mutating table problem" occurs when:
 a) A table has too many columns  b) A row-level trigger queries the same table currently being modified, mid-statement  c) Two triggers have the same name  d) A trigger fires on the wrong event

**Q5.** In `Employees`'s `AFTER UPDATE` trigger, which reference holds the row's values before the change?
 a) NEW  b) OLD  c) CURRENT  d) PREVIOUS

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) BEFORE**
2. **b) Make complex, non-updatable Views updatable**
3. **c) MySQL**
4. **b) A row-level trigger queries the same table currently being modified, mid-statement**
5. **b) OLD**

</details>

---

## 10.15 Cheat Sheet — Module 10

```
┌───────────────────────────────────────────────────────────────────┐
│                       TRIGGERS — QUICK REFERENCE                       │
├───────────────────────────┬───────────────────────────────────────┤
│ BEFORE trigger                │ Fires before write; CAN modify NEW,    │
│                                 │ CAN abort — used for validation/normalize│
│ AFTER trigger                    │ Fires after write; row already stored;  │
│                                    │ used for auditing, cascading updates    │
│ INSTEAD OF trigger                  │ Replaces the operation entirely;        │
│                                       │ used to make Views updatable (PG only) │
│ OLD                                    │ Row values BEFORE change (UPDATE/DELETE)│
│ NEW                                     │ Row values AFTER change (INSERT/UPDATE) │
│ Row-level (FOR EACH ROW)                  │ Fires once PER affected row            │
│ Statement-level (FOR EACH STATEMENT)         │ Fires ONCE per statement (PG only,    │
│                                                │ not in MySQL)                          │
│ Mutating table problem                          │ Row-level trigger reading its own      │
│                                                    │ table mid-statement = inconsistent     │
│ Recursive trigger risk                              │ Guard with "did anything change" checks│
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 10.16 Exercises

### Beginner
1. Write a `BEFORE INSERT` trigger that rejects any employee row with a negative salary.
2. Write an `AFTER INSERT` trigger that logs every new employee's ID and name into an `EmployeeCreationLog` table.

### Intermediate
3. Write a row-level `AFTER UPDATE` trigger that logs old and new salary values whenever salary changes (and only when it actually changes).
4. Write an `INSTEAD OF UPDATE` trigger for a join-based view, explaining exactly what base-table changes it performs.

### Advanced
5. Design a trigger enforcing "an employee's salary cannot exceed 90% of their department's total budget," discussing why this requires a trigger rather than a `CHECK` constraint, and how you'd avoid the mutating table problem.
6. Design a guard condition preventing infinite recursion for a trigger on `Employees` that itself performs an `UPDATE` on `Employees` under certain conditions.

### Output Prediction
7. Given a row-level `AFTER` trigger on `Orders` that inserts into `OrderAuditLog`, predict how many times the trigger fires (and how many audit rows are created) for a single `UPDATE` statement affecting 200 rows.

### Debugging
8. A trigger meant to update a denormalized `order_count` column on `Customers` is causing a "stack depth limit exceeded" error. Diagnose the likely cause (recursive trigger) and propose a fix.

### Scenario-Based
9. In a system-design interview, you're asked whether a "keep a denormalized summary column in sync" requirement should be implemented via a trigger or an asynchronous background job. Present both approaches' tradeoffs.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)

---

## Chapter Summary

- A **Trigger** fires **automatically** in response to `INSERT`/`UPDATE`/`DELETE`, unlike a Procedure, which must be explicitly called.
- **`BEFORE`** triggers can modify incoming data and abort the operation — ideal for validation/normalization; **`AFTER`** triggers see the already-committed change — ideal for auditing and cascading side effects; **`INSTEAD OF`** triggers replace the operation entirely — the standard way to make complex Views updatable.
- **Row-level** triggers fire once per affected row; **statement-level** triggers (PostgreSQL/Oracle/SQL Server, **not MySQL**) fire once per statement — a critical performance consideration for bulk DML.
- **Audit triggers** and **business rule triggers** are the two most common real-world trigger patterns; the latter is needed precisely because `CHECK` constraints cannot perform cross-row/cross-table validation.
- **Recursive triggers** and the **mutating table problem** are advanced, senior-level pitfalls — always guard against unintended recursion, and avoid querying a table from its own row-level trigger mid-statement.
- **MySQL lacks `INSTEAD OF` triggers and statement-level triggers entirely** — a significant, frequently-tested cross-platform gap.

---

## Cross-References for Later Parts / Modules
- **ACID transactions and how trigger side effects participate in the same transaction as the triggering statement** → Parts 6–7
- **Foreign Keys and CHECK constraints as the "simple rule" alternative to triggers** → Part 3 (recap) / Part 5 (Normalization & Constraints)
- **Query execution internals — how the optimizer/executor invokes trigger logic within a statement's execution plan** → Part 8

---

> ✅ **Module 10 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 11"** to proceed to **Module 11: SQL Execution Internals (Introduction)** — the SQL Parser, Query Rewrite, Optimizer, Execution Plan, Execution Engine, and Result Generation, at a high level (with the full deep dive reserved for Part 8).
