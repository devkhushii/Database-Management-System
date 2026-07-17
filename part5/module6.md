# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 6: Keys from Functional Dependencies

> **Where we are:** Modules 2–5 gave us all the raw tools — FDs, closure, axioms, and canonical cover. This module is a focused, **exercise-heavy synthesis module**: we now use those tools systematically to find **all** candidate keys of a relation, distinguish super keys from alternate keys, classify prime vs non-prime attributes, and drill this until it's second nature — because this exact skill is asked in nearly every DBMS interview and GATE paper.

---

### 6.1 Recap — What We Already Know

From Module 3:
- **Super key:** `K⁺` = all attributes of R.
- **Candidate key:** a *minimal* super key (no proper subset is also a super key).

From Module 4:
- Armstrong's Axioms justify every closure step we take.

From Module 5:
- The **canonical cover** gives us the leanest FD set to work with — finding keys is usually easier and less error-prone starting from `Fc` rather than a bloated original `F`.

This module's job: turn "compute one closure" into a **repeatable systematic procedure** for finding *every* candidate key of a relation, no matter how many attributes or FDs are involved.

---

### 6.2 Finding ALL Candidate Keys — The Systematic Approach

Unlike finding *one* candidate key (Module 3), finding **all** candidate keys requires a structured search, because there's no shortcut that guarantees you've found every one just by testing a handful of "obvious" attribute sets.

**Step-by-step systematic algorithm:**

```
Algorithm: Find-All-Candidate-Keys(R, F)

1. Identify attributes that appear ONLY on the LHS of every FD they're involved in,
   or NEVER appear on any RHS at all — call this set L (the "must-have" attributes).
   → These MUST be part of EVERY candidate key (Module 3, Section 3.5's shortcut).

2. Identify attributes that appear ONLY on the RHS of every FD they're involved in,
   or NEVER appear on any LHS at all — call this set N (the "never-need" attributes).
   → These can NEVER be part of any candidate key (except in the degenerate case where
      the relation has NO non-trivial FDs at all).

3. Compute L⁺.
   - If L⁺ = all attributes of R → L itself is the ONLY candidate key. Done.
   - If L⁺ ≠ all attributes of R → we need to add more attributes from the
     "middle" set M = (all attributes) - L - N, and test combinations.

4. For increasing sizes k = 1, 2, 3, ... test each combination of k attributes
   from M added to L:
       compute (L ∪ {combination})⁺
       if it equals all attributes of R → this is a super key; check minimality
       (no smaller combination already found is a subset of it)
   Continue until all minimal super keys (candidate keys) are found.
```

**Why this works:** Step 1's set `L` is a hard *lower bound* — every candidate key must contain it (proven in Module 3). Step 2's set `N` is a hard *exclusion* — no candidate key can contain any of it, because an attribute that never appears on any LHS can never help determine anything beyond what's already reachable, and if it's genuinely not derivable from anything, it can only be "added" to a key by being included directly — but a *minimal* key would never include a purely-dependent attribute unless forced to (this is proven rigorously via the closure test in Step 4). This shrinks the search space from "all possible subsets of R's attributes" (exponential) down to just the middle attributes `M`, which is usually a very small set in practice.

---

### 6.3 Fully Worked Example — Finding All Candidate Keys

**Given:**
```
R(A, B, C, D, E)

F = {
    A → BC,
    CD → E,
    B → D,
    E → A
}
```

