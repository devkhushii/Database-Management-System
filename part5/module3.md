# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 3: Attribute Closure

> **Where we are:** Module 2 gave us the vocabulary of functional dependencies. Now we build the **algorithmic engine** that turns a raw FD set into something actionable: candidate keys, super keys, and the ability to answer "does this FD logically follow from the ones I already have?" This single algorithm — **Attribute Closure** — is the computational backbone of everything from here through Module 13.

---

### 3.1 What Is Attribute Closure?

**Definition (informal):**
The **closure of an attribute set X**, denoted `X⁺`, is the set of **all attributes that can be functionally determined by X**, given a set of functional dependencies `F`.

**Definition (formal):**
Given a relation `R` and a set of functional dependencies `F` over `R`'s attributes, the closure `X⁺` (with respect to `F`) is the maximal set of attributes `A` such that `X → A` can be **logically derived** from `F` using Armstrong's Axioms (formally proven in Module 4).

**Why does it exist?**
Because a raw list of FDs (e.g., `A → B`, `B → C`, `A → D`) doesn't directly tell you everything X determines — some dependencies are *implied* rather than stated explicitly. `X⁺` computes the **complete, exhaustive set** of everything reachable from X by chaining dependencies.

**Analogy — Reachability in a Graph:**
Think of attributes as nodes in a directed graph, and each FD `X → Y` as an edge (or set of edges) from X to each attribute in Y. Computing `X⁺` is exactly like a **graph reachability / BFS traversal** starting from node X — you're finding every node you can reach by following dependency edges, possibly through several hops.

```
   A ──▶ B ──▶ C
   │
   └───▶ D

   A⁺ = {A, B, C, D}   (everything reachable from A)
```

---

### 3.2 What Problem Does Closure Solve?

Attribute closure directly answers several practical questions that come up constantly in design and in interviews:

| Question | How closure answers it |
|---|---|
| "Is `X → Y` implied by my FD set `F`?" | Compute `X⁺`. If `Y ⊆ X⁺`, then yes, `F ⊨ X → Y`. |
| "Is `X` a super key of `R`?" | Compute `X⁺`. If `X⁺` contains **all** attributes of `R`, then `X` is a super key. |
| "Is `X` a candidate key of `R`?" | `X` is a super key (above) **and** no proper subset of `X` is also a super key (minimality). |
| "What is the minimal cover of `F`?" | Closure is used repeatedly inside the minimal cover algorithm (Module 5). |
| "Can I safely decompose `R` without losing information?" | Closure is used to test lossless-join decomposition (Module 13). |

This is why closure is often called the **single most important computational tool** in relational dependency theory — nearly every subsequent algorithm in this Part calls closure as a subroutine.

---

### 3.3 The Attribute Closure Algorithm

**Input:** A set of attributes `X`, and a set of functional dependencies `F` over relation `R`.
**Output:** `X⁺` — the closure of X with respect to F.

**Algorithm (standard iterative version):**

```
Algorithm: Attribute-Closure(X, F)

1. result ← X                       // start with X itself
2. repeat
3.     old_result ← result
4.     for each FD (α → β) in F:
5.         if α ⊆ result:
6.             result ← result ∪ β
7. until result = old_result         // stop when no more attributes get added
8. return result
```

**Why does this work (intuition)?**
We keep scanning the FD list, and every time we find an FD whose left-hand side is *already* fully contained in what we've collected so far, we know its right-hand side must also be reachable — so we add it. We repeat this scanning until a full pass adds nothing new (a "fixed point"). This guarantees we've found *everything* reachable, not just what's reachable in one pass — because an added attribute might unlock a *new* FD on a later pass.

**Termination guarantee:** Since `R` has a finite number of attributes, and `result` only grows (never shrinks) each iteration, the algorithm must terminate — in the worst case, `result` grows to include every attribute in `R`.

---

### 3.4 Worked Example — Step by Step

