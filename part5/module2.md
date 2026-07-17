# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 2: Functional Dependencies (FD)

> **Where we are:** Module 1 established *why* and *when* database design happens. From here on, we build the mathematical toolkit that makes **Logical Design** (and specifically, normalization) rigorous. Functional Dependencies are the single most important concept in this entire Part — nearly everything from Module 3 onward (closures, keys, Armstrong's Axioms, all normal forms) is built directly on top of FDs.

---

### 2.1 What Is a Functional Dependency?

**Definition (informal):**
A **Functional Dependency (FD)** is a constraint between two sets of attributes in a relation, saying: *"if you know the value of attribute set X, you can always determine the value of attribute set Y."*

We write this as:

```
X → Y     (read as: "X functionally determines Y", or "Y is functionally dependent on X")
```

**Definition (formal, mathematical):**
Let `R` be a relation with attribute set `{A1, A2, ..., An}`, and let `X` and `Y` be subsets of these attributes. We say `X → Y` holds on `R` if and only if:

> For every pair of tuples `t1` and `t2` in any valid instance of `R`, if `t1[X] = t2[X]`, then `t1[Y] = t2[Y]`.

In plain English: **no two rows can agree on X but disagree on Y.** If they agree on X, they *must* agree on Y.

**Determinant and Dependent Attribute:**

| Term | Meaning | In `X → Y` |
|---|---|---|
| **Determinant** | The attribute(s) on the left-hand side, whose value determines the other side | `X` |
| **Dependent attribute** | The attribute(s) on the right-hand side, whose value is determined by X | `Y` |

**Example:**

```
Table: Student
+------------+--------------+-------+
| roll_no    | name         | dept  |
+------------+--------------+-------+
| 101        | Aditi        | CSE   |
| 102        | Rohan        | ECE   |
| 103        | Aditi        | CSE   |
+------------+--------------+-------+
```

Here `roll_no → name` and `roll_no → dept` hold: knowing the roll number always tells you the name and department, because no two rows share a `roll_no`. But `name → dept` does **not** necessarily hold in general (even though it happens to hold in *this particular instance*, since both "Aditi" rows have "CSE") — because two different students could coincidentally share a name but belong to different departments. This distinction — a dependency holding "by coincidence" in one instance vs. holding **for all valid instances** because of the underlying business rule — is critical and revisited in Section 2.10.

---

### 2.2 Why Does Functional Dependency Exist as a Concept?

**What problem does it solve?**
Before FDs, relational schema design was guesswork — "this looks redundant" or "this feels fine" based on intuition. FDs give us a **formal, provable way** to:

1. Detect redundancy mathematically (rather than by eyeballing sample data).
2. Determine candidate keys rigorously (Module 3, Module 6).
3. Decide exactly which normal form a relation satisfies (Modules 8–12).
4. Guide **lossless, dependency-preserving decomposition** of a bad relation into good ones (Module 13).

**Analogy — The Vending Machine:**
Think of a functional dependency as a vending machine's behavior: if you press button `B4` (the determinant), you will **always** get the same snack (the dependent attribute) — never a different one on a different day. If pressing `B4` sometimes gives chips and sometimes gives chocolate, the machine violates the "B4 → snack" dependency, and something is structurally wrong (or `B4` isn't actually determining the snack — maybe there's a hidden factor, meaning your "key" is incomplete).

---

### 2.3 How Functional Dependencies Are Identified in Practice

FDs are **not derived from a sample of rows** — they come from **business rules / real-world semantics**. This is one of the most commonly misunderstood points by students.

```
             ┌─────────────────────────────┐
             │   Business Rule / Domain     │
             │   Knowledge (real world)     │
             └──────────────┬──────────────┘
                            │  gives rise to
                            ▼
             ┌─────────────────────────────┐
             │   Functional Dependencies     │
             │        (X → Y)                │
             └──────────────┬──────────────┘
                            │  drive
                            ▼
             ┌─────────────────────────────┐
             │  Keys, Normalization,         │
             │  Schema Design                │
             └─────────────────────────────┘
```

**Example:** In a `Course` relation, the rule "each course code corresponds to exactly one course title" is a business rule. This *gives rise to* the FD `course_code → course_title`. You don't "check the data" to discover this FD — you confirm it with the domain (or it's an inherent rule of the system, like a course code being globally unique by design).