*(This is the same FD set used in Module 3's and Module 5's exercises — let's now solve it completely and systematically.)*

**Step 1 — Find L (must-have attributes):**

Scan every FD's LHS and RHS:
```
A → BC    : A is on LHS
CD → E    : C, D are on LHS
B → D     : B is on LHS
E → A     : E is on LHS
```

Every attribute (A, B, C, D, E) appears on **some** LHS somewhere. So at first glance, none is *forced* to be in L by the "never appears as dependent" shortcut alone — we need to check more carefully. Let's check which attributes **never appear on the RHS of any FD**:

```
RHS of A → BC   : B, C
RHS of CD → E   : E
RHS of B → D    : D
RHS of E → A    : A
```

RHS attributes seen: `{B, C, E, D, A}` — actually **every** attribute appears on some RHS too! This means the shortcut from Section 3.5 doesn't immediately eliminate anything here — this FD set is fully "interconnected" (each attribute both determines something and is determined by something). So `L = ∅` initially, and we must search more broadly. This is a good illustration that **the shortcut helps but doesn't always avoid a real search.**

**Step 2 — Compute closures of small candidate sets:**

Let's test `{A}`:
```
A⁺: start {A}
  A → BC applies: add B, C → {A,B,C}
  B → D applies (B ⊆ result): add D → {A,B,C,D}
  CD → E applies (C,D ⊆ result): add E → {A,B,C,D,E}
Full set reached! A⁺ = {A,B,C,D,E}
```

`{A}` is a **super key**. Since it's a single attribute, it's automatically minimal → **`{A}` is a candidate key.**

Let's test `{E}`:
```
E⁺: start {E}
  E → A applies: add A → {A,E}
  A → BC applies (A ⊆ result): add B, C → {A,B,C,E}
  B → D applies: add D → {A,B,C,D,E}
Full set reached! E⁺ = {A,B,C,D,E}
```

`{E}` is also a **super key**, and (single attribute) automatically minimal → **`{E}` is a candidate key.**

Let's test `{B}`:
```
B⁺: start {B}
  B → D applies: add D → {B,D}
  (no FD has LHS ⊆ {B,D} — CD→E needs C too, which we don't have)
B⁺ = {B, D}   ≠ full set
```

`{B}` is **not** a super key.

Let's test `{C}`:
```
C⁺: start {C}
  (no FD has LHS ⊆ {C} alone — CD→E needs D too)
C⁺ = {C}
```

Not a super key.

Let's test `{D}`:
```
D⁺ = {D}   (no FD fires)
```

Not a super key.

Let's test `{C, D}`:
```
{C,D}⁺: start {C,D}
  CD → E applies: add E → {C,D,E}
  E → A applies: add A → {A,C,D,E}
  A → BC applies: add B (C already there) → {A,B,C,D,E}
Full set reached!
```

`{C,D}` is a super key. Check minimality: is `{C}` alone a super key? No (tested above). Is `{D}` alone a super key? No (tested above). So `{C,D}` is **minimal** → **`{C,D}` is a candidate key.**

Let's check `{B,C}` for completeness (since B and C together might also work):
```
{B,C}⁺: start {B,C}
  B → D applies: add D → {B,C,D}
  CD → E applies (C,D ⊆ result): add E → {B,C,D,E}
  E → A applies: add A → {A,B,C,D,E}
Full set reached!
```

`{B,C}` is a super key. Check minimality: `{B}` alone — not a super key (tested). `{C}` alone — not a super key (tested). So `{B,C}` is **minimal** → **`{B,C}` is ALSO a candidate key.**

**Final Answer — All Candidate Keys of R:**
```
{A}, {E}, {C,D}, {B,C}
```

**Sanity check — are there any others?** Any super key must contain enough attributes to reach full closure. We've found all single-attribute candidates (`{A}` and `{E}` work; `{B}`, `{C}`, `{D}` don't). For two-attribute combinations not already containing `A` or `E` (since those would just be non-minimal supersets of `{A}` or `{E}`), we've checked `{C,D}` and `{B,C}` — both work. What about `{B,D}`? Since `{B}⁺ = {B,D}` already (from the `B → D` FD), adding D to B gives nothing new beyond what `B` alone gives — so `{B,D}⁺ = {B,D}` too, **not** a super key. So the complete, verified list is exactly these four.

---

### 6.4 Prime and Non-Prime Attributes — Full Treatment

**Definition (recap from Module 3, now formalized):**

- **Prime attribute:** an attribute that belongs to **at least one** candidate key.
- **Non-prime attribute:** an attribute that belongs to **no** candidate key.

**Applying this to our worked example (Section 6.3):**

```
Candidate keys: {A}, {E}, {C,D}, {B,C}

Attributes appearing in at least one candidate key: A, E, C, D, B
→ ALL five attributes (A, B, C, D, E) are PRIME in this relation!
```