**Given:**
```
R(A, B, C, D, E, F)

F = {
    A → B,
    B → C,
    A → D,
    D → E,
    {C, E} → F
}

Compute: A⁺
```

**Trace:**

| Step | `result` before | FD checked | LHS ⊆ result? | Action | `result` after |
|---|---|---|---|---|---|
| Init | — | — | — | `result ← {A}` | `{A}` |
| Pass 1 | `{A}` | `A → B` | Yes (`{A} ⊆ {A}`) | add B | `{A, B}` |
| Pass 1 | `{A, B}` | `B → C` | Yes (`{B} ⊆ {A,B}`) | add C | `{A, B, C}` |
| Pass 1 | `{A, B, C}` | `A → D` | Yes | add D | `{A, B, C, D}` |
| Pass 1 | `{A,B,C,D}` | `D → E` | Yes | add E | `{A,B,C,D,E}` |
| Pass 1 | `{A,B,C,D,E}` | `{C,E} → F` | Yes (`{C,E} ⊆ result`) | add F | `{A,B,C,D,E,F}` |
| Pass 2 | `{A,...,F}` | (check all FDs again) | nothing new | — | `{A,B,C,D,E,F}` (unchanged → stop) |

**Result:** `A⁺ = {A, B, C, D, E, F}` — the entire relation. This means **`A` alone is a super key of R** (Section 3.6).

**Visual dependency graph for this example:**

```
        ┌────┐
        │ A  │
        └─┬──┘
     ┌────┴────┐
     ▼         ▼
   ┌───┐     ┌───┐
   │ B │     │ D │
   └─┬─┘     └─┬─┘
     ▼         ▼
   ┌───┐     ┌───┐
   │ C │     │ E │
   └─┬─┘     └─┬─┘
     └────┬────┘
          ▼
        ┌───┐
        │ F │
        └───┘
```

---

### 3.5 A Second Worked Example — Closure That Does NOT Cover Everything

**Given:**
```
R(A, B, C, D, E)

F = {
    A → B,
    C → D
}

Compute: A⁺
```

**Trace:**

| Step | `result` | FD checked | LHS ⊆ result? | Action |
|---|---|---|---|---|
| Init | `{A}` | — | — | — |
| Pass 1 | `{A}` | `A → B` | Yes | add B → `{A,B}` |
| Pass 1 | `{A,B}` | `C → D` | No (`C ⊄ {A,B}`) | skip |
| Pass 2 | `{A,B}` | both FDs re-checked | nothing new | stop |