> **Common Mistake:** Looking at a small data sample and inferring an FD that doesn't actually hold in general (e.g., seeing `name → dept` hold in a 3-row sample and assuming it's a real dependency). Always ask: *"Could this ever be violated by a valid future row?"* If yes, it's not a real FD.

---

### 2.4 Trivial Functional Dependency

**Definition:**
`X → Y` is **trivial** if `Y` is a subset of `X` (i.e., `Y ⊆ X`).

**Why does it exist / what does it mean?**
A trivial FD is *always* true, for any relation, by simple set logic — it carries **no real information** about the data. If you already know X, and Y is part of X, of course you know Y.

**Examples:**
```
{roll_no, name} → {roll_no}        -- trivial (roll_no is part of the LHS)
{roll_no, name} → {name}           -- trivial
{roll_no} → {roll_no}              -- trivial (the most degenerate case)
```

---

### 2.5 Non-Trivial Functional Dependency

**Definition:**
`X → Y` is **non-trivial** if `Y` is **not** a subset of `X` (i.e., at least one attribute in `Y` is not in `X`).

**Examples:**
```
{roll_no} → {name}                  -- non-trivial (name ∉ {roll_no})
{roll_no} → {name, dept}            -- non-trivial (neither name nor dept is in {roll_no})
{roll_no, name} → {dept}            -- non-trivial
```

Non-trivial FDs are the ones that actually matter for design — they represent real constraints, not tautologies.

---

### 2.6 Completely Non-Trivial Functional Dependency

**Definition:**
`X → Y` is **completely non-trivial** (also called "completely non-trivial FD") if `X` and `Y` share **no attributes at all** — i.e., `X ∩ Y = ∅`.

**Why distinguish this from plain non-trivial?**
A non-trivial FD can still have *partial* overlap between X and Y. A completely non-trivial FD has *zero* overlap — it's the "purest" form of a meaningful dependency.

**Example:**
```
{roll_no, name} → {dept, city}     -- non-trivial, AND completely non-trivial
                                       (no attribute appears on both sides)

{roll_no, name} → {name, dept}     -- non-trivial, but NOT completely non-trivial
                                       (name appears on both sides)
```

| FD Type | Condition | Carries new information? |
|---|---|---|
| Trivial | `Y ⊆ X` | No — always true |
| Non-trivial | `Y ⊄ X` (at least one attribute of Y not in X) | Yes, at least partially |
| Completely non-trivial | `X ∩ Y = ∅` | Yes, entirely |

---

### 2.7 Full Functional Dependency

**Definition:**
`X → Y` is a **full functional dependency** if `Y` is functionally dependent on the **entire** set `X`, but **not** on any proper subset of `X`.

Formally: `X → Y` is full if there is no proper subset `X' ⊂ X` such that `X' → Y` also holds.

**Why does it matter?**
This concept is the exact foundation of **Second Normal Form (2NF)** in Module 9. A relation violates 2NF precisely when some non-key attribute depends on only *part* of a composite key rather than the *whole* key.

**Example:**

```
Table: Enrollment(student_id, course_id, student_name, grade)

FD: {student_id, course_id} → grade
```

Is this a **full** FD? Check: does `{student_id} → grade` hold alone? No (a student has different grades in different courses). Does `{course_id} → grade` hold alone? No (different students get different grades in the same course). Since neither proper subset determines `grade`, this **is** a full functional dependency.

---

### 2.8 Partial Functional Dependency

**Definition:**
`X → Y` is a **partial functional dependency** if `Y` is functionally dependent on a **proper subset** of `X` (where `X` is a composite/multi-attribute determinant, typically a composite key).

Formally: `X → Y` is partial if there **exists** a proper subset `X' ⊂ X` such that `X' → Y` holds.

**Example (continuing the Enrollment table above):**

```
Table: Enrollment(student_id, course_id, student_name, grade)

FD: {student_id, course_id} → student_name
```

Check: does `{student_id} → student_name` hold alone? **Yes** — a given student always has the same name, regardless of which course we're looking at. Since `student_name` depends on just `student_id` (a *proper subset* of the composite key `{student_id, course_id}`), this is a **partial dependency**.

