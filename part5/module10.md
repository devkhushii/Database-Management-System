# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 10: Third Normal Form (3NF)

> **Where we are:** Module 9 eliminated partial dependencies (2NF). Now we climb one more step and eliminate **transitive dependencies** — the concept from Module 2 (Section 2.10). 3NF also introduces the famous **"prime attribute escape hatch"**, a subtlety that is the single most-tested distinction between 3NF and the stricter BCNF (Module 11) in interviews and GATE papers.

---

### 10.1 What Is Third Normal Form (3NF)?

**Definition:**
A relation is in **Third Normal Form (3NF)** if and only if:
1. It is already in **2NF** (Module 9), **and**
2. For every non-trivial functional dependency `X → A` in the relation, **at least one** of the following holds:
   - **(a)** `X` is a **super key** of the relation, **OR**
   - **(b)** `A` is a **prime attribute** (i.e., `A` belongs to at least one candidate key).

**Why does it exist?**
Because even after eliminating partial dependencies (2NF), a relation can still suffer from **transitive dependencies** — where a non-prime attribute depends on another non-prime attribute, which in turn depends on the key. This transitive chain causes exactly the same *kind* of redundancy and anomaly risk as partial dependencies, just through a different structural pattern.

**What problem does it solve?**
3NF eliminates redundancy caused by **non-key attributes depending on other non-key attributes** rather than depending directly on a key.

---

### 10.2 The "Escape Hatch" — Why Clause (b) Exists

This is the single most important thing to understand about 3NF, and the exact reason it's **less strict** than BCNF (Module 11).

**Without clause (b)** (i.e., if 3NF only required "X is a super key"), the rule would be **identical to BCNF**. Clause (b) is a deliberate **relaxation**: it says *"even if X is NOT a super key, it's still okay — AS LONG AS the dependent attribute A is prime (part of some candidate key)."*

**Why would a relaxation like this be acceptable?**
Because the theoretical justification is: if `A` is prime, it means `A` is *itself* part of some candidate key — meaning `A` isn't just an arbitrary dependent fact, it has its own special status in identifying rows. Historically, this relaxation exists because a small, specific class of relations satisfies 3NF but **cannot** be losslessly decomposed into pure BCNF form while ALSO preserving all original dependencies (see Module 13's dependency-preservation discussion) — 3NF guarantees **dependency preservation** is always achievable, while BCNF sometimes has to sacrifice it. This trade-off is explored fully once we reach Module 11 and Module 13.

---

### 10.3 The Classic 3NF Violation — Worked Example

**Given (already in 2NF, but not yet checked for 3NF):**

```
Table: Employee(emp_id, emp_name, dept_id, dept_name)

Candidate Key: {emp_id}

FDs:
  emp_id → emp_name
  emp_id → dept_id
  dept_id → dept_name          ← this is the problem
  emp_id → dept_name           (derivable via transitivity from the above two)
```

**Checking 3NF for each FD:**

