# The Complete DBMS Handbook — Part 4

## 📘 MODULE 7: SET OPERATIONS

> **Continuing from Module 6 (Advanced Aggregation).** Set operations were first introduced formally in Module 1 (Section 1.6, ∪/∩/−) as Relational Algebra primitives. This module gives their **SQL surface syntax** (`UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT`/`MINUS`) full practical treatment — syntax rules, duplicate handling, `NULL` behavior, and performance.

---

## 7.0 Chapter Roadmap

```
Module 7: Set Operations
│
├── 7.1 Recap & Syntax Rules
├── 7.2 UNION
├── 7.3 UNION ALL
├── 7.4 INTERSECT
├── 7.5 EXCEPT / MINUS
├── 7.6 Duplicate Elimination — Deep Dive
├── 7.7 NULL Handling in Set Operations
├── 7.8 Combining Multiple Set Operations & Precedence
├── 7.9 Set Operations vs JOIN — When to Use Which
├── 7.10 PostgreSQL vs MySQL Notes
├── 7.11 Best Practices
├── 7.12 Common Mistakes & Edge Cases
├── 7.13 Performance Notes
├── 7.14 Interview Questions & MCQs
├── 7.15 Cheat Sheet
└── 7.16 Exercises
```

---

## 7.1 Recap & Syntax Rules

Recall from Module 1 (Section 1.4, 1.6) that RA's ∪, ∩, − require **union-compatibility**: the two input relations must have the **same number of columns**, with **compatible (comparable) data types** in corresponding positions. SQL's set operators inherit this rule exactly:

```sql
SELECT col1, col2 FROM TableA
{ UNION | UNION ALL | INTERSECT | EXCEPT }
SELECT col1, col2 FROM TableB;
```

**Mandatory Rules:**
1. Both `SELECT` lists must have the **same number of columns**.
2. Corresponding columns must have **compatible data types** (implicit conversion rules apply, e.g., `INT` and `DECIMAL` are usually compatible; `INT` and `VARCHAR` typically are not, without explicit casting).
3. **Column names in the final result come from the FIRST `SELECT`** — column names/aliases in the second (and later) `SELECT`s are ignored for output naming purposes.
4. Only **one `ORDER BY`** is allowed, and it must appear at the **very end** of the entire combined statement (applying to the final combined result, not to an individual branch).

---

## 7.2 UNION

**Definition:** Combines the result sets of two (or more) `SELECT` statements, **removing duplicate rows** — direct SQL syntax for RA's ∪ (Module 1, Section 1.6.1).

**Sample Tables:**

`CurrentEmployees(emp_id, name)`

| emp_id | name |
|---|---|
| 1 | Aditi |
| 2 | Rohan |

`FormerEmployees(emp_id, name)`

| emp_id | name |
|---|---|
| 2 | Rohan |
| 3 | Suresh |

```sql
SELECT emp_id, name FROM CurrentEmployees
UNION
SELECT emp_id, name FROM FormerEmployees;
```

**Result:**

| emp_id | name |
|---|---|
| 1 | Aditi |
| 2 | Rohan |
| 3 | Suresh |

(Rohan, appearing in both source tables, appears **only once**.)

### Internal Working
To eliminate duplicates, the engine typically performs either a **sort** (sort all combined rows, then scan and drop adjacent duplicates) or a **hash-based deduplication** (build a hash set of seen rows, skip already-seen ones) — both require examining **every column of every row** to detect duplicates, which is the source of `UNION`'s extra cost compared to `UNION ALL` (Section 7.6, 7.13).

---

## 7.3 UNION ALL

