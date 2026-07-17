# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 18: Interview Mastery

> **Where we are:** Modules 1–17 built the complete theory and practice of database design, from raw functional dependencies through real-world schemas and OLAP architecture. This module is a **consolidated, cross-topic practice module** — no new theory, just curated, tiered questions spanning everything in this Part, designed to simulate the actual flow of a real interview or exam.

---

### 18.1 How to Use This Module

Work through each section **in order**, attempting every question **before** checking any reference back to earlier modules. For whiteboard-style questions, actually talk through your reasoning out loud (or write it out fully) — the *reasoning process* is what's being evaluated in a real interview, not just the final answer. Where a question maps directly to a concept from an earlier module, the module is cited so you can review if you get stuck — but try first without looking.

---

### 18.2 Functional Dependency Questions (Module 2)

**Q1 (Beginner).** Define functional dependency and give an example from a domain NOT used anywhere in this handbook.

**Q2 (Beginner).** Is `{A} → {A, B}` a valid way to write a non-trivial FD, or does it contain a trivial component? Explain.

**Q3 (Intermediate).** Given `Invoice(invoice_no, invoice_date, customer_id, customer_name, item_code, item_price, quantity)`, propose a reasonable FD set based on typical business rules, and explicitly flag any dependency you're UNCERTAIN about without more business context.

**Q4 (Intermediate).** Explain the difference between a full and a partial functional dependency using an example with a 3-attribute composite key (not a 2-attribute one, to test deeper understanding).

**Q5 (Senior).** A colleague says: "If `A → B` holds in my current 500-row table, it's a valid functional dependency." Explain what's wrong with this reasoning, referencing how FDs should actually be derived.

---

### 18.3 Normalization Questions (Modules 7–12)

**Q6 (Beginner).** List the three classic anomalies and briefly describe each with an original example.

**Q7 (Beginner).** True or false, with justification: "Every relation in 3NF is automatically in 2NF." Then: "Every relation in BCNF is automatically in 3NF."

**Q8 (Intermediate).** Given `R(A,B,C,D)` with candidate key `{A,B}` and FDs `{A,B}→C`, `{A,B}→D`, `C→D`, determine the highest normal form R currently satisfies (1NF through BCNF), and decompose it fully to BCNF if needed.

**Q9 (Intermediate).** Explain, using a concrete example, why a relation can be in BCNF and STILL exhibit redundancy. What normal form addresses this gap?

**Q10 (Senior).** Design a scenario (original, not from this handbook) where a relation satisfies 3NF via the "escape hatch" but fails BCNF. Show the candidate keys, the violating FD, and both the 3NF justification and BCNF decomposition.

**Q11 (Senior).** Explain when you would deliberately STOP at 3NF rather than push to BCNF in a real production system, and what you're trading off by doing so.

---

### 18.4 Candidate Key Questions (Modules 3, 6)

**Q12 (Beginner).** Compute the closure `A⁺` for `R(A,B,C,D)` with `F = {A→B, B→C, C→D}`. Is `A` a candidate key?

**Q13 (Intermediate).** `R(A,B,C,D,E)` with `F = {AB→C, C→D, D→E, E→A}`. Find ALL candidate keys of R. Classify every attribute as prime or non-prime.

**Q14 (Intermediate).** Explain why a relation can have MULTIPLE candidate keys that overlap in their attributes, with an example.

**Q15 (Senior).** A relation has candidate keys `{order_id}` and `{customer_id, order_date, sequence_no}`. As the schema designer, which would you choose as the primary key, and what factors drive that decision?

---

### 18.5 Armstrong's Axioms Problems (Module 4)

**Q16 (Beginner).** State the three basic axioms from memory and give one example use of each.

**Q17 (Intermediate).** Prove, using Armstrong's Axioms (citing each rule used), that given `A→B` and `AB→C`, we can derive `A→C`.

**Q18 (Advanced).** Given `F = {A→B, B→C, AC→D}`, derive `A→D` step by step, citing each axiom.

**Q19 (Advanced).** Explain, with a counterexample, why `XY→Z` does NOT imply `X→Z` or `Y→Z` individually.

---

### 18.6 Closure Problems (Module 3)

**Q20 (Beginner).** Compute `{B}⁺` for `R(A,B,C,D)` with `F = {A→B, B→C, A→D}`.

**Q21 (Intermediate).** `R(A,B,C,D,E,F)` with `F = {A→BC, CD→E, B→D, E→A}`. Compute `{A}⁺` and `{C,D}⁺` and use them to determine if both are super keys.

