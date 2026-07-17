# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 4: Armstrong's Axioms

> **Where we are:** Module 3 gave us the closure *algorithm* and used it informally. This module gives us the **formal proof system** underneath that algorithm — the set of inference rules that let us rigorously derive new FDs from existing ones, and *prove* that our closure computations are correct rather than just "seeming right." Armstrong's Axioms are named after William W. Armstrong, who proved them **sound and complete** in 1974 — meaning they can derive *exactly* the FDs that logically follow from a given set, no more and no less.

---

### 4.1 What Are Armstrong's Axioms?

**Definition:**
Armstrong's Axioms are a set of **inference rules** for functional dependencies. Given a set of FDs `F` that hold on a relation `R`, these axioms let you derive additional FDs that are **guaranteed to also hold** on `R` — without needing to inspect actual data.

There are **three basic (primary) axioms**, from which **four derived (secondary) rules** can be proven. Together, they form a complete system: any FD that logically follows from `F` can be derived using just these rules.

**Why do they exist / what problem do they solve?**
Without formal axioms, "does this FD follow from that FD set?" would be an ambiguous, potentially data-dependent question. Armstrong's Axioms make FD derivation a **purely symbolic, provable process** — exactly the theoretical justification for why the closure algorithm in Module 3 works correctly.

**Soundness and Completeness (important theoretical concept):**
- **Sound:** every FD derived using the axioms *actually holds* on any relation instance satisfying the original FD set (the axioms never produce a "false" FD).
- **Complete:** every FD that *actually holds* (logically implied by F) can be derived using the axioms (the axioms never miss a valid FD).

Together, soundness + completeness mean: **F⁺ (the closure of the FD set F — every FD implied by F) can be computed exactly by repeatedly applying Armstrong's Axioms.** This is the formal foundation for the attribute closure algorithm from Module 3.

---

## PART A: The Three Basic (Primary) Axioms

### 4.2 Axiom 1 — Reflexivity

**Statement:**
> If `Y ⊆ X`, then `X → Y`.

**In words:** If Y is a subset of X, then X trivially determines Y.

**Why this is obviously true:**
If you know the values of all attributes in X, and Y is a subset of X, you already know the values in Y — there's nothing left to "determine." This is exactly the definition of a **trivial FD** from Module 2.

**Example:**
```
Given: R(A, B, C)
{A, B} ⊇ {A}  →  by Reflexivity: {A, B} → {A}
```

**Proof sketch (semantic, from first principles):**
Let `t1, t2` be any two tuples in `R` such that `t1[X] = t2[X]`. Since `Y ⊆ X`, it follows directly that `t1[Y] = t2[Y]` (because `t1[Y]` and `t2[Y]` are just parts of `t1[X]` and `t2[X]`, which are already equal). This satisfies the formal definition of `X → Y` from Module 2. ∎

---

### 4.3 Axiom 2 — Augmentation

**Statement:**
> If `X → Y`, then `XZ → YZ` for any attribute set `Z`.

(Here `XZ` is shorthand for `X ∪ Z`, and `YZ` for `Y ∪ Z` — standard notation in FD theory.)

**In words:** Adding the same extra attributes to both sides of a known FD preserves the dependency.

**Why does this matter?**
Augmentation lets you "pad" an FD with additional context attributes without breaking it — extremely useful when working with composite keys, since you often need to reason about FDs in the presence of extra attributes.

**Example:**
```
Given: A → B  (holds on R(A, B, C))
By Augmentation (adding C to both sides): AC → BC
```

**Proof sketch:**
Let `t1, t2` be tuples with `t1[XZ] = t2[XZ]`. This means `t1[X] = t2[X]` (since X ⊆ XZ) **and** `t1[Z] = t2[Z]` (since Z ⊆ XZ). Since `X → Y` holds (given), and `t1[X] = t2[X]`, we know `t1[Y] = t2[Y]`. Combined with `t1[Z] = t2[Z]`, we get `t1[YZ] = t2[YZ]`. This is exactly the definition of `XZ → YZ`. ∎

---

### 4.4 Axiom 3 — Transitivity

**Statement:**
> If `X → Y` and `Y → Z`, then `X → Z`.

**In words:** Functional dependency "chains" — if X determines Y, and Y determines Z, then X determines Z directly.

**Why does this matter?**
This is the formal justification for the **transitive dependency** concept introduced in Module 2 (Section 2.10), and it's the single most-used axiom inside the closure algorithm's iterative loop (Module 3).