**Definition:** Combines the result sets of two (or more) `SELECT` statements, **keeping all rows, including duplicates** — this is the **bag-union** operation (recall Module 1's Bag vs Set distinction, Section 1.3).

```sql
SELECT emp_id, name FROM CurrentEmployees
UNION ALL
SELECT emp_id, name FROM FormerEmployees;
```

**Result:**

| emp_id | name |
|---|---|
| 1 | Aditi |
| 2 | Rohan |
| 2 | Rohan | ← duplicate KEPT
| 3 | Suresh |

> 🎯 **`UNION ALL` has NO deduplication step at all** — it is a pure, cheap concatenation of the two result sets, row by row, with no sorting or hashing required. This makes it **significantly faster** than `UNION` whenever you know duplicates either can't occur (e.g., the two source tables have genuinely disjoint keys) or don't matter for your use case.

### UNION vs UNION ALL — Side-by-Side

| Aspect | UNION | UNION ALL |
|---|---|---|
| Duplicates | Removed | Kept |
| Performance | Slower (sort/hash pass required) | Faster (simple concatenation) |
| Matches pure RA ∪ semantics? | ✅ Yes | ❌ No (RA's ∪ always removes duplicates; this is SQL's practical bag-union extension) |
| Default choice when unsure? | Use only if duplicates are genuinely unwanted | **Prefer this by default** if you know duplicates can't/don't matter |

---

## 7.4 INTERSECT

**Definition:** Returns only the rows that appear in **both** result sets, with duplicates removed — direct SQL syntax for RA's ∩ (Module 1, Section 1.6.2).

```sql
SELECT emp_id, name FROM CurrentEmployees
INTERSECT
SELECT emp_id, name FROM FormerEmployees;
```

**Result:**

| emp_id | name |
|---|---|
| 2 | Rohan |

### INTERSECT ALL (Standard SQL, engine-support varies)
Some engines (PostgreSQL) also support `INTERSECT ALL`, which keeps duplicates according to the **minimum count** present in either side (bag-intersection semantics) — e.g., if a row appears 3 times in the first query and 2 times in the second, `INTERSECT ALL` keeps it **2 times** in the result. This is a subtle, rarely-used but occasionally-tested feature.

**Real-world use:** "Customers who bought BOTH Product A and Product B," "Users active on BOTH the mobile app and web app this month," "Employees on BOTH the training-completed list and the performance-review-due list."

---

## 7.5 EXCEPT / MINUS

**Definition:** Returns rows present in the **first** result set but **NOT** in the second, with duplicates removed — direct SQL syntax for RA's − (Module 1, Section 1.6.3). **Not commutative.**

```sql
-- PostgreSQL, SQL Server, MySQL 8.0.31+ syntax:
SELECT emp_id, name FROM CurrentEmployees
EXCEPT
SELECT emp_id, name FROM FormerEmployees;

-- Oracle syntax (uses MINUS instead of EXCEPT):
SELECT emp_id, name FROM CurrentEmployees
MINUS
SELECT emp_id, name FROM FormerEmployees;
```

**Result:**

| emp_id | name |
|---|---|
| 1 | Aditi |

(Rohan is excluded, since he appears in `FormerEmployees`.)

**Reversed direction:**
```sql
SELECT emp_id, name FROM FormerEmployees
EXCEPT
SELECT emp_id, name FROM CurrentEmployees;
```
**Result:** `{(3, Suresh)}` — a **completely different** result, proving non-commutativity, exactly as in Module 1.

### EXCEPT vs MINUS — Naming Only, Same Semantics

| Engine | Keyword |
|---|---|
| PostgreSQL | `EXCEPT` |
| SQL Server | `EXCEPT` |
| MySQL (8.0.31+) | `EXCEPT` |
| Oracle | `MINUS` |

> This is purely a **vocabulary difference between vendors** — the underlying semantics (set difference, not commutative, deduplicates) are **identical** everywhere. A frequently asked "gotcha" trivia question in Oracle-focused interviews.

**Real-world use:** "Customers who signed up last month but have NOT placed any order," "Employees assigned to Project A but NOT to Project B," "Products in the catalog that have NEVER been sold" (compare against `SELECT DISTINCT product_id FROM OrderItems`).

> **Connection back to Module 3:** `EXCEPT` is a set-level, whole-row alternative to the row-by-row `NOT EXISTS`/`NOT IN` anti-join patterns from Module 3 — for simple "rows in A not matched in B on ALL columns" cases, `EXCEPT` is often more concise; for partial-column matching or more complex correlated logic, `NOT EXISTS` remains more flexible.

---

## 7.6 Duplicate Elimination — Deep Dive

### Why UNION/INTERSECT/EXCEPT Always Deduplicate (But UNION ALL Doesn't)

This directly traces back to Module 1's foundational fact: **pure Relational Algebra treats relations as sets**, and ∪, ∩, − are defined in classical set theory, which has **no concept of "the same element twice."** SQL's `UNION`, `INTERSECT`, and `EXCEPT` were designed to match this mathematical definition precisely. `UNION ALL` (and the rarely-used `INTERSECT ALL`/`EXCEPT ALL`) are SQL's **pragmatic extensions** for when bag semantics (matching SQL's default table behavior) are actually what's wanted.