**Result:** `A⁺ = {A, B}`. Note that `E` never appears on **any** right-hand side in `F` — attributes that never appear as a dependent in any FD can *never* be added to any closure (except trivially, if they're in the starting set X itself). This is a useful shortcut: scan `F` first, and any attribute absent from every RHS is either part of every candidate key, or simply unreachable and must be added to X manually if you're testing key-ness.

Since `A⁺ ≠ {A,B,C,D,E}` (the full attribute set), **`A` alone is NOT a super key** of R.

---

### 3.6 Using Closure for Candidate Key Detection

**Super Key — Definition:**
A set of attributes `K` is a **super key** of relation `R` if `K⁺` = all attributes of `R`. In other words, `K` functionally determines every attribute in the relation (including itself, trivially).

**Candidate Key — Definition:**
A set of attributes `K` is a **candidate key** if:
1. `K` is a super key (`K⁺` = all attributes of R), **and**
2. `K` is **minimal** — no proper subset of `K` is also a super key.

**Algorithm: Finding whether a given set K is a candidate key**

```
Algorithm: Is-Candidate-Key(K, R, F)

1. if Attribute-Closure(K, F) ≠ all attributes of R:
2.     return "K is NOT a super key" (and therefore not a candidate key)
3. for each proper subset K' of K:
4.     if Attribute-Closure(K', F) = all attributes of R:
5.         return "K is a super key, but NOT minimal → NOT a candidate key"
6. return "K is a candidate key"
```

**Worked example:**

```
R(A, B, C, D)
F = { A → B, B → C, C → D }

Test: is {A} a candidate key?
```

- `A⁺`: `A → B` gives `{A,B}`; `B → C` gives `{A,B,C}`; `C → D` gives `{A,B,C,D}`. Full attribute set reached → `{A}` **is a super key**.
- Is `{A}` minimal? Its only proper subset is `∅`, and `∅⁺` = `∅` (nothing) ≠ full set. So yes, `{A}` is minimal.
- **Conclusion: `{A}` is a candidate key.**

**A composite-key example:**

```
R(A, B, C, D)
F = { {A,B} → C, {A,B} → D, C → A }

Test: is {A, B} a candidate key? Is {B, C} also a candidate key?
```

- `{A,B}⁺`: `{A,B} → C` gives `{A,B,C}`; `{A,B} → D` gives `{A,B,C,D}`. Full set reached → super key. Check subsets: `{A}⁺ = {A}` (no FD has LHS ⊆ {A} except trivial), not full. `{B}⁺ = {B}`, not full. So `{A,B}` is **minimal** → **candidate key**.
- `{B,C}⁺`: `C → A` gives `{A,B,C}`; now `{A,B} → C` and `{A,B} → D` both apply (`{A,B} ⊆ {A,B,C}`) → add C (already there) and D → `{A,B,C,D}`. Full set reached → super key. Check subsets: `{B}⁺ = {B}` not full; `{C}⁺`: `C → A` gives `{A,C}`, not full. So `{B,C}` is minimal → **also a candidate key**.

This shows a relation can have **multiple candidate keys** — a point formalized further in Module 6.

---

### 3.7 Super Key vs Candidate Key — Comparison Table

| Aspect | Super Key | Candidate Key |
|---|---|---|
| **Definition** | Any attribute set whose closure = all attributes of R | A *minimal* super key (no proper subset is also a super key) |
| **Minimality required?** | No | Yes |
| **Count in a relation** | Potentially many (any superset of a candidate key is also a super key) | Also potentially many, but each is minimal |
| **Relationship** | Every candidate key is a super key | Not every super key is a candidate key |
| **Example** (R(A,B,C,D), A→B→C→D chain) | `{A}`, `{A,B}`, `{A,C}`, `{A,B,C,D}` are all super keys | Only `{A}` is a candidate key (the minimal one) |

**Visual — Super Keys vs Candidate Keys:**

```
                    ┌───────────────────────────────┐
                    │        ALL SUPER KEYS           │
                    │   {A}, {A,B}, {A,C}, {A,B,C},   │
                    │   {A,B,D}, {A,B,C,D}, ...        │
                    │  ┌───────────────────────┐      │
                    │  │   CANDIDATE KEYS        │      │
                    │  │        {A}              │      │
                    │  │   (minimal super keys)  │      │
                    │  └───────────────────────┘      │
                    └───────────────────────────────┘
```

---

### 3.8 Minimal Keys — Formal Notes

"Minimal key" is simply another common name for **candidate key** — emphasizing the minimality property. A key is minimal if **removing any single attribute from it destroys the super-key property** (i.e., the resulting closure no longer covers all of R's attributes).

**Practical test for minimality:** For a candidate super key `K` with attributes `{k1, k2, ..., kn}`, check each `K - {ki}` (K with one attribute removed) individually. If **any** of these reduced sets is still a super key, then `K` was not minimal to begin with.

> **Common Mistake:** Only checking *some* proper subsets instead of systematically checking all single-attribute removals (and, if needed, further removals) — this can lead to incorrectly labeling a non-minimal key as a candidate key. Always be exhaustive, especially in GATE-style problems with larger attribute sets.

---

### 3.9 Prime and Non-Prime Attributes (Preview)

While full treatment is in Module 6, it's worth noting the direct connection to closure here:

- **Prime attribute:** any attribute that is part of **at least one** candidate key.
- **Non-prime attribute:** any attribute that is part of **no** candidate key.

Since finding candidate keys requires computing closures, this classification is a direct downstream product of the closure algorithm. This distinction becomes essential when defining 2NF and 3NF precisely (Modules 9–10), since those normal forms are defined in terms of dependencies **on prime vs non-prime attributes**.

---

### 3.10 Industry Implementation Notes

- No production DBMS computes attribute closures at runtime — this is purely a **design-time** analytical tool used by database designers (often manually, or with schema-design/CASE tools) before the schema is implemented.
- Some advanced schema design and refactoring tools (e.g., academic tools like normalization calculators, or features in certain data modeling suites) implement the closure algorithm to auto-suggest candidate keys from a supplied FD set — useful when migrating or auditing legacy schemas.
- In practice, engineers rarely write out closures formally for production schemas with dozens of columns — but the *underlying reasoning* ("does this small set of columns actually determine everything else uniquely?") is exactly what's happening informally when a senior engineer picks a primary key or evaluates a proposed unique constraint.
- Because closure computation is set-based and mechanical, it maps naturally to code — a common systems-design or scripting exercise is to implement `Attribute-Closure(X, F)` as a small program (useful for auditing large FD sets in bulk, e.g., during a legacy data migration project).

---

### 3.11 Advantages of the Closure-Based Approach

- Converts "is this a key?" from an intuition-based judgment into a mechanical, always-correct algorithm.
- Scales to arbitrarily large FD sets and attribute counts — unlike manual inspection, closure computation is systematic.
- Forms the reusable subroutine for nearly every later algorithm in this Part (minimal cover, all normal form checks, lossless decomposition verification).
- Provides a rigorous way to prove **whether one FD is implied by others** — critical for detecting redundant dependencies (Module 5).

### 3.12 Disadvantages / Limitations

- Purely mechanical — it's only as correct as the input FD set. If the FD set is wrong or incomplete (garbage in), the resulting "candidate keys" will be wrong too (garbage out).
- For relations with **many** attributes and FDs, manually tracing closures (as in the worked examples above) becomes tedious and error-prone — this is exactly the kind of task suited to automation/scripting for large legacy schemas.
- Closure alone doesn't tell you *why* a dependency exists (the business reason) — it's a computational tool, not a substitute for domain understanding.

---

### 3.13 Best Practices

- Always compute the closure of a **candidate** key set starting from scratch (don't assume — verify with the algorithm), especially in exams and interviews where a wrong assumption cascades into every subsequent answer.
- When testing for candidate keys among several possible attribute sets, first eliminate any set whose closure clearly doesn't reach all attributes (fails super-key test) before spending time checking minimality.
- Use the "attributes that never appear on any RHS must be in every candidate key" shortcut (Section 3.5) to speed up candidate-key hunting on paper/exams.
- Keep a clear table (like the worked-example traces above) when computing closures by hand — it prevents skipped-FD errors, which are the single most common source of mistakes.

---

### 3.14 Common Mistakes

- Stopping the closure algorithm after the **first pass** instead of iterating until a fixed point — this misses dependencies that only become "unlocked" in a later pass (see Section 3.4, where `{C,E} → F` only fires after both C and E have already been added).
- Forgetting that `X ⊆ X⁺` always (X trivially determines itself) — a common bookkeeping slip when writing out `result` sets.
- Testing super-key-ness correctly, but then skipping the minimality check entirely and declaring any super key a "candidate key."
- Confusing **candidate key** with **primary key** — every primary key is a candidate key, but a relation may have several candidate keys, only one of which is *chosen* (by the designer) as the primary key. Closure identifies *all* candidate keys; picking the primary key is a design decision layered on top.

---

### 3.15 Edge Cases

- **Closure of the empty set:** `∅⁺` is either `∅`, or in rare cases, the set of attributes determined by any FD with an empty (or always-true) left-hand side — in standard relational FD theory, this essentially never adds anything, so `∅⁺ = ∅` in virtually all practical problems.
- **Relation with a single candidate key equal to all attributes:** If no non-trivial FDs exist at all, the only candidate key is the full attribute set itself (every attribute is needed to determine every attribute, trivially).
- **Multiple candidate keys sharing attributes:** As shown in Section 3.6, candidate keys can overlap (e.g., `{A,B}` and `{B,C}` both being candidate keys, sharing `B`) — don't assume candidate keys are disjoint.
- **Cyclical-looking FD chains:** `A → B`, `B → C`, `C → A` — here `A⁺ = B⁺ = C⁺ = {A,B,C,...}` (whatever else follows), meaning `A`, `B`, and `C` are all individually candidate keys (assuming minimality holds) — a classic "trick" pattern in exam problems.

---

### 3.16 Performance Implications

- While closure itself isn't a runtime operation, the **key chosen based on closure analysis** directly affects runtime performance: a well-chosen primary key (usually the smallest / simplest candidate key) minimizes index size and join cost.
- Composite candidate keys (multi-attribute) generally lead to **larger indexes** and more expensive joins/foreign key references than single-attribute keys — this is a common reason designers introduce a surrogate key (e.g., an auto-incrementing `id`) even when a natural composite candidate key exists, a trade-off revisited in Module 15.

---

### 3.17 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | Compute a simple closure by hand given 3–4 FDs. |
| Beginner | Explain the difference between super key and candidate key. |
| Intermediate | Given an FD set, find **all** candidate keys of a relation systematically. |
| Intermediate | Explain why closure computation must iterate to a fixed point, not stop after one pass. |
| Senior SDE-2/SDE-3 | Given a large, messy legacy table's column list and business rules, derive the FD set and use closure reasoning to justify a proposed primary key — including trade-offs of composite vs surrogate keys. |
| GATE | Numerical closure computation problems, often with 5–8 attributes and multiple FDs, asking for `X⁺` or the full set of candidate keys. |

---

### 3.18 Chapter Summary

- **Attribute closure `X⁺`** = the complete set of attributes functionally determined by X, given an FD set F — computed by iteratively applying FDs until nothing new is added (a fixed point).
- Closure is used to test: whether an FD is implied by F, whether a set is a **super key** (closure = all attributes), and whether a set is a **candidate key** (super key + minimal).
- Every candidate key is a super key, but not every super key is minimal enough to be a candidate key.
- A relation can have **multiple candidate keys**, possibly overlapping.
- Attributes that never appear on any FD's right-hand side must be included in every candidate key.
- Closure is the reusable engine behind minimal cover (Module 5), normal form checks (Modules 8–12), and lossless decomposition verification (Module 13).

---

### 3.19 Cheat Sheet

```
X⁺  = closure of X w.r.t. F
    = repeatedly apply: if α ⊆ result and (α→β) ∈ F, then result ∪= β
    = stop when a full pass adds nothing new

Super Key   : X⁺ = ALL attributes of R
Candidate Key: X is a super key AND no proper subset of X is a super key (minimal)
Prime Attribute     : belongs to at least one candidate key
Non-Prime Attribute  : belongs to no candidate key

Shortcut: attribute never on any FD's RHS ⟹ must be in EVERY candidate key
```

---

### 3.20 MCQs

**Q1.** What does `X⁺` represent?
a) The set of attributes X depends on
b) The set of all attributes functionally determined by X
c) The primary key of the relation
d) The set of trivial FDs involving X

