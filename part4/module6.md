# The Complete DBMS Handbook — Part 4

## 📘 MODULE 6: ADVANCED AGGREGATION

> **Continuing from Module 5 (Window Functions).** Where Module 5 preserved rows while adding computed values, this module returns to `GROUP BY`-style row collapsing — but extends it far beyond a single flat grouping, letting one query compute **multiple levels of subtotals and grand totals simultaneously**.

---

## 6.0 Chapter Roadmap

```
Module 6: Advanced Aggregation
│
├── 6.1 Recap: How Plain GROUP BY Works
├── 6.2 GROUPING SETS
├── 6.3 ROLLUP
├── 6.4 CUBE
├── 6.5 GROUPING() Function
├── 6.6 FILTER Clause
├── 6.7 HAVING Internals
├── 6.8 PostgreSQL vs MySQL Notes
├── 6.9 Best Practices
├── 6.10 Common Mistakes & Edge Cases
├── 6.11 Performance Notes
├── 6.12 Interview Questions & MCQs
├── 6.13 Cheat Sheet
└── 6.14 Exercises
```

---

## 6.1 Recap: How Plain GROUP BY Works

**Sample Table `Sales`:**

| region | product | quarter | revenue |
|---|---|---|---|
| North | Widget | Q1 | 1000 |
| North | Gadget | Q1 | 1500 |
| South | Widget | Q1 | 800 |
| South | Gadget | Q1 | 1200 |
| North | Widget | Q2 | 1100 |
| South | Widget | Q2 | 900 |

A plain `GROUP BY region, product` collapses rows into **one group per unique (region, product) combination**:
```sql
SELECT region, product, SUM(revenue) AS total_revenue
FROM Sales
GROUP BY region, product;
```

| region | product | total_revenue |
|---|---|---|
| North | Widget | 2100 |
| North | Gadget | 1500 |
| South | Widget | 1700 |
| South | Gadget | 1200 |

### The Business Problem This Module Solves

A common real-world reporting need is: *"Give me totals by (region, product), AND totals by region alone, AND totals by product alone, AND a single grand total — all in one report."* Doing this with plain `GROUP BY` requires **running 4 separate queries and manually stitching them together with `UNION ALL`** — tedious, error-prone, and inefficient (the engine re-scans the data 4 times). **`GROUPING SETS`, `ROLLUP`, and `CUBE`** solve this in a **single query, single scan**.

---

## 6.2 GROUPING SETS

**Definition:** `GROUPING SETS` lets you specify **multiple, explicit grouping combinations** in a single `GROUP BY` clause — the engine computes each specified grouping and `UNION ALL`s the results together internally, in one pass.

```sql
SELECT region, product, SUM(revenue) AS total_revenue
FROM Sales
GROUP BY GROUPING SETS (
    (region, product),   -- grouping level 1: by region AND product
    (region),             -- grouping level 2: by region only
    (product),             -- grouping level 3: by product only
    ()                       -- grouping level 4: grand total (no grouping at all)
);
```

**Result (order may vary; conceptually 4 blocks of rows):**

| region | product | total_revenue |
|---|---|---|
| North | Widget | 2100 |
| North | Gadget | 1500 |
| South | Widget | 1700 |
| South | Gadget | 1200 |
| North | NULL | 3600 |
| South | NULL | 2900 |
| NULL | Widget | 3800 |
| NULL | Gadget | 2700 |
| NULL | NULL | 6500 |

> 🎯 **Critical Rule: `NULL` here means "this column was NOT part of this particular grouping," not "the actual data value was NULL."** This is a genuinely important, frequently-tested distinction — see Section 6.5 (`GROUPING()`) for how to disambiguate a "real" `NULL` in the data from this "aggregation placeholder" `NULL`.

### Manual Equivalent (What GROUPING SETS Replaces)
```sql
SELECT region, product, SUM(revenue) FROM Sales GROUP BY region, product
UNION ALL
SELECT region, NULL, SUM(revenue) FROM Sales GROUP BY region
UNION ALL
SELECT NULL, product, SUM(revenue) FROM Sales GROUP BY product
UNION ALL
SELECT NULL, NULL, SUM(revenue) FROM Sales;
```

#### Why It Exists
- **Single-scan efficiency**: the engine reads the base table **once** and computes all requested grouping levels together, rather than 4 separate full scans + a `UNION ALL`.
- **Report-friendliness**: subtotal/grand-total reports (classic BI/OLAP requirement) become a single, declarative query.