### How Deduplication Is Physically Implemented

```
Two common physical strategies (engine/optimizer chooses based on cost):

1. SORT-BASED:
   Combine both result sets → SORT all rows (on all columns) →
   scan sorted output, skip immediate consecutive duplicates.
   Cost: O(n log n) for the sort.

2. HASH-BASED:
   Build an in-memory (or spill-to-disk) hash set of "already seen" full rows →
   for each incoming row, check hash set; if new, add to output AND to hash set;
   if already present, skip it.
   Cost: O(n) average, but requires memory proportional to distinct row count.
```

> The choice between these two strategies is a cost-based optimizer decision, directly analogous to how `GROUP BY`/`DISTINCT` deduplication is implemented (Module 1, Section 1.5.2) — the exact same underlying mechanism powers `π` with duplicate elimination, `SELECT DISTINCT`, and `UNION`/`INTERSECT`/`EXCEPT`.

---

## 7.7 NULL Handling in Set Operations

**Critical Rule:** For deduplication purposes in `UNION`/`INTERSECT`/`EXCEPT`, **two `NULL` values in the same column position ARE treated as equal to each other** (i.e., as duplicates) — this is a **special-cased exception** to SQL's usual three-valued logic, where `NULL = NULL` normally evaluates to `UNKNOWN`, not `TRUE`.

```sql
SELECT NULL AS x
UNION
SELECT NULL AS x;
-- Result: ONE row containing NULL (the two NULLs were treated as duplicates and merged)
```

> 🎯 **This exact NULL-equality special case (also true for `DISTINCT` and `GROUP BY`) is a frequently tested, genuinely counter-intuitive fact** — students who've correctly learned that "`NULL = NULL` is `UNKNOWN`, not `TRUE`, in `WHERE` clauses" (Module 1, Section 1.5.1) are often surprised that `DISTINCT`/`UNION`/`GROUP BY` treat NULLs as equal to each other for grouping/deduplication purposes specifically. **The rule that "NULL is never equal to anything" applies to comparison operators in `WHERE`/`ON`/`HAVING`, not to the internal grouping/deduplication logic of `DISTINCT`, `GROUP BY`, `UNION`, `INTERSECT`, or `EXCEPT`.**

---

## 7.8 Combining Multiple Set Operations & Precedence

When multiple set operators appear in one statement, **`INTERSECT` typically binds tighter (higher precedence) than `UNION` and `EXCEPT`** in most engines (analogous to how `∧` binds tighter than `∨` in boolean logic, or `*` binds tighter than `+` in arithmetic) — but this **varies by engine**, so **explicit parentheses are strongly recommended** whenever mixing operators.

```sql
-- AMBIGUOUS without parentheses — behavior may differ by engine:
SELECT x FROM A
UNION
SELECT x FROM B
INTERSECT
SELECT x FROM C;

-- EXPLICIT, unambiguous, portable:
SELECT x FROM A
UNION
(SELECT x FROM B INTERSECT SELECT x FROM C);
```