| FD | Is LHS a super key? | Is RHS prime? | 3NF satisfied? |
|---|---|---|---|
| `emp_id → emp_name` | Yes (`emp_id` is the candidate key) | N/A (clause a already satisfies it) | ✅ Yes |
| `emp_id → dept_id` | Yes | N/A | ✅ Yes |
| `dept_id → dept_name` | **No** (`dept_id` is not a super key — it doesn't determine `emp_name`) | **No** (`dept_name` is not part of any candidate key — the only candidate key is `{emp_id}`) | ❌ **VIOLATES 3NF** |

**Conclusion:** The FD `dept_id → dept_name` violates 3NF, because `dept_id` is not a super key, AND `dept_name` is not a prime attribute. This is exactly the **transitive dependency** pattern from Module 2, Section 2.10: `emp_id → dept_id → dept_name`, where `dept_name` depends on `emp_id` only *transitively*, through the non-key attribute `dept_id`.

---

### 10.4 Detecting 3NF Violations — Systematic Procedure

```
Algorithm: Check-3NF(R, F)

1. Verify R is in 2NF (Module 9). If not, fix that first.
2. Compute the CANONICAL COVER of F (Module 5) — working with a clean,
   minimal FD set makes this check far less error-prone.
3. Find ALL candidate keys of R (Module 6) → determine the set of PRIME attributes.
4. For EVERY functional dependency X → A in the canonical cover:
     a. Compute X⁺ using F. Is X⁺ = all attributes of R (i.e., is X a super key)?
        If YES → this FD satisfies 3NF (clause a). Move to next FD.
     b. If NO → check: is A a PRIME attribute?
        If YES → this FD still satisfies 3NF (clause b, the escape hatch).
        If NO → 3NF VIOLATION.
5. R is in 3NF if and only if EVERY FD in the canonical cover passes step 4.
```

**Why use the canonical cover (Step 2)?** Checking 3NF against a bloated, non-minimal FD set can cause you to miss violations hidden inside composite or redundant FDs — the canonical cover's singleton-RHS, no-extraneous-attribute form makes the "for every FD X → A" check in Step 4 clean and unambiguous (exactly one attribute A per FD to evaluate).

---

### 10.5 Converting a Relation to 3NF — Decomposition

```
Algorithm: Convert-To-3NF (basic decomposition approach)

1. For each transitive dependency X → A (where X is not a super key,
   and A is not prime), identified via Check-3NF above:
     a. Create a new relation R1(X, A, [any other attributes
        transitively dependent on exactly X]).
     b. Remove A (and those other attributes) from the original relation,
        but KEEP X in the original relation (X remains as a foreign key
        reference to the new relation).
2. Repeat until every remaining relation passes the 3NF check.

(Note: this basic approach is intuitive and works for straightforward cases.
 The rigorous, ALWAYS-correct, dependency-preserving algorithm — the
 "3NF Synthesis Algorithm" using the canonical cover — is presented in
 full in Module 13, since it also needs the lossless-join theory from
 that module to be properly justified.)
```

**Applying this to Section 10.3's example:**

Transitive dependency found: `dept_id → dept_name` (via `emp_id → dept_id`).

**Decomposition:**

```
Employee(emp_id, emp_name, dept_id)      -- dept_id KEPT as a foreign key
Department(dept_id, dept_name)
```

**Verification:**
- `Employee(emp_id, emp_name, dept_id)`: candidate key `{emp_id}`. FDs: `emp_id → emp_name` (X is super key ✓), `emp_id → dept_id` (X is super key ✓). **3NF satisfied.**
- `Department(dept_id, dept_name)`: candidate key `{dept_id}`. FD: `dept_id → dept_name` (X is super key ✓, since `dept_id` is now the ENTIRE key of this smaller relation). **3NF satisfied.**

---

### 10.6 Visual — Before and After 3NF Decomposition

```
BEFORE (violates 3NF — transitive dependency):
┌─────────────────────────────────────────────┐
│  Employee(emp_id, emp_name, dept_id, dept_name)│
│  Key: {emp_id}                                  │
│                                                  │
│  emp_id ──▶ dept_id ──▶ dept_name                │
│      (transitive: emp_id → dept_name via         │
│       non-key attribute dept_id)                  │
└─────────────────────────────────────────────┘

AFTER (3NF-compliant decomposition):
┌───────────────────────────┐     ┌───────────────────────┐
│ Employee                     │     │ Department               │
│ emp_id (PK)                  │────▶│ dept_id (PK)              │
│ emp_name                      │ FK  │ dept_name                 │
│ dept_id (FK)                   │     │                           │
└───────────────────────────┘     └───────────────────────┘
```

---

### 10.7 The "Escape Hatch" in Action — A Worked Example Where 3NF Allows What BCNF Wouldn't

This is the **single most important example type** for understanding the 3NF/BCNF distinction, so let's build it carefully (we'll return to this exact example in Module 11).

**Given:**

```
Table: StudentAdvisor(student_id, subject, advisor)

Business rules:
  - A student can study multiple subjects, each with a designated advisor for that subject.
  - Each advisor advises exactly ONE subject (an advisor is a subject specialist).
  - A student can only have ONE advisor per subject (obviously).

FDs:
  {student_id, subject} → advisor      (a student's advisor for a given subject)
  advisor → subject                     (each advisor specializes in exactly one subject)
```

**Finding candidate keys:**
- `{student_id, subject}⁺`: directly gives `advisor` via the first FD → `{student_id, subject, advisor}` = full set. This is a super key; is it minimal? `{student_id}` alone: no FD fires, closure = `{student_id}`, not full. `{subject}` alone: no FD fires (nothing has `subject` alone as LHS), not full. So `{student_id, subject}` is a **candidate key**.
- `{student_id, advisor}⁺`: `advisor → subject` gives `{student_id, advisor, subject}` = full set. Minimal? `{student_id}` alone: not full (checked above). `{advisor}` alone: `advisor → subject` gives `{advisor, subject}`, not full (missing student_id). So `{student_id, advisor}` is **also a candidate key**.

