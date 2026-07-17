# The Complete DBMS Handbook — Part 4

## 📘 MODULE 4: COMMON TABLE EXPRESSIONS (CTEs)

> **Continuing from Module 3 (Advanced SQL Queries).** Recall from Module 1 (Section 1.10) that the RA **Assignment Operator (←)** is the theoretical ancestor of the CTE — a CTE is simply SQL's readable syntax for "name an intermediate result, then use it." Recall also Module 3's teaser of `WITH RECURSIVE` — this module gives it full treatment.

---

## 4.0 Chapter Roadmap

```
Module 4: Common Table Expressions
│
├── 4.1 What is a CTE? The WITH Clause
├── 4.2 Why CTEs Exist
├── 4.3 Multiple CTEs (Chaining & Referencing)
├── 4.4 Recursive CTEs — Core Mechanics
├── 4.5 Recursive Trees — Hierarchical Queries
├── 4.6 Recursive Graph Traversal
├── 4.7 Cycle Detection & Termination Safety
├── 4.8 CTE vs Derived Table vs View vs Temp Table
├── 4.9 PostgreSQL vs MySQL Notes
├── 4.10 Best Practices
├── 4.11 Performance Considerations
├── 4.12 Common Mistakes & Edge Cases
├── 4.13 Interview Questions & MCQs
├── 4.14 Cheat Sheet
└── 4.15 Exercises
```

---

## 4.1 What is a CTE? The WITH Clause

**Definition:** A **Common Table Expression (CTE)** is a **named, temporary result set** defined using a `WITH` clause, which exists **only for the duration of the single statement** that follows it. Once defined, you can reference the CTE **by name** as if it were a real table, anywhere in the main query.

**Basic Syntax:**
```sql
WITH cte_name (optional_col1, optional_col2, ...) AS (
    SELECT ...    -- the CTE's defining query
)
SELECT ...
FROM cte_name
...;
```

**Simple Example:**
```sql
WITH HighEarners AS (
    SELECT emp_id, name, salary
    FROM Employees
    WHERE salary > 60000
)
SELECT name, salary
FROM HighEarners
ORDER BY salary DESC;
```

### Real-life Analogy
A CTE is like giving a **nickname to a complicated calculation** in a math proof: instead of repeating "the sum of the first n terms of this series" every time, you write "let S = the sum of the first n terms" once, then just use `S` everywhere after. It doesn't change what's being computed — it changes how **readable** the computation is.

### Internal Working
Conceptually, a non-recursive CTE is treated by the optimizer very similarly to a **derived table** (Module 3, Section 3.5): the database may **inline/merge** the CTE's definition directly into the main query plan (as if you'd pasted the subquery inline), or it may **materialize** it (compute it once into a temporary result and reuse that). Which strategy is chosen is a **cost-based optimizer decision** and — critically — **differs by database engine and version** (see Section 4.9 and 4.11).

```
Execution Tree (conceptual):

        Main Query
             │
      references "HighEarners"
             │
     ┌───────┴────────┐
     │  CTE definition │   ← may be inlined (merged into main plan)
     │  σ_(salary>60000)│     or materialized (computed once, reused)
     │  (Employees)     │
     └─────────────────┘
```

---

## 4.2 Why CTEs Exist

| Problem Solved | Explanation |
|---|---|
| **Readability** | Deeply nested subqueries become unreadable ("subquery soup"). CTEs let you name and sequence intermediate steps top-to-bottom, like a recipe. |
| **Reusability within a query** | The same intermediate result can be referenced **multiple times** in the main query without repeating the subquery text (though re-computation behavior depends on materialization — see 4.11). |
| **Recursion** | CTEs are the **only standard SQL mechanism** for recursive queries — hierarchical/graph traversal is otherwise impossible in plain declarative SQL (Module 3, Section 3.7). |
| **Modularity** | Complex ETL-style, multi-stage transformations can be broken into clearly-named logical stages, closely mirroring how a programmer would write a multi-step function. |
| **Formal basis** | Directly corresponds to the RA Assignment operator (`X ← expr`, Module 1 Section 1.10) — giving CTEs a rigorous theoretical foundation, not just a syntactic convenience. |