#### Best Practice
**Always use explicit parentheses** when combining more than one type of set operator in a single statement — never rely on implicit precedence rules, since they are one of the least-standardized, most engine-varying aspects of SQL.

---

## 7.9 Set Operations vs JOIN — When to Use Which

| Need | Use |
|---|---|
| Combine rows **vertically** (stack result sets with the same columns, from possibly different tables) | Set operations (`UNION`/`UNION ALL`) |
| Combine columns **horizontally** (bring together related columns from two tables based on a key) | `JOIN` |
| Find rows matching on ALL columns between two same-shaped queries | `INTERSECT`/`EXCEPT` |
| Find rows matching on a SPECIFIC key/column relationship | `JOIN` (or `EXISTS`/`IN`, Module 3) |

```
JOIN combines COLUMNS (horizontal):          UNION combines ROWS (vertical):

  A(id,name)  ⋈  B(id,dept)                    A(id,name)
  ────────────────────────                       ────────
  id | name | dept                                1  | Aditi
  1  | Aditi| Sales                                2  | Rohan
                                                  UNION
                                                  B(id,name)
                                                    ────────
                                                    3  | Suresh
                                                  ══════════
                                                    1 | Aditi
                                                    2 | Rohan
                                                    3 | Suresh
```

> A very common **interview mistake** is reaching for a `JOIN` when the actual need is to **stack two similarly-shaped result sets from different sources** (e.g., "combine this month's and last month's sales into one report") — that's a `UNION ALL` use case, not a `JOIN`.

---

## 7.10 PostgreSQL vs MySQL Notes

| Feature | PostgreSQL | MySQL |
|---|---|---|
| `UNION` / `UNION ALL` | ✅ Full support | ✅ Full support |
| `INTERSECT` | ✅ Full support | ✅ Supported from MySQL 8.0.31+ (not in earlier 8.x or 5.x versions) |
| `EXCEPT` | ✅ Full support | ✅ Supported from MySQL 8.0.31+ (older versions need a `LEFT JOIN ... WHERE ... IS NULL` workaround, see below) |
| `INTERSECT ALL` / `EXCEPT ALL` | ✅ Supported | ❌ Not supported |
| Set operator precedence | `INTERSECT` binds tighter than `UNION`/`EXCEPT` (left-to-right among equal precedence) | Similar, but always verify with parentheses |

**Pre-8.0.31 MySQL `EXCEPT` Workaround:**
```sql
SELECT a.emp_id, a.name
FROM CurrentEmployees a
LEFT JOIN FormerEmployees b ON a.emp_id = b.emp_id AND a.name = b.name
WHERE b.emp_id IS NULL;
```
This is functionally an **anti-join** (recall Module 1, Section 1.8.6) — reinforcing that `EXCEPT` and anti-joins are two syntaxes for closely related ideas.

---

## 7.11 Best Practices

- **Default to `UNION ALL` unless you specifically need deduplication** — it's cheaper, and explicit about intent ("I know/expect these to be disjoint, or duplicates are fine").
- **Always match column counts and compatible types explicitly** — don't rely on implicit type coercion across branches; cast explicitly if types differ (e.g., `CAST(col AS VARCHAR)`).
- **Use explicit parentheses** whenever combining multiple different set operators in one statement.
- **Prefer `EXCEPT`/`INTERSECT` for whole-row set comparisons**; prefer `NOT EXISTS`/`EXISTS` (Module 3) when you only need to check a **subset** of columns or need correlated, more complex logic.

---