#### Interview Importance: ⭐⭐⭐⭐ — Common in Data/Analytics Engineer and Senior Backend interviews involving reporting dashboards.

---

## 6.3 ROLLUP

**Definition:** `ROLLUP` is **syntactic shorthand** for a specific, commonly-needed **hierarchical** subset of `GROUPING SETS` — it generates subtotals for a **progressively shrinking prefix** of the given column list, ending in a grand total. It assumes the columns have a **hierarchical relationship** (e.g., region → product, or year → quarter → month).

```sql
SELECT region, product, SUM(revenue) AS total_revenue
FROM Sales
GROUP BY ROLLUP (region, product);
```

**This is exactly equivalent to:**
```sql
GROUP BY GROUPING SETS (
    (region, product),   -- full detail
    (region),             -- subtotal by region (drop the LAST column)
    ()                      -- grand total (drop everything)
);
```

**Result:**

| region | product | total_revenue |
|---|---|---|
| North | Widget | 2100 |
| North | Gadget | 1500 |
| North | NULL | 3600 ← subtotal for North |
| South | Widget | 1700 |
| South | Gadget | 1200 |
| South | NULL | 2900 ← subtotal for South |
| NULL | NULL | 6500 ← grand total |

> **Notice `ROLLUP` does NOT produce a `(product)`-only subtotal row** (unlike the full `GROUPING SETS` example in 6.2) — this is the key distinguishing feature: **`ROLLUP(A, B)` only ever drops columns from the RIGHT, one at a time** (`(A,B)` → `(A)` → `()`), reflecting a hierarchical "drill-up" pattern (e.g., day → month → year), not every possible combination.

### The General ROLLUP Pattern

`ROLLUP(A, B, C)` produces exactly these **N+1** grouping sets (for N columns):
```
(A, B, C)
(A, B)
(A)
()
```

### Real-life Analogy
`ROLLUP` is exactly like a **financial statement**: line items → subtotals per category → grand total. You'd naturally roll up "Product A revenue in North" + "Product B revenue in North" into a "North Total," then roll up all regional totals into a "Company Grand Total" — always **collapsing from the most specific level upward**, never skipping to an unrelated combination.

#### Interview Importance: ⭐⭐⭐⭐ — Very common for "generate a sales report with subtotals" style questions.

---

## 6.4 CUBE

**Definition:** `CUBE` generates subtotals for **every possible combination** of the given columns — the full "power set" of groupings, not just the hierarchical prefix subset that `ROLLUP` produces.

```sql
SELECT region, product, SUM(revenue) AS total_revenue
FROM Sales
GROUP BY CUBE (region, product);
```

**This is exactly equivalent to the FULL `GROUPING SETS` example from Section 6.2** — `(region, product)`, `(region)`, `(product)`, `()` — **all 2² = 4 combinations** for 2 columns.

### CUBE vs ROLLUP — The Core Distinction

For `N` columns:
- `ROLLUP(A, B, ..., N)` produces **N + 1** grouping sets (only hierarchical prefixes, dropping from the right).
- `CUBE(A, B, ..., N)` produces **2^N** grouping sets (every possible subset, including all "skip a middle column" combinations).

```
For 2 columns (region, product):

ROLLUP produces 3 grouping sets:          CUBE produces 4 grouping sets:
  (region, product)                          (region, product)
  (region)                                    (region)
  ()                                            (product)   ← CUBE has this; ROLLUP does NOT
                                                  ()

For 3 columns (region, product, quarter):

ROLLUP produces 4 grouping sets:          CUBE produces 8 grouping sets:
  (region, product, quarter)                 (region, product, quarter)
  (region, product)                          (region, product)
  (region)                                    (region, quarter)   ← CUBE only
  ()                                            (region)
                                                  (product, quarter) ← CUBE only
                                                  (product)
                                                  (quarter)          ← CUBE only
                                                  ()
```

> 🎯 **The N+1 vs 2^N formula is a very frequently tested exact-count question** — e.g., *"How many grouping sets does `CUBE(a, b, c)` produce?"* → **2³ = 8**. *"How many does `ROLLUP(a, b, c)` produce?"* → **3 + 1 = 4**.

