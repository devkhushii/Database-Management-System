# Part 3 — The Complete SQL Handbook
## Module 2: SQL Environment Setup

> **Goal of this module:** Get PostgreSQL, MySQL, and SQLite installed and running on your machine, understand the GUI and CLI tools available for each, learn how a client actually *connects* to a database server (tying back to Module 1's lifecycle diagram), and get a sample database loaded so Module 3 onward can be 100% hands-on.

---

### 📍 Where This Module Fits

```
Module 1: What SQL is & how it thinks  →  Module 2: Setting up your workshop  →  Module 3: DDL (hands-on begins)
```

This is a **practical, tooling-focused** module. There's less theory here and more "do this, then this." Even so, we'll explain *why* each choice matters — e.g., why PostgreSQL uses a client-server model differently from SQLite's embedded model — because these architectural distinctions get asked about in interviews too.

---

## 2.1 The Three Deployment Models You'll Meet

Before installing anything, it helps to understand that not all databases are architected the same way.

| Model | Description | Examples |
|---|---|---|
| **Client-Server** | A database **server process** runs continuously (as a background service/daemon), listening on a network port. Clients (CLI tools, apps, GUI tools) connect to it over TCP/IP or Unix sockets. | PostgreSQL, MySQL, SQL Server, Oracle |
| **Embedded** | The database engine runs **inside your application's own process** — there's no separate server, no network connection, just a library reading/writing a local file. | SQLite |
| **Cloud-managed** | A client-server database, but the server is fully hosted and managed by a cloud provider (you never install anything yourself). | Amazon RDS, Google Cloud SQL, Supabase, Neon |

```
Client-Server Model                          Embedded Model (SQLite)
─────────────────────                        ────────────────────────
┌────────┐   TCP/IP    ┌──────────┐          ┌─────────────────────┐
│ Client │ ◄─────────► │  Server  │          │   Your Application   │
│(psql,  │  (port 5432 │ Process  │          │  ┌────────────────┐ │
│ app)   │   /3306)    │(postgres/│          │  │ SQLite Library │ │
└────────┘             │  mysqld) │          │  │  (linked-in)   │ │
                        └────┬─────┘          │  └───────┬────────┘ │
                             │                 └──────────┼─────────┘
                        ┌────▼─────┐                      │
                        │  Data on │                 ┌────▼─────┐
                        │   Disk   │                 │ .db file │
                        └──────────┘                 │ on disk  │
                                                       └──────────┘
```

> 🧠 **Real-life analogy:** A **client-server** database is like a restaurant — a kitchen (server) stays open and serves many customers (clients) who walk in and place orders over a counter (network connection). An **embedded** database like SQLite is like cooking at home — there's no "restaurant," just you directly using the ingredients (the file) yourself, inside your own kitchen (your application's process).

This distinction explains *why* installing PostgreSQL/MySQL involves starting a **service**, while "installing" SQLite is often just... having a single file and a library. Keep this model in mind — it will resurface when we discuss connection pooling and concurrency in later parts.

---

## 2.2 Installing PostgreSQL

### What You're Installing

PostgreSQL ships as:
- **`postgres`** — the actual database server process (the "postmaster")
- **`psql`** — the official command-line client
- Supporting utility binaries (`pg_dump`, `pg_restore`, `createdb`, etc.)

### Installation by Platform

**🐧 Linux (Debian/Ubuntu)**
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```
`postgresql-contrib` adds useful extensions (e.g., `pgcrypto`, `uuid-ossp`) that we'll reference later when discussing UUIDs (Module 5) and security.

**🍎 macOS**
```bash
# Using Homebrew (recommended)
brew install postgresql@16
brew services start postgresql@16
```

**🪟 Windows**
- Download the installer from the official PostgreSQL site (via **EDB's** graphical installer) or use the Windows package manager:
```powershell
winget install PostgreSQL.PostgreSQL
```
- The installer sets up the service automatically and prompts you to set a password for the default `postgres` superuser.

### Verifying Installation

```bash
psql --version
# psql (PostgreSQL) 16.x

# Check the service is running
sudo systemctl status postgresql      # Linux
brew services list                    # macOS
```

### First Connection

```bash
sudo -u postgres psql
```

This drops you into an interactive `psql` shell. You'll see a prompt like:

```
postgres=#
```

The `#` (as opposed to `>`) indicates you're connected as a **superuser**.

