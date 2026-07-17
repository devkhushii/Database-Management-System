# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 8: First Normal Form (1NF)

> **Where we are:** Module 7 proved *why* we normalize (anomalies, redundancy). Starting now, Modules 8–12 walk through the actual normal forms in strict, increasing order: **1NF → 2NF → 3NF → BCNF → 4NF/5NF/DKNF**. Each normal form is a *stricter* rule than the one before it — meaning a relation must satisfy 1NF before 2NF is even a meaningful question, must satisfy 2NF before 3NF applies, and so on. 1NF is the **absolute foundation** — every relation in a properly relational database must satisfy it.

---

### 8.1 What Is First Normal Form (1NF)?

**Definition:**
A relation is in **First Normal Form (1NF)** if and only if every attribute in every row contains only **atomic** (indivisible) values, and each attribute holds values of a **single, consistent type**, with **no repeating groups** or **multi-valued** entries within a single cell.

**Why does it exist?**
The entire relational model, as conceived by E.F. Codd, is built on the mathematical concept of a **relation as a set of tuples over well-defined domains**. A "domain" in this mathematical sense only makes sense if every value is atomic — you can't meaningfully reason about functional dependencies, keys, or even basic set operations (union, intersection) if a single cell secretly contains a list, a nested table, or a comma-separated string pretending to be one value. **1NF is what makes a table a genuine mathematical relation** rather than just a grid of text.

**What problem does it solve?**
Without 1NF, you can't reliably:
- Filter or search on individual values within a multi-valued field (e.g., `WHERE 'Python' IN skills` becomes a fragile string-matching hack instead of a clean query).
- Update a single value in a multi-valued cell without rewriting the entire cell.
- Enforce any meaningful constraint (uniqueness, foreign keys) on the individual values buried inside a non-atomic cell.
- Apply functional dependency theory at all — FD theory assumes atomic domains as a starting axiom.

---

### 8.2 Atomicity — What Does "Atomic" Actually Mean?

**Definition:**
A value is **atomic** if it cannot be meaningfully **subdivided further** from the perspective of the relational model and the queries the application needs to run on it.

**Important nuance:** Atomicity is somewhat **context-dependent**. A `full_name` field like `"Aditi Sharma"` might be perfectly atomic for an application that never needs to search by first or last name separately — but if the business requires searching/sorting by last name, then `full_name` is *not* sufficiently atomic for that use case, and should be split into `first_name` and `last_name`.

**Classic examples of NON-atomic values:**

| Non-Atomic Value (violates 1NF) | Why It's a Problem |
|---|---|
| `"Python, SQL, Java"` (comma-separated skills in one cell) | Can't query "who knows SQL?" without fragile string parsing |
| `{9876543210, 9123456780}` (a set/array of phone numbers in one cell) | Can't enforce per-number uniqueness or easily add/remove one number |
| A nested table (a "table within a table" cell, sometimes seen in old hierarchical or document-style imports) | Breaks the flat, tabular structure the relational model assumes entirely |
| `"12 MG Road, Bangalore, 560001"` (a full address crammed into one field) when the business needs to query by city or postal code | City/postal code are not independently queryable |

---

### 8.3 A Concrete Violation and Its Fix

**Violates 1NF:**

```
Table: Student
+-----------+--------+------------------------+
| roll_no   | name   | phone_numbers            |
+-----------+--------+------------------------+
| 101       | Aditi  | 9876543210, 9123456780   |
| 102       | Rohan  | 9988776655               |
+-----------+--------+------------------------+
```

The `phone_numbers` column holds **multiple values** in a single cell for Aditi — this directly violates atomicity.

**Fix — Option A: Separate rows (if a student can have multiple phone numbers, and the relationship itself needs no additional attributes)**

```
Table: Student_Phones
+-----------+--------------+
| roll_no   | phone_number  |
+-----------+--------------+
| 101       | 9876543210    |
| 101       | 9123456780    |
| 102       | 9988776655    |
+-----------+--------------+
```

**Fix — Option B: Fixed separate columns (only appropriate if the number of phone numbers is small AND genuinely fixed, e.g., "primary" and "emergency" contact — otherwise this reintroduces a different design smell, Section 8.7)**

```
Table: Student
+-----------+--------+----------------+------------------+
| roll_no   | name   | primary_phone   | emergency_phone   |
+-----------+--------+----------------+------------------+
| 101       | Aditi  | 9876543210      | 9123456780         |
+-----------+--------+----------------+------------------+
```

---

### 8.4 Rules for 1NF Compliance — Complete List

