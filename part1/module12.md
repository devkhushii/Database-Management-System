# The Complete DBMS Handbook — Part 1

## Module 12: Types of Databases

> **Module Roadmap:** Several database types have already surfaced across this handbook — the relational model (Module 2.7), NoSQL categories (Module 2.9), hierarchical/network models (Module 2.5-2.6), and distributed/cloud databases (Module 2.11-2.12). This module doesn't repeat those explanations from scratch — instead, it gives you the **complete, organized classification**, grouped along clear axes (by data model, by distribution, by deployment/purpose), with fresh coverage of the types not yet fully explored (Federated, Time Series, In-Memory, Embedded) and a master comparison table tying everything together. Think of this as the "periodic table" of database types — Module 2 told you the historical story; this module organizes the elements.

---

### 12.0 Three Axes of Classification

Database types aren't a single flat list — they can be classified along **three independent axes** simultaneously. A single real-world database can belong to one category from each axis at once (e.g., MongoDB is a **Document** database, that is typically **Distributed**, and is available as both **self-hosted and Cloud**).

```
┌─────────────────────────────────────────────────────────────┐
│   AXIS 1: DATA MODEL          AXIS 2: DISTRIBUTION              │
│   (how data is structured)     (how/where data physically lives)  │
│                                                                    │
│   • Relational                 • Centralized                         │
│   • Hierarchical                • Distributed                          │
│   • Network                      • Federated                            │
│   • Object-Oriented                                                       │
│   • Graph                                                                  │
│   • Key-Value                                                               │
│   • Document                                                                 │
│   • Column-Family                                                              │
│                                                                                   │
│   AXIS 3: SPECIALIZED PURPOSE / DEPLOYMENT                                          │
│   • Time Series      • In-Memory      • Cloud      • Embedded                          │
└─────────────────────────────────────────────────────────────┘
```

---

### 12.1 Axis 1 — By Data Model

#### 12.1.1 Relational Databases *(full detail: Module 2.7)*
Data stored as tables (rows/columns), related via keys, queried with SQL. **Examples:** PostgreSQL, MySQL, Oracle.

#### 12.1.2 Hierarchical Databases *(full detail: Module 2.5)*
Tree structure, one parent per child. **Examples:** IBM IMS (legacy).

#### 12.1.3 Network Databases *(full detail: Module 2.6)*
Graph-like structure, multiple parents allowed. **Examples:** IDMS (legacy, CODASYL-based).

#### 12.1.4 Object-Oriented Databases *(full detail: Module 2.8)*
Data stored as native programming objects, with inheritance. **Examples:** db4o, ObjectDB (niche use today).

#### 12.1.5 Graph Databases

**Definition:** A database optimized specifically for storing and querying **relationships** as first-class citizens — data is represented as **nodes** (entities) and **edges** (relationships between entities), often with properties attached to both.

**Why it exists:** Relational databases *can* model relationships (via foreign keys and JOINs), but deeply interconnected data (social networks, recommendation engines, fraud detection networks) becomes slow to query as JOIN depth increases. Graph databases are purpose-built to traverse relationships efficiently, regardless of depth.

**Example (conceptual):**
```
   (Ravi)──[FRIENDS_WITH]──►(Anita)
     │                          │
   [WORKS_AT]                [WORKS_AT]
     ▼                          ▼
  (TechCorp)◄──[WORKS_AT]───(Faisal)
```
A query like "find all friends-of-friends who work at the same company" is a fast, natural traversal in a graph database, but requires multiple expensive JOINs in a relational database.

**Examples:** Neo4j, ArangoDB, Amazon Neptune.
**Best for:** Social networks, recommendation engines, fraud detection, knowledge graphs.

#### 12.1.6 Key-Value Databases *(full detail: Module 2.9)*
Simplest NoSQL model — every value is retrieved by a unique key, like a giant dictionary/hashmap. **Examples:** Redis, Amazon DynamoDB, Riak.

#### 12.1.7 Document Databases *(full detail: Module 2.9)*
Semi-structured, JSON-like documents, flexible schema. **Examples:** MongoDB, CouchDB, Firebase Firestore.

#### 12.1.8 Column-Family (Wide-Column) Databases *(full detail: Module 2.9)*
Optimized for very large-scale writes and specific column-based access patterns. **Examples:** Cassandra, HBase, Google Bigtable.

---

### 12.2 Axis 2 — By Distribution

#### 12.2.1 Centralized Databases

**Definition:** All data resides on a single machine/server — no distribution across multiple locations.

