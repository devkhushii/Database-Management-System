# Part 3 — The Complete SQL Handbook
## Module 5: SQL Data Types

> **Goal of this module:** A complete, exhaustive tour of every SQL data type category — numeric, character, boolean, date/time, binary, UUID, ENUM, JSON, XML, arrays, and spatial types — with storage sizes, ranges, engine-specific behavior, and the tradeoffs that drive real schema design decisions.

---

### 📍 Where This Module Fits

```
Module 4: Tables exist  →  Module 5: What can go IN them (You are here)  →  Module 6: Constraints (rules on that data)
```

---

## 5.1 Why Data Types Matter So Much

A **data type** constrains what values a column can hold, how much storage it consumes, how it's compared/sorted, and what operations are valid on it. Choosing the right type isn't cosmetic — it directly affects **storage size**, **query performance**, **data integrity**, and **correctness** of arithmetic/comparisons.

> 🧠 **Real-life analogy:** Data types are like the labeled compartments in a toolbox — a socket-wrench compartment physically cannot (and should not) hold nails. Forcing the wrong "shape" of data into a column is exactly like cramming a hammer into a slot meant for screws: it might technically fit somewhere, but everything downstream (search, sorting, comparisons) becomes unreliable.

Every type below will be explained with: what it is, why it exists, internal storage, SQL standard behavior, PostgreSQL vs MySQL differences, and practical guidance.

---

## 5.2 Numeric Types

### Integer Family

| Type | Storage | Range (signed) | Standard? |
|---|---|---|---|
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | ANSI standard |
| `INT` / `INTEGER` | 4 bytes | -2,147,483,648 to 2,147,483,647 | ANSI standard |
| `BIGINT` | 8 bytes | -9.2 quintillion to +9.2 quintillion | ANSI standard |

```sql
CREATE TABLE inventory (
    id       BIGINT,
    quantity INT,
    reorder_threshold SMALLINT
);
```

**Internal working:** Integers are stored as fixed-width binary two's-complement values — this is why arithmetic and comparisons on them are extremely fast (a single CPU instruction, essentially), and why exceeding the range causes an **overflow error** rather than silent wraparound in both PostgreSQL and MySQL (a deliberate safety behavior, unlike some lower-level languages).

> ⚠️ **Common mistake:** Using `INT` for a primary key on a table expected to grow past ~2 billion rows (e.g., a high-volume events/logs table). Once you hit the `INT` ceiling, every subsequent insert fails — a painful, downtime-causing migration to `BIGINT` after the fact. **Best practice:** default to `BIGINT`/`BIGSERIAL` for primary keys on any table you can't be certain will stay small, since the storage cost difference (4 vs 8 bytes) is trivial compared to the migration pain of getting it wrong.

### Exact Decimal Types: `DECIMAL` / `NUMERIC`

```sql
price NUMERIC(10, 2)   -- up to 10 total digits, 2 after the decimal point
```

`NUMERIC` and `DECIMAL` are **synonyms** in both PostgreSQL and MySQL (and per the ANSI standard). They store numbers **exactly**, digit by digit, rather than as an approximate binary representation.

**Why they exist:** Binary floating-point (`FLOAT`/`DOUBLE`, below) cannot represent most decimal fractions exactly (e.g., `0.1` in binary floating point is actually a very close approximation, not exactly 0.1). For **money, prices, and any value requiring exact decimal precision**, this approximation is unacceptable — a classic, frequently-tested reason.

```
FLOAT/DOUBLE (binary approximation):        NUMERIC/DECIMAL (exact):
0.1 + 0.2 = 0.30000000000000004 (!)         0.1 + 0.2 = 0.3 (exact)
```

> ⚠️ **Extremely common, extremely important interview point:** **Never use `FLOAT`/`DOUBLE` for currency.** This is one of the most universally agreed-upon rules in database schema design, and violating it is a classic beginner (and sometimes even mid-level) mistake that causes real accounting discrepancies in production financial systems.

### Approximate Types: `FLOAT`, `DOUBLE`, `REAL`

| Type | Storage | Precision |
|---|---|---|
| `REAL` / `FLOAT4` | 4 bytes | ~6 decimal digits |
| `DOUBLE PRECISION` / `FLOAT8` | 8 bytes | ~15 decimal digits |

**When to actually use them:** Scientific/engineering computations, sensor readings, machine learning feature storage — cases where a tiny approximation error is acceptable and raw computational speed/storage compactness matters more than exactness.

### PostgreSQL vs MySQL — Numeric Differences

