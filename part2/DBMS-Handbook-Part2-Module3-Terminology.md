# DBMS Handbook — Part 2
## Module 3: Relational Model Terminology — Deep Dive with Multiple Examples

> **Builds on:** Module 2 (The Relational Model — where each term was formally defined once).
>
> **Purpose of this module:** Module 2 gave you the *formal definitions*. This module exists purely to **drill those definitions into fluency** through repeated, varied, real-world worked examples — across banking, e-commerce, healthcare, and social media domains — so that in an interview, you can instantly and correctly label any piece of a schema. Fluency with this vocabulary is what separates candidates who "sort of know DBMS" from those who speak it precisely.

---

## Table of Contents

1. [Why This Module Exists (Read First)](#1-why-this-module-exists-read-first)
2. [The Ten Core Terms — Quick Recap Table](#2-the-ten-core-terms--quick-recap-table)
3. [Worked Example 1: Banking System](#3-worked-example-1-banking-system)
4. [Worked Example 2: E-Commerce System](#4-worked-example-2-e-commerce-system)
5. [Worked Example 3: Healthcare System](#5-worked-example-3-healthcare-system)
6. [Worked Example 4: Social Media System](#6-worked-example-4-social-media-system)
7. [Candidate Value — A Closer Look](#7-candidate-value--a-closer-look)
8. [Terminology Confusion Matrix (Frequently Mixed-Up Pairs)](#8-terminology-confusion-matrix-frequently-mixed-up-pairs)
9. [Rapid-Fire Labeling Drill](#9-rapid-fire-labeling-drill)
10. [Chapter Summary](#10-chapter-summary)
11. [Cheat Sheet](#11-cheat-sheet)
12. [Exercises](#12-exercises)

---

## 1. Why This Module Exists (Read First)

In interviews and exams, it's extremely common to be shown a table and asked rapid-fire questions like:

- "What's the degree of this relation?"
- "Name a candidate value for this attribute."
- "Is this the schema or an instance?"

Getting these wrong — even while understanding the *underlying database design perfectly* — costs easy points and signals imprecision to interviewers. This module is **pure repetition-based mastery**: the same ten terms, applied across four completely different domains, until labeling them becomes automatic.

> 💡 **Study Tip:** Don't just read the examples below — cover the "labeled" explanation and try to answer each labeling question yourself first. Active recall beats passive reading for vocabulary fluency.

---

## 2. The Ten Core Terms — Quick Recap Table

| Term | One-Line Definition | Defined in Module 2 |
|---|---|---|
| Relation | A table — a set of tuples sharing a schema | §6 |
| Tuple | One row of a relation | §7 |
| Attribute | One column of a relation | §8 |
| Domain | The set of legal values for an attribute | §9 |
| Candidate Value | A specific, actual value an attribute takes in some tuple | §7 (new focus below) |
| Null | Represents unknown/missing/inapplicable data | §17 |
| Degree | Number of attributes in a relation | §10 |
| Cardinality | Number of tuples in a relation | §11 |
| Schema | The structural blueprint of a relation/database | §12, §14 |
| Instance | The actual current data of a relation/database | §13, §15 |

---

## 3. Worked Example 1: Banking System

### The Relation

```
Relation Name: Account

Schema: Account(account_no, holder_name, branch, balance, account_type)

Instance (current data):
+-------------+--------------+----------+---------+---------------+
| account_no  | holder_name  | branch   | balance | account_type  |
+-------------+--------------+----------+---------+---------------+
| ACC1001     | Rahul Mehta  | Andheri  | 45000   | Savings       |
| ACC1002     | Sana Iyer    | Andheri  | 120000  | Current       |
| ACC1003     | Rahul Mehta  | Bandra   | 3000    | Savings       |
| ACC1004     | Zara Khan    | Bandra   | NULL    | Savings       |
+-------------+--------------+----------+---------+---------------+
```

### Labeled Walkthrough

| Question | Answer |
|---|---|
| What is the **relation name**? | `Account` |
| What is the **relation schema**? | `Account(account_no, holder_name, branch, balance, account_type)` |
| What is the **relation instance**? | The 4 rows currently shown above |
| Name one **tuple**. | `(ACC1003, Rahul Mehta, Bandra, 3000, Savings)` |
| Name one **attribute**. | `balance` (also valid: `account_no`, `holder_name`, `branch`, `account_type`) |
| What is the **domain** of `balance`? | The set of all valid monetary amounts — typically non-negative decimals up to some system-defined maximum (business rule: often `balance >= 0`, enforced via `CHECK`, Module 5) |
| Name a **candidate value** for `branch`. | `"Andheri"` — one specific value the `branch` attribute actually holds in this instance |
| What is the **degree** of `Account`? | **5** (five attributes: account_no, holder_name, branch, balance, account_type) |
| What is the **cardinality** of this instance? | **4** (four tuples/rows) |
| Where does **NULL** appear, and what might it mean? | `ACC1004`'s `balance` is NULL — likely meaning the account was just opened and the opening deposit hasn't been recorded yet (an "unknown/not-yet-available" NULL, per Module 2 §17) |
| Is `holder_name` a good candidate for a key? | **No** — "Rahul Mehta" appears twice (ACC1001 and ACC1003), so `holder_name` doesn't uniquely identify a tuple. (Full key theory is covered in **Module 4**.) |

### Notice

Even though `holder_name` repeats, this is **not a violation of any relational property** — relations disallow *duplicate tuples* (identical across *all* attributes), not duplicate *values within a single attribute*. Rahul Mehta simply owns two different accounts — two distinct, non-identical tuples.

---

## 4. Worked Example 2: E-Commerce System

### The Relation

```
Relation Name: Product

Schema: Product(product_id, name, category, price, stock_qty)

Instance:
+-------------+------------------------+-------------+--------+-----------+
| product_id  | name                   | category    | price  | stock_qty |
+-------------+------------------------+-------------+--------+-----------+
| P501        | Wireless Mouse         | Electronics | 799.00 | 120       |
| P502        | Bluetooth Speaker      | Electronics | 2499.00| 35        |
| P503        | Cotton T-Shirt         | Apparel     | 499.00 | 200       |
| P504        | Yoga Mat               | Fitness     | 899.00 | 0         |
+-------------+------------------------+-------------+--------+-----------+
```

### Labeled Walkthrough

| Question | Answer |
|---|---|
| Degree of `Product`? | **5** |
| Cardinality of this instance? | **4** |
| A tuple representing the yoga mat? | `(P504, "Yoga Mat", "Fitness", 899.00, 0)` |
| Domain of `category`? | A finite, business-defined set of strings — e.g., `{"Electronics", "Apparel", "Fitness", "Home", "Books", ...}` — a strong candidate for an `ENUM` type or a separate `Category` reference table (foreshadowing normalization, Part 3) |
| Candidate value for `price`? | `2499.00` |
| Is `stock_qty = 0` the same as `stock_qty = NULL`? | **No** — `0` is a known, real value meaning "out of stock but tracked"; `NULL` would mean "we don't know the stock level," an entirely different business meaning (Module 2, §17) |
| If we `ALTER TABLE Product ADD COLUMN rating DECIMAL(2,1);`, what changes? | The **relation schema** changes (degree becomes 6); a **new relation instance** immediately exists too, since every existing tuple now implicitly has `rating = NULL` |

### Deep Insight: Schema Change Ripple Effect

This example demonstrates a subtle but important idea: a schema change (`ALTER TABLE`) doesn't just change the *schema* — it instantaneously produces a **new instance**, because every existing tuple must now conform to the new schema (typically by getting `NULL` in the new column). This is why schema migrations in production systems must be handled carefully — they affect **every existing row simultaneously**.

---

## 5. Worked Example 3: Healthcare System

### The Relation

```
Relation Name: Patient

Schema: Patient(patient_id, full_name, dob, blood_group, allergy_notes)

Instance:
+-------------+---------------+------------+--------------+------------------+
| patient_id  | full_name     | dob        | blood_group  | allergy_notes    |
+-------------+---------------+------------+--------------+------------------+
| PT2001      | Meena Rao     | 1990-04-12 | O+           | NULL             |
| PT2002      | Arjun Sethi   | 1985-11-02 | B+           | "Penicillin"     |
| PT2003      | Meena Rao     | 1990-04-12 | O+           | NULL             |
+-------------+---------------+------------+--------------+------------------+
```

### Labeled Walkthrough — With a Deliberate Trick

| Question | Answer |
|---|---|
| Degree? | **5** |
| Cardinality? | **3** |
| Are the tuples for `PT2001` and `PT2003` duplicate tuples (violating relational theory)? | **No** — even though *every non-key attribute value* is identical, the `patient_id` values (`PT2001` vs `PT2003`) differ, so these are two **distinct tuples**. This could represent identical twins, or (in real life) a data-entry duplicate-patient-record bug — an important real-world data quality issue |
| What does `allergy_notes = NULL` for `PT2001` most plausibly mean? | "Not applicable" or "no known allergies" — **not** "unknown," ideally; but note the crucial real-world ambiguity: without a separate boolean flag like `allergies_checked`, NULL here is dangerously ambiguous between *"we haven't asked"* and *"we asked, and there are none"* — a real patient-safety-relevant modeling flaw! |
| Suggest a schema fix for the ambiguity above. | Split into two attributes: `allergies_screened (BOOLEAN)` and `allergy_notes (TEXT, nullable)` — now NULL unambiguously means "not yet screened" only when `allergies_screened = FALSE` |

### Why This Example Matters

This worked example demonstrates that **terminology precision has real consequences** — the NULL ambiguity here isn't just academic; in a real hospital system, confusing "no known allergies" with "we never checked" could be dangerous. Interviewers at senior levels sometimes probe exactly this kind of NULL-semantics reasoning in schema design discussions.

---

## 6. Worked Example 4: Social Media System

### The Relation

```
Relation Name: Post

Schema: Post(post_id, author_id, content, likes_count, posted_at)

Instance:
+-----------+-------------+--------------------------+-------------+---------------------+
| post_id   | author_id   | content                  | likes_count | posted_at           |
+-----------+-------------+--------------------------+-------------+---------------------+
| PST9001   | U100        | "Loving this weather!"   | 245         | 2026-07-01 09:15:00 |
| PST9002   | U101        | "New blog post is live"  | 12          | 2026-07-02 14:30:00 |
| PST9003   | U100        | "Coffee time ☕"          | 88          | 2026-07-03 08:05:00 |
+-----------+-------------+--------------------------+-------------+---------------------+
```

### Labeled Walkthrough

| Question | Answer |
|---|---|
| Degree? | **5** |
| Cardinality? | **3** |
| Domain of `likes_count`? | Non-negative integers, `{0, 1, 2, 3, ...}` — a real-world `CHECK (likes_count >= 0)` constraint candidate (Module 5) |
| Is `author_id` here a foreign key? | Very likely yes — it almost certainly references a `User(user_id)` relation elsewhere in the database schema (foreign keys formally covered in **Module 4**) |
| Candidate value for `content`? | `"Coffee time ☕"` |
| Two tuples share the same `author_id` (U100) — is this a problem? | **No** — one author can have many posts; this is completely normal and expected (this is a preview of the **One-to-Many** relationship type, formally covered in **Module 10**) |

---

## 7. Candidate Value — A Closer Look

### Definition Recap

A **candidate value** is simply **any specific, legal value that an attribute actually holds (or could hold) in some tuple** — it is a concrete instance drawn from the attribute's domain.

### Why "Candidate Value" Is a Distinct, Useful Term

It's important not to confuse:

| Term | Scope | Example |
|---|---|---|
| **Domain** | *All possible* legal values (the full set) | Domain of `blood_group` = `{"O+", "O-", "A+", "A-", "B+", "B-", "AB+", "AB-"}` |
| **Candidate Value** | *One specific* value actually appearing (or being considered) in a tuple | `"O+"` — one particular candidate value taken from that domain |

### Analogy

If a domain is the entire multiple-choice answer sheet ("choose A, B, C, or D"), a candidate value is the **one answer a specific student actually bubbled in**.

> ⚠️ **Do not confuse "candidate value" with "candidate key"** (Module 4) — these are **completely unrelated concepts** that happen to share the word "candidate." A candidate value is about a single attribute's data; a candidate key is about a *set of attributes* that could uniquely identify a tuple. This confusion is extremely common among students and is a favorite "gotcha" in oral exams.

---

## 8. Terminology Confusion Matrix (Frequently Mixed-Up Pairs)

This table consolidates every "same word, different meaning" and "different word, sounds similar" trap encountered so far — extremely high interview/exam value.

| Confusing Pair | Term 1 Meaning | Term 2 Meaning | How to Never Mix Them Up Again |
|---|---|---|---|
| Degree vs. Cardinality | # of **columns** | # of **rows** | "**D**egree = **D**own the top (header row, columns)"; "Cardinality — think 'cardinal number' = **counting rows**" |
| Degree of Relation vs. Degree of Relationship | # of attributes in a table | # of entity types in an ER relationship (unary/binary/ternary) | Ask: "table or ER diagram?" — context disambiguates |
| Cardinality (Relation) vs. Cardinality Constraint (ER) | # of rows in a table | The 1:1 / 1:N / M:N type of a relationship | Ask: "counting rows, or describing a relationship type?" |
| Candidate Value vs. Candidate Key | One actual data value | A minimal set of attributes that could uniquely identify a row | "Value" = data; "Key" = structural/identifying concept (Module 4) |
| Schema vs. Instance | The structure/blueprint | The actual current data | "Schema is the **shape**, Instance is the **stuff**" |
| Relation vs. Relationship | A table (mathematical set of tuples) | An association between two entities in ER modeling (Module 9) | "Relation = table. Relationship = **link between** tables/entities." |
| Table vs. File | DBMS-managed, constraint-enforced structure | Raw, unmanaged byte storage | "A table has a **DBMS watching over it**; a file does not." |
| Domain vs. Data Type | Semantically valid value set (business-defined) | Storage-level type category (system-defined) | "Data type is the **container size**; Domain is the **actual allowed contents**." |

---

## 9. Rapid-Fire Labeling Drill

Use this relation for a final self-test. Try to answer every question **before** checking the answer key at the end of this module (in the Exercises' answer key).

```
Relation Name: Flight

Schema: Flight(flight_no, origin, destination, departure_time, seats_available)

Instance:
+-------------+----------+--------------+------------------+------------------+
| flight_no   | origin   | destination  | departure_time   | seats_available  |
+-------------+----------+--------------+------------------+------------------+
| AI202       | DEL      | BOM          | 2026-08-01 06:00 | 12               |
| AI305       | BOM      | BLR          | 2026-08-01 09:30 | 0                |
| AI202       | DEL      | BLR          | 2026-08-02 06:00 | 45               |
| 6E118       | DEL      | BOM          | 2026-08-01 14:00 | NULL             |
+-------------+----------+--------------+------------------+------------------+
```

**Questions:**

1. What is the degree of `Flight`?
2. What is the cardinality of this instance?
3. Is `flight_no` alone sufficient to uniquely identify a tuple? (Look carefully at rows 1 and 3.)
4. Give one candidate value for `destination`.
5. What might `seats_available = NULL` mean for `6E118`, versus `seats_available = 0` for `AI305`? Explain the business-meaning difference.
6. Name one full tuple from this instance.
7. If a new column `aircraft_type` is added via `ALTER TABLE`, what two things change simultaneously?

*(Answers appear in the Exercises section's answer key, Q11–17.)*

---

## 10. Chapter Summary

- All ten core relational-model terms (relation, tuple, attribute, domain, candidate value, null, degree, cardinality, schema, instance) were re-applied across **four different real-world domains** — banking, e-commerce, healthcare, and social media — to build labeling fluency.
- **Candidate value** (a specific data value) is a distinct and often-confused concept from **candidate key** (Module 4) — same word "candidate," entirely different meaning.
- Real-world worked examples revealed practical nuance beyond bare definitions: e.g., duplicate *values* in one attribute (like `holder_name`) is normal and expected, while duplicate *entire tuples* is a genuine relational-theory violation (or a real data-quality bug, as in the healthcare twin-patient example).
- NULL's meaning is **context-dependent and business-critical** — the healthcare example showed how ambiguous NULL semantics can create real patient-safety risks, reinforcing that precise data modeling has real-world consequences, not just academic ones.
- A consolidated **Confusion Matrix** (§8) captures every commonly mixed-up term pair for fast pre-interview revision.

---

## 11. Cheat Sheet

```
FOR ANY GIVEN TABLE, ASK YOURSELF:
  "How many COLUMNS?"        → Degree
  "How many ROWS?"           → Cardinality
  "What does the STRUCTURE look like (unfilled)?"  → Schema
  "What does the DATA look like RIGHT NOW?"        → Instance
  "One full row?"            → Tuple
  "One column?"              → Attribute
  "All legal values for a column?"  → Domain
  "One actual value in a cell?"     → Candidate Value
  "Missing/unknown/N-A marker?"     → NULL

NEVER CONFUSE:
  Candidate VALUE  ≠  Candidate KEY   (data vs. structural identifier — Module 4)
  Degree (relation)   ≠  Degree (ER relationship)
  Cardinality (relation)  ≠  Cardinality CONSTRAINT (ER relationship type)
  Relation  ≠  Relationship
  Domain  ≠  Data Type
```

---

## 12. Exercises

### Beginner

1. For the `Product` relation in §4, what is the domain of `stock_qty`?
2. True/False: Two tuples can have identical values in every attribute *except* the primary key, and still be perfectly valid, distinct tuples.
3. Fill in the blank: A __________ is one specific, actual value taken from an attribute's domain, while a __________ is the full set of values that attribute is allowed to take.

### Intermediate

4. In the `Patient` example (§5), redesign the schema to remove the NULL ambiguity around allergy screening. Write out the new relation schema.
5. Explain, using the `Flight` example (§9), why `flight_no` alone is a poor candidate for uniquely identifying a tuple, and suggest what combination of attributes might work better. (This foreshadows **composite keys**, Module 4.)
6. In the e-commerce example (§4), explain precisely why adding a column via `ALTER TABLE` changes both the schema *and* produces a new instance simultaneously.

### Advanced / Conceptual

7. Design your own relation (any domain of your choice, minimum degree 4), populate it with a 4-row instance including at least one NULL, and write out the full labeled walkthrough (degree, cardinality, one tuple, one attribute, domain of one attribute, one candidate value, and the business meaning of your NULL) exactly in the style of §3–§6.
8. Explain why "candidate value" and "candidate key" are dangerously easy to confuse verbally in a spoken interview, and how you would clarify which one you mean if an interviewer used the ambiguous word "candidate" alone.

### MCQs

**Q1.** In the `Account` example (§3), what is the cardinality of the shown instance?
a) 5  b) 4  c) 3  d) 20

**Q2.** "O+" in the `Patient` example is best described as a:
a) Domain  b) Candidate value  c) Schema  d) Degree

**Q3.** Which pair of terms is genuinely unrelated despite sharing a word?
a) Schema / Instance  b) Candidate Value / Candidate Key  c) Relation / Tuple  d) Domain / Data Type

**Q4.** If `ALTER TABLE Product ADD COLUMN rating DECIMAL;` is run, the degree of `Product`:
a) Stays the same  b) Increases by 1  c) Decreases by 1  d) Becomes undefined