## 7.12 Common Mistakes & Edge Cases

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Using `UNION` by default without considering `UNION ALL` | Unnecessary performance cost from deduplication when duplicates were never a concern | Default to `UNION ALL` unless dedup is genuinely required |
| Mismatched column counts/types across `UNION` branches | SQL error, or silent, incorrect implicit type coercion | Explicitly match column count and cast types as needed |
| Assuming `NULL = NULL` is `UNKNOWN` inside `UNION`/`DISTINCT` deduplication | It's actually treated as equal for grouping/deduplication purposes (Section 7.7) | Remember this special case explicitly |
| Assuming `EXCEPT`/`MINUS` is commutative | `A EXCEPT B ≠ B EXCEPT A` | Always double-check direction, exactly like RA's `−` (Module 1) |
| Mixing `UNION`/`INTERSECT`/`EXCEPT` without parentheses | Ambiguous/engine-dependent precedence | Always parenthesize explicitly |
| Forgetting `ORDER BY` only applies once, at the very end | Trying to `ORDER BY` an individual branch mid-statement causes a syntax error in most engines | Place a single `ORDER BY` after the entire combined statement |

---

## 7.13 Performance Notes

- **`UNION ALL` is a pure concatenation** — O(n) with no sort/hash step — always the cheaper option when duplicates are acceptable.
- **`UNION`/`INTERSECT`/`EXCEPT` require a full deduplication pass** (sort or hash, Section 7.6) over the **combined result set**, which is O(n log n) or O(n) average respectively, plus memory/disk overhead for large result sets.
- When each branch of a `UNION ALL` can be executed via an **index-friendly, highly selective filter**, the overall set operation is often cheap; the cost usually dominates when branches individually scan large portions of their source tables.
- For "does this row exist in table B" type checks on a **single, specific relationship**, `EXISTS`/`JOIN` (Module 3) is often more optimizer-friendly (can leverage indexes on the specific join key) than a whole-row `INTERSECT`/`EXCEPT`, which must compare **all selected columns**.

---

## 7.14 Interview Questions & MCQs

### Conceptual
1. What is the exact difference between `UNION` and `UNION ALL`, both semantically and performance-wise?
2. Why does SQL treat two `NULL`s as equal for `UNION`/`DISTINCT`/`GROUP BY` purposes, even though `NULL = NULL` is `UNKNOWN` in `WHERE`?
3. Is `EXCEPT` commutative? Justify with an example.
4. What is the relationship between `EXCEPT` and Oracle's `MINUS`?
5. Why should you always use explicit parentheses when combining `UNION`, `INTERSECT`, and `EXCEPT` in one statement?

### Applied
6. Write a query using `UNION ALL` to combine `Orders2025` and `Orders2026` into one report, and explain why `UNION` would be unnecessarily expensive here (assuming order IDs are unique across years by design).
7. Write a query using `INTERSECT` to find customers who purchased in BOTH January and February (given per-month order tables or a single table filtered by date).
8. Write a query using `EXCEPT` to find products that have never been ordered, and then rewrite it using `NOT EXISTS` (Module 3) — discuss the tradeoffs.
9. Write the MySQL pre-8.0.31 `LEFT JOIN ... WHERE ... IS NULL` workaround for an `EXCEPT` query, and explain why it works.

### MCQs

**Q1.** Which operator does NOT remove duplicates?
 a) UNION  b) UNION ALL  c) INTERSECT  d) EXCEPT

**Q2.** `A EXCEPT B` returns:
 a) Rows in both A and B  b) Rows in A but not B  c) Rows in B but not A  d) All rows from A and B combined

**Q3.** For deduplication in `UNION`, two NULL values in the same column position are treated as:
 a) Not equal (UNKNOWN)  b) Equal (duplicates)  c) An error  d) Automatically converted to 0

**Q4.** Oracle uses which keyword instead of `EXCEPT`?
 a) DIFFERENCE  b) NOT  c) MINUS  d) SUBTRACT

**Q5.** Which is generally the cheapest set operation to execute?
 a) UNION  b) UNION ALL  c) INTERSECT  d) EXCEPT

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) UNION ALL**
2. **b) Rows in A but not B**
3. **b) Equal (duplicates)**
4. **c) MINUS**
5. **b) UNION ALL**