### Real-life Analogy
`CUBE` is like a Rubik's Cube of dimensions — you can slice and view the data from **every angle simultaneously**: by region alone, by product alone, by quarter alone, by any pair, or by all three together. `ROLLUP` only lets you "zoom out" along one fixed hierarchical axis; `CUBE` lets you look at the data from **every possible angle at once**.

#### Interview Importance: ⭐⭐⭐ — Less common than `ROLLUP` in day-to-day reporting, but a favorite theory/OLAP-adjacent interview question, especially for Data Engineer roles.

---

## 6.5 GROUPING() Function

**Definition:** `GROUPING(column)` returns `1` if the given column was **"rolled up away"** in the current row (i.e., it's a subtotal/grand-total row where that column doesn't apply), and `0` if the column is a genuine grouping value for that row — **this is exactly the tool needed to disambiguate the "placeholder NULL" (from GROUPING SETS/ROLLUP/CUBE) from a "real NULL" that might genuinely exist in your data.**

```sql
SELECT
    region,
    product,
    SUM(revenue) AS total_revenue,
    GROUPING(region) AS is_region_rolled_up,
    GROUPING(product) AS is_product_rolled_up
FROM Sales
GROUP BY ROLLUP (region, product);
```

| region | product | total_revenue | is_region_rolled_up | is_product_rolled_up |
|---|---|---|---|---|
| North | Widget | 2100 | 0 | 0 |
| North | Gadget | 1500 | 0 | 0 |
| North | NULL | 3600 | 0 | **1** |
| South | Widget | 1700 | 0 | 0 |
| South | Gadget | 1200 | 0 | 0 |
| South | NULL | 2900 | 0 | **1** |
| NULL | NULL | 6500 | **1** | **1** |

### A Common, Practical Use — Labeling Subtotal Rows for Reports
```sql
SELECT
    CASE WHEN GROUPING(region) = 1 THEN 'ALL REGIONS' ELSE region END AS region,
    CASE WHEN GROUPING(product) = 1 THEN 'ALL PRODUCTS' ELSE product END AS product,
    SUM(revenue) AS total_revenue
FROM Sales
GROUP BY ROLLUP (region, product);
```
This produces business-readable labels like `"North" / "ALL PRODUCTS" / 3600` for the subtotal row — a very common real-world reporting pattern.

> ⚠️ **Why this matters (the exact bug it prevents):** If `region` genuinely had a `NULL` value in the raw data (e.g., an unclassified sale), a naive `WHERE region IS NULL` check on the output **cannot tell** whether that `NULL` came from real data or from `ROLLUP`'s subtotal mechanism — `GROUPING(region)` is the only reliable way to distinguish them.

---

## 6.6 FILTER Clause

**Definition:** The `FILTER (WHERE condition)` clause, attached to an aggregate function, restricts **which rows that specific aggregate considers** — **without affecting other aggregates in the same query or requiring a separate `CASE WHEN`.**

```sql
SELECT
    region,
    COUNT(*) AS total_sales,
    COUNT(*) FILTER (WHERE product = 'Widget') AS widget_sales,
    SUM(revenue) FILTER (WHERE quarter = 'Q1') AS q1_revenue
FROM Sales
GROUP BY region;
```

**Result:**

| region | total_sales | widget_sales | q1_revenue |
|---|---|---|---|
| North | 3 | 2 | 2500 |
| South | 3 | 2 | 2000 |

### FILTER vs the Older CASE WHEN Workaround

**Before `FILTER` existed (and still needed in engines without it, like MySQL — see 6.8), the same result required:**
```sql
SELECT
    region,
    COUNT(*) AS total_sales,
    SUM(CASE WHEN product = 'Widget' THEN 1 ELSE 0 END) AS widget_sales,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue ELSE 0 END) AS q1_revenue
FROM Sales
GROUP BY region;
```

| Aspect | `FILTER` | `CASE WHEN` |
|---|---|---|
| Readability | Cleaner — condition is visually attached to its aggregate | Works, but visually separates the condition from the aggregate's "shape" |
| `NULL` vs `0` handling for COUNT | `FILTER` naturally excludes non-matching rows entirely | Needs careful `CASE WHEN ... THEN 1 ELSE 0 END` (with `SUM`) or `THEN col ELSE NULL END` (with `COUNT`) to get correct semantics |
| Standard SQL | ✅ Yes (SQL:2003 standard) | ✅ Universally supported (works in every engine) |
| Engine support | PostgreSQL ✅, MySQL ❌ (see 6.8) | ✅ Universal |