<details><summary>Answer</summary>b) The set of all attributes functionally determined by X</details>

**Q2.** A set `K` is a super key if:
a) `K⁺` contains at least one attribute not in K
b) `K⁺` equals the full attribute set of R
c) `K` has only one attribute
d) `K` appears on the left side of every FD

<details><summary>Answer</summary>b) `K⁺` equals the full attribute set of R</details>

**Q3.** Why must the closure algorithm iterate until a "fixed point" rather than stopping after one pass?
a) Because FDs can be applied only once
b) Because newly added attributes in one pass might satisfy the LHS of an FD only checked in a later pass
c) Because the algorithm must run exactly N times
d) There is no such requirement

<details><summary>Answer</summary>b) Newly added attributes can unlock additional FDs on subsequent passes</details>

**Q4.** If an attribute never appears on the right-hand side of any FD in F, what must be true?
a) It's a non-prime attribute
b) It must belong to every candidate key
c) It can never belong to any candidate key
d) It's a trivial attribute

<details><summary>Answer</summary>b) It must belong to every candidate key</details>

**Q5.** A super key that is NOT minimal is:
a) Not a super key at all
b) A candidate key
c) A super key, but not a candidate key
d) A trivial FD

<details><summary>Answer</summary>c) A super key, but not a candidate key</details>

