# DBMS Handbook — Part 5: Database Design, Functional Dependencies, Normalization & Schema Design

## Module 16: Real-World Database Design

> **Where we are:** Modules 1–15 gave us every theoretical tool and every practical convention. This module is the **synthesis module** — applying all of it, end to end, to nine real, recognizable system types. For each system, we walk through: identifying entities, deriving functional dependencies, applying normalization, and explaining every non-obvious design decision. Treat this module as a template for how to approach ANY real-world schema design interview question.

---

## 16.1 Banking System

### Requirements & Entities
A bank needs to track customers, accounts (a customer can have multiple accounts; an account can — in joint accounts — have multiple customers), transactions, branches, and loans.

### Schema

```sql
Branch(branch_id PK, branch_name, ifsc_code UNIQUE, city, address)

Customer(customer_id PK, full_name, dob, email UNIQUE, phone UNIQUE, kyc_verified, created_at)

Account(account_id PK, account_number UNIQUE, branch_id FK→Branch, account_type
        CHECK(account_type IN ('SAVINGS','CURRENT','FIXED_DEPOSIT')),
        balance DECIMAL(15,2) NOT NULL DEFAULT 0, opened_at, status
        CHECK(status IN ('ACTIVE','FROZEN','CLOSED')))

Customer_Account(customer_id FK→Customer, account_id FK→Account,
                  relationship_type CHECK(relationship_type IN ('PRIMARY','JOINT')),
                  PRIMARY KEY(customer_id, account_id))
        -- junction table: supports joint accounts (many-to-many)

Transaction(transaction_id PK, account_id FK→Account,
            transaction_type CHECK(transaction_type IN ('DEPOSIT','WITHDRAWAL','TRANSFER_IN','TRANSFER_OUT')),
            amount DECIMAL(15,2) NOT NULL CHECK(amount > 0),
            related_account_id FK→Account NULL,  -- populated for transfers
            transaction_timestamp, balance_after DECIMAL(15,2))

Loan(loan_id PK, customer_id FK→Customer, branch_id FK→Branch,
     principal_amount DECIMAL(15,2), interest_rate DECIMAL(5,2),
     tenure_months, disbursed_at, status CHECK(status IN ('ACTIVE','CLOSED','DEFAULTED')))
```