#### Best Practice
- Prefer `FILTER` for readability when your engine supports it (PostgreSQL); fall back to `CASE WHEN` for cross-engine compatibility (MySQL, older engines).

---

## 6.7 HAVING Internals

**Definition:** `HAVING` filters **groups** (post-aggregation), as opposed to `WHERE`, which filters **rows** (pre-aggregation). This distinction was already introduced in Module 1 (Section 1.11's logical execution order), but here we go deeper into *why* and *how* it works internally.

### WHERE vs HAVING — Precise Execution-Order Recap

```
Logical Execution Order (from Module 1, Section 1.11):

1. FROM        (+ JOINs)
2. WHERE       ← filters INDIVIDUAL ROWS, BEFORE grouping. Cannot reference aggregate functions.
3. GROUP BY    ← rows are collapsed into groups here
4. HAVING      ← filters GROUPS, AFTER aggregation. CAN reference aggregate functions.
5. SELECT
6. ORDER BY
```

**Concrete Example Showing Why This Matters:**
```sql
SELECT region, SUM(revenue) AS total_revenue
FROM Sales
WHERE quarter = 'Q1'              -- ◀ row filter: only Q1 rows even enter the grouping step
GROUP BY region
HAVING SUM(revenue) > 2000;        -- ◀ group filter: only regions whose Q1 total exceeds 2000
```

> ⚠️ **Why `WHERE SUM(revenue) > 2000` is a syntax error:** at the point `WHERE` is logically evaluated, **grouping hasn't happened yet** — there is no "group" for `SUM()` to aggregate over. `SUM()` only makes sense *after* `GROUP BY` has run, which is exactly why the SQL standard restricts aggregate functions to `HAVING`, `SELECT`, and `ORDER BY` — never `WHERE`.

### HAVING Without GROUP BY

`HAVING` can technically be used **without** an explicit `GROUP BY` — in that case, the **entire table is treated as one single group** (same "one big partition" idea from Module 5):
```sql
SELECT SUM(revenue) AS total_revenue
FROM Sales
HAVING SUM(revenue) > 5000;   -- either returns the one grand-total row, or zero rows
```

### HAVING with GROUPING SETS/ROLLUP/CUBE
`HAVING` can filter on the **subtotal rows** too, often combined with `GROUPING()`:
```sql
SELECT region, product, SUM(revenue) AS total_revenue
FROM Sales
GROUP BY ROLLUP (region, product)
HAVING GROUPING(product) = 1;   -- keep ONLY the region-level subtotal rows (and grand total)
```

---

## 6.8 PostgreSQL vs MySQL Notes

| Feature | PostgreSQL | MySQL |
|---|---|---|
| `GROUPING SETS` | ✅ Full support | ✅ Supported from MySQL 8.0+ (not in 5.x) |
| `ROLLUP` | ✅ Full support (`GROUP BY ROLLUP(...)`) | ✅ Supported; also has legacy `WITH ROLLUP` syntax (`GROUP BY region, product WITH ROLLUP`) which behaves similarly but with some differences in `NULL`-labeling and lacks full `GROUPING SETS`/`CUBE` flexibility |
| `CUBE` | ✅ Full support | ✅ Supported from MySQL 8.0+ |
| `GROUPING()` function | ✅ Supported | ✅ Supported from MySQL 8.0+ |
| `FILTER (WHERE ...)` clause | ✅ Supported | ❌ **Not supported** — must use `CASE WHEN` inside the aggregate instead |

> 🎯 **The `FILTER` clause gap is a very practically relevant, frequently-tested cross-platform difference** — if you're asked to write portable SQL, default to the `CASE WHEN` pattern; if PostgreSQL-specific, `FILTER` is preferred for clarity.

---

## 6.9 Best Practices

- Use `ROLLUP` for naturally hierarchical dimensions (date hierarchies: day → month → year; org hierarchies: employee → team → department).
- Use `CUBE` only when you genuinely need **every** combination — it grows as **2^N**, so `CUBE` on 5+ columns can produce an enormous, often impractical number of grouping combinations (2⁵ = 32); prefer explicit `GROUPING SETS` to hand-pick only the combinations your report actually needs.
- Always pair `ROLLUP`/`CUBE`/`GROUPING SETS` output with `GROUPING()` in real reporting code to correctly label subtotal/grand-total rows, rather than leaving ambiguous `NULL`s for downstream consumers to misinterpret.
- Prefer `FILTER` over `CASE WHEN` for readability when your target engine supports it; otherwise standardize on `CASE WHEN` for portability.

---

## 6.10 Common Mistakes & Edge Cases

| Mistake | Why It's Wrong | Fix |
|---|---|---|
| Treating `ROLLUP`'s placeholder `NULL` as a real data `NULL` | Silently mixes subtotal rows with genuine NULL-valued data rows in downstream filtering/reporting | Use `GROUPING(column)` to disambiguate |
| Assuming `ROLLUP(a, b)` produces a `(b)`-only subtotal | `ROLLUP` only drops from the right, hierarchically — it never produces `(b)` alone | Use `CUBE` or explicit `GROUPING SETS` if you need that combination |
| Using an aggregate function in `WHERE` | Aggregates aren't defined yet at the `WHERE` stage (Section 6.7) | Use `HAVING` for post-aggregation filtering |
| Assuming `CUBE` scales linearly with column count | It's exponential (2^N) — very expensive for many columns | Use `GROUPING SETS` to hand-pick only needed combinations |
| Using `FILTER` in MySQL | Not supported — syntax error | Use `CASE WHEN` inside the aggregate instead |

---

## 6.11 Performance Notes

- `GROUPING SETS`/`ROLLUP`/`CUBE` are computed via a **single pass** over the base data by the optimizer (typically via a sort-based or hash-based multi-level aggregation strategy) — this is **significantly more efficient** than manually running N separate `GROUP BY` queries and `UNION ALL`-ing them, because the base table is scanned only once.
- `CUBE`'s exponential growth (2^N grouping sets) directly translates into exponentially more aggregation work and exponentially larger result sets — always sanity-check the expected row count before running `CUBE` on more than 3–4 columns in production.
- `FILTER`-based conditional aggregates typically perform comparably to `CASE WHEN` equivalents, since both ultimately guide which rows contribute to which aggregate accumulator during the same single scan.

---

## 6.12 Interview Questions & MCQs

### Conceptual
1. What problem do `GROUPING SETS`, `ROLLUP`, and `CUBE` solve that plain `GROUP BY` cannot?
2. What is the difference between `ROLLUP` and `CUBE`, in terms of both the grouping combinations produced and the underlying use case?
3. Why does a `NULL` appear in `ROLLUP`/`CUBE` output, and how do you distinguish it from a genuine data `NULL`?
4. Explain, with the logical SQL execution order, why aggregate functions cannot be used in `WHERE`.
5. What does the `FILTER` clause do, and what is its pre-SQL:2003-standard equivalent?

### Applied
6. Given `Sales(region, product, quarter, revenue)`, write a query using `ROLLUP` to produce subtotals by region and a grand total.
7. Write the equivalent `CUBE(region, product)` query using explicit `GROUPING SETS`, listing all 4 grouping combinations by hand.
8. Write a query using `GROUPING()` to label subtotal rows as `'ALL PRODUCTS'` in a `ROLLUP(region, product)` report.
9. Write a query using `FILTER` (or `CASE WHEN` for portability) to compute, in one query, both total sales and Q1-only sales per region.
10. How many grouping sets does `CUBE(a, b, c, d)` produce? How many does `ROLLUP(a, b, c, d)` produce? Show the formula.

### MCQs

**Q1.** `ROLLUP(a, b, c)` produces how many grouping sets?
 a) 3  b) 4  c) 8  d) 6