**Advantages:** Simple to manage, no distributed-systems complexity, strong consistency by default (no coordination needed between nodes).
**Disadvantages:** Single point of failure, limited by that one machine's hardware capacity — connects back to Module 8.7 (Failure Impact) and Module 6.9 (Scalability limits).
**Example use case:** A small business's internal inventory system running on one server.

#### 12.2.2 Distributed Databases *(full detail: Module 2.12)*
Data spread across multiple physical machines/locations, appearing as one logical database. **Examples:** Google Spanner, Cassandra, CockroachDB.

#### 12.2.3 Federated Databases

**Definition:** A system that provides a **unified, virtual view** over multiple *independent, autonomous* databases (which may even be different DBMS products entirely) — without physically merging their data into one repository.

**Why it exists:** Large organizations (especially after mergers/acquisitions) often end up with multiple separate, independently-managed databases (e.g., one division running Oracle, another running PostgreSQL) that still need to be queried together for company-wide reporting, without the massive effort of physically consolidating them into one system.

**Diagram:**
```
              ┌─────────────────────────┐
              │   Federated Query Layer    │
              │  (unified virtual view)     │
              └──────────┬─────────────┘
             ┌────────────┼────────────┐
             ▼               ▼               ▼
      ┌───────────┐  ┌───────────┐  ┌───────────┐
      │ Division A   │  │ Division B   │  │ Division C   │
      │ (Oracle DB)   │  │ (PostgreSQL)  │  │ (MySQL DB)    │
      │ - independent │  │ - independent │  │ - independent │
      │   & autonomous│  │   & autonomous│  │   & autonomous│
      └───────────┘  └───────────┘  └───────────┘
```

**Real-life analogy:** A federated database is like an international trade alliance (e.g., a group of independent countries with their own governments and laws) that agrees to a shared, unified customs/trade process — each country remains fully autonomous internally, but external queries ("what's the alliance's total trade volume?") can be answered through the shared layer.

**Key distinction from Distributed Databases:** Distributed databases are typically **owned and centrally coordinated by one entity**, deliberately split for performance/scale (12.2.2). Federated databases are **independently owned/managed systems** that choose to expose a unified query interface while remaining otherwise autonomous.

**Interview Importance:** ⭐⭐⭐ (Common "compare and contrast" question against Distributed Databases.)

---

### 12.3 Axis 3 — By Specialized Purpose / Deployment

#### 12.3.1 Time Series Databases

**Definition:** A database specifically optimized for storing and querying data points indexed by **time** — typically massive volumes of sequential, timestamped measurements.

**Why it exists:** General-purpose relational databases can technically store timestamped data, but Time Series databases are purpose-optimized for the specific access patterns of this data: extremely high write throughput (constant new sensor readings), efficient time-range queries ("average temperature over the last hour"), and specialized compression for sequential numeric data.

**Example data:**
```
timestamp             | sensor_id | temperature
2026-07-16 10:00:00   |   S1       |   24.5
2026-07-16 10:00:01   |   S1       |   24.6
2026-07-16 10:00:02   |   S1       |   24.5
... (millions of rows per day, per sensor)
```

**Examples:** InfluxDB, TimescaleDB (a PostgreSQL extension), Prometheus.
**Best for:** IoT sensor data, financial market data (stock ticks), application performance monitoring/metrics.

#### 12.3.2 In-Memory Databases

**Definition:** A database that stores its primary data set entirely (or predominantly) in **RAM** rather than on disk, dramatically reducing access latency at the cost of higher hardware (memory) expense and, without careful design, higher risk of data loss on power failure.

**Why it exists:** Disk I/O (even fast SSDs) is significantly slower than RAM access. For use cases demanding extreme speed — caching, real-time leaderboards, session storage — keeping data in memory eliminates disk I/O as a bottleneck entirely.