| Aspect | PostgreSQL | MySQL |
|---|---|---|
| `NUMERIC` max precision | Effectively unlimited (up to 131,072 digits before the decimal point) | Max 65 digits total |
| Overflow behavior | Raises an error | Raises an error (strict mode) or may truncate/warn (non-strict mode — a real footgun) |
| `UNSIGNED` integers | **Not supported** — PostgreSQL has no unsigned integer types at all | Supported (`INT UNSIGNED`, doubling the positive range by disallowing negatives) |

> ⚠️ **Real gotcha:** Developers moving from MySQL to PostgreSQL are often surprised there's no `UNSIGNED` keyword in Postgres — the idiomatic Postgres workaround for "this should never be negative" is a `CHECK` constraint (`CHECK (quantity >= 0)`, covered fully in Module 6), not a type-level restriction.

---

## 5.3 Character Types

| Type | Description |
|---|---|
| `CHAR(n)` | **Fixed-length**, blank-padded to exactly `n` characters |
| `VARCHAR(n)` | **Variable-length**, up to `n` characters, no padding |
| `TEXT` | Variable-length, effectively unlimited (engine-dependent maximum) |

### CHAR vs VARCHAR — Internal Working

```sql
name CHAR(10)      -- 'Amit' is stored padded as 'Amit      ' (10 chars, space-padded)
name VARCHAR(10)   -- 'Amit' is stored as exactly 'Amit' (4 chars used, no padding)
```

`CHAR` always consumes its declared width on disk (in most engines); `VARCHAR` consumes only what's actually used, plus a small length-prefix overhead (typically 1–2 bytes).

> 🧠 **Real-life analogy:** `CHAR(10)` is like a fixed-size labeled box that's always exactly the same size on a shelf, even if you only put a small item inside (wasted space, but very predictable and fast to access by position). `VARCHAR` is like a resizable pouch — it only takes up as much room as its actual contents.

### PostgreSQL vs MySQL — Character Type Differences