This is an important and slightly counter-intuitive result for students: **an attribute can be non-prime with respect to one candidate key's "coverage" but still be prime overall**, because primality is about membership in *any* candidate key, not all of them. Here, every attribute happens to participate in at least one of the four candidate keys, so there are **zero non-prime attributes** in this particular relation.

**A contrasting example with non-prime attributes:**

```
R(A, B, C, D)
F = { A → B, A → C, A → D }

A⁺ = {A,B,C,D} → {A} is the only candidate key.
Prime attributes: {A}
Non-prime attributes: {B, C, D}
```

Here, B, C, and D are all non-prime — none of them appears in any candidate key (since `{A}` alone is the *only* candidate key).

---

### 6.5 Why Prime vs Non-Prime Matters (Forward Reference)

This classification is not just academic bookkeeping — it is the **exact definition** used in Modules 9 and 10:

| Normal Form | Uses prime/non-prime how |
|---|---|
| **2NF** (Module 9) | No **non-prime** attribute may have a **partial dependency** on any candidate key |
| **3NF** (Module 10) | For every non-trivial FD `X → A`, either X is a super key, OR **A is a prime attribute** (this second clause is the famous "escape hatch" that distinguishes 3NF from the stricter BCNF) |

You cannot correctly check 2NF or 3NF without first correctly identifying **all** candidate keys and classifying every attribute as prime or non-prime — which is exactly why this module exists as a dedicated checkpoint before we move into the normal forms themselves.

---

### 6.6 Alternate Keys

**Definition:**
When a relation has **more than one** candidate key, the database designer must choose **one** of them to be the **primary key**. All the remaining, unchosen candidate keys are called **alternate keys**.

**Example (continuing Section 6.3's relation):**
```
Candidate keys: {A}, {E}, {C,D}, {B,C}

If the designer chooses {A} as the PRIMARY KEY, then:
  {E}, {C,D}, {B,C} are all ALTERNATE KEYS.
```

**Practical note:** In SQL, alternate keys are typically implemented using `UNIQUE` constraints (since only one key can be declared `PRIMARY KEY`, but multiple `UNIQUE` constraints can coexist).

```sql
CREATE TABLE Example (
    A INT PRIMARY KEY,
    E INT UNIQUE,
    C INT,
    D INT,
    B INT,
    UNIQUE (C, D),
    UNIQUE (B, C)
);
```

---

### 6.7 Choosing WHICH Candidate Key Becomes the Primary Key

There's no single "mathematically correct" answer — this is a **design decision** guided by practical considerations (this connects back to Module 1's design trade-offs):

| Consideration | Guidance |
|---|---|
| **Simplicity** | Prefer a single-attribute key over a composite one, if available (easier joins, smaller indexes — Module 3, Section 3.16) |
| **Stability** | Prefer a key whose value never changes over the entity's lifetime (e.g., a generated ID over a mutable email address, even if the email is also a valid candidate key) |
| **Meaninglessness (surrogate keys)** | Many production systems introduce an artificial auto-incrementing `id` as primary key even when a natural composite candidate key exists, specifically to avoid coupling the key to business data that might change (covered further in Module 15) |
| **Null-tolerance** | A candidate key attribute that could theoretically be NULL in edge cases (e.g., an optional government ID) is a poor primary key choice, since primary keys cannot be NULL |
| **Foreign key convenience** | A shorter, simpler primary key is easier and cheaper to reference from other tables via foreign keys |

---

### 6.8 Comparison Table — All Key-Related Terms

| Term | Definition | Relationship to others |
|---|---|---|
| **Super Key** | Attribute set whose closure = all attributes of R | Broadest category; every other key type is a super key |
| **Candidate Key** | A *minimal* super key | Subset of super keys |
| **Primary Key** | The *one* candidate key chosen by the designer to uniquely identify rows | A specific candidate key |
| **Alternate Key** | Any candidate key NOT chosen as primary | The remaining candidate keys |
| **Prime Attribute** | Any attribute belonging to at least one candidate key | Derived from the set of all candidate keys |
| **Non-Prime Attribute** | Any attribute belonging to no candidate key | Derived from the set of all candidate keys |

