# DBMS Handbook — Part 2
## Module 17: Industry Case Studies

> **Builds on:** EVERY module in this Part — this is the **capstone application module**, where every concept (ER components, keys, constraints, cardinality, participation, EER, mapping rules, schema design, and the mistake checklist) gets applied to eight complete, realistic system designs.
>
> **How to use this module:** Each case study follows the same structure: (1) Requirements, (2) Entity identification with type classification (strong/weak/associative), (3) Key relationships with cardinality + participation, (4) ASCII ER diagram, (5) Full relational schema, (6) Design notes referencing specific earlier modules. Read actively — before checking each case study's design, try sketching your own first.

---

## Table of Contents

1. [Case Study 1: Banking System](#1-case-study-1-banking-system)
2. [Case Study 2: Hospital Management System](#2-case-study-2-hospital-management-system)
3. [Case Study 3: E-Commerce System](#3-case-study-3-e-commerce-system)
4. [Case Study 4: University Management System](#4-case-study-4-university-management-system)
5. [Case Study 5: Railway Reservation System](#5-case-study-5-railway-reservation-system)
6. [Case Study 6: Social Media System](#6-case-study-6-social-media-system)
7. [Case Study 7: Food Delivery System](#7-case-study-7-food-delivery-system)
8. [Case Study 8: Inventory Management System](#8-case-study-8-inventory-management-system)
9. [Cross-Case-Study Pattern Recognition](#9-cross-case-study-pattern-recognition)
10. [Chapter Summary](#10-chapter-summary)
11. [Exercises](#11-exercises)

---

## 1. Case Study 1: Banking System

### Requirements

Customers hold one or more Accounts at Branches; Accounts have Transactions (weak entity, Module 8); a Customer can jointly hold an Account with other Customers (M:N); Loans are issued to Customers by Branches.

### Entity Classification

| Entity | Type | Key |
|---|---|---|
| `Customer` | Strong | `customer_id` (surrogate) |
| `Branch` | Strong | `branch_id` |
| `Account` | Strong | `account_no` |
| `Transaction` | Weak (owner: `Account`) | `(account_no, txn_seq)` |
| `Loan` | Strong | `loan_id` |
| `AccountHolder` | Associative (Customer ↔ Account, M:N) | `(customer_id, account_no)` |

### ASCII ER Diagram

```
┌──────────┐      M:N      ┌───────────┐
│ Customer │══════════════║  Account   ║
└────┬─────┘  AccountHolder└─────┬─────┘
     │                            │ 1:N (weak, total)
     │ 1:N                        ▼
     ▼                     ╔═══════════════╗
┌──────────┐               ║ Transaction   ║
│   Loan   │               ╚═══════════════╝
└────┬─────┘
     │ N:1
     ▼
┌──────────┐      1:N       ┌───────────┐
│  Branch  │───────────────>│  Account  │ (also 1:N to Loan)
└──────────┘
```

### Relational Schema

```sql
CREATE TABLE Branch (
    branch_id    INT PRIMARY KEY,
    branch_name  VARCHAR(50),
    city         VARCHAR(50)
);
CREATE TABLE Customer (
    customer_id  INT PRIMARY KEY,
    name         VARCHAR(50),
    email        VARCHAR(100) UNIQUE
);
CREATE TABLE Account (
    account_no    INT PRIMARY KEY,
    branch_id     INT NOT NULL,
    account_type  VARCHAR(20),
    balance       DECIMAL(12,2) CHECK (balance >= 0),   -- Module 5 §11
    FOREIGN KEY (branch_id) REFERENCES Branch(branch_id)
);
CREATE TABLE AccountHolder (              -- M:N junction (Module 10 §5)
    customer_id  INT,
    account_no   INT,
    PRIMARY KEY (customer_id, account_no),
    FOREIGN KEY (customer_id) REFERENCES Customer(customer_id),
    FOREIGN KEY (account_no)  REFERENCES Account(account_no)
);
CREATE TABLE Transaction (                -- Weak entity (Module 8)
    account_no   INT,
    txn_seq      INT,
    amount       DECIMAL(12,2) NOT NULL,
    txn_type     VARCHAR(10) CHECK (txn_type IN ('DEPOSIT','WITHDRAWAL')),
    txn_date     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (account_no, txn_seq),
    FOREIGN KEY (account_no) REFERENCES Account(account_no) ON DELETE CASCADE
);
CREATE TABLE Loan (
    loan_id      INT PRIMARY KEY,
    customer_id  INT NOT NULL,
    branch_id    INT NOT NULL,
    amount       DECIMAL(12,2) CHECK (amount > 0),
    FOREIGN KEY (customer_id) REFERENCES Customer(customer_id),
    FOREIGN KEY (branch_id)   REFERENCES Branch(branch_id)
);
```

### Design Notes

- `balance >= 0` is a `CHECK` domain constraint (Module 5, §11) — though real banks often allow controlled overdrafts, modeled here as the simple default case.
- `AccountHolder` correctly captures **joint accounts** — a genuine M:N business rule, not a mistake (Module 16, §6's lesson: verify, don't assume, cardinality).
- `Transaction` follows Module 8's exact weak-entity formula: composite key = owner's PK + partial key (`txn_seq`), `ON DELETE CASCADE`.

---

## 2. Case Study 2: Hospital Management System

### Requirements

Patients have Appointments with Doctors (M:N, with attributes); Doctors belong to Departments; Patients are admitted to Rooms (weak entity, Module 8); Prescriptions link Doctor, Patient, and Drug (ternary, Module 9 §7).

### Entity Classification

| Entity | Type | Key |
|---|---|---|
| `Patient` | Strong | `patient_id` |
| `Doctor` | Strong | `doctor_id` |
| `Department` | Strong | `dept_id` |
| `Appointment` | Associative (M:N) | `appointment_id` (surrogate) |
| `Room` | Weak (owner: `Ward`) | `(ward_id, room_number)` |
| `Prescription` | Ternary relationship → own table | `prescription_id` (surrogate, Module 9 §9) |

### ASCII ER Diagram

```
┌──────────┐   M:N (Appointment)   ┌──────────┐
│ Patient  │═══════════════════════║  Doctor  │
└────┬─────┘                       └────┬─────┘
     │                                   │ N:1
     │  (Prescribes: ternary)            ▼
     │                            ┌──────────────┐
     └───────────>╔═══════════╗<─│  Department  │
                   ║Prescription║  └──────────────┘
                   ╚═══════════╝
                        ▲
                        │
                   ┌──────────┐
                   │   Drug   │
                   └──────────┘
```

### Relational Schema

```sql
CREATE TABLE Department (
    dept_id    INT PRIMARY KEY,
    dept_name  VARCHAR(50)
);
CREATE TABLE Doctor (
    doctor_id  INT PRIMARY KEY,
    name       VARCHAR(50),
    dept_id    INT NOT NULL,               -- total participation
    FOREIGN KEY (dept_id) REFERENCES Department(dept_id)
);
CREATE TABLE Patient (
    patient_id  INT PRIMARY KEY,
    name        VARCHAR(50),
    dob         DATE
);
CREATE TABLE Appointment (                  -- Associative entity (Module 6 §9)
    appointment_id  INT AUTO_INCREMENT PRIMARY KEY,
    patient_id      INT NOT NULL,
    doctor_id       INT NOT NULL,
    appt_datetime   TIMESTAMP NOT NULL,
    status          VARCHAR(20) DEFAULT 'SCHEDULED',
    FOREIGN KEY (patient_id) REFERENCES Patient(patient_id),
    FOREIGN KEY (doctor_id)  REFERENCES Doctor(doctor_id)
);
CREATE TABLE Ward (
    ward_id    INT PRIMARY KEY,
    ward_name  VARCHAR(50)
);
CREATE TABLE Room (                          -- Weak entity (Module 8)
    ward_id      INT,
    room_number  VARCHAR(10),
    patient_id   INT,                        -- nullable: room may be vacant
    PRIMARY KEY (ward_id, room_number),
    FOREIGN KEY (ward_id) REFERENCES Ward(ward_id) ON DELETE CASCADE,
    FOREIGN KEY (patient_id) REFERENCES Patient(patient_id)
);
CREATE TABLE Drug (
    drug_id    INT PRIMARY KEY,
    drug_name  VARCHAR(50)
);
CREATE TABLE Prescription (                  -- Ternary relationship (Module 9 §7)
    prescription_id  INT AUTO_INCREMENT PRIMARY KEY,
    doctor_id        INT NOT NULL,
    patient_id       INT NOT NULL,
    drug_id          INT NOT NULL,
    prescribed_date  DATE NOT NULL,
    dosage           VARCHAR(30),
    FOREIGN KEY (doctor_id)  REFERENCES Doctor(doctor_id),
    FOREIGN KEY (patient_id) REFERENCES Patient(patient_id),
    FOREIGN KEY (drug_id)    REFERENCES Drug(drug_id)
);
```

### Design Notes

- `Prescription` uses a **surrogate key** rather than the pure composite `(doctor_id, drug_id, patient_id)` from Module 9's textbook example — a deliberate, pragmatic choice allowing a doctor to prescribe the *same* drug to the *same* patient on multiple different dates (the composite alone would forbid that repeat).
- This directly echoes Module 8, §9's real-world pragmatism note about banking `Transaction`s — favor surrogate keys when the "pure" composite key would incorrectly forbid legitimate repeated facts.
- The NULL ambiguity trap from Module 3, §5 (allergy_notes) is worth revisiting here for `Room.patient_id` — NULL clearly means "vacant," which is unambiguous in this specific case.

---

## 3. Case Study 3: E-Commerce System

### Requirements

Customers place Orders containing multiple Products (M:N via `OrderItem`, weak/associative — Module 8 §8); Products belong to Categories; Customers can have multiple Addresses (multi-valued/complex, Module 7 §12); Reviews link Customer and Product (M:N with attributes).

### Relational Schema

```sql
CREATE TABLE Customer (
    customer_id  INT PRIMARY KEY,
    name         VARCHAR(50),
    email        VARCHAR(100) UNIQUE
);
CREATE TABLE CustomerAddress (             -- Complex attribute → own table (Module 7 §12)
    customer_id   INT,
    address_type  VARCHAR(10),              -- 'Home', 'Work', etc.
    street        VARCHAR(100),
    city          VARCHAR(50),
    postal_code   VARCHAR(10),
    PRIMARY KEY (customer_id, address_type),
    FOREIGN KEY (customer_id) REFERENCES Customer(customer_id)
);
CREATE TABLE Category (
    category_id    INT PRIMARY KEY,
    category_name  VARCHAR(50),
    parent_id      INT,                     -- Recursive relationship (Module 9 §5)!
    FOREIGN KEY (parent_id) REFERENCES Category(category_id)
);
CREATE TABLE Product (
    product_id   INT PRIMARY KEY,
    name         VARCHAR(100),
    price        DECIMAL(10,2) CHECK (price >= 0),
    category_id  INT NOT NULL,
    FOREIGN KEY (category_id) REFERENCES Category(category_id)
);
CREATE TABLE `Order` (
    order_id     INT PRIMARY KEY,
    customer_id  INT NOT NULL,
    order_date   DATE DEFAULT (CURRENT_DATE),
    status       VARCHAR(20) DEFAULT 'PENDING',
    FOREIGN KEY (customer_id) REFERENCES Customer(customer_id)
);
CREATE TABLE OrderItem (                     -- Weak entity (Module 8 §8)
    order_id     INT,
    line_number  INT,
    product_id   INT NOT NULL,
    quantity     INT CHECK (quantity > 0),
    unit_price   DECIMAL(10,2) NOT NULL,     -- snapshot price at time of order!
    PRIMARY KEY (order_id, line_number),
    FOREIGN KEY (order_id) REFERENCES `Order`(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES Product(product_id)
);
CREATE TABLE Review (                         -- M:N with attributes
    customer_id  INT,
    product_id   INT,
    rating       INT CHECK (rating BETWEEN 1 AND 5),
    comment      TEXT,
    review_date  DATE DEFAULT (CURRENT_DATE),
    PRIMARY KEY (customer_id, product_id),    -- one review per customer per product
    FOREIGN KEY (customer_id) REFERENCES Customer(customer_id),
    FOREIGN KEY (product_id)  REFERENCES Product(product_id)
);
```

### Design Notes

- `Category.parent_id` is a **self-referencing (recursive) relationship** (Module 9, §5) — directly enabling nested category trees.
- `OrderItem.unit_price` is a deliberate, *intentional* redundancy — recall Module 14, §6's nuance: even though `Product.price` already exists, an order line item **must snapshot the price at time of purchase**, since `Product.price` will change over time and historical orders must remain accurate. This is a textbook example of *correct*, deliberate redundancy, not a design mistake.

---

## 4. Case Study 4: University Management System

> **Note:** This case study was already built, step-by-step, applying every mapping rule explicitly, in **Module 13, §11** — refer there for the complete design, walkthrough, and full traceability back to specific rules. It is not repeated here to avoid duplication; instead, treat Module 13, §11 as this Part's "University" reference design.

---

## 5. Case Study 5: Railway Reservation System

### Requirements

Trains run on Routes (Station-to-Station, many stops — this is genuinely an ordered, multi-entity relationship); Passengers book Tickets for a specific Train on a specific Date; a Ticket reserves a specific Seat (1:1 per ticket); Trains have multiple Coaches, and Coaches have multiple Seats (weak entity chain, Module 8 §10).

### Relational Schema

```sql
CREATE TABLE Train (
    train_id    INT PRIMARY KEY,
    train_name  VARCHAR(50)
);
CREATE TABLE Coach (                          -- Weak entity (owner: Train)
    train_id     INT,
    coach_number VARCHAR(5),
    coach_class  VARCHAR(20),                 -- 'Sleeper', 'AC', etc.
    PRIMARY KEY (train_id, coach_number),
    FOREIGN KEY (train_id) REFERENCES Train(train_id) ON DELETE CASCADE
);
CREATE TABLE Seat (                            -- Weak entity chain! (owner: Coach, Module 8 §10)
    train_id     INT,
    coach_number VARCHAR(5),
    seat_number  INT,
    PRIMARY KEY (train_id, coach_number, seat_number),
    FOREIGN KEY (train_id, coach_number) REFERENCES Coach(train_id, coach_number)
        ON DELETE CASCADE
);
CREATE TABLE Station (
    station_id    INT PRIMARY KEY,
    station_name  VARCHAR(50)
);
CREATE TABLE RouteStop (                        -- ordered M:N (Train ↔ Station)
    train_id      INT,
    stop_sequence INT,                          -- ORDER matters here!
    station_id    INT NOT NULL,
    arrival_time  TIME,
    PRIMARY KEY (train_id, stop_sequence),
    FOREIGN KEY (train_id)   REFERENCES Train(train_id) ON DELETE CASCADE,
    FOREIGN KEY (station_id) REFERENCES Station(station_id)
);
CREATE TABLE Passenger (
    passenger_id  INT PRIMARY KEY,
    name          VARCHAR(50),
    age           INT CHECK (age > 0)
);
CREATE TABLE Ticket (
    ticket_id     INT AUTO_INCREMENT PRIMARY KEY,
    passenger_id  INT NOT NULL,
    train_id      INT NOT NULL,
    coach_number  VARCHAR(5) NOT NULL,
    seat_number   INT NOT NULL,
    travel_date   DATE NOT NULL,
    FOREIGN KEY (passenger_id) REFERENCES Passenger(passenger_id),
    FOREIGN KEY (train_id, coach_number, seat_number)
        REFERENCES Seat(train_id, coach_number, seat_number),
    UNIQUE (train_id, coach_number, seat_number, travel_date)  -- prevents DOUBLE-BOOKING!
);
```

### Design Notes

- `Seat` demonstrates Module 8, §10's **weak entity chain**: `Train` owns `Coach`, and `Coach` owns `Seat` — the composite key grows at each level.
- `RouteStop.stop_sequence` captures **order**, something plain M:N junction tables (Module 10, §5) don't normally need — a good example of a relationship needing an extra "position/order" attribute beyond the standard pattern.
- The `UNIQUE (train_id, coach_number, seat_number, travel_date)` constraint on `Ticket` is the **critical business rule enforcement**: it's what actually prevents the same physical seat, on the same date, from being double-booked — an excellent real example of a `UNIQUE` constraint (Module 5, §8) directly encoding a core business rule, not just an abstract "identification" concern.

---

## 6. Case Study 6: Social Media System

### Requirements

Users create Posts; Users can Like Posts (M:N); Users can Comment on Posts (weak entity, owner = Post); Users Follow other Users (recursive M:N, Module 9 §5).

### Relational Schema

```sql
CREATE TABLE `User` (
    user_id    INT PRIMARY KEY,
    username   VARCHAR(30) UNIQUE NOT NULL,
    email      VARCHAR(100) UNIQUE
);
CREATE TABLE Post (
    post_id     INT PRIMARY KEY,
    user_id     INT NOT NULL,
    content     TEXT,
    posted_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES `User`(user_id)
);
CREATE TABLE Comment (                         -- Weak entity (owner: Post)
    post_id       INT,
    comment_seq   INT,
    user_id       INT NOT NULL,                -- WHO commented (separate FK, non-identifying!)
    comment_text  TEXT,
    commented_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (post_id, comment_seq),
    FOREIGN KEY (post_id) REFERENCES Post(post_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES `User`(user_id)
);
CREATE TABLE PostLike (                         -- M:N junction
    post_id  INT,
    user_id  INT,
    liked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (post_id, user_id),
    FOREIGN KEY (post_id) REFERENCES Post(post_id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES `User`(user_id)
);
CREATE TABLE Follows (                           -- Recursive M:N (Module 9 §5)
    follower_id  INT,
    followee_id  INT,
    followed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    FOREIGN KEY (follower_id) REFERENCES `User`(user_id),
    FOREIGN KEY (followee_id) REFERENCES `User`(user_id),
    CHECK (follower_id != followee_id)          -- a user can't follow themselves!
);
```

### Design Notes

- `Comment` is a great example of Module 8, §8's aside: it has its **identifying** FK to `Post` (its owner), PLUS a **separate, non-identifying** FK to `User` (who wrote it) — exactly the `OrderLineItem`→`Product` pattern revisited.
- `Follows` demonstrates a **recursive M:N relationship** (Module 9, §5) — both `follower_id` and `followee_id` reference the same `User` table.
- The `CHECK (follower_id != followee_id)` directly enforces a business rule (Module 5, §14) that no `CHECK`-based domain constraint alone could catch — a nice, simple, single-row rule perfectly suited to `CHECK`.

---

## 7. Case Study 7: Food Delivery System

### Requirements

Customers place Orders from Restaurants; Restaurants have Menu Items; Orders contain multiple Menu Items (M:N with quantity); a Delivery Agent is assigned to each Order (1:1 per order, but 1:N per agent over time).

### Relational Schema

```sql
CREATE TABLE Customer (
    customer_id  INT PRIMARY KEY,
    name         VARCHAR(50)
);
CREATE TABLE Restaurant (
    restaurant_id  INT PRIMARY KEY,
    name           VARCHAR(50),
    city           VARCHAR(50)
);
CREATE TABLE MenuItem (
    menu_item_id   INT PRIMARY KEY,
    restaurant_id  INT NOT NULL,
    item_name      VARCHAR(50),
    price          DECIMAL(8,2) CHECK (price >= 0),
    FOREIGN KEY (restaurant_id) REFERENCES Restaurant(restaurant_id)
);
CREATE TABLE DeliveryAgent (
    agent_id  INT PRIMARY KEY,
    name      VARCHAR(50)
);
CREATE TABLE FoodOrder (
    order_id       INT PRIMARY KEY,
    customer_id    INT NOT NULL,
    restaurant_id  INT NOT NULL,
    agent_id       INT,                          -- nullable: not yet assigned (PARTIAL participation!)
    order_time     TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status         VARCHAR(20) DEFAULT 'PLACED',
    FOREIGN KEY (customer_id)   REFERENCES Customer(customer_id),
    FOREIGN KEY (restaurant_id) REFERENCES Restaurant(restaurant_id),
    FOREIGN KEY (agent_id)      REFERENCES DeliveryAgent(agent_id)
);
CREATE TABLE OrderItem (                          -- Weak/associative entity
    order_id       INT,
    menu_item_id   INT,
    quantity       INT CHECK (quantity > 0),
    item_price     DECIMAL(8,2) NOT NULL,          -- snapshot, per Case Study 3's insight
    PRIMARY KEY (order_id, menu_item_id),
    FOREIGN KEY (order_id) REFERENCES FoodOrder(order_id) ON DELETE CASCADE,
    FOREIGN KEY (menu_item_id) REFERENCES MenuItem(menu_item_id)
);
```

### Design Notes

- `FoodOrder.agent_id` is **nullable** — correctly reflecting **partial participation** (Module 11, §3, §7): an order might not yet have an assigned delivery agent.
- This is a direct, practical echo of Case Study 1's `Loan`/Case Study 2's `Appointment` pattern: cardinality "1" on the agent side (per order) combined with "N" over time (one agent handles many orders across their shift) — the classic 1:N shape (Module 10, §3).

---

## 8. Case Study 8: Inventory Management System

### Requirements

Warehouses store Products; Products can be stored across multiple Warehouses (M:N with quantity — a "stock level" relationship attribute); Suppliers supply Products to Warehouses (ternary relationship: Supplier–Product–Warehouse, echoing Module 9's `Prescribes` pattern).

### Relational Schema

```sql
CREATE TABLE Warehouse (
    warehouse_id  INT PRIMARY KEY,
    location      VARCHAR(50)
);
CREATE TABLE Product (
    product_id  INT PRIMARY KEY,
    name        VARCHAR(50),
    sku         VARCHAR(20) UNIQUE            -- natural/business key (Module 4 §15)
);
CREATE TABLE Supplier (
    supplier_id  INT PRIMARY KEY,
    name         VARCHAR(50)
);
CREATE TABLE Stock (                           -- M:N with attribute (stock level)
    warehouse_id  INT,
    product_id    INT,
    quantity      INT NOT NULL CHECK (quantity >= 0),
    PRIMARY KEY (warehouse_id, product_id),
    FOREIGN KEY (warehouse_id) REFERENCES Warehouse(warehouse_id),
    FOREIGN KEY (product_id)   REFERENCES Product(product_id)
);
CREATE TABLE Supply (                           -- Ternary relationship (Module 9 §7)
    supplier_id   INT,
    product_id    INT,
    warehouse_id  INT,
    supply_date   DATE,
    unit_cost     DECIMAL(8,2) NOT NULL,
    PRIMARY KEY (supplier_id, product_id, warehouse_id, supply_date),
    FOREIGN KEY (supplier_id)  REFERENCES Supplier(supplier_id),
    FOREIGN KEY (product_id)   REFERENCES Product(product_id),
    FOREIGN KEY (warehouse_id) REFERENCES Warehouse(warehouse_id)
);
```

### Design Notes

- `Product.sku` is a classic **natural/business key** (Module 4, §10, §15) enforced as `UNIQUE` alongside the surrogate `product_id` — the "best of both worlds" pattern from Module 4, §16.
- `Supply` directly mirrors Case Study 2's `Prescription` ternary pattern (Module 9, §7) — but here, uniquely, `supply_date` is included **in the primary key itself** rather than using a surrogate key, since a genuinely new, distinct supply event each day is the natural, correct granularity (unlike the `Prescription` case, where a surrogate was preferred) — a good illustration that Rule 9's "composite key of all participants" (Module 13, §10) is a reasonable *default*, not an absolute mandate; the right choice depends on the specific business semantics.

---

## 9. Cross-Case-Study Pattern Recognition

Having built eight complete systems, step back and notice the **recurring structural patterns** that appeared again and again — recognizing these patterns *quickly* in a live interview is the single highest-leverage skill this entire Part has been building toward:

| Pattern | Appeared In | Core Rule |
|---|---|---|
| **Weak entity for "line items"** | `Transaction`, `OrderItem` (x2), `Comment`, `Room` | Module 8: composite key = owner PK + partial key |
| **M:N junction table** | `AccountHolder`, `Appointment`, `Review`, `PostLike`, `Stock` | Module 10 §5: composite key of both sides |
| **Recursive relationship** | `Category.parent_id`, `Follows` | Module 9 §5: self-referencing FK, often nullable |
| **Ternary relationship** | `Prescription`, `Supply` | Module 9 §7: composite/surrogate key spanning 3 FKs |
| **Weak entity chain** | `Train → Coach → Seat` | Module 8 §10: composite key grows at each level |
| **Deliberate redundancy (price snapshot)** | `OrderItem.unit_price` (x2), `Supply.unit_cost` | Module 14 §6: historical accuracy justifies redundancy |
| **Nullable FK for partial participation** | `FoodOrder.agent_id`, `Room.patient_id` | Module 11 §7: partial → nullable |
| **UNIQUE constraint enforcing a core business rule** | `Ticket`'s seat-booking UNIQUE | Module 5 §8: not just for keys — for real anti-double-booking logic |
| **Natural key as UNIQUE alongside surrogate PK** | `Customer.email`, `Product.sku` | Module 4 §16: best-of-both-worlds pattern |

---

## 10. Chapter Summary

- Eight complete, realistic system designs — **Banking, Hospital, E-Commerce, University (via Module 13), Railway, Social Media, Food Delivery, Inventory** — demonstrate every concept from this Part applied together, end-to-end, from requirements through full SQL schemas.
- Recurring structural patterns (weak entities for line-items, M:N junction tables, recursive relationships, ternary relationships, weak-entity chains, deliberate redundancy for historical snapshots, nullable FKs for partial participation, and business-rule-enforcing `UNIQUE` constraints) appear **repeatedly across unrelated domains** — recognizing these patterns quickly, rather than re-deriving them from scratch each time, is the mark of real design fluency.
- Several case studies made **deliberate, justified departures** from the "textbook default" (e.g., `Prescription` and `Supply` each handling their ternary-relationship key differently, based on real business semantics) — reinforcing Module 13's core lesson that mapping rules are strong defaults, not rigid, context-free mandates.

---

## 11. Exercises

### Beginner

1. For each of the 8 case studies, name at least one weak entity or M:N junction table used.
2. True/False: Every ternary relationship in this module used the exact same key strategy.
3. Identify which case study used a recursive relationship for a category/hierarchy tree.

### Intermediate

4. In Case Study 5 (Railway), explain why `UNIQUE (train_id, coach_number, seat_number, travel_date)` is essential, and describe what real-world bug would occur without it.
5. Compare `Prescription` (Case Study 2) and `Supply` (Case Study 8) — both are ternary relationships, but use different key strategies. Explain the business reasoning behind each choice.
6. In Case Study 6 (Social Media), explain why `Comment` needs TWO foreign keys (to `Post` and to `User`), and which one is the "identifying" relationship (Module 8).

### Advanced / Conceptual

7. Design a NINTH case study of your own choice (e.g., a Gym Membership System, a Movie Ticket Booking System, or a Ride-Sharing System), following the exact same six-part structure (requirements, entity classification, cardinality/participation, ER diagram, relational schema, design notes) used throughout this module.
8. Run the Full Schema Audit Checklist (Module 16, §10) against Case Study 3 (E-Commerce) and Case Study 7 (Food Delivery). Identify at least one potential improvement or edge case each design might be missing (e.g., what happens to `OrderItem` price snapshots if currency changes, or how `FoodOrder` status transitions might need a more formal state machine / business rule).
9. Across all 8 case studies, identify every instance of "deliberate redundancy" (a snapshot value duplicating another table's current value) and explain, for each, why it's justified rather than a Module 16-style mistake.

### Practice Links

- [dbdiagram.io](https://dbdiagram.io) — recreate any of these 8 case study schemas visually to practice reading SQL back into ER form.
- [LeetCode Database — Top SQL 50 (many problems use schemas structurally identical to these case studies — e.g., Orders/OrderItems, Employees/Departments)](https://leetcode.com/studyplan/top-sql-50/)
- [DataLemur — real company SQL interview questions, several modeled on e-commerce/social-media schemas nearly identical to Case Studies 3 and 6](https://datalemur.com/questions)

---

**Next: Module 18 — Interview Perspective (Frequently Asked Conceptual Questions, Scenario-Based Questions, Diagram-Based Questions, Schema Design Questions, Common Mistakes Interviewers Test For).**

> Say "continue" or "next module" to proceed to Module 18.
