
# Database Management System (DBMS) Handbook

A structured, module-wise handbook covering database fundamentals, SQL, relational theory, database design, transactions, indexing, query processing, recovery, internals, distributed databases, and production engineering.

> **Navigation:** Select a part below, then open any module. Each module link points directly to its Markdown file. Topic names are taken from the headings in the corresponding module.

## Contents

- [part1 — DBMS Fundamentals](#dbms-fundamentals)
- [part2 — Data Models & ER Modeling](#data-models-er-modeling)
- [part3 — SQL Fundamentals & Querying](#sql-fundamentals-querying)
- [part4 — Relational Algebra, Relational Calculus & Advanced SQL](#relational-algebra-relational-calculus-advanced-sql)
- [part5 — Database Design & Normalization](#database-design-normalization)
- [part6 — Transactions & Concurrency Control](#transactions-concurrency-control)
- [part7 — Indexing, File Organization & Storage Structures](#indexing-file-organization-storage-structures)
- [part8 — Query Processing & Query Optimization](#query-processing-query-optimization)
- [part9 — Database Recovery, Logging & Durability](#database-recovery-logging-durability)
- [part10 — Database Internals & Storage Engine Architecture](#database-internals-storage-engine-architecture)
- [part11 — Distributed Databases, Replication, Sharding & Modern Architectures](#distributed-databases-replication-sharding-modern-architectures)
- [part12 — Production Database Engineering & DBA Mastery](#production-database-engineering-dba-mastery)
- [part13 (optional) — Research & Advanced Engineering (Optional)](#research-advanced-engineering-optional)

---

## part1 — DBMS Fundamentals

Data and information, data-storage evolution, file systems, the need for DBMS, DBMS architecture and components, database users and types, ACID, CAP, and real-world applications.

| Module | Topics covered |
|---|---|
| [Module 1: Introduction to Data](part1/module1.md) | 1.0 Why Start With "Data" At All?; 1.1 What is Data?; 1.2 Information vs Data; 1.3 Metadata; 1.4 Structured, Semi-Structured, and Unstructured Data |
| [Module 2: Evolution of Data Storage](part1/module2.md) | 2.0 The Big Picture: A Timeline; 2.1 Manual Records / Paper-Based Systems; 2.2 Flat Files; 2.3 Sequential Files; 2.4 Indexed Files |
| [Module 3: File System](part1/module3.md) | 3.1 What is a File System?; 3.2 File Organization; 3.3 Common File Formats in File-Based Processing; 3.3.1 Flat Files (recap); 3.3.2 CSV (Comma-Separated Values) |
| [Module 4: Need for DBMS](part1/module4.md) | 4.0 The Core Idea in One Sentence; 4.1 Solving Data Redundancy; 4.2 Solving Data Inconsistency; 4.3 Solving Data Isolation; 4.4 Solving Integrity Problems |
| [Module 5: What is DBMS?](part1/module5.md) | 5.1 Definition of DBMS; 5.2 Goals of a DBMS; 5.3 Core Features of a DBMS; 5.4 Components of a DBMS (High-Level Preview); 5.5 Data Independence — A Closer Look |
| [Module 6: Characteristics of DBMS](part1/module6.md) | 6.1 Data Independence (Expanded); 6.2 Controlled Redundancy; 6.3 Data Integrity; 6.4 Consistency; 6.5 Security |
| [Module 7: Advantages of DBMS](part1/module7.md) | 7.1 Reduced Data Redundancy; 7.2 Improved Data Consistency; 7.3 Better Data Integrity; 7.4 Enhanced Data Security; 7.5 Efficient Data Access (Powerful Querying) |
| [Module 8: Limitations of DBMS](part1/module8.md) | 8.1 Cost; 8.2 Complexity; 8.3 Resource Usage (Performance Overhead); 8.4 Maintenance; 8.5 Migration Difficulty |
| [Module 9: Database Architecture](part1/module9.md) | 9.1 The Three-Schema Architecture (ANSI/SPARC Architecture); 9.2 One-Tier Architecture; 9.3 Two-Tier Architecture (Client-Server); 9.4 Three-Tier Architecture; 9.5 N-Tier Architecture |
| [Module 10: DBMS Components](part1/module10.md) | 10.0 The Full Picture (Recap + Expansion of Module 5.4); 10.1 Storage Manager; 10.1.1 Buffer Manager; 10.2 Query Processor; 10.2.1 Parser |
| [Module 11: Database Users](part1/module11.md) | 11.0 The Big Picture; 11.1 Database Administrator (DBA); 11.2 Database Designers; 11.3 Application Developers (Programmers); 11.4 End Users |
| [Module 12: Types of Databases](part1/module12.md) | 12.0 Three Axes of Classification; 12.1 Axis 1 — By Data Model; 12.1.1 Relational Databases *(full detail: Module 2.7)*; 12.1.2 Hierarchical Databases *(full detail: Module 2.5)*; 12.1.3 Network Databases *(full detail: Module 2.6)* |
| [Module 13: DBMS vs RDBMS](part1/module13.md) | 13.1 The Core Distinction; 13.2 Feature-by-Feature Comparison; 13.3 Codd's Rules — The Formal Benchmark for "True" RDBMS; 13.4 Why RDBMS Became the Dominant DBMS Type |
| [Module 14: Popular Database Systems](part1/module14.md) | 14.1 PostgreSQL; 14.2 MySQL; 14.3 MariaDB; 14.4 Oracle Database; 14.5 Microsoft SQL Server |
| [Module 15: CAP Theorem (Introduction)](part1/module15.md) | 15.1 Background — Why This Theorem Exists; 15.2 The Three Properties; 15.2.1 Consistency (C); 15.2.2 Availability (A); 15.2.3 Partition Tolerance (P) |
| [Module 16: ACID Properties (Introduction)](part1/module16.md) | 16.0 What is a Transaction? (Necessary Foundation); 16.1 Atomicity ("All or Nothing"); 16.2 Consistency ("Valid State to Valid State"); 16.3 Isolation ("Concurrent Transactions Don't Interfere"); 16.4 Durability ("Once Committed, Permanent") |
| [Module 17: Database Development Lifecycle](part1/module17.md) | 17.0 The Eight Phases at a Glance; 17.1 Requirement Analysis; 17.2 ER (Entity-Relationship) Design; 17.3 Logical Design; 17.4 Physical Design |
| [Module 18: Industry Applications](part1/module18.md) | 18.1 Banking; 18.2 E-commerce; 18.3 Healthcare; 18.4 Education; 18.5 Social Media |
| [Module 19: Common Misconceptions](part1/module19.md) | 19.1 Database vs. DBMS; 19.2 SQL vs. Database; 19.3 DBMS vs. Spreadsheet; 19.4 DBMS vs. Data Warehouse; 19.5 Database vs. Server |
| [Module 20: Preview of Future Parts](part1/module20.md) | 20.0 What You've Actually Built (A Foundation, Not Trivia); 20.1 Self-Assessment — Are You Ready for Part 2?; 20.2 What's Coming in Part 2 and Beyond; 20.2.1 SQL Fundamentals (Hands-On); 20.2.2 ER Modeling (Deep Dive) |

[Back to contents](#contents)

## part2 — Data Models & ER Modeling

Data models, relational concepts and terminology, keys, integrity constraints, ER/EER modeling, attributes, entities, relationships, cardinality, participation, schema design, and ER-to-relational mapping.

| Module | Topics covered |
|---|---|
| [Module 1: Data Models](part2/DBMS-Handbook-Part2-Module1-DataModels.md) | 1. What is a Data Model?; 2. Why Data Models Exist; 3. Levels of Abstraction; 4. Types of Data Models; 5. Object-Based Data Models |
| [Module 2: The Relational Model](part2/DBMS-Handbook-Part2-Module2-RelationalModel.md) | 1. History — E. F. Codd; 2. Why the Relational Model?; 3. Mathematical Foundation; 4. Set Theory Basics; 5. Predicate Logic Overview |
| [Module 3: Relational Model Terminology — Deep Dive with Multiple Examples](part2/DBMS-Handbook-Part2-Module3-Terminology.md) | 1. Why This Module Exists (Read First); 2. The Ten Core Terms — Quick Recap Table; 3. Worked Example 1: Banking System; 4. Worked Example 2: E-Commerce System; 5. Worked Example 3: Healthcare System |
| [Module 4: Keys (Complete)](part2/DBMS-Handbook-Part2-Module4-Keys.md) | 1. Why Keys Exist — The Big Picture; 2. The Key Hierarchy (Overview Diagram); 3. Super Key; 4. Candidate Key; 5. Primary Key |
| [Module 5: Integrity Constraints](part2/DBMS-Handbook-Part2-Module5-IntegrityConstraints.md) | 1. What Are Integrity Constraints, and Why Do They Exist?; 2. Classification of Constraints; 3. Domain Constraint; 4. Key Constraint; 5. Entity Integrity |
| [Module 6: The Entity Relationship (ER) Model](part2/DBMS-Handbook-Part2-Module6-ERModel.md) | 1. Why the ER Model?; 2. History of the ER Model; 3. Purpose of ER Modeling; 4. Conceptual Modeling — Where ER Fits; 5. ER Components — The Building Blocks |
| [Module 7: Attributes](part2/DBMS-Handbook-Part2-Module7-Attributes.md) | 1. What Is an Attribute (ER Perspective)?; 2. The Full Attribute Classification Tree; 3. Simple (Atomic) Attribute; 4. Composite Attribute; 5. Single-Valued Attribute |
| [Module 8: Entity Sets](part2/DBMS-Handbook-Part2-Module8-EntitySets.md) | 1. Recap: Strong vs. Weak Entities; 2. Owner Entity (Formal Definition); 3. Identifying Relationship (Formal Definition); 4. Partial Key — Full Deep Dive; 5. The Complete Weak Entity Identification Formula |
| [Module 9: Relationships](part2/DBMS-Handbook-Part2-Module9-Relationships.md) | 1. What Is a Relationship (Formal Definition)?; 2. Relationship Set; 3. Degree of Relationship; 4. Unary (Degree 1) Relationships; 5. Recursive Relationships — Full Deep Dive |
| [Module 10: Cardinality Constraints](part2/DBMS-Handbook-Part2-Module10-Cardinality.md) | 1. What Is a Cardinality Constraint?; 2. One-to-One (1:1); 3. One-to-Many (1:N); 4. Many-to-One (N:1); 5. Many-to-Many (M:N) |
| [Module 11: Participation Constraints](part2/DBMS-Handbook-Part2-Module11-Participation.md) | 1. What Is a Participation Constraint?; 2. Total Participation; 3. Partial Participation; 4. Mandatory vs. Optional Participation — Same Idea, Different Words; 5. Notation for Participation |
| [Module 12: Enhanced ER (EER) Model](part2/DBMS-Handbook-Part2-Module12-EER.md) | 1. Why the EER Model Exists; 2. Generalization; 3. Specialization; 4. Generalization vs. Specialization — Same Diagram, Opposite Direction of Thought; 5. ISA Hierarchy and Inheritance |
| [Module 13: Converting ER to Relational Model](part2/DBMS-Handbook-Part2-Module13-ERtoRelational.md) | 1. Why This Conversion Process Matters; 2. Rule 1: Strong Entity Conversion; 3. Rule 2: Weak Entity Conversion; 4. Rule 3: Relationship Mapping (1:1, 1:N, M:N); 5. Rule 4: Multivalued Attribute Mapping |
| [Module 14: Schema Design Basics](part2/DBMS-Handbook-Part2-Module14-SchemaDesign.md) | 1. Why Schema Design "Basics" Still Matter After Mastering ER Mapping; 2. Naming Conventions; 3. Table Naming; 4. Column Naming; 5. Data Consistency |
| [Module 15: Data Dictionary](part2/DBMS-Handbook-Part2-Module15-DataDictionary.md) | 1. What Is Metadata?; 2. The System Catalog; 3. Catalog Tables; 4. INFORMATION_SCHEMA — The Standardized View; 5. Why the Data Dictionary Is Important |
| [Module 16: Common Modeling Mistakes](part2/DBMS-Handbook-Part2-Module16-CommonMistakes.md) | 1. Why a Dedicated "Mistakes" Module?; 2. Wrong Primary Keys; 3. Duplicate Entities; 4. Missing Relationships; 5. Poor Naming |
| [Module 17: Industry Case Studies](part2/DBMS-Handbook-Part2-Module17-CaseStudies.md) | 1. Case Study 1: Banking System; 2. Case Study 2: Hospital Management System; 3. Case Study 3: E-Commerce System; 4. Case Study 4: University Management System; 5. Case Study 5: Railway Reservation System |
| [Module 18: Interview Perspective](part2/DBMS-Handbook-Part2-Module18-InterviewPerspective.md) | 1. The Four Question Formats; 2. Frequently Asked Conceptual Questions; 3. Scenario-Based Questions; 4. Diagram-Based Questions; 5. Full Schema Design Questions |

[Back to contents](#contents)

## part3 — SQL Fundamentals & Querying

SQL introduction and setup, database/table operations, data types, constraints, DML, SELECT queries, operators, functions, aggregation, joins, subqueries, views, execution order, best practices, and interview practice.

| Module | Topics covered |
|---|---|
| [Module 1: Introduction to SQL](part3/module1.md) | 1.1 What is SQL? |
| [Module 2: SQL Environment Setup](part3/module2.md) | 2.1 The Three Deployment Models You'll Meet; 2.2 Installing PostgreSQL |
| [Module 3: Database Operations (DDL)](part3/module3.md) | 3.1 What is DDL, Really?; 3.2 CREATE DATABASE; 3.3 DROP DATABASE; 3.4 ALTER DATABASE; 3.5 USE DATABASE (MySQL) — and Its Absence in PostgreSQL |
| [Module 4: Table Operations](part3/module4.md) | 4.1 What is a Table?; 4.2 CREATE TABLE; 4.3 DROP TABLE; 4.4 ALTER TABLE; 4.5 RENAME TABLE |
| [Module 5: SQL Data Types](part3/module5.md) | 5.1 Why Data Types Matter So Much; 5.2 Numeric Types; 5.3 Character Types; 5.4 Boolean Type; 5.5 Date & Time Types |
| [Module 6: Constraints](part3/module6.md) | 6.1 What Are Constraints and Why Do They Exist?; 6.2 PRIMARY KEY; 6.3 FOREIGN KEY; 6.4 Referential Actions: CASCADE, RESTRICT, SET NULL, SET DEFAULT, NO ACTION; 6.5 UNIQUE |
| [Module 7: DML (Data Manipulation)](part3/module7.md) | 7.1 What is DML, Revisited; 7.2 INSERT — Single Row; 7.3 INSERT — Multiple Rows; 7.4 INSERT ... SELECT; 7.5 UPSERT (INSERT ... ON CONFLICT / ON DUPLICATE KEY UPDATE) |
| [Module 8: Basic Queries](part3/module8.md) | 8.1 The SELECT Statement — Overview; 8.2 SELECT and Column Projection; 8.3 FROM; 8.4 DISTINCT; 8.5 WHERE |
| [Module 9: SQL Operators](part3/module9.md) | 9.1 Arithmetic Operators; 9.2 Comparison Operators; 9.3 Logical Operators; 9.4 BETWEEN; 9.5 IN and NOT IN |
| [Module 10: SQL Functions](part3/module10.md) | 10.1 What Is a SQL Function?; 10.2 Numeric Functions; 10.3 String Functions; 10.4 Date Functions; 10.5 Time Functions |
| [Module 11: Aggregate Functions](part3/module11.md) | 11.1 What Is an Aggregate Function?; 11.2 COUNT, SUM, AVG, MIN, MAX; 11.3 STDDEV and VARIANCE; 11.4 GROUP BY; 11.5 HAVING |
| [Module 12: Joins (Complete)](part3/module12.md) | 12.1 Why Joins Exist; 12.2 INNER JOIN; 12.3 LEFT JOIN (LEFT OUTER JOIN); 12.4 RIGHT JOIN (RIGHT OUTER JOIN); 12.5 FULL OUTER JOIN |
| [Module 13: Subqueries](part3/module13.md) | 13.1 What Is a Subquery?; 13.2 Scalar Subqueries; 13.3 Row Subqueries; 13.4 Table Subqueries (Derived Tables); 13.5 Correlated Subqueries |
| [Module 14: Views](part3/module14.md) | 14.1 What Is a View?; 14.2 Internal Working; 14.3 Basic View Operations; 14.4 Views for Security; 14.5 Materialized Views (Introduction) |
| [Module 15: SQL Execution Order](part3/module15.md) | 15.1 The Big Reveal — Written Order vs. Logical Execution Order; 15.2 Step-by-Step Walkthrough; 15.3 Every Earlier Module's Rule, Now Explained by One Underlying Principle; 15.4 Where WINDOW FUNCTIONS Fit (Preview of Part 4); 15.5 A Second Worked Example — Tracing a Query with DISTINCT |
| [Module 16: SQL Best Practices](part3/module16.md) | 16.1 Why a Dedicated Best-Practices Module?; 16.2 Naming Conventions; 16.3 Formatting and Readability; 16.4 NULL Handling Discipline; 16.5 Query-Writing Discipline |
| [Module 17: Common SQL Mistakes](part3/module17.md) | 17.1 NULL Mistakes; 17.2 JOIN Mistakes; 17.3 Aggregate Mistakes; 17.4 Duplicate Rows; 17.5 Cartesian Products (Revisited) |
| [Module 18: SQL Interview Questions](part3/module18.md) | 18.1 Beginner-Level Conceptual Questions; 18.2 Intermediate-Level Conceptual Questions; 18.3 Advanced-Level Conceptual Questions (Senior SDE-2/SDE-3); 18.4 Output Prediction Questions; 18.5 Query-Writing Challenges |
| [Module 19: Industry Case Studies](part3/module19.md) | 19.1 Case Study: Banking System; 19.2 Case Study: E-commerce System; 19.3 Case Study: Student Management System; 19.4 Case Study: Hospital Management System; 19.5 Case Study: Inventory Management System |
| [Module 20: Preview of Part 4](part3/module20.md) | 20.1 How Far You've Come; 20.2 Relational Algebra — The Theoretical Foundation Beneath SQL; 20.3 Relational Calculus — The Other Side of the Same Coin; 20.4 Preview: Window Functions; 20.5 Preview: CTEs (Common Table Expressions) |

[Back to contents](#contents)

## part4 — Relational Algebra, Relational Calculus & Advanced SQL

Relational algebra and calculus, relational operators, joins, SQL translation, advanced SQL features, and query-optimization foundations.

| Module | Topics covered |
|---|---|
| [MODULE 1: RELATIONAL ALGEBRA](part4/module1.md) | 1.0 Chapter Roadmap; 1.1 What is Relational Algebra?; 1.2 Why Does Relational Algebra Exist?; 1.3 Mathematical Foundation; 1.4 Unary vs Binary Operations |
| [MODULE 2: RELATIONAL CALCULUS](part4/module2.md) | 2.0 Chapter Roadmap; 2.1 What is Relational Calculus?; 2.2 Why Does Relational Calculus Exist?; 2.3 Predicate Logic Primer (Prerequisite); 2.4 Tuple Relational Calculus (TRC) |
| [MODULE 3: ADVANCED SQL QUERIES](part4/module3.md) | 3.0 Chapter Roadmap; 3.1 Nested Queries (Subqueries) — Overview; 3.2 Non-Correlated vs Correlated Subqueries; 3.3 EXISTS / NOT EXISTS; 3.4 ANY / SOME / ALL |
| [MODULE 4: COMMON TABLE EXPRESSIONS (CTEs)](part4/module4.md) | 4.0 Chapter Roadmap; 4.1 What is a CTE? The WITH Clause; 4.2 Why CTEs Exist; 4.3 Multiple CTEs (Chaining & Referencing); 4.4 Recursive CTEs — Core Mechanics |
| [MODULE 5: WINDOW FUNCTIONS](part4/module5.md) | 5.0 Chapter Roadmap; 5.1 What is a Window Function? The OVER Clause; 5.2 Why Window Functions Exist; 5.3 PARTITION BY and ORDER BY within OVER; 5.4 Ranking Functions |
| [MODULE 6: ADVANCED AGGREGATION](part4/module6.md) | 6.0 Chapter Roadmap; 6.1 Recap: How Plain GROUP BY Works; 6.2 GROUPING SETS; 6.3 ROLLUP; 6.4 CUBE |
| [MODULE 7: SET OPERATIONS](part4/module7.md) | 7.0 Chapter Roadmap; 7.1 Recap & Syntax Rules; 7.2 UNION; 7.3 UNION ALL; 7.4 INTERSECT |
| [MODULE 8: ADVANCED DATABASE OBJECTS](part4/module8.md) | 8.0 Chapter Roadmap; 8.1 Views; 8.2 Materialized Views; 8.3 Views vs Materialized Views — Full Comparison; 8.4 Sequences |
| [MODULE 9: STORED PROCEDURES & FUNCTIONS](part4/module9.md) | 9.0 Chapter Roadmap; 9.1 Procedures vs Functions — The Core Distinction; 9.2 Stored Procedures; 9.3 User-Defined Functions (UDFs); 9.4 Parameters (IN, OUT, INOUT) |
| [MODULE 10: TRIGGERS](part4/module10.md) | 10.0 Chapter Roadmap; 10.1 What is a Trigger?; 10.2 Why Triggers Exist; 10.3 BEFORE Triggers; 10.4 AFTER Triggers |
| [MODULE 11: SQL EXECUTION INTERNALS (INTRODUCTION)](part4/module11.md) | 11.0 Chapter Roadmap; 11.1 The Big Picture: From SQL Text to Result Set; 11.2 Stage 1: SQL Parser; 11.3 Stage 2: Query Rewrite; 11.4 Stage 3: Optimizer |
| [MODULE 12: ADVANCED SQL BEST PRACTICES](part4/module12.md) | 12.0 Chapter Roadmap; 12.1 Writing Readable SQL; 12.2 Query Decomposition; 12.3 Reusing CTEs Effectively; 12.4 Avoiding Unnecessary Subqueries |
| [MODULE 13: COMMON SQL PITFALLS](part4/module13.md) | 13.0 Chapter Roadmap; 13.1 Duplicate Rows; 13.2 Incorrect Joins; 13.3 Wrong Aggregations; 13.4 Recursive Query Mistakes |
| [MODULE 14: INDUSTRY CASE STUDIES](part4/module14.md) | 14.0 Chapter Roadmap; 14.1 Banking; 14.2 HR Analytics; 14.3 Sales Analytics; 14.4 Inventory |
| [MODULE 15: INTERVIEW MASTERY](part4/module15.md) | 15.0 Chapter Roadmap; 15.1 How Advanced SQL Interviews Are Actually Structured; 15.2 Frequently Asked Advanced SQL Questions (Conceptual Bank); 15.3 Window Function Question Bank; 15.4 CTE Question Bank |

[Back to contents](#contents)

## part5 — Database Design & Normalization

Database-design principles, functional dependencies, normalization, schema design, decomposition, and design case studies.

| Module | Topics covered |
|---|---|
| [Module 1: Database Design Fundamentals](part5/module1.md) | 1.1 What Is Database Design?; 1.2 Why Database Design Matters; 1.3 Design Goals; 1.4 The Database Design Process — Overview; 1.5 Conceptual Design |
| [Module 2: Functional Dependencies (FD)](part5/module2.md) | 2.1 What Is a Functional Dependency?; 2.2 Why Does Functional Dependency Exist as a Concept?; 2.3 How Functional Dependencies Are Identified in Practice; 2.4 Trivial Functional Dependency; 2.5 Non-Trivial Functional Dependency |
| [Module 3: Attribute Closure](part5/module3.md) | 3.1 What Is Attribute Closure?; 3.2 What Problem Does Closure Solve?; 3.3 The Attribute Closure Algorithm; 3.4 Worked Example — Step by Step; 3.5 A Second Worked Example — Closure That Does NOT Cover Everything |
| [Module 4: Armstrong's Axioms](part5/module4.md) | 4.1 What Are Armstrong's Axioms?; 4.2 Axiom 1 — Reflexivity; 4.3 Axiom 2 — Augmentation; 4.4 Axiom 3 — Transitivity; 4.5 Why Only These Three? (Soundness + Completeness, Informally) |
| [Module 5: Canonical Cover / Minimal Cover](part5/module5.md) | 5.1 What Is a Canonical Cover?; 5.2 Extraneous Attributes; 5.3 Redundant Dependencies; 5.4 The Canonical Cover Algorithm — Full Steps; 5.5 Full Worked Example — Computing a Canonical Cover |
| [Module 6: Keys from Functional Dependencies](part5/module6.md) | 6.1 Recap — What We Already Know; 6.2 Finding ALL Candidate Keys — The Systematic Approach; 6.3 Fully Worked Example — Finding All Candidate Keys; 6.4 Prime and Non-Prime Attributes — Full Treatment; 6.5 Why Prime vs Non-Prime Matters (Forward Reference) |
| [Module 7: Normalization](part5/module7.md) | 7.1 What Is Normalization?; 7.2 Why Normalize? — The Driving Motivation; 7.3 Data Redundancy; 7.4 Update Anomaly; 7.5 Insertion Anomaly |
| [Module 8: First Normal Form (1NF)](part5/module8.md) | 8.1 What Is First Normal Form (1NF)?; 8.2 Atomicity — What Does "Atomic" Actually Mean?; 8.3 A Concrete Violation and Its Fix; 8.4 Rules for 1NF Compliance — Complete List; 8.5 Examples — Compliant vs Non-Compliant |
| [Module 9: Second Normal Form (2NF)](part5/module9.md) | 9.1 What Is Second Normal Form (2NF)?; 9.2 When Is 2NF Automatically Satisfied?; 9.3 The Classic 2NF Violation — Worked Example; 9.4 Detecting 2NF Violations — Systematic Procedure; 9.5 Converting a Relation to 2NF — The Decomposition Process |
| [Module 10: Third Normal Form (3NF)](part5/module10.md) | 10.1 What Is Third Normal Form (3NF)?; 10.2 The "Escape Hatch" — Why Clause (b) Exists; 10.3 The Classic 3NF Violation — Worked Example; 10.4 Detecting 3NF Violations — Systematic Procedure; 10.5 Converting a Relation to 3NF — Decomposition |
| [Module 11: Boyce-Codd Normal Form (BCNF)](part5/module11.md) | 11.1 What Is BCNF?; 11.2 BCNF's Rule, Restated Simply; 11.3 Revisiting `StudentAdvisor` — Now Checking BCNF; 11.4 Decomposing StudentAdvisor into BCNF; 11.5 The BCNF Decomposition Algorithm — General Form |
| [Module 12: Higher Normal Forms](part5/module12.md) | 12.1 Recap — Multivalued Dependency (MVD); 12.2 What Is Fourth Normal Form (4NF)?; 12.3 The Classic 4NF Violation — Full Worked Example; 12.4 Decomposing to 4NF; 12.5 Visual — 4NF Decomposition |
| [Module 13: Decomposition](part5/module13.md) | 13.1 What Is Decomposition?; 13.2 Lossless-Join Decomposition; 13.3 Worked Example — Testing Lossless-Join; 13.4 Extending the Test to Multi-Way Decompositions; 13.5 Dependency Preservation |
| [Module 14: Denormalization](part5/module14.md) | 14.1 What Is Denormalization?; 14.2 Why Denormalize? — The Core Motivations; 14.3 A Worked Example — Denormalizing for Read Performance; 14.4 Advantages of Denormalization; 14.5 Disadvantages of Denormalization |
| [Module 15: Schema Design Best Practices](part5/module15.md) | 15.1 Naming Conventions; 15.2 Data Type Selection; 15.3 Nullability; 15.4 Constraints; 15.5 Future Scalability |
| [Module 16: Real-World Database Design](part5/module16.md) | 16.1 Banking System; 16.2 E-commerce; 16.3 Hospital Management; 16.4 University Management; 16.5 Library Management |
| [Module 17: Industry Trade-offs](part5/module17.md) | 17.1 Normalization vs Performance — The Recurring Theme, Formalized; 17.2 OLTP — Online Transaction Processing; 17.3 OLAP — Online Analytical Processing; 17.4 OLTP vs OLAP — Full Comparison Table; 17.5 Star Schema (Introduction) |
| [Module 18: Interview Mastery](part5/module18.md) | 18.1 How to Use This Module; 18.2 Functional Dependency Questions (Module 2); 18.3 Normalization Questions (Modules 7–12); 18.4 Candidate Key Questions (Modules 3, 6); 18.5 Armstrong's Axioms Problems (Module 4) |

[Back to contents](#contents)

## part6 — Transactions & Concurrency Control

Transaction fundamentals, ACID implementation, schedules, serializability, concurrency anomalies, locking, deadlocks, timestamp and optimistic protocols, and distributed transactions.

| Module | Topics covered |
|---|---|
| [Module 1: Transaction Fundamentals](part6/1.md) | 1.1 What Is a Transaction?; 1.2 Transaction Lifecycle; 1.3 Transaction States (In Depth); 1.4 Transaction Boundaries; 1.5 Autocommit |
| [Module 2: ACID Properties](part6/2.md) | 2.1 Overview: Why ACID as a Package?; 2.2 Atomicity; 2.3 Consistency; 2.4 Isolation; 2.5 Durability |
| [Module 3: Transaction Schedules](part6/3.md) | 3.1 What Is a Schedule?; 3.2 Serial Schedule; 3.3 Non-Serial Schedule; 3.4 Read/Write Operations & Conflict Basics; 3.5 Blind Writes |
| [Module 4: Serializability](part6/4.md) | 4.1 Why Serializability Matters; 4.2 Conflict Equivalence; 4.3 Conflict Serializability; 4.4 Precedence Graph (Serialization Graph); 4.5 View Equivalence |
| [Module 5: Concurrency Problems](part6/5.md) | 5.1 Why Study These Problems Explicitly?; 5.2 Lost Update; 5.3 Dirty Read (Reading Uncommitted Data); 5.4 Dirty Write; 5.5 Non-Repeatable Read (Fuzzy Read) |
| [Module 6: Lock-Based Concurrency Control](part6/6.md) | 6.1 The Lock Manager; 6.2 Shared Lock (S); 6.3 Exclusive Lock (X); 6.4 Lock Compatibility Matrix; 6.5 Update Lock (U) |
| [Module 7: Two-Phase Locking (2PL)](part6/7.md) | 7.1 The Core Idea; 7.2 Basic 2PL; 7.3 Strict 2PL; 7.4 Rigorous 2PL; 7.5 Conservative 2PL (Static 2PL) |
| [Module 8: Timestamp-Based Concurrency Control](part6/8.md) | 8.1 The Core Idea; 8.2 Timestamps — How They're Assigned; 8.3 Read and Write Timestamps on Data Items; 8.4 The Basic Timestamp Ordering (TO) Protocol Rules; 8.5 Worked Example — Basic Timestamp Ordering |
| [Module 9: Optimistic Concurrency Control (OCC)](part6/9.md) | 9.1 Why OCC? The Core Philosophy; 9.2 The Three Phases of OCC; 9.3 Read Phase; 9.4 Validation Phase; 9.5 Write Phase |
| [Module 10: Multi-Version Concurrency Control (MVCC)](part6/10.md) | 10.1 What Is MVCC?; 10.2 Why MVCC? What Problem Does It Solve?; 10.3 Version Chains; 10.4 Visibility Rules; 10.5 Snapshot Isolation |
| [Module 11: Isolation Levels](part6/11.md) | 11.1 What Is an Isolation Level?; 11.2 Read Uncommitted; 11.3 Read Committed; 11.4 Repeatable Read; 11.5 Serializable |
| [Module 12: Deadlocks](part6/12.md) | 12.1 What Is a Deadlock?; 12.2 The Four Necessary Conditions for Deadlock; 12.3 Wait-For Graph; 12.4 Deadlock Detection; 12.5 Deadlock Prevention |
| [Module 13: Transaction Control Language (TCL)](part6/13.md) | 13.1 What Is TCL?; 13.2 BEGIN / START TRANSACTION; 13.3 COMMIT; 13.4 ROLLBACK; 13.5 SAVEPOINT |
| [Module 14: Distributed Transactions (Introduction)](part6/14.md) | 14.1 Why Distributed Transactions?; 14.2 Two-Phase Commit (2PC) — Overview; 14.3 2PC — Failure Scenarios and the Blocking Problem; 14.4 Three-Phase Commit (3PC) — Overview; 14.5 XA Transactions — Overview |
| [Module 15: Industry Case Studies](part6/15.md) | 15.1 Banking — Funds Transfer; 15.2 ATM Systems; 15.3 E-commerce Checkout; 15.4 Airline Reservation; 15.5 Hospital Systems |
| [Module 16: Performance Considerations](part6/16.md) | 16.1 Lock Contention; 16.2 Hot Rows; 16.3 Long-Running Transactions; 16.4 Transaction Batching; 16.5 Lock Escalation Effects (Recap and Performance Framing) |
| [Module 17: Interview Mastery](part6/17.md) | 17.1 ACID Interview Questions — Rapid Fire; 17.2 Isolation Level Interview Questions — Rapid Fire; 17.3 Locking Interview Questions — Rapid Fire; 17.4 Deadlock Interview Questions — Rapid Fire; 17.5 MVCC Interview Questions — Rapid Fire |
| [Module 18: Preview of Part 7](part6/18.md) | 18.1 Why This Bridge Module Matters; 18.2 Connection to Indexes; 18.3 Connection to B+ Trees; 18.4 Connection to Storage Structures; 18.5 Connection to the Buffer Manager |

[Back to contents](#contents)

## part7 — Indexing, File Organization & Storage Structures

Storage fundamentals, file organization, indexing concepts and structures, B/B+ trees, hashing, index design, and physical data access.

| Module | Topics covered |
|---|---|
| [Module 1: Storage Fundamentals](part7/1.md) | 1.0 Why This Module Exists; 1.1 Why Databases Need Storage Engines; 1.2 Logical Storage vs Physical Storage; 1.3 Primary Storage vs Secondary Storage; 1.4 RAM vs SSD vs HDD — Deep Comparison |
| [Module 2: Physical Database Storage](part7/2.md) | 2.0 Why This Module Exists; 2.1 Page Layout — The Slotted Page Structure; 2.2 Fixed-Length Records vs Variable-Length Records; 2.3 Record Identifiers (RID); 2.4 Tuple Storage — Putting It All Together |
| [Module 3: File Organization](part7/3.md) | 3.0 Why This Module Exists; 3.1 Heap Files; 3.2 Sequential Files; 3.3 Sorted Files; 3.4 Clustered Files |
| [Module 4: Buffer Manager & Buffer Pool](part7/4.md) | 4.0 Why This Module Exists; 4.1 What is Buffer Management?; 4.2 Buffer Pool; 4.3 Buffer Frames; 4.4 Page Pinning |
| [Module 5: Index Fundamentals](part7/5.md) | 5.0 Why This Module Exists; 5.1 What is an Index?; 5.2 Why Indexes Exist; 5.3 Clustered Index; 5.4 Non-Clustered Index |
| [Module 6: B-Tree & B+ Tree](part7/6.md) | 6.0 Why Trees?; 6.1 B-Tree — Structure; 6.2 B+ Tree — Structure (the version databases actually use); 6.3 Fan-out and Height; 6.4 B+ Tree Search |
| [Module 7: Hash Indexing](part7/7.md) | 7.0 Why Hashing?; 7.1 Static Hashing; 7.2 Dynamic Hashing (Overview); 7.3 Extendible Hashing; 7.4 Linear Hashing |
| [Module 8: Advanced Index Types](part7/8.md) | 8.0 Why This Module Exists; 8.1 Multicolumn Indexes (Recap + Extension); 8.2 Descending Indexes; 8.3 Reverse Indexes; 8.4 BRIN Indexes (Block Range Index) |
| [Module 9: Storage Engine Internals](part7/9.md) | 9.0 Why This Module Exists |
| [Module 10: Index Selection & Optimization](part7/10.md) | 10.0 Why This Module Exists; 10.1 How Query Optimizers Choose Indexes; 10.2 Selectivity; 10.3 Cardinality; 10.4 Statistics and Histograms |
| [Module 11: Index Maintenance](part7/11.md) | 11.0 Why This Module Exists; 11.1 Rebuilding Indexes; 11.2 Reorganizing Indexes; 11.3 Fragmentation (Operational Recap); 11.4 Statistics Update |
| [Module 12: Performance Trade-offs](part7/12.md) | 12.0 Why This Module Exists; 12.1 Why Too Many Indexes Are Bad; 12.2 Write Amplification; 12.3 Read Optimization vs Write Cost — The Central Tension; 12.4 Insert Cost, Update Cost, Delete Cost — Broken Down Per Operation |
| [Module 13: Industry Case Studies](part7/13.md) | 13.0 How to Use This Module; 13.1 Banking; 13.2 E-commerce; 13.3 Inventory Management; 13.4 Healthcare |
| [Module 14: Interview Mastery](part7/14.md) | 14.0 How to Use This Module; 14.1 B+ Tree Interview Questions (Full Reference: Module 6); 14.2 Hash Index Interview Questions (Full Reference: Module 7); 14.3 Clustered vs Non-Clustered Index (Full Reference: Modules 3, 5, 9); 14.4 Composite Index Interview Questions (Full Reference: Modules 5, 8, 10) |

[Back to contents](#contents)

## part8 — Query Processing & Query Optimization

Query-processing lifecycle, parsing and rewriting, logical and physical optimization, access paths, join and sorting algorithms, aggregation, and execution plans.

| Module | Topics covered |
|---|---|
| [Module 1: Query Processing Fundamentals](part8/1.md) | 1.0 Why This Module Exists; 1.1 What is Query Processing?; 1.2 The Query Lifecycle (End-to-End Overview); 1.3 SQL Parsing; 1.3.1 Lexical Analysis (Tokenization) |
| [Module 2: Query Optimizer](part8/2.md) | 2.0 Why This Module Exists; 2.1 What is a Query Optimizer?; 2.2 Logical Plan vs. Physical Plan (Critical Distinction); 2.3 Rule-Based Optimization (RBO); 2.4 Cost-Based Optimization (CBO) |
| [Module 3: Query Transformation](part8/3.md) | 3.0 Why This Module Exists; 3.1 Predicate Pushdown; 3.2 Projection Pushdown; 3.3 Join Reordering; 3.4 Constant Folding |
| [Module 4: Statistics & Cardinality Estimation](part8/4.md) | 4.0 Why This Module Exists; 4.1 Table Statistics; 4.2 Column Statistics; 4.3 Histograms; 4.3.1 Frequency Histograms (Equi-Width) |
| [Module 5: Access Paths](part8/5.md) | 5.0 Why This Module Exists; 5.1 Sequential Scan; 5.2 Index Scan; 5.3 Index-Only Scan; 5.4 Bitmap Index Scan & Bitmap Heap Scan |
| [Module 6: Join Algorithms](part8/6.md) | 6.0 Why This Module Exists; 6.1 Nested Loop Join; 6.2 Block Nested Loop Join; 6.3 Indexed Nested Loop Join; 6.4 Hash Join |
| [Module 7: Sorting Algorithms in Databases](part8/7.md) | 7.0 Why This Module Exists; 7.1 Internal Sorting; 7.2 External Sorting; 7.3 External Merge Sort; 7.4 Multi-way Merge |
| [Module 8: Aggregation Processing](part8/8.md) | 8.0 Why This Module Exists; 8.1 Hash Aggregation; 8.2 Sort Aggregation; 8.3 Group Aggregation; 8.4 Streaming Aggregation |
| [Module 9: Query Execution Plans](part8/9.md) | 9.0 Why This Module Exists; 9.1 Execution Plan; 9.2 Logical Plan vs. Physical Plan (Revisited, Now With Real Syntax); 9.3 Cost: Estimated vs. Actual; 9.4 Estimated Rows vs. Actual Rows (The Most Important Diagnostic in This Module) |
| [Module 10: Parallel Query Execution](part8/10.md) | 10.0 Why This Module Exists; 10.1 Parallel Scans; 10.2 Parallel Joins; 10.3 Parallel Aggregation; 10.4 Parallel Sorting |
| [Module 11: Query Performance Tuning](part8/11.md) | 11.0 Why This Module Exists; 11.1 Slow Query Analysis: A Systematic Methodology; 11.2 Query Rewriting; 11.3 Reducing I/O; 11.4 Choosing Better Joins |
| [Module 12: Optimizer Hints (Overview)](part8/12.md) | 12.0 Why This Module Exists; 12.1 What Are Optimizer Hints?; 12.2 Oracle Hints; 12.3 SQL Server Hints; 12.4 MySQL Hints |
| [Module 13: Industry Case Studies](part8/13.md) | 13.0 How to Use This Module; 13.1 Case Study: Banking (Transaction Ledger Queries); 13.2 Case Study: E-Commerce (Product Search & Filtering); 13.3 Case Study: Healthcare (Patient Record Joins with Strict Correctness Requirements); 13.4 Case Study: Analytics (Large-Scale Aggregation Over Event Data) |
| [Module 14: Interview Mastery](part8/14.md) | 14.0 How to Use This Module; 14.1 Query Optimization Questions; 14.2 Join Algorithm Questions; 14.3 Execution Plan Questions; 14.4 Index Selection Questions |

[Back to contents](#contents)

## part9 — Database Recovery, Logging & Durability

Failure types, logging, WAL, undo/redo recovery, checkpoints, ARIES, buffer recovery, backup and restore, disaster recovery, and recovery in modern database systems.

| Module | Topics covered |
|---|---|
| [Module 1: Recovery Fundamentals](part9/1.md) | 1.1 What is Recovery?; 1.2 Durability — The "D" in ACID; 1.3 Failure Handling — The Core Responsibility; 1.4 Recovery Goals; 1. Guarantee Atomicity |
| [Module 2: Types of Failures](part9/2.md) | 2.1 Transaction Failure; 2.2 System Crash; 2.3 Process Failure; 2.4 Media Failure (Disk Failure); 2.5 Power Failure |
| [Module 3: Logging Fundamentals](part9/3.md) | 3.1 What is Logging?; 3.2 Log Records; 3.3 Log Sequence Number (LSN); 3.4 Log Buffer; 3.5 Persistent Log |
| [Module 4: Write-Ahead Logging (WAL)](part9/4.md) | 4.1 What is WAL?; 4.2 The WAL Rules — Formalized; 4.3 WAL Protocol — Step-by-Step Timeline; 4.4 Flush Order — The Non-Negotiable Sequencing; 4.5 Commit Protocol |
| [Module 5: Recovery Techniques](part9/5.md) | 5.1 Log-Based Recovery — Overview; 5.2 Undo Logging; 5.3 Redo Logging; 5.4 Undo/Redo Logging (Combined); 5.5 Shadow Paging |
| [Module 6: Checkpoints](part9/6.md) | 6.1 What is a Checkpoint?; 6.2 Why Checkpoints Exist — Deeper Dive; 6.3 Sharp Checkpoints; 6.4 Fuzzy Checkpoints; 6.5 Incremental Checkpoints |
| [Module 7: ARIES Recovery Algorithm](part9/7.md) | 7.1 What is ARIES?; 7.2 Physiological Logging; 7.3 Core Data Structures — Formalized; 7.4 The Three Phases of ARIES — Overview; 7.5 Phase 1: Analysis |
| [Module 8: Buffer Recovery](part9/8.md) | 8.1 Dirty Pages vs Clean Pages; 8.2 Flush Policies — The Two Core Questions, Formalized; 8.3 The Four Combinations; 8.4 STEAL + FORCE; 8.5 STEAL + NO-FORCE — The Industry Standard |
| [Module 9: Backup & Restore](part9/9.md) | 9.1 Full Backup; 9.2 Incremental Backup; 9.3 Differential Backup; 9.4 Physical Backup; 9.5 Logical Backup |
| [Module 10: Disaster Recovery](part9/10.md) | 10.1 Recovery Point Objective (RPO); 10.2 Recovery Time Objective (RTO); 10.3 Disaster Recovery Planning; 10.4 High Availability vs Disaster Recovery; 10.5 Geo-Redundancy |
| [Module 11: Recovery in Modern Databases](part9/11.md) | 11.1 PostgreSQL; 11.1.1 WAL Segments; 11.1.2 Checkpoints; 11.1.3 Archive Logs; 11.1.4 PITR in PostgreSQL — Concretely |
| [Module 12: Performance Considerations](part9/12.md) | 12.1 WAL Overhead; 12.2 Group Commit; 12.3 Log Buffer Size; 12.4 Checkpoint Frequency; 12.5 Recovery Time |
| [Module 13: Industry Case Studies](part9/13.md) | 13.1 Banking; 13.2 Payment Systems; 13.3 Airline Reservation; 13.4 Healthcare; 13.5 Cloud Databases |
| [Module 14: Interview Mastery](part9/14.md) | 14.1 WAL Interview Questions; 14.2 ARIES Questions; 14.3 Checkpoint Questions; 14.4 Undo/Redo Questions; 14.5 Recovery Algorithm Questions |

[Back to contents](#contents)

## part10 — Database Internals & Storage Engine Architecture

Database-engine architecture, storage engines, PostgreSQL and InnoDB internals, MVCC, buffer pools, memory management, query execution internals, storage optimization, and performance.

| Module | Topics covered |
|---|---|
| [Module 1: Database Engine Architecture](part10/1.md) | 1.0 Why This Module Exists; 1.1 What Is a Database Engine?; 1.2 The Big Picture: Engine Components; 1.3 The Complete SQL Request Lifecycle; 1.4 Component Deep Dives |
| [Module 2: Database Storage Engine](part10/2.md) | 2.0 Why This Module Exists; 2.1 What Is a Storage Engine (Physical Layer)?; 2.2 The Page: The Fundamental Unit of Storage; 2.3 Storage Models: Heap, Clustered, and Append-Only; 2.3.1 Heap Storage |
| [Module 3: PostgreSQL Internals](part10/3.md) | 3.0 Why This Module Exists; 3.1 PostgreSQL Process Architecture; 3.2 The Postmaster; 3.3 Backend Processes; 3.4 Shared Buffers |
| [Module 4: MySQL InnoDB Internals](part10/4.md) | 4.0 Why This Module Exists; 4.1 InnoDB Architecture Overview; 4.2 The InnoDB Buffer Pool; 4.3 Redo Log (InnoDB's WAL); 4.4 Undo Log |
| [Module 5: MVCC Internals](part10/5.md) | 5.0 Why This Module Exists; 5.1 What Is MVCC and Why Does It Exist?; 5.2 Transaction IDs (XIDs / Transaction Sequence Numbers); 5.3 Version Chains; 5.4 Snapshot Visibility & Read Views |
| [Module 6: Buffer Pool Internals](part10/6.md) | 6.0 Why This Module Exists; 6.1 What Is a Buffer Pool?; 6.2 Buffer Frames; 6.3 Pin Count; 6.4 Dirty Pages |
| [Module 7: Memory Management](part10/7.md) | 7.0 Why This Module Exists; 7.1 The Two Fundamental Categories: Shared vs. Local Memory; 7.2 Work Memory / Sort Memory / Hash Memory; 7.3 Temp Memory & Maintenance Memory; 7.4 Memory Contexts (PostgreSQL-Specific) |
| [Module 8: Query Execution Engine Internals](part10/8.md) | 8.0 Why This Module Exists; 8.1 The Volcano / Iterator Model; 8.2 Pipelining vs. Blocking Operators; 8.3 Pull vs. Push Execution Models (Broader Context); 8.4 Executor Node Types — Scan Nodes |
| [Module 9: Storage Optimizations](part10/9.md) | 9.0 Why This Module Exists; 9.1 Compression: The General Trade-off; 9.2 Dictionary Compression; 9.3 Prefix Compression; 9.4 Page Compression / Transparent Compression |
| [Module 10: Internal Background Processes](part10/10.md) | 10.0 Why This Module Exists; 10.1 PostgreSQL Background Processes — Complete Reference; 10.1.1 Postmaster (recap, Module 3); 10.1.2 Backend Processes (recap, Module 3); 10.1.3 WAL Writer (recap, Module 3) |
| [Module 11: Database Metadata](part10/11.md) | 11.0 Why This Module Exists; 11.1 What Is Database Metadata?; 11.2 Catalog Tables (PostgreSQL: `pg_catalog`); 11.3 Information Schema; 11.4 System Catalog (InnoDB / MySQL's Data Dictionary — Recap + Detail) |
| [Module 12: Database File Structure](part10/12.md) | 12.0 Why This Module Exists; 12.1 The Physical Hierarchy: Tablespace → Segment → Extent → Page; 12.2 PostgreSQL File Layout — Complete Reference; 12.3 MySQL/InnoDB File Layout — Complete Reference; 12.4 WAL Files / Redo Logs — Comparative Deep Dive (File-Level View) |
| [Module 13: Performance Engineering](part10/13.md) | 13.0 Why This Module Exists; 13.1 SSD vs. HDD — The Foundational Physical Reality; 13.2 Sequential vs. Random I/O (Formalized, Tied to Prior Modules); 13.3 Caching Layers — The Full Stack; 13.4 OS Page Cache Interaction (Deep Dive) |
| [Module 14: Industry Case Studies](part10/14.md) | 14.0 Why This Module Exists; 14.1 Case Study: `SELECT` with a Sequential Scan; 14.2 Case Study: `SELECT` with an Index Scan (High Selectivity); 14.3 Case Study: `INSERT`; 14.4 Case Study: `UPDATE` |
| [Module 15: Interview Mastery](part10/15.md) | 15.0 How to Use This Module; 15.1 PostgreSQL Internal Questions (Rapid-Fire); 15.2 InnoDB Questions (Rapid-Fire); 15.3 Buffer Pool Questions (Rapid-Fire); 15.4 MVCC Internals Questions (Rapid-Fire) |

[Back to contents](#contents)

## part11 — Distributed Databases, Replication, Sharding & Modern Architectures

Distributed database fundamentals, replication, sharding, consistency, distributed transactions, CAP/PACELC, consensus, NoSQL systems, cloud databases, NewSQL, and industry architectures.

| Module | Topics covered |
|---|---|
| [Module 1 — Distributed Database Fundamentals](part11/1.md) | 1.1 Why Distributed Databases Exist; 1.2 Evolution of Databases (Timeline); 1.3 Centralized vs Distributed DBMS; 1.4 Goals of Distributed Databases; 1.5 Challenges of Distributed Databases |
| [Module 2 — Distributed Database Design](part11/2.md) | 2.1 What Is Distributed Database Design?; 2.2 Data Fragmentation; 2.2.1 Horizontal Fragmentation; 2.2.2 Vertical Fragmentation; 2.2.3 Hybrid (Mixed) Fragmentation |
| [Module 3 — Replication](part11/3.md) | 3.1 What Is Replication?; 3.2 Why Replication? (Problems It Solves); 3.3 Replication Architectures — Overview; 3.4 Single Leader Replication (Primary-Replica / Master-Slave); 3.5 Multi Leader Replication (Master-Master) |
| [Module 4 — Sharding](part11/4.md) | 4.1 What Is Sharding?; 4.2 Why Sharding? (Problems It Solves); 4.3 Partition Keys (Shard Keys); 4.4 Sharding Strategies; 4.4.1 Range Sharding |
| [Module 5 — Distributed Transactions](part11/5.md) | 5.1 The Problem: Distributed ACID; 5.2 Two-Phase Commit (2PC); 5.3 Three-Phase Commit (3PC); 5.4 XA Transactions; 5.5 SAGA Pattern |
| [Module 6 — CAP Theorem](part11/6.md) | 6.1 History; 6.2 Brewer's Theorem — The Three Properties; 6.3 The Theorem Itself; 6.4 Practical Examples; 6.5 Database Classification Table |
| [Module 7 — PACELC Theorem](part11/7.md) | 7.1 Why PACELC?; 7.2 Breaking Down the Acronym; 7.3 The "Else" Branch — Latency vs Consistency; 7.4 PACELC Classification of Real Systems; 7.5 CAP vs PACELC — Side-by-Side |
| [Module 8 — Consensus Algorithms](part11/8.md) | 8.1 The Consensus Problem; 8.2 Leader Election — The Most Common Application; 8.3 Quorum and Majority Voting; 8.4 Raft; 8.5 Paxos |
| [Module 9 — NoSQL Databases](part11/9.md) | 9.1 Why NoSQL?; 9.2 NoSQL vs Relational Databases — Comparison Table; 9.3 The Four Major NoSQL Types; 9.3.1 Key-Value Stores; 9.3.2 Document Stores |
| [Module 10 — MongoDB](part11/10.md) | 10.1 What Is MongoDB?; 10.2 BSON; 10.3 Collections and Documents; 10.4 CRUD Operations; 10.5 Aggregation Pipeline |
| [Module 11 — Redis](part11/11.md) | 11.1 What Is Redis?; 11.2 Data Structures; 11.3 Persistence; 11.3.1 RDB (Redis Database File) — Snapshotting; 11.3.2 AOF (Append-Only File) — Operation Logging |
| [Module 12 — Cassandra](part11/12.md) | 12.1 What Is Cassandra?; 12.2 Architecture — The Ring; 12.3 Tokens and Partitioners; 12.4 Replication Factor (RF); 12.5 Consistency Levels |
| [Module 13 — Graph Databases](part11/13.md) | 13.1 Recap: Why Graph Databases?; 13.2 Neo4j — The Leading Graph Database; 13.3 Cypher — Neo4j's Query Language; 13.4 Traversal; 13.5 Graph Algorithms |
| [Module 14 — Cloud Databases](part11/14.md) | 14.1 Why Cloud Databases?; 14.2 Amazon RDS (Relational Database Service); 14.3 Amazon Aurora; 14.4 Amazon DynamoDB; 14.5 Google Cloud SQL |
| [Module 15 — NewSQL](part11/15.md) | 15.1 What Is NewSQL?; 15.2 Why NewSQL?; 15.3 The Core Architectural Recipe; 15.4 Google Spanner (Deep Dive); 15.5 CockroachDB |
| [Module 16 — Industry Architectures](part11/16.md) | 16.1 How to Approach These Case Studies (Interview Framing); 16.2 Netflix; 16.3 Uber; 16.4 Amazon; 16.5 Google |
| [Module 17 — Interview Mastery](part11/17.md) | 17.1 How to Use This Module; 17.2 CAP Theorem Questions; 17.3 PACELC Questions; 17.4 Replication Questions; 17.5 Sharding Questions |

[Back to contents](#contents)

## part12 — Production Database Engineering & DBA Mastery

Production lifecycle, security, administration, monitoring, performance, high availability, backup and disaster recovery, migrations, DevOps, database selection, failures, architectures, and capstone projects.

| Module | Topics covered |
|---|---|
| [Module 1: Production Database Lifecycle](part12/1.md) | 1.0 Why This Module Exists; 1.1 The Full Lifecycle — Big Picture; 1.2 Stage 1 — Development; 1.3 Stage 2 — Testing; 1.4 Stage 3 — Staging |
| [Module 2: Database Security](part12/2.md) | 2.0 Why This Module Exists; 2.1 Authentication — "Who are you?"; 2.2 Authorization — "What are you allowed to do?"; 2.2.1 Users; 2.2.2 Roles |
| [Module 3: Database Administration (DBA)](part12/3.md) | 3.0 Why This Module Exists; 3.1 Database Installation; 3.2 Configuration & Parameter Tuning; 3.3 Tablespaces; 3.4 Storage Allocation & Capacity Planning |
| [Module 4: Monitoring & Observability](part12/4.md) | 4.0 Why This Module Exists; 4.1 The Observability Pyramid for Databases; 4.2 Core Database Metrics; 4.2.1 Query Latency; 4.2.2 Throughput |
| [Module 5: Performance Engineering](part12/5.md) | 5.0 Why This Module Exists; 5.1 Benchmarking |
| [Module 6: High Availability](part12/6.md) | 6.0 Why This Module Exists; 6.1 High Availability — Core Concepts; 6.2 Active-Passive Architecture; 6.3 Active-Active Architecture; 6.4 Failover — Automatic vs. Manual |
| [Module 7: Backup & Disaster Recovery (Production)](part12/7.md) | 7.0 Why This Module Exists — HA vs. DR; 7.1 Backup Policies; 7.2 Backup Scheduling; 7.3 Point-in-Time Recovery (PITR) |
| [Module 8: Database Migrations](part12/8.md) | 8.0 Why This Module Exists; 8.1 Schema Migration vs. Data Migration; 8.2 Online vs. Offline Migration; 8.3 Locking Behavior — Why Migrations Are Risky; 8.4 Zero-Downtime Migration Techniques |
| [Module 9: DevOps & Databases](part12/9.md) | 9.0 Why This Module Exists; 9.1 Infrastructure as Code (IaC) |
| [Module 10: Database Selection](part12/10.md) | 10.0 Why This Module Exists; 10.1 The Fundamental Question: What Shape Is Your Data and Access Pattern?; 10.2 Relational Databases (SQL); 10.2.1 PostgreSQL; 10.2.2 MySQL |
| [Module 11: Common Production Failures](part12/11.md) | 11.0 Why This Module Exists; 11.1 Slow Queries; 11.2 Lock Contention; 11.3 Deadlocks; 11.4 Connection Exhaustion |
| [Module 12: Real-World Production Architectures](part12/12.md) | 12.0 A Note on This Module; 12.1 Google; 12.2 Amazon; 12.3 Netflix; 12.4 Uber |
| [Module 13: Complete Interview Mastery](part12/13.md) | 13.1 Theory Questions; 13.2 SQL Questions; 13.3 Normalization Questions; 13.4 Transaction Questions; 13.5 ACID Questions |
| [Module 14: Complete Revision](part12/14.md) | 14.1 The One-Page Revision Sheet; 14.2 Visual Mind Map — Part 12 Overview; 14.3 Formula Sheet; 14.4 Master Decision Trees (Consolidated); 14.5 Master Comparison Table — All Databases at a Glance (M10 recap) |
| [Module 15: Capstone Projects](part12/15.md) | 15.1 Capstone: Banking System; 15.2 Capstone: E-commerce Platform; 15.3 Capstone: Hospital Management System; 15.4 Capstone: Food Delivery App; 15.5 Capstone: Ride-Sharing App |
| [Module 16: Final Mastery Challenge](part12/16.md) | See the linked module for its detailed section index, examples, exercises, and revision material. |

[Back to contents](#contents)

## part13 (optional) — Research & Advanced Engineering (Optional)

Database history and research, modern architectures, LSM trees, columnar and HTAP systems, time-series, graph and streaming databases, vector databases, AI + databases, database-engine implementation, PostgreSQL contributions, and benchmarking.

| Module | Topics covered |
|---|---|
| [Module 1 — History of Database Systems](part13%28optional%29/1.md) | 1.1 Why History Matters for a Systems Engineer; 1.2 Hierarchical DBMS (1960s); 1.3 Network DBMS (late 1960s–1970s); 1.4 The Relational Revolution (1970–1980s); 1.5 Object-Oriented / Object-Relational Databases (1980s–1990s) |
| [Module 2 — Modern Database Architectures](part13%28optional%29/2.md) | 2.1 The Fundamental Question Every Architecture Answers; 2.2 Shared Nothing; 2.3 Shared Disk; 2.4 Shared Memory; 2.5 Comparison Table |
| [Module 3 — Database Research Papers (Part A of B)](part13%28optional%29/3.md) | 3.1 How to Read a Database Research Paper; 3.2 ARIES — Algorithm for Recovery and Isolation Exploiting Semantics; 3.3 Bigtable; 3.4 Dynamo; 3.5 DynamoDB |
| [Module 4 — LSM Trees (Log-Structured Merge Trees)](part13%28optional%29/4.md) | 4.1 Origin and Core Idea; 4.2 The Write Path; 4.3 SSTables in Detail; 4.4 The Read Path; 4.5 Bloom Filters |
| [Module 5 — Columnar Databases](part13%28optional%29/5.md) | 5.1 The Fundamental Layout Decision; 5.2 Row Store — When and Why; 5.3 Column Store — Internal Mechanics; 5.4 Vectorized Execution; 5.5 Compression |
| [Module 6 — HTAP Databases (Hybrid Transactional/Analytical Processing)](part13%28optional%29/6.md) | 6.1 Restating the Problem Precisely; 6.2 Why This Is Hard — The Physical Contradiction; 6.3 Strategy B in Depth — Real-Time Synchronized Dual Storage (TiFlash Pattern); 6.4 Strategy C in Depth — Delta/Main Store Merge (HANA/Hekaton Pattern); 6.5 Concurrency Control Implications for HTAP |
| [Module 7 — Time-Series Databases](part13%28optional%29/7.md) | 7.1 What Makes Time-Series Data Different; 7.2 The Core Architectural Pattern: Time-Partitioned, Series-Indexed Storage; 7.3 Time-Series-Specific Compression; 7.4 Gorilla — Facebook's In-Memory Time-Series Compression; 7.5 InfluxDB — Purpose-Built TSDB Internals |
| [Module 8 — Graph Processing](part13%28optional%29/8.md) | 8.1 Why Relational Databases Struggle with Graph Queries; 8.2 Index-Free Adjacency; 8.3 The Property Graph Model; 8.4 Two Fundamentally Different Graph Workloads; 8.5 Pregel — The Bulk Synchronous Parallel Model for Graph Analytics |
| [Module 9 — Streaming Databases](part13%28optional%29/9.md) | 9.1 Reframing: A Streaming Database Is a Database That Computes Incrementally; 9.2 Incremental View Maintenance (IVM) — The Theoretical Foundation; 9.3 Event Time vs. Processing Time; 9.3.1 Watermarks; 9.4 Windowing |
| [Module 10 — Vector Databases](part13%28optional%29/10.md) | 10.1 What Problem Vector Databases Actually Solve; 10.2 The ANN Algorithm Landscape — Three Families; 10.3 HNSW — Hierarchical Navigable Small World Graphs; 10.4 IVF — Inverted File Index; 10.5 Product Quantization (PQ) |
| [Module 11 — AI + Databases](part13%28optional%29/11.md) | 11.1 Two Directions, Not One Field; 11.2 Retrieval-Augmented Generation (RAG); 11.3 Learned Indexes; 11.4 AI Query Optimizers; 11.5 AI-Assisted Query Planning in Practice Today |
| [Module 12 — Building a Small Database Engine](part13%28optional%29/12.md) | 12.1 Architecture Overview; 12.2 Storage Engine — Pages and a B+ Tree; 12.3 Buffer Pool; 12.4 Write-Ahead Log (WAL) and ARIES-Style Recovery; 12.5 Transaction Manager — Locking |
| [Module 13 — Contributing to PostgreSQL](part13%28optional%29/13.md) | 13.1 Source Code Layout; 13.2 The Query Processing Entry Point — `tcop`; 13.3 The Planner (`src/backend/optimizer/`); 13.4 The Executor (`src/backend/executor/`); 13.5 The Heap — PostgreSQL's Row Storage |
| [Module 14 — Database Benchmarking](part13%28optional%29/14.md) | 14.1 Why Standardized Benchmarks Exist; 14.2 TPC-C — OLTP Benchmark; 14.3 TPC-H — Decision Support (Analytical) Benchmark; 14.4 TPC-DS — Modern Decision Support Benchmark; 14.5 YCSB — Yahoo! Cloud Serving Benchmark |
| [Module 15 — Capstone: Build a Miniature Relational Database](part13%28optional%29/15.md) | 15.1 Capstone Scope and Design Goals; 15.2 Complete Architecture; 15.3 Storage Engine — Page-Backed B+ Tree and the Catalog; 15.4 Query Engine — Completing UPDATE, DELETE, and JOIN; 15.5 Transactions and Recovery — Unchanged in Mechanism, Now Protecting Real Data |

[Back to contents](#contents)

---

## Repository Structure

```text
Database-Management-System/
├── part1/  (20 modules)
├── part2/  (18 modules)
├── part3/  (20 modules)
├── part4/  (15 modules)
├── part5/  (18 modules)
├── part6/  (18 modules)
├── part7/  (14 modules)
├── part8/  (14 modules)
├── part9/  (14 modules)
├── part10/  (15 modules)
├── part11/  (17 modules)
├── part12/  (16 modules)
├── part13(optional)/  (15 modules)
└── README.md
```

**Total:** 13 parts (including 1 optional part), 214 module files.