**Example:**
```
Given: emp_id → dept_id   and   dept_id → dept_name
By Transitivity: emp_id → dept_name
```

**Proof sketch:**
Let `t1, t2` be tuples with `t1[X] = t2[X]`. Since `X → Y` holds, `t1[Y] = t2[Y]`. Since `Y → Z` holds and `t1[Y] = t2[Y]`, it follows that `t1[Z] = t2[Z]`. This is exactly the definition of `X → Z`. ∎

---

### 4.5 Why Only These Three? (Soundness + Completeness, Informally)

Armstrong (1974) proved that **Reflexivity, Augmentation, and Transitivity together are both sound and complete** — meaning any FD that could ever be logically derived from a given FD set can be produced using just these three rules (possibly applied many times in combination). Every other rule you'll see below (Union, Decomposition, Composition, Pseudotransitivity) is **not strictly necessary** — each can be **proven** using only the three basic axioms. They exist purely for **convenience**, because deriving them from scratch every time would be tedious.

---

## PART B: The Four Derived (Secondary) Rules

### 4.6 Derived Rule 1 — Union (a.k.a. Additivity)

**Statement:**
> If `X → Y` and `X → Z`, then `X → YZ`.

**In words:** If X determines Y and X separately determines Z, then X determines Y and Z together.

**Why is this useful?**
It lets you **combine** multiple FDs with the same determinant into a single, consolidated FD — useful for simplifying an FD set before further analysis (and directly relevant to the minimal cover algorithm in Module 5).

**Formal Proof (derived from the 3 basic axioms):**

```
1. X → Y                                   (given)
2. X → Z                                   (given)
3. X → XY                                  (Augmentation on step 1, adding X to both sides:
                                             X → Y  ⟹  XX → YX, and XX = X, YX = XY)
4. XY → YZ                                 (Augmentation on step 2, adding Y to both sides:
                                             X → Z  ⟹  XY → ZY, and ZY = YZ)
5. X → YZ                                  (Transitivity on steps 3 and 4:
                                             X → XY and XY → YZ  ⟹  X → YZ)
∎
```

**Example:**
```
Given: A → B  and  A → C
By Union: A → BC
```

---

### 4.7 Derived Rule 2 — Decomposition (a.k.a. Projectivity)

**Statement:**
> If `X → YZ`, then `X → Y` and `X → Z`.

**In words:** A single FD whose right-hand side is a combined attribute set can be **split** into separate FDs for each part.

**Why is this useful?**
This is the exact **reverse** of Union — useful when you're given a "bundled" FD and need to reason about individual attributes separately (very common when checking full vs partial dependency, Module 2, Sections 2.7–2.8).

**Formal Proof:**

```
1. X → YZ                                  (given)
2. YZ → Y                                  (Reflexivity, since Y ⊆ YZ)
3. YZ → Z                                  (Reflexivity, since Z ⊆ YZ)
4. X → Y                                   (Transitivity on steps 1 and 2)
5. X → Z                                   (Transitivity on steps 1 and 3)
∎
```

**Example:**
```
Given: A → BC
By Decomposition: A → B   and   A → C
```

---

### 4.8 Derived Rule 3 — Composition

**Statement:**
> If `X → Y` and `Z → W`, then `XZ → YW`.

**In words:** Two independent FDs can be "combined side by side" into one FD over the union of their determinants and dependents.

**Why is this useful?**
Useful when merging FD information from two unrelated parts of a schema during analysis — less commonly used directly than Union/Decomposition/Transitivity, but a valuable proof tool and a recognized GATE-syllabus rule.

**Formal Proof:**

```
1. X → Y                                   (given)
2. Z → W                                   (given)
3. XZ → YZ                                 (Augmentation on step 1, adding Z to both sides)
4. YZ → YW                                 (Augmentation on step 2, adding Y to both sides)
5. XZ → YW                                 (Transitivity on steps 3 and 4)
∎
```

**Example:**
```
Given: A → B   and   C → D
By Composition: AC → BD
```

---

### 4.9 Derived Rule 4 — Pseudotransitivity

**Statement:**
> If `X → Y` and `YZ → W`, then `XZ → W`.

**In words:** A generalized version of Transitivity, where the "middle" dependency has extra attributes (Z) tagging along.

**Why is this useful?**
This shows up frequently in **minimal cover** derivations (Module 5) and in more complex candidate key / closure proofs where the chain of dependency isn't a clean `X → Y → Z`, but has extra baggage attached at the middle step.

**Formal Proof:**