</details>

---

## 7.15 Cheat Sheet — Module 7

```
┌───────────────────────────────────────────────────────────────────┐
│                   SET OPERATIONS — QUICK REFERENCE                    │
├───────────────────────────┬───────────────────────────────────────┤
│ UNION                       │ A ∪ B, deduplicated                    │
│ UNION ALL                    │ A ∪ B, duplicates KEPT (cheapest)      │
│ INTERSECT                     │ A ∩ B, deduplicated                   │
│ EXCEPT (Oracle: MINUS)          │ A − B, deduplicated, NOT commutative │
│ Column count/type rule           │ Must match across all branches       │
│ Output column names               │ Taken from the FIRST SELECT          │
│ NULL in dedup context               │ Treated as EQUAL (special case!)    │
│ ORDER BY placement                    │ Once, at the very end only         │
│ Mixed operator precedence               │ Engine-dependent — always use ()  │
│ MySQL EXCEPT/INTERSECT support           │ 8.0.31+ only                      │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 7.16 Exercises

### Beginner
1. Write a `UNION` query combining two employee lists, and a `UNION ALL` version of the same query — explain the output difference if a name appears in both.
2. Write an `INTERSECT` query to find common values between two single-column tables.

### Intermediate
3. Write an `EXCEPT` query to find customers in `Customers2025` who are NOT in `Customers2026`, then explain what changes if you reverse the operand order.
4. Demonstrate, with a concrete example, that two `NULL`s are treated as equal in a `UNION` but not in a `WHERE col1 = col2` comparison.

### Advanced
5. Write a query combining `UNION` and `INTERSECT` with explicit parentheses to compute `(A ∪ B) ∩ C`.
6. Write the MySQL pre-8.0.31 workaround for `A EXCEPT B`, and explain how it functions as an anti-join (link back to Module 1, Section 1.8.6).

### Output Prediction
7. Given `A = {(1,'x'), (2,'y'), (NULL,'z')}` and a `UNION` of `A` with itself, what is the exact deduplicated output?

### Debugging
8. A junior engineer writes a query using `UNION` where `UNION ALL` was actually intended (and duplicates are legitimate business data, not errors) — describe the bug's symptom and the fix.

### Scenario-Based
9. In an interview, you're asked: "Find employees who are in Team A's list this month AND were also in Team A's list last month" (i.e., stayed on the team). Which set operation fits, and why?

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

- **`UNION`/`INTERSECT`/`EXCEPT`** are the SQL surface syntax for RA's ∪/∩/− (Module 1) — all require matching column count/types and **deduplicate** their output by default.
- **`UNION ALL`** is the cheap, non-deduplicating bag-union — **prefer it by default** unless deduplication is genuinely required.
- **`EXCEPT`/`MINUS`** (same operator, different vendor keyword) is **not commutative** — direction matters.
- **`NULL` is treated as equal to `NULL`** specifically for deduplication purposes in `UNION`/`DISTINCT`/`GROUP BY` — a notable exception to normal three-valued `WHERE`-clause logic.
- **Always use explicit parentheses** when mixing multiple set operators, since precedence rules vary by engine.
- Set operations combine rows **vertically**; `JOIN` combines columns **horizontally** — don't confuse the two use cases.

---

## Cross-References for Later Parts / Modules
- **Views / Materialized Views built on top of UNION-based reports** → Module 8 (next)
- **NOT EXISTS / Anti-Join as the row-level, correlated alternative to EXCEPT** → Module 3 (recap)
- **Query optimizer's sort/hash strategy selection for set operations** → Part 8 (Query Execution Internals)

---

> ✅ **Module 7 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 8"** to proceed to **Module 8: Advanced Database Objects** — Views, Materialized Views, Sequences, Synonyms, Temporary Tables, Global Temporary Tables, and common usage patterns.