| Aspect | PostgreSQL | MySQL |
|---|---|---|
| `TEXT` performance vs `VARCHAR` | **Identical** — PostgreSQL internally stores both the same way (TOAST mechanism for large values); there's essentially **no performance penalty** for using `TEXT` over `VARCHAR(n)` | Historically `TEXT` had some differences (can't have a default value pre-8.0 in some configs, indexing requires a prefix length); `VARCHAR` and `TEXT` are not fully interchangeable |
| Recommended default | **Just use `TEXT`** unless you have a genuine, enforced length business rule (PostgreSQL community's strong, well-known recommendation) | `VARCHAR(n)` is more idiomatically used, with explicit lengths |
| Trailing spaces in `CHAR` | Stripped on retrieval in some contexts, but stored padded | Similar behavior, engine-dependent nuances |

> 💡 **This surprises almost every MySQL-background engineer moving to PostgreSQL:** in PostgreSQL, `VARCHAR(n)` without a length (`VARCHAR`) behaves identically to `TEXT`, and even `VARCHAR(n)` with a length has no storage/performance advantage over `TEXT` with a separate `CHECK (length(col) <= n)` constraint — the length limit is purely a **constraint check**, not a storage optimization, in PostgreSQL's implementation. This is genuinely different from how many engineers assume types work, and it's a real interview talking point.

### Best Practices / Common Mistakes

- **Best practice:** Use `VARCHAR(n)` when there's a real, meaningful business-enforced maximum (e.g., a country code, a fixed-format ID); use `TEXT`/unbounded `VARCHAR` for free-form content (descriptions, comments).
- **Common mistake:** Using `CHAR(n)` for variable-length data like names — wastes storage and requires trimming logic that's easy to forget, causing subtle bugs when comparing padded vs unpadded strings.

---

## 5.4 Boolean Type

```sql
is_active BOOLEAN DEFAULT TRUE;
```

**PostgreSQL:** Has a genuine, native `BOOLEAN` type, storing `TRUE`, `FALSE`, or `NULL` (three-valued logic — foreshadowing `NULL` handling covered extensively later). Accepts many literal shorthand inputs (`'t'`, `'true'`, `'1'`, `'yes'` all parse as `TRUE`).

**MySQL:** Has **no true native boolean type**. `BOOLEAN`/`BOOL` are simply **aliases for `TINYINT(1)`** — a 1-byte integer. `TRUE` and `FALSE` are literally just aliases for the integers `1` and `0`.

> ⚠️ **Genuinely important, frequently tested difference:** In MySQL, `SELECT TRUE = 1;` returns true because they're the same underlying value — there is no real type-level distinction being enforced. In PostgreSQL, boolean is a first-class type, and `SELECT TRUE = 1;` actually raises a **type error** (`operator does not exist: boolean = integer`), because PostgreSQL's type system is much stricter about implicit conversions (a broader PostgreSQL characteristic we'll see repeat throughout this handbook).

---

## 5.5 Date & Time Types

| Type | Stores |
|---|---|
| `DATE` | Calendar date only (year, month, day) — no time component |
| `TIME` | Time of day only (hour, minute, second, fractional seconds) — no date |
| `TIMESTAMP` | Combined date **and** time, with no timezone awareness |
| `TIMESTAMPTZ` (`TIMESTAMP WITH TIME ZONE`) | Combined date and time, **timezone-aware** |
| `INTERVAL` | A **duration/span** of time (e.g., "3 days", "2 hours 30 minutes") — not a point in time |

### TIMESTAMP vs TIMESTAMPTZ — The Single Most Important Date/Time Concept

```sql
created_at  TIMESTAMP          -- stores exactly what you give it, no conversion
created_at  TIMESTAMPTZ         -- internally normalized to UTC, converted back to
                                 -- the requesting session's timezone on read
```

**Internal working of `TIMESTAMPTZ` in PostgreSQL:** Despite the name suggesting it "stores a timezone," it does **not** actually store the original timezone offset at all. What really happens: on `INSERT`, PostgreSQL converts the incoming value to **UTC** internally (using whatever timezone context was provided or assumed), and stores *only* that UTC instant. On `SELECT`, it converts that stored UTC instant into the **current session's `timezone` setting** for display. The original timezone the value was entered in is **permanently lost** — only the absolute instant in time is preserved.

```
INSERT (session timezone = 'America/New_York'):
  '2026-07-16 09:00:00' → stored internally as UTC: 2026-07-16 13:00:00

SELECT (session timezone changed to 'Asia/Kolkata'):
  UTC 13:00:00 → displayed as: 2026-07-16 18:30:00 (IST)
```

> 🧠 **Real-life analogy:** `TIMESTAMPTZ` is like recording an event's exact moment on a universal clock in space (UTC) — no matter which country's local clock you originally read it from, the *actual instant* is stored precisely. When anyone later "asks what time it was," the database translates that universal instant into *their own* local clock — but it never remembers which local clock the original observer used.

> ⚠️ **Extremely common, extremely damaging real-world mistake:** Using plain `TIMESTAMP` (no timezone) for things like `created_at`/`updated_at` columns in an application that serves users across multiple timezones or that runs its database server and application server in different timezone configurations. This leads to genuinely confusing bugs (events appearing to happen "in the future," inconsistent day-boundary calculations) that are a very common source of production incidents. **Best practice, near-universal industry consensus: default to `TIMESTAMPTZ` for anything representing "when did this actually happen," unless you have a specific reason to store naive/wall-clock time.**

### MySQL's Date/Time Peculiarities

| Aspect | Behavior |
|---|---|
| `TIMESTAMP` | Stored internally as UTC and converted based on the session's `time_zone` setting (similar spirit to Postgres's `TIMESTAMPTZ`) — but with a much **narrower valid range**: 1970-01-01 to 2038-01-19 (the classic "Year 2038 problem," inherited from 32-bit Unix time representation) |
| `DATETIME` | Stored exactly as given, **no timezone conversion at all** (closer in spirit to Postgres's plain `TIMESTAMP`) — much wider range: 1000-01-01 to 9999-12-31 |

> ⚠️ **This inverts beginners' intuitions and trips people up constantly:** in MySQL, it's `DATETIME` (not `TIMESTAMP`) that behaves like PostgreSQL's naive `TIMESTAMP`, and MySQL's `TIMESTAMP` is the timezone-aware one — the exact opposite naming convention from PostgreSQL's `TIMESTAMP` vs `TIMESTAMPTZ`. **Always double-check which engine you're in before assuming type-name intuitions carry over.**

### INTERVAL

```sql
SELECT CURRENT_DATE + INTERVAL '7 days';    -- PostgreSQL
SELECT DATE_ADD(CURDATE(), INTERVAL 7 DAY); -- MySQL (no native INTERVAL type; used only inline in date functions)
```

**PostgreSQL** has a genuine, storable `INTERVAL` **data type** — you can have a column of type `INTERVAL` holding a duration. **MySQL has no such storable type** — `INTERVAL` only exists as syntax used inline within date-arithmetic functions, not as something you can declare a column to be.

---

## 5.6 Binary Types

| Type | Purpose |
|---|---|
| `BYTEA` (PostgreSQL) | Raw binary data (byte arrays) |
| `BLOB` (MySQL) — with size variants `TINYBLOB`/`BLOB`/`MEDIUMBLOB`/`LONGBLOB` | Raw binary data, at various maximum sizes |

**Use cases:** Storing small binary blobs (thumbnails, encrypted tokens, checksums) directly in the database — though **industry best practice** generally favors storing large binary files (full images, videos, documents) in dedicated object storage (like S3) and keeping only a **reference/URL** in the database, since large binary blobs bloat table size, slow backups, and pollute the database's buffer cache with data that databases aren't optimized to serve efficiently.

> 💡 **Interview talking point:** Being asked "would you store user-uploaded images directly in the database?" is a great opportunity to explain this exact tradeoff — small, tightly-coupled binary data (a signature, a small icon) can be reasonable; large media files are almost always better in object storage with the database holding just a pointer.

---

## 5.7 UUID

```sql
id UUID PRIMARY KEY DEFAULT gen_random_uuid();   -- PostgreSQL (requires pgcrypto or built-in in 13+)
id CHAR(36) PRIMARY KEY;                          -- MySQL — no native UUID type; typically stored as CHAR(36) string,
                                                    -- or as BINARY(16) for a more compact representation
```

**What a UUID is:** A **Universally Unique Identifier** — a 128-bit value, conventionally displayed as a 36-character hyphenated hex string (e.g., `f47ac10b-58cc-4372-a567-0e02b2c3d479`), designed to be **globally unique without any central coordinating authority** (unlike auto-increment counters, which require a single authoritative counter source).

### Why UUIDs Exist / When to Use Them

| Auto-Increment ID | UUID |
|---|---|
| Requires a single authoritative source (one table's counter) | Can be generated **independently**, anywhere — client-side, in a distributed system, offline — with virtually zero collision probability |
| Predictable/sequential — can leak business information (e.g., "order #1002 was placed right after order #1001," revealing order volume) | Non-sequential, non-guessable — doesn't leak ordering/volume information |
| Compact (4–8 bytes) | Larger (16 bytes raw, 36 bytes as text) — real storage/index-size cost |
| Naturally clusters well in a B-tree index (sequential inserts) | **Random UUIDs cause index fragmentation** — a genuine, frequently-tested performance downside |

> ⚠️ **Senior-level performance gotcha:** Because standard random UUIDs (`UUIDv4`) insert in effectively random order into a B-tree primary key index, they cause significantly worse **index locality** than sequential auto-increment IDs — leading to more disk I/O and cache misses on large tables. This is precisely why newer standards like **UUIDv7** (time-ordered UUIDs, combining a timestamp prefix with randomness) have gained popularity — they preserve UUID's distributed-generation benefits while restoring much of the sequential-insert performance benefit. Being able to discuss this tradeoff is a strong signal in senior interviews.

**PostgreSQL vs MySQL:** PostgreSQL has a genuine native `UUID` type (16-byte compact storage, with built-in validation). MySQL has no native UUID type — it's conventionally stored as `CHAR(36)` (readable but storage-heavy) or `BINARY(16)` (compact but requires conversion functions like `UUID_TO_BIN()`/`BIN_TO_UUID()` to work with human-readable UUID strings).

---

## 5.8 ENUM

```sql
-- PostgreSQL — must be created as a named type first
CREATE TYPE order_status AS ENUM ('pending', 'shipped', 'delivered', 'cancelled');
CREATE TABLE orders (
    id INT PRIMARY KEY,
    status order_status NOT NULL DEFAULT 'pending'
);

-- MySQL — declared inline directly on the column
CREATE TABLE orders (
    id INT PRIMARY KEY,
    status ENUM('pending', 'shipped', 'delivered', 'cancelled') NOT NULL DEFAULT 'pending'
);
```

**What it is:** A type restricting a column to one of a fixed, predefined set of string values.

### PostgreSQL vs MySQL — A Meaningfully Different Implementation

| Aspect | PostgreSQL | MySQL |
|---|---|---|
| Definition style | A separate, named, reusable **custom type**, created once and referenced by multiple tables/columns | Declared **inline** per column — not reusable as a named type |
| Adding a new value later | `ALTER TYPE order_status ADD VALUE 'refunded';` — generally straightforward | Requires `ALTER TABLE ... MODIFY COLUMN` re-specifying the entire enum list — more disruptive |
| Internal storage | Stored efficiently as small integers internally, compared/sorted by their **declared definition order** (not alphabetically) | Also stored internally as small integers (1-based index into the defined list), same ordering behavior |
| Removing a value | **Not directly supported** in Postgres without recreating the type entirely — a real, commonly-hit limitation | Also awkward — requires a full column redefinition |

> ⚠️ **A widely-debated schema design tradeoff, worth knowing for interviews:** Many experienced engineers actually **avoid `ENUM` types** in production schemas, preferring instead a small **lookup/reference table** (a regular table listing valid statuses, referenced via a foreign key — covered in Module 6) specifically *because* `ENUM` values are awkward to add/remove/reorder later without disruptive schema migrations, whereas adding a row to a lookup table is a trivial `INSERT`. This is a genuinely good, frequently-referenced senior-level talking point about weighing type-level convenience against long-term schema flexibility.

---

## 5.9 JSON

```sql
-- PostgreSQL — two variants
data JSON;     -- stores an exact text copy of the input; re-parses every time it's queried
data JSONB;    -- stores a decomposed BINARY representation; parsed once at write time

-- MySQL (5.7+)
data JSON;     -- MySQL has one JSON type, internally stored in an optimized binary format
                -- (conceptually closer to Postgres's JSONB than its JSON)
```

### JSON vs JSONB (PostgreSQL) — A Critical, Frequently-Tested Distinction

| Aspect | `JSON` | `JSONB` |
|---|---|---|
| Storage | Exact text copy (preserves whitespace, key order, duplicate keys) | Decomposed binary format (no whitespace, no guaranteed key order, duplicate keys collapsed — last one wins) |
| Write speed | Slightly faster (no parsing at write time) | Slightly slower (parses/converts at write time) |
| Read/query speed | Slower (must re-parse the text on every access) | **Significantly faster** (already in a queryable binary structure) |
| Indexing support | Cannot be indexed with a GIN index directly | **Can be indexed** (GIN indexes — previewed here, fully covered in Part 4), enabling fast containment/key-lookup queries |
| Industry recommendation | Rarely used in practice; mainly when exact original text fidelity matters | **The overwhelming default choice** for real applications |

> 💡 **Extremely common, near-universal best practice in PostgreSQL:** Use `JSONB`, not `JSON`, unless you have a specific, unusual reason to preserve the exact original text formatting. This is one of the most frequently cited "gotcha" recommendations in PostgreSQL schema design discussions and interview answers.

### Why JSON Columns Exist in a Relational Database at All

This connects back to Part 2's normalization discussion: sometimes data is genuinely **semi-structured** — its shape varies row to row, or it's sourced from an external system with a flexible schema (e.g., arbitrary user-configurable metadata, webhook payloads, feature flags). Rather than forcing a rigid, heavily-normalized structure (many extra tables) for data that's inherently variable-shaped, a JSON column lets you keep the *core* structured data properly normalized while accommodating flexible, less-critical fields inline.

```sql
SELECT data->>'email' AS email          -- PostgreSQL: ->> extracts as text
FROM users
WHERE data @> '{"active": true}';       -- @> "contains" operator (uses GIN index if present)
```

> ⚠️ **Common mistake / anti-pattern:** Overusing JSON columns to avoid "doing normalization properly" — storing structured, relational, frequently-queried data (like an `orders` table's core `customer_id`, `total`, `status`) inside a JSON blob defeats the query optimizer's ability to use normal indexes efficiently and violates the normalization principles from Part 2. JSON columns are best reserved genuinely **semi-structured, sparse, or schema-flexible** data — not as a replacement for proper table design.

---

## 5.10 XML

```sql
data XML;   -- PostgreSQL native XML type, with xpath()-based querying functions
-- MySQL has no native XML type; XML is typically stored as TEXT and processed
-- with functions like ExtractValue()/UpdateXML(), which are more limited than
-- PostgreSQL's xpath support
```

**Industry usage note:** XML support exists (and is part of the SQL standard, added in SQL:2003/2006) largely for legacy/enterprise integration purposes — many older enterprise systems (especially in banking, insurance, government) exchange data as XML. For **new** application development, **JSON has almost entirely displaced XML** as the preferred semi-structured format, due to its lighter syntax and near-universal support across modern programming languages and web APIs. Expect XML columns mainly when interfacing with legacy enterprise systems, not in greenfield schema design.

---

## 5.11 Arrays

```sql
-- PostgreSQL — genuine native array type, works with ANY base type
tags TEXT[];
scores INT[];

CREATE TABLE articles (
    id INT PRIMARY KEY,
    tags TEXT[] DEFAULT '{}'
);

INSERT INTO articles (id, tags) VALUES (1, ARRAY['sql', 'database', 'tutorial']);

SELECT * FROM articles WHERE 'sql' = ANY(tags);   -- containment check
```

**MySQL has no native array type at all.** The idiomatic MySQL workarounds are: (a) a separate **junction/child table** (the properly normalized approach, covered in Part 2 and Module 12's joins), or (b) since MySQL 5.7+, storing the list inside a `JSON` array column and querying it with JSON functions.

> ⚠️ **A genuine, significant PostgreSQL vs MySQL feature gap:** Native arrays are a distinctly PostgreSQL capability among the two databases this handbook focuses on. This matters practically: schemas designed natively in PostgreSQL sometimes lean on array columns for genuinely list-shaped attributes (like `tags`), a pattern that has **no direct equivalent** when porting to MySQL — you'd need to redesign using a junction table or JSON, which is exactly the kind of practical portability issue Module 1's "dialects" discussion foreshadowed.

> 💡 **Best practice debate:** Even in PostgreSQL, where arrays are natively supported, many experienced engineers still prefer a proper normalized junction table over an array column for anything that needs to be **individually queried, indexed, joined against, or has its own attributes** (e.g., "when was this tag added," "who added it") — arrays are best reserved for simple, denormalized, rarely-queried-in-isolation lists.

---

## 5.12 Spatial Types (Overview)

Both PostgreSQL (via the **PostGIS** extension) and MySQL (with **native spatial type support** since 5.7, following the OpenGIS standard) support geographic/geometric data types: `POINT`, `LINESTRING`, `POLYGON`, and related types, along with spatial indexing (R-tree-based) and functions for distance, containment, and intersection calculations.

```sql
-- MySQL
location POINT;
SELECT ST_Distance_Sphere(location, POINT(77.5946, 12.9716)) AS distance_meters FROM stores;

-- PostgreSQL (requires: CREATE EXTENSION postgis;)
location GEOGRAPHY(POINT);
SELECT ST_Distance(location, ST_MakePoint(77.5946, 12.9716)) FROM stores;
```

> 💡 **Industry usage:** Spatial types power location-based features directly in the database — "find all restaurants within 5km" — common in ride-hailing, food delivery, and real-estate applications. **PostGIS** is widely regarded in the industry as the more mature, feature-rich, and standards-compliant spatial extension compared to MySQL's built-in spatial support, and is a genuinely popular reason some teams choose PostgreSQL specifically for location-heavy products. This is intentionally an overview here — deep spatial querying is a specialized topic outside this handbook's core SQL scope.

---

## 5.13 Storage Size & Type Selection — Consolidated Comparison Table

| Category | Type | Typical Storage | Best For |
|---|---|---|---|
| Integer | `SMALLINT` | 2 bytes | Small bounded counts (e.g., age, small enums-as-ints) |
| Integer | `INT` | 4 bytes | General-purpose IDs, counts |
| Integer | `BIGINT` | 8 bytes | Primary keys on large/unpredictable-growth tables |
| Exact decimal | `NUMERIC(p,s)` | Variable | **Money, exact-precision values** |
| Approximate | `DOUBLE` | 8 bytes | Scientific/statistical data, never currency |
| Text | `VARCHAR(n)` | Variable + overhead | Business-length-limited text |
| Text | `TEXT` | Variable | Free-form, unbounded text |
| Boolean | `BOOLEAN` | 1 byte | True/false flags (native in Postgres; `TINYINT(1)` alias in MySQL) |
| Date/time | `TIMESTAMPTZ` | 8 bytes | **Default choice for "when did this happen"** |
| Date/time | `DATE` | 4 bytes | Calendar dates without time relevance (birthdays, deadlines) |
| Identifier | `UUID` | 16 bytes | Distributed/offline ID generation, non-guessable IDs |
| Semi-structured | `JSONB` (PG) / `JSON` (MySQL) | Variable | Sparse, flexible-shape attributes |
| List | `ARRAY` (PostgreSQL only) | Variable | Simple, rarely-queried-in-isolation lists |

---

## 5.14 Best Practices (Type Selection)

- Default to `BIGINT` for primary keys unless you're certain the table will stay small.
- **Never** use `FLOAT`/`DOUBLE` for money — always `NUMERIC`/`DECIMAL`.
- Prefer `TIMESTAMPTZ` (PostgreSQL) / timezone-aware `TIMESTAMP` (MySQL) for event timestamps; reserve naive date/time types for genuinely timezone-agnostic concepts (a birthday is the same calendar date everywhere, regardless of timezone).
- In PostgreSQL, just use `TEXT` unless a real business rule caps the length — there's no storage/performance benefit to `VARCHAR(n)` there.
- In PostgreSQL, prefer `JSONB` over `JSON` in virtually all real application scenarios.
- Think carefully before choosing `ENUM` vs. a lookup table for anything expected to grow or change its valid values over the table's lifetime.

## 5.15 Common Mistakes

- Storing money as `FLOAT` (Section 5.2).
- Using naive `TIMESTAMP` for user-facing, multi-timezone event data (Section 5.5).
- Assuming `BOOLEAN` is strictly type-checked in MySQL the way it is in PostgreSQL (Section 5.4).
- Storing large binary files (images, videos) directly in the database instead of object storage (Section 5.6).
- Reaching for `ENUM` or a JSON blob to avoid "proper" relational modeling out of laziness, rather than genuine semi-structured need (Sections 5.8–5.9).

## 5.16 Edge Cases

- PostgreSQL's `NUMERIC` with no explicit precision/scale (`NUMERIC` alone) stores values with **arbitrary precision**, unbounded — useful but can silently consume more storage than expected if not explicitly constrained.
- MySQL's `TIMESTAMP` "Year 2038 problem" (Section 5.5) is a genuine, real constraint still relevant today for any system that might store far-future dates.
- Comparing `JSON` values for equality in PostgreSQL can behave unexpectedly due to exact-text storage (two JSON objects with identical data but different key order or whitespace won't be considered textually identical) — another reason `JSONB` (which normalizes representation) is generally preferred.

## 5.17 Performance Implications

- Fixed-width types (`INT`, `CHAR(n)`) generally allow denser, more predictable storage layouts and can be marginally faster to scan than variable-width types, though this difference is often negligible compared to indexing strategy (Part 4).
- `JSONB`'s binary format enables genuinely fast, indexable queries into semi-structured data — a real performance advantage over `JSON`'s re-parse-on-every-read behavior.
- Random UUIDs as primary keys can meaningfully degrade insert/index performance at scale compared to sequential integers, due to B-tree fragmentation (Section 5.7) — a real, quantifiable production concern at high write volumes.

## 5.18 Industry Usage

Type selection is a foundational schema design skill exercised on essentially every backend engineering team — the specific choices covered in this module (BIGINT vs INT for IDs, NUMERIC for money, TIMESTAMPTZ for events, JSONB for flexible metadata, UUID vs auto-increment tradeoffs) represent genuinely common, real decisions made when designing production schemas at companies of every size, from startups to large-scale platforms.

## 5.19 Interview Importance

| Level | Typical Questions |
|---|---|
| College/GATE | List numeric/character/date type categories and their storage sizes |
| Placement | "Why shouldn't you use `FLOAT` for currency?" |
| Mid-level | "Explain `TIMESTAMP` vs `TIMESTAMPTZ`." / "When would you use `JSONB` over a normalized table?" |
| Senior SDE-2/SDE-3 | "Discuss the performance tradeoffs of UUID vs auto-increment primary keys at scale." / "How would you design a schema needing both strict relational integrity and flexible per-row metadata?" |

---

## 5.20 Chapter Summary

- **Numeric types** split into exact (`INT`/`BIGINT`/`NUMERIC`) and approximate (`FLOAT`/`DOUBLE`) — money always uses exact types.
- **Character types** (`CHAR`/`VARCHAR`/`TEXT`) differ mainly in padding and declared limits; in PostgreSQL, `TEXT` is essentially always the pragmatic default.
- **`TIMESTAMPTZ`** stores an absolute instant (normalized to UTC internally) and converts on display — critical for correctness in any multi-timezone system; MySQL's `TIMESTAMP` vs `DATETIME` naming is the inverse of what Postgres engineers might expect.
- **UUIDs** enable distributed, unguessable ID generation at the cost of index locality and larger storage compared to auto-increment integers.
- **`ENUM`**, **`JSON`/`JSONB`**, and **arrays** all offer flexible, semi-structured alternatives to strict relational modeling — powerful when used deliberately, an anti-pattern when used to avoid proper schema design.
- PostgreSQL and MySQL diverge meaningfully across nearly every type category covered here — always verify engine-specific behavior rather than assuming type-name intuitions transfer directly.

---

## 5.21 Revision Notes / Cheat Sheet

```
Money/exact         → NUMERIC(p,s) / DECIMAL(p,s)     — NEVER FLOAT/DOUBLE
Large PK             → BIGINT / BIGSERIAL / BIGINT IDENTITY
Free text            → TEXT (Postgres) / VARCHAR or TEXT (MySQL)
Event timestamp      → TIMESTAMPTZ (Postgres) / TIMESTAMP (MySQL, tz-aware)
Naive date/time      → TIMESTAMP (Postgres) / DATETIME (MySQL)   ⚠ inverted naming!
Boolean               → BOOLEAN (native, Postgres) / TINYINT(1) alias (MySQL)
Distributed unique ID → UUID (native in Postgres; CHAR(36)/BINARY(16) in MySQL)
Flexible/sparse data  → JSONB (Postgres, preferred) / JSON (MySQL)
Fixed value set        → ENUM (both, differing implementations) OR lookup table (often preferred)
List of simple values  → ARRAY (Postgres only) / JSON array or junction table (MySQL)
```

---

## 5.22 Multiple Choice Questions (MCQs)

1. Which type should you use for storing currency values?
   a) `FLOAT` b) `DOUBLE` c) **`NUMERIC`/`DECIMAL`** d) `REAL`

2. In PostgreSQL, `TIMESTAMPTZ` internally stores values as:
   a) The original timezone offset given b) **UTC, converting on display to the session's timezone** c) Local server time always d) A string