**Visual — the full hierarchy:**

```
┌───────────────────────────────────────────────────────────┐
│                        SUPER KEYS                            │
│   (any attribute set whose closure = all of R's attributes)  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   CANDIDATE KEYS                        │   │
│  │        (minimal super keys — no proper subset works)    │   │
│  │                                                          │   │
│  │   ┌──────────────┐        ┌───────────────────────┐   │   │
│  │   │ PRIMARY KEY   │        │   ALTERNATE KEY(S)      │   │   │
│  │   │ (chosen ONE)  │        │  (all the other CKs)    │   │   │
│  │   └──────────────┘        └───────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘

Attributes appearing in ANY candidate key  →  PRIME
Attributes appearing in NO candidate key    →  NON-PRIME
```

---

### 6.9 Industry Implementation Notes

- SQL's `PRIMARY KEY` constraint enforces uniqueness + non-nullability on the chosen key, and is typically also used by the storage engine to physically cluster/organize the table (e.g., InnoDB's clustered index in MySQL).
- `UNIQUE` constraints implement alternate keys — enforcing uniqueness without the "there can be only one" restriction that `PRIMARY KEY` has.
- Modern schema design very commonly uses **surrogate keys** (auto-increment integers, or UUIDs) as the primary key even when natural composite candidate keys exist, precisely because of the stability and simplicity considerations from Section 6.7 — this is one of the most common real-world departures from "pure" relational theory, made deliberately and knowingly.
- Distributed systems (sharded databases, microservices) often prefer **UUIDs** over auto-increment integers as surrogate keys, to avoid coordination overhead when generating unique IDs across multiple independent nodes — a physical-design consideration layered on top of the logical-design key choice.

---

### 6.10 Advantages of Rigorous Key Identification

- Prevents accidentally choosing a "key" that isn't actually unique in all valid future data (a subtle bug that often only surfaces much later, in production, as a `UNIQUE` constraint violation or worse, silent data corruption).
- Enables correct 2NF/3NF/BCNF classification (Modules 9–11), which depends entirely on correctly-identified candidate keys and prime/non-prime status.
- Surfaces **alternate keys** that might otherwise be missed, which are valuable for query optimization (an alternate key is a natural indexing candidate) and data integrity (`UNIQUE` constraints catch bugs early).

### 6.11 Disadvantages / Limitations

- Exhaustively finding *all* candidate keys for a relation with many attributes and FDs is combinatorially expensive by hand — practical, real-world schemas rely on the domain-driven "obvious" keys plus spot-checking, rather than a full exhaustive search (which is more of an academic/GATE exercise).
- Like all FD-based analysis, it's only as correct as the underlying FD set — an incomplete FD set can hide a true candidate key or falsely suggest one that doesn't actually hold in all valid data.

---

### 6.12 Best Practices

- Always start with the **must-have set L** (attributes absent from every RHS) — even when it doesn't fully solve the problem outright (as in Section 6.3), it narrows the search space.
- Test single-attribute candidates before multi-attribute ones — smaller candidate keys are both more likely in well-designed schemas and cheaper to verify.
- Once you've found a super key, ALWAYS explicitly verify minimality by checking every proper subset (or at least every single-attribute-removed subset) — don't assume.
- Cross-reference: once you have all candidate keys, list out prime vs non-prime attributes explicitly as a table — this becomes directly reusable input for the 2NF/3NF checks in Modules 9–10.

---

### 6.13 Common Mistakes

- Stopping after finding **one** candidate key and assuming it's the only one — always continue searching the middle attribute set `M` (Section 6.2) unless you've proven no other minimal super key exists.
- Confusing "primary key" (a design choice) with "the only candidate key" (a mathematical fact) — a relation can mathematically have several candidate keys; picking one as primary doesn't erase the others' status as alternate keys.
- Misclassifying a prime attribute as non-prime because it doesn't appear in the *particular* candidate key the student happened to find first, forgetting to check membership across **all** candidate keys.
- Treating foreign keys as if they were part of key-finding — foreign keys are a referential integrity mechanism (relating two tables), entirely distinct from the candidate-key analysis done here on a single relation's own attributes.