---

## 4.3 Multiple CTEs (Chaining & Referencing)

You can define **multiple CTEs** in a single `WITH` clause, separated by commas. **Later CTEs can reference earlier ones** (but not vice versa — no forward references, and no CTE can reference itself unless it's recursive).

```sql
WITH DeptAvg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM Employees
    GROUP BY dept_id
),
AboveAvgEmployees AS (
    SELECT e.name, e.salary, e.dept_id, d.avg_salary
    FROM Employees e
    JOIN DeptAvg d ON e.dept_id = d.dept_id
    WHERE e.salary > d.avg_salary
)
SELECT name, salary, dept_id
FROM AboveAvgEmployees
ORDER BY salary DESC;
```

**Dependency Chain Diagram:**
```
DeptAvg  ──feeds into──▶  AboveAvgEmployees  ──feeds into──▶  Main SELECT
```

### Why Chaining Matters
This mirrors exactly how a senior engineer thinks about a complex query: **break the problem into stages**, name each stage's output, and build the final answer from clearly-defined intermediate steps — precisely the "assignment operator" chaining shown in Module 1, Section 1.10:
```
TempResult1 ← step 1
TempResult2 ← step 2 using TempResult1
FinalResult ← step 3 using TempResult2
```

### Best Practice: One Concept Per CTE
Each CTE should ideally represent **one clear logical transformation** ("filter," "aggregate," "join," "rank") — avoid cramming multiple unrelated transformations into a single CTE, exactly as you'd avoid a function that does five unrelated things.

---

## 4.4 Recursive CTEs — Core Mechanics

**Definition:** A recursive CTE is a CTE that references **itself** in its own definition, using the syntax `WITH RECURSIVE` (PostgreSQL/MySQL 8+/SQL Server uses `WITH` without the keyword in some dialects — see 4.9). It consists of exactly **two parts**, combined with `UNION` or `UNION ALL`:

```sql
WITH RECURSIVE cte_name AS (
    -- 1) ANCHOR MEMBER (base case) — runs ONCE, produces the starting rows
    SELECT ...
    FROM SomeTable
    WHERE <base condition>

    UNION ALL   -- (or UNION, if duplicate elimination is desired — costlier)

    -- 2) RECURSIVE MEMBER — references cte_name itself; repeats until no new rows
    SELECT ...
    FROM SomeTable s
    JOIN cte_name c ON <recursive join condition>
)
SELECT * FROM cte_name;
```

### Execution Model (Critical to Understand — Iterative, Not Truly "Recursive" Internally)

Despite the name, most database engines execute a recursive CTE using an **iterative, fixpoint algorithm** (not literal function-call recursion):

```
Step 0:  Run the ANCHOR member  →  Working Set W0
Step 1:  Run the RECURSIVE member using W0 as "cte_name" → produces W1
         Add W1 to the final result
Step 2:  Run the RECURSIVE member using W1 (only the NEWLY added rows) as "cte_name" → produces W2
         Add W2 to the final result
   ...
Step k:  Run the RECURSIVE member using W(k-1) → produces Wk
Step k+1: If Wk is EMPTY (no new rows produced) → STOP. This is the "fixpoint."

Final Result = W0 ∪ W1 ∪ W2 ∪ ... ∪ Wk
```

> 🎯 **This is called computing a "fixpoint"** — the iteration stops naturally once a round produces **zero new rows**. This is a formal computer science concept (least fixpoint computation) and is occasionally asked directly in senior/theory-heavy interviews: *"How does a database know when to stop a recursive CTE?"* Answer: **it stops when an iteration produces no new rows** — it does NOT use a fixed depth limit by default (though engines do have safety limits to prevent infinite loops on malformed/cyclic data — see 4.7).

---

## 4.5 Recursive Trees — Hierarchical Queries

### Worked Example — Organizational Chart (Employee → Manager hierarchy)

**Sample Table `Employees`:**

| emp_id | name | manager_id |
|---|---|---|
| 1 | Zara (CEO) | NULL |
| 2 | Rohan | 1 |
| 3 | Kavya | 1 |
| 4 | Suresh | 2 |
| 5 | Meera | 2 |
| 6 | Aditi | 4 |

```
                 Zara (CEO)
                 /        \
             Rohan        Kavya
             /    \
         Suresh   Meera
           │
         Aditi
```

**Query — "List every employee with their depth level in the hierarchy, starting from the CEO":**
```sql
WITH RECURSIVE OrgChart AS (
    -- ANCHOR: the CEO (top of hierarchy)
    SELECT emp_id, name, manager_id, 1 AS level, name AS path
    FROM Employees
    WHERE manager_id IS NULL

    UNION ALL

    -- RECURSIVE: find direct reports of everyone already in OrgChart
    SELECT e.emp_id, e.name, e.manager_id, oc.level + 1,
           oc.path || ' > ' || e.name AS path
    FROM Employees e
    JOIN OrgChart oc ON e.manager_id = oc.emp_id
)
SELECT emp_id, name, level, path
FROM OrgChart
ORDER BY level, emp_id;
```

**Result:**

| emp_id | name | level | path |
|---|---|---|---|
| 1 | Zara (CEO) | 1 | Zara (CEO) |
| 2 | Rohan | 2 | Zara (CEO) > Rohan |
| 3 | Kavya | 2 | Zara (CEO) > Kavya |
| 4 | Suresh | 3 | Zara (CEO) > Rohan > Suresh |
| 5 | Meera | 3 | Zara (CEO) > Rohan > Meera |
| 6 | Aditi | 4 | Zara (CEO) > Rohan > Suresh > Aditi |

### Iteration-by-Iteration Trace

```
Iteration 0 (Anchor): W0 = { Zara (level 1) }
Iteration 1: JOIN Employees to W0 on manager_id=Zara.emp_id
             W1 = { Rohan (level 2), Kavya (level 2) }
Iteration 2: JOIN Employees to W1
             W2 = { Suresh (level 3), Meera (level 3) }   [children of Rohan; Kavya has none]
Iteration 3: JOIN Employees to W2
             W3 = { Aditi (level 4) }                      [child of Suresh]
Iteration 4: JOIN Employees to W3
             W4 = { }  (empty — Aditi has no reports)  → STOP (fixpoint reached)

Final = W0 ∪ W1 ∪ W2 ∪ W3  (6 rows total)
```

### Reverse Direction — Finding All Ancestors (Bottom-Up)

**"Find the full management chain above Aditi (emp_id=6)"**
```sql
WITH RECURSIVE ManagementChain AS (
    SELECT emp_id, name, manager_id, 1 AS level
    FROM Employees
    WHERE emp_id = 6            -- ◀ start from Aditi, going UP

    UNION ALL

    SELECT e.emp_id, e.name, e.manager_id, mc.level + 1
    FROM Employees e
    JOIN ManagementChain mc ON e.emp_id = mc.manager_id   -- ◀ note direction reversed
)
SELECT * FROM ManagementChain ORDER BY level;
```
**Result:** Aditi → Suresh → Rohan → Zara (CEO) — the entire chain of command above her.

> **Key Insight:** The **only difference** between "find all descendants" and "find all ancestors" is the **direction of the join condition** in the recursive member (`e.manager_id = oc.emp_id` vs `e.emp_id = oc.manager_id`). This symmetry is a favorite whiteboard question.

---

## 4.6 Recursive Graph Traversal

Hierarchies (trees) are a special case of the more general problem: **graph traversal**, where nodes can have **multiple parents**, **cycles**, or represent networks (social graphs, flight routes, dependency graphs).

### Worked Example — Flight Routes (Graph, Not a Tree)

**Sample Table `Routes`:**

| origin | destination | distance |
|---|---|---|
| Bengaluru | Delhi | 1740 |
| Delhi | Mumbai | 1150 |
| Mumbai | Chennai | 1030 |
| Bengaluru | Chennai | 290 |
| Chennai | Kolkata | 1370 |

**"Find all cities reachable from Bengaluru, and the number of hops"**
```sql
WITH RECURSIVE Reachable AS (
    SELECT origin, destination, 1 AS hops,
           ARRAY[origin, destination] AS visited   -- PostgreSQL array syntax; see cycle note below
    FROM Routes
    WHERE origin = 'Bengaluru'

    UNION ALL

    SELECT r.origin, r.destination, rc.hops + 1,
           rc.visited || r.destination
    FROM Routes r
    JOIN Reachable rc ON r.origin = rc.destination
    WHERE NOT r.destination = ANY(rc.visited)      -- ◀ CYCLE PREVENTION (Section 4.7)
)
SELECT DISTINCT destination, MIN(hops) AS min_hops
FROM Reachable
GROUP BY destination
ORDER BY min_hops;
```

**Result:**

| destination | min_hops |
|---|---|
| Chennai | 1 |
| Delhi | 1 |
| Mumbai | 2 |
| Kolkata | 2 |

### Tree vs Graph Recursive Query — Key Differences

| Aspect | Tree (Hierarchy) | Graph (Network) |
|---|---|---|
| Parent count per node | Exactly one (except root) | Can be multiple |
| Cycles possible? | No (by definition of a tree) | Yes — must explicitly guard against them |
| Example | Org chart, category tree, folder structure | Flight routes, social network, dependency graph |
| Extra safety needed | Usually none | **Cycle detection is mandatory** (Section 4.7) |

---

## 4.7 Cycle Detection & Termination Safety

### The Danger: Infinite Loops

If the underlying data contains a **cycle** (e.g., `A manages B`, `B manages A` — a data integrity bug, or a legitimate cyclic graph like flight routes: `Bengaluru → Chennai → Bengaluru`), a naive recursive CTE **without cycle protection will loop forever** (or until it hits an engine-imposed safety limit and errors out).

### The Standard Cycle-Prevention Pattern

**Track the path visited so far, and stop recursing if the next node is already in that path:**
```sql
WITH RECURSIVE SafeTraversal AS (
    SELECT origin, destination,
           ARRAY[origin, destination] AS path,   -- track visited nodes
           false AS is_cycle
    FROM Routes
    WHERE origin = 'Bengaluru'

    UNION ALL

    SELECT r.origin, r.destination,
           st.path || r.destination,
           r.destination = ANY(st.path)          -- mark as cycle if already visited
    FROM Routes r
    JOIN SafeTraversal st ON r.origin = st.destination
    WHERE NOT st.is_cycle                          -- ◀ STOP expanding once a cycle is flagged
)
SELECT * FROM SafeTraversal;
```

### Engine-Level Safety Nets

| Engine | Safety Mechanism |
|---|---|
| PostgreSQL | No automatic row limit by default — **you must implement cycle detection yourself** (as above), or the query can run indefinitely on cyclic data. PostgreSQL 14+ also supports the standard `CYCLE` clause: `... SEARCH DEPTH FIRST BY ... SET ordercol CYCLE destination SET is_cycle TO true DEFAULT false USING path`. |
| MySQL | Has a `cte_max_recursion_depth` session variable (default 1000) — the query **errors out** once this depth is exceeded, acting as a safety net, but this is a blunt instrument, not true cycle detection. |
| SQL Server | Has a default `MAXRECURSION 100` option (can be changed via `OPTION (MAXRECURSION n)`, or `0` for unlimited — use with extreme caution). |

> ⚠️ **Best Practice:** Never rely solely on an engine's default recursion-depth limit as your "cycle protection" — always implement explicit path-tracking (as shown above) whenever the underlying data could plausibly contain cycles. Relying on `MAXRECURSION`/`cte_max_recursion_depth` alone just converts an infinite loop into a query **error**, rather than a **correct result**.

---

## 4.8 CTE vs Derived Table vs View vs Temp Table

| Feature | CTE | Derived Table | View | Materialized View | Temp Table |
|---|---|---|---|---|---|
| Scope | Single statement | Single statement | Persistent, reusable | Persistent, reusable | Session/transaction |
| Persisted in schema? | ❌ No | ❌ No | ✅ Yes (definition only) | ✅ Yes (definition + data) | ✅ Yes (data, temporarily) |
| Can be recursive? | ✅ Yes | ❌ No | ❌ No (not directly) | ❌ No | N/A |
| Physically stored data? | Depends on optimizer (may or may not materialize) | Depends on optimizer | No (re-executes underlying query each time, unless materialized) | ✅ Yes, stored physically | ✅ Yes, stored physically |
| Indexable? | ❌ No (can't add an index to a CTE directly) | ❌ No | ❌ No (only underlying tables) | ✅ Yes | ✅ Yes |
| Best for | Readability, multi-step logic, recursion | One-off inline computation | Reusable business logic abstraction across queries | Reusable + expensive-to-recompute aggregates | Large intermediate results reused across multiple statements |

> This table is a preview bridge to **Module 8 (Views, Materialized Views, Temporary Tables)**, where each of these is covered in its own full depth.

---

## 4.9 PostgreSQL vs MySQL Notes

| Feature | PostgreSQL | MySQL |
|---|---|---|
| `WITH RECURSIVE` keyword | ✅ Required for recursive CTEs | ✅ Required (MySQL 8.0+; not available at all in 5.x) |
| CTE materialization behavior | Historically **always materialized** (a CTE was an "optimization fence"); **PostgreSQL 12+ changed this** — non-recursive CTEs are now inlined by default unless marked `MATERIALIZED` explicitly, or referenced multiple times, or recursive | Non-recursive CTEs are typically **merged/inlined** by the optimizer (similar to derived tables) |
| Explicit materialization control | ✅ `WITH x AS MATERIALIZED (...)` / `AS NOT MATERIALIZED (...)` (PG 12+) | ❌ No direct equivalent hint |
| Recursion depth safety | No default limit (must self-implement cycle detection) + optional `CYCLE` clause (PG 14+) | `cte_max_recursion_depth` system variable (default 1000) |
| Array types (used for path-tracking in cycle detection) | ✅ Native `ARRAY` type, very convenient for path tracking | ❌ No native array type — typically simulated using string concatenation (`GROUP_CONCAT`-style path strings) with a `LIKE` or `FIND_IN_SET` check for cycle detection |

> 🎯 **This PostgreSQL 12 materialization change is a very commonly tested "gotcha" for engineers who learned CTEs pre-2019**: older assumptions like "a CTE is always an optimization fence / always materialized" are **no longer true by default in modern PostgreSQL** — always verify with `EXPLAIN` for your specific engine version.

---

## 4.10 Best Practices

- **Name CTEs descriptively** (`ActiveCustomersLast30Days`, not `cte1` or `temp`) — readability is the entire point of CTEs.
- **Keep each CTE focused on one transformation** — filter, aggregate, join, or rank — not a mix of unrelated logic.
- **Always include an explicit termination-safe recursive member** — never assume the data can't have cycles unless you've verified it structurally (e.g., a strict tree with a unique-parent constraint).
- **Always include a depth/level counter** in recursive CTEs, even if not strictly needed for the final output — it's invaluable for debugging and for capping runaway recursion (`WHERE level < 50`) as an extra safety net.
- **Verify materialization behavior** for your specific engine/version using `EXPLAIN` before assuming a CTE referenced multiple times is (or isn't) computed only once.
- **Avoid overusing multiple CTEs when a single well-written join would be clearer** — CTEs aid readability, but excessive fragmentation can also hurt readability; use judgment.

---

## 4.11 Performance Considerations

- **Non-recursive CTE referenced only once:** typically inlined by modern optimizers — performance is usually identical to writing the equivalent derived table/subquery directly.
- **Non-recursive CTE referenced multiple times:** may be **materialized once and reused** (avoiding recomputation) OR **recomputed each time it's referenced**, depending on the engine and version — this can matter a lot for expensive CTEs (e.g., a CTE with a heavy aggregate, referenced 3 times in the main query).
- **Recursive CTEs are inherently iterative** and their cost scales with **(number of iterations) × (cost per iteration)** — a deep hierarchy (many levels) or a densely connected graph (many edges per node) can be expensive; always index the join columns used in the recursive member (e.g., `manager_id`, `origin`/`destination`).
- **Materialized CTEs consume memory/temp-disk space** proportional to their result size — for very large intermediate CTEs, consider whether a real (indexed) temporary table might outperform a CTE.

---

## 4.12 Common Mistakes & Edge Cases

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Forgetting the `RECURSIVE` keyword | Query fails or is misinterpreted as non-recursive in strict engines | Always write `WITH RECURSIVE` when the CTE references itself |
| Using `UNION` instead of `UNION ALL` in a large recursive CTE | `UNION` forces duplicate elimination at every iteration — expensive; often unnecessary if path-based cycle detection already prevents true duplicates | Use `UNION ALL` unless you specifically need row-level deduplication |
| No cycle protection on graph data | Infinite loop / hitting engine's default recursion limit and erroring | Implement explicit path-tracking (Section 4.7) |
| Referencing a later CTE from an earlier one | Not allowed (no forward references in standard, non-recursive CTEs) | Reorder CTEs so dependencies flow strictly top-to-bottom |
| Assuming a CTE always behaves like a materialized temp table | Engine/version-dependent (Section 4.9) | Check `EXPLAIN` for the specific engine/version in use |
| Deeply hierarchical recursive CTE with the anchor member returning too many "top-level" rows accidentally | Explosive growth in the number of rows per level | Ensure the anchor member's `WHERE` clause is precise (e.g., truly filters to root nodes only) |

---

## 4.13 Interview Questions & MCQs

### Conceptual
1. What is a CTE, and how does it relate to the Relational Algebra Assignment Operator (Module 1)?
2. What are the two mandatory parts of a recursive CTE, and what role does each play?
3. Explain, step by step, how a database engine executes a recursive CTE (the "fixpoint" model).
4. Why is `UNION ALL` generally preferred over `UNION` inside a recursive CTE?
5. What is the difference between a CTE and a derived table? When would you prefer one over the other?

### Applied (High-Frequency FAANG/SDE-2/3 Patterns)
6. Write a recursive CTE to compute the full org chart depth level for every employee, given `Employees(emp_id, name, manager_id)`.
7. Write a recursive CTE to find all cities reachable from a given starting city in a flight-routes graph, with cycle protection.
8. Explain how you would prevent infinite recursion in a graph that contains cycles, and write the SQL for it.
9. Write a query using multiple chained CTEs to: (a) compute department averages, (b) find employees above their department average, (c) rank those employees by salary within department.
10. Explain PostgreSQL's CTE materialization behavior change introduced in version 12, and why it matters for query performance tuning.

### MCQs

**Q1.** A recursive CTE must contain:
 a) Only a recursive member  b) An anchor member and a recursive member, combined via UNION/UNION ALL  c) A `WHERE RECURSIVE` clause  d) A trigger