```
1. X → Y                                   (given)
2. YZ → W                                  (given)
3. XZ → YZ                                 (Augmentation on step 1, adding Z to both sides)
4. XZ → W                                  (Transitivity on steps 3 and 2:
                                             XZ → YZ and YZ → W ⟹ XZ → W)
∎
```

**Example:**
```
Given: A → B   and   BC → D
By Pseudotransitivity: AC → D
```

---

### 4.10 Summary Table — All Seven Rules

| # | Rule | Statement | Type |
|---|---|---|---|
| 1 | **Reflexivity** | If `Y ⊆ X`, then `X → Y` | Basic |
| 2 | **Augmentation** | If `X → Y`, then `XZ → YZ` | Basic |
| 3 | **Transitivity** | If `X → Y` and `Y → Z`, then `X → Z` | Basic |
| 4 | **Union** | If `X → Y` and `X → Z`, then `X → YZ` | Derived |
| 5 | **Decomposition** | If `X → YZ`, then `X → Y` and `X → Z` | Derived |
| 6 | **Composition** | If `X → Y` and `Z → W`, then `XZ → YW` | Derived |
| 7 | **Pseudotransitivity** | If `X → Y` and `YZ → W`, then `XZ → W` | Derived |

**Visual — dependency between the rules:**

```
                    ┌────────────────────────────────────┐
                    │        THE 3 BASIC AXIOMS            │
                    │  Reflexivity, Augmentation,           │
                    │  Transitivity  (sound + complete)     │
                    └──────────────────┬───────────────────┘
                                       │  used to prove
              ┌────────────┬───────────┼───────────┬────────────┐
              ▼            ▼           ▼           ▼            
         ┌─────────┐  ┌───────────┐ ┌───────────┐ ┌──────────────────┐
         │  Union   │  │Decompos-  │ │Composition│ │Pseudotransitivity │
         │          │  │  ition    │ │           │ │                    │
         └─────────┘  └───────────┘ └───────────┘ └──────────────────┘
```

---

### 4.11 Connecting Axioms Back to the Closure Algorithm (Module 3)

The closure algorithm's core loop — *"if `α ⊆ result` and `(α → β) ∈ F`, then `result ∪= β`"* — is a direct, repeated application of:

1. **Reflexivity** (to justify that everything already in `result` is trivially "determined")
2. **Augmentation** (to justify applying an FD even when `result` contains extra attributes beyond `α`)
3. **Transitivity** (to justify chaining — the reason a "fixed point" iteration is needed, since newly discovered attributes can trigger further FDs)
4. **Union** (implicitly — the algorithm accumulates all newly discovered attributes into one growing `result` set)

This is *why* the closure algorithm is provably correct: every step it performs is licensed by a proven-sound inference rule.

---

### 4.12 Industry Implementation Notes

- Armstrong's Axioms are **not implemented as runtime logic** in any commercial DBMS — like closure, they are a **design-time, pen-and-paper (or whiteboard) reasoning tool**.
- Where they *do* show up in tooling: academic/teaching normalization calculators, and some formal schema-verification research tools that programmatically prove whether a candidate schema satisfies a target normal form, using axiom-based derivation under the hood.
- In real engineering practice, senior engineers rarely write out formal axiom proofs — but the **underlying reasoning patterns** (e.g., "since A determines B, and B determines C, A must determine C — so I don't need to separately verify or index on C") are used informally, constantly, when reviewing schema design proposals.

---

### 4.13 Advantages

- Provides a **rigorous, provable foundation** for every FD-derivation claim — no more "this seems obviously true" hand-waving.
- The soundness + completeness guarantee (Section 4.5) means: if you can't derive an FD using these axioms, it genuinely does **not** follow from your FD set — a powerful negative-result tool (proving something is **not** implied is just as useful as proving it is).
- The derived rules (Union, Decomposition, Composition, Pseudotransitivity) provide practical shortcuts, avoiding the need to re-derive common patterns from the three basic axioms every time.

### 4.14 Disadvantages / Limitations

- Purely symbolic manipulation of FDs — doesn't help you *discover* the initial FD set from business rules (that's a modeling/analysis skill, not an axiom).
- Manual axiom-based proofs can become long and tedious for complex FD sets — in practice, the **iterative closure algorithm** (Module 3) is almost always used instead for day-to-day candidate-key hunting, with axioms reserved for formal proofs, exams, and theoretical justification.
- Doesn't directly extend to Multivalued Dependencies or Join Dependencies (Module 2, Sections 2.11–2.12) — those require their own, separate axiom systems (introduced but not exhaustively covered in this Part).