---

### 6.14 Edge Cases

- **A relation where every attribute is part of some candidate key** (zero non-prime attributes), as demonstrated in Section 6.3 — this is a legitimate and not-uncommon outcome, especially in tightly interconnected relations.
- **A relation with exactly one candidate key equal to all attributes** — occurs when there are no non-trivial FDs at all; every attribute is required to identify a row uniquely, and there are no alternate keys.
- **Overlapping candidate keys** (candidate keys that share some attributes, like `{C,D}` and `{B,C}` sharing `C` in Section 6.3) — a common and valid outcome; don't assume candidate keys must be disjoint.
- **A candidate key that seems "too large" intuitively** — always trust the closure computation over intuition; sometimes a seemingly large attribute combination is genuinely the minimal way to reach full closure, especially in FD sets with limited connectivity.

---

### 6.15 Performance Implications

- The number and size of candidate keys directly informs index design: alternate keys are prime candidates for `UNIQUE` indexes, which both enforce integrity and speed up lookups on those attributes.
- A relation with many small, overlapping candidate keys (as in Section 6.3) gives the designer real flexibility in choosing a compact primary key — a relation with only one, large composite candidate key offers no such flexibility, and often signals a genuine business complexity (or, sometimes, a modeling error worth revisiting).

---

### 6.16 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | Define primary key, candidate key, alternate key, and explain how they relate. |
| Beginner | Given a small FD set, identify the (single) candidate key. |
| Intermediate | Given an FD set with 2–3 candidate keys, find **all** of them and classify prime/non-prime attributes. |
| Senior SDE-2/SDE-3 | Given a real schema scenario, discuss which candidate key you'd choose as primary and why — covering stability, simplicity, and downstream foreign-key/indexing implications. |
| GATE | Classic "find all candidate keys" numerical problems, often the single highest-frequency GATE DBMS question type. |

---

### 6.17 Chapter Summary

- Finding **all** candidate keys requires a systematic search: identify the "must-have" set (attributes absent from every RHS), then test combinations from the remaining "middle" attributes.
- **Prime attributes** belong to at least one candidate key; **non-prime attributes** belong to none — this classification is essential input for 2NF and 3NF checks (Modules 9–10).
- When multiple candidate keys exist, the designer picks one as the **primary key**; the rest become **alternate keys**, typically enforced via `UNIQUE` constraints.
- Choosing which candidate key becomes primary is a **design decision** guided by simplicity, stability, null-tolerance, and foreign-key convenience — not a purely mathematical one.
- Surrogate keys are a common, deliberate real-world departure from using a "natural" candidate key as primary.

---

### 6.18 Cheat Sheet

```
FINDING ALL CANDIDATE KEYS:
  1. L = attributes never on any RHS (or only on LHS)     → must be in EVERY candidate key
  2. N = attributes never on any LHS (or only on RHS)      → can be in NO candidate key
  3. Compute L⁺. If = all attributes → L is the only candidate key.
  4. Else, test L ∪ {subsets of middle attributes M}, smallest first,
     checking closure = all attributes AND minimality.

PRIME ATTRIBUTE      : in AT LEAST ONE candidate key
NON-PRIME ATTRIBUTE   : in NO candidate key

PRIMARY KEY   : the ONE candidate key chosen by the designer
ALTERNATE KEY  : any OTHER candidate key (usually enforced via UNIQUE constraint)
```

---

### 6.19 MCQs

**Q1.** If a relation has candidate keys `{A}`, `{B}`, and `{C,D}`, and the designer chooses `{A}` as primary key, then `{B}` and `{C,D}` are called:
a) Super keys
b) Foreign keys
c) Alternate keys
d) Non-prime attributes

<details><summary>Answer</summary>c) Alternate keys</details>

**Q2.** An attribute that belongs to NO candidate key is called:
a) A prime attribute
b) A non-prime attribute
c) A foreign key
d) A super key

