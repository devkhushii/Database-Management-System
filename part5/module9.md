# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 9: Second Normal Form (2NF)

> **Where we are:** Module 8 established 1NF as the atomicity foundation. Now we climb one step: 2NF directly targets the **partial dependency** concept from Module 2 (Section 2.8), using the **candidate key** and **prime/non-prime attribute** machinery from Module 6. If a relation has only a single-attribute candidate key, 2NF is automatically satisfied (as we'll see) — 2NF only becomes a meaningful question when composite candidate keys are involved.

---

### 9.1 What Is Second Normal Form (2NF)?

**Definition:**
A relation is in **Second Normal Form (2NF)** if and only if:
1. It is already in **1NF** (Module 8), **and**
2. Every **non-prime attribute** is **fully functionally dependent** on **every candidate key** — i.e., there is **no partial dependency** of any non-prime attribute on any candidate key.

Recall from Module 2 (Section 2.8): a partial dependency exists when a non-prime attribute depends on only *part* of a composite candidate key, rather than the whole key.

**Why does it exist?**
Because partial dependencies are a direct, provable **source of the update/insertion/deletion anomalies** from Module 7 — specifically, they cause a non-prime attribute (like a student's name) to be redundantly repeated across every row that shares the same "partial key" value (like the student's ID, repeated across every course they've enrolled in).

**What problem does it solve?**
2NF eliminates redundancy caused by attributes that are actually properties of just **one part** of a composite key, rather than the whole entity the key represents.

---

### 9.2 When Is 2NF Automatically Satisfied?

**Important rule:** If a relation's **only candidate key is a single attribute** (not composite), then the relation is **automatically in 2NF** (assuming it's already in 1NF) — because a partial dependency, by definition (Module 2, Section 2.8), requires a *proper subset* of the determinant, and a single-attribute key has no proper, non-empty subset to depend on partially.

**This means:** 2NF violations can **only** occur in relations with **composite candidate keys**. This is worth internalizing early, since it immediately tells you where to look (and where NOT to bother looking) when checking a schema for 2NF compliance.

---

### 9.3 The Classic 2NF Violation — Worked Example

**Given (in 1NF, but NOT yet checked for 2NF):**

```
Table: Enrollment(student_id, course_id, student_name, instructor, grade)

Candidate Key: {student_id, course_id}

FDs:
  {student_id, course_id} → grade        (a student's grade in a specific course)
  student_id → student_name              (a student's name depends only on student_id)
  course_id → instructor                 (a course's instructor depends only on course_id)
```

**Checking each non-prime attribute:**

| Non-Prime Attribute | Depends On | Full or Partial? |
|---|---|---|
| `grade` | `{student_id, course_id}` (the whole key) | **Full** — depends on the entire composite key |
| `student_name` | `student_id` alone (a *proper subset* of the key) | **Partial** — violates 2NF! |
| `instructor` | `course_id` alone (a *proper subset* of the key) | **Partial** — violates 2NF! |

**Conclusion:** This relation **violates 2NF**, because `student_name` and `instructor` are both partially dependent on the composite key `{student_id, course_id}`.

**Why this causes redundancy (connecting back to Module 7):**
- `student_name` for a given student gets repeated in **every row** where that student appears (once per course they're enrolled in).
- `instructor` for a given course gets repeated in **every row** where that course appears (once per student enrolled in it).

This is exactly the redundancy — and the resulting update/insertion/deletion anomalies — demonstrated in Module 7's `StudentCourse` example.

---

### 9.4 Detecting 2NF Violations — Systematic Procedure

```
Algorithm: Check-2NF(R, F)

1. Verify R is in 1NF. If not, fix that first (Module 8) — 2NF is meaningless otherwise.
2. Find ALL candidate keys of R (Module 6's systematic search).
3. If every candidate key is a SINGLE attribute → R is automatically in 2NF. STOP.
4. Otherwise, for each candidate key K (which has 2+ attributes):
     For each proper, non-empty subset K' of K:
         Compute (K')⁺ using F.
         For each non-prime attribute A of R:
             if A ∈ (K')⁺:
                 A has a PARTIAL dependency on K → 2NF VIOLATION.
5. If no non-prime attribute has a partial dependency on any candidate key → R is in 2NF.
```

**Critical detail:** Step 4 explicitly restricts the check to **non-prime attributes only**. A prime attribute (one that belongs to some candidate key) is allowed to depend on a subset of another candidate key without violating 2NF — this is a commonly-missed subtlety, elaborated in Section 9.7.

---

### 9.5 Converting a Relation to 2NF — The Decomposition Process

```
Algorithm: Convert-To-2NF(R, F)

1. For each candidate key K, identify every non-prime attribute A with a
   partial dependency on some proper subset K' ⊂ K.
2. For each such partial dependency K' → A:
     a. Create a new relation R1(K', A, [any other attributes also
        partially dependent on exactly K']).
     b. Remove A (and those other attributes) from the original relation.
3. The original relation retains: the full candidate key K, plus any
   attributes that are FULLY dependent on the WHOLE key.
4. Repeat until no non-prime attribute has a partial dependency on any
   remaining candidate key in any resulting relation.
```

**Applying this to Section 9.3's example:**

Partial dependencies found:
- `student_id → student_name` (partial, since `student_id` is a proper subset of `{student_id, course_id}`)
- `course_id → instructor` (partial, same reasoning)

**Decomposition:**

```
Students(student_id, student_name)
Courses(course_id, instructor)
Enrollment(student_id, course_id, grade)     -- grade is FULLY dependent on the whole key
```

**Verification — is this now in 2NF?**
- `Students(student_id, student_name)`: candidate key is `{student_id}` (single attribute) → automatically 2NF (Section 9.2).
- `Courses(course_id, instructor)`: candidate key is `{course_id}` (single attribute) → automatically 2NF.
- `Enrollment(student_id, course_id, grade)`: candidate key is `{student_id, course_id}`; the only non-prime attribute `grade` is **fully** dependent on the whole key → 2NF satisfied.

**All three resulting tables are in 2NF**, and — as a bonus — this decomposition exactly matches the anomaly-fixing decomposition previewed informally in Module 7, Section 7.9, except now we've arrived at it through a **rigorous, repeatable algorithm** rather than intuition.

---

### 9.6 Visual — Before and After 2NF Decomposition

```
BEFORE (violates 2NF):
┌──────────────────────────────────────────────────────┐
│  Enrollment(student_id, course_id, student_name,       │
│             instructor, grade)                          │
│  Key: {student_id, course_id}                            │
│                                                          │
│  student_id ────────▶ student_name    (PARTIAL — bad!)   │
│  course_id ─────────▶ instructor       (PARTIAL — bad!)  │
│  {student_id,                                            │
│   course_id} ───────▶ grade            (FULL — fine)      │
└──────────────────────────────────────────────────────┘

AFTER (2NF-compliant decomposition):
┌────────────────────┐  ┌────────────────────┐  ┌──────────────────────────┐
│ Students             │  │ Courses              │  │ Enrollment                 │
│ student_id (PK)      │  │ course_id (PK)        │  │ student_id, course_id (PK)  │
│ student_name          │  │ instructor            │  │ grade                       │
└────────────────────┘  └────────────────────┘  └──────────────────────────┘
```

---

### 9.7 Subtlety — Partial Dependency Involving a PRIME Attribute Is Allowed

**Important nuance most students miss:**
2NF's definition explicitly says *"every **non-prime** attribute must be fully dependent..."* — it says nothing about prime attributes. This means a **prime attribute CAN have a partial dependency on a candidate key without violating 2NF.**

**Example:**

```
R(A, B, C)
F = { {A,B} → C, C → A }

Candidate keys: {A,B} and {B,C}  (verify via closure, as in Module 6's exercises)
Prime attributes: A, B, C  (all three appear in some candidate key — {A,B} and {B,C})
Non-prime attributes: NONE
```

Here, `C → A` is technically a dependency where `A` (part of the candidate key `{A,B}`) depends on `C` alone (part of the OTHER candidate key `{B,C}`) — but since `A` is a **prime attribute**, this does **not** violate 2NF, because 2NF's rule only restricts non-prime attributes. If there are no non-prime attributes at all (as in this example), the relation is **automatically in 2NF**, no matter how "tangled" the dependencies among prime attributes might look.

> **Why this matters for interviews:** This exact trap — a relation with no non-prime attributes, or with a partial dependency involving only prime attributes — is a very common "gotcha" in senior-level and GATE-style 2NF questions, designed to test whether you truly understand the prime/non-prime distinction from Module 6, rather than just pattern-matching "partial dependency = automatically bad."

---

### 9.8 2NF — Comparison Table (1NF vs 2NF)

| Aspect | 1NF | 2NF |
|---|---|---|
| **Core requirement** | Atomic values, no repeating groups | 1NF + no partial dependency of non-prime attributes on any candidate key |
| **Applies to** | Every relation | Only meaningfully restrictive for relations with COMPOSITE candidate keys |
| **Automatically satisfied when...** | N/A (always a baseline requirement) | Every candidate key is a single attribute |
| **Fixes which anomaly root cause?** | Non-atomic/repeating-group redundancy | Redundancy from attributes depending on only PART of a composite key |
| **Typical fix** | Split multi-valued columns into separate tables | Split off partially-dependent attributes into their own table, keyed by the relevant partial key |

---

### 9.9 Industry Implementation Notes

- 2NF violations are extremely common in **hastily bottom-up-designed junction/association tables** — e.g., an `order_items` table that stores `product_name` and `product_price` directly (partially dependent on `product_id` alone) instead of just referencing a separate `products` table.
- Real-world ORMs (Django, Rails ActiveRecord, SQLAlchemy, Hibernate) naturally encourage 2NF-compliant designs when you define proper foreign-key relationships between models — the "partial dependency" anti-pattern usually creeps in specifically when developers denormalize *without realizing it*, often by copy-pasting columns "for convenience" during rapid prototyping.
- Some analytics/reporting schemas (data warehouse **fact tables**, previewed in Module 17) *deliberately* violate 2NF-style purity by storing denormalized descriptive attributes directly in fact rows — this is a conscious performance trade-off (Module 14), not an oversight, and is generally acceptable specifically because those tables are not the transactional source of truth.

---

### 9.10 Advantages of 2NF Compliance

- Directly eliminates the redundancy (and the resulting update/insertion/deletion anomalies from Module 7) caused by attributes belonging to only part of a composite key.
- Produces cleaner, more semantically meaningful tables — each resulting table tends to represent a genuinely cohesive concept (e.g., "Students," "Courses," rather than a tangled mix).
- A necessary stepping stone toward 3NF and BCNF — you cannot meaningfully check 3NF (Module 10) on a relation that still has uncorrected 2NF violations.

### 9.11 Disadvantages / Trade-offs

- Introduces additional tables and joins compared to the original "flat" design — a real query-time cost that must be weighed against the integrity benefits (Module 1's core trade-off).
- For very small, rarely-updated composite-key tables, the "risk" of a 2NF violation's anomalies manifesting might be low enough that some teams deliberately tolerate it — though this should be a conscious choice, documented as such, not an accidental oversight.

---

### 9.12 Best Practices

- Always find **all** candidate keys first (Module 6) before checking for 2NF — missing a candidate key can cause you to misclassify a prime attribute as non-prime, leading to an incorrect "violation" finding (Section 9.7's trap).
- Check 2NF **only** on relations with composite candidate keys — don't waste time checking relations with single-attribute keys, since they're automatically compliant (Section 9.2).
- When decomposing to fix a 2NF violation, immediately re-verify that each resulting table is properly in 1NF and 2NF — decomposition can sometimes introduce new candidate keys or subtly change which attributes are prime in the new, smaller tables.

---

### 9.13 Common Mistakes

- Checking for 2NF violations without first finding **all** candidate keys — leads to misclassifying prime attributes as non-prime (Section 9.7).
- Assuming ANY partial dependency is automatically a 2NF violation, forgetting the crucial "non-prime attribute" qualifier (Section 9.7's trap).
- Trying to check 2NF on a relation with only single-attribute candidate keys — wasted effort, since it's automatically satisfied (Section 9.2).
- After decomposing, forgetting to check whether the SAME attribute now needs further decomposition in ANOTHER resulting table (multi-step normalization, revisited fully as part of Module 13's decomposition algorithms).

---

### 9.14 Edge Cases

- **A relation where all attributes are prime** (as in Section 9.7's example) — automatically in 2NF regardless of how the dependencies look among prime attributes.
- **A relation with multiple composite candidate keys**, where an attribute is non-prime with respect to ALL of them but only partially dependent on ONE of the several candidate keys — this still counts as a 2NF violation; the rule must hold with respect to **every** candidate key, not just one.
- **A relation already trivially in 2NF because it has no non-prime attributes at all** (every attribute participates in some candidate key) — this can happen with junction/associative tables that have no additional descriptive attributes beyond the keys themselves (a "pure" many-to-many linking table).

---

### 9.15 Performance Implications

- 2NF decomposition typically produces smaller, more focused tables, each requiring less storage per logical entity (no repeated `student_name` or `instructor` data).
- The trade-off is the need for **joins** to reconstruct a "full picture" — e.g., a report showing student name, course instructor, and grade together now requires joining three tables instead of reading one flat one. This exact trade-off is revisited with concrete performance guidance in Module 14 (Denormalization) and Module 17 (Industry Trade-offs).

---

### 9.16 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | Define 2NF and partial dependency; identify a simple 2NF violation. |
| Beginner | Explain why a relation with a single-attribute key is automatically in 2NF. |
| Intermediate | Given a relation with a composite key and several attributes, identify all partial dependencies and decompose into 2NF. |
| Senior SDE-2/SDE-3 | Given a real "flat" junction-table-style schema exhibiting update anomalies, diagnose the 2NF violation and propose (and justify) a decomposition. |
| GATE | Classic "given this relation and FD set, is it in 2NF? If not, decompose it" numerical problems — extremely common question type. |

---

### 9.17 Chapter Summary

- **2NF** requires 1NF **plus** no partial dependency of any **non-prime** attribute on any candidate key.
- 2NF is **automatically satisfied** when every candidate key is a single attribute — violations only occur with **composite** candidate keys.
- The fix is to **decompose** the relation, moving partially-dependent attributes into their own table keyed by just the relevant part of the composite key.
- A critical, commonly-missed subtlety: **partial dependency involving only prime attributes does NOT violate 2NF** — the rule applies exclusively to non-prime attributes.
- 2NF decomposition directly eliminates the redundancy (and resulting anomalies) demonstrated in Module 7.

---

### 9.18 Cheat Sheet

```
2NF = 1NF + NO PARTIAL DEPENDENCY of any NON-PRIME attribute on any candidate key

AUTOMATIC 2NF: if every candidate key is a SINGLE attribute → done, no check needed.

CHECK (only for composite keys):
  For each candidate key K, for each proper subset K' ⊂ K:
    does K' → (some non-prime attribute)?  → VIOLATION if yes

TRAP: partial dependency on a PRIME attribute is ALLOWED — only non-prime attributes matter.

FIX: split partially-dependent non-prime attributes into their own table,
     keyed by the relevant partial key.
```

---

### 9.19 MCQs

**Q1.** A relation with only single-attribute candidate keys is:
a) Never in 2NF
b) Automatically in 2NF (assuming 1NF holds)
c) Automatically in 3NF as well
d) Impossible to normalize

<details><summary>Answer</summary>b) Automatically in 2NF</details>

**Q2.** 2NF's "no partial dependency" rule applies specifically to:
a) All attributes
b) Only prime attributes
c) Only non-prime attributes
d) Only foreign key attributes