**Why does it matter?**
Partial dependencies are the root cause of **2NF violations** (Module 9) — they mean the composite key is "over-determining" some attribute, which leads directly to redundancy (student_name gets repeated for every course the student takes) and update anomalies.

---

### 2.9 Full vs Partial Dependency — Side-by-Side

| Aspect | Full FD | Partial FD |
|---|---|---|
| **Depends on** | The *entire* composite determinant | A *proper subset* of the composite determinant |
| **Applies only when** | X has 2+ attributes | X has 2+ attributes (irrelevant for single-attribute X) |
| **Redundancy risk** | Low (as far as this FD is concerned) | High — attribute repeats for every value of the "extra" key part |
| **Related normal form** | Satisfying "no partial dependency" is required for 2NF | Presence of this = 2NF violation |
| **Example** | `{student_id, course_id} → grade` | `{student_id, course_id} → student_name` (reduces to `student_id → student_name`) |

---

### 2.10 Transitive Dependency

**Definition:**
Given FDs `X → Y` and `Y → Z`, if `X → Z` also holds (which it does, by the transitivity axiom — see Module 4), and importantly, **Y is not a candidate key** (i.e., Y does not determine X back), then `Z` is said to be **transitively dependent** on `X` through `Y`.

Formally: `X → Z` is transitive if there exists some `Y` such that `X → Y`, `Y → Z` hold, and `Y ↛ X` (Y does not determine X).