A relation is in 1NF if **all** of the following hold:

1. **Every attribute contains only atomic (single) values** — no lists, sets, arrays, or comma-separated groups within one cell.
2. **Every attribute contains values of a single, consistent type/domain** — a column shouldn't mix, say, numeric IDs in some rows and free text in others.
3. **There are no repeating groups** — you should not need multiple similarly-named columns like `phone1, phone2, phone3` to represent an inherently variable-length list of the same kind of fact (this is a specific violation pattern called a **repeating group**, distinct from simple non-atomicity — Section 8.7).
4. **Each row is uniquely identifiable** — there must be some key (even if just the full row) that distinguishes every row; a relation, by the mathematical definition, is a **set** of tuples, and sets cannot contain exact duplicates.
5. **The order of rows and columns carries no meaning** — the relational model treats a relation as an unordered set of tuples; if your application logic secretly depends on "row 3 always coming after row 2," that's a modeling smell, not something the relational model itself provides guarantees about.

---

### 8.5 Examples — Compliant vs Non-Compliant

**NON-compliant (multi-valued attribute):**
```
Employee(emp_id, name, skills)
101 | Aditi | "Python, SQL, Docker"
```

**Compliant (after fixing atomicity via a separate table):**
```
Employee(emp_id, name)
101 | Aditi

Employee_Skills(emp_id, skill)
101 | Python
101 | SQL
101 | Docker
```

**NON-compliant (repeating groups as columns):**
```
Employee(emp_id, name, skill_1, skill_2, skill_3)
101 | Aditi | Python | SQL | Docker
102 | Rohan | Java | NULL | NULL
```

**Compliant (same fix — a separate table with one row per skill):**
```
Employee(emp_id, name)
101 | Aditi
102 | Rohan

Employee_Skills(emp_id, skill)
101 | Python
101 | SQL
101 | Docker
102 | Java
```

---

### 8.6 Mathematical Intuition — Why 1NF Is a Precondition for Everything Else

Recall from Module 2 that a functional dependency `X → Y` is defined as: *"for every pair of tuples, if they agree on X, they agree on Y."* This definition implicitly assumes you can meaningfully **compare** `t1[Y]` and `t2[Y]` as single values. If `Y` were a multi-valued/non-atomic cell, "agreement" becomes ambiguous — does `{Python, SQL}` "agree" with `{SQL, Python}` (same set, different order)? With `{Python}` (a subset)? The entire mathematical apparatus of FDs, closures, and keys (Modules 2–6) silently assumes 1NF as a starting point. **This is precisely why 1NF must come first — every later normal form and every algorithm we've built is only well-defined on top of atomic domains.**

---

### 8.7 Repeating Groups — A Closer Look

**Definition:**
A **repeating group** is a set of columns that logically represent the *same kind of fact*, repeated a fixed (but often arbitrary) number of times as separate columns — e.g., `skill_1, skill_2, skill_3`, or `phone_1, phone_2`.