**Q2.** `CUBE(a, b)` produces how many grouping sets?
 a) 2  b) 3  c) 4  d) 8

**Q3.** `GROUPING(col) = 1` means:
 a) The column's value is a genuine NULL in the source data
 b) The column was aggregated away (rolled up) in this row
 c) The column is the primary key
 d) The column has a duplicate value

**Q4.** Which clause filters groups AFTER aggregation?
 a) WHERE  b) HAVING  c) GROUP BY  d) FILTER

**Q5.** Which engine does NOT support the `FILTER (WHERE ...)` clause?
 a) PostgreSQL  b) MySQL  c) Both support it  d) Neither supports it

<details>
<summary><strong>Click to reveal MCQ Answers</strong></summary>

1. **b) 4** — (a,b,c), (a,b), (a), ()
2. **c) 4** — 2² = 4
3. **b) The column was aggregated away (rolled up) in this row**
4. **b) HAVING**
5. **b) MySQL**

</details>

---

## 6.13 Cheat Sheet — Module 6

```
┌───────────────────────────────────────────────────────────────────┐
│                 ADVANCED AGGREGATION — QUICK REFERENCE                │
├───────────────────────────┬───────────────────────────────────────┤
│ GROUPING SETS ((a,b),(a),())│ Explicit, hand-picked grouping combos │
│ ROLLUP(a, b)                 │ Hierarchical prefixes: (a,b),(a),()   │
│                                │ → N+1 grouping sets for N columns    │
│ CUBE(a, b)                     │ Every combination: (a,b),(a),(b),()  │
│                                  │ → 2^N grouping sets for N columns   │
│ GROUPING(col)                    │ 1 = rolled up (placeholder NULL),   │
│                                    │ 0 = genuine grouping value          │
│ FILTER (WHERE cond)                │ Conditional aggregate (PG only;     │
│                                      │ use CASE WHEN for MySQL)           │
│ WHERE                                 │ Filters ROWS, before grouping       │
│ HAVING                                 │ Filters GROUPS, after aggregation   │
└───────────────────────────┴───────────────────────────────────────┘
```