> ⚠️ **Common mistake:** Forgetting that PostgreSQL, by default on Linux, uses **peer authentication** for local connections — meaning your *OS username* must match a *database username*, or you must explicitly connect as the `postgres` OS user (as shown above). This trips up nearly every first-time Linux installer. We'll revisit authentication methods (`peer`, `md5`, `scram-sha-256`) briefly here and in more depth when we cover DCL later in the handbook.

---

## 2.3 Installing MySQL

### What You're Installing

- **`mysqld`** — the MySQL server daemon
- **`mysql`** — the official CLI client
- Optionally, **MySQL Shell** (`mysqlsh`) — a newer, more powerful CLI with JavaScript/Python scripting support

### Installation by Platform

**🐧 Linux (Debian/Ubuntu)**
```bash
sudo apt update
sudo apt install mysql-server
sudo mysql_secure_installation
```
`mysql_secure_installation` is an interactive script that removes insecure defaults (anonymous users, test databases) and sets a root password — **always run this after a fresh install.**

**🍎 macOS**
```bash
brew install mysql
brew services start mysql
mysql_secure_installation
```

**🪟 Windows**
- Download **MySQL Installer for Windows** from the official MySQL site, or:
```powershell
winget install Oracle.MySQL
```
- The graphical installer lets you choose a setup type (Developer Default, Server Only, etc.) and configure the root password interactively.

### Verifying Installation

```bash
mysql --version
# mysql  Ver 8.0.x for Linux on x86_64

sudo systemctl status mysql     # Linux
```

### First Connection

```bash
mysql -u root -p
```

You'll be prompted for the root password, then dropped into:

```
mysql>
```

> 💡 **Note on MySQL forks:** You may also encounter **MariaDB**, a community-driven fork of MySQL created after Oracle's acquisition of MySQL, designed to remain open-source and largely SQL-compatible. Most syntax and concepts in this handbook apply directly to MariaDB as well, with a few divergences we'll flag when relevant.

---

## 2.4 Installing SQLite

Because SQLite is **embedded** (see Section 2.1), "installing" it is different — there's no service to start.

**🐧 Linux**
```bash
sudo apt install sqlite3
```

**🍎 macOS**
SQLite ships **pre-installed** on macOS. Verify with:
```bash
sqlite3 --version
```

**🪟 Windows**
```powershell
winget install SQLite.SQLite
```
Or download the precompiled binaries directly from sqlite.org.

### Creating Your First SQLite Database

```bash
sqlite3 my_first_database.db
```

This single command **creates the file if it doesn't exist** and opens an interactive shell — no server, no service, no daemon. This is precisely the embedded model from Section 2.1 in action.

```
sqlite>
```

> 🧠 **Why this matters for interviews:** Being asked *"when would you choose SQLite over PostgreSQL?"* is common. The honest answer: SQLite is excellent for **embedded use cases** — mobile apps, browsers, small desktop tools, local caching, testing/CI environments — where you don't need concurrent multi-user write access or network access. It is *not* designed to be a high-concurrency, multi-user production backend for a web-scale application — that's squarely PostgreSQL/MySQL territory.

---

## 2.5 GUI Tools

While the CLI is essential (and often faster for experienced engineers), GUI tools help beginners visualize schemas, browse data, and build queries with autocomplete.

### 🔹 pgAdmin (PostgreSQL)

