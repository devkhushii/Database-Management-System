# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 7: Normalization

> **Where we are:** Modules 2–6 built the complete mathematical toolkit — FDs, closure, axioms, canonical cover, and systematic key-finding. Starting with this module, we put that toolkit to work. Module 7 formally proves *why* normalization is needed by rigorously defining the anomalies that were only informally previewed in Module 1. Modules 8–12 then give you the actual normal forms (1NF through 5NF/DKNF), each one solving a specific class of these anomalies.

---

### 7.1 What Is Normalization?

**Definition:**
**Normalization** is the systematic process of organizing the attributes and tables of a relational database to **minimize data redundancy** and **eliminate undesirable characteristics** — specifically, insertion, update, and deletion anomalies — by decomposing relations according to their functional dependencies, guided by a series of increasingly strict rules called **normal forms**.

**Why does it exist?**
Because, left unchecked, a "naturally" designed table (especially one built bottom-up from a flat spreadsheet or a hastily-designed form) tends to bundle together **multiple independent facts** into a single relation. This bundling is the root cause of nearly every practical data-integrity problem a backend engineer encounters.

**The core insight normalization is built on:**
> **A relation should represent ONE cohesive concept, and every non-key attribute should depend on the WHOLE key, and NOTHING BUT the key.**

This single sentence (often taught as the mnemonic *"the key, the whole key, and nothing but the key, so help me Codd"* — a pun on E.F. Codd, the inventor of the relational model) is the intuitive summary of what Modules 8–11 (1NF through BCNF) formalize precisely.

---

### 7.2 Why Normalize? — The Driving Motivation

Let's build the motivating example that Modules 8–11 will keep returning to, and use it to concretely demonstrate each anomaly.

**A poorly designed, unnormalized relation:**

```
Table: StudentCourse
+-----------+--------------+---------+-------------+---------------+-------+
| student_id| student_name | course  | instructor  | instructor_office | grade |
+-----------+--------------+---------+-------------+---------------+-------+
| S1        | Aditi        | DBMS    | Dr. Rao     | Room 204      | A     |
| S1        | Aditi        | OS      | Dr. Iyer    | Room 310      | B+    |
| S2        | Rohan        | DBMS    | Dr. Rao     | Room 204      | A-    |
| S3        | Meera        | DBMS    | Dr. Rao     | Room 204      | B     |
+-----------+--------------+---------+-------------+---------------+-------+
```

This single table bundles together **at least three independent concepts**: student information, course/instructor information, and enrollment/grade information. Let's see exactly what goes wrong.

---

### 7.3 Data Redundancy

**What is it?**
**Data redundancy** is the unnecessary, repeated storage of the same fact in multiple rows of a table.