**Why is this a distinct violation pattern (not just "non-atomicity")?**
Technically, each individual cell in a repeating-group design (`skill_1 = 'Python'`) **is** atomic on its own — the violation is structural: the design is trying to represent an inherently **variable-length, repeating** fact using a **fixed number of columns**, which:
- Wastes space for rows with fewer values (lots of `NULL`s, as seen in Rohan's row in Section 8.5).
- **Silently truncates data** for rows that need more values than there are columns (what happens if an employee has 5 skills but the table only has 3 skill columns?).
- Makes querying "does this employee have skill X?" awkward — you need to check `skill_1 OR skill_2 OR skill_3` instead of a single clean condition.

**The fix is always the same:** move the repeating fact into its **own table**, with one row per occurrence, related back via a foreign key — exactly as shown in Section 8.5's compliant example.

---

### 8.8 Violations — A Broader Catalogue

| Violation Pattern | Example | Why It Breaks 1NF |
|---|---|---|
| Comma-separated multi-value cell | `"Python, SQL, Java"` | Non-atomic single cell |
| Array/set-typed column (even if the DBMS technically supports an array type) | PostgreSQL `TEXT[]` column storing multiple skills | Technically representable, but the *logical model* still isn't in 1NF — Section 8.10 discusses this nuance |
| Repeating groups as separate numbered columns | `skill_1, skill_2, skill_3` | Fixed-width representation of a variable-length fact |
| A "type-code + generic value" column combining multiple domains | A `value` column that holds a date in some rows and a number in others, depending on a separate `type` column | Violates "single, consistent domain" rule |
| Nested/embedded sub-tables within a cell | A JSON blob or XML fragment stored in a single column representing a whole sub-record | Breaks flat tabular structure entirely |

---

### 8.9 The Conversion Process — Step by Step

```
Algorithm: Convert-To-1NF(R)

1. Identify every attribute that is multi-valued, non-atomic, or part of a repeating group.
2. For each such attribute:
   a. Create a NEW table containing:
        - the primary key of the original table (as a foreign key), and
        - a single column for the individual atomic value.
   b. Remove the offending column(s) from the original table.
3. The original table's primary key + the new table's foreign key
   together typically form the new table's own key
   (often the FULL primary key, since a single entity can have
    multiple associated values).
4. Verify: does every remaining attribute in every table now hold
   a single, atomic value? If yes, all resulting tables are in 1NF.
```

**Worked example — applying this to Section 8.5's Employee_Skills case:**

Original (violates 1NF): `Employee(emp_id, name, skills)` where `skills` is multi-valued.

1. Identify: `skills` is the offending multi-valued attribute.
2. Create new table: `Employee_Skills(emp_id, skill)`.
3. Remove `skills` from `Employee` → `Employee(emp_id, name)`.
4. New key for `Employee_Skills`: `{emp_id, skill}` (since one employee can have many skills, and — depending on your business rule — a single skill likely doesn't repeat for the same employee, so the pair together uniquely identifies a row).
5. Verify: `Employee(emp_id, name)` — both atomic. `Employee_Skills(emp_id, skill)` — both atomic. **1NF achieved.**

---

### 8.10 Industry Implementation Notes

- Modern DBMSs like PostgreSQL support native `ARRAY` and `JSONB` column types, which can technically **store** multi-valued or nested data directly in a single column. This is a deliberate, pragmatic departure from strict 1NF for specific use cases (e.g., storing a flexible list of tags), but it's important to recognize this as a **conscious trade-off**, not "1NF doesn't matter anymore" — you lose easy per-value querying, indexing, and referential integrity on the individual elements unless you use specialized indexing (like PostgreSQL's GIN indexes on JSONB) to compensate.
- MySQL has historically discouraged (and in older versions, lacked native support for) array types, generally pushing designers toward the "separate table" 1NF-compliant approach shown above.
- NoSQL/document databases (MongoDB, etc.) explicitly **embrace** nested, non-atomic, repeating-group-style documents as their core data model — this is not a "1NF violation" in the relational sense because those systems don't claim to be relational databases in the first place; 1NF is a rule specifically for the **relational model**, not a universal law of all data storage.
- **Interview nuance:** if asked "does using a JSON column violate 1NF?", the strong, well-reasoned answer is: *"Strictly, from a relational-theory standpoint, yes if the JSON contains genuinely repeating/structured sub-data — but it's frequently used as a deliberate, informed trade-off for flexibility, at the cost of some query/integrity guarantees."* This shows depth rather than a rigid yes/no.

---

### 8.11 Advantages of 1NF Compliance

- Enables the entire relational algebra and FD-based reasoning toolkit (Modules 2–7) to apply meaningfully.
- Makes every individual value independently queryable, indexable, and constrainable.
- Prevents silent data truncation (the repeating-group "not enough columns" problem, Section 8.7).
- Simplifies application code — no need for fragile string-splitting or array-parsing logic to work with individual values.

### 8.12 Disadvantages / Trade-offs

- Strict 1NF decomposition can lead to **more tables and more joins** for what might feel like a simple, tightly-coupled piece of data (e.g., splitting phone numbers into their own table when a person realistically only ever has 1–2).
- In some genuinely flexible, schema-light use cases (tags, arbitrary key-value metadata), a JSON/array column can be a pragmatic, deliberate choice over rigid 1NF decomposition — as discussed in Section 8.10.

---

### 8.13 Best Practices

- Apply the **"can I meaningfully query/filter/constrain on an individual value inside this cell?"** test — if the answer is no (because it's buried inside a multi-valued cell), it's a 1NF violation for your use case.
- Prefer a genuinely separate table over "fixed repeating-group columns" (`skill_1, skill_2, ...`) even when the count seems small and stable today — business requirements evolve, and a separate table gracefully handles "what if someone needs 4 skills next year?"
- If choosing to use a DBMS's native array/JSON type as a deliberate trade-off (Section 8.10), document *why*, and ensure appropriate indexing (e.g., GIN indexes) is in place to compensate for the lost per-value queryability.

---

### 8.14 Common Mistakes

- Assuming a comma-separated string column is "fine because it's technically one VARCHAR value" — atomicity is about **logical**, not just physical/type-level, indivisibility.
- Fixing a multi-valued attribute by adding more numbered columns (`skill_1, skill_2, skill_3, skill_4, ...`) instead of creating a proper separate table — this doesn't actually solve the underlying problem (Section 8.7).
- Forgetting to re-derive the correct **key** for the newly created table during 1NF conversion (Section 8.9, Step 3) — the new table's key is often the *combination* of the original key and the new atomic attribute, not just the original key alone (since the original key alone would no longer be unique across the new table's rows).
- Treating 1NF as "just about splitting comma-separated cells" and forgetting the other rules (single consistent domain per column, no duplicate rows, row/column order not carrying meaning) — 1NF is a slightly broader concept than the popular "atomic values only" shorthand suggests.

---

### 8.15 Edge Cases

- **A single-valued attribute that LOOKS multi-valued but isn't** — e.g., a `full_address` field storing "12 MG Road, Bangalore, 560001" as one string is not automatically a 1NF violation; whether it needs splitting depends entirely on whether the business requires querying its sub-parts independently (Section 8.2's context-dependence nuance).
- **An attribute that is naturally a small, fixed-size structured value** (e.g., a `(latitude, longitude)` pair often modeled as two separate columns, or sometimes as one composite/POINT type in a spatially-aware DBMS) — this occupies a gray area; many DBMSs offer native `POINT` or geography types specifically because treating coordinates as one unit is often *more* natural for the domain, not less — a good discussion point in senior interviews about pragmatic modeling vs strict textbook 1NF.
- **JSON/array columns used purely as an opaque "blob"** the application never queries into (e.g., a `raw_import_payload` column kept only for audit/debugging purposes, never filtered or joined on) — arguably a legitimate, low-risk exception, since none of 1NF's *practical* benefits (queryability, indexing) are actually needed for that specific column's use case.

---

### 8.16 Performance Implications

- 1NF-compliant designs (separate tables for multi-valued facts) enable proper indexing on individual values — e.g., an index on `Employee_Skills.skill` supports fast "find all employees with skill X" queries, which would require a slow, unindexable string-scan (`LIKE '%SQL%'`) on a comma-separated column.
- The trade-off is additional **join cost** when reconstructing the "full picture" of an entity (e.g., an employee with all their skills) — mitigated in practice via appropriate indexing on the foreign key column in the new table.
- Array/JSONB columns (Section 8.10) can offer a middle ground — avoiding a join while still allowing indexed queries into the structure, at the cost of some referential integrity guarantees on the individual elements.

---

### 8.17 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | Define 1NF and atomicity; identify a 1NF violation in a sample table. |
| Beginner | Convert a table with a comma-separated multi-valued column into 1NF form. |
| Intermediate | Explain repeating groups as a distinct violation pattern from simple non-atomicity. |
| Senior SDE-2/SDE-3 | Discuss the trade-offs of using JSONB/array columns vs strict 1NF decomposition in a real system design scenario. |
| GATE | Identify whether a given relation instance satisfies 1NF; occasionally trick questions involving composite attributes that look non-atomic but are contextually fine. |

---

### 8.18 Chapter Summary

- **1NF** requires every attribute to hold atomic, single-domain values, with no repeating groups, and requires the relation to behave as a proper mathematical set of tuples.
- **Atomicity is context-dependent** — what counts as "atomic enough" depends on the queries the application actually needs to run.
- **Repeating groups** (numbered columns like `skill_1, skill_2, ...`) are a distinct, common violation pattern, fixed the same way as simple multi-valued cells: move to a separate table.
- 1NF is the **mathematical precondition** for every subsequent normal form and for FD-based reasoning to even make sense.
- Modern DBMS features (JSON, arrays) offer pragmatic, deliberate departures from strict 1NF for specific use cases — a valid engineering trade-off when consciously chosen.

---

### 8.19 Cheat Sheet

```
1NF REQUIRES:
  ✓ Every attribute holds a single, ATOMIC value (no lists/sets in one cell)
  ✓ Every column has a single, consistent DOMAIN/type
  ✓ NO repeating groups (no skill_1, skill_2, skill_3 style columns)
  ✓ Every row is uniquely identifiable (no exact duplicate rows — it's a SET)
  ✓ Row/column order carries no logical meaning

FIX: move multi-valued/repeating attribute → separate table,
     linked back via the original table's key as a foreign key.

Modern exception (deliberate trade-off): JSONB / ARRAY columns
  — pragmatic, but sacrifices per-value indexing/integrity unless compensated for.
```

---

### 8.20 MCQs

**Q1.** Which of the following violates 1NF?
a) A `phone_number` column storing exactly one number per row
b) A `skills` column storing `"Python, SQL, Java"` in a single cell
c) A `name` column storing text values
d) A `salary` column storing integers