---

### 3.21 Exercises

**Beginner:**
1. Compute `A⁺` for `R(A,B,C,D)` with `F = {A → B, B → C, C → D}`. Is `A` a candidate key?
2. Compute `B⁺` for the same F. Is `B` a super key?

**Intermediate — Closure Problems:**
3. `R(A,B,C,D,E)`, `F = {A → B, A → C, C → D, D → E}`. Compute `A⁺`. List all candidate keys of R.
4. `R(A,B,C,D)`, `F = {AB → C, C → D, D → A}`. Determine if `{A,B}` is a candidate key. Is `{B,C}` also a candidate key?

**Advanced — Candidate Key Detection:**
5. `R(A,B,C,D,E,F)`, `F = {A → BC, CD → E, B → D, E → A}`. Find **all** candidate keys of R. (Hint: check which attributes never appear on any RHS — they must be part of every candidate key.)
6. Prove, using closure, that if `X → Y` and `Y → Z` both hold, then `X → Z` must also hold (this is the Transitivity axiom — you'll see the formal proof in Module 4, but try deriving it yourself using the closure algorithm's logic first).

**Practice Problems (Candidate Key hunting):**
7. `R(A,B,C,D,E)`, `F = {AB → C, D → E, C → A}`. List every minimal super key (candidate key) of R. Show your closure computation for each candidate you test.

