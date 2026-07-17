# The Complete DBMS Handbook — Part 4

## 📘 MODULE 9: STORED PROCEDURES & FUNCTIONS

> **Continuing from Module 8 (Advanced Database Objects).** So far, every construct in this Part has been **declarative** (a single `SELECT`-based expression). This module introduces **procedural** SQL — variables, control flow, loops, and error handling — the tools needed to encapsulate business logic (including Module 8's "manual materialized view refresh") directly inside the database.

---

## 9.0 Chapter Roadmap

```
Module 9: Stored Procedures & Functions
│
├── 9.1 Procedures vs Functions — The Core Distinction
├── 9.2 Stored Procedures
├── 9.3 User-Defined Functions (UDFs)
├── 9.4 Parameters (IN, OUT, INOUT)
├── 9.5 Return Types
├── 9.6 Variables & Declarations
├── 9.7 Control Flow (IF/CASE/LOOP/WHILE)
├── 9.8 Error Handling
├── 9.9 PostgreSQL vs MySQL Notes
├── 9.10 Performance Considerations
├── 9.11 Best Practices
├── 9.12 Common Mistakes & Edge Cases
├── 9.13 Interview Questions & MCQs
├── 9.14 Cheat Sheet
└── 9.15 Exercises
```

---

## 9.1 Procedures vs Functions — The Core Distinction

| Aspect | Stored Procedure | User-Defined Function (UDF) |
|---|---|---|
| Can be called from within a `SELECT` statement? | ❌ No (typically) | ✅ Yes — usable as an expression |
| Must return a value? | ❌ No (may return zero, one, or multiple result sets/OUT params) | ✅ Yes — must return exactly one value (scalar) or a table |
| Can modify data (`INSERT`/`UPDATE`/`DELETE`)? | ✅ Yes, freely | ⚠️ Restricted or disallowed in many engines (functions are expected to be side-effect-free, especially when used inside `SELECT`) |
| Can manage transactions (`COMMIT`/`ROLLBACK`)? | ✅ Yes, in most engines | ❌ Generally not allowed |
| Typical use case | Business-logic workflows, batch jobs, multi-step operations | Reusable calculations/transformations embedded directly in queries |
| Invocation | `CALL procedure_name(...)` | Used inline: `SELECT function_name(...)` |

### Real-life Analogy
A **Function** is like a **calculator**: you give it inputs, it computes and hands back exactly one answer, and it doesn't change anything else in the world around it. A **Procedure** is like a **checklist a bank teller follows** for "processing a loan application" — it might update several records, send a notification, log an audit entry, and not necessarily return any single "answer" at all — it performs a **workflow**, not just a computation.

---

## 9.2 Stored Procedures

**Definition:** A **Stored Procedure** is a named, precompiled block of procedural SQL code, stored in the database, that can perform **multiple operations** (queries, DML, control flow, even transaction management) and is invoked explicitly via `CALL`.

**PostgreSQL Example:**
```sql
CREATE PROCEDURE give_raise(emp_id_param INT, raise_amount NUMERIC)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE Employees
    SET salary = salary + raise_amount
    WHERE emp_id = emp_id_param;

    INSERT INTO SalaryChangeLog (emp_id, change_amount, changed_at)
    VALUES (emp_id_param, raise_amount, now());

    COMMIT;   -- procedures CAN manage transactions
END;
$$;

CALL give_raise(2, 5000);
```

**MySQL Example:**
```sql
DELIMITER //
CREATE PROCEDURE give_raise(IN emp_id_param INT, IN raise_amount DECIMAL(10,2))
BEGIN
    UPDATE Employees
    SET salary = salary + raise_amount
    WHERE emp_id = emp_id_param;

    INSERT INTO SalaryChangeLog (emp_id, change_amount, changed_at)
    VALUES (emp_id_param, raise_amount, NOW());
END //
DELIMITER ;

CALL give_raise(2, 5000);
```

### Why Stored Procedures Exist

| Problem Solved | Explanation |
|---|---|
| **Encapsulate multi-step business logic** | A "give a raise" operation involves an `UPDATE` AND an audit `INSERT` — bundling them into one procedure ensures they're always executed together, consistently, from any calling application. |
| **Reduce network round-trips** | Instead of the application sending 5 separate SQL statements over the network, it sends ONE `CALL`, and all 5 steps execute inside the database engine itself — significant latency savings for chatty multi-step operations. |
| **Centralized logic / consistency** | Multiple applications (a web app, a batch job, an admin tool) can all call the same procedure, guaranteeing consistent business-rule enforcement rather than each reimplementing the logic separately (and potentially inconsistently) in their own code. |
| **Security** | Grant `EXECUTE` permission on a procedure without granting direct table `UPDATE`/`INSERT` permissions — the procedure runs with its own defined privileges, tightly scoping what a user can actually change. |

#### Advantages
- Reduces network round-trips for multi-step operations.
- Centralizes and enforces business logic consistently.
- Can manage transactions explicitly (`COMMIT`/`ROLLBACK`) for atomic multi-step workflows.

#### Disadvantages
- Harder to version-control and test compared to application-layer code (though modern practices — migrations, procedure-as-code in repos — mitigate this).
- Logic split between application code and database code can reduce overall system readability/maintainability if overused.
- Vendor-specific procedural language syntax (PL/pgSQL vs MySQL's procedural SQL vs Oracle's PL/SQL vs T-SQL) reduces portability across database engines.

#### Interview Importance: ⭐⭐⭐ — Common in Senior Backend/Database Engineer interviews discussing "where should business logic live" tradeoffs.

---

## 9.3 User-Defined Functions (UDFs)

**Definition:** A **User-Defined Function** is a named, reusable block of code that **must return a value** (scalar or table-valued) and is designed to be used **as an expression**, typically inline within `SELECT`, `WHERE`, or computed columns.

### Scalar Function Example

```sql
-- PostgreSQL:
CREATE FUNCTION calculate_bonus(salary NUMERIC, performance_rating INT)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
BEGIN
    IF performance_rating >= 4 THEN
        RETURN salary * 0.15;
    ELSIF performance_rating = 3 THEN
        RETURN salary * 0.08;
    ELSE
        RETURN 0;
    END IF;
END;
$$;

-- Usage — directly inside a SELECT, exactly like a built-in function:
SELECT name, salary, calculate_bonus(salary, performance_rating) AS bonus
FROM Employees;
```

### Table-Valued Function (TVF) Example

**Definition:** Returns a full **result set (table)**, not a single scalar value — usable directly in a `FROM` clause, like a parameterized view.

```sql
-- PostgreSQL:
CREATE FUNCTION employees_in_dept(dept_id_param INT)
RETURNS TABLE (emp_id INT, name TEXT, salary NUMERIC)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT e.emp_id, e.name, e.salary
    FROM Employees e
    WHERE e.dept_id = dept_id_param;
END;
$$;

-- Usage — called like a parameterized table in FROM:
SELECT * FROM employees_in_dept(20);
```

> 🎯 **Table-Valued Functions are conceptually a "View with parameters"** — this is a very useful mental model for interviews: a regular View is fixed/unparameterized, while a TVF lets you pass arguments to customize the returned result set.

### Why UDFs Exist

| Problem Solved | Explanation |
|---|---|
| **Reusable computation logic embedded in queries** | Complex business calculations (bonus formulas, tax computations, string parsing) defined once and reused across many queries as a simple function call. |
| **Readability** | `calculate_bonus(salary, rating)` is far more readable inline than a repeated, multi-line `CASE WHEN` expression duplicated across dozens of queries. |
| **Parameterized, reusable "views"** | Table-Valued Functions let you build reusable, parameterized query logic that a plain View cannot express (Views take no arguments). |

#### Interview Importance: ⭐⭐⭐⭐ — TVFs vs Views is a common "when would you use which" design question; scalar UDF performance pitfalls (Section 9.10) are a classic senior-level gotcha.

---

## 9.4 Parameters (IN, OUT, INOUT)

| Parameter Mode | Meaning |
|---|---|
| `IN` (default) | Value passed **into** the procedure/function; the procedure cannot modify the caller's original variable |
| `OUT` | Value **returned** from the procedure back to the caller via the parameter itself; typically starts as `NULL` and is set inside the procedure |
| `INOUT` | Value passed in **and** can be modified, with the modified value visible to the caller after the call returns |

> **Only Stored Procedures support `OUT`/`INOUT` parameters** in most engines — Functions communicate results purely through their single `RETURN` value (or `RETURNS TABLE` for TVFs), not via output parameters, reinforcing Section 9.1's distinction.

```sql
-- PostgreSQL procedure with OUT parameter:
CREATE PROCEDURE get_employee_count(dept_id_param INT, OUT emp_count INT)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT COUNT(*) INTO emp_count FROM Employees WHERE dept_id = dept_id_param;
END;
$$;

CALL get_employee_count(20, NULL);   -- result is returned in the output
```

---

## 9.5 Return Types

| Return Type | Applies To | Example |
|---|---|---|
| Scalar (single value: INT, NUMERIC, TEXT, BOOLEAN, etc.) | Functions | `RETURNS NUMERIC` |
| Table (result set) | Table-Valued Functions | `RETURNS TABLE (col1 TYPE, col2 TYPE, ...)` |
| `VOID` / no return | Procedures (typically) | Procedures don't declare a return type in most engines; they communicate via OUT params or side effects |
| Multiple result sets | Procedures (engine-dependent) | MySQL procedures can `SELECT` multiple times, each producing a separate result set returned to the caller |

---

## 9.6 Variables & Declarations

**Definition:** Procedural blocks can declare **local variables** to hold intermediate values during execution — declared in a `DECLARE` section (PostgreSQL/Oracle style) or inline (`DECLARE` statements in MySQL).

```sql
-- PostgreSQL PL/pgSQL:
CREATE FUNCTION total_dept_salary(dept_id_param INT)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    total_salary NUMERIC;      -- variable declaration
    emp_count INT;
BEGIN
    SELECT SUM(salary), COUNT(*) INTO total_salary, emp_count
    FROM Employees WHERE dept_id = dept_id_param;

    IF emp_count = 0 THEN
        RETURN 0;
    END IF;

    RETURN total_salary;
END;
$$;
```

### Assignment Syntax
```sql
-- PostgreSQL:
total_salary := total_salary + 1000;
-- or
SELECT SUM(salary) INTO total_salary FROM Employees;

-- MySQL:
SET total_salary = total_salary + 1000;
SELECT SUM(salary) INTO total_salary FROM Employees;
```

---

## 9.7 Control Flow (IF / CASE / LOOP / WHILE)

### IF / ELSIF / ELSE

```sql
IF performance_rating >= 4 THEN
    bonus := salary * 0.15;
ELSIF performance_rating = 3 THEN
    bonus := salary * 0.08;
ELSE
    bonus := 0;
END IF;
```

### CASE (Procedural Form)

```sql
CASE performance_rating
    WHEN 5 THEN bonus := salary * 0.20;
    WHEN 4 THEN bonus := salary * 0.15;
    ELSE bonus := 0;
END CASE;
```

### LOOP / WHILE (Iteration)

```sql
-- PostgreSQL — WHILE loop
CREATE PROCEDURE apply_annual_raises()
LANGUAGE plpgsql
AS $$
DECLARE
    emp_cursor CURSOR FOR SELECT emp_id, salary FROM Employees;
    emp_row RECORD;
BEGIN
    OPEN emp_cursor;
    LOOP
        FETCH emp_cursor INTO emp_row;
        EXIT WHEN NOT FOUND;

        UPDATE Employees
        SET salary = emp_row.salary * 1.05
        WHERE emp_id = emp_row.emp_id;
    END LOOP;
    CLOSE emp_cursor;
END;
$$;
```

> ⚠️ **Row-by-row cursor loops (as shown above) are almost always an anti-pattern** compared to a single **set-based** SQL statement:
> ```sql
> -- Set-based equivalent — dramatically faster, no cursor/loop needed at all:
> UPDATE Employees SET salary = salary * 1.05;
> ```
> This connects directly to a core SQL philosophy point: **SQL is a set-based (declarative) language — the database engine is highly optimized for operating on entire sets of rows at once, NOT for row-by-row procedural iteration.** Cursors/loops should be reserved for genuinely row-dependent logic that cannot be expressed as a single set-based statement (e.g., calling an external API per row, complex conditional branching that varies wildly per row in ways a `CASE` expression can't cleanly capture). **This exact "avoid cursors, prefer set-based SQL" principle is one of the most important senior-level SQL best practices, and is very frequently tested in interviews** ("Why is this stored procedure slow? — it's using a cursor where a single UPDATE would work").

---

## 9.8 Error Handling

### PostgreSQL — EXCEPTION Blocks

```sql
CREATE PROCEDURE transfer_funds(from_acct INT, to_acct INT, amount NUMERIC)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE Accounts SET balance = balance - amount WHERE acct_id = from_acct;
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Source account % not found', from_acct;
    END IF;

    UPDATE Accounts SET balance = balance + amount WHERE acct_id = to_acct;

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE NOTICE 'Transfer failed: %', SQLERRM;
        RAISE;   -- re-raise to propagate the error to the caller
END;
$$;
```

### MySQL — DECLARE HANDLER

```sql
DELIMITER //
CREATE PROCEDURE transfer_funds(IN from_acct INT, IN to_acct INT, IN amount DECIMAL(10,2))
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;   -- re-raise the original error to the caller
    END;

    START TRANSACTION;
    UPDATE Accounts SET balance = balance - amount WHERE acct_id = from_acct;
    UPDATE Accounts SET balance = balance + amount WHERE acct_id = to_acct;
    COMMIT;
END //
DELIMITER ;
```

### Why Proper Error Handling Matters (Preview: Transactions, Part 6/7)
The `transfer_funds` example above is a classic **atomicity** requirement (a preview of ACID transactions, covered in full in Parts 6–7): if the second `UPDATE` fails after the first one succeeds, without proper error handling and `ROLLBACK`, money would **vanish** from the source account without ever reaching the destination — a critical, unacceptable bug in financial systems. Proper error handling inside procedures is what makes multi-step atomic workflows safe.

---

## 9.9 PostgreSQL vs MySQL Notes

| Feature | PostgreSQL | MySQL |
|---|---|---|
| Procedural language | PL/pgSQL (also supports PL/Python, PL/Perl, etc. as extensions) | Proprietary MySQL procedural SQL extension (not a separate named language) |
| Separate `PROCEDURE` vs `FUNCTION` object types | ✅ Yes, distinct (`CREATE PROCEDURE` since PG 11+; `CREATE FUNCTION` always existed) | ✅ Yes, distinct (`CREATE PROCEDURE` vs `CREATE FUNCTION`) |
| Table-Valued Functions | ✅ `RETURNS TABLE (...)` or `RETURNS SETOF` | ❌ Not directly supported — MySQL functions can only return scalar values; achieving similar results typically requires a procedure returning a result set instead |
| Transaction control inside routines | Procedures: ✅ full `COMMIT`/`ROLLBACK`; Functions: ❌ generally not allowed | Procedures: ✅ `START TRANSACTION`/`COMMIT`/`ROLLBACK`; Functions: ❌ not allowed |
| Error handling mechanism | `EXCEPTION ... WHEN ... THEN` blocks | `DECLARE ... HANDLER FOR ...` blocks |
| Cursors | ✅ Supported | ✅ Supported |

---

## 9.10 Performance Considerations

- **Reduced network round-trips**: bundling multiple statements into one procedure call is often a genuine performance win for chatty client-server interactions, especially over high-latency network links.
- **⚠️ Scalar UDFs called inside a `SELECT` over many rows can be a serious, often invisible performance trap**: many database engines execute a scalar UDF **once per row**, effectively disabling certain optimizer techniques (like index-based predicate evaluation) if the UDF wraps a filtered/compared column — this can silently turn an otherwise fast, indexable query into a slow, row-by-row computation.
```sql
-- ⚠️ Potential performance trap: UDF wraps the filter column, defeating index usage
SELECT * FROM Employees WHERE calculate_bonus(salary, performance_rating) > 5000;

-- Better: keep filters "sargable" (Module 1, Section 1.5.1) — avoid wrapping indexed
-- columns in function calls where possible, or pre-compute where feasible
```
- **Avoid row-by-row cursor loops** wherever a single set-based SQL statement can achieve the same result (Section 9.7) — this is consistently one of the highest-leverage stored-procedure performance fixes in real production systems.
- **Compiled/cached execution plans**: Stored procedures are typically **parsed and plan-cached** once, then reused across calls, which can reduce repeated parsing/planning overhead compared to ad-hoc queries sent fresh every time (though modern database engines also cache plans for parameterized ad-hoc queries, narrowing this historical advantage).

---

## 9.11 Best Practices

- Reserve **Stored Procedures** for genuine multi-step workflows with side effects (DML, transaction control); reserve **Functions** for pure(-ish), reusable computations usable inline in queries.
- Avoid cursors/row-by-row loops; always ask "can this be expressed as a single set-based statement?" first.
- Always include proper **error handling with rollback** for any multi-step procedure that modifies data across more than one table/statement.
- Be cautious using scalar UDFs directly on indexed, frequently-filtered columns — verify with `EXPLAIN` that index usage isn't silently defeated.
- Keep business logic **either** primarily in the application layer **or** primarily in the database layer for a given system — avoid splitting the *same* piece of logic across both, which leads to duplicated, potentially inconsistent implementations.

---

## 9.12 Common Mistakes & Edge Cases

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using a cursor/loop for something expressible as a single `UPDATE`/`INSERT ... SELECT` | Row-by-row processing is dramatically slower than set-based operations | Rewrite as a single set-based statement |
| Wrapping an indexed column in a scalar UDF inside `WHERE` | Can defeat index usage entirely (non-sargable predicate) | Avoid function-wrapped filters on indexed columns; verify with `EXPLAIN` |
| Forgetting error handling in a multi-statement procedure with side effects | Partial failures leave data in an inconsistent state | Always wrap multi-step DML in proper transaction + exception handling |
| Assuming Functions can freely perform DML like Procedures | Many engines restrict or disallow DML inside functions, especially ones used in `SELECT` | Use Procedures for DML-heavy workflows |
| Confusing `OUT` parameters (Procedures) with `RETURN` values (Functions) | Different mechanisms, not interchangeable | Use `RETURN`/`RETURNS TABLE` for Functions; `OUT`/`INOUT` for Procedures |

---

## 9.13 Interview Questions & MCQs

### Conceptual
1. What is the fundamental difference between a Stored Procedure and a User-Defined Function?
2. Why can't Functions typically be used to manage transactions (`COMMIT`/`ROLLBACK`), while Procedures can?
3. What is a Table-Valued Function, and how does it compare conceptually to a View?
4. Why are row-by-row cursor loops generally discouraged in favor of set-based SQL?
5. Explain, with an example, how a scalar UDF used in a `WHERE` clause can silently defeat index usage.

### Applied
6. Write a Stored Procedure that atomically transfers funds between two accounts, including proper error handling and rollback on failure.
7. Write a scalar UDF to compute an employee's bonus based on salary and performance rating, then use it inline in a `SELECT`.
8. Rewrite a cursor-based "give everyone a 5% raise" procedure as a single set-based `UPDATE` statement, and explain the performance implications.
9. Design a Table-Valued Function that returns all employees in a given department, and explain when you'd prefer this over a plain parameter-less View.

### MCQs

**Q1.** Which of these can be used directly inside a `SELECT` statement as an expression?
 a) Stored Procedure  b) Function  c) Both equally  d) Neither