**Q2.** A recursive CTE execution stops when:
 a) A fixed number of iterations (always 100) is reached  b) An iteration produces zero new rows (fixpoint)  c) The database runs out of memory  d) The `ORDER BY` clause is satisfied

**Q3.** Which of the following is the standard cycle-prevention technique in a recursive CTE traversing a graph?
 a) Using `DISTINCT` in the final SELECT  b) Tracking a `visited path` and excluding already-visited nodes in the recursive member  c) Using `LIMIT 1`  d) Adding an index on the primary key

**Q4.** In PostgreSQL 12+, a non-recursive CTE referenced only once is, by default:
 a) Always materialized  b) Typically inlined/merged into the main query plan  c) Converted into a view  d) Executed twice

**Q5.** MySQL's safety mechanism against runaway recursive CTEs is:
 a) `MAXRECURSION` option  b) `cte_max_recursion_depth` system variable  c) Automatic cycle detection  d) `RECURSION_LIMIT` clause

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) An anchor member and a recursive member, combined via UNION/UNION ALL**
2. **b) An iteration produces zero new rows (fixpoint)**
3. **b) Tracking a `visited path` and excluding already-visited nodes in the recursive member**
4. **b) Typically inlined/merged into the main query plan**
5. **b) `cte_max_recursion_depth` system variable**