---

## 6.14 Exercises

### Beginner
1. Write a `ROLLUP` query on `Sales(region, revenue)` to get per-region totals plus a grand total.
2. Write a `HAVING` clause to filter regions whose total revenue exceeds 2000.

### Intermediate
3. Write a `CUBE(region, product)` query and manually list which grouping-set rows appear in the output that would NOT appear in the equivalent `ROLLUP`.
4. Write a query using `GROUPING()` to replace placeholder NULLs with readable labels like `'All Regions'`.

### Advanced
5. Write an explicit `GROUPING SETS` query that reproduces `ROLLUP(year, quarter, month)`'s output, listing all 4 grouping sets by hand.
6. Write a query using `FILTER` (with a `CASE WHEN` fallback noted) to compute, per region, total revenue, Q1 revenue, and Q2 revenue in a single query.

### Output Prediction
7. Given `CUBE(a, b, c)`, how many total grouping-set rows worth of output are theoretically possible (ignoring data-specific cardinality), and list them all symbolically.

### Debugging
8. A report built on `ROLLUP(region, product)` output filters `WHERE product IS NULL` to find "region subtotal rows," but this accidentally also includes real unclassified-product sales rows. Diagnose and fix using `GROUPING()`.

### Scenario-Based
9. You're asked to build a single-query sales dashboard showing: per-product-per-region detail, per-region subtotal, per-product subtotal, and a grand total. Which construct (`GROUPING SETS`, `ROLLUP`, or `CUBE`) is the right fit, and why?

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [HackerRank SQL Domain](https://www.hackerrank.com/domains/sql)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

- **`GROUPING SETS`** lets you compute multiple explicit grouping combinations in a single, single-scan query.
- **`ROLLUP(a, b, ..., n)`** produces **N+1** hierarchical-prefix grouping sets, ideal for drill-up reports (e.g., day → month → year).
- **`CUBE(a, b, ..., n)`** produces **2^N** grouping sets — every possible combination — but grows exponentially, so use with caution on many columns.
- **`GROUPING(col)`** disambiguates a "rolled-up placeholder NULL" from a genuine data NULL — essential for correct, readable reporting output.
- **`FILTER (WHERE ...)`** provides clean conditional aggregation in PostgreSQL; MySQL requires the `CASE WHEN` workaround.
- **`HAVING`** filters groups after aggregation, while **`WHERE`** filters rows before — aggregate functions are only valid from `HAVING` onward in the logical execution order.

---

## Cross-References for Later Parts / Modules
- **Set Operations (UNION/INTERSECT/EXCEPT) — the manual alternative GROUPING SETS replaces** → Module 7 (next)
- **Materialized Views for precomputing expensive CUBE/ROLLUP reports** → Module 8
- **OLAP & Data Warehousing concepts (star schema, fact/dimension tables)** → later Parts
- **Query optimizer's multi-level aggregation execution strategy** → Part 8 (Query Execution Internals)

---

> ✅ **Module 6 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 7"** to proceed to **Module 7: Set Operations** — `UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT`/`MINUS`, duplicate elimination, and performance differences.
