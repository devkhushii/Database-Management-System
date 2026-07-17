# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 13: Decomposition

> **Where we are:** Modules 8–12 walked through every normal form, and along the way we informally decomposed relations whenever we found a violation. This module now makes that process **fully rigorous**: we formally define what it means for a decomposition to be "safe" (lossless-join) and "complete" (dependency-preserving), give test algorithms for both properties, and finally present the **3NF Synthesis Algorithm** — the fully general, always-correct procedure that ties together canonical cover (Module 5), keys (Module 6), and everything else into one clean pipeline.

---

### 13.1 What Is Decomposition?

**Definition:**
**Decomposition** is the process of replacing a relation `R` with a set of smaller relations `R1, R2, ..., Rn` (each a subset of R's attributes) such that every attribute of R appears in at least one `Ri`, typically done to eliminate redundancy and satisfy a target normal form.

**Why does decomposition need formal safety criteria?**
Because it's entirely possible to decompose a relation in a way that **looks** reasonable but actually **destroys information** or **loses the ability to enforce business rules** — both of which are worse than the redundancy problem you were trying to fix in the first place. This module formalizes exactly what "safe" decomposition means.

---

### 13.2 Lossless-Join Decomposition

**Definition:**
A decomposition of `R` into `R1` and `R2` is a **lossless-join decomposition** if, for every valid instance of `R`, joining `R1` and `R2` back together (via natural join) reproduces **exactly** the original relation `R` — no more rows (no spurious tuples) and no fewer rows (no lost information).

Formally: `R1 ⋈ R2 = R` for every valid instance.

**Why is this critical?**
If a decomposition is **lossy**, joining the decomposed tables back together can produce **spurious tuples** — fabricated rows that look plausible but don't represent real facts. This is a **severe data integrity risk**: queries against the decomposed schema could silently return **wrong answers** that appear completely legitimate.

**The Lossless-Join Test (for a 2-way decomposition):**

```
Algorithm: Is-Lossless-Join(R, R1, R2, F)

Let R1 ∩ R2 = the set of attributes common to both R1 and R2.

The decomposition {R1, R2} is LOSSLESS if and only if AT LEAST ONE
of the following holds (using the FD set F):

    (R1 ∩ R2) → R1        (the common attributes functionally determine
                             ALL of R1's attributes)
                                    OR
    (R1 ∩ R2) → R2        (the common attributes functionally determine
                             ALL of R2's attributes)

(Test by computing the closure of (R1 ∩ R2) using F, and checking
 if it contains all of R1's attributes OR all of R2's attributes.)
```

**Why does this test work (intuition)?** If the common attributes functionally determine everything in one of the two pieces, then that piece can always be uniquely "reconstructed" from the shared attributes — there's no ambiguity about which row of R1 to match with which row of R2 during the join, so no spurious combinations can arise.

---

### 13.3 Worked Example — Testing Lossless-Join

**Given:**

```
R(A, B, C, D)
F = { A → B, A → C, C → D }

Proposed decomposition:
R1(A, B, C)
R2(C, D)
```

**Test:**
```
R1 ∩ R2 = {C}

Compute C⁺ using F: C → D gives {C, D}. Does C determine anything about A or B? No FD has C alone
(or with D) as LHS producing A or B. So C⁺ = {C, D}.

Is C⁺ ⊇ all attributes of R1 = {A,B,C}?  → {C,D} ⊇ {A,B,C}?  NO.
Is C⁺ ⊇ all attributes of R2 = {C,D}?    → {C,D} ⊇ {C,D}?    YES!
```

Since `(R1 ∩ R2) → R2` holds (the common attribute `C` determines all of R2's attributes), **this decomposition IS lossless-join.**

**A contrasting LOSSY example:**

```
R(A, B, C)
F = { A → B }

Proposed (bad) decomposition:
R1(A, C)
R2(B, C)
```

```
R1 ∩ R2 = {C}
Compute C⁺ using F: no FD has C as (part of) its LHS at all. C⁺ = {C}.

Is C⁺ ⊇ {A, C} (all of R1)?  → {C} ⊇ {A,C}? NO.
Is C⁺ ⊇ {B, C} (all of R2)?  → {C} ⊇ {B,C}? NO.
```

**Neither condition holds → this decomposition is LOSSY.** Joining `R1(A,C)` and `R2(B,C)` back together on `C` would produce spurious combinations of A and B values that never actually existed together in the original data (since C, the join attribute, doesn't functionally pin down either A or B).

**Concrete illustration of the spurious tuple problem:**

```
Original R:
A=1, B=1, C=1
A=2, B=2, C=1

Decomposed R1(A,C):        Decomposed R2(B,C):
A=1, C=1                    B=1, C=1
A=2, C=1                    B=2, C=1

Rejoining R1 ⋈ R2 on C=1 gives ALL combinations:
A=1,B=1,C=1  ✓ (real)
A=1,B=2,C=1  ✗ SPURIOUS — never existed in original R!
A=2,B=1,C=1  ✗ SPURIOUS — never existed in original R!
A=2,B=2,C=1  ✓ (real)
```

This is exactly why the lossless-join test is essential, and why it must be checked **every single time** a decomposition is proposed, not assumed.

---

### 13.4 Extending the Test to Multi-Way Decompositions

For decomposing into **more than two** relations, the simplest, most reliable rigorous approach in practice is: **decompose one FD-violation at a time** (as we've done in every previous module), verifying the lossless-join property at **each individual 2-way split**. Since each individual split is verified lossless, and lossless-join composes (a lossless decomposition of a lossless decomposition remains lossless), the overall multi-way result is guaranteed lossless. This is exactly why the BCNF decomposition algorithm (Module 11, Section 11.5) works one FD-violation at a time recursively, rather than trying to test an N-way lossless-join condition directly (which requires a more complex "chase" algorithm not covered in depth in most standard courses).

---

### 13.5 Dependency Preservation

**Definition:**
A decomposition of `R` into `R1, R2, ..., Rn` is **dependency-preserving** if the union of the functional dependencies that hold on each individual `Ri` (when projected from the original F) is **equivalent** to the original FD set `F` — meaning every FD in F can still be verified by checking constraints on the individual decomposed tables **alone**, without needing to join them back together.

**Formal definition using projection:** The projection of F onto a set of attributes `Ri`, written `Πri(F)`, is the set of all FDs `X → Y` (with `X, Y ⊆ Ri`) that are implied by F (i.e., derivable via `F⁺`). The decomposition is dependency-preserving if:

```
(Πr1(F) ∪ Πr2(F) ∪ ... ∪ Πrn(F))⁺ = F⁺
```

**Why is this important (recap from Module 10 and 11)?**
If a decomposition is NOT dependency-preserving, some original business rule can only be checked by **joining tables back together** — an expensive, awkward way to enforce what used to be a simple, local constraint (like a `UNIQUE` key check on one table). This was exactly the problem demonstrated with `StudentAdvisor`'s BCNF decomposition (Module 11, Section 11.4).

**The Dependency Preservation Test:**

```
Algorithm: Is-Dependency-Preserving(F, R1, R2, ..., Rn)

1. Let F' = the union of Πri(F) for all i (i.e., collect every FD
   from the original F set — or more precisely, every FD DERIVABLE
   from F — whose attributes fit ENTIRELY within some single Ri).
2. Compute F'⁺ (the closure of this collected FD set).
3. Compute F⁺ (the closure of the original FD set).
4. If F'⁺ = F⁺ → dependency-preserving.
   If F'⁺ ⊊ F⁺ (F' is missing some FD that F implies) → NOT dependency-preserving.

(In practice, a simpler, equivalent check: for every FD X → Y in the
 ORIGINAL F, verify that Y ⊆ X⁺ when computing X's closure using ONLY
 the FDs that fit within a single Ri — i.e., can this FD still be
 verified using only "local," single-table information?)
```

---

### 13.6 Worked Example — Testing Dependency Preservation

**Revisiting the StudentAdvisor BCNF decomposition (Module 11, Section 11.4):**

```
Original: StudentAdvisor(student_id, subject, advisor)
F = { {student_id, subject} → advisor,  advisor → subject }

Decomposed:
Advisor_Subject(advisor, subject)
Student_Advisor(student_id, advisor)
```

**Test:** Does `Πr1(F) ∪ Πr2(F)` still imply `{student_id, subject} → advisor`?

- `Πr1(F)` (FDs fitting inside `Advisor_Subject(advisor, subject)`): `advisor → subject` fits entirely (both attributes are in this table). ✓
- `Πr2(F)` (FDs fitting inside `Student_Advisor(student_id, advisor)`): does any FD fit here? `{student_id, subject} → advisor` does NOT fit (subject isn't in this table). No FD with LHS/RHS entirely within `{student_id, advisor}` is derivable from F (checking: `{student_id}⁺` using original F = just `{student_id}`, since nothing has `student_id` alone as LHS; `{advisor}⁺` = `{advisor, subject}`, which doesn't help produce anything purely within `{student_id, advisor}` beyond trivial FDs).
- Collected `F' = { advisor → subject }` only.
- Does `F'⁺` imply `{student_id, subject} → advisor`? Computing `{student_id, subject}⁺` using ONLY `F' = {advisor → subject}`: nothing fires (advisor isn't in the starting set, and nothing in F' has `student_id` or `subject` alone as LHS). `{student_id,subject}⁺ = {student_id, subject}` — does NOT include `advisor`.

**Conclusion: dependency NOT preserved.** This formally confirms, via the rigorous test, exactly what we argued informally in Module 11 — the original constraint `{student_id, subject} → advisor` can no longer be verified using only the decomposed tables individually; it requires a join.

---

### 13.7 Lossless-Join vs Dependency-Preserving — These Are INDEPENDENT Properties

This is one of the most important high-level takeaways of this entire module, and a frequent source of confusion:

| | Lossless-Join | Dependency-Preserving |
|---|---|---|
| **What it guarantees** | No information is lost or fabricated when rejoining | All original constraints can be checked locally, without joins |
| **Guaranteed by BCNF decomposition?** | **Always** | **Not always** |
| **Guaranteed by 3NF synthesis (Section 13.9)?** | **Always** | **Always** |
| **Can a decomposition have one without the other?** | **Yes** — a decomposition can be lossless but NOT dependency-preserving (StudentAdvisor's BCNF case), or (in poorly designed cases) fail to be lossless at all |

**The take-home message:** BCNF **prioritizes** eliminating all redundancy, accepting that dependency preservation might be sacrificed. **3NF's synthesis algorithm** (next section) achieves **both** properties simultaneously, always — which is precisely the theoretical payoff of 3NF's escape hatch (Module 10, Section 10.2/10.8).

---

### 13.8 The 3NF Synthesis Algorithm — Full Procedure

This is the **crown jewel algorithm** of this entire Part — it takes a raw relation and FD set, and produces a 3NF decomposition that is **guaranteed, always, to be both lossless-join AND dependency-preserving.**

```
Algorithm: 3NF-Synthesis(R, F)

STEP 1: Compute the CANONICAL COVER Fc of F (Module 5).

STEP 2: For each functional dependency X → A in Fc,
        create a relation schema Ri = X ∪ A.
        (If multiple FDs share the same LHS X, e.g. X→A and X→B,
         MERGE them into a single relation Ri = X ∪ {A, B} — this
         is simply applying the Union rule from Module 4 to avoid
         creating unnecessarily many tiny tables.)

STEP 3: If NONE of the relations created in Step 2 contains a candidate
        key of the ORIGINAL relation R, add ONE additional relation
        consisting of just some candidate key of R.
        (This step guarantees the LOSSLESS-JOIN property — without it,
         the decomposition could theoretically fail to reconstruct R
         correctly. This extra table is often small, sometimes even a
         "pure key" table with no other attributes.)

STEP 4 (optional cleanup): If any relation Ri created above is a subset
        of another relation Rj, remove the redundant smaller Ri
        (it adds nothing not already captured by Rj).

RESULT: The resulting set of relations {R1, R2, ..., Rn} is a
        3NF decomposition of R that is GUARANTEED lossless-join
        AND dependency-preserving.
```

**Why does this algorithm guarantee both properties (intuition)?**
- **Dependency-preserving:** by construction, every FD in the canonical cover becomes (or is merged into) its own relation — so every original constraint has a "home" table where it can be checked locally via a simple key constraint.
- **Lossless-join:** Step 3's safeguard ensures a candidate key of the original R is always present somewhere in the decomposition, which is a sufficient (proven) condition for guaranteeing the full set can be losslessly rejoined.

---

### 13.9 Full Worked Example — 3NF Synthesis from Scratch

**Given:**

```
R(A, B, C, D, E)
F = { A → BC, CD → E, B → D, E → A }

(This is our familiar running example from Modules 3, 5, and 6!)
```

**Step 1 — Canonical Cover:** From Module 5-style analysis (splitting RHS, checking extraneous attrs/redundancy — let's verify quickly): Split `A → BC` into `A→B, A→C`. Full split set: `{A→B, A→C, CD→E, B→D, E→A}`. Check for extraneous attributes and redundancy (following Module 5's algorithm) — in this case, this set turns out to already be minimal (no attribute removals or FD removals succeed; verifying this is left as a good self-check exercise using the Module 5 algorithm). 

```
Canonical Cover Fc = { A→B, A→C, CD→E, B→D, E→A }
```

**Step 2 — Create one relation schema per FD (merging shared LHS):**
```
A→B, A→C  share LHS "A"  → merge into: R1(A, B, C)
CD→E                      → R2(C, D, E)
B→D                        → R3(B, D)
E→A                         → R4(E, A)
```

**Step 3 — Check: does any Ri contain a candidate key of the original R?**
From Module 6, Section 6.3, we already determined R's candidate keys are: `{A}, {E}, {C,D}, {B,C}`. Does `R1(A,B,C)` contain `{A}`? **Yes!** So Step 3's safeguard is already satisfied — no extra "pure key" table is needed.

**Step 4 — Check for redundant subset relations:** Is any Ri a subset of another? `R3(B,D)` — is this a subset of R1, R2, or R4? No. `R4(E,A)` — subset of any other? No. All four relations are needed.

**Final 3NF Synthesis Result:**
```
R1(A, B, C)     -- key: A
R2(C, D, E)     -- key: C,D
R3(B, D)        -- key: B
R4(E, A)        -- key: E
```

This decomposition is **guaranteed** (by the algorithm's correctness proof) to be both lossless-join and dependency-preserving — every original FD (`A→B`, `A→C`, `CD→E`, `B→D`, `E→A`) is directly checkable as a local key constraint in one of these four tables.

---

### 13.10 Comparing 3NF Synthesis Output to Ad Hoc Decomposition

**Important, honest caveat:** the 3NF synthesis algorithm's output (Section 13.9) can sometimes look like **more tables than a human designer might intuitively produce** by decomposing FD-violation-by-FD-violation (as we did throughout Modules 9–11). Both approaches are theoretically valid 3NF decompositions — the synthesis algorithm's specific advantage is its **guarantee** of both properties simultaneously via a fully mechanical procedure, valuable especially for automated tooling or when a rigorous, provable guarantee is required (e.g., in an exam answer or safety-critical system design), whereas ad hoc decomposition requires the designer to manually verify both properties after the fact (as we did in Sections 13.3 and 13.6).

---

### 13.11 Verification Techniques — Summary Checklist

Whenever you (or an automated tool) propose ANY decomposition, always verify:

```
✓ LOSSLESS-JOIN CHECK:
    For each 2-way split, does (R1 ∩ R2) → R1  OR  (R1 ∩ R2) → R2 hold?

✓ DEPENDENCY-PRESERVATION CHECK:
    Does the union of FDs projectable onto each Ri, closed under F,
    equal the original F's closure?

✓ NORMAL FORM CHECK:
    Does EVERY resulting Ri actually satisfy the TARGET normal form
    (2NF/3NF/BCNF/4NF, per Modules 9–12)? (Decomposing to fix ONE
    violation doesn't automatically guarantee no OTHER violation
    remains in the same or another resulting table.)
```

---

### 13.12 Industry Implementation Notes

- The 3NF synthesis algorithm is the **theoretical basis** for automated schema-design and normalization tools (some academic CASE tools implement it directly); most real-world engineers apply the underlying reasoning informally (find FD violations, decompose, sanity-check for lossless-join and dependency preservation) rather than running the full mechanical algorithm by hand on production schemas.
- **Real migrations and refactors** (e.g., splitting a monolithic `users` table during a microservices decomposition) implicitly rely on lossless-join thinking — "can we reconstruct the full user record correctly from the split services' data?" — even without invoking the formal FD/closure machinery explicitly.
- Database schema-comparison and migration-review tools sometimes flag "orphaned" or "unreachable" foreign key relationships, which is a practical, tooling-level echo of the lossless-join verification concept.

---

### 13.13 Advantages

- Formal lossless-join and dependency-preservation tests remove all guesswork and intuition from decomposition safety — a decomposition either passes these tests or it doesn't.
- The 3NF synthesis algorithm provides a **fully general, always-correct** procedure — valuable both as an exam/interview tool and as the theoretical foundation for schema-design automation.
- Explicitly separating "is this lossless?" from "is this dependency-preserving?" (Section 13.7) clarifies exactly what trade-off is being made whenever BCNF sacrifices the latter.

### 13.14 Disadvantages / Limitations

- The 3NF synthesis algorithm can produce a **larger number of smaller tables** than an experienced designer might choose ad hoc, potentially at some query-convenience cost (Section 13.10).
- Lossless-join and dependency-preservation testing for a full N-way decomposition (beyond simple 2-way splits) requires more advanced techniques (the "chase" algorithm) not covered in full depth in most standard courses/interviews, though the FD-violation-by-FD-violation approach (Section 13.4) sidesteps this for most practical cases.

---

### 13.15 Best Practices

- **Always** test lossless-join for any proposed decomposition, even one that "looks obviously fine" — the spurious-tuple risk (Section 13.3) is subtle and easy to miss by eyeballing.
- When BCNF decomposition sacrifices dependency preservation, use the 3NF synthesis algorithm as an alternative if preserving all original constraints as local, checkable rules is a priority for your system.
- Document explicitly, for any production schema decomposition, whether dependency preservation holds — and if not, what compensating enforcement mechanism (trigger, application check) covers the gap (as flagged already in Module 11, Section 11.9).

---

### 13.16 Common Mistakes

- Assuming any decomposition along a "sensible-looking" FD is automatically lossless — always run the explicit `(R1∩R2)→R1 OR (R1∩R2)→R2` test.
- Confusing lossless-join with dependency preservation — they are **independent properties**, and a decomposition can have either, both, or (if done carelessly) neither.
- Forgetting Step 3 of the 3NF synthesis algorithm (ensuring some candidate key of the original relation appears in the result) — skipping this step can, in rare cases, produce a decomposition that fails to be lossless.
- Stopping after achieving lossless-join and dependency preservation without ALSO verifying that every resulting relation genuinely satisfies the target normal form (Section 13.11's checklist) — these are three separate checks, not one.

---

### 13.17 Edge Cases

- **A decomposition where BOTH lossless-join conditions hold simultaneously** (`(R1∩R2)→R1` AND `(R1∩R2)→R2`) — this means R1 and R2 actually share the exact same key structure; a perfectly safe, "doubly-guaranteed" lossless decomposition.
- **A 3NF synthesis result where Step 3's extra key-only table turns out to be necessary** — occurs when none of the FD-derived relations happen to contain any candidate key of the original R; a good, instructive edge case to construct deliberately as a practice exercise.
- **Multi-way decompositions where testing lossless-join requires the "chase" technique** (beyond simple pairwise checks) — flagged here for awareness; a genuinely advanced topic beyond this handbook's practical scope, but worth knowing the name of for further study.

---

### 13.18 Performance Implications

- A lossless-join decomposition guarantees **correctness** upon rejoining but says nothing about **join performance** — reconstructing a full entity view from a heavily-decomposed schema (like the 3NF synthesis output in Section 13.9, with four separate tables) may require multiple joins, a real cost that must be weighed against the redundancy-elimination benefit (this is exactly Module 14's subject).
- Dependency-preserving decompositions save on **write-time** enforcement cost (simple local key checks vs. expensive cross-table triggers) — this is a genuine, measurable operational benefit, not just a theoretical nicety.

---

### 13.19 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | Define lossless-join decomposition in your own words. |
| Intermediate | Given a proposed 2-way decomposition, test whether it's lossless using the `(R1∩R2)→R1 OR R2` rule. |
| Senior SDE-2/SDE-3 | Explain the difference between lossless-join and dependency-preserving, using the StudentAdvisor BCNF example; explain when you'd use the 3NF synthesis algorithm instead of ad hoc BCNF decomposition. |
| GATE | Classic "is this decomposition lossless?" and "is this decomposition dependency-preserving?" numerical problems — very high frequency question types. |

---

### 13.20 Chapter Summary

- **Lossless-join decomposition** guarantees no information is lost or spuriously fabricated when rejoining decomposed tables — tested via `(R1∩R2) → R1 OR (R1∩R2) → R2`.
- **Dependency preservation** guarantees every original FD can still be checked using only local, single-table constraints — tested by comparing the closure of the union of projected FDs against the original F's closure.
- These two properties are **independent** — BCNF decomposition always achieves lossless-join but not always dependency preservation; the **3NF synthesis algorithm** achieves **both**, always.
- The 3NF synthesis algorithm: compute canonical cover → one relation per FD (merging shared LHS) → ensure a candidate key of the original relation is present → remove redundant subset relations.
- Always verify lossless-join, dependency preservation, AND target-normal-form compliance as three **separate** checks for any proposed decomposition.

---

### 13.21 Cheat Sheet

```
LOSSLESS-JOIN TEST (2-way split R1, R2):
  (R1 ∩ R2) → R1   OR   (R1 ∩ R2) → R2      (using F's closure)

DEPENDENCY PRESERVATION TEST:
  closure of (FDs projectable onto each Ri, combined) == closure of original F

LOSSLESS-JOIN and DEPENDENCY PRESERVATION are INDEPENDENT properties.
  BCNF decomposition:  ALWAYS lossless, NOT ALWAYS dependency-preserving
  3NF synthesis:        ALWAYS lossless AND ALWAYS dependency-preserving

3NF SYNTHESIS ALGORITHM:
  1. Compute canonical cover Fc
  2. One relation per FD in Fc (merge same-LHS FDs)
  3. Ensure some candidate key of original R appears in the result
     (add a pure-key table if not)
  4. Remove any relation that's a subset of another
```

---

### 13.22 MCQs

**Q1.** A decomposition of R into R1 and R2 is lossless-join if:
a) R1 and R2 have the same number of attributes
b) `(R1∩R2) → R1` or `(R1∩R2) → R2` holds
c) R1 and R2 are both in BCNF
d) R1 and R2 share no attributes

<details><summary>Answer</summary>b) `(R1∩R2) → R1` or `(R1∩R2) → R2`</details>

**Q2.** Which algorithm guarantees BOTH lossless-join AND dependency preservation, always?
a) BCNF decomposition
b) 3NF synthesis algorithm
c) 4NF decomposition
d) Canonical cover alone