<details><summary>Answer</summary>b) The `skills` column violates atomicity</details>

**Q2.** What is a "repeating group"?
a) A set of rows that are exact duplicates
b) A set of columns representing the same kind of fact, repeated a fixed number of times (e.g., skill_1, skill_2, skill_3)
c) A group of tables sharing a foreign key
d) A recursive functional dependency

<details><summary>Answer</summary>b) A set of columns representing the same repeating kind of fact</details>

**Q3.** Is atomicity an absolute, context-independent property?
a) Yes, a value is either atomic or not, regardless of the application
b) No, whether a value needs further splitting depends on the queries/business needs of the application
c) Atomicity only applies to numeric columns
d) Atomicity is determined solely by the DBMS vendor

<details><summary>Answer</summary>b) Atomicity is context-dependent</details>

**Q4.** Using a PostgreSQL `JSONB` column to store a flexible list of tags is:
a) Always strictly 1NF-compliant with no trade-offs
b) Never acceptable in any real system
c) A deliberate, pragmatic trade-off that sacrifices some per-value query/integrity guarantees
d) Required by the relational model

<details><summary>Answer</summary>c) A deliberate trade-off</details>

**Q5.** Why is 1NF considered a precondition for functional dependency theory to apply meaningfully?
a) FDs require numeric attributes only
b) FD comparison (`t1[Y] = t2[Y]`) assumes atomic, comparable values
c) FDs are unrelated to normal forms
d) 1NF has no connection to FD theory