**Candidate keys:** `{student_id, subject}` and `{student_id, advisor}`.
**Prime attributes:** `student_id`, `subject`, `advisor` — **all three attributes are prime** (each appears in at least one candidate key)!

**Checking 3NF for `advisor → subject`:**
- Is `advisor` (the LHS) a super key? No — `{advisor}` alone only gives `{advisor, subject}`, not the full set (missing `student_id`).
- Is `subject` (the RHS) a prime attribute? **Yes** — `subject` is part of the candidate key `{student_id, subject}`.
- Since clause (b) is satisfied (RHS is prime), **this FD does NOT violate 3NF**, even though the LHS is not a super key!

**Conclusion: `StudentAdvisor` IS in 3NF**, thanks entirely to the escape hatch. But notice: `advisor → subject` is still a real, non-trivial dependency where the determinant is NOT a super key — this is **exactly the pattern that BCNF forbids without any escape hatch** (Module 11). This single example will become our reference point for demonstrating precisely how BCNF is stricter than 3NF.

---

### 10.8 Dependency Preservation — A Forward-Looking Note

**Why does the escape hatch exist from a deeper theoretical standpoint?**
If we tried to "fix" `StudentAdvisor` the way BCNF would insist (decomposing further because `advisor → subject` has a non-super-key determinant), we'd get something like:

```
Advisor_Subject(advisor, subject)
Student_Advisor(student_id, advisor)
```

But now, checking whether a given `(student_id, subject)` pair is valid (i.e., "is this student actually assigned to this subject with this specific advisor?") requires **joining both tables back together** — the original constraint `{student_id, subject} → advisor` is no longer directly enforceable by a simple key/constraint on any single table; it can only be checked by reconstructing the join. This is called **losing dependency preservation**, and it's a real, practical cost (you'd need an expensive cross-table trigger or application-level check to enforce a rule that used to be a simple key constraint). 3NF's escape hatch exists precisely to **guarantee that a dependency-preserving decomposition is always possible** — a guarantee BCNF does not make. We formalize "dependency preservation" fully in Module 13.

---

### 10.9 3NF — Comparison Table (2NF vs 3NF)

| Aspect | 2NF | 3NF |
|---|---|---|
| **Core requirement** | 2NF + no partial dependency of non-prime attrs | 2NF + no transitive dependency (with an escape hatch for prime RHS attributes) |
| **Targets which dependency pattern?** | Partial dependency (non-prime attr depends on PART of composite key) | Transitive dependency (non-prime attr depends on ANOTHER non-prime attr) |
| **Has an "escape hatch"?** | No | **Yes** — RHS being prime excuses an otherwise-bad dependency |
| **Guarantees dependency preservation?** | N/A at this stage | **Yes, always achievable** (this is the theoretical payoff of the escape hatch) |
| **Stricter alternative** | 3NF | **BCNF** (Module 11) — removes the escape hatch entirely |

---

### 10.10 Industry Implementation Notes

- **3NF is widely regarded as the "sweet spot" normal form for most transactional (OLTP) production schemas** — it eliminates the vast majority of practical redundancy and anomaly risk while still guaranteeing dependency preservation (Section 10.8), making it easier to enforce business rules directly via simple key constraints.
- Many real-world design guides and industry style guides explicitly recommend **"normalize to 3NF, then selectively denormalize only where profiling shows a genuine performance need"** (Module 14) — rather than chasing BCNF or higher normal forms by default, precisely because of the dependency-preservation trade-off discussed above.
- ORM-generated schemas (Django, Rails, etc.) typically produce 3NF-compliant structures by default when models and their foreign-key relationships are defined following standard conventions.

---

### 10.11 Advantages of 3NF Compliance

- Eliminates transitive-dependency-driven redundancy and its associated anomalies.
- **Always achievable while preserving dependency enforcement** — a guarantee not available at the BCNF level (Module 13).
- Widely considered a practical, "good enough for most systems" stopping point — many production schemas target 3NF specifically, rather than pushing to BCNF or higher.

### 10.12 Disadvantages / Trade-offs

- The escape hatch means 3NF can still permit a **small amount of residual redundancy** in specific edge cases (like `StudentAdvisor` in Section 10.7) — BCNF exists precisely to close this gap, at the cost of dependency preservation.
- As with all normalization, more tables generally means more joins — the same core trade-off from Modules 1, 7, and 9 applies here too.

---

### 10.13 Best Practices