<details><summary>Answer</summary>b) 3NF synthesis algorithm</details>

**Q3.** Is BCNF decomposition guaranteed to be dependency-preserving?
a) Yes, always
b) No, not always
c) Only if the relation has one candidate key
d) Dependency preservation is irrelevant to BCNF

<details><summary>Answer</summary>b) No, not always</details>

**Q4.** In the 3NF synthesis algorithm, what is Step 3's purpose?
a) To remove extraneous attributes
b) To ensure a candidate key of the original relation is present, guaranteeing lossless-join
c) To merge all relations into one
d) To check for 4NF violations

<details><summary>Answer</summary>b) Guarantee lossless-join by including a candidate key</details>

**Q5.** If a decomposition fails the lossless-join test, what practical problem can result?
a) Slower queries only
b) Spurious tuples appearing when the tables are rejoined
c) The tables become impossible to index
d) No practical problem, it's purely theoretical

<details><summary>Answer</summary>b) Spurious tuples on rejoin — a real data integrity risk</details>

---

### 13.23 Exercises

**Beginner:**
1. Define lossless-join decomposition and dependency preservation in your own words, explaining how they differ.
2. Given `R(A,B,C)` with `F={A→B}`, test whether decomposing into `R1(A,B)` and `R2(A,C)` is lossless-join.