<details><summary>Answer</summary>b) A non-prime attribute</details>

**Q3.** In SQL, alternate keys are typically enforced using:
a) `FOREIGN KEY`
b) `CHECK`
c) `UNIQUE`
d) `NOT NULL`

<details><summary>Answer</summary>c) `UNIQUE`</details>

**Q4.** Which of the following is NOT a valid reason to prefer a surrogate key over a natural composite candidate key as primary key?
a) Stability over time
b) Simpler joins and smaller indexes
c) It automatically eliminates the need for any UNIQUE constraints
d) Avoiding coupling the key to mutable business data

<details><summary>Answer</summary>c) — surrogate keys don't eliminate the need for UNIQUE constraints on the natural key(s); those remain valuable as alternate keys for integrity</details>

**Q5.** If a relation's candidate keys collectively cover every attribute in the relation, then:
a) The relation has zero prime attributes
b) The relation has zero non-prime attributes
c) The relation cannot be normalized further
d) The relation has only one candidate key

<details><summary>Answer</summary>b) The relation has zero non-prime attributes</details>

---

### 6.20 Exercises

**Beginner:**
1. Define prime and non-prime attribute, and explain how they differ from "candidate key" itself.
2. Given `R(A,B,C)` with `F = {A → B, A → C}`, find the candidate key(s) and classify all attributes as prime or non-prime.

**Intermediate — Candidate Key Problems:**
3. `R(A,B,C,D)`, `F = {A → B, B → C, C → D, D → A}`. Find ALL candidate keys. (Hint: this is the circular chain from Module 5's exercises.)
4. `R(A,B,C,D,E)`, `F = {AB → C, C → D, D → E, E → A}`. Find ALL candidate keys and classify prime/non-prime attributes.

**Advanced — Full Systematic Search:**
5. `R(A,B,C,D,E,F)`, `F = {A → BC, CD → EF, B → D, E → C}`. Using the systematic approach from Section 6.2, find ALL candidate keys. Show your work for at least three tested attribute combinations.
6. Prove that if a relation `R` has NO non-trivial functional dependencies at all, its ONLY candidate key is the full set of all its attributes.

**Whiteboard / Design Decision:**
7. A `Users` table has candidate keys `{user_id}` (an internal auto-increment integer), `{email}`, and `{phone_number}`. As a senior engineer, which would you choose as the primary key, and why? Which would you make `UNIQUE` alternate keys? Consider what happens if a user is allowed to have no phone number on file.

**Case Study:**
8. You're auditing a legacy `Products` table and discover, via closure analysis, that it has candidate keys `{sku}` and `{barcode}`, but `barcode` can occasionally be `NULL` for internally-manufactured products that haven't been assigned a barcode yet. Explain why this disqualifies `barcode` from being the primary key even though it's mathematically a candidate key, and what `UNIQUE` constraint caveat (if any) you'd need in most DBMSs to still enforce uniqueness on `barcode` where it IS present.

**GATE-style Numerical:**
9. `R(A,B,C,D,E)`, `F = {A → B, BC → D, D → E, E → A}`. Find all candidate keys of R and state how many prime attributes the relation has.

**Interview Questions:**
- *Beginner:* "What's the difference between a candidate key and a primary key?"
- *Intermediate:* "How would you find all candidate keys of a relation with a large FD set?"
- *Senior:* "In a real production schema, when would you choose a natural composite candidate key as the primary key instead of introducing a surrogate key?"

**Practice links:**
- GeeksforGeeks — Candidate Key practice: https://www.geeksforgeeks.org/dbms/introduction-of-relational-model-and-codd-rules-in-dbms/
- GateOverflow — Candidate Key / Prime Attribute tagged questions: https://gateoverflow.in/

---

**End of Module 6.**

*Next up — Module 7: Normalization — starting with WHY we normalize at all: the formal treatment of data redundancy, and the update/insert/delete anomalies previewed all the way back in Module 1, now proven rigorously using everything we've built (FDs, closure, keys). Reply "next" / "continue" when ready.*