- Always compute the **canonical cover** (Module 5) before checking 3NF — it makes the "for every FD, check clause a or b" process unambiguous and far less error-prone than working with a bloated raw FD set.
- Always find **all** candidate keys (Module 6) before determining prime/non-prime status — an incomplete candidate-key search can cause you to wrongly flag an FD as a 3NF violation when the escape hatch actually applies (or vice versa).
- When you find a 3NF violation, trace it back to whether it's a **partial** dependency (should've been caught at 2NF — check your 2NF work again) or a genuine **transitive** dependency (the 3NF-specific pattern) — this distinction matters for correctly diagnosing what went wrong.

---

### 10.14 Common Mistakes

- Forgetting the escape hatch entirely and flagging every non-super-key-determinant FD as a violation, without checking whether the RHS is prime (Section 10.7's exact scenario).
- Confusing 3NF with BCNF — assuming "X is a super key" is the ONLY allowed condition (that's BCNF's rule, Module 11) and forgetting clause (b) exists specifically to distinguish the two.
- Checking 3NF against a raw, non-canonical FD set, missing violations hidden inside a composite or redundant FD.
- Not re-verifying candidate keys in the DECOMPOSED tables after fixing one 3NF violation — a decomposition can change which attributes are prime/non-prime in the new, smaller tables, potentially surfacing (or resolving) further issues.

---

### 10.15 Edge Cases

- **A relation where the escape hatch is the ONLY thing preventing a "violation"** (Section 10.7) — a classic, must-know example type for interviews and exams; always double check prime/non-prime status carefully in these cases.
- **A relation with multiple candidate keys where a "transitive-looking" dependency is actually between two candidate keys** (not a true non-key-to-non-key transitive chain) — re-examine Module 2, Section 2.10's precise definition ("Y is NOT a key") before concluding a dependency is genuinely transitive.
- **A relation already automatically in 3NF because it has no non-prime attributes at all** — similar to the 2NF edge case (Module 9, Section 9.14), if every attribute is prime, clause (b) trivially satisfies every FD.

---

### 10.16 Performance Implications

- 3NF decomposition, like 2NF, trades reduced redundancy/storage for additional join cost at query time.
- Because 3NF (via its escape hatch) sometimes tolerates a small amount of residual redundancy that BCNF would eliminate, 3NF-compliant schemas can occasionally be **marginally faster to query** (fewer joins in specific edge cases like `StudentAdvisor`) while still being "normalized" in a meaningful, anomaly-reducing sense — part of why 3NF remains the practical default target for many production systems (Section 10.10).

---

### 10.17 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | Define 3NF; identify a simple transitive dependency violation. |
| Beginner | Explain the difference between 2NF's and 3NF's target dependency patterns. |
| Intermediate | Given a relation with a transitive dependency, decompose it into 3NF. |
| Senior SDE-2/SDE-3 | Explain the "escape hatch" (clause b) with a worked example, and explain WHY it exists in terms of dependency preservation — this is a very common "do you actually understand the theory" filter question. |
| GATE | "Is this relation in 3NF?" numerical problems, frequently including at least one escape-hatch trap. |

---

### 10.18 Chapter Summary

- **3NF** requires 2NF **plus**: for every non-trivial FD `X → A`, either `X` is a super key, **or** `A` is a prime attribute (the "escape hatch").
- This targets **transitive dependencies** — non-prime attributes depending on other non-prime attributes rather than directly on a key.
- The escape hatch exists because it **guarantees dependency-preserving decomposition is always achievable** at the 3NF level — a guarantee BCNF does not make.
- A relation can satisfy 3NF while still having an FD whose determinant is not a super key, **as long as** the dependent attribute is prime — this is the most commonly tested 3NF/BCNF distinction.
- 3NF is widely regarded as the practical "sweet spot" for most production OLTP schemas.

---

### 10.19 Cheat Sheet

```
3NF = 2NF + for every non-trivial FD X → A:
        (a) X is a super key,  OR
        (b) A is a PRIME attribute (the "escape hatch")

TARGETS: transitive dependencies (non-prime → non-prime, via a non-key attribute)

KEY DIFFERENCE FROM BCNF: 3NF has the escape hatch (clause b); BCNF does NOT.

WHY THE ESCAPE HATCH EXISTS: guarantees a DEPENDENCY-PRESERVING decomposition
                             is always possible (BCNF sometimes can't guarantee this)

FIX: decompose along the transitive dependency, X → A becomes its own table,
     X remains in the original as a foreign key.
```

---

### 10.20 MCQs

**Q1.** 3NF requires that for every non-trivial FD `X → A`:
a) X must always be a super key
b) X must be a super key OR A must be a prime attribute
c) A must always be a prime attribute
d) X must be a candidate key specifically (not just any super key)