### Normalization Decisions
- **`Customer_Account` junction table** is required precisely because Customer↔Account is many-to-many (joint accounts) — a naive single `customer_id` FK on `Account` would violate the actual business relationship (Module 1, Section 1.15's "many-to-many with attributes" edge case; here the extra attribute is `relationship_type`).
- **`Transaction.balance_after`** is a deliberate, justified **denormalization** (Module 14) — recomputing a running balance from the full transaction history on every query would be prohibitively expensive; storing the balance snapshot at transaction time is standard industry practice (and often legally required for audit trails), accepting that it must never be retroactively edited.
- **Never store `balance` as a computed/derived value only** — `Account.balance` is the authoritative current balance, updated transactionally alongside every `Transaction` insert (typically inside a single DB transaction with proper isolation, previewed in Module 19).
- **`amount DECIMAL(15,2)`, never FLOAT** — directly applying Module 15, Section 15.2's rule for monetary values.

---

## 16.2 E-commerce

### Requirements & Entities
Customers, products (with categories), orders (with multiple line items), payments, and shipping addresses.

### Schema

```sql
Customer(customer_id PK, name, email UNIQUE, phone, created_at)

Address(address_id PK, customer_id FK→Customer, address_line, city, state, postal_code,
         address_type CHECK(address_type IN ('BILLING','SHIPPING')), is_default)

Category(category_id PK, category_name, parent_category_id FK→Category NULL)
        -- self-referencing FK for nested categories (Module 6, Section 6.7-style
        -- self-referencing relationship, first flagged in Module 1, Section 1.15)

Product(product_id PK, sku UNIQUE, product_name, category_id FK→Category,
        price DECIMAL(10,2) NOT NULL CHECK(price >= 0), stock_quantity, is_active)

Order(order_id PK, customer_id FK→Customer, shipping_address_id FK→Address,
      order_status CHECK(order_status IN ('PLACED','CONFIRMED','SHIPPED','DELIVERED','CANCELLED')),
      order_date, total_amount DECIMAL(12,2))

Order_Item(order_item_id PK, order_id FK→Order, product_id FK→Product,
           quantity CHECK(quantity > 0),
           unit_price_at_purchase DECIMAL(10,2),   -- SNAPSHOT, not a live join to Product.price
           UNIQUE(order_id, product_id))

Payment(payment_id PK, order_id FK→Order UNIQUE, payment_method
        CHECK(payment_method IN ('CARD','UPI','NETBANKING','COD')),
        amount DECIMAL(12,2), payment_status CHECK(payment_status IN ('PENDING','SUCCESS','FAILED','REFUNDED')),
        paid_at)
```

### Normalization Decisions
- **`Order_Item.unit_price_at_purchase` is a deliberate historical snapshot**, NOT derived live from `Product.price` — this is the exact **historical/temporal data edge case** flagged in Module 1 (Section 1.15) and Module 15 (Section 15.10): if `Product.price` changes next month, historical orders must still show the price actually paid. This is *not* a normalization violation — it's recording a genuinely distinct fact ("the price at time of purchase") that is functionally independent of the current `Product.price`.
- **`Order.total_amount`** is a computed aggregate stored directly on the order — a justified denormalization (Module 14) avoiding the need to re-sum `Order_Item` rows on every order-summary view, especially valuable since an order's total, once placed, essentially never changes.
- **`Category.parent_category_id`** self-reference supports nested categories (Electronics → Laptops → Gaming Laptops) without needing a separate table — this is the self-referencing pattern with careful nullability (top-level categories have `NULL` parent).
- **`Order_Item` junction table with its own attributes** (`quantity`, `unit_price_at_purchase`) is exactly the "many-to-many relationship with attributes must become its own table" rule from Module 1, Section 1.15.

---

## 16.3 Hospital Management

### Requirements & Entities
Patients, doctors (with specializations), appointments, prescriptions, and admissions (for inpatient stays).

### Schema

```sql
Department(department_id PK, department_name UNIQUE, floor_number)

Doctor(doctor_id PK, name, specialization, department_id FK→Department,
       license_number UNIQUE, phone)

Patient(patient_id PK, name, dob, gender, blood_group, phone, emergency_contact)

Appointment(appointment_id PK, patient_id FK→Patient, doctor_id FK→Doctor,
            appointment_datetime, status CHECK(status IN ('SCHEDULED','COMPLETED','CANCELLED','NO_SHOW')),
            reason_for_visit)

Prescription(prescription_id PK, appointment_id FK→Appointment,
             medicine_name, dosage, duration_days, notes)
        -- one appointment can generate multiple prescription line items

Admission(admission_id PK, patient_id FK→Patient, admitting_doctor_id FK→Doctor,
          room_number, admitted_at, discharged_at NULL,
          admission_status CHECK(admission_status IN ('ADMITTED','DISCHARGED')))

Bill(bill_id PK, patient_id FK→Patient, admission_id FK→Admission NULL,
     appointment_id FK→Appointment NULL,
     total_amount DECIMAL(12,2), payment_status CHECK(payment_status IN ('PENDING','PAID')))
```

### Normalization Decisions
- **`Doctor.specialization` and `department_id` are NOT transitively derived** — a doctor's specialization is their own attribute (`doctor_id → specialization`), separate from which department they're currently assigned to (`doctor_id → department_id`); these are two independent FDs, not a transitive chain, because a department can house doctors of multiple specializations (e.g., "Cardiology" department could have both a cardiologist and a cardiac surgeon).
- **`Bill` has TWO nullable FKs (`admission_id`, `appointment_id`)** — this models the real-world fact that a bill can arise from either an outpatient appointment OR an inpatient admission, but not both simultaneously; a `CHECK` constraint (`CHECK ((admission_id IS NULL) != (appointment_id IS NULL))`) would enforce "exactly one must be set," a practical alternative to a full polymorphic-association redesign (Module 1, Section 1.15).
- **`Prescription` is a separate table from `Appointment`**, not columns on `Appointment` — because a single appointment can result in **multiple** prescribed medicines (a 1NF repeating-group violation, Module 8, if crammed into `medicine_1, medicine_2, ...` columns on `Appointment` itself).

---

## 16.4 University Management

### Requirements & Entities
Students, courses, instructors, enrollments (with grades), and departments.

### Schema

```sql
Department(department_id PK, department_name UNIQUE, hod_instructor_id FK→Instructor NULL)

Instructor(instructor_id PK, name, department_id FK→Department, email UNIQUE)

Student(student_id PK, name, dob, department_id FK→Department, enrollment_year)

Course(course_id PK, course_code UNIQUE, course_title, credits, department_id FK→Department)

Course_Offering(offering_id PK, course_id FK→Course, instructor_id FK→Instructor,
                semester, year, UNIQUE(course_id, semester, year))
        -- a course can be OFFERED in different semesters/years by different instructors

Enrollment(student_id FK→Student, offering_id FK→Course_Offering,
           grade, enrollment_date,
           PRIMARY KEY(student_id, offering_id))
```

### Normalization Decisions
- **`Course_Offering` exists as a distinct entity from `Course`** — this is a deliberately more sophisticated design than the simpler `Enrollment(student_id, course_id, grade)` example used throughout Modules 7–11. Real universities offer the *same* course across multiple semesters/years, potentially with different instructors — collapsing "course" and "offering" into one table would incorrectly tie a course's identity to a single semester's instructor, and would violate 2NF the moment we tried to also track which specific semester a grade was earned in.
- **`Department.hod_instructor_id` referencing `Instructor`, while `Instructor.department_id` references `Department`** — a legitimate **circular reference** between two tables (a department has one head-of-department instructor; an instructor belongs to one department) — requires careful handling: typically, one of the two FKs is made nullable and/or the tables are created before the constraint is added (or deferred constraint checking is used), since you can't insert a department with a HOD that doesn't exist yet, nor an instructor in a department that doesn't exist yet, without one side temporarily being NULL.
- **`Enrollment`'s composite key is `{student_id, offering_id}`**, not `{student_id, course_id}` — this is the direct, necessary consequence of separating `Course` from `Course_Offering`: a grade is earned for a specific offering (specific semester's instance), not for the abstract course.