**Trade-off — Durability:** Pure in-memory databases risk losing data on a crash/power loss unless they implement additional persistence mechanisms (e.g., Redis's optional snapshotting or append-only file logging) — directly connecting back to the **Durability** ACID property (previewed in Module 1, deep-dived later) and the general **Persistence** characteristic (Module 6.11).

**Examples:** Redis, Memcached, SAP HANA (hybrid in-memory + disk).
**Best for:** Caching, session stores, real-time analytics, leaderboards.

#### 12.3.3 Cloud Databases *(full detail: Module 2.11)*
Fully managed, provider-hosted databases (Database-as-a-Service). **Examples:** Amazon RDS, Google Cloud SQL, MongoDB Atlas.

#### 12.3.4 Embedded Databases

**Definition:** A lightweight database engine that runs **directly within the application's own process**, with no separate server process, no network communication, and typically minimal configuration — closely related to the "One-Tier Architecture" concept from Module 9.2.

**Why it exists:** Many applications (mobile apps, desktop software, small IoT devices) need reliable, structured local data storage, but running and maintaining a full, separate client-server DBMS (like PostgreSQL) would be excessive overhead, complexity, and resource cost for their needs.

**Example:** SQLite is embedded directly inside billions of mobile apps and web browsers — when your phone's messaging app stores your chat history locally, it's very likely using an embedded database like SQLite under the hood, with zero separate "database server" running anywhere.

**Examples:** SQLite, Realm, LevelDB, RocksDB.
**Best for:** Mobile apps, desktop applications, browsers, small IoT/edge devices, local caching layers within larger applications.

---

## 📊 Master Comparison Table — All Types at a Glance

| Type | Axis | Structure | Example Systems | Best For |
|---|---|---|---|---|
| Relational | Data Model | Tables (rows/cols) | PostgreSQL, MySQL, Oracle | General-purpose transactional systems |
| Hierarchical | Data Model | Tree | IBM IMS | Legacy, strictly nested data |
| Network | Data Model | Graph (multi-parent) | IDMS | Legacy, complex relationships (pre-relational era) |
| Object-Oriented | Data Model | Native objects | db4o | Complex object-graph applications (niche) |
| Graph | Data Model | Nodes & edges | Neo4j, Amazon Neptune | Social networks, recommendations, fraud detection |
| Key-Value | Data Model | Key → value pairs | Redis, DynamoDB | Caching, session storage |
| Document | Data Model | JSON-like documents | MongoDB, CouchDB | Content management, flexible schemas |
| Column-Family | Data Model | Wide columns | Cassandra, HBase | Massive-scale writes, time-series-like data |
| Centralized | Distribution | Single machine | Any single-server DB | Small-scale, simple systems |
| Distributed | Distribution | Multiple coordinated machines | Google Spanner, Cassandra | Global-scale, high-availability systems |
| Federated | Distribution | Multiple autonomous, independently-owned DBs | Custom federation layers | Cross-organizational/merged-entity querying |
| Time Series | Purpose | Timestamp-indexed | InfluxDB, TimescaleDB | IoT, monitoring, financial tick data |
| In-Memory | Purpose | RAM-resident | Redis, Memcached | Extreme-speed caching, real-time systems |
| Cloud | Deployment | Managed, provider-hosted | Amazon RDS, MongoDB Atlas | Elastic, low-ops-overhead deployments |
| Embedded | Deployment | In-process, serverless | SQLite, Realm | Mobile/desktop apps, edge devices |

```
   A SINGLE real-world database can be MULTIPLE types at once:

   MongoDB  =  Document (Data Model) + Distributed (Distribution)
               + available as Cloud (MongoDB Atlas) or self-hosted

   Redis    =  Key-Value (Data Model) + In-Memory (Purpose)
               + can be Centralized or Distributed (Redis Cluster)

   SQLite   =  Relational (Data Model) + Embedded (Deployment)
               + inherently Centralized (single-file, single-process)
```

**Best Practices:**
- When choosing a database for a project, explicitly reason across all three axes: "What data model fits my access patterns? Do I need distribution for scale/availability? Do I need a specialized purpose-built engine (time series, in-memory) or an embedded option?"

**Common Mistakes:**
- Treating "SQL vs NoSQL" as the only decision axis — as this module shows, distribution and specialized purpose are equally important, independent dimensions that cut across the SQL/NoSQL divide.

**Interview Importance:** ⭐⭐⭐⭐⭐ ("What types of databases are there, and when would you use each?" is a cornerstone system design interview question.)

---

## 📌 Chapter Summary

- Database types can be classified along **three independent axes**: **Data Model** (Relational, Hierarchical, Network, Object-Oriented, Graph, Key-Value, Document, Column-Family), **Distribution** (Centralized, Distributed, Federated), and **Specialized Purpose/Deployment** (Time Series, In-Memory, Cloud, Embedded).
- A real-world database product typically sits at the intersection of multiple axes simultaneously (e.g., MongoDB is Document + Distributed + available as Cloud).
- **Graph databases** excel where relational JOINs become too slow for deeply interconnected data; **Federated databases** unify independently-owned systems without physically merging them (distinct from Distributed databases, which are centrally coordinated); **Time Series** and **In-Memory** databases are purpose-optimized for specific, demanding access patterns; **Embedded databases** (like SQLite) eliminate the need for a separate server process entirely, ideal for mobile/desktop/edge use cases.

---

## 🧾 Cheat Sheet

| Type | One-Line Definition |
|---|---|
| Graph DB | Nodes + edges, optimized for relationship traversal |
| Centralized DB | All data on one machine |
| Federated DB | Unified view over independent, autonomous DBs |
| Time Series DB | Optimized for timestamp-indexed data |
| In-Memory DB | Data lives in RAM for extreme speed |
| Embedded DB | Runs inside the app process, no separate server |

---

## 📝 Exercises

### Beginner
1. Name the three axes along which database types can be classified in this module.
2. What is the key advantage of a Graph database over a relational database for deeply interconnected data?
3. True/False: A single real-world database product can only ever be classified under one axis.

### Intermediate
4. Explain the difference between a Distributed database and a Federated database, with an example scenario for each.
5. Why might a mobile app development team choose SQLite (Embedded) over connecting to a remote PostgreSQL server?
6. What is the main trade-off (mentioned in this module) that In-Memory databases must address regarding data durability?

### Advanced / Conceptual
7. A large company just acquired three smaller companies, each with its own independently-run database on a different DBMS product. Explain, using this module's concepts, how a Federated database approach could help them get unified reporting without a risky, disruptive full data migration.
8. Compare a Time Series database to a general-purpose relational database for storing IoT sensor data — what specific access patterns make the specialized Time Series option a better fit at scale?
9. Classify Redis along all three axes discussed in this module (Data Model, Distribution, Purpose/Deployment), explaining your reasoning for each.

### Scenario-Based / Case Study
10. A social media startup is building a "people you may know" feature that requires traversing friends-of-friends relationships up to 3 levels deep, at scale. Using the concepts in section 12.1.5, explain why a Graph database might outperform a purely relational approach for this specific feature, while the rest of their application (user profiles, posts) might still reasonably use a relational database.

### MCQs
11. Which type of database is specifically optimized for storing and querying nodes and relationships?
    a) Document Database
    b) Graph Database
    c) Column-Family Database
    d) Time Series Database