</details>

---

## 4.14 Cheat Sheet — Module 4

```
┌───────────────────────────────────────────────────────────────────┐
│                       CTE — QUICK REFERENCE                          │
├───────────────────────────┬───────────────────────────────────────┤
│ WITH name AS (...) SELECT │ Basic CTE syntax                        │
│ WITH a AS(...), b AS(...) │ Multiple chained CTEs (b can use a)      │
│ WITH RECURSIVE            │ Required keyword for self-referencing CTE│
│ Anchor member              │ Base case, runs once                    │
│ Recursive member            │ Self-referencing part, runs iteratively │
│ UNION ALL (recursive)       │ Preferred — avoids costly per-step dedup │
│ Fixpoint                    │ Stops when an iteration adds 0 new rows │
│ Cycle protection             │ Track visited path; exclude repeats     │
│ MySQL safety net              │ cte_max_recursion_depth (default 1000)│
│ SQL Server safety net           │ MAXRECURSION option (default 100)   │
│ PostgreSQL 12+ materialization    │ Non-recursive CTEs inlined by default; │
│                                    │ use MATERIALIZED to force caching     │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 4.15 Exercises

### Beginner
1. Write a simple CTE that filters employees with `salary > 50000`, then selects from it.
2. Write two chained CTEs: one computing department averages, another filtering departments with average salary above 55000.

### Intermediate
3. Write a recursive CTE to list all direct and indirect subordinates of a given manager.
4. Write a recursive CTE to compute the level/depth of every node in a category tree (`Categories(id, name, parent_id)`).

### Advanced
5. Write a recursive CTE over a graph (`Routes(origin, destination)`) with explicit path-tracking cycle protection, to find all destinations reachable within at most 3 hops from a given city.
6. Explain and demonstrate (with SQL) the difference in behavior between `UNION` and `UNION ALL` inside a recursive CTE on data that legitimately contains duplicate paths.

### Output Prediction
7. Given a recursive CTE without any cycle protection, run against data containing a genuine cycle (A→B→A). What will happen in PostgreSQL vs MySQL vs SQL Server, respectively?

### Debugging
8. A recursive CTE meant to compute an org chart returns way more rows than there are employees. What's the most likely bug, and how would you fix it?

### Scenario-Based
9. You're asked in an interview to compute the "total reporting chain size" for every manager (i.e., how many employees, direct and indirect, report up to them eventually). Design the recursive CTE approach.

### Mini Project
10. Using a `Categories(category_id, name, parent_id)` table representing an e-commerce category tree, write: (a) a recursive CTE listing every category with its full breadcrumb path (e.g., "Electronics > Phones > Smartphones"), (b) a query counting products per category **including** all of that category's descendant categories.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [SQLBolt — Interactive SQL Lessons](https://sqlbolt.com/)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

- A **CTE** is a named, temporary result set defined via `WITH`, scoped to a single statement — the SQL realization of Module 1's RA Assignment Operator.
- **Multiple CTEs** can be chained, with later CTEs referencing earlier ones (never the reverse, except in recursion).
- A **recursive CTE** has exactly two parts — an **anchor member** (base case) and a **recursive member** (self-referencing), combined via `UNION`/`UNION ALL`, executed via an **iterative fixpoint algorithm** that stops when an iteration adds zero new rows.
- **Hierarchies (trees)** are naturally safe from cycles; **graphs** are not — **explicit cycle detection via path-tracking is mandatory** for graph traversal.
- CTE **materialization behavior is engine/version-specific** — notably, PostgreSQL changed its default behavior in version 12.
- Always verify recursive CTE performance and safety using `EXPLAIN`, proper indexing on join columns, and explicit depth/cycle guards.

---

## Cross-References for Later Parts / Modules
- **Window Functions as an alternative to some CTE-based ranking patterns** → Module 5 (next)
- **Views / Materialized Views / Temp Tables (full comparison)** → Module 8
- **Query optimizer's inlining vs materialization decisions** → Part 8 (Query Execution Internals)
- **Indexing strategy for recursive join columns** → Part 8

---

> ✅ **Module 4 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 5"** to proceed to **Module 5: Window Functions** — the `OVER` clause, ranking functions (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`), aggregate window functions, value functions (`LEAD`, `LAG`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`), and window frames (`ROWS`/`RANGE`/`GROUPS`).