**Why the "Y is not a key" condition matters:**
If `Y → X` also held (i.e., X and Y determine each other — they're both candidate keys, or Y is an alternate key), then Z depending on X "through" Y isn't really *transitive* in the problematic sense — X and Y are essentially interchangeable identifiers. The dangerous case is when Y is a **non-key** attribute sitting "in between" the key and another non-key attribute.

**Example:**

```
Table: Employee(emp_id, dept_id, dept_name)

FD1: emp_id → dept_id           (each employee belongs to one department)
FD2: dept_id → dept_name        (each department has one name)
FD3: emp_id → dept_name         (follows from FD1 + FD2 by transitivity)
```

Here, `dept_name` depends on `emp_id` **transitively**, through the non-key attribute `dept_id`. This is a problem: `dept_name` gets repeated for every employee in the same department — classic redundancy, and the direct target of **Third Normal Form (3NF)** in Module 10.

**Diagram:**

```
emp_id ──────▶ dept_id ──────▶ dept_name
   │                                ▲
   └────────────transitive──────────┘
        (emp_id → dept_name, via dept_id)
```

> **Callout — revisiting Section 2.1's caveat:** This is exactly the kind of dependency that must come from a genuine business rule ("each department has exactly one name," "each employee belongs to exactly one department at a time"), not from eyeballing a small sample where it might coincidentally look true.

---

### 2.11 Multivalued Dependency (Introduction)

**What is it?**
A **Multivalued Dependency (MVD)**, written `X →→ Y`, is a more general kind of constraint than a functional dependency. It says: *"for each value of X, there is a set of associated values of Y, and this set is independent of the other attributes in the relation."*

Unlike an FD (which says X determines a *single* value of Y), an MVD says X determines a *set* of values of Y — and crucially, **Y and the "rest" of the attributes vary independently of each other** given X.

**Why introduce it here?**
MVDs arise when a table incorrectly tries to combine **two independent multi-valued facts** about the same entity into one relation, causing a specific kind of redundancy that FDs alone cannot detect or fix.

**Example (preview — full treatment in Module 12):**

```
Table: Employee_Skills_Languages(emp_id, skill, language)
```

Suppose employee `E1` has skills `{Python, SQL}` and speaks languages `{English, Hindi}`, and these two facts are **completely independent** of each other (which skill has nothing to do with which language). To represent all combinations, you'd need:

```
E1, Python, English
E1, Python, Hindi
E1, SQL,    English
E1, SQL,    Hindi
```

This is redundant — the skills and languages are cross-multiplied even though they're unrelated. This is a Multivalued Dependency problem: `emp_id →→ skill` and `emp_id →→ language`. **No FD captures this** — `emp_id` doesn't functionally determine a single skill, but it does determine a *set* of skills independent of language.

We formally define and resolve this in **Module 12 (Fourth Normal Form)**. For now, just recognize: **MVDs are about independent multi-valued facts crammed into one relation.**

---

### 2.12 Join Dependency (Introduction)

**What is it?**
A **Join Dependency (JD)**, written `⋈(R1, R2, ..., Rn)`, is the most general form of dependency. It says: a relation `R` can be losslessly reconstructed by joining several of its projections `R1, R2, ..., Rn` back together — but this lossless reconstruction is **only** possible with *all* of these projections (any subset joined together would lose information or introduce spurious tuples).

**Why introduce it here?**
Functional Dependencies are a special case of Multivalued Dependencies, and Multivalued Dependencies are a special case of Join Dependencies. This is the most general tool in the dependency theory toolbox:

```
Functional Dependency  ⊂  Multivalued Dependency  ⊂  Join Dependency
     (X → Y)                  (X →→ Y)                 (⋈(R1,...,Rn))
    (most specific)                                    (most general)
```

**Why does it matter?**
Some relations have redundancy that **cannot** be eliminated using FDs or even MVDs alone — only a 3-way (or more) join dependency captures the constraint. This leads to **Fifth Normal Form (5NF)**, covered in Module 12.

We revisit this formally with a full worked example in Module 12. For now, the key takeaway is the **hierarchy of dependency types** above — each is a generalization of the previous, and each corresponds to progressively higher normal forms.

---

### 2.13 The Dependency Hierarchy — Visual Summary

```
┌───────────────────────────────────────────────────────────────┐
│                     JOIN DEPENDENCY (JD)                        │
│              "R can be losslessly reconstructed                │
│               from a join of specific projections"              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              MULTIVALUED DEPENDENCY (MVD)                 │   │
│  │        "X determines a SET of Y-values, independent       │   │
│  │              of the rest of the attributes"                │   │
│  │   ┌───────────────────────────────────────────────────┐   │   │
│  │   │           FUNCTIONAL DEPENDENCY (FD)                │   │   │
│  │   │     "X determines exactly ONE value of Y"           │   │   │
│  │   └───────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────┘

     Drives:  1NF/2NF/3NF/BCNF    4NF              5NF
```

---

### 2.14 Industry Implementation Notes

- Relational DBMSs (PostgreSQL, MySQL, Oracle, SQL Server) do **not** store or enforce arbitrary functional dependencies directly — FDs are a *design-time* concept.
- The closest *runtime* enforcement mechanisms are: `PRIMARY KEY` and `UNIQUE` constraints (enforce that a key determines all other attributes uniquely), `FOREIGN KEY` constraints (enforce referential integrity, related but distinct from FDs), and `CHECK` constraints (enforce arbitrary business rules, sometimes approximating an FD).
- Schema design tools (e.g., dbdiagram.io, ER/Studio, MySQL Workbench's schema validator) sometimes offer "normalization advisors" that infer likely FDs from sample data or column names — but these are heuristics, not guarantees; the real FDs must come from business analysis (Section 2.3).
- In interviews, when given a flat relation and asked to normalize it, you are typically **given the FDs explicitly** (since they can't be reliably inferred from a few sample rows) — this is a common GATE and placement-test pattern.

---

### 2.15 Advantages of Formalizing Functional Dependencies

- Converts subjective "this looks redundant" judgments into objective, provable facts.
- Enables algorithmic determination of keys (Module 3), normal forms (Modules 8–12), and safe decomposition (Module 13).
- Provides a shared vocabulary between database designers, backend engineers, and DBAs for discussing schema correctness.
- Forms the theoretical basis for automated schema design and normalization tools.

### 2.16 Disadvantages / Limitations

- FDs must be correctly and completely identified from business rules — an incomplete or wrong FD set leads to an incorrectly normalized (or under/over-normalized) schema.
- Real-world business rules change over time; FDs that were valid at design time can become invalid, requiring schema re-evaluation (Module 15).
- Some real-world constraints (see MVD, JD above) genuinely cannot be expressed as simple FDs, requiring more advanced theory.
- Overzealous FD enforcement (via triggers/constraints) at the physical level can hurt write performance — this ties back into the normalization-vs-performance trade-off from Module 1.

---

### 2.17 Best Practices

- Always derive FDs from **documented business rules**, not from eyeballing sample data.
- Explicitly write out the full FD set before starting normalization — don't normalize "by feel."
- Watch for **hidden composite keys** — an FD that looks like it's violated might actually mean your assumed key is incomplete (e.g., you assumed `student_id → grade`, but grade actually needs `{student_id, course_id}`).
- Re-validate FDs whenever business requirements change (e.g., "a student can now retake a course" might invalidate a previously-valid FD).
- Document FDs alongside the schema (as comments, or in a data dictionary) — future engineers need to know *why* the schema is shaped the way it is.

---

### 2.18 Common Mistakes

- Confusing "this holds in my sample data" with "this is a true functional dependency" (Section 2.1, 2.3).
- Forgetting to check trivial FDs when computing closures (Module 3) — while trivial, they still count in a complete FD set.
- Misidentifying a partial dependency as full (or vice versa) when the determinant has 2+ attributes — always explicitly test each proper subset.
- Assuming `X → Y` implies `Y → X` — functional dependency is **not symmetric**. (`roll_no → name` does not imply `name → roll_no`, since names aren't necessarily unique.)
- Ignoring multivalued dependencies entirely and trying to "fix" MVD-driven redundancy using only FD-based normalization (up to 3NF/BCNF) — this won't eliminate the redundancy; you need 4NF (Module 12).

---

### 2.19 Performance Implications

- A relation that violates good FD structure (e.g., unresolved transitive or partial dependencies) tends to have **wider rows with repeated data**, increasing storage and I/O per row scanned.
- Update operations become more expensive and error-prone when a single logical fact (e.g., `dept_name`) is physically duplicated across many rows, because every duplicate must be updated in lockstep (or a bug arises).
- Conversely, over-normalizing based on excessive FD granularity can lead to **too many small tables**, requiring many joins for common queries — this is where Module 14 (Denormalization) becomes relevant as a deliberate, informed trade-off, not a mistake.

---

### 2.20 Interview Importance & Sample Questions

| Level | Sample Question |
|---|---|
| Beginner | "What is a functional dependency? Give an example." |
| Beginner | "What's the difference between a trivial and non-trivial FD?" |
| Intermediate | "Given `{A,B} → C` and `{A} → C`, is `{A,B} → C` a full or partial dependency?" |
| Intermediate | "Explain transitive dependency with an example, and why it matters for 3NF." |
| Senior SDE-2/SDE-3 | "You're given a flat `Orders` table with 15 columns and a set of business rules. Derive the functional dependencies, and explain how you'd validate them with the product team before normalizing." |
| Senior SDE-2/SDE-3 | "Explain the difference between a functional dependency and a multivalued dependency, and give a real schema example where FD-based normalization (up to BCNF) is insufficient." |

---

### 2.21 Chapter Summary

- A **Functional Dependency** `X → Y` means: knowing X's value always tells you Y's value.
- FDs come from **business rules**, not from sample data.
- **Trivial** FDs (`Y ⊆ X`) are always true and carry no information; **non-trivial** and **completely non-trivial** FDs carry real constraints.
- **Full** dependencies rely on the *entire* composite determinant; **partial** dependencies rely on a *subset* of it — partial dependencies drive 2NF violations.
- **Transitive** dependencies pass through an intermediate non-key attribute — they drive 3NF violations.
- **Multivalued Dependencies (MVDs)** and **Join Dependencies (JDs)** generalize FDs to capture independent multi-valued facts and complex reconstruction constraints, respectively — driving 4NF and 5NF (Module 12).
- The hierarchy is: **FD ⊂ MVD ⊂ JD**, from most specific to most general.

---

### 2.22 Cheat Sheet

```
X → Y                : X functionally determines Y
Trivial               : Y ⊆ X                          (always true)
Non-trivial            : Y ⊄ X                           (at least one new attribute)
Completely non-trivial : X ∩ Y = ∅                        (zero overlap)
Full FD                : X → Y, no proper subset X'⊂X has X'→Y
Partial FD              : X → Y, SOME proper subset X'⊂X has X'→Y
Transitive FD           : X→Y, Y→Z, Y is NOT a key, therefore X→Z is transitive
MVD (X →→ Y)            : X determines a SET of Y-values, independent of rest
JD  ⋈(R1,...,Rn)         : R reconstructible only by joining ALL listed projections

Hierarchy: FD ⊂ MVD ⊂ JD  (most specific → most general)
Drives:    1NF/2NF/3NF/BCNF   4NF   5NF
```

---

### 2.23 MCQs

**Q1.** Which of the following is always true, regardless of the data?
a) `roll_no → name`
b) `{roll_no, name} → roll_no`
c) `name → roll_no`
d) `dept → dept_name`