<details><summary>Answer</summary>b) X is a super key OR A is prime</details>

**Q2.** The "escape hatch" in 3NF exists primarily to guarantee:
a) Faster query performance
b) That a dependency-preserving decomposition is always achievable
c) That every relation has exactly one candidate key
d) Atomicity of all attributes

<details><summary>Answer</summary>b) Dependency-preserving decomposition is always achievable</details>

**Q3.** In `Employee(emp_id, emp_name, dept_id, dept_name)` with `emp_id → dept_id → dept_name`, and only candidate key `{emp_id}`, does `dept_id → dept_name` violate 3NF?
a) No, because dept_id is a super key
b) No, because dept_name is prime
c) Yes, because dept_id is not a super key AND dept_name is not prime
d) 3NF does not apply to this relation

<details><summary>Answer</summary>c) Yes, this is a genuine 3NF violation (a transitive dependency)</details>

**Q4.** Which normal form REMOVES the "escape hatch" that 3NF provides?
a) 1NF
b) 2NF
c) BCNF
d) 4NF only

<details><summary>Answer</summary>c) BCNF</details>

**Q5.** True or False: A relation can be in 3NF even if it has a non-trivial FD whose determinant is not a super key.
a) False, this is never allowed in 3NF
b) True, as long as the dependent attribute is a prime attribute
c) True, but only if the relation has no candidate keys
d) False, this only applies to BCNF

<details><summary>Answer</summary>b) True, when the escape hatch (dependent attribute is prime) applies</details>

---

### 10.21 Exercises

**Beginner:**
1. Define 3NF and transitive dependency in your own words.
2. Identify the transitive dependency in `Book(isbn, title, publisher_id, publisher_name)` assuming `isbn` is the only candidate key. Decompose into 3NF.

**Intermediate — 3NF Violation Detection:**
3. Given `R(A,B,C,D)` with candidate key `{A}` and FDs `A → B`, `B → C`, `A → D`, check whether R is in 3NF. If not, decompose it.
4. Revisit the `StudentAdvisor(student_id, subject, advisor)` example from Section 10.7. Verify the candidate keys and prime attributes yourself from scratch, and confirm the escape hatch reasoning independently.

**Advanced — Escape Hatch Practice:**
5. Construct your OWN original example (different from `StudentAdvisor`) of a relation that satisfies 3NF via the escape hatch but would violate the stricter BCNF rule (you'll verify this against BCNF formally in Module 11 — for now, just identify the FD whose determinant is not a super key but whose RHS is prime).
6. `R(A,B,C,D,E)` with candidate keys `{A,B}` and `{A,D}`, and FD `B → C`. Determine whether `C` is prime or non-prime, and whether `B → C` violates 3NF.

**Case Study:**
7. A `Payroll` table stores `emp_id, emp_name, dept_id, dept_name, dept_budget`. Assuming `emp_id` is the only candidate key, identify ALL transitive dependencies (there may be more than one distinct chain), and produce a complete 3NF decomposition.

**Whiteboard Design:**
8. Design a 3NF-compliant schema for a `Real_Estate_Listing` system that currently stores `listing_id, agent_id, agent_name, agent_office_city, property_address, price` in one flat table. Identify the transitive dependencies driving the redundancy and produce the decomposition.

**GATE-style Numerical:**
9. `R(A,B,C,D,E,F)` with candidate key `{A}`, and FDs `A → B`, `A → C`, `C → D`, `D → E`, `A → F`. Identify all FDs that violate 3NF (if any), and produce the fully decomposed 3NF schema.

**Interview Questions:**
- *Beginner:* "What is a transitive dependency, and how does 3NF address it?"
- *Intermediate:* "Walk me through checking whether a given relation is in 3NF."
- *Senior:* "Explain 3NF's escape hatch clause, and why it exists in terms of dependency preservation — then explain how BCNF differs."

**Practice links:**
- GeeksforGeeks — Third Normal Form (3NF): https://www.geeksforgeeks.org/dbms/third-normal-form-3nf/
- GateOverflow — 3NF tagged questions: https://gateoverflow.in/

---

**End of Module 10.**

*Next up — Module 11: Boyce-Codd Normal Form (BCNF) — removing the 3NF escape hatch entirely, understanding exactly why BCNF is stricter, and confronting the lossless-join vs dependency-preservation trade-off head-on. Reply "next" / "continue" when ready.*