3. Which MySQL date/time type behaves like PostgreSQL's naive `TIMESTAMP` (no timezone conversion)?
   a) `TIMESTAMP` b) **`DATETIME`** c) `DATE` d) `TIME`

4. Which JSON variant in PostgreSQL supports GIN indexing for fast queries?
   a) `JSON` b) **`JSONB`** c) Both equally d) Neither

5. A key performance downside of random UUIDs as primary keys is:
   a) They take more CPU to compare b) **They cause B-tree index fragmentation from non-sequential inserts** c) They can't be indexed at all d) They're not unique enough

---

## 5.23 Exercises

### 🟢 Beginner
1. Create a table with one column of each major category covered in this module (integer, decimal, text, boolean, date, timestamp) and insert one valid row.
2. Explain, in your own words, why `0.1 + 0.2` can produce an unexpected result with `FLOAT` but not with `NUMERIC`.
3. Look up and write down the storage size (in bytes) for `SMALLINT`, `INT`, and `BIGINT`.

### 🟡 Intermediate
4. Create a PostgreSQL table with a `JSONB` column, insert a row with nested JSON data, and write a query extracting one specific nested field.
5. Create the same logical table in both PostgreSQL and MySQL for storing a list of "tags" per article — using a native `ARRAY` in Postgres, and researching/implementing the MySQL equivalent (JSON array or junction table).
6. Demonstrate the `TIMESTAMPTZ` behavior from Section 5.5: insert a row, then change your session's timezone setting and re-select the same row, observing the displayed value change.