12. A database that provides a unified view over multiple independent, autonomously-managed databases is called a:
    a) Distributed Database
    b) Centralized Database
    c) Federated Database
    d) In-Memory Database

13. SQLite is best classified as which type of database (by deployment)?
    a) Cloud Database
    b) Embedded Database
    c) Distributed Database
    d) Federated Database

*(Answer key: 11-b, 12-c, 13-b)*

### Fill in the Blanks
14. A __________ database is optimized for high-throughput, timestamp-indexed data such as IoT sensor readings.
15. A __________ database stores its primary data in RAM for extremely low-latency access.

*(Answers: 14 - Time Series, 15 - In-Memory)*

### Interview Questions (Beginner → Senior)
- **Beginner:** What are the main types of databases, and how are they classified?
- **Beginner:** What is a Graph database, and when would you use one instead of a relational database?
- **Intermediate:** Explain the difference between a Distributed and a Federated database.
- **Intermediate:** Why might a company choose an In-Memory database like Redis for a specific feature, while keeping their primary data in PostgreSQL?
- **Senior (SDE-2/3):** You're designing a system that needs (a) strong transactional guarantees for financial records, (b) fast friend-recommendation queries, and (c) extremely low-latency session storage. Propose a polyglot-persistence architecture (referencing Module 8's discussion of this concept) using specific database types from this module for each need, and justify each choice.
- **Senior (SDE-2/3):** A company is debating whether to merge three independently-run divisional databases into one centralized system, versus building a Federated query layer over them. Discuss the trade-offs (cost, risk, autonomy, consistency) of each approach.

### Practical Assignments
- For each of the 15 database types in the Master Comparison Table, write down one real company or product (other than those already mentioned in this module) that you believe uses that type, and briefly justify your guess.
- If you have access to SQLite (often pre-installed on many systems) and Redis, try creating a simple table/key-value pair in each, and reflect on how different the experience feels compared to a full client-server DBMS.
- Classify three apps on your own phone (e.g., a messaging app, a fitness tracker, a banking app) by the type of database you believe each most likely uses for its local/offline storage.

### Practice Links
- Conceptual module — instead of SQL problems, read the official "Appropriate Uses For SQLite" documentation page (referenced back in Module 8) alongside Neo4j's official "Why Graph Databases" page for two contrasting, real vendor perspectives on database type selection.

---

## ⏭️ What's Next

**Module 13: DBMS vs RDBMS** will take a precise, focused look at a distinction commonly confused by beginners — since a Relational Database (12.1.1) is just one specific *type* of database among the many covered in this module, this next module clarifies exactly what makes an RDBMS a specific subset of the broader DBMS category, with a full feature-by-feature comparison table.

---

*End of Module 12. Reply "continue" or "next module" to proceed to Module 13: DBMS vs RDBMS.*