**Where it appears above:**
- `student_name` "Aditi" is stored **twice** (once per course she's enrolled in).
- `instructor` "Dr. Rao" and `instructor_office` "Room 204" are stored **three times** (once per student enrolled in DBMS).

**Why is this bad?**
- **Wasted storage:** trivial for 4 rows, but at production scale (millions of enrollments), redundant instructor office data multiplied across every enrollment row adds up to meaningful, unnecessary storage cost.
- **Root cause of every other anomaly below:** redundancy isn't just wasteful — it's the *mechanism* by which update, insert, and delete anomalies become possible in the first place. If a fact is stored in only one place, none of these anomalies can occur for that fact.

---

### 7.4 Update Anomaly

**Definition:**
An **update anomaly** occurs when updating one occurrence of a redundantly-stored fact requires updating **multiple rows** to keep the data consistent — and if even one row is missed, the database ends up in a **logically inconsistent state** (the same real-world fact has two different values in the database simultaneously).

**Concrete example (from Section 7.2's table):**
Suppose Dr. Rao moves offices, from Room 204 to Room 410. This fact needs to be updated in **every row** where `instructor = 'Dr. Rao'` — that's rows for S1, S2, and S3 above (and potentially hundreds more in a real system).

```sql
-- If we forget even ONE row:
UPDATE StudentCourse SET instructor_office = 'Room 410'
WHERE student_id IN ('S1', 'S2')   -- oops, forgot S3!
```

Now the database claims Dr. Rao's office is **both** Room 410 (for S1 and S2's rows) **and** Room 204 (for S3's row) — simultaneously. This is a genuine logical contradiction stored in the database, and nothing in the schema itself prevented it.

**Why does this happen?**
Because `instructor_office` (a fact about the *instructor*) is stored redundantly inside a table whose real grain is *enrollment* (student + course), not *instructor*. The fact is in the "wrong place" — bundled with data it doesn't functionally belong with **at that granularity**.

---

### 7.5 Insertion Anomaly

**Definition:**
An **insertion anomaly** occurs when it's **impossible to add a valid, meaningful fact** to the database without also being forced to add unrelated, possibly not-yet-available data — because the schema has bundled unrelated concepts together.

**Concrete example:**
Suppose the university wants to add a new course, "Machine Learning," taught by a newly hired instructor, Dr. Sharma — but **no student has enrolled yet**.

Looking at the `StudentCourse` table's structure, every row **requires** a `student_id` and `student_name` (they're presumably part of what makes a row meaningful, and might even be part of the key). There is **no way to insert the fact "Machine Learning is taught by Dr. Sharma"** into this table without also inventing a fake student enrollment — which is either impossible (if `student_id` is a foreign key to a real, validated Students table) or a data-quality violation (a fake enrollment row polluting real data).

**Why does this happen?**
Because "which instructor teaches which course" is a fact independent of "which students are enrolled" — but the schema has forced these two independent facts to live in the same row, so you can't record one without the other.

---

### 7.6 Deletion Anomaly

**Definition:**
A **deletion anomaly** occurs when deleting a row to remove one fact **unintentionally destroys other, unrelated facts** that happened to be stored in the same row — because, again, the schema has bundled independent concepts together.

**Concrete example:**
Suppose student S2 (Rohan) drops the "DBMS" course — his only enrollment. If we delete his row:

```sql
DELETE FROM StudentCourse WHERE student_id = 'S2' AND course = 'DBMS';
```

This was the **only** row containing Rohan's enrollment. Deleting it doesn't just remove the fact "Rohan is enrolled in DBMS" — if this row also happened to be the *only* place where some other information was recorded (in a more extreme version of this example, imagine it were also the only row mentioning a particular instructor for a particular course that no one else had enrolled in yet), that unrelated information would be silently destroyed too.

A cleaner illustration: imagine "OS" course had only ever had student S1 enrolled, and S1 drops OS. Deleting that row wouldn't just remove S1's enrollment in OS — it would **completely erase the fact that "Dr. Iyer teaches OS, in Room 310"** from the entire database, even though that's a true, independent fact that should have nothing to do with whether any particular student is currently enrolled.

**Why does this happen?**
Because "course + instructor + office" information has no independent existence in this schema — it only exists *as a side effect* of an enrollment row existing. When the last enrollment row referencing a given course disappears, that course's instructor information vanishes with it, even though logically it shouldn't.

---

### 7.7 Anomalies — Side-by-Side Comparison Table

| Anomaly | Triggered By | What Goes Wrong | Root Cause |
|---|---|---|---|
| **Update Anomaly** | Modifying a redundantly-stored fact | Inconsistent data if not ALL copies are updated | Same fact stored in multiple rows |
| **Insertion Anomaly** | Trying to add a new, valid fact | Forced to also add unrelated/unavailable data, or can't insert at all | Independent facts bundled into one row |
| **Deletion Anomaly** | Removing one fact (e.g., last enrollment) | Unintentionally destroys other, unrelated facts | A fact's existence is falsely tied to another fact's existence |

**Visual — Root Cause Diagram:**

```
                    ┌─────────────────────────────────┐
                    │   MULTIPLE INDEPENDENT FACTS      │
                    │   crammed into ONE relation        │
                    │  (Student info + Course/Instructor │
                    │        info + Enrollment info)      │
                    └─────────────────┬─────────────────┘
                                     │
                                     ▼
                    ┌─────────────────────────────────┐
                    │         DATA REDUNDANCY            │
                    │ (same fact repeated across rows)   │
                    └─────────────────┬─────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                      ▼
      ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
      │ UPDATE ANOMALY │    │INSERT ANOMALY │    │DELETE ANOMALY │
      └───────────────┘    └───────────────┘    └───────────────┘
```

---

### 7.8 Data Integrity — The Bigger Picture

**What is data integrity?**
**Data integrity** is the overall correctness, consistency, and reliability of data stored in a database over time — the guarantee that the database accurately reflects the real-world facts it's meant to represent, without contradictions.

**How anomalies threaten it:**
Every anomaly above is fundamentally a **data integrity failure waiting to happen**:
- Update anomalies threaten **consistency** (the same fact having two different values).
- Insertion anomalies threaten **completeness** (valid facts that legitimately exist can't be recorded).
- Deletion anomalies threaten **durability of independent facts** (facts vanishing as an unintended side effect of unrelated deletions).

**Why normalization is the primary defense:**
Normalization directly attacks the **root cause** (bundled independent facts / redundancy) rather than trying to patch each anomaly symptom individually with application-level safeguards (like "remember to update all rows" — which is fragile and error-prone, exactly the kind of thing that should be structurally impossible rather than a matter of remembering).

---

### 7.9 The Fix — A Preview

Let's briefly preview how proper decomposition (which Modules 8–13 formalize rigorously) fixes all three anomalies at once, using our running example:

```
Students(student_id, student_name)
Courses(course, instructor, instructor_office)
Enrollments(student_id, course, grade)
```

- **Update anomaly fixed:** Dr. Rao's office is now stored in **exactly one row** in `Courses` — updating it once is sufficient and there's no possibility of inconsistency.
- **Insertion anomaly fixed:** "Machine Learning, taught by Dr. Sharma" can be inserted directly into `Courses`, with zero students enrolled yet — no fake enrollment needed.
- **Deletion anomaly fixed:** Deleting Rohan's `DBMS` row from `Enrollments` has zero effect on the `Courses` table — the fact "Dr. Rao teaches DBMS in Room 204" persists independently.

This decomposition is not arbitrary — it's derived precisely by identifying the functional dependencies at play (`student_id → student_name`; `course → instructor`; `instructor → instructor_office` — actually revealing a transitive dependency, Module 2 Section 2.10!; `{student_id, course} → grade`) and applying the normal form rules from Modules 8–11 systematically.

---

### 7.10 Industry Implementation Notes

- Anomalies aren't just an academic concern — they're a **daily reality** in poorly-designed production systems. A classic real-world example: an e-commerce `orders` table storing `customer_name` and `customer_address` directly on every order row (rather than referencing a `customers` table) will exhibit exactly the update anomaly described above whenever a customer's address changes and historical orders need to reflect it (or, worse, when they *shouldn't* — see Module 15's discussion of intentional historical snapshotting, which is a deliberate exception, not an accidental anomaly).
- Modern ORMs (Object-Relational Mappers) and schema migration tools don't automatically prevent anomalies — the underlying schema design must still be done correctly; the tooling just makes *applying* a well-normalized schema more convenient.
- Database code review at senior/staff engineering levels frequently involves explicitly asking: *"if I update/insert/delete X, what unintended side effects could ripple through this schema?"* — which is exactly anomaly-analysis, informally applied.

---

### 7.11 Advantages of Normalization

- Eliminates (or drastically reduces) update, insertion, and deletion anomalies by construction, rather than relying on application-level discipline.
- Reduces storage waste from redundant data.
- Improves data consistency — each fact has one authoritative location.
- Makes the schema easier to evolve — a normalized schema's tables tend to map cleanly to independent real-world concepts, so adding a new concept is usually just adding a new table, not restructuring existing ones.

### 7.12 Disadvantages of (Over-)Normalization

- More tables generally means more **joins** are needed to reconstruct a full picture (e.g., "show me a student's name alongside their grade and instructor" now requires a 3-table join) — a real performance cost, especially at read-heavy scale.
- Highly normalized schemas can be harder for newcomers or analysts to query intuitively, since related information is spread across many tables.
- In some read-heavy, write-rarely scenarios, a controlled amount of redundancy (denormalization, Module 14) is a legitimate, deliberate engineering trade-off rather than a design mistake — this is the essential tension previewed in Module 1's design trade-offs and addressed head-on in Module 14.

---

### 7.13 Best Practices

- Always identify functional dependencies **first** (Module 2), then let them drive the normalization decisions — don't normalize "by feel" or by guessing which tables "look right."
- When reviewing an existing schema for anomaly risk, explicitly trace through: "if I update/insert/delete a single row here, what real-world facts could become inconsistent or lost?"
- Treat normalization as a **means to an end** (data integrity), not a goal in itself — over-normalizing tables that will never realistically face these anomalies (e.g., a rarely-changing configuration table) adds needless complexity.
- Document *why* a table is structured the way it is (which FDs drove the decomposition) — this prevents future engineers from "helpfully" re-introducing redundancy without realizing why it was avoided.

---

### 7.14 Common Mistakes

- Assuming redundancy is always bad — sometimes controlled redundancy (via deliberate denormalization, Module 14) is the *correct* engineering trade-off for a specific, well-understood access pattern; normalization should be applied thoughtfully, not dogmatically.
- Confusing an anomaly with a simple bug — anomalies are **structural** problems (built into the schema's shape), not implementation bugs that can be fixed by "writing more careful application code." The fix is schema-level, not code-level.
- Forgetting that all three anomaly types typically stem from the **same underlying redundancy** — fixing the redundancy (via correct normalization) fixes all three simultaneously; there's no need for three separate "fixes."
- Trying to reason about anomalies without first identifying the FDs — anomaly analysis without an explicit FD understanding tends to be superficial and can miss subtler cases (like the "OS course disappears" example in Section 7.6, which is easy to overlook without formally tracing the dependency).

---

### 7.15 Edge Cases

- **A table with genuinely no anomaly risk:** A table representing a single, indivisible concept with no composite/multi-valued structure (e.g., a `Countries(country_code, country_name)` reference table) may already be in a high normal form "by accident," with nothing further to normalize — not every table needs decomposition.
- **Anomalies that only manifest rarely:** The "last enrollment deleted" deletion anomaly (Section 7.6) might not manifest for years in a growing system (courses rarely have zero remaining enrollments) — but the *risk* is structurally present the whole time, and will eventually surface, often at an inconvenient moment (e.g., end-of-semester mass drops).
- **Intentional denormalization that reintroduces anomaly risk:** A deliberately denormalized reporting table (Module 14) reintroduces these exact anomalies by design — this is acceptable **only** when the table is understood to be a derived, regularly-rebuilt snapshot, not a source of truth that receives ad hoc updates.

---

### 7.16 Performance Implications

- Anomalies themselves don't directly cause performance problems — but the **redundancy** that causes them does inflate row width and total storage, which can indirectly slow down full table scans and increase I/O.
- The *fix* for anomalies (normalization, splitting into more tables) trades this storage/consistency cost for a **join cost** at query time — this is the central trade-off explored fully in Modules 14 and 17.

---

### 7.17 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | Define update, insertion, and deletion anomalies, each with a simple example. |
| Beginner | Explain why redundancy causes anomalies. |
| Intermediate | Given a flat, unnormalized table, identify a concrete instance of each of the three anomalies. |
| Senior SDE-2/SDE-3 | Given a real production schema with reported data-integrity bugs, diagnose which anomaly is responsible and propose a normalized redesign, justified by explicit functional dependencies. |
| GATE | "Which of the following is an example of an insertion anomaly?" style multiple-choice/scenario questions are extremely common. |

---

### 7.18 Chapter Summary

- **Normalization** organizes relations to minimize redundancy and eliminate anomalies, guided by functional dependencies.
- **Data redundancy** (a fact stored in more than one place) is the root cause of all three classic anomalies.
- **Update anomaly:** a redundantly stored fact can become inconsistent if not every copy is updated.
- **Insertion anomaly:** a valid fact can't be recorded without also recording unrelated, possibly unavailable data.
- **Deletion anomaly:** deleting one fact can unintentionally destroy an unrelated fact that had no independent storage.
- The fix is to **decompose** relations so each table represents one cohesive concept, guided by the FDs — which is exactly what Modules 8–13 formalize step by step.

---

### 7.19 Cheat Sheet

```
REDUNDANCY → root cause of all anomalies

UPDATE ANOMALY    : same fact stored multiple times; update ONE, others go stale/inconsistent
INSERTION ANOMALY  : can't add a valid fact without also adding unrelated/unavailable data
DELETION ANOMALY   : deleting one fact accidentally destroys an unrelated, independently-true fact

FIX: decompose the relation so each table = ONE cohesive concept,
     driven by functional dependencies (Modules 8–13)

MNEMONIC: "The key, the whole key, and nothing but the key" (informal summary of 2NF+3NF)
```

---

### 7.20 MCQs

**Q1.** Which anomaly occurs when you cannot record a new, valid fact without also being forced to record unrelated data?
a) Update anomaly
b) Insertion anomaly
c) Deletion anomaly
d) Redundancy anomaly