**Q22 (Senior).** Explain why the attribute-closure algorithm must iterate to a "fixed point" rather than stopping after a single pass over the FD set, using an example where a single pass would give an incorrect (incomplete) result.

---

### 18.7 Decomposition Exercises (Module 13)

**Q23 (Intermediate).** `R(A,B,C)` with `F = {A→B}`. Test whether decomposing into `R1(A,C)` and `R2(B,C)` is lossless-join. Show your work.

**Q24 (Intermediate).** For the decomposition in Q23 (assuming you found it lossy), propose a CORRECT lossless-join decomposition instead.

**Q25 (Senior).** Explain the difference between lossless-join and dependency-preserving decomposition. Give an example of a decomposition that has one property but not the other.

**Q26 (Senior).** Run the full 3NF Synthesis Algorithm on `R(A,B,C,D,E)` with `F = {A→B, B→C, CD→E}`: compute the canonical cover, create relations per FD, and verify whether a candidate-key-containing relation is present.

---

### 18.8 Case Studies (Cross-Module Synthesis)

**Case Study A.** A legacy `Payroll` table stores: `emp_id, emp_name, dept_id, dept_name, dept_budget, pay_period, hours_worked, hourly_rate, gross_pay`. Assume `emp_id` is the only candidate key.
1. Identify all functional dependencies you'd reasonably expect.
2. Identify every 2NF and 3NF violation.
3. Produce a fully normalized (3NF) schema.
4. Identify one column that might be intentionally denormalized for reporting purposes, and justify it.

