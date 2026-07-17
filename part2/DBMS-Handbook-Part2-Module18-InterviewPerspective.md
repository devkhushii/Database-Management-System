# DBMS Handbook — Part 2
## Module 18: Interview Perspective

> **Builds on:** Every module in this Part. This module is a **pure synthesis and rehearsal module** — no new theory, only interview-format practice, organized by question type, drawing on every concept already mastered.
>
> **How to use this module:** Treat this as a mock-interview session. For each question, pause and formulate your own answer before reading the model answer.

---

## Table of Contents

1. [The Four Question Formats](#1-the-four-question-formats)
2. [Frequently Asked Conceptual Questions](#2-frequently-asked-conceptual-questions)
3. [Scenario-Based Questions](#3-scenario-based-questions)
4. [Diagram-Based Questions](#4-diagram-based-questions)
5. [Full Schema Design Questions](#5-full-schema-design-questions)
6. [What Interviewers Are ACTUALLY Testing For](#6-what-interviewers-are-actually-testing-for)
7. [A Model Answer Framework (Use This Every Time)](#7-a-model-answer-framework-use-this-every-time)
8. [Chapter Summary](#8-chapter-summary)
9. [Final Exercises](#9-final-exercises)

---

## 1. The Four Question Formats

Every DBMS/data-modeling interview question you'll encounter falls into one of four formats. Recognizing the format immediately tells you what kind of answer is expected:

| Format | What's Expected | Example |
|---|---|---|
| **Conceptual** | A precise definition or comparison, often with a "why" | "What's the difference between a candidate key and a super key?" |
| **Scenario-Based** | Apply a concept to a described situation, usually with a judgment call | "Would you use a surrogate or natural key for a user table, and why?" |
| **Diagram-Based** | Read or draw an ER diagram, or interpret notation | "Given this ER diagram, what cardinality is this relationship?" |
| **Full Schema Design** | Design an entire system from a one-line prompt | "Design a database for a ride-sharing app." |

---

## 2. Frequently Asked Conceptual Questions

For each, a concise model answer with its source module noted.

**Q: What is the difference between a Super Key and a Candidate Key?**
> A super key is any attribute set that uniquely identifies a tuple, possibly with redundant attributes. A candidate key is a *minimal* super key — no attribute can be removed without losing uniqueness. Every candidate key is a super key, but not vice versa. *(Module 4, §3-§4)*

**Q: Why can't a Primary Key be NULL?**
> This is Entity Integrity: a primary key's job is to uniquely identify a real-world entity. A NULL primary key would mean "this row represents some entity, but we don't know which" — a logical contradiction, since every entity must be identifiable to exist meaningfully in the database. *(Module 5, §5)*

**Q: What's the difference between a Primary Key and a Unique Key?**
> Both enforce uniqueness, but a Primary Key can never be NULL and there's exactly one per table; a Unique Key can allow NULL (often multiple NULLs) and a table can have many. *(Module 4, §13)*

**Q: Explain the difference between DELETE CASCADE, SET NULL, and RESTRICT.**
> CASCADE automatically deletes child rows when the parent is deleted; SET NULL sets the child's foreign key to NULL instead; RESTRICT (or NO ACTION) blocks the parent deletion entirely if child rows still reference it. The right choice depends on whether the child can meaningfully exist without that parent. *(Module 4, §9)*

**Q: What's the difference between a weak entity and an associative entity?**
> A weak entity depends on ONE owner entity for its existence and identity (existence dependency, e.g., Dependent of Employee). An associative entity represents a many-to-many relationship between TWO OR MORE entities that has accumulated its own meaningful attributes (e.g., Enrollment between Student and Course) — it's not existence-dependent on just one parent the same way. *(Module 6, §8-§9)*

**Q: Why is NULL = NULL not TRUE in SQL?**
> SQL uses three-valued logic (True/False/Unknown) because NULL represents "unknown," not a real value. Comparing two unknowns can't logically be declared equal or unequal — the result is Unknown, not True. This is why `IS NULL` exists as a dedicated operator. *(Module 2, §17)*

**Q: What is the difference between Generalization and Specialization?**
> They produce the identical resulting ISA hierarchy — the only difference is direction of reasoning. Generalization is bottom-up (spotting shared attributes among existing specific entities and abstracting a superclass). Specialization is top-down (splitting an existing general entity into specific subclasses). *(Module 12, §2-§4)*

**Q: Why can't a ternary relationship be decomposed into three binary relationships?**
> Doing so loses the true three-way association — you can no longer determine which specific combination of all three entities occurred together as one linked fact; you only know pairwise facts, which is genuinely ambiguous when reconstructed. *(Module 9, §7)*

---

## 3. Scenario-Based Questions

**Q: "We're building a `Users` table. Should the primary key be the email address or an auto-increment integer?"**
> Prefer a surrogate integer (or UUID) as the primary key, with `email UNIQUE NOT NULL` as an alternate key. Emails can change, and if used directly as a PK, every foreign key referencing it across the schema would need cascading updates — a real, painful, avoidable risk. This is the "best of both worlds" natural-vs-surrogate pattern. *(Module 4, §16)*

**Q: "A table has `phone1`, `phone2`, `phone3` columns. What's wrong, and how would you fix it?"**
> This is a rigid, non-future-proof modeling of what's really a multi-valued attribute. It caps the number of phones arbitrarily and wastes NULL-filled columns for most rows. Fix: extract into a separate `UserPhone(user_id, phone_number)` table with a composite key — this scales to any number of phones. *(Module 7, §6; Module 14, §7)*

**Q: "Our `Book.author_id` column assumes one author per book, but we now need to support co-authored books. What changes?"**
> This requires converting a 1:N relationship into M:N: create a `BookAuthor(book_id, author_id)` junction table, migrate existing `author_id` values into it, then drop the `author_id` column from `Book`, and update all queries that previously joined directly on it. *(Module 10, §9)*

**Q: "Two teams each built a `Customer`-like table independently — `Customer` and `Client` — representing the same real people. How do you fix this?"**
> This is the "duplicate entities" mistake, usually a symptom of insufficient cross-team requirements alignment during ER modeling. Fix: consolidate into one shared entity with a single canonical identifier, and layer any team-specific extra attributes on top via foreign keys rather than duplicating the core identity data — going forward, establish shared entity ownership before independent modeling begins. *(Module 16, §3)*

**Q: "You have `Department.head_emp_id NOT NULL` and `Employee.dept_id NOT NULL`, both referencing each other. What happens when you try to insert data?"**
> This is a circular, mutually-mandatory foreign key dependency — neither row can be inserted first without violating the other's constraint. Fix: make one FK nullable (insert the "looser" side first, then update), use deferred constraint checking within one transaction, or break the cycle entirely with a separate `DepartmentHead` assignment table. *(Module 16, §7)*

---

## 4. Diagram-Based Questions

**Q: Given this notation — `Department (1,1) ──Manages── (0,1) Employee` — describe the full relationship in plain English.**
> Every Department must have exactly one manager (cardinality 1, total participation — minimum 1). Each Employee manages at most one department, and most employees manage none (cardinality 0 or 1, partial participation — minimum 0). *(Module 10 §7, Module 11 §5-§6)*

**Q: In Chen's notation, what does a double-outlined rectangle connected to a double-outlined diamond represent?**
> A weak entity connected via its identifying relationship to its owner entity. The weak entity's key attribute would additionally be shown with a dashed underline (its partial key). *(Module 6, §8; Module 8)*

**Q: What does the "d" vs "o" marking inside an EER ISA circle mean?**
> "d" = Disjoint (an instance can belong to at most one subclass); "o" = Overlapping (an instance can belong to multiple subclasses simultaneously). This is independent of the single/double line marking total vs. partial completeness. *(Module 12, §6-§9)*

**Q: Draw the ER diagram implied by this SQL:**
```sql
CREATE TABLE OrderItem (
    order_id INT, line_number INT, product_id INT,
    PRIMARY KEY (order_id, line_number),
    FOREIGN KEY (order_id) REFERENCES `Order`(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Product(product_id)
);
```
> `OrderItem` is a weak entity owned by `Order` (composite key = order's PK + partial key `line_number`, `ON DELETE CASCADE` confirms total, mandatory participation). It ALSO has a separate, non-identifying relationship to `Product` (strong entity, ordinary FK, no cascade specified). *(Module 8, §8; Module 13, §3)*

---

## 5. Full Schema Design Questions

### The Expected Process (Narrate This Explicitly, Every Time)

```
1. Clarify requirements (ask questions if genuinely ambiguous — Module 6 §3)
2. Identify entities, classify as strong/weak/associative (Module 6, 8)
3. Identify relationships; determine cardinality AND participation for
   each side, explicitly verifying assumptions rather than guessing
   (Module 9, 10, 11)
4. Identify attributes; flag any multi-valued/composite/derived ones (Module 7)
5. Consider whether any entity needs Generalization/Specialization (Module 12)
6. Apply the mapping rules to produce the relational schema (Module 13)
7. Review against the Schema Audit Checklist (Module 16 §10)
```

### Worked Example: "Design a database for a ride-sharing app (like Uber/Lyft)"

**Step 1 — Clarify:** Do riders rate drivers (and vice versa)? Can a ride have multiple stops? Are there different vehicle types? *(Assume: yes to ratings, single destination per ride for simplicity, multiple vehicle types.)*

**Step 2-3 — Entities & Relationships:**
```
Rider (Strong) ── requests (1:N) ──> Ride
Driver (Strong) ── accepts (1:N over time) ──> Ride
Ride (Strong, associative-ish) ── uses ──> VehicleType (N:1)
Ride ── has ──> Rating (weak, owner: Ride — a rating only makes sense
                 tied to one specific completed ride)
```

**Step 4 — Attributes:** `Rider.phone_numbers` — likely single-valued for simplicity, but flag it as a candidate multi-valued attribute if the real requirements demanded it.

**Step 5 — EER:** Not clearly needed here unless "Driver" and "Rider" turn out to share a common `Person` superclass (a person could be both) — worth raising as a design question, referencing Module 12's Overlapping+Partial pattern.

**Step 6 — Schema:**
```sql
CREATE TABLE Rider (rider_id INT PRIMARY KEY, name VARCHAR(50), phone VARCHAR(15));
CREATE TABLE Driver (driver_id INT PRIMARY KEY, name VARCHAR(50), license_no VARCHAR(20) UNIQUE);
CREATE TABLE VehicleType (vehicle_type_id INT PRIMARY KEY, type_name VARCHAR(20));
CREATE TABLE Ride (
    ride_id         INT AUTO_INCREMENT PRIMARY KEY,
    rider_id        INT NOT NULL,
    driver_id       INT,                    -- nullable: not yet matched (partial participation)
    vehicle_type_id INT NOT NULL,
    pickup_location VARCHAR(100),
    dropoff_location VARCHAR(100),
    status          VARCHAR(20) DEFAULT 'REQUESTED',
    requested_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (rider_id) REFERENCES Rider(rider_id),
    FOREIGN KEY (driver_id) REFERENCES Driver(driver_id),
    FOREIGN KEY (vehicle_type_id) REFERENCES VehicleType(vehicle_type_id)
);
CREATE TABLE Rating (                        -- Weak entity (owner: Ride)
    ride_id      INT PRIMARY KEY,             -- 1:1 with Ride, so PK alone suffices
    rider_rating INT CHECK (rider_rating BETWEEN 1 AND 5),
    driver_rating INT CHECK (driver_rating BETWEEN 1 AND 5),
    FOREIGN KEY (ride_id) REFERENCES Ride(ride_id) ON DELETE CASCADE
);
```

**Step 7 — Audit:** `Ride.driver_id` nullable correctly reflects that a ride can exist before a driver accepts it (Module 11 §7); `Rating` correctly uses `ride_id` alone as its PK since it's a 1:1 weak relationship (not the usual composite, since there's no separate partial key needed — a valid, simpler variant of Module 8's formula when the "many" side cardinality is actually 1).

### Interview Importance

**Extremely High** — this narrated, step-by-step process IS the deliverable in a live design interview; the final SQL matters less than demonstrating this structured reasoning.

---

## 6. What Interviewers Are ACTUALLY Testing For

Beyond correctness, interviewers are consistently evaluating these underlying qualities:

| Quality | How It Shows Up |
|---|---|
| **Do you verify assumptions instead of guessing?** | Asking about cardinality/business rules before committing to a design (Module 10 §8, Module 16 §6) |
| **Do you know WHY, not just WHAT?** | Explaining *why* entity integrity requires non-null PKs, not just reciting the rule |
| **Do you recognize trade-offs, not just "correct" answers?** | Natural vs. surrogate keys, normalization vs. denormalization, aggregation vs. associative entity — all have legitimate trade-offs, and naming them shows maturity |
| **Can you course-correct when given new information?** | If the interviewer says "actually, books CAN have multiple authors," do you smoothly redesign, or get flustered? |
| **Do you communicate your reasoning out loud?** | A silently-drawn correct diagram is worth far less than a narrated, step-by-step design process |

### Interview Importance

**This section describes the meta-skill underlying every other section of this module** — internalizing it is more valuable than memorizing any single Q&A pair.

---

## 7. A Model Answer Framework (Use This Every Time)

For ANY conceptual question, structure your answer as:

```
1. DEFINITION — state the precise definition (one or two sentences)
2. WHY — briefly explain why the concept exists / what problem it solves
3. EXAMPLE — give a small, concrete example
4. CONTRAST (if applicable) — distinguish it from a commonly-confused
   related concept
```

For ANY scenario or design question, structure your answer as:

```
1. CLARIFY — ask about genuinely ambiguous requirements, or state your
   assumption explicitly if asking isn't warranted
2. IDENTIFY — name the entities/relationships/constraints involved
3. DECIDE — make the design choice, and justify it against the
   SPECIFIC scenario given (not just textbook default)
4. TRADE-OFF — name at least one alternative approach and why you
   didn't choose it
```

---

## 8. Chapter Summary

- Interview questions fall into four recognizable formats: **Conceptual, Scenario-Based, Diagram-Based, and Full Schema Design** — recognizing the format immediately clarifies what kind of answer is expected.
- A curated set of the most frequently asked conceptual and scenario-based questions was answered concisely, each tracing back to its source module for full depth.
- The full **Design Question process** (clarify → identify entities/relationships → attributes → EER if needed → map to relational → audit) was demonstrated end-to-end with a ride-sharing app example, following the exact same structured approach used throughout Module 17's case studies.
- Interviewers are testing for **underlying qualities** — verifying assumptions, understanding "why" not just "what," recognizing genuine trade-offs, and communicating reasoning out loud — far more than for rote memorization of a single "correct" schema.
- The **Model Answer Framework** (Definition → Why → Example → Contrast, for conceptual questions; Clarify → Identify → Decide → Trade-off, for design questions) is a reusable structure to apply to *any* question this handbook hasn't explicitly covered.

---

## 9. Final Exercises

### Mock Interview Practice (Timed, Recommended: 5-10 minutes each, spoken aloud)

1. "Explain the difference between a weak entity and a multi-valued attribute, and when you'd choose one modeling approach over the other."
2. "Design a database for a hotel booking system, including room types, amenities, and guest reviews." (Full schema design — narrate every step from §5's process.)
3. "A junior engineer added `ON DELETE CASCADE` to every single foreign key in the schema 'for consistency.' Is this a good practice? Explain the risks."
4. "Given a `Person → Student/Employee` EER hierarchy that is Overlapping and Partial, walk through why Strategy 2 (superclass + subclass tables) from Module 13 is the appropriate mapping choice, and what would go wrong with Strategy 1."
5. "Design a database for a movie theater booking system (Movies, Showtimes, Theaters, Seats, Bookings) — pay special attention to preventing double-booking of a specific seat for a specific showtime."
6. "What's wrong with this schema, and how would you fix it?"
```sql
CREATE TABLE orders (
    id INT,
    customer_name VARCHAR(50),
    customer_email VARCHAR(50),
    total FLOAT,
    items VARCHAR(500)   -- comma-separated list of product names
);
```
7. "Explain, as if to a non-technical product manager, why we can't just add a `manager_name` column directly to the `Employee` table instead of using a foreign key."

### Self-Assessment

After attempting the above, review your answers against:
- Did you use the **Model Answer Framework** (§7) structure?
- Did you **name the specific module/concept** each answer draws from (even mentally)?
- Did you **verify assumptions** rather than silently guessing, wherever the question was ambiguous?
- Did you **name at least one trade-off or alternative** wherever genuinely relevant?

---

**This completes the interview-preparation core of Part 2. One module remains: Module 19 — Preview of Part 3, connecting everything learned here to SQL implementation and Normalization.**

> Say "continue" or "next module" to proceed to Module 19 (the final module of Part 2).