<details><summary>Answer</summary>c) Only non-prime attributes</details>

**Q3.** In `Enrollment(student_id, course_id, student_name, grade)` with key `{student_id, course_id}`, the dependency `student_id → student_name` is:
a) A full dependency, satisfying 2NF
b) A partial dependency, violating 2NF (assuming student_name is non-prime)
c) A transitive dependency
d) Not a valid functional dependency

<details><summary>Answer</summary>b) A partial dependency, violating 2NF</details>

**Q4.** If a relation has no non-prime attributes at all (every attribute belongs to some candidate key), is it automatically in 2NF?
a) No, it must still be checked
b) Yes, automatically, since 2NF's rule only restricts non-prime attributes
c) Only if it has a single candidate key
d) 2NF doesn't apply to such relations

<details><summary>Answer</summary>b) Yes, automatically</details>

**Q5.** The standard fix for a 2NF violation is to:
a) Add more indexes
b) Decompose the relation, moving the partially-dependent attribute into its own table keyed by the relevant partial key
c) Merge the relation with another table
d) Convert the attribute to a JSON column

<details><summary>Answer</summary>b) Decompose into a separate table keyed by the partial determinant</details>

---

### 9.20 Exercises

**Beginner:**
1. Define 2NF and partial dependency in your own words.
2. Explain why a relation like `Student(student_id, name, dob)` (single-attribute key) is automatically in 2NF.