**Intermediate — Lossless-Join Testing:**
3. `R(A,B,C,D)` with `F={A→B, B→C}`. Test whether `R1(A,B,C)` and `R2(A,D)` form a lossless-join decomposition.
4. `R(A,B,C,D,E)` with `F={A→BC, CD→E}`. Test whether `R1(A,B,C)` and `R2(C,D,E)` form a lossless-join decomposition.

**Advanced — Dependency Preservation Testing:**
5. For the decomposition in Exercise 4, ALSO test whether it's dependency-preserving. Show your work using the projection-based test from Section 13.5.
6. Revisit the `StudentAdvisor` BCNF decomposition (Section 13.6). Independently re-derive the conclusion that dependency preservation is lost, showing every step of the closure computation yourself.

**3NF Synthesis Practice:**
7. `R(A,B,C,D,E)` with `F = {A→B, B→C, CD→E}`. Run the FULL 3NF synthesis algorithm from scratch: compute canonical cover, create relations per FD, check for a candidate key, and produce the final decomposition.
8. Construct your own relation and FD set (at least 4 attributes, 3+ FDs) where 3NF synthesis's Step 3 (adding a pure-key relation) is actually NECESSARY — i.e., none of the FD-derived relations happen to contain a candidate key of the original relation.

**Case Study:**
9. A team proposes decomposing a `Orders` table into `OrderHeader` and `OrderShipping` based on a quick glance at the columns, without formally testing lossless-join or dependency preservation. As a senior reviewer, write out the specific two tests you would insist they run before approving this schema change, using the algorithms from this module.

**GATE-style Numerical:**
10. `R(A,B,C,D)` with `F = {AB→C, C→D, D→A}`. A proposed decomposition is `R1(A,B,C)` and `R2(C,D)`. Test both lossless-join and dependency preservation, showing all work.

**Interview Questions:**
- *Beginner:* "What does it mean for a decomposition to be lossless-join?"
- *Intermediate:* "How would you test whether a proposed decomposition preserves all original functional dependencies?"
- *Senior:* "Walk me through the 3NF synthesis algorithm, and explain why it guarantees both lossless-join and dependency preservation, unlike BCNF decomposition."

**Practice links:**
- GeeksforGeeks — Lossless Decomposition & Dependency Preserving Decomposition: https://www.geeksforgeeks.org/dbms/dependency-preserving-decomposition/
- GateOverflow — Decomposition tagged questions: https://gateoverflow.in/

---

**End of Module 13.**

*This completes the FULL theoretical core of Part 5 — from raw functional dependencies through every normal form and rigorous decomposition theory. Next up — Module 14: Denormalization — the deliberate, informed reversal of some of this normalization for performance, covering when and how to do it responsibly. Reply "next" / "continue" when ready.*