### 🔴 Advanced / Debugging
7. A junior engineer stores `price FLOAT` in a payments table and later discovers small discrepancies in reported totals. Explain the root cause and the fix, referencing Section 5.2.
8. Compare the tradeoffs of using `ENUM('pending','shipped','delivered')` vs. a separate `order_statuses` lookup table for an `orders.status` column. Under what future scenario would the lookup table clearly win?

### 🧩 Scenario-Based
9. You're designing a globally distributed e-commerce system where order IDs must be generated by services in multiple regions without a central coordinator. Would you choose auto-increment or UUID primary keys? Justify your answer using Section 5.7's tradeoffs, and mention UUIDv7 as a possible middle ground.

### 🏗️ Mini Project
10. Extend the mini-schema you built in Module 4's exercises (e.g., `students`/`courses`/`enrollments`) by explicitly choosing and justifying a data type for every single column, written as a short design-decision table (column name → type → one-sentence justification).

### 🔗 Practice Links
- [LeetCode Database Problems](https://leetcode.com/problemset/database/)
- [PostgreSQL Docs — Data Types](https://www.postgresql.org/docs/current/datatype.html)
- [MySQL Docs — Data Types](https://dev.mysql.com/doc/refman/8.0/en/data-types.html)
- [SQLBolt](https://sqlbolt.com/)

---

## ✅ Module 5 Complete

You now have a comprehensive command of SQL's data type system — the storage, correctness, and performance tradeoffs behind every major type category, and the real, frequently-tested differences between how PostgreSQL and MySQL implement each one.

**Next up: Module 6 — Constraints**, where you'll learn to enforce data integrity rules (`PRIMARY KEY`, `FOREIGN KEY`, `CHECK`, `UNIQUE`, `NOT NULL`) directly at the database level.

---

*Reply "continue" or "next module" when you're ready for Module 6.*