**Intermediate — 2NF Violation Detection:**
3. Given `OrderItem(order_id, product_id, product_name, product_price, quantity)` with key `{order_id, product_id}`, identify any partial dependencies and decompose into 2NF.
4. Given `Booking(hotel_id, room_type, guest_id, guest_name, hotel_city, price_per_night)` with key `{hotel_id, room_type, guest_id}` (assume this is the actual candidate key), identify all partial dependencies and propose a 2NF-compliant decomposition.

**Advanced Normalization Problems:**
5. `R(A,B,C,D)` with candidate keys `{A,B}` and `{B,C}` (verify this using closure), and FD `C → D`. Is `D` prime or non-prime? Does `C → D` violate 2NF? Justify carefully using Section 9.7's subtlety.
6. Construct your own example (different from any in this module) of a relation where a partial dependency exists but does NOT violate 2NF, because the dependent attribute is prime.

**Case Study:**
7. Revisit Module 7's `StudentCourse` example (`student_id, student_name, course, instructor, instructor_office, grade`). Assuming the only candidate key is `{student_id, course}`, formally identify every partial dependency, and produce the full 2NF decomposition, explicitly naming each resulting table and its key.

**Whiteboard Design:**
8. Design a 2NF-compliant schema for a `Conference_Registration` system where `(attendee_id, session_id)` is the composite key, and the table also currently stores `attendee_name`, `attendee_email`, `session_title`, `session_room`, and `attendance_confirmed`. Identify which attributes are partially dependent and produce the decomposition.

**GATE-style Numerical:**
9. `R(A,B,C,D,E)` with candidate key `{A,B}`, and FDs `A → C`, `B → D`, `{A,B} → E`. Determine whether R is in 2NF. If not, provide the decomposed relations.

**Interview Questions:**
- *Beginner:* "What is 2NF, and when is it automatically satisfied?"
- *Intermediate:* "How do you detect and fix a partial dependency?"
- *Senior:* "In a data warehouse fact table, would you enforce strict 2NF? Why or why not?"

**Practice links:**
- GeeksforGeeks — Second Normal Form (2NF): https://www.geeksforgeeks.org/dbms/second-normal-form-2nf/
- GateOverflow — 2NF tagged questions: https://gateoverflow.in/

---

**End of Module 9.**

*Next up — Module 10: Third Normal Form (3NF) — eliminating transitive dependencies, and introducing the famous "prime attribute escape hatch" that distinguishes 3NF from the stricter BCNF (Module 11). Reply "next" / "continue" when ready.*