---

### 4.15 Best Practices

- When asked to "prove" an FD follows from a given set (common in GATE and university exams), always cite **which specific axiom** justifies each step — a bare assertion without axiom citation is typically marked incomplete.
- Prefer using the **derived rules** (Union, Decomposition, etc.) directly once you're comfortable with them — they save time versus re-deriving from the three basics every time, as long as you can cite them correctly if asked to justify.
- Cross-check axiom-based proofs against a closure computation (Module 3) — if `Y ⊆ X⁺`, then `X → Y` must be derivable via the axioms; the two techniques should always agree, and this is a good self-check habit.

---

### 4.16 Common Mistakes

- **Misapplying Augmentation** by adding attributes to only *one* side (e.g., claiming `X → Y` implies `XZ → Y`, which is actually just a weaker, still-valid consequence via Reflexivity/Transitivity — but is **not** what Augmentation itself states; Augmentation specifically adds Z to *both* sides).
- **Confusing Union with Composition** — Union combines two FDs with the **same** determinant (`X → Y`, `X → Z` ⟹ `X → YZ`); Composition combines two FDs with **different** determinants (`X → Y`, `Z → W` ⟹ `XZ → YW`). These are easy to mix up under exam time pressure.
- Assuming the derived rules are "extra power" beyond the three basic axioms — they add **convenience**, not **new derivable FDs**. Anything provable with a derived rule is also provable (just more slowly) with the three basics.
- Forgetting **Decomposition** only applies to the right-hand side — you cannot decompose the left-hand side of an FD in the same way (i.e., `XY → Z` does **not** imply `X → Z` or `Y → Z` individually — this is a very common and important trap).

---

### 4.17 Edge Cases

- **Reflexivity with X = Y:** `X → X` always holds trivially (since `X ⊆ X`) — often overlooked but technically a valid application of Reflexivity.
- **Augmentation with Z overlapping X or Y:** the rule still holds even if `Z` shares attributes with `X` or `Y` — set union naturally absorbs duplicates (`X ∪ Z` just doesn't double-count shared attributes).
- **Pseudotransitivity reducing to plain Transitivity:** if `Z = ∅` in the Pseudotransitivity statement, it reduces exactly to ordinary Transitivity — a good sanity check that the more general rule is consistent with the basic one.
- **Attempting to "decompose" a composite left-hand side:** as flagged in Section 4.16, this is invalid and a classic exam trap — always double check whether decomposition is being (mis)applied to the LHS instead of the RHS.

---

### 4.18 Performance Implications

Armstrong's Axioms themselves have no direct runtime performance impact (they're a design-time proof system). However, the **schema decisions** justified by axiom-based reasoning — which attributes truly form a key, which dependencies are redundant — directly determine index design and query performance downstream, exactly as discussed in Modules 1–3.

---

### 4.19 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | State the three basic axioms (Reflexivity, Augmentation, Transitivity) from memory. |
| Beginner | Give an example use of Transitivity. |
| Intermediate | Prove the Union rule from the three basic axioms, step by step. |
| Intermediate | Explain the difference between Union and Composition with examples. |
| Senior SDE-2/SDE-3 | Given a claimed FD, construct a full axiom-based derivation proof (or show it's *not* derivable), and connect this to why a proposed schema key is (or isn't) valid. |
| GATE | Multi-step derivation proofs; identifying which axiom justifies each line of a given proof; "true/false" statements about whether an FD is implied by a given set. |

---

### 4.20 Chapter Summary

- Armstrong's Axioms are **sound and complete** inference rules for functional dependencies.
- The **three basic axioms** are: **Reflexivity** (`Y⊆X ⟹ X→Y`), **Augmentation** (`X→Y ⟹ XZ→YZ`), and **Transitivity** (`X→Y, Y→Z ⟹ X→Z`).
- The **four derived rules** — **Union**, **Decomposition**, **Composition**, and **Pseudotransitivity** — add convenience but no new derivational power; each is provable from the three basics.
- These axioms are the formal justification for the **closure algorithm** from Module 3.
- A common and important trap: **Decomposition only applies to the RHS**, never the LHS of an FD.

---

### 4.21 Cheat Sheet