<details><summary>Answer</summary>b) Insertion anomaly</details>

**Q2.** What is the ROOT CAUSE common to all three classic anomalies?
a) Missing primary keys
b) Data redundancy from bundling independent facts
c) Too many tables
d) Lack of indexes

<details><summary>Answer</summary>b) Data redundancy from bundling independent facts</details>

**Q3.** Deleting the only enrollment row for a course, which also happens to be the only place the course's instructor is recorded, and thereby losing the instructor information — is an example of:
a) Update anomaly
b) Insertion anomaly
c) Deletion anomaly
d) Referential integrity violation

<details><summary>Answer</summary>c) Deletion anomaly</details>

**Q4.** Is a properly normalized schema completely immune to ALL forms of data inconsistency (e.g., application bugs)?
a) Yes, normalization guarantees complete immunity to any inconsistency
b) No, normalization eliminates STRUCTURAL redundancy-driven anomalies, but application-level bugs can still cause other issues
c) Normalization only matters for performance, not correctness
d) Normalization is irrelevant to data integrity

<details><summary>Answer</summary>b) Normalization eliminates structural anomalies; it isn't a substitute for correct application logic elsewhere</details>

**Q5.** Deliberately reintroducing redundancy in a reporting table for performance reasons is called:
a) Normalization
b) Denormalization
c) Decomposition
d) Canonicalization