<details><summary>Answer</summary>b) FD's tuple-comparison definition assumes atomic values</details>

---

### 8.21 Exercises

**Beginner:**
1. Define 1NF and atomicity in your own words.
2. Identify the 1NF violation in: `Book(isbn, title, authors)` where `authors` stores `"J.K. Rowling, Mary GrandPré"` in one cell. Propose a fix.

**Intermediate — Conversion Problems:**
3. Convert this table to 1NF: `Student(roll_no, name, subject_1, subject_2, subject_3)` where students take a variable number of subjects (some rows have `NULL` in unused subject columns).
4. Given `Order(order_id, customer_name, product_list)` where `product_list` stores multiple product names in one cell, convert to a 1NF-compliant design. What should the new table's key be?

**Advanced Normalization Problems:**
5. A table `Movie(movie_id, title, genres, actors)` has both `genres` and `actors` as multi-valued columns. Convert this to 1NF, and discuss whether these two multi-valued facts should go into ONE combined new table or TWO separate new tables. Justify your answer (hint: think about whether genres and actors are independent facts — this connects forward to Module 12's multivalued dependency treatment).

**Case Study:**
6. You inherit a legacy `Customer` table with columns `cust_id, name, email_1, email_2, email_3` where most customers only use `email_1` and the other two are `NULL`. Explain, as a design review comment, why this violates 1NF, what could go wrong if a customer needs a 4th email, and how you would redesign it.

**Whiteboard / Design Discussion:**
7. Discuss: should a `(latitude, longitude)` pair be modeled as two separate atomic columns, or is it acceptable to treat it as one composite/POINT value in DBMSs that support geographic types? Argue both sides, referencing Section 8.15's edge case discussion.

**GATE-style:**
8. Given a relation instance with a column containing set-valued entries (e.g., `{CSE, ECE}` for a "departments handled" column), state whether the relation is in 1NF, and justify your answer using the formal definition of atomicity.

**Interview Questions:**
- *Beginner:* "What is 1NF, and why does it matter?"
- *Intermediate:* "How would you fix a table where a column stores multiple comma-separated values?"
- *Senior:* "When, if ever, would you choose a JSONB/array column over strict 1NF decomposition in a production PostgreSQL schema — and what do you give up by doing so?"

**Practice links:**
- GeeksforGeeks — First Normal Form (1NF): https://www.geeksforgeeks.org/dbms/first-normal-form-1nf/
- GateOverflow — 1NF tagged questions: https://gateoverflow.in/

---

**End of Module 8.**

*Next up — Module 9: Second Normal Form (2NF) — eliminating partial dependencies, building directly on the "full vs partial FD" distinction from Module 2 and the candidate-key work from Module 6. Reply "next" / "continue" when ready.*