**Case Study:**
8. You inherit a legacy `Employees` table with columns `emp_code, ssn, email, full_name, manager_emp_code`. Business rules state: `ssn` uniquely identifies an employee, `email` is also guaranteed unique per employee, and `emp_code` is a company-internal unique identifier. Using closure-style reasoning (informally), identify the candidate keys, and discuss which one you'd choose as the primary key and why (consider Section 3.16's performance notes).

**GATE-style Numerical:**
9. `R(A,B,C,D,E,F,G)`, `F = {A → B, ABCD → E, EF → G, G → A}`. Compute `A⁺`. Is `A` alone a candidate key? If not, find a minimal set that is.

**Interview Questions:**
- *Beginner:* "What's the algorithm to compute the closure of an attribute set?"
- *Intermediate:* "How would you find all candidate keys of a relation given its FD set?"
- *Senior:* "Given a wide legacy table, how would you programmatically derive candidate keys once you've confirmed the FD set with the business team — would you compute closures by hand, or is there a case for automating this for a 40-column table?"

**Practice links:**
- GeeksforGeeks — Attribute Closure & Candidate Key: https://www.geeksforgeeks.org/dbms/attribute-closure-in-dbms/
- GateOverflow — Closure & Candidate Key tagged questions: https://gateoverflow.in/

---

**End of Module 3.**

*Next up — Module 4: Armstrong's Axioms — the formal inference rules (Reflexivity, Augmentation, Transitivity, plus derived rules) that justify every step of the closure algorithm you just learned, complete with proofs. Reply "next" / "continue" when ready.*