```
BASIC AXIOMS
  Reflexivity      : Y ⊆ X          ⟹  X → Y
  Augmentation     : X → Y          ⟹  XZ → YZ
  Transitivity     : X→Y, Y→Z       ⟹  X → Z

DERIVED RULES
  Union            : X→Y, X→Z       ⟹  X → YZ
  Decomposition    : X → YZ         ⟹  X→Y  and  X→Z   (RHS only!)
  Composition      : X→Y, Z→W       ⟹  XZ → YW
  Pseudotransitivity: X→Y, YZ→W     ⟹  XZ → W

Trap: Decomposition does NOT apply to the LHS.
      XY → Z  does NOT imply  X → Z  or  Y → Z
```

---

### 4.22 MCQs

**Q1.** Which of the following is one of the THREE basic Armstrong's Axioms?
a) Union
b) Composition
c) Augmentation
d) Pseudotransitivity

<details><summary>Answer</summary>c) Augmentation</details>

**Q2.** Given `A → B` and `A → C`, which rule directly gives `A → BC`?
a) Composition
b) Union
c) Decomposition
d) Pseudotransitivity

<details><summary>Answer</summary>b) Union</details>

**Q3.** Given `XY → Z`, which of the following is a valid conclusion?
a) `X → Z`
b) `Y → Z`
c) Neither (a) nor (b) can be concluded in general
d) Both (a) and (b)

<details><summary>Answer</summary>c) Neither can be concluded — Decomposition applies only to the RHS, not the LHS</details>

**Q4.** Which rule is used to prove: "If `A→B` and `BC→D`, then `AC→D`"?
a) Transitivity
b) Union
c) Pseudotransitivity
d) Reflexivity

<details><summary>Answer</summary>c) Pseudotransitivity</details>

**Q5.** Armstrong's Axioms are described as "sound and complete." What does "complete" mean here?
a) Every FD derivable by the axioms actually holds on the relation
b) Every FD that actually logically follows from F can be derived using the axioms
c) The axioms can only be applied a finite number of times
d) The relation has no redundancy

<details><summary>Answer</summary>b) Every FD that logically follows from F can be derived using the axioms</details>

---

### 4.23 Exercises

**Beginner:**
1. State all three basic Armstrong's Axioms from memory, with one original example each.
2. Given `P → Q`, use Augmentation to derive a new FD by adding attribute `R` to both sides.

**Intermediate — Proofs:**
3. Prove the **Decomposition** rule from the three basic axioms, showing each step and which axiom justifies it (model your answer on Section 4.7).
4. Prove the **Composition** rule from the three basic axioms (model your answer on Section 4.8).
5. Given `A → B`, `B → C`, and `C → D`, use Transitivity twice to derive `A → D`. Write out each intermediate step explicitly.

**Advanced — Derivation Challenges:**
6. Given `F = {A → B, B → C, AC → D}`, derive `A → D` using a valid sequence of Armstrong's Axioms. Cite each axiom used.
7. Given `F = {AB → C, C → A}`, determine whether `BC → ABC` can be derived. Show your derivation or explain why it cannot be done.
8. Explain, with a specific counterexample, why `XY → Z` does NOT imply `X → Z` in general (Section 4.16's trap) — construct a small sample relation instance where `XY → Z` holds but `X → Z` clearly fails.

**Whiteboard / Proof-Based Exercise:**
9. A colleague claims: "Since `A → B` and `A → C`, and `B → D`, we can conclude `A → BCD` in one step using a single axiom." Identify exactly which sequence of axioms (there may be more than one) is actually needed, and correct the claim if it oversimplifies the number of steps required.

**GATE-style:**
10. Given `F = {A → B, BC → D, D → E, CE → F}` on relation `R(A,B,C,D,E,F)`, use Armstrong's Axioms (citing each one used) to derive `AC → F`.

**Interview Questions:**
- *Beginner:* "What are Armstrong's three basic axioms?"
- *Intermediate:* "Prove that the Union rule follows from the basic axioms."
- *Senior:* "Why is it important that Armstrong's Axioms are both sound AND complete, from a schema-design correctness standpoint?"

**Practice links:**
- GeeksforGeeks — Armstrong's Axioms in DBMS: https://www.geeksforgeeks.org/dbms/armstrongs-axioms-in-functional-dependency-in-dbms/
- GateOverflow — Armstrong's Axioms tagged questions: https://gateoverflow.in/

---

**End of Module 4.**

*Next up — Module 5: Canonical Cover / Minimal Cover — how to strip an FD set down to its leanest, non-redundant form (removing extraneous attributes and redundant dependencies), using closure and the axioms you now have as tools. Reply "next" / "continue" when ready.*