- The official, most widely used GUI for PostgreSQL.
- Runs as a **web application** (even the "desktop" version is a local web app rendered in a bundled browser shell).
- Features: visual schema browser, query tool with syntax highlighting, `EXPLAIN` plan visualizer (extremely useful once we reach Part 4's optimization content), backup/restore wizards, user/role management UI.

**Installation:** Bundled with the Windows PostgreSQL installer; separate download for macOS/Linux, or `pip install pgadmin4` for a lightweight setup.

### 🔹 MySQL Workbench (MySQL)

- The official GUI for MySQL, from Oracle.
- Features: visual schema/ER diagram designer (draw tables and relationships graphically, then generate the SQL — ties back to Part 2's ER modeling!), query editor, server administration (user management, performance dashboards), data import/export wizards.

**Installation:** Download from the official MySQL site, or:
```bash
sudo apt install mysql-workbench    # Linux
brew install --cask mysql-workbench # macOS
```

### 🔹 DBeaver (Universal — Any Database)

- A **free, open-source, universal database GUI** that supports PostgreSQL, MySQL, SQLite, Oracle, SQL Server, and dozens more through a plugin-driven JDBC architecture.
- Why it's popular in industry: engineers who work across multiple database systems (very common in real jobs) prefer **one consistent tool** instead of learning a different GUI per vendor.
- Features: ER diagram auto-generation from existing schemas, data editor with filtering/sorting, SQL editor with autocomplete, data export to CSV/JSON/Excel.

**Installation:**
```bash
sudo snap install dbeaver-ce        # Linux
brew install --cask dbeaver-community  # macOS
```
Or download the installer for Windows from dbeaver.io.

### Comparison Table

| Tool | Databases Supported | Best For | Type |
|---|---|---|---|
| **pgAdmin** | PostgreSQL only | Deep Postgres-specific administration | Web-based (local) |
| **MySQL Workbench** | MySQL/MariaDB only | Visual ER design + MySQL administration | Native desktop |
| **DBeaver** | Almost everything | Engineers working across multiple DB engines | Native desktop (Java-based) |

> 💡 **Best practice for this handbook:** We recommend installing **DBeaver** as your primary GUI since you'll be working with both PostgreSQL and MySQL throughout this handbook, and consistent tooling reduces cognitive overhead. Use pgAdmin/MySQL Workbench only if you want vendor-specific administrative features later.

---

## 2.6 VS Code Extensions for SQL

If you prefer working from your code editor (common for backend engineers who also write application code alongside SQL), these extensions are industry-standard:

| Extension | Publisher | What It Adds |
|---|---|---|
| **SQLTools** | Matheus Teixeira | Universal connection manager + query runner inside VS Code, with driver plugins for Postgres/MySQL/SQLite |
| **PostgreSQL** | Chris Kolkman / Microsoft | Postgres-specific IntelliSense, syntax highlighting |
| **MySQL** | Jun Han / Weijan Chen | MySQL connection management and query execution |
| **SQLite Viewer** | Florian Klampfer / Antti | Opens `.db`/`.sqlite` files as browsable tables directly in VS Code |
| **Rainbow CSV** | mechatroner | Helps visually inspect CSV exports from query results (color-codes columns) |

**Installation (any):**
```
Ctrl+Shift+X → search extension name → Install
```

> 🧠 **Why mention this at all?** Many backend engineers, especially those working in polyglot codebases, live inside VS Code all day. Being able to run and inspect SQL queries **without context-switching to a separate GUI app** is a genuine productivity habit worth building early.

---

## 2.7 Connecting to a Database — Tying Back to Module 1's Lifecycle

Recall from **Module 1, Section 1.9–1.10** that the SQL processing lifecycle begins with: `Client → Parser → Optimizer → Executor`. The very first step — **establishing a connection** — deserves its own explanation now that we're hands-on.

### Anatomy of a Connection String

Every client-server database connection (whether via CLI, GUI, or application driver) is defined by these core parameters:

```
Host:     Where the server is running (e.g., localhost, 127.0.0.1, or a remote IP/domain)
Port:     Which network port the server listens on
             PostgreSQL default → 5432
             MySQL default      → 3306
User:     The database username authenticating the connection
Password: Credential for that user
Database: The specific database (schema/catalog) to connect to on that server
```

**PostgreSQL connection string (URI format):**
```
postgresql://myuser:mypassword@localhost:5432/mydatabase
```

**MySQL connection string:**
```
mysql://myuser:mypassword@localhost:3306/mydatabase
```

### What Happens When You Connect (Step by Step)

```
1. Client resolves host + port
2. TCP handshake established with the server process (postgres/mysqld)
3. Client sends authentication credentials
4. Server verifies credentials against its internal user/auth tables
5. On success: server allocates a "connection" (in Postgres, typically a
   dedicated backend process per connection; in MySQL, a dedicated thread)
6. Client is now free to send queries — and Module 1's lifecycle
   (Parser → Optimizer → Executor) kicks in for each query sent
```

> ⚠️ **Performance note (previewed now, expanded later):** In PostgreSQL, each connection spawns a **separate OS process** — this is relatively expensive to create, which is exactly why production systems use **connection poolers** like **PgBouncer**. MySQL instead uses a **lightweight thread-per-connection** model, which is cheaper to spin up but still benefits from pooling at scale. This connects directly to real system design interview discussions about database connection management — file this away for when we discuss scaling in later parts.

### Practicing Connections Across Tools

| Tool | Command/Action |
|---|---|
| `psql` CLI | `psql -h localhost -p 5432 -U myuser -d mydatabase` |
| `mysql` CLI | `mysql -h localhost -P 3306 -u myuser -p mydatabase` |
| `sqlite3` CLI | `sqlite3 path/to/file.db` (no host/port — it's a local file) |
| pgAdmin | Add Server → fill Host/Port/Username/Password in GUI dialog |
| MySQL Workbench | New Connection → fill Host/Port/Username → Test Connection |
| DBeaver | New Database Connection → choose driver → fill credentials |

---

## 2.8 Sample Databases

Learning SQL on *toy, tiny, made-up tables* only gets you so far. Throughout this handbook (especially from Module 8 onward, and heavily in Module 19's industry case studies), we'll use realistic **sample databases** that mirror what you'd encounter on the job or in interview take-home tests.

### Recommended Sample Databases

| Sample DB | Domain | Best For | Source |
|---|---|---|---|
| **Sakila** | DVD rental store (films, actors, rentals, payments) | Joins, aggregation practice — official MySQL sample DB | [dev.mysql.com/doc/sakila](https://dev.mysql.com/doc/sakila/en/) |
| **Pagila** | Same schema as Sakila, ported to PostgreSQL | Practicing identical queries across both engines side-by-side | GitHub: `devrimgunduz/pagila` |
| **Northwind** | Classic trading company (orders, products, suppliers, customers) | Beginner-friendly relational structure; extremely common in tutorials and interview prep | Multiple community ports available for Postgres/MySQL/SQLite |
| **Chinook** | Digital music store (tracks, albums, artists, invoices) | Great for practicing multi-level joins and aggregate reporting | GitHub: `lerocha/chinook-database` |
| **LeetCode Database Schemas** | Various (per-problem) | Directly interview-relevant practice | Built into LeetCode's Database problem set |

### Loading Sakila into MySQL (Example Workflow)

```bash
# 1. Download and extract the Sakila schema + data files from the official site
# 2. Load the schema
mysql -u root -p < sakila-schema.sql
# 3. Load the data
mysql -u root -p < sakila-data.sql
# 4. Verify
mysql -u root -p -e "USE sakila; SHOW TABLES;"
```

### Loading Pagila into PostgreSQL (Example Workflow)

```bash
# 1. Clone the repository
git clone https://github.com/devrimgunduz/pagila.git
cd pagila
# 2. Create the database
createdb pagila
# 3. Load schema, then data
psql -d pagila -f pagila-schema.sql
psql -d pagila -f pagila-data.sql
```

> 💡 **Why this handbook will lean on Sakila/Pagila:** Because the **same logical schema exists for both MySQL and PostgreSQL**, it's the perfect way to demonstrate real **PostgreSQL vs MySQL syntax differences** side-by-side using identical data — exactly the comparative approach this handbook promises throughout every module.

---

## 2.9 Setting Up Your Practice Environment — Recommended Path

For readers following this handbook end-to-end, here's the setup we recommend before proceeding to Module 3:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Install PostgreSQL + MySQL locally (Section 2.2 / 2.3)    │
│ 2. Install SQLite (Section 2.4) — zero-config backup option  │
│ 3. Install DBeaver as your universal GUI (Section 2.5)       │
│ 4. Optionally add VS Code SQL extensions (Section 2.6)       │
│ 5. Load Pagila (Postgres) and Sakila (MySQL) sample DBs      │
│ 6. Confirm you can connect to both via CLI AND DBeaver       │
└─────────────────────────────────────────────────────────────┘
```

> 🧠 **Alternative: no-install option.** If local installation is inconvenient (locked-down work laptop, low disk space), you can practice using free browser-based sandboxes: **SQLBolt** (interactive, beginner-friendly), **DB-Fiddle** or **SQLFiddle** (multi-engine online scratchpads), or **LeetCode/HackerRank's** built-in SQL runners. We'll link these again in the exercises below.

---

## 2.10 PostgreSQL vs MySQL — Setup & Tooling Differences Summary

| Aspect | PostgreSQL | MySQL |
|---|---|---|
| Default port | 5432 | 3306 |
| Default superuser | `postgres` | `root` |
| CLI tool | `psql` | `mysql` |
| Official GUI | pgAdmin | MySQL Workbench |
| Connection model | Process-per-connection | Thread-per-connection |
| Default local auth (Linux) | Often `peer` (OS user match) | Often password-based |
| Post-install hardening | Less required by default | `mysql_secure_installation` strongly recommended |
| License | PostgreSQL License (permissive, similar to MIT/BSD) | GPL (community) / Commercial (Enterprise), owned by Oracle |

---

## 2.11 Best Practices

- Always run `mysql_secure_installation` immediately after installing MySQL — never leave default/blank passwords, even on a local dev machine (builds good security habits early).
- Create a **dedicated, non-superuser database user** for daily practice instead of always using `postgres`/`root` — this mirrors real-world least-privilege practice and will make more sense once we cover DCL (`GRANT`/`REVOKE`) later.
- Keep at least one **sample database per engine** (e.g., Pagila for Postgres, Sakila for MySQL) permanently available for cross-engine comparison exercises used throughout this handbook.
- Learn the **CLI first**, even if you'll use a GUI daily — CLI fluency is often directly tested in interviews and is indispensable when working on remote servers with no GUI access.

## 2.12 Common Mistakes

| Mistake | Consequence |
|---|---|
| Forgetting to start the database service after installation | "Connection refused" errors that confuse beginners into thinking their syntax is wrong |
| Using `root`/`postgres` superuser for everyday practice | Bad habit that hides permission-related bugs you'll hit in real production environments |
| Not verifying installation version | Following tutorials/features that don't exist in an older installed version |
| Mixing up default ports (5432 vs 3306) in connection strings | Confusing "connection refused" errors when accidentally pointing a MySQL client at Postgres's port or vice versa |

## 2.13 Edge Cases

- On some Linux distributions, PostgreSQL's default **peer authentication** means `psql` with no extra flags only works if your **shell username matches a database role** — a frequent point of confusion covered in Section 2.2.
- Docker-based setups (increasingly common in real teams) sidestep most local-install issues entirely — worth mentioning that many engineers today run `docker run postgres` / `docker run mysql` instead of native installs, especially for short-lived practice environments. (We won't dive deep into Docker here since it's outside SQL itself, but it's a valid alternative path.)
- SQLite has **no user/password authentication at all** — because it's an embedded file, "access control" is really just OS-level file permissions, a genuinely different security model than client-server databases, and a legitimate interview talking point about SQLite's limitations.

## 2.14 Performance Implications

Environment choices here directly foreshadow performance topics in Part 4: process-per-connection (Postgres) vs. thread-per-connection (MySQL) architectures have different memory/CPU tradeoffs at high concurrency, which is exactly why connection pooling tools (PgBouncer for Postgres, built-in thread pooling for MySQL) exist in production systems — we'll revisit this with real numbers once we cover transactions and concurrency control.

## 2.15 Industry Usage

Nearly every backend engineering job will expect you to comfortably install, configure, and connect to at least one of PostgreSQL or MySQL locally for development, in addition to interacting with managed cloud instances (RDS, Cloud SQL) in production — the exact same connection-string mental model from Section 2.7 applies identically to cloud-hosted databases, just with a different host address.

## 2.16 Interview Importance

Environment setup itself is **rarely directly interviewed**, but the *concepts* behind it are: expect questions like *"explain the client-server model,"* *"what's the difference between an embedded and a server-based database,"* or *"why would you use SQLite over PostgreSQL for X use case?"* — all concepts this module equipped you to answer confidently.

---

## 2.17 Chapter Summary

- Databases follow either a **client-server model** (PostgreSQL, MySQL) or an **embedded model** (SQLite) — this architectural difference explains almost everything about how you install and connect to them.
- PostgreSQL and MySQL both require installing a **server process** plus a **CLI client**; SQLite requires neither — it's just a library and a file.
- **GUI tools** (pgAdmin, MySQL Workbench, DBeaver) and **VS Code extensions** provide visual alternatives to the CLI, with DBeaver being the most practical choice for working across multiple engines, as this handbook will.
- Connecting to a database always involves **host, port, user, password, and database name** — and this connection step is literally "Stage 0" before Module 1's Parser → Optimizer → Executor lifecycle begins.
- **Sample databases** (Sakila, Pagila, Northwind, Chinook) give you realistic schemas to practice on — this handbook will lean on Sakila/Pagila specifically because they let us compare PostgreSQL and MySQL side-by-side on identical data.

---

## 2.18 Revision Notes / Cheat Sheet

```
PostgreSQL:  server=postgres, client=psql, port=5432, gui=pgAdmin
MySQL:       server=mysqld,   client=mysql, port=3306, gui=MySQL Workbench
SQLite:      embedded, no server, client=sqlite3, no port/auth

Connection string shape:
  <engine>://<user>:<password>@<host>:<port>/<database>

Sample DBs to install: Sakila (MySQL) + Pagila (PostgreSQL)
Universal GUI recommendation: DBeaver
```

---

## 2.19 Multiple Choice Questions (MCQs)

1. Which of these is an **embedded** database (no separate server process)?
   a) PostgreSQL b) MySQL c) **SQLite** d) Oracle

2. What is PostgreSQL's default network port?
   a) 3306 b) **5432** c) 1433 d) 8080

3. Which command hardens a fresh MySQL installation by removing insecure defaults?
   a) `mysql --harden` b) **`mysql_secure_installation`** c) `mysql --secure` d) `SECURE DATABASE;`

4. Which official GUI tool is specific to PostgreSQL only?
   a) MySQL Workbench b) DBeaver c) **pgAdmin** d) SQLite Viewer

5. In PostgreSQL, each new client connection typically results in:
   a) A shared thread pool entry b) **A dedicated OS process** c) No resource allocation d) A cached HTTP session

---

## 2.20 Exercises

### 🟢 Beginner
1. Install PostgreSQL and MySQL on your machine, and confirm both services are running using their respective status/verification commands.
2. Connect to each database using its CLI tool (`psql`, `mysql`) and run a simple command to list existing databases (`\l` in psql, `SHOW DATABASES;` in mysql).
3. Create an SQLite database file called `practice.db` and open it with `sqlite3`.

### 🟡 Intermediate
4. Install DBeaver and successfully create connections to your local PostgreSQL, MySQL, and SQLite databases inside a single DBeaver workspace.
5. Download and load the **Pagila** sample database into your local PostgreSQL instance, and the **Sakila** sample database into your local MySQL instance.
6. Write out, from memory, the five components of a database connection string (host, port, user, password, database) and explain what each one means.

### 🔴 Advanced / Scenario-Based
7. Explain why a **connection pooler** (like PgBouncer) is more critical for PostgreSQL's architecture than it might be for an embedded database like SQLite. Reference the process-per-connection vs. embedded model distinction from this module.
8. A teammate on a locked-down corporate laptop cannot install PostgreSQL locally. Propose two alternative ways they could still practice SQL hands-on, based on the options mentioned in this module.

### 🧩 Debugging Exercise
9. You run `psql -U myuser -d mydatabase` and get `psql: error: connection to server... failed: FATAL: Peer authentication failed`. Based on Section 2.2 and 2.7, explain what's likely going wrong and how you'd fix it.

### 🏗️ Mini Project
10. Using DBeaver (or your GUI of choice), connect to both your Pagila (PostgreSQL) and Sakila (MySQL) databases, and produce a short written comparison of their table names/structure just from browsing the schema visually — no querying yet. This will prime you directly for Module 3's DDL content.

### 🔗 Practice Links
- [SQLBolt — no-install interactive practice](https://sqlbolt.com/)
- [DB-Fiddle — online multi-engine SQL sandbox](https://www.db-fiddle.com/)
- [Sakila Sample Database (official MySQL docs)](https://dev.mysql.com/doc/sakila/en/)
- [Pagila (PostgreSQL port of Sakila) on GitHub](https://github.com/devrimgunduz/pagila)
- [Chinook Sample Database on GitHub](https://github.com/lerocha/chinook-database)

---

## ✅ Module 2 Complete

You now have a fully functional SQL practice environment: PostgreSQL, MySQL, and SQLite installed, GUI/CLI tooling set up, and realistic sample databases loaded — everything Module 3 onward needs to become fully hands-on.

**Next up: Module 3 — Database Operations (DDL)**, where you'll write your first real SQL: `CREATE DATABASE`, `DROP DATABASE`, `ALTER DATABASE`, and explore the `information_schema`.

---

*Reply "continue" or "next module" when you're ready for Module 3.*