<details><summary>Answer</summary>b) `{roll_no, name} → roll_no` — this is a trivial FD (RHS ⊆ LHS), true by definition.</details>

**Q2.** In `Enrollment(student_id, course_id, grade)` with key `{student_id, course_id}`, if `{student_id, course_id} → grade` and no proper subset determines grade, this FD is:
a) Partial
b) Transitive
c) Full
d) Trivial

<details><summary>Answer</summary>c) Full</details>

**Q3.** Given `A → B` and `B → C`, with `B` not a candidate key, `A → C` is an example of:
a) Partial dependency
b) Transitive dependency
c) Multivalued dependency
d) Trivial dependency

<details><summary>Answer</summary>b) Transitive dependency</details>

**Q4.** Which statement about functional dependency is TRUE?
a) `X → Y` implies `Y → X`
b) FDs can always be inferred purely from sample data
c) FD is not symmetric — `X → Y` does not imply `Y → X`
d) All FDs are trivial by definition

<details><summary>Answer</summary>c) FD is not symmetric</details>

**Q5.** A relation storing an employee's independent sets of skills and languages in one table, causing cross-multiplied rows, is best described by which type of dependency?
a) Functional Dependency
b) Multivalued Dependency
c) Trivial Dependency
d) Partial Dependency

<details><summary>Answer</summary>b) Multivalued Dependency</details>