**Q2.** Which parameter mode allows a value to be both passed in AND modified for the caller to see afterward?
 a) IN  b) OUT  c) INOUT  d) RETURN

**Q3.** The primary performance issue with row-by-row cursor loops is:
 a) They use more disk space  b) They bypass set-based optimization, processing one row at a time  c) They cannot use variables  d) They are not supported in most engines

**Q4.** In PostgreSQL, which object type can manage transactions (`COMMIT`/`ROLLBACK`) directly?
 a) Function only  b) Procedure only  c) Both  d) Neither

**Q5.** A scalar UDF wrapping an indexed column in a `WHERE` clause typically:
 a) Improves performance automatically  b) Has no effect on index usage  c) Can defeat/disable index usage (non-sargable) d) Is not allowed by SQL syntax

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) Function**
2. **c) INOUT**
3. **b) They bypass set-based optimization, processing one row at a time**
4. **b) Procedure only**
5. **c) Can defeat/disable index usage (non-sargable)**

</details>

---

## 9.14 Cheat Sheet — Module 9

```
┌───────────────────────────────────────────────────────────────────┐
│              STORED PROCEDURES & FUNCTIONS — QUICK REFERENCE           │
├───────────────────────────┬───────────────────────────────────────┤
│ Procedure                    │ Multi-step workflow, DML+transactions, │
│                                │ called via CALL, no required return   │
│ Function                       │ Reusable computation, MUST return a    │
│                                  │ value, usable inline in SELECT/WHERE  │
│ Table-Valued Function              │ Function returning a TABLE (like a   │
│                                      │ "parameterized View")                │
│ IN / OUT / INOUT                      │ Procedure parameter directions        │
│ DECLARE                                 │ Local variable declaration            │
│ IF/ELSIF/ELSE, CASE                       │ Procedural conditional control flow   │
│ LOOP/WHILE + CURSOR                         │ Row-by-row iteration — AVOID when a  │
│                                               │ set-based statement can do the job   │
│ EXCEPTION block (PG) /                         │ Error handling + ROLLBACK for atomic │
│ DECLARE HANDLER (MySQL)                          │ multi-step operations                │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 9.15 Exercises

### Beginner
1. Write a scalar Function that returns `'Senior'` if years of experience > 5, else `'Junior'`.
2. Write a Procedure that inserts a new employee row, given name, dept, and salary as parameters.

### Intermediate
3. Write a Procedure with an `OUT` parameter that returns the total headcount of a given department.
4. Write a Table-Valued Function returning all orders for a given customer ID.

### Advanced
5. Write a Procedure implementing an atomic funds transfer between two accounts, including proper error handling and rollback.
6. Rewrite a cursor-based "apply a 10% discount to all products in a category" procedure into a single set-based `UPDATE`, and explain the performance improvement.

### Output Prediction
7. Given a scalar UDF used inside `WHERE udf(indexed_col) > 100`, predict (in general terms) what will show up in the `EXPLAIN` output compared to `WHERE indexed_col > some_precomputed_threshold`.

### Debugging
8. A stored procedure meant to update salaries and log the change sometimes leaves an orphaned log entry with no corresponding salary change, after certain failures. Diagnose (hint: missing transaction/error handling) and fix.

### Scenario-Based
9. In an interview, you're asked to justify whether a piece of complex, multi-step business logic (e.g., "process a return and refund") should live in a Stored Procedure or in application code. Present the tradeoffs on both sides.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)

---

## Chapter Summary

- **Procedures** perform multi-step workflows (DML, transaction control), invoked via `CALL`, and are not usable as inline expressions.
- **Functions** must return a value (scalar or table) and are designed to be used inline within `SELECT`/`WHERE` — most engines restrict DML/transaction control inside them.
- **Table-Valued Functions** are conceptually "Views with parameters."
- **Parameters** come in `IN` (default), `OUT`, and `INOUT` modes — only Procedures typically support `OUT`/`INOUT`.
- **Avoid row-by-row cursor loops** — SQL is fundamentally set-based, and a single set-based statement is almost always dramatically faster than an equivalent cursor loop.
- **Proper error handling with rollback** is essential for any multi-step procedure with side effects, to guarantee atomicity (previewing ACID transactions in Parts 6–7).
- **Scalar UDFs on indexed, filtered columns can silently defeat index usage** — always verify with `EXPLAIN`.

---

## Cross-References for Later Parts / Modules
- **Triggers, which automatically invoke procedural logic in response to DML events** → Module 10 (next)
- **ACID transactions, COMMIT/ROLLBACK semantics in depth** → Parts 6–7
- **Query optimizer's handling of function calls in predicates (sargability)** → Part 8 (Query Execution Internals)

---

> ✅ **Module 9 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 10"** to proceed to **Module 10: Triggers** — BEFORE/AFTER/INSTEAD OF triggers, row-level vs statement-level, audit triggers, business rule triggers, recursive triggers, and best practices.