**Case Study B.** A ride-sharing startup's `Trips` table has grown to include: `trip_id, rider_id, rider_name, driver_id, driver_name, vehicle_plate, pickup_city, pickup_zone, fare, distance_km, trip_date`. The engineering team has noticed that updating a driver's name requires updating thousands of historical trip rows.
1. Diagnose the anomaly and the normal form violation causing it.
2. Propose a normalized redesign.
3. Discuss whether `rider_name`/`driver_name` being duplicated per trip could ever be a DELIBERATE, justified design choice (hint: think about historical accuracy, Module 16 Section 16.2's `unit_price_at_purchase` reasoning) — under what condition would duplicating these specific names actually be defensible rather than a bug?

**Case Study C.** You're asked in an interview: "Design a database for a subscription-based streaming service (like a video-on-demand platform)." Sketch out:
1. The major entities (Users, Subscriptions, Content, Viewing_History, Payments, etc.)
2. At least one many-to-many relationship requiring a junction table.
3. At least one deliberate denormalization decision with justification (Module 14).
4. How you'd structure a Star Schema (Module 17) for analyzing "watch time by content genre and subscriber region" reporting.

---

### 18.9 Whiteboard Database Design Problems

Work through each as a live design exercise — entities, keys, relationships, normalization reasoning, and at least one denormalization call, exactly as demonstrated across Module 16.

1. **Parking Garage Management System** — track parking spots, vehicles, entry/exit times, and fee calculation.
2. **Online Learning Platform** — courses, video lessons, student progress tracking, quizzes, and certificates.
3. **Restaurant Table Reservation System** — tables, reservations, waitlists, and customer preferences.
4. **Fitness Tracking App** — users, workouts, exercises, sets/reps, and progress-over-time tracking (consider whether this needs any temporal/historical modeling, Module 15 Section 15.10).
5. **Airline Reservation System** — flights, seats, bookings, passengers, and fare classes (consider the seat-double-booking prevention question, similar to Module 16's movie ticket exercise).

---

### 18.10 GATE-Style Numerical Question Bank

**Q27.** `R(A,B,C,D,E)` with `F = {A→BC, CD→E, B→D, E→A}`. (a) Compute `A⁺`. (b) List all candidate keys. (c) State how many prime and non-prime attributes exist.

**Q28.** `R(A,B,C,D)` with `F = {AB→CD, C→A}`. Determine whether R is in BCNF. If not, decompose it and verify each resulting relation satisfies BCNF.

**Q29.** `R(A,B,C,D,E,F)` with `F = {A→B, ABCD→E, EF→G, G→A}` (assume a 7th attribute G exists, so R actually has attributes A–G). Compute `A⁺`. Is `{A}` alone a candidate key?

**Q30.** Given `F = {A→BC, B→C, A→B, AB→C}` on `R(A,B,C)`, compute the canonical cover, showing all three algorithm steps explicitly.

**Q31.** `R(A,B,C,D,E)` with candidate keys `{A,B}` and `{A,D}` (verified), and FD `D→B`. Determine whether R satisfies 3NF, BCNF, both, or neither — justify explicitly using the prime-attribute and super-key tests.

**Q32.** A relation `R(A,B,C,D,E,F)` has MVDs `A→→B` and `A→→C` with the only candidate key being the full attribute set. Determine whether R is in 4NF, and decompose if not.

---

### 18.11 Interview Questions by Seniority Level — Full Bank

**Beginner-level interview questions:**
- What is a functional dependency? Give an example.
- What is the difference between a candidate key and a primary key?
- What are the three classic normalization anomalies?
- What is 1NF, and why does atomicity matter?
- What is the difference between 2NF and 3NF?

**Intermediate-level interview questions:**
- Walk me through finding all candidate keys of a relation given its FD set.
- Given this flat table [interviewer provides one], identify the FDs and normalize it to 3NF.
- What's the difference between BCNF and 3NF, and can you give an example where they differ?
- How would you decide whether to denormalize a specific slow-performing query?
- Explain the difference between lossless-join and dependency-preserving decomposition.

**Senior SDE-2/SDE-3-level interview questions:**
- Design a database schema for [any real-world system]. Walk me through your entities, keys, relationships, and at least one place you'd denormalize, with justification.
- Explain why BCNF decomposition can sacrifice dependency preservation, using a concrete example, and how you'd compensate for that in production.
- When would you choose a Star Schema over a Snowflake Schema for an analytics system, and why?
- Tell me about a real (or realistic hypothetical) data-integrity bug rooted in a schema design anomaly, and how you'd diagnose and fix it.
- How do you approach the trade-off between normalization and query performance in a system you're designing from scratch?

**Whiteboard system design discussions (senior-level, open-ended):**
- "Design Twitter's/a social network's database" — expect discussion of the Follow self-referencing many-to-many table (Module 16, Section 16.9), the like_count denormalization, and how you'd handle a viral post's read load.
- "Design an e-commerce checkout system's database" — expect discussion of price-snapshotting (Module 16, Section 16.2), order status tracking, and payment idempotency considerations.
- "How would you evolve a schema as a startup scales from 100 to 100 million users?" — expect discussion of Module 1's database lifecycle, Module 15's scalability practices (BIGINT keys, partition-key planning), and Module 17's eventual OLTP/OLAP split.

---

### 18.12 Self-Assessment Checklist

Before considering yourself interview-ready on this Part's material, confirm you can, **without referring back to any module**:

```
□ Define and give an example of: trivial, non-trivial, full, partial, and transitive FDs
□ Compute an attribute closure by hand, iterating to a fixed point
□ State and apply all 3 basic + 4 derived Armstrong's Axioms
□ Compute a canonical cover from a messy FD set
□ Find ALL candidate keys of a relation systematically, and classify prime/non-prime attributes
□ Explain each anomaly (update/insert/delete) with an original example
□ Check 1NF, 2NF, 3NF, and BCNF compliance for any given relation
□ Explain the 3NF "escape hatch" and construct an example where it applies
□ Decompose a relation to 2NF, 3NF, or BCNF, verifying lossless-join and dependency preservation
□ Explain 4NF and 5NF with the classic MVD/JD examples
□ Run the 3NF Synthesis Algorithm end to end
□ Justify at least 3 deliberate denormalization decisions with real-world reasoning
□ Design a complete schema for a novel real-world system, live, from scratch
□ Explain OLTP vs OLAP and design a basic Star Schema
```

---

### 18.13 Chapter Summary

This module intentionally introduced **no new theory** — it exists purely as a comprehensive, cross-referenced practice bank spanning:
- Functional dependencies and their classifications (Module 2)
- Normalization and anomalies (Module 7)
- Candidate keys and closure (Modules 3, 6)
- Armstrong's Axioms (Module 4)
- Decomposition properties (Module 13)
- Real-world schema design and denormalization (Modules 14, 16)
- Industry-level OLTP/OLAP trade-offs (Module 17)

Working through Sections 18.2–18.11 thoroughly, ideally multiple times over spaced intervals, is the recommended final preparation step before a DBMS-focused interview, GATE exam, or placement test.

---

**End of Module 18.**

*Next up — Module 19: Preview of Part 6 — a short bridge module explaining how everything learned in this Part (Database Design) connects forward to Transactions, ACID properties, Locking, Concurrency, Isolation Levels, and Recovery, which will be covered in full depth in the next Part of this series. Reply "next" / "continue" when ready.*