---

### 2.24 Exercises

**Beginner:**
1. Define functional dependency in your own words and give two original examples (not from this module).
2. Is `{A, B} → A` trivial or non-trivial? Justify.
3. Is `{A} → {B, C}` completely non-trivial? Justify.

**Intermediate — Functional Dependency Problems:**
4. Given relation `R(A, B, C, D)` with FDs `{A, B} → C`, `A → D`, determine whether `{A,B} → C` is full or partial.
5. Given `emp_id → dept_id` and `dept_id → dept_location`, is `emp_id → dept_location` transitive? What condition on `dept_id` must hold for this to count as transitive rather than just a derived FD?
6. For the relation `Book(isbn, title, author_name, author_nationality)`, assuming `isbn → title`, `isbn → author_name`, and `author_name → author_nationality`, identify any transitive dependency and explain the redundancy it would cause.

**Advanced — Dependency Classification Practice:**
7. For relation `R(A, B, C, D, E)` with FD set `{A,B} → C`, `A → D`, `{A,B} → E`, classify each FD as full or partial with respect to the determinant `{A,B}` (where applicable).
8. Construct your own example of a relation where a multivalued dependency exists but no meaningful functional dependency captures the redundancy. Explain why FD-based reasoning alone fails here.

**Whiteboard / Case Study:**
9. You're designing a `Hospital` database and are told: "each patient has one primary doctor, each doctor works in exactly one department, and each department has a unique name." Write out the resulting FDs, and identify which one (if any) is transitive with respect to `patient_id`.

**GATE-style Numerical:**
10. Given relation `R(A,B,C,D,E)` and FD set `F = {A → BC, CD → E, B → D, E → A}`, list which FDs (if any) in `F` are trivial, and identify any transitive dependency chain that can be derived from `F` (e.g., does `A` transitively determine any attribute via an intermediate non-key attribute?).

**Interview Questions:**
- *Beginner:* "What's the difference between trivial and non-trivial FDs?"
- *Intermediate:* "How do you distinguish a full functional dependency from a partial one?"
- *Senior:* "In a large legacy table with 30 columns, how would you go about identifying the true set of functional dependencies before attempting to normalize it?"

**Practice links:**
- GeeksforGeeks — Functional Dependency practice: https://www.geeksforgeeks.org/dbms/functional-dependency-in-dbms/
- GateOverflow — FD & Normalization tagged questions: https://gateoverflow.in/

---

**End of Module 2.**

*Next up — Module 3: Attribute Closure — the algorithmic tool that turns a set of FDs into candidate keys, super keys, and minimal keys. Reply "next" / "continue" when ready.*