---

## 16.5 Library Management

### Requirements & Entities
Books (potentially multiple copies), members, and loans (borrowing transactions), with due dates and fines.

### Schema

```sql
Author(author_id PK, name, nationality)

Book(book_id PK, isbn UNIQUE, title, publisher, publication_year, category)

Book_Author(book_id FK→Book, author_id FK→Author, PRIMARY KEY(book_id, author_id))
        -- many-to-many: a book can have multiple authors, an author writes multiple books

Book_Copy(copy_id PK, book_id FK→Book, shelf_location,
          condition CHECK(condition IN ('NEW','GOOD','WORN','DAMAGED')),
          status CHECK(status IN ('AVAILABLE','BORROWED','LOST','UNDER_REPAIR')))
        -- separate from Book: a library owns MULTIPLE PHYSICAL COPIES of the same book/ISBN

Member(member_id PK, name, email UNIQUE, membership_type CHECK(membership_type IN ('STUDENT','FACULTY','PUBLIC')),
       joined_at)

Loan(loan_id PK, copy_id FK→Book_Copy, member_id FK→Member,
     borrowed_at, due_date, returned_at NULL,
     fine_amount DECIMAL(8,2) DEFAULT 0)
```

### Normalization Decisions
- **`Book_Copy` is distinct from `Book`** — this is the single most important design decision here, and a very common real-world modeling mistake to get wrong. `Book(isbn, title, ...)` describes the abstract *work*; a library owns many physical *copies* of that work, each with its own condition and current borrowing status. Conflating these (tracking "is this book currently borrowed?" directly on `Book`) would incorrectly imply the library only owns ONE copy of each title.
- **`Book_Author` junction table** — books can have multiple authors (a many-to-many relationship), directly mirroring the classic pattern from Module 1, Section 1.15.
- **`Loan.fine_amount`** could theoretically be computed from `(returned_at - due_date) × daily_rate`, but is stored directly as a deliberate denormalization — since the fine, once assessed/paid, becomes a fixed historical fact that shouldn't retroactively change even if the library's daily fine rate policy changes later (the same historical-snapshot reasoning as Section 16.2's `unit_price_at_purchase`).

---

## 16.6 Inventory System

### Requirements & Entities
Warehouses, products (SKUs), stock levels per warehouse, and stock movements (in/out transactions).

### Schema

```sql
Warehouse(warehouse_id PK, warehouse_name, location, capacity)

Product(product_id PK, sku UNIQUE, product_name, unit_of_measure, reorder_threshold)

Stock(warehouse_id FK→Warehouse, product_id FK→Product, quantity_on_hand NOT NULL DEFAULT 0,
      PRIMARY KEY(warehouse_id, product_id))
        -- CURRENT snapshot of stock per warehouse per product

Stock_Movement(movement_id PK, warehouse_id FK→Warehouse, product_id FK→Product,
                movement_type CHECK(movement_type IN ('INBOUND','OUTBOUND','TRANSFER_IN','TRANSFER_OUT','ADJUSTMENT')),
                quantity, related_warehouse_id FK→Warehouse NULL,  -- for transfers
                movement_timestamp, reference_document)
```

### Normalization Decisions
- **`Stock` is a materialized current-state table, `Stock_Movement` is the full audit log** — this is a deliberate, textbook denormalization pattern (Module 14): `Stock.quantity_on_hand` could theoretically be derived by summing all relevant `Stock_Movement` rows, but doing so on every inventory check would be prohibitively expensive at scale. `Stock` is kept in sync via application logic or triggers every time a `Stock_Movement` is recorded (Module 14, Section 14.10) — this is precisely the "pre-computed aggregate" denormalization technique.
- **`Stock`'s composite key `{warehouse_id, product_id}`** correctly models that stock levels are tracked **per warehouse**, not globally — a product can be in stock at one warehouse and out of stock at another.
- **`Stock_Movement.related_warehouse_id`** handles inter-warehouse transfers, avoiding a separate polymorphic table while keeping the common single-warehouse movements (inbound/outbound) simple.

---

## 16.7 Food Delivery

### Requirements & Entities
Customers, restaurants (with menu items), orders, delivery agents, and order status tracking.

### Schema

```sql
Restaurant(restaurant_id PK, name, address, city, cuisine_type, is_open, rating DECIMAL(3,2))

Menu_Item(item_id PK, restaurant_id FK→Restaurant, item_name, price DECIMAL(8,2), is_available)

Customer(customer_id PK, name, phone UNIQUE, default_address)

Delivery_Agent(agent_id PK, name, phone UNIQUE, vehicle_type, current_status
               CHECK(current_status IN ('AVAILABLE','ON_DELIVERY','OFFLINE')))

Order(order_id PK, customer_id FK→Customer, restaurant_id FK→Restaurant,
      delivery_agent_id FK→Delivery_Agent NULL,
      order_status CHECK(order_status IN ('PLACED','ACCEPTED','PREPARING','OUT_FOR_DELIVERY','DELIVERED','CANCELLED')),
      order_placed_at, total_amount DECIMAL(10,2), delivery_address)

Order_Item(order_id FK→Order, item_id FK→Menu_Item, quantity,
           price_at_order_time DECIMAL(8,2),   -- snapshot, same reasoning as e-commerce
           PRIMARY KEY(order_id, item_id))

Order_Status_History(history_id PK, order_id FK→Order, status, changed_at)
        -- append-only log tracking every status transition, for real-time tracking + analytics
```

### Normalization Decisions
- **`Order_Status_History` as a separate append-only log**, rather than only `Order.order_status` — this supports the real-time tracking UX ("preparing → out for delivery → delivered" timeline) that food delivery apps prominently feature; `Order.order_status` remains the fast, current-state lookup (denormalized current value), while the history table is the full audit trail — another instance of the "current state table + full history log" pattern from Section 16.6.
- **`Delivery_Agent.current_status`** is intentionally a fast, frequently-updated denormalized field (not derived from a join against active orders) precisely because agent-matching algorithms need to query "who is AVAILABLE right now" extremely frequently and with low latency — a strong justified case for denormalization (Module 14, Section 14.2).
- **`Order_Item.price_at_order_time`** — identical historical-snapshot reasoning as the e-commerce and library examples above; menu prices change over time and shouldn't retroactively alter historical order totals.

---

## 16.8 Ride Sharing

### Requirements & Entities
Riders, drivers (with vehicles), ride requests, and trip records (with fare calculation).

### Schema

```sql
Rider(rider_id PK, name, phone UNIQUE, rating DECIMAL(3,2))

Driver(driver_id PK, name, phone UNIQUE, license_number UNIQUE, rating DECIMAL(3,2),
       current_status CHECK(current_status IN ('AVAILABLE','ON_TRIP','OFFLINE')))

Vehicle(vehicle_id PK, driver_id FK→Driver UNIQUE, license_plate UNIQUE,
        vehicle_type CHECK(vehicle_type IN ('SEDAN','SUV','HATCHBACK','BIKE')))
        -- 1:1 with Driver in this simplified model (one active vehicle per driver)

Ride(ride_id PK, rider_id FK→Rider, driver_id FK→Driver NULL,
     pickup_location, dropoff_location,
     requested_at, accepted_at NULL, started_at NULL, completed_at NULL,
     ride_status CHECK(ride_status IN ('REQUESTED','ACCEPTED','IN_PROGRESS','COMPLETED','CANCELLED')),
     fare_amount DECIMAL(8,2) NULL, distance_km DECIMAL(6,2) NULL)

Payment(payment_id PK, ride_id FK→Ride UNIQUE, amount DECIMAL(8,2),
        payment_method CHECK(payment_method IN ('CARD','WALLET','CASH')),
        payment_status CHECK(payment_status IN ('PENDING','SUCCESS','FAILED')))
```

### Normalization Decisions
- **`Ride.driver_id` is nullable** — models the real lifecycle where a ride is `REQUESTED` before any driver has accepted it; this is the "optional relationship" edge case flagged in Module 1, Section 1.15.
- **`Ride` stores multiple timestamp columns (`requested_at`, `accepted_at`, `started_at`, `completed_at`)** rather than a single generic "timestamp" — each represents a genuinely distinct business event, and this is preferable to a separate `Ride_Status_History` table (unlike the food-delivery case, Section 16.7) specifically because a ride's lifecycle has a small, fixed number of well-defined stages (unlike food delivery's more variable, potentially-repeated status changes) — a good illustration of how the "current state + history log" pattern isn't a one-size-fits-all rule; it's applied where the specific access pattern (frequent, detailed timeline display) actually justifies it.
- **`Vehicle` modeled as effectively 1:1 with `Driver`** in this simplified version — a real system might support a driver owning multiple vehicles (used on different days), which would require making `Vehicle.driver_id` non-unique and adding an `is_active` flag, or tracking vehicle assignment per-ride instead of per-driver; the simplified 1:1 version is a reasonable starting design, explicitly flagged as an area for future extension (Module 1, Section 1.12's lifecycle/evolution theme).
- **`Payment.ride_id UNIQUE`** enforces the 1:1 relationship between a ride and its payment.

---

## 16.9 Social Media

### Requirements & Entities
Users, posts, comments, likes, and follower relationships (a self-referencing many-to-many).

### Schema

```sql
User(user_id PK, username UNIQUE, email UNIQUE, bio, joined_at)

Post(post_id PK, user_id FK→User, content, media_url NULL, posted_at, like_count DEFAULT 0)

Comment(comment_id PK, post_id FK→Post, user_id FK→User, content, commented_at)

Like(user_id FK→User, post_id FK→Post, liked_at, PRIMARY KEY(user_id, post_id))
        -- many-to-many junction: a user can like many posts; a post can be liked by many users

Follow(follower_id FK→User, followee_id FK→User, followed_at,
       PRIMARY KEY(follower_id, followee_id),
       CHECK(follower_id != followee_id))
        -- self-referencing many-to-many: users follow other users
```

### Normalization Decisions
- **`Follow` is a self-referencing many-to-many junction table on `User`** — this is the "self-referencing relationship" pattern (Module 1, Section 1.15) applied to a many-to-many case (unlike the simpler `Employee.manager_id` one-to-many self-reference example used earlier in this Part); the `CHECK(follower_id != followee_id)` constraint prevents the nonsensical "a user follows themselves" case.
- **`Post.like_count`** is a deliberately denormalized, pre-computed aggregate (Module 14) — recomputing `COUNT(*)` from the `Like` table on every single post render (which happens extremely frequently in a social feed) would be a severe performance bottleneck; `like_count` is kept in sync via a trigger or application-level increment/decrement whenever a `Like` row is inserted/deleted. This exact pattern is the "pre-computed aggregate" denormalization technique from Module 14, Section 14.9, applied to arguably the single most performance-critical read path in the entire schema.
- **`Like` junction table has NO surrogate key of its own** — `{user_id, post_id}` is a perfectly good, natural composite candidate key (a user can only like a given post once), so introducing an unnecessary surrogate key here would add complexity without benefit (Module 6, Section 6.7's key-choice guidance, applied in the opposite direction — sometimes the natural key genuinely IS the better choice).

---

### 16.10 Cross-Cutting Themes Across All Nine Systems

| Theme | Where It Appeared |
|---|---|
| **Junction tables for many-to-many, sometimes with their own attributes** | Banking (Customer_Account), E-commerce (Order_Item), Library (Book_Author), Social Media (Like, Follow) |
| **Historical/snapshot denormalization for prices/fines that must not retroactively change** | E-commerce, Library, Food Delivery |
| **Current-state table + full history/audit log, applied selectively** | Inventory (Stock + Stock_Movement), Food Delivery (Order.status + Order_Status_History) |
| **Pre-computed aggregate denormalization for extremely hot read paths** | Social Media (like_count), Inventory (Stock.quantity_on_hand), Banking (Account.balance) |
| **Self-referencing relationships (1:many and many:many)** | E-commerce (Category), University (Department↔Instructor), Social Media (Follow) |
| **Optional/nullable relationships modeling real business lifecycles** | Ride Sharing (Ride.driver_id), Hospital (Bill's two nullable FKs) |
| **Distinguishing an abstract "thing" from its concrete instances** | Library (Book vs Book_Copy), University (Course vs Course_Offering) |

**This table itself is a valuable interview artifact** — recognizing these recurring patterns across superficially different domains is exactly the skill senior-level "design me a schema for X" interview questions are testing.

---

### 16.11 Interview Importance

| Level | Sample Expectation |
|---|---|
| Beginner | Design a simple 3–4 table schema for one of these systems, get the obvious entities and FKs right. |
| Intermediate | Identify at least one many-to-many relationship requiring a junction table, and one nullable/optional relationship. |
| Senior SDE-2/SDE-3 | Design a full schema live, explicitly calling out normalization decisions, at least one deliberate denormalization with justification, and at least one non-obvious edge case (self-reference, polymorphic-ish relationship, or historical snapshot). |

---

### 16.12 Exercises

**Whiteboard Design Practice (pick any 2–3 and design independently before comparing to this module):**
1. Design a **Movie Ticket Booking** system (theaters, screens, showtimes, seats, bookings) — pay special attention to how you'd prevent double-booking the same seat for the same showtime (a uniqueness constraint design question).
2. Design a **Hotel Booking** system (hotels, room types, rooms, reservations) — consider how you'd model date-range availability without allowing overlapping bookings for the same room.
3. Design a **Job Portal** (companies, job postings, candidates, applications) — identify the many-to-many relationship and any status-tracking history needs.
4. Extend the **Ride Sharing** schema (Section 16.8) to support a driver owning multiple vehicles, used on different days — redesign the `Vehicle`/`Driver` relationship accordingly.
5. Extend the **Social Media** schema (Section 16.9) to support comment replies (a comment can reply to another comment) — identify what kind of self-referencing pattern this requires and any recursion-depth considerations worth discussing.

**Interview Questions:**
- *Senior:* "Design a database for [any system]. Walk me through your entities, your key relationships, and at least one place you'd consider denormalizing — and why."

---

**End of Module 16.**

*Next up — Module 17: Industry Trade-offs — Normalization vs Performance, OLTP vs OLAP design, and an introduction to Star and Snowflake Schemas, connecting everything in this Part to the world of analytical/data-warehouse design. Reply "next" / "continue" when ready.*