<details><summary>Answer</summary>b) Denormalization</details>

---

### 7.21 Exercises

**Beginner:**
1. Define update, insertion, and deletion anomalies in your own words, with one original example each (different from the StudentCourse example used in this module).
2. Explain, in one or two sentences, why data redundancy is the common root cause of all three anomalies.

**Intermediate — Anomaly Identification:**
3. Given a flat `Employee_Project` table with columns `emp_id, emp_name, project_id, project_name, project_budget, hours_worked`, identify a concrete scenario for each of the three anomalies.
4. For the same table, write out the functional dependencies you believe hold (based on typical business rules), and explain which FD is the direct cause of the redundancy driving the anomalies.

**Advanced — Diagnostic Practice:**
5. A production incident report states: "Customer support updated a customer's shipping address for one order, but three other historical orders still show the old address, causing a shipping mix-up." Diagnose which anomaly this is, and explain (at a high level, without yet applying the formal normal-form rules from later modules) what schema change would resolve it.
6. Explain why a table can be "anomaly-free by accident" even without formally going through normalization — give an example of such a table.

**Case Study:**
7. You're reviewing a startup's `Bookings` table for a hotel-booking platform, which stores `booking_id, guest_name, guest_email, hotel_name, hotel_address, room_type, check_in_date, check_out_date, price`. Walk through all three anomaly types with concrete scenarios specific to this table, and explain (informally, in plain language) how you would split it into multiple tables to fix them.

**Whiteboard Design:**
8. Design (informally, on paper — full formal decomposition comes in Module 13) a corrected version of the `StudentCourse` table from Section 7.2 that avoids all three anomalies. Justify each table boundary using the functional dependencies you can identify.

**Interview Questions:**
- *Beginner:* "What's the difference between an update anomaly and a deletion anomaly?"
- *Intermediate:* "Given this flat table [interviewer provides one], find an example of each anomaly."
- *Senior:* "Tell me about a real data-integrity bug you've encountered or can imagine, and diagnose whether it was rooted in a schema design anomaly or an application-level bug."

**Practice links:**
- GeeksforGeeks — Anomalies in DBMS: https://www.geeksforgeeks.org/dbms/anomalies-in-dbms/
- GateOverflow — Anomalies / Normalization tagged questions: https://gateoverflow.in/

---

**End of Module 7.**

*Next up — Module 8: First Normal Form (1NF) — the foundational rule of atomicity that every relation must satisfy before any other normal form even applies. Reply "next" / "continue" when ready.*