### Answer Key (MCQs + Rapid-Fire Drill from §9)

- MCQ: 1-b, 2-b, 3-b, 4-b
- §9 Q1 (degree): **5**
- §9 Q2 (cardinality): **4**
- §9 Q3: **No** — `flight_no = AI202` appears in both row 1 and row 3 with *different* destinations/dates, so `flight_no` alone cannot uniquely identify a tuple; a combination like `(flight_no, departure_time)` would be a stronger candidate (previewing composite keys, Module 4).
- §9 Q4: e.g., `"BOM"`, `"BLR"`, or `"DEL"` — any actual value from the `destination` column.
- §9 Q5: `NULL` for `6E118` likely means "seat availability data hasn't been synced/loaded yet" (unknown); `0` for `AI305` means "confirmed fully booked" (known, real value) — a critical business distinction for a booking system.
- §9 Q6: e.g., `(AI202, DEL, BOM, 2026-08-01 06:00, 12)`.
- §9 Q7: The **relation schema** (degree becomes 6) AND a **new relation instance** (every existing row now has `aircraft_type = NULL` by default) change simultaneously.

### Practice Links

- [HackerRank SQL — Basic Select (practice reading/labeling real table data)](https://www.hackerrank.com/domains/sql)
- [SQLBolt — Lesson 1: SELECT queries (practice identifying schema vs. instance visually)](https://sqlbolt.com/)
- *(This module is vocabulary/labeling-focused — the richest practice is re-doing §3–§6's labeled walkthroughs on schemas of your own design.)*

---

**Next: Module 4 — Keys (Complete): Super Key, Candidate Key, Primary Key, Alternate Key, Composite Key, Compound Key, Foreign Key, Natural Key, Surrogate Key, Secondary Key, Unique Key, Partial Key, Business Key.**

> Say "continue" or "next module" to proceed to Module 4.
