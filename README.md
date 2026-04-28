# Oracle Database — Comprehensive Study Notes

---

## TABLE OF CONTENTS

1. [Database Concepts](#1-database-concepts)
2. [Oracle Database Architecture](#2-oracle-database-architecture)
3. [Memory Management & Startup Lifecycle](#3-memory-management--startup-lifecycle)
4. [Types of Database Files](#4-types-of-database-files)
5. [Initialization Parameter Files](#5-initialization-parameter-files)
6. [Data Dictionary & V$ Views](#6-data-dictionary--v-views)
7. [Tablespaces & Data Files (Info)](#7-tablespaces--data-files-info)
8. [SQL*Plus Commands](#8-sqlplus-commands)
9. [Oracle Multitenant Architecture](#9-oracle-multitenant-architecture)
10. [Database Memory Concepts (Deep Dive)](#10-database-memory-concepts-deep-dive)
11. [Database Storage Structures](#11-database-storage-structures)
12. [Tablespaces (Management)](#12-tablespaces-management)
13. [Undo Tablespace](#13-undo-tablespace)
14. [Data Files Management](#14-data-files-management)
15. [Managing Space Allocation](#15-managing-space-allocation)
16. [Managing DB Segments](#16-managing-db-segments)
17. [Database Users & Security](#17-database-users--security)
18. [Managing DB Connectivity](#18-managing-db-connectivity)
19. [Shared Server Architecture](#19-shared-server-architecture)
20. [Fast Recovery Area & Redo Log Management](#20-fast-recovery-area--redo-log-management)
21. [Oracle Backups](#21-oracle-backups)
22. [Database Recovery](#22-database-recovery)
23. [Storage and Space Management](#23-storage-and-space-management)

---

# 1. DATABASE CONCEPTS

## 1.1 Database Basics

A **database** is an organized collection of structured information or data, typically stored electronically in a computer system. A database is usually controlled by a **Database Management System (DBMS)**.

### Key Characteristics of a Database:
- **Persistence** — data survives beyond application execution
- **Shared access** — multiple users and applications can access data simultaneously
- **Data integrity** — rules enforced to keep data accurate and consistent
- **Concurrency control** — simultaneous access without corruption
- **Transaction support** — grouping operations into atomic units (ACID)

### ACID Properties (Critical Concept)

| Property | Definition | Example |
|---|---|---|
| **Atomicity** | Transaction is all-or-nothing | Bank transfer: both debit and credit happen or neither |
| **Consistency** | DB moves from one valid state to another | Balance cannot go negative if constraint exists |
| **Isolation** | Concurrent transactions don't interfere | Two users booking same seat see consistent data |
| **Durability** | Committed data survives crashes | Power loss after COMMIT doesn't lose the data |

```sql
-- Example: ACID-compliant bank transfer
BEGIN
  UPDATE accounts SET balance = balance - 500 WHERE acct_id = 101;
  UPDATE accounts SET balance = balance + 500 WHERE acct_id = 202;
  COMMIT;  -- Both updates permanent
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;  -- Neither update persists on error
END;
```

---

## 1.2 Types of Databases

### By Data Model:

| Type | Description | Examples |
|---|---|---|
| **Relational (RDBMS)** | Data in tables with rows/columns, relationships via keys | Oracle, MySQL, PostgreSQL, SQL Server |
| **Hierarchical** | Tree structure, parent-child relationships | IBM IMS |
| **Network** | Graph structure, records linked via sets | CODASYL databases |
| **Object-Oriented** | Data stored as objects like OOP | db4o, ObjectDB |
| **NoSQL — Document** | JSON/BSON documents | MongoDB, CouchDB |
| **NoSQL — Key-Value** | Simple key-value pairs | Redis, DynamoDB |
| **NoSQL — Column-Family** | Wide-column stores | Cassandra, HBase |
| **NoSQL — Graph** | Nodes and edges for relationships | Neo4j, Amazon Neptune |
| **NewSQL** | Relational + horizontal scalability | Google Spanner, CockroachDB |
| **Time-Series** | Optimized for time-stamped data | InfluxDB, TimescaleDB |

### By Deployment:

- **Centralized** — all data on one server
- **Distributed** — data spread across multiple nodes
- **Cloud** — hosted on cloud infrastructure (Oracle Autonomous DB)
- **In-Memory** — primarily stored in RAM (Oracle TimesTen)

---

## 1.3 Database Models

### The Relational Model (E.F. Codd, 1970)

The foundation of Oracle. Data is organized into **relations (tables)**:
- A **relation** = a table with rows (tuples) and columns (attributes)
- **Domain** = the set of allowed values for an attribute
- **Schema** = the structure definition (column names + types)
- **Instance** = actual data in the table at a point in time

#### Key Constraints:
- **Primary Key (PK)** — uniquely identifies each row; NOT NULL + UNIQUE
- **Foreign Key (FK)** — references PK in another table; enforces referential integrity
- **Unique Key** — UNIQUE but may allow one NULL
- **Check Constraint** — enforces domain rules (e.g., `salary > 0`)
- **NOT NULL** — column must have a value

```sql
CREATE TABLE departments (
    dept_id    NUMBER(4)    PRIMARY KEY,
    dept_name  VARCHAR2(30) NOT NULL UNIQUE
);

CREATE TABLE employees (
    emp_id     NUMBER(6)     PRIMARY KEY,
    first_name VARCHAR2(20)  NOT NULL,
    last_name  VARCHAR2(25)  NOT NULL,
    email      VARCHAR2(50)  UNIQUE NOT NULL,
    hire_date  DATE          DEFAULT SYSDATE,
    salary     NUMBER(8,2)   CHECK (salary > 0),
    dept_id    NUMBER(4)     REFERENCES departments(dept_id)
);
```

### Entity-Relationship (ER) Model

Used for **conceptual design** before physical implementation:

- **Entity** — a real-world object (e.g., EMPLOYEE, DEPARTMENT)
- **Attribute** — property of an entity (e.g., name, salary)
- **Relationship** — association between entities (WORKS_IN)
- **Cardinality** — 1:1, 1:N, M:N

#### Cardinality Examples:
```
EMPLOYEE (N) ----WORKS_IN---- (1) DEPARTMENT     [Many employees in one dept]
STUDENT  (M) ----ENROLLS---- (N) COURSE          [Many-to-many]
EMPLOYEE (1) ----HAS-------- (1) PASSPORT         [One-to-one]
```

### Object-Relational Model (Oracle's Extension)

Oracle extends the pure relational model with:
- **Object types** (user-defined types)
- **Nested tables** and **VARRAYs**
- **Object views**
- **REF types** (object references)

```sql
-- Oracle object type example
CREATE TYPE address_type AS OBJECT (
    street  VARCHAR2(100),
    city    VARCHAR2(50),
    country VARCHAR2(30)
);

CREATE TABLE customers (
    cust_id   NUMBER PRIMARY KEY,
    name      VARCHAR2(100),
    address   address_type    -- Nested object
);
```

---

## 1.4 Data Abstraction (Three-Level Architecture)

The ANSI/SPARC three-level architecture provides **data independence**:

```
┌────────────────────────────────────────────────────┐
│           EXTERNAL LEVEL (View Level)              │
│   User View 1   │   User View 2   │  User View 3   │
│  (SELECT-based  │  (App schema)   │  (Reporting)   │
│     views)      │                 │                │
└────────────────────────────────────────────────────┘
         ↕ Logical Data Independence
┌────────────────────────────────────────────────────┐
│          CONCEPTUAL LEVEL (Logical Level)          │
│                                                    │
│   Tables, Views, Indexes, Constraints, Triggers    │
│        (Oracle Schema Objects)                     │
└────────────────────────────────────────────────────┘
         ↕ Physical Data Independence
┌────────────────────────────────────────────────────┐
│           INTERNAL LEVEL (Physical Level)          │
│                                                    │
│   Data Files, Blocks, Extents, Segments,           │
│   Indexes stored on disk, Row format               │
└────────────────────────────────────────────────────┘
```

- **Physical Data Independence** — can change storage (move datafiles, compress) without changing logical schema
- **Logical Data Independence** — can change logical schema (add column) without breaking external views

---

## 1.5 Normalization

Normalization is the process of **organizing a database to reduce data redundancy and improve data integrity** by applying a series of normal forms.

### Why Normalize?
- Eliminate **insertion anomalies** (can't insert without unrelated data)
- Eliminate **update anomalies** (update one record, miss others)
- Eliminate **deletion anomalies** (delete row, lose unrelated info)

---

### First Normal Form (1NF)

**Rules:**
1. All columns must have atomic (indivisible) values
2. Each column must contain values of a single type
3. Each column must have a unique name
4. Order of rows/columns doesn't matter

```
❌ VIOLATES 1NF (multi-valued attribute):
┌──────────┬──────────────┬────────────────────────────┐
│ emp_id   │ name         │ phone_numbers              │
├──────────┼──────────────┼────────────────────────────┤
│ 101      │ Alice        │ 555-1234, 555-5678         │ ← Not atomic!
└──────────┴──────────────┴────────────────────────────┘

✅ SATISFIES 1NF:
┌──────────┬──────────────┬────────────────┐
│ emp_id   │ name         │ phone_number   │
├──────────┼──────────────┼────────────────┤
│ 101      │ Alice        │ 555-1234       │
│ 101      │ Alice        │ 555-5678       │
└──────────┴──────────────┴────────────────┘
```

---

### Second Normal Form (2NF)

**Prerequisite:** Must be in 1NF  
**Rule:** Every non-key attribute must be **fully functionally dependent** on the **entire primary key** (not just part of it — eliminates partial dependencies)  
**Applies to:** Tables with composite primary keys

```
❌ VIOLATES 2NF:
Table: ORDER_ITEMS (order_id, product_id, quantity, product_name)
PK = (order_id, product_id)

product_name depends ONLY on product_id (partial dependency!)

✅ SATISFIES 2NF (split the table):
ORDER_ITEMS (order_id, product_id, quantity)   ← PK = (order_id, product_id)
PRODUCTS    (product_id, product_name)          ← PK = product_id
```

---

### Third Normal Form (3NF)

**Prerequisite:** Must be in 2NF  
**Rule:** No non-key attribute should be transitively dependent on the primary key (eliminate transitive dependencies)

```
❌ VIOLATES 3NF:
Table: EMPLOYEES (emp_id, name, dept_id, dept_name)
PK = emp_id

dept_name depends on dept_id (not directly on emp_id) — TRANSITIVE DEPENDENCY!
emp_id → dept_id → dept_name

✅ SATISFIES 3NF:
EMPLOYEES   (emp_id, name, dept_id)    ← PK = emp_id
DEPARTMENTS (dept_id, dept_name)       ← PK = dept_id
```

---

### Boyce-Codd Normal Form (BCNF)

**Prerequisite:** Must be in 3NF  
**Rule:** For every functional dependency X → Y, X must be a **superkey** (stronger than 3NF)

```
❌ VIOLATES BCNF:
COURSE_TEACHER (student, course, teacher)
FD1: student, course → teacher  (superkey ✓)
FD2: teacher → course           (teacher is NOT a superkey ✗)

✅ SATISFIES BCNF:
Split into:
STUDENT_COURSE (student, teacher)
TEACHER_COURSE (teacher, course)
```

---

### Fourth Normal Form (4NF)

**Rule:** Eliminate **multi-valued dependencies** (MVDs)

```
❌ VIOLATES 4NF:
EMPLOYEE_SKILL_HOBBY (emp_id, skill, hobby)
emp_id →→ skill   (multi-valued dependency)
emp_id →→ hobby   (multi-valued dependency)
This causes redundant combinations.

✅ SATISFIES 4NF:
EMPLOYEE_SKILL (emp_id, skill)
EMPLOYEE_HOBBY (emp_id, hobby)
```

---

### Fifth Normal Form (5NF) / Project-Join Normal Form (PJNF)

**Rule:** A table is in 5NF if every **join dependency** in the table is implied by the candidate keys.

---

### Denormalization

Sometimes deliberately **violating normal forms** for **performance**:
- Fewer JOINs in queries = faster reads
- Use in data warehouses, reporting databases
- Oracle: Materialized views can store pre-joined, pre-aggregated data

```sql
-- Denormalized: store dept_name directly in employees for fast reads
ALTER TABLE employees ADD dept_name VARCHAR2(30);

-- Keep in sync with trigger (denormalization maintenance cost)
CREATE OR REPLACE TRIGGER sync_dept_name
AFTER UPDATE OF dept_name ON departments
FOR EACH ROW
BEGIN
  UPDATE employees SET dept_name = :NEW.dept_name
  WHERE dept_id = :NEW.dept_id;
END;
```

---

# 2. ORACLE DATABASE ARCHITECTURE

## 2.1 Overview: Oracle Instance vs. Oracle Database

One of the most critical distinctions in Oracle:

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORACLE INSTANCE                              │
│                                                                 │
│  ┌──────────────────────────┐  ┌──────────────────────────┐    │
│  │        SGA               │  │   Background Processes   │    │
│  │  ┌──────────────────┐    │  │   SMON  PMON  DBWn       │    │
│  │  │  Shared Pool     │    │  │   LGWR  CKPT  ARCn       │    │
│  │  │  Buffer Cache    │    │  │   MMON  MMNL  RECO       │    │
│  │  │  Redo Log Buffer │    │  └──────────────────────────┘    │
│  │  │  Large Pool      │    │                                   │
│  │  │  Java Pool       │    │  ┌──────────────────────────┐    │
│  │  │  Streams Pool    │    │  │      PGA (per session)   │    │
│  │  └──────────────────┘    │  └──────────────────────────┘    │
│  └──────────────────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
                        │ READ/WRITE
┌─────────────────────────────────────────────────────────────────┐
│                    ORACLE DATABASE (Files)                      │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │  Control │  │   Redo   │  │   Data   │  │   Archive    │   │
│  │  Files   │  │   Logs   │  │  Files   │  │   Log Files  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │
│                                                                 │
│  ┌──────────┐  ┌──────────┐                                     │
│  │  Temp    │  │  PFILE/  │                                     │
│  │  Files   │  │  SPFILE  │                                     │
│  └──────────┘  └──────────┘                                     │
└─────────────────────────────────────────────────────────────────┘
```

**Key Rule:** An **Instance** = Memory (SGA) + Background Processes. The **Database** = the physical files on disk. They are separate! You can mount a database to an instance, or start an instance without mounting a database.

---

## 2.2 Data Blocks (Oracle Blocks)

The **smallest unit of I/O** in Oracle. Also called **logical blocks** or **Oracle blocks**.

### Block Structure:
```
┌────────────────────────────────────────────────────┐
│                  BLOCK HEADER                      │
│  Block address, segment type, transaction entries  │
│  ITL (Interested Transaction List) — lock info     │
├────────────────────────────────────────────────────┤
│                   TABLE DIRECTORY                  │
│   (heap-organized tables in this block)            │
├────────────────────────────────────────────────────┤
│                   ROW DIRECTORY                    │
│   Pointers to row locations in this block          │
├──────────────────────────────┬─────────────────────┤
│         FREE SPACE           │     ROW DATA        │
│  (grows downward from top ↓) │  (grows upward ↑)   │
│                              │  Row N              │
│                              │  Row N-1            │
│                              │  ...                │
│                              │  Row 1              │
└──────────────────────────────┴─────────────────────┘
```

### Block Parameters:

| Parameter | Description | Default | Common Values |
|---|---|---|---|
| `DB_BLOCK_SIZE` | Standard block size | 8192 (8KB) | 4K, 8K, 16K, 32K |
| `PCTFREE` | % free space reserved for updates | 10 | 0–99 |
| `PCTUSED` | Min % used before new inserts allowed | 40 | 0–99 |
| `INITRANS` | Initial ITL entries | 1 (tables), 2 (indexes) | |
| `MAXTRANS` | Max concurrent transactions | 255 | |

```sql
-- Create table with custom block settings
CREATE TABLE orders (
    order_id    NUMBER PRIMARY KEY,
    order_date  DATE,
    total       NUMBER(10,2)
)
PCTFREE 20        -- Keep 20% free for row updates (rows expected to grow)
PCTUSED 60        -- Only accept new rows when used < 60%
INITRANS 4        -- Start with 4 ITL entries (high concurrency)
STORAGE (
    INITIAL  64K  -- First extent size
    NEXT     64K  -- Subsequent extent sizes
    MINEXTENTS 1
    MAXEXTENTS UNLIMITED
);
```

### Row Format in a Block:
Each row contains:
- **Row Header** — number of columns, lock byte, chained row pointer
- **Column Data** — column length + column value for each column

### Block Administration:

```sql
-- Check block information
SELECT segment_name, blocks, bytes 
FROM dba_segments 
WHERE owner = 'HR';

-- Dump a block (diagnostic)
ALTER SYSTEM DUMP DATAFILE 1 BLOCK 100;
-- Then check trace file in USER_DUMP_DEST

-- Analyze block space utilization
ANALYZE TABLE employees COMPUTE STATISTICS;
SELECT blocks, empty_blocks, avg_space, chain_cnt 
FROM dba_tables 
WHERE owner = 'HR' AND table_name = 'EMPLOYEES';
```

---

## 2.3 SGA — System Global Area

The **SGA** is a shared memory region allocated when the instance starts. All server processes and background processes share it.

### SGA Components:

```
┌─────────────────────────────────────────────────────────────┐
│                        SGA                                  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              DATABASE BUFFER CACHE                  │   │
│  │  Caches data blocks read from datafiles             │   │
│  │  DEFAULT pool, KEEP pool, RECYCLE pool              │   │
│  │  2K, 4K, 8K, 16K, 32K block caches                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  SHARED POOL                        │   │
│  │  Library Cache (parsed SQL, execution plans)        │   │
│  │  Dictionary Cache (cached data dictionary)          │   │
│  │  Result Cache (query result caching)                │   │
│  │  Shared SQL Area, PL/SQL procedure cache            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────┐  ┌──────────────────────────────┐    │
│  │  REDO LOG BUFFER │  │         LARGE POOL           │    │
│  │  Redo entries    │  │  Shared server UGA           │    │
│  │  waiting to be   │  │  RMAN I/O buffers            │    │
│  │  written to redo │  │  Parallel query buffers      │    │
│  │  log files       │  └──────────────────────────────┘    │
│  └──────────────────┘                                       │
│                                                             │
│  ┌──────────────────┐  ┌──────────────────────────────┐    │
│  │    JAVA POOL     │  │       STREAMS POOL           │    │
│  │  Java VM memory  │  │  Oracle Streams / GoldenGate │    │
│  └──────────────────┘  └──────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              FIXED SGA (overhead)                   │   │
│  │  Instance state, background process info            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Database Buffer Cache

The largest SGA component. Stores copies of data blocks.

**Buffer States:**
- **FREE** — not currently used, available
- **PINNED** — being accessed by a user process
- **DIRTY** — modified, not yet written to disk
- **CLEAN** — matches the on-disk version

**Buffer Cache Pools:**

| Pool | Purpose | Configuration |
|---|---|---|
| **DEFAULT** | Standard buffer caching | `DB_CACHE_SIZE` |
| **KEEP** | Objects you want to keep cached | `DB_KEEP_CACHE_SIZE` |
| **RECYCLE** | Large objects, don't keep cached | `DB_RECYCLE_CACHE_SIZE` |

```sql
-- Assign a table to the KEEP pool
ALTER TABLE lookup_codes STORAGE (BUFFER_POOL KEEP);

-- Check buffer cache hit ratio (should be > 95%)
SELECT 1 - (physical_reads / (db_block_gets + consistent_gets)) AS hit_ratio
FROM v$buffer_pool_statistics
WHERE name = 'DEFAULT';

-- Check current buffer cache size
SELECT component, current_size/1024/1024 AS size_mb 
FROM v$sga_dynamic_components;
```

**LRU (Least Recently Used) Algorithm:**
Oracle uses a modified LRU list. Full table scans use a special "touch count" mechanism so large scans don't flush the entire buffer cache.

### Shared Pool

Contains:
1. **Library Cache** — stores parsed SQL statements (parse trees, execution plans, PL/SQL code)
   - **Soft parse** — SQL found in library cache, reused (FAST)
   - **Hard parse** — SQL not found, must be fully parsed (SLOW, locks library cache)
   
2. **Data Dictionary Cache (Row Cache)** — caches rows from data dictionary tables (DBA_TABLES, etc.)

3. **Result Cache** — caches query result sets

```sql
-- Force SQL to use result cache
SELECT /*+ result_cache */ count(*) FROM employees;

-- Check library cache hit ratio (should be > 99%)
SELECT gethitratio, pinhitratio 
FROM v$librarycache 
WHERE namespace = 'SQL AREA';

-- Check shared pool usage
SELECT name, bytes/1024/1024 AS mb
FROM v$sgastat
WHERE pool = 'shared pool'
ORDER BY bytes DESC;

-- Clear shared pool (diagnostic only, NEVER in production)
ALTER SYSTEM FLUSH SHARED_POOL;
```

**Bind Variables — Critical for Shared Pool:**
```sql
-- BAD — causes hard parse for every different value
SELECT * FROM emp WHERE emp_id = 101;  -- hard parse
SELECT * FROM emp WHERE emp_id = 102;  -- hard parse again! (different literal)

-- GOOD — single parse, reused for different values
SELECT * FROM emp WHERE emp_id = :emp_id;  -- hard parse once, soft parse always
```

### Redo Log Buffer

- A **circular buffer** in SGA
- Server processes write redo entries to the buffer
- **LGWR** (Log Writer) flushes it to redo log files
- LGWR writes when: buffer is 1/3 full, every 3 seconds, COMMIT issued, DBWn writes

```sql
-- Check redo log buffer statistics
SELECT name, value FROM v$sysstat 
WHERE name IN ('redo log space requests', 'redo buffer allocation retries');
-- Both should be near zero; high values = buffer too small

-- Check redo log buffer size
SELECT name, value FROM v$parameter WHERE name = 'log_buffer';
```

### Large Pool

Optional, reduces pressure on shared pool for:
- Shared server (UGA stored here instead of shared pool)
- RMAN backup/restore I/O buffers
- Parallel execution message buffers
- Oracle XA transactions

```sql
ALTER SYSTEM SET large_pool_size = 64M;
```

---

## 2.4 PGA — Program Global Area

Private memory allocated for **each server process** (not shared). Contains:
- **SQL Work Areas** — sort area, hash join area, bitmap merge
- **Session Memory** — session variables, logon information
- **Cursor State** — open cursor information
- **Stack Space** — session variables in a dedicated server

```
┌─────────────────────────────────────────┐
│              PGA (per session)          │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │        SQL WORK AREAS           │    │
│  │  Sort Area (ORDER BY, GROUP BY) │    │
│  │  Hash Area (Hash Joins)         │    │
│  │  Bitmap Merge Area              │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │        SESSION MEMORY           │    │
│  │  Login info, session state      │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │        CURSOR STATE             │    │
│  │  Open cursor data, bind vars    │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

```sql
-- Check PGA usage
SELECT * FROM v$pgastat;

-- Check PGA per session
SELECT s.sid, s.username, p.pga_used_mem/1024/1024 AS pga_mb
FROM v$session s, v$process p
WHERE s.paddr = p.addr
AND s.type = 'USER'
ORDER BY 3 DESC;

-- PGA_AGGREGATE_TARGET - total PGA Oracle can use
ALTER SYSTEM SET pga_aggregate_target = 512M;
```

---

## 2.5 Background Processes

Oracle starts many background processes when the instance starts. Each has a specific job.

### Critical Background Processes:

#### DBWn — Database Writer (1 to 36 processes: DBW0–DBW9, DBWa–DBWz)

**Job:** Writes **dirty buffers** from the buffer cache to data files.

**When DBWn writes:**
- When a server process can't find a free buffer (LRU list threshold)
- At checkpoint
- When tablespace is taken OFFLINE/READ ONLY
- `ALTER TABLESPACE BEGIN BACKUP`
- DROP/TRUNCATE TABLE

```sql
-- Multiple database writers for high I/O workloads
ALTER SYSTEM SET db_writer_processes = 4;
-- Check DBWn activity
SELECT name, value FROM v$sysstat WHERE name LIKE 'physical writes%';
```

#### LGWR — Log Writer

**Job:** Writes redo log **buffer** to the online redo log **files**.

**When LGWR writes:**
- When redo log buffer is 1/3 full
- Every 3 seconds (timeout)
- Before DBWn writes a dirty buffer (Write-Ahead Logging rule)
- When any transaction COMMITs

**Critical for Durability:** COMMIT only returns to user AFTER LGWR confirms redo is written to disk.

```sql
-- Check LGWR statistics
SELECT name, value FROM v$sysstat 
WHERE name IN ('redo writes', 'redo write time');
```

#### CKPT — Checkpoint Process

**Job:** Updates the checkpoint position in the control file and data file headers. Signals DBWn to write.

**Types of Checkpoints:**
- **Full Checkpoint** — all dirty buffers written; occurs at normal shutdown, `ALTER SYSTEM CHECKPOINT`
- **Incremental Checkpoint** — continuous writing in background
- **Partial Checkpoint** — specific files (tablespace offline)
- **Log Switch Checkpoint** — triggered when a redo log group fills

```sql
-- Force a full checkpoint
ALTER SYSTEM CHECKPOINT;
-- Check checkpoint info
SELECT checkpoint_change# FROM v$database;
SELECT file#, checkpoint_change# FROM v$datafile;
```

#### SMON — System Monitor

**Jobs:**
1. **Instance recovery** at startup — applies redo (roll forward), then rolls back uncommitted transactions
2. **Coalesce free space** in temporary segments and dictionary-managed tablespaces
3. **Clean up temporary segments** — drops temp segments after a crash
4. **Recover dead transactions** for failed instances (RAC)
5. **Shrink rollback segments** as needed

#### PMON — Process Monitor

**Jobs:**
1. **Detects failed user processes** — cleans up after crashed sessions
2. Releases locks and resources held by dead processes
3. Rolls back uncommitted work of dead transactions
4. Releases SGA resources (latches, pins)
5. **Registers the database service** with the listener (dynamic registration)

```sql
-- Manually kill a session (PMON will clean up dead processes automatically)
ALTER SYSTEM KILL SESSION '123,456' IMMEDIATE;
```

#### ARCn — Archiver Process (0 to 30 processes)

**Job:** Copies filled redo log groups to **archive log destinations** before they are overwritten.

- Only active when database is in **ARCHIVELOG mode**
- Copies redo logs to local directories, remote sites (Data Guard), Oracle Secure Backup

```sql
-- Enable ARCHIVELOG mode
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Check archive log mode
SELECT log_mode FROM v$database;

-- Check archiver status
SELECT archiver FROM v$instance;

-- Set archive log destination
ALTER SYSTEM SET log_archive_dest_1 = 'LOCATION=/u01/arch';
```

#### MMON — Manageability Monitor

**Jobs:**
- Captures AWR (Automatic Workload Repository) snapshots
- Issues alerts based on metrics thresholds
- Captures statistics from memory to disk

#### MMNL — Manageability Monitor Light

**Job:** Captures statistics at high frequency (shorter interval than MMON), active session history (ASH).

#### RECO — Recoverer Process

**Job:** Resolves **in-doubt distributed transactions** after a network or remote database failure. Periodically tries to reconnect and resolve.

#### CJQ0 — Job Queue Coordinator

**Job:** Manages Oracle Scheduler jobs. Spawns `Jnnn` job queue slave processes.

#### DMON — Data Guard Monitor

**Job:** Manages Data Guard broker configuration.

#### Other Notable Processes:

| Process | Purpose |
|---|---|
| `LREG` | Listener Registration (replaced PMON registration in 12c+) |
| `FBDA` | Flashback Data Archiver (tracks history for Flashback Archive) |
| `SMCO` | Space Management Coordinator |
| `Wnnn` | Space Management Worker processes |
| `VKTM` | Virtual Keeper of Time (high-resolution time) |
| `GTX0–9` | Global Transaction processes (RAC) |
| `LMSn` | Lock Manager Service (RAC) |
| `LMD` | Lock Manager Daemon (RAC) |
| `DIAG` | Diagnostic Daemon |

---

## 2.6 Common Oracle Errors

### ORA- Error Reference

```
ORA-00001: unique constraint violated
  → INSERT/UPDATE violates a UNIQUE or PRIMARY KEY constraint

ORA-00017: session requested to set trace event
  → Trace requested by another session

ORA-00028: your session has been killed
  → DBA killed your session with ALTER SYSTEM KILL SESSION

ORA-00054: resource busy and acquire with NOWAIT specified
  → LOCK TABLE or SELECT FOR UPDATE NOWAIT failed; resource locked

ORA-00055: too many DML locks
  → Increase DML_LOCKS parameter

ORA-00060: deadlock detected while waiting for resource
  → Two sessions blocked each other; Oracle resolves by rolling back one

ORA-00257: archiver error. Connect internal only, until freed
  → Archive log destination is full! DB is suspended until space freed
  → Fix: Free space in archive dest or add destination

ORA-00600: internal error code, arguments: [...]
  → Oracle kernel bug; contact Oracle Support with trace file

ORA-01031: insufficient privileges
  → User lacks required system or object privilege

ORA-01033: ORACLE initialization or shutdown in progress
  → DB is starting up or shutting down

ORA-01034: ORACLE not available
  → Instance not started; start the instance

ORA-01078: failure in processing system parameters
  → Problem reading SPFILE or PFILE

ORA-01109: database not open
  → DB is mounted but not open; ALTER DATABASE OPEN required

ORA-01555: snapshot too old: rollback segment too small
  → Long-running query; undo data was overwritten
  → Fix: Increase UNDO_RETENTION or UNDO tablespace size

ORA-04031: unable to allocate N bytes of shared memory
  → Shared pool is fragmented or too small
  → Fix: Increase SHARED_POOL_SIZE, use bind variables

ORA-04068: existing state of packages has been discarded
  → Package recompiled while session had it open

ORA-12154: TNS:could not resolve the connect identifier
  → tnsnames.ora entry missing or incorrect

ORA-12514: TNS:listener does not currently know of service
  → Service not registered with listener; check lsnrctl status

ORA-19502: write error on file
  → Disk I/O error writing to datafile or temp file

ORA-27102: out of memory
  → OS cannot allocate requested SGA memory
  → Fix: Check OS memory limits (shmmax, shmall on Linux)
```

---

# 3. MEMORY MANAGEMENT & STARTUP LIFECYCLE

## 3.1 Memory Management Methods

Oracle offers three approaches to managing SGA and PGA memory:

### Method 1: Automatic Memory Management (AMM)

**Oracle 11g+** — Oracle manages BOTH SGA and PGA automatically.

- Single parameter: `MEMORY_TARGET` (total SGA + PGA)
- Oracle dynamically transfers memory between SGA and PGA based on demand
- Optional: `MEMORY_MAX_TARGET` (upper ceiling)

```sql
-- Enable AMM
ALTER SYSTEM SET memory_target = 2G SCOPE = SPFILE;
ALTER SYSTEM SET memory_max_target = 3G SCOPE = SPFILE;
-- Set SGA-specific params to 0 (AMM takes over)
ALTER SYSTEM SET sga_target = 0 SCOPE = SPFILE;
ALTER SYSTEM SET pga_aggregate_target = 0 SCOPE = SPFILE;
SHUTDOWN IMMEDIATE;
STARTUP;

-- Verify
SHOW PARAMETER memory_target;
SELECT * FROM v$memory_target_advice;

-- Check current distribution
SELECT component, current_size/1024/1024 AS current_mb, 
       min_size/1024/1024 AS min_mb
FROM v$memory_dynamic_components;
```

**Limitations of AMM:**
- Cannot use with HugePages on Linux (conflicts)
- Less predictable than ASMM in some workloads
- Not available if using `/dev/shm` is disabled

---

### Method 2: Automatic Shared Memory Management (ASMM)

**Oracle 10g+** — Oracle manages SGA components automatically but PGA is configured separately.

- `SGA_TARGET` — total SGA size (Oracle manages components within it)
- `PGA_AGGREGATE_TARGET` — Oracle manages PGA per-process allocation
- Oracle auto-tunes: Buffer Cache, Shared Pool, Large Pool, Java Pool, Streams Pool

```sql
-- Enable ASMM
ALTER SYSTEM SET sga_target = 1G SCOPE = SPFILE;
ALTER SYSTEM SET sga_max_size = 2G SCOPE = SPFILE;
ALTER SYSTEM SET pga_aggregate_target = 512M SCOPE = SPFILE;
-- Keep memory_target = 0 (disables AMM)
ALTER SYSTEM SET memory_target = 0 SCOPE = SPFILE;
SHUTDOWN IMMEDIATE;
STARTUP;

-- Check ASMM advice
SELECT * FROM v$sga_target_advice;
SELECT * FROM v$pga_target_advice;

-- Check auto-tuned components
SELECT component, current_size/1024/1024 AS current_mb,
       last_oper_type, last_oper_time
FROM v$sga_dynamic_components;
```

**Non-auto-tunable components** (manually set even with ASMM):
- `LOG_BUFFER` — redo log buffer
- `DB_KEEP_CACHE_SIZE` — keep pool
- `DB_RECYCLE_CACHE_SIZE` — recycle pool
- `DB_nK_CACHE_SIZE` — non-standard block size caches

---

### Method 3: Manual Memory Management

Full control — DBA sets every component individually.

```sql
-- SGA Components
ALTER SYSTEM SET db_cache_size = 512M;
ALTER SYSTEM SET shared_pool_size = 256M;
ALTER SYSTEM SET large_pool_size = 64M;
ALTER SYSTEM SET java_pool_size = 64M;
ALTER SYSTEM SET streams_pool_size = 32M;
ALTER SYSTEM SET log_buffer = 16M;

-- Disable auto-management
ALTER SYSTEM SET sga_target = 0;
ALTER SYSTEM SET memory_target = 0;

-- PGA per process
ALTER SYSTEM SET sort_area_size = 1M;       -- deprecated but still works
ALTER SYSTEM SET hash_area_size = 2M;
ALTER SYSTEM SET pga_aggregate_target = 0;  -- disables automatic PGA

-- View current manual settings
SELECT name, value FROM v$parameter 
WHERE name IN ('db_cache_size','shared_pool_size','large_pool_size',
               'java_pool_size','log_buffer','sort_area_size');
```

---

### Comparison Table

| Feature | AMM | ASMM | Manual |
|---|---|---|---|
| Controls | SGA + PGA | SGA + PGA separately | All manually |
| Parameters | `MEMORY_TARGET` | `SGA_TARGET` + `PGA_AGGREGATE_TARGET` | Individual params |
| Flexibility | Highest | High | Full control |
| HugePages compatible | No | Yes | Yes |
| Oracle version | 11g+ | 10g+ | All |
| Recommended for | Simpler environments | Production RAC/Linux | Advanced tuning |

---

## 3.2 Database Startup Lifecycle

Oracle startup has four distinct phases:

```
                    SHUTDOWN
                       │
                    NOMOUNT           ← Instance started (SGA + BG processes)
                       │                No database attached
                    MOUNT             ← Control file read
                       │                Database files identified but not opened
                    OPEN              ← Data files and redo logs opened
                       │                Database accessible to users
                  [NORMAL OPERATION]
```

### Phase 1: NOMOUNT (Instance Started)

**What happens:**
1. Oracle reads the **parameter file** (SPFILE or PFILE)
2. Allocates **SGA** based on parameters
3. Starts **background processes** (PMON, SMON, DBWn, LGWR, CKPT, etc.)
4. Opens the **alert log** and trace files

**What's NOT done:** Control file, data files, redo logs are not accessed.

**Use case:** Creating a new database, recreating control files

```sql
STARTUP NOMOUNT;
-- or
STARTUP NOMOUNT PFILE='/u01/init.ora';

-- Check state
SELECT status FROM v$instance;  -- Returns: STARTED
```

### Phase 2: MOUNT (Database Mounted)

**What happens:**
1. Oracle reads the **control file** (location from parameter file)
2. Control file identifies all data files, redo log files
3. Database status, checkpoint SCN, archive log information read
4. Files are NOT yet opened for user access

**Use cases:**
- Full database recovery
- Enabling/disabling ARCHIVELOG mode
- Renaming data files
- Enabling Flashback Database

```sql
-- From SHUTDOWN:
STARTUP MOUNT;

-- From NOMOUNT:
ALTER DATABASE MOUNT;

SELECT status FROM v$instance;  -- Returns: MOUNTED

-- Useful mount-state operations:
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE NOARCHIVELOG;
ALTER DATABASE RENAME FILE '/old/path/file.dbf' TO '/new/path/file.dbf';
RECOVER DATABASE;
ALTER DATABASE ENABLE FLASHBACK DATABASE;
```

### Phase 3: OPEN (Database Open)

**What happens:**
1. Oracle opens all **data files** and **online redo log files**
2. Verifies all files are consistent (checks SCN in control file vs file headers)
3. If crash occurred, SMON performs **instance recovery** automatically
4. Database is available to users

```sql
-- From SHUTDOWN:
STARTUP;  -- goes through NOMOUNT → MOUNT → OPEN automatically

-- From MOUNT:
ALTER DATABASE OPEN;

SELECT status FROM v$instance;  -- Returns: OPEN

-- Open read-only (for standby, reporting)
ALTER DATABASE OPEN READ ONLY;

-- Open with RESETLOGS (after incomplete recovery)
ALTER DATABASE OPEN RESETLOGS;
```

### Database Shutdown Modes

```sql
-- NORMAL: Waits for all users to disconnect; no instance recovery needed on restart
SHUTDOWN NORMAL;

-- IMMEDIATE: Disconnects all users, rolls back uncommitted transactions; 
--            checkpoints and closes cleanly; no instance recovery needed
SHUTDOWN IMMEDIATE;

-- TRANSACTIONAL: Waits for active transactions to complete; 
--                then disconnects; no instance recovery needed
SHUTDOWN TRANSACTIONAL;

-- ABORT: Kills instance immediately (like pulling power cord)
--        No checkpoint, no rollback; INSTANCE RECOVERY required on next startup
--        Use only if other methods fail
SHUTDOWN ABORT;
```

| Shutdown Mode | Waits for Users | Waits for Transactions | Checkpoint | Recovery Needed |
|---|---|---|---|---|
| NORMAL | Yes | Yes | Yes | No |
| TRANSACTIONAL | No | Yes | Yes | No |
| IMMEDIATE | No | No (rolls back) | Yes | No |
| ABORT | No | No | No | **YES** |

---

### Restricted Mode

```sql
-- Open in restricted mode (only users with RESTRICTED SESSION privilege)
STARTUP RESTRICT;
-- Or:
ALTER SYSTEM ENABLE RESTRICTED SESSION;
ALTER SYSTEM DISABLE RESTRICTED SESSION;

-- Force session disconnection during maintenance
ALTER SYSTEM KILL SESSION '12,345';
```

---

### Read-Only Mode

```sql
-- Open read-only (queries allowed, no DML/DDL)
ALTER DATABASE OPEN READ ONLY;

-- Check mode
SELECT open_mode FROM v$database;
-- Returns: READ WRITE or READ ONLY
```

---

# 4. TYPES OF DATABASE FILES

## 4.1 Control Files

The **control file** is the most critical Oracle file. It contains:
- Database name and identifier (DBID)
- Database creation timestamp
- Current log sequence number
- Names and locations of all data files and redo log files
- Checkpoint information (SCN, time)
- RMAN backup catalog (if using control file as catalog)
- Archive log history
- Flashback log information

**Critical facts:**
- Must be available at startup MOUNT phase
- Continuously updated during operation (every checkpoint)
- Should be **multiplexed** (2–3 copies on different disks)
- If all copies lost: must recreate using `CREATE CONTROLFILE` or restore from backup

```sql
-- View control file locations
SELECT name, status FROM v$controlfile;

-- View current control file parameter
SHOW PARAMETER control_files;

-- Add a control file copy (multiplexing) - requires restart
ALTER SYSTEM SET control_files = 
  '/u01/oradata/orcl/control01.ctl',
  '/u02/oradata/orcl/control02.ctl',
  '/u03/oradata/orcl/control03.ctl'
  SCOPE = SPFILE;
SHUTDOWN IMMEDIATE;
-- Manually copy the control file to new location
-- On OS: cp /u01/oradata/orcl/control01.ctl /u03/oradata/orcl/control03.ctl
STARTUP;

-- Backup control file to trace (creates a CREATE CONTROLFILE script)
ALTER DATABASE BACKUP CONTROLFILE TO TRACE;
-- File appears in diagnostic dest: adrci or alert log points to trace file

-- Backup control file to binary file
ALTER DATABASE BACKUP CONTROLFILE TO '/u01/backup/control.bkp';

-- View control file contents (metadata)
SELECT * FROM v$controlfile_record_section;
```

---

## 4.2 Online Redo Log Files

**Purpose:** Record all changes made to the database (the redo stream). Essential for:
- Instance recovery (apply redo after ABORT)
- Media recovery (replay changes on restored data files)
- Standby database synchronization

**Structure:**
- Organized in **groups** (typically 2–5 groups)
- Each group has **1 or more members** (copies) — multiplexing for protection
- Circular reuse: Oracle writes to group 1, then group 2, then group 3, then back to group 1
- **Log switch** = transitioning from one group to the next

```
GROUP 1: member1=/u01/redo01a.log, member2=/u02/redo01b.log  [CURRENT]
GROUP 2: member1=/u01/redo02a.log, member2=/u02/redo02b.log  [ACTIVE]
GROUP 3: member1=/u01/redo03a.log, member2=/u02/redo03b.log  [INACTIVE]
```

**Log Group Status:**
- `CURRENT` — currently being written to
- `ACTIVE` — needed for instance recovery (not yet checkpointed)
- `INACTIVE` — not needed for instance recovery
- `UNUSED` — never been written to (just added)

```sql
-- View redo log groups
SELECT group#, members, bytes/1024/1024 AS size_mb, status, archived
FROM v$log;

-- View redo log members
SELECT group#, member, status FROM v$logfile ORDER BY group#;

-- Add a redo log group
ALTER DATABASE ADD LOGFILE GROUP 4 (
  '/u01/oradata/orcl/redo04a.log',
  '/u02/oradata/orcl/redo04b.log'
) SIZE 200M;

-- Add a member to existing group
ALTER DATABASE ADD LOGFILE MEMBER '/u03/oradata/orcl/redo01c.log'
TO GROUP 1;

-- Drop a redo log group (must be INACTIVE and ARCHIVED)
ALTER DATABASE DROP LOGFILE GROUP 4;

-- Drop a member (cannot drop if it's the only member)
ALTER DATABASE DROP LOGFILE MEMBER '/u03/oradata/orcl/redo01c.log';

-- Force a log switch
ALTER SYSTEM SWITCH LOGFILE;

-- Force a checkpoint
ALTER SYSTEM CHECKPOINT;

-- Clear a log file (reinitialize — use if log is corrupted and not needed)
ALTER DATABASE CLEAR LOGFILE GROUP 3;
-- If unarchived (dangerous!):
ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 3;
```

---

## 4.3 Archive Log Files

Created when `ARCHIVELOG` mode is enabled. They are **copies of filled redo log groups**.

```sql
-- Check archive mode
SELECT log_mode, archiver FROM v$database;

-- View archive log history
SELECT name, sequence#, first_change#, next_change#, archived, status
FROM v$archived_log
ORDER BY sequence# DESC;

-- Configure archive destinations
ALTER SYSTEM SET log_archive_dest_1 = 'LOCATION=/u01/arch VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=orcl';
ALTER SYSTEM SET log_archive_dest_2 = 'SERVICE=standby ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=stdby';

-- Set archive log format
ALTER SYSTEM SET log_archive_format = 'arch_%t_%s_%r.arc' SCOPE=SPFILE;
-- %t = thread number, %s = sequence number, %r = resetlogs ID

-- Check archive log destination status
SELECT dest_id, dest_name, status, target, archiver, schedule
FROM v$archive_dest WHERE status != 'INACTIVE';

-- Manual archive (rarely needed)
ALTER SYSTEM ARCHIVE LOG CURRENT;
ALTER SYSTEM ARCHIVE LOG SEQUENCE 1234;
```

---

## 4.4 Data Files

Store actual database data (tables, indexes, etc.). Every data file belongs to exactly one tablespace.

```sql
-- View all data files
SELECT file_id, file_name, tablespace_name, bytes/1024/1024 AS size_mb,
       status, autoextensible, maxbytes/1024/1024 AS max_mb
FROM dba_data_files
ORDER BY tablespace_name;

-- View data file status in control file
SELECT file#, name, status, checkpoint_change# 
FROM v$datafile;
```

(See Section 14 for full data file management.)

---

## 4.5 Temp Files

Used by temporary tablespace for sort operations and hash joins that spill to disk.

```sql
-- View temp files
SELECT file_id, file_name, tablespace_name, bytes/1024/1024 AS size_mb,
       autoextensible
FROM dba_temp_files;

-- Add temp file
ALTER TABLESPACE temp ADD TEMPFILE '/u01/oradata/orcl/temp02.dbf' SIZE 1G AUTOEXTEND ON;
```

---

## 4.6 Password File

Stores passwords for users with **SYSDBA**, **SYSOPER**, **SYSBACKUP**, **SYSKM**, **SYSDG** privileges. Enables remote authentication.

```bash
# Create password file
orapwd file=$ORACLE_HOME/dbs/orapwORCL password=syspassword entries=10

# On Linux: located at $ORACLE_HOME/dbs/orapwSID
# On Windows: %ORACLE_HOME%\database\pwdSID.ora
```

```sql
-- Control password file usage
SHOW PARAMETER remote_login_passwordfile;
-- EXCLUSIVE = one DB uses the file (most common)
-- SHARED    = multiple DBs share the file (read-only)
-- NONE      = no password file; OS auth only

-- View users in password file
SELECT username, sysdba, sysoper, sysasm FROM v$pwfile_users;

-- Grant SYSDBA
GRANT SYSDBA TO backup_admin;
```

---

## 4.7 Alert Log File

**The most important diagnostic file.** Records:
- Database startup/shutdown with parameters
- All DDL statements (CREATE, ALTER, DROP)
- Errors (ORA- errors)
- Checkpoints, log switches
- Physical/logical corruption notices

```bash
# Location (Oracle 11g+): $ORACLE_BASE/diag/rdbms/<dbname>/<SID>/trace/alert_<SID>.log
# Example:
tail -f $ORACLE_BASE/diag/rdbms/orcl/orcl/trace/alert_orcl.log
```

```sql
-- Query alert log via V$ view (Oracle 11g+)
SELECT originating_timestamp, message_text
FROM v$diag_alert_ext
WHERE originating_timestamp > SYSDATE - 1/24  -- Last hour
ORDER BY originating_timestamp;
```

---

## 4.8 Trace Files

Generated by background or server processes for diagnostic info.

```sql
-- Enable SQL tracing for current session
ALTER SESSION SET SQL_TRACE = TRUE;
ALTER SESSION SET TRACEFILE_IDENTIFIER = 'my_trace';

-- Enable 10046 event (SQL trace with waits and bind variables)
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

-- Find your trace file
SELECT value FROM v$diag_info WHERE name = 'Default Trace File';

-- Format trace with TKPROF
-- (On OS)
-- tkprof tracefile.trc output.txt sys=no sort=exeela explain=user/pass
```

---

# 5. INITIALIZATION PARAMETER FILES

## 5.1 PFILE (Parameter File / init.ora)

A **plain text file** read at instance startup.

- **Location:** `$ORACLE_HOME/dbs/initSID.ora` (Linux)
- **Format:** `parameter_name = value`
- **Key properties:**
  - Static — changes require restart
  - Manual editing (text editor)
  - Original Oracle parameter file type

```ini
# Example PFILE (initORCL.ora)
DB_NAME                 = ORCL
DB_BLOCK_SIZE           = 8192
MEMORY_TARGET           = 1G
SGA_TARGET              = 768M
PGA_AGGREGATE_TARGET    = 256M
SHARED_POOL_SIZE        = 0        # auto-managed
DB_CACHE_SIZE           = 0        # auto-managed
PROCESSES               = 300
SESSIONS                = 335      # typically 1.1 * PROCESSES + 5
UNDO_MANAGEMENT         = AUTO
UNDO_TABLESPACE         = UNDOTBS1
CONTROL_FILES           = (/u01/oradata/orcl/control01.ctl,
                            /u02/oradata/orcl/control02.ctl)
DB_RECOVERY_FILE_DEST   = /u01/fra
DB_RECOVERY_FILE_DEST_SIZE = 10G
AUDIT_TRAIL             = DB
DIAGNOSTIC_DEST         = /u01/app/oracle
LOG_ARCHIVE_DEST_1      = 'LOCATION=/u01/arch'
```

---

## 5.2 SPFILE (Server Parameter File)

A **binary file** managed by Oracle. The preferred method since Oracle 9i.

- **Location:** `$ORACLE_HOME/dbs/spfileSID.ora` (Linux)
- **Format:** Binary (not directly editable)
- **Key properties:**
  - Changes take effect immediately (`SCOPE=BOTH`), at next startup (`SCOPE=SPFILE`), or in current session only (`SCOPE=MEMORY`)
  - Self-documenting: history of changes stored
  - Supports `ALTER SYSTEM` commands
  - Required for some features (AMM, ASMM parameter persistence)

### Creating and Managing SPFILE:

```sql
-- Create SPFILE from PFILE
CREATE SPFILE FROM PFILE;
CREATE SPFILE='/u01/custom/spfile.ora' FROM PFILE='/u01/init.ora';

-- Create PFILE from SPFILE (for viewing/backup)
CREATE PFILE FROM SPFILE;
CREATE PFILE='/tmp/pfile_backup.ora' FROM SPFILE;

-- Create PFILE from memory (current running parameters)
CREATE PFILE='/tmp/current_params.ora' FROM MEMORY;

-- Which file is being used?
SELECT name, value FROM v$parameter WHERE name = 'spfile';
-- If value is non-null: SPFILE is in use
-- If value is null: PFILE is in use

-- Check PFILE location Oracle would look for
SHOW PARAMETER spfile;
```

---

## 5.3 Parameter Types

### System-Wide vs. Session-Level:

| Scope | Who Can Change | When |
|---|---|---|
| `SCOPE=SPFILE` | DBA | Takes effect at next restart |
| `SCOPE=MEMORY` | DBA (or user for session params) | Takes effect immediately, lost on restart |
| `SCOPE=BOTH` | DBA | Immediate + persisted in SPFILE |
| `ALTER SESSION SET` | Current user | Current session only |

```sql
-- Examples of scope:
-- Static parameter (can only use SCOPE=SPFILE)
ALTER SYSTEM SET db_block_size = 16384 SCOPE = SPFILE;  -- restart required

-- Dynamic parameter (all scopes work)
ALTER SYSTEM SET sga_target = 2G SCOPE = BOTH;          -- immediate + persisted
ALTER SYSTEM SET sga_target = 2G SCOPE = MEMORY;        -- immediate only
ALTER SYSTEM SET sga_target = 2G SCOPE = SPFILE;        -- next restart only

-- Session-level parameter
ALTER SESSION SET nls_date_format = 'DD-MON-YYYY HH24:MI:SS';
ALTER SESSION SET optimizer_mode = FIRST_ROWS_10;
```

### Parameter Categories:

| Category | Description | Examples |
|---|---|---|
| **Basic** | Commonly configured | `DB_NAME`, `MEMORY_TARGET`, `PROCESSES` |
| **Advanced** | Expert tuning | `OPTIMIZER_FEATURES_ENABLE`, `_HIDDEN_PARAMS` |
| **Static** | Cannot change without restart | `DB_BLOCK_SIZE`, `DB_NAME` |
| **Dynamic** | Can change while running | `SGA_TARGET`, `PGA_AGGREGATE_TARGET` |
| **Hidden** | Undocumented, prefixed `_` | `_B_TREE_BITMAP_PLANS`, `_SMALL_TABLE_THRESHOLD` |

```sql
-- View all current parameters
SELECT name, value, description, isdefault, ismodified
FROM v$parameter
ORDER BY name;

-- View only non-default parameters
SELECT name, value FROM v$parameter 
WHERE isdefault = 'FALSE'
ORDER BY name;

-- View hidden parameters (use with caution!)
SELECT name, value, description 
FROM v$parameter 
WHERE name LIKE '\_%' ESCAPE '\';

-- Reset parameter to default
ALTER SYSTEM RESET sga_target SCOPE = BOTH SID = '*';
```

---

# 6. DATA DICTIONARY & V$ VIEWS

## 6.1 Data Dictionary Overview

The **data dictionary** is Oracle's internal repository describing the database structure. It consists of:
- **Base tables** — stored in the `SYSTEM` tablespace, owned by `SYS`; not directly accessed
- **Data dictionary views** — user-friendly views on the base tables
- **Dynamic performance views** (V$ views) — reflect current instance state

```
┌──────────────────────────────────────────────────────┐
│                DATA DICTIONARY                       │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │              BASE TABLES (SYS)                 │  │
│  │  OBJ$, TAB$, COL$, SEG$, UND$, FET$, EXT$...  │  │
│  │  (Never directly queried by users)             │  │
│  └────────────────────────────────────────────────┘  │
│                       ↓                              │
│  ┌────────────────────────────────────────────────┐  │
│  │           DICTIONARY VIEWS                     │  │
│  │   USER_*  : Objects owned by current user      │  │
│  │   ALL_*   : Objects accessible to current user │  │
│  │   DBA_*   : All objects in database (DBA only) │  │
│  │   CDB_*   : Cross-container (CDB only, 12c+)   │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │        DYNAMIC PERFORMANCE VIEWS (V$)          │  │
│  │   Reflect instance memory structures           │  │
│  │   Reset when instance restarts                 │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

---

## 6.2 Dictionary View Prefixes

| Prefix | Scope | Who Can See | Notes |
|---|---|---|---|
| `USER_` | Objects owned by current user | Any user | No `OWNER` column |
| `ALL_` | Objects current user can access | Any user | Includes `OWNER` column |
| `DBA_` | All objects in database | Requires DBA or explicit grant | Includes `OWNER` column |
| `CDB_` | All containers (12c+) | DBA in CDB root | Includes `CON_ID` column |

```sql
-- Examples: Same info, different scope
SELECT table_name FROM user_tables;                   -- My tables
SELECT owner, table_name FROM all_tables;             -- Tables I can see
SELECT owner, table_name FROM dba_tables;             -- ALL tables in DB
SELECT con_id, owner, table_name FROM cdb_tables;     -- All CDB tables (12c+)
```

---

## 6.3 Key Dictionary Views Reference

### Objects and Schema:

```sql
-- All objects in schema
SELECT object_name, object_type, status, last_ddl_time
FROM user_objects ORDER BY object_type;

-- Tables
SELECT table_name, num_rows, blocks, avg_row_len, last_analyzed
FROM user_tables;

-- Columns
SELECT table_name, column_id, column_name, data_type, 
       data_length, nullable, data_default
FROM user_tab_columns 
WHERE table_name = 'EMPLOYEES'
ORDER BY column_id;

-- Indexes
SELECT index_name, table_name, uniqueness, status, index_type
FROM user_indexes;

-- Constraints
SELECT constraint_name, constraint_type, table_name, 
       search_condition, r_constraint_name, status
FROM user_constraints 
WHERE table_name = 'EMPLOYEES';
-- Constraint types: P=Primary, U=Unique, R=Foreign Key, C=Check

-- Views
SELECT view_name, text FROM user_views;

-- Sequences
SELECT sequence_name, min_value, max_value, increment_by, 
       cycle_flag, order_flag, cache_size, last_number
FROM user_sequences;

-- Synonyms
SELECT synonym_name, table_owner, table_name, db_link
FROM user_synonyms;

-- Source code for procedures/functions/packages
SELECT name, type, line, text FROM user_source
WHERE name = 'MY_PROCEDURE'
ORDER BY line;

-- Triggers
SELECT trigger_name, trigger_type, triggering_event, 
       table_name, status
FROM user_triggers;
```

### Users and Security:

```sql
-- All users
SELECT username, user_id, account_status, lock_date, 
       expiry_date, default_tablespace, profile
FROM dba_users ORDER BY username;

-- Roles granted to users
SELECT grantee, granted_role, admin_option, default_role
FROM dba_role_privs WHERE grantee = 'HR';

-- System privileges
SELECT grantee, privilege, admin_option
FROM dba_sys_privs WHERE grantee = 'HR';

-- Object privileges
SELECT grantee, owner, table_name, privilege, grantable
FROM dba_tab_privs WHERE grantee = 'HR';
```

### Storage:

```sql
-- Tablespaces
SELECT tablespace_name, status, contents, extent_management,
       segment_space_management, bigfile
FROM dba_tablespaces;

-- Segments (tables, indexes consuming space)
SELECT owner, segment_name, segment_type, tablespace_name,
       bytes/1024/1024 AS size_mb, extents, blocks
FROM dba_segments
ORDER BY bytes DESC;

-- Extents
SELECT owner, segment_name, extent_id, file_id, block_id, 
       bytes/1024 AS size_kb, blocks
FROM dba_extents
WHERE owner = 'HR' AND segment_name = 'EMPLOYEES';

-- Free space in tablespaces
SELECT tablespace_name, bytes/1024/1024 AS free_mb, blocks
FROM dba_free_space
ORDER BY tablespace_name;
```

---

## 6.4 Dynamic Performance Views (V$ Views)

V$ views are **memory structures** reflecting the current state of the instance. They:
- Are owned by `SYS`
- Reset when the instance restarts (some persist via AWR)
- Provide real-time monitoring
- Accessible to `SELECT ANY DICTIONARY` or `SELECT_CATALOG_ROLE`

### Most Important V$ Views:

```sql
-- Instance information
SELECT instance_number, instance_name, host_name, version, 
       startup_time, status, database_status
FROM v$instance;

-- Database information
SELECT name, db_unique_name, dbid, open_mode, log_mode, 
       flashback_on, created
FROM v$database;

-- Current sessions
SELECT sid, serial#, username, status, schemaname, 
       osuser, machine, program, sql_id, event
FROM v$session 
WHERE type = 'USER';

-- Current SQL being executed
SELECT s.sid, s.username, q.sql_text, q.elapsed_time/1000000 AS elapsed_sec
FROM v$session s, v$sql q
WHERE s.sql_id = q.sql_id
AND s.status = 'ACTIVE';

-- Wait events
SELECT event, total_waits, time_waited, average_wait
FROM v$system_event
WHERE event NOT LIKE 'SQL*Net%'
ORDER BY time_waited DESC;

-- Session-level wait events
SELECT sid, event, wait_time, seconds_in_wait, state
FROM v$session_wait
WHERE wait_class != 'Idle';

-- Locks
SELECT l.sid, s.username, l.type, l.id1, l.id2, l.lmode, l.request
FROM v$lock l, v$session s
WHERE l.sid = s.sid
AND l.type IN ('TM', 'TX');

-- Lock blocking (who is blocking whom)
SELECT DISTINCT l1.sid AS blocking_sid, s1.username AS blocking_user,
                l2.sid AS waiting_sid,  s2.username AS waiting_user
FROM v$lock l1, v$lock l2, v$session s1, v$session s2
WHERE l1.block = 1
AND l2.request > 0
AND l1.id1 = l2.id1 AND l1.id2 = l2.id2
AND l1.sid = s1.sid AND l2.sid = s2.sid;

-- SGA memory components
SELECT name, bytes/1024/1024 AS mb FROM v$sgastat
WHERE pool IS NULL
UNION ALL
SELECT pool||' / '||name, bytes/1024/1024 FROM v$sgastat
WHERE pool IS NOT NULL
ORDER BY 2 DESC;

-- Buffer cache efficiency
SELECT db_block_gets, consistent_gets, physical_reads,
       ROUND(1 - physical_reads/(db_block_gets+consistent_gets), 4) AS hit_ratio
FROM v$buffer_pool_statistics WHERE name = 'DEFAULT';

-- Redo log information
SELECT group#, sequence#, bytes/1024/1024 AS size_mb, members, status, archived
FROM v$log;

-- Data files
SELECT file#, name, status, bytes/1024/1024 AS size_mb, 
       checkpoint_change#
FROM v$datafile;

-- Control files
SELECT name, status FROM v$controlfile;

-- Parameters (runtime)
SELECT name, value, description FROM v$parameter
WHERE isdefault = 'FALSE';

-- Top SQL by CPU
SELECT sql_id, executions, cpu_time/1000000 AS cpu_sec,
       elapsed_time/1000000 AS elapsed_sec,
       SUBSTR(sql_text, 1, 80) AS sql_text
FROM v$sql
ORDER BY cpu_time DESC
FETCH FIRST 10 ROWS ONLY;

-- I/O statistics per datafile
SELECT name, phyrds, phywrts, readtim, writetim
FROM v$filestat f, v$datafile d
WHERE f.file# = d.file#
ORDER BY phyrds + phywrts DESC;
```

### GV$ Views (RAC — Global V$ Views):

In a Real Application Clusters (RAC) environment, `GV$` views show data from all instances:

```sql
SELECT inst_id, instance_name, status FROM gv$instance;
SELECT inst_id, sid, username, status FROM gv$session WHERE type = 'USER';
```

---

# 7. TABLESPACES & DATA FILES (INFO)

## 7.1 Tablespace Overview

A **tablespace** is a logical storage container in Oracle. It:
- Groups related database segments (tables, indexes)
- Maps to one or more **data files** (or temp files) on disk
- Can be taken offline/online independently
- Can be backed up independently (transportable tablespaces)

```
Logical Structure:            Physical Structure:
─────────────────────         ───────────────────────
DATABASE                      ←→  Set of files
  └── TABLESPACE               ←→  1+ data files on disk
        └── SEGMENT            ←→  Contiguous space in a file
              └── EXTENT       ←→  Set of consecutive blocks
                    └── BLOCK  ←→  Smallest I/O unit (e.g., 8KB)
```

### Types of Tablespaces:

| Type | Purpose | Contents |
|---|---|---|
| **SYSTEM** | Oracle internal data dictionary | Base tables (SYS) |
| **SYSAUX** | Auxiliary system data | AWR, ASH, optimizer stats |
| **USERS** | Default user data | User tables/indexes |
| **TEMP** | Temporary operations | Sort segments, hash areas |
| **UNDO** | Rollback/undo data | Undo segments |
| **User-defined** | Application data | Custom tables/indexes |

```sql
-- Tablespace space usage report
SELECT t.tablespace_name,
       t.status,
       t.contents,
       ROUND(NVL(f.free_bytes,0)/1024/1024, 2) AS free_mb,
       ROUND(d.total_bytes/1024/1024, 2) AS total_mb,
       ROUND(100 * NVL(f.free_bytes,0)/d.total_bytes, 2) AS pct_free
FROM dba_tablespaces t
LEFT JOIN (SELECT tablespace_name, SUM(bytes) AS free_bytes 
           FROM dba_free_space GROUP BY tablespace_name) f
       ON t.tablespace_name = f.tablespace_name
LEFT JOIN (SELECT tablespace_name, SUM(bytes) AS total_bytes 
           FROM dba_data_files GROUP BY tablespace_name) d
       ON t.tablespace_name = d.tablespace_name
ORDER BY t.tablespace_name;
```

---

# 8. SQL*PLUS COMMANDS

## 8.1 Connecting and Starting SQL*Plus

```bash
# Connect as SYSDBA (from OS)
sqlplus / as sysdba
sqlplus sys/password@orcl as sysdba

# Connect as normal user
sqlplus hr/hr@orcl
sqlplus hr/hr@//localhost:1521/orcl

# Execute a script on startup
sqlplus hr/hr@orcl @/path/to/script.sql

# Run a single command and exit
sqlplus -s hr/hr@orcl << EOF
SELECT count(*) FROM employees;
EXIT;
EOF
```

## 8.2 SQL*Plus Commands Reference

### Connection Commands:
```sql
CONNECT sys/password@orcl AS SYSDBA  -- Switch connection
DISCONNECT                            -- Disconnect but stay in SQL*Plus
EXIT [n]                             -- Exit SQL*Plus (optional return code)
QUIT                                 -- Same as EXIT
```

### Display and Formatting:
```sql
-- Column formatting
COLUMN employee_id     FORMAT 9999       HEADING "Emp ID"
COLUMN first_name      FORMAT A20        HEADING "First Name"
COLUMN salary          FORMAT $99,999.99 HEADING "Salary"
COLUMN hire_date       FORMAT A12        HEADING "Hire Date"

-- Page and line size
SET PAGESIZE 50         -- Rows per page (0 = no page breaks)
SET LINESIZE 150        -- Characters per line
SET WRAP ON/OFF         -- Wrap or truncate long lines

-- Number formatting
SET NUMFORMAT 999,999   -- Default number format
SET NUMWIDTH 12         -- Default number width

-- Output headers and footers
TTITLE 'Employee Report'              -- Title at top of page
BTITLE 'Confidential'                 -- Footer
TTITLE OFF                            -- Turn off title

-- Break on column (for grouped reports)
BREAK ON department_id SKIP 1
COMPUTE SUM OF salary ON department_id
SELECT department_id, last_name, salary FROM employees ORDER BY department_id;
CLEAR BREAKS
CLEAR COMPUTES
```

### Script Execution:
```sql
-- Run a script
@ /path/to/script.sql
START /path/to/script.sql

-- Run from current directory
@script.sql

-- Run with arguments (&1, &2 are substitution variables)
@script.sql arg1 arg2
-- In script: SELECT * FROM &1 WHERE dept = &2;

-- Echo commands as they execute
SET ECHO ON/OFF

-- Show SQL being run
SET VERIFY ON/OFF         -- Shows old vs new value for substitution vars
```

### Output Control:
```sql
SET FEEDBACK ON           -- Shows "N rows selected"
SET FEEDBACK 100          -- Only show if >= 100 rows
SET FEEDBACK OFF          -- No row count

SET HEADING ON/OFF        -- Show/hide column headings
SET TIMING ON             -- Show execution time after each statement
SET AUTOTRACE ON          -- Show explain plan + statistics
SET AUTOTRACE TRACEONLY   -- Plan only, suppress results
SET AUTOTRACE ON EXPLAIN  -- Execution plan only

-- Spool output to file
SPOOL /tmp/output.txt
SELECT * FROM employees;
SPOOL OFF

SPOOL /tmp/output.txt APPEND  -- Append to existing file
```

### Variable Handling:
```sql
-- Define a substitution variable
DEFINE var1 = 'HR'
DEFINE emp_id = 101

-- Use in SQL
SELECT * FROM &var1..employees WHERE employee_id = &emp_id;

-- Undefine
UNDEFINE var1

-- Accept user input
ACCEPT dept_name CHAR PROMPT 'Enter department name: '
SELECT * FROM departments WHERE department_name = '&dept_name';

-- DEFINE without value to list all definitions
DEFINE

-- Disable substitution
SET DEFINE OFF
```

### Editing:
```sql
-- Edit last SQL statement
EDIT        -- Opens in OS editor (vi, notepad)

-- List buffer
LIST        -- Shows current SQL buffer
L           -- Short form

-- Run buffer
/           -- Execute SQL in buffer
RUN         -- List and execute
R           -- Short form

-- Change a line
CHANGE /old/new/   -- Substitutes text in current line
C /old/new/        -- Short form

-- Append to buffer
APPEND text
A text             -- Short form

-- Input
INPUT             -- Enter new lines
I                 -- Short form

-- Clear buffer
CLEAR BUFFER
```

### Session Information:
```sql
SHOW USER              -- Current connected user
SHOW RELEASE           -- Oracle version
SHOW PARAMETER name    -- Show parameter value
SHOW SGA               -- SGA summary
SHOW ERRORS            -- Show compilation errors for last DDL
SHOW ERRORS PROCEDURE proc_name    -- Show errors for specific object
DESC table_name        -- Describe table structure
DESCRIBE employees     -- Full form
```

### Help:
```sql
HELP                -- Help menu
HELP TOPICS         -- List help topics
HELP SELECT         -- Help on SELECT
HELP SET            -- Help on SET commands
```

---

# 9. ORACLE MULTITENANT ARCHITECTURE

## 9.1 Overview

Introduced in **Oracle 12c**, the Multitenant option allows multiple **Pluggable Databases (PDBs)** to share a single **Container Database (CDB)** instance.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CDB (Container Database)                     │
│                                                                 │
│  ┌──────────────────────┐    ┌────────────────────────────────┐ │
│  │    CDB$ROOT (Root)   │    │  CDB Metadata / SGA / BG Procs│ │
│  │  SYSTEM, SYSAUX      │    │  (Shared by all PDBs)         │ │
│  │  Oracle-supplied objs│    └────────────────────────────────┘ │
│  └──────────────────────┘                                       │
│                                                                 │
│  ┌──────────────────────┐                                       │
│  │     PDB$SEED         │                                       │
│  │  Template for new    │                                       │
│  │  PDB creation        │                                       │
│  └──────────────────────┘                                       │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    PDB1      │  │    PDB2      │  │    PDB3      │          │
│  │  (Sales DB)  │  │  (HR DB)    │  │  (Finance)   │          │
│  │  Own schema  │  │  Own schema  │  │  Own schema  │          │
│  │  Own files   │  │  Own files   │  │  Own files   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### Key Concepts:

| Concept | Description |
|---|---|
| **CDB** | The overall container; has CDB$ROOT and all PDBs |
| **CDB$ROOT** | Root container; contains common objects, Oracle system objects |
| **PDB$SEED** | Template PDB; read-only; used to create new PDBs |
| **PDB** | Pluggable DB; self-contained application database |
| **Common User** | `C##` prefix; exists in root and all PDBs |
| **Local User** | Exists only in one PDB |
| **CON_ID** | Container ID: 0=none, 1=CDB$ROOT, 2=PDB$SEED, 3+=PDBs |

---

## 9.2 CDB/PDB Administration

```sql
-- Check if running in CDB
SELECT cdb FROM v$database;
SELECT con_id, con_name, open_mode FROM v$containers;

-- Current container
SHOW CON_NAME;
SELECT sys_context('USERENV','CON_NAME') AS container FROM dual;

-- Switch containers (from CDB$ROOT)
ALTER SESSION SET CONTAINER = pdb1;
ALTER SESSION SET CONTAINER = CDB$ROOT;

-- Create a PDB from seed
CREATE PLUGGABLE DATABASE pdb1 
  ADMIN USER pdb_admin IDENTIFIED BY password
  FILE_NAME_CONVERT = ('/u01/oradata/orcl/pdbseed/', 
                       '/u01/oradata/orcl/pdb1/');

-- Create PDB from existing PDB (clone)
CREATE PLUGGABLE DATABASE pdb2 FROM pdb1
  FILE_NAME_CONVERT = ('/u01/oradata/orcl/pdb1/',
                       '/u01/oradata/orcl/pdb2/');

-- Open/close PDBs
ALTER PLUGGABLE DATABASE pdb1 OPEN;
ALTER PLUGGABLE DATABASE pdb1 CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE ALL OPEN;          -- Open all PDBs
ALTER PLUGGABLE DATABASE ALL EXCEPT pdb2 OPEN;

-- Save state (auto-open on CDB restart)
ALTER PLUGGABLE DATABASE pdb1 SAVE STATE;

-- Drop PDB
ALTER PLUGGABLE DATABASE pdb1 CLOSE;
DROP PLUGGABLE DATABASE pdb1 INCLUDING DATAFILES;

-- Unplug PDB (export to XML manifest)
ALTER PLUGGABLE DATABASE pdb1 CLOSE;
ALTER PLUGGABLE DATABASE pdb1 UNPLUG INTO '/tmp/pdb1.xml';

-- Plug in from manifest
CREATE PLUGGABLE DATABASE pdb1 USING '/tmp/pdb1.xml'
  FILE_NAME_CONVERT = ('/old/path/', '/new/path/');
```

### Common Users vs. Local Users:

```sql
-- Common user (visible in all containers)
CREATE USER c##dba_common IDENTIFIED BY password CONTAINER = ALL;
GRANT DBA TO c##dba_common CONTAINER = ALL;

-- Local user (only in current PDB)
ALTER SESSION SET CONTAINER = pdb1;
CREATE USER app_user IDENTIFIED BY password;  -- Local user (no C## prefix needed in PDB)

-- Common role
CREATE ROLE c##app_role CONTAINER = ALL;
```

### CDB Views:

```sql
-- All containers
SELECT con_id, name, open_mode FROM v$pdbs;

-- All datafiles across all PDBs (from CDB$ROOT)
SELECT con_id, file_id, file_name, tablespace_name 
FROM cdb_data_files ORDER BY con_id;

-- All users across all PDBs
SELECT con_id, username, account_status FROM cdb_users ORDER BY con_id;

-- All segments across all PDBs
SELECT con_id, owner, segment_name, bytes/1024/1024 AS mb
FROM cdb_segments ORDER BY bytes DESC;
```

### Benefits of Multitenant:

1. **Consolidation** — many databases, one instance (shared SGA/BGP)
2. **Rapid provisioning** — clone PDB in seconds
3. **Isolation** — each PDB has own users, data, schema
4. **Patching** — patch/upgrade CDB, all PDBs benefit
5. **Portability** — unplug from one CDB, plug into another
6. **Resource management** — Oracle Resource Manager controls PDB resource usage

```sql
-- Resource management in CDB
CREATE CDB PLAN cdb_plan;

CREATE CDB PLAN DIRECTIVE cdb_plan_directive
  PLAN = cdb_plan
  PLUGGABLE_DATABASE = pdb1
  SHARES = 3
  UTILIZATION_LIMIT = 70;   -- Max 70% CPU

ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'cdb_plan';
```

---

# 10. DATABASE MEMORY CONCEPTS (DEEP DIVE)

## 10.1 Memory Architecture Summary

```
ORACLE PROCESS MEMORY ARCHITECTURE
═══════════════════════════════════════════════════════════════════

OS MEMORY
├── SGA (Shared Global Area) — SHARED BY ALL PROCESSES
│   ├── Database Buffer Cache
│   │   ├── DEFAULT pool  (standard data blocks)
│   │   ├── KEEP pool     (tables cached permanently)
│   │   └── RECYCLE pool  (large objects, not kept)
│   ├── Shared Pool
│   │   ├── Library Cache
│   │   │   ├── SQL Area  (shared SQL)
│   │   │   ├── PL/SQL Area
│   │   │   └── Cursor Cache
│   │   ├── Dictionary Cache (Row Cache)
│   │   └── Result Cache
│   ├── Redo Log Buffer  (circular, pre-commit buffer)
│   ├── Large Pool       (optional)
│   ├── Java Pool        (optional, for Java stored procs)
│   ├── Streams Pool     (optional, for GoldenGate)
│   └── Fixed SGA        (static overhead)
│
└── PGA (Program Global Area) — PRIVATE PER PROCESS
    ├── SQL Work Areas
    │   ├── Sort Area  (ORDER BY, GROUP BY, CREATE INDEX)
    │   ├── Hash Area  (hash joins)
    │   └── Bitmap Merge Area
    ├── Session Memory  (login info, NLS settings)
    └── Cursor Cache   (session-level cursor state)
```

## 10.2 Buffer Cache Deep Dive

### Buffer Cache Touch Count Algorithm:

Oracle uses a **modified LRU** (Least Recently Used) with two sub-lists:
- **Hot portion** (MRU end) — frequently accessed buffers
- **Cold portion** (LRU end) — rarely accessed buffers

When a buffer is read:
- First access: goes to cold end
- Second access: promoted to hot end ("touch count" = 2)
- Aged out from cold end when space needed

### Full Table Scans and Buffer Cache:

```sql
-- Small table: all blocks put in hot portion of buffer cache
-- Large table (> DB_BIG_TABLE_THRESHOLD): uses "direct path read" bypassing cache

-- Force cache for large table (hints):
SELECT /*+ CACHE(e) */ * FROM employees e;

-- Or set table-level:
ALTER TABLE large_lookup_table CACHE;    -- puts in hot end
ALTER TABLE large_lookup_table NOCACHE;  -- default

-- Check which tables are cached
SELECT table_name, cache FROM dba_tables WHERE cache = 'Y';
```

### Buffer Cache Sizing:

```sql
-- Buffer cache hit ratio (target > 95%)
SELECT 1 - (phyrds.value / (dbgets.value + cggets.value)) AS hit_ratio
FROM v$sysstat phyrds, v$sysstat dbgets, v$sysstat cggets
WHERE phyrds.name = 'physical reads'
AND dbgets.name   = 'db block gets'
AND cggets.name   = 'consistent gets';

-- Use DB_CACHE_ADVICE to size buffer cache
ALTER SYSTEM SET DB_CACHE_ADVICE = ON;
-- After workload runs:
SELECT size_for_estimate, buffers_for_estimate,
       estd_physical_read_factor, estd_physical_reads
FROM v$db_cache_advice
WHERE name = 'DEFAULT' AND block_size = 8192
ORDER BY size_for_estimate;
```

## 10.3 Shared Pool Deep Dive

### Library Cache: Soft Parse vs. Hard Parse

```
Hard Parse Flow:
┌────────────┐   SQL not      ┌─────────────┐   ┌────────────┐
│ User sends │──────────────→ │  Parse SQL  │→  │  Optimize  │
│   SELECT   │   in cache     │  (syntax,   │   │  (generate │
└────────────┘                │  semantics) │   │   plan)    │
                              └─────────────┘   └────────────┘
                                                      ↓
                                              ┌────────────┐
                                              │ Load into  │
                                              │ Library    │
                                              │ Cache      │
                                              └────────────┘

Soft Parse Flow:
┌────────────┐  SQL found     ┌─────────────┐
│ User sends │──────────────→ │  Reuse plan │→ Execute immediately
│   SELECT   │  in cache      │  from cache │
└────────────┘                └─────────────┘
```

### Cursor Sharing:

```sql
-- Force cursor sharing even for literal SQL (use with caution)
ALTER SYSTEM SET cursor_sharing = FORCE;
-- EXACT = default, only identical SQL shared
-- FORCE = similar SQL with different literals treated as same
-- SIMILAR = deprecated in 12.2+

-- Session SQL cursor cache
ALTER SESSION SET SESSION_CACHED_CURSORS = 100;

-- Check library cache performance
SELECT namespace, gethits, gethitratio, pins, pinhitratio,
       reloads, invalidations
FROM v$librarycache
ORDER BY namespace;
```

### Shared Pool Fragmentation:

The shared pool can become fragmented over time, causing ORA-04031 errors.

```sql
-- Check for ORA-04031 risk
SELECT to_char(aggregate_size/1024/1024,'999,999.99') AS total_mb,
       to_char(unused/1024/1024,'999,999.99')         AS free_mb,
       to_char(recreatable/1024/1024,'999,999.99')    AS recreatable_mb,
       to_char(pinned/1024/1024,'999,999.99')         AS pinned_mb
FROM v$shared_pool_reserved;

-- Prevent large objects from being aged out of shared pool
ALTER SYSTEM SET shared_pool_reserved_size = 32M;  -- 10% of shared_pool_size

-- Keep objects pinned in shared pool
EXECUTE DBMS_SHARED_POOL.KEEP('PACKAGE_NAME', 'P');  -- P=Package
EXECUTE DBMS_SHARED_POOL.KEEP('PROCEDURE_NAME', 'P');

-- Show what's pinned
SELECT owner, name, type, kept FROM v$db_object_cache WHERE kept = 'YES';
```

## 10.4 PGA Memory Management

### Work Area Sizing:

```sql
-- Automatic PGA management (recommended)
ALTER SYSTEM SET pga_aggregate_target = 1G;
ALTER SYSTEM SET workarea_size_policy = AUTO;  -- default with PGA_AGGREGATE_TARGET > 0

-- Check PGA usage and effectiveness
SELECT * FROM v$pgastat;
/*
Key statistics to check:
- aggregate PGA target parameter: configured target
- aggregate PGA auto target: what Oracle actually allocated
- total PGA inuse: currently used
- total PGA allocated: allocated (may be > inuse)
- maximum PGA allocated: peak usage
- cache hit percentage: should be > 90%
- over allocation count: times Oracle exceeded target
*/

-- PGA advice (suggests optimal PGA_AGGREGATE_TARGET)
SELECT pga_target_for_estimate/1024/1024 AS target_mb,
       estd_pga_cache_hit_percentage, estd_overalloc_count
FROM v$pga_target_advice
ORDER BY pga_target_for_estimate;

-- Check per-session PGA
SELECT s.sid, s.username, s.program,
       p.pga_used_mem/1024   AS used_kb,
       p.pga_alloc_mem/1024  AS alloc_kb,
       p.pga_max_mem/1024    AS max_kb
FROM v$session s JOIN v$process p ON s.paddr = p.addr
WHERE s.type = 'USER'
ORDER BY p.pga_used_mem DESC;
```

---

# 11. DATABASE STORAGE STRUCTURES

## 11.1 Logical Storage Hierarchy

```
DATABASE
  └── TABLESPACE (logical container)
        └── SEGMENT (a single database object)
              └── EXTENT (set of consecutive blocks)
                    └── ORACLE BLOCK (smallest I/O unit)
                          └── OS BLOCK (OS-level, usually 512B-4KB)
```

### The Segment

A **segment** is a set of extents allocated for a specific database object. There is a 1-to-1 mapping between database objects and segments (with some exceptions).

**Segment Types:**

| Segment Type | Description |
|---|---|
| **Table segment** | Stores table row data |
| **Index segment** | Stores B-tree or bitmap index entries |
| **Undo segment** | Stores undo (rollback) data |
| **Temporary segment** | Sort/hash operations |
| **LOB segment** | Stores BLOB/CLOB data exceeding inline threshold |
| **Cluster segment** | Table clusters |
| **Rollback segment** | Old undo type (manual undo management) |
| **Bootstrap segment** | Used during database startup |

```sql
-- View segments
SELECT owner, segment_name, segment_type, tablespace_name,
       bytes/1024/1024 AS size_mb, extents, blocks
FROM dba_segments
WHERE owner = 'HR'
ORDER BY bytes DESC;

-- Segment growth history
SELECT snap_id, obj#, object_name, growth
FROM dba_hist_seg_stat ss, dba_hist_seg_stat_obj so
WHERE ss.obj# = so.obj#
ORDER BY snap_id;
```

### The Extent

A **contiguous set of Oracle blocks** within a data file. Oracle allocates space in extents.

```sql
-- View extents for a segment
SELECT extent_id, file_id, block_id, bytes/1024 AS size_kb, blocks
FROM dba_extents
WHERE owner = 'HR' AND segment_name = 'EMPLOYEES'
ORDER BY extent_id;
```

### Extent Allocation Policies:

1. **Locally Managed Tablespaces (LMT)** — extent maps stored in tablespace header (bitmap), NOT in data dictionary
   - `UNIFORM SIZE n` — all extents are exactly the same size
   - `AUTOALLOCATE` — Oracle chooses extent size based on segment size

2. **Dictionary Managed Tablespaces (DMT)** — extent maps in data dictionary (legacy, not recommended)

```sql
-- Create locally managed tablespace with uniform extents
CREATE TABLESPACE uniform_ts
DATAFILE '/u01/oradata/orcl/uniform01.dbf' SIZE 1G
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M;

-- Create locally managed with autoallocate (default)
CREATE TABLESPACE auto_ts
DATAFILE '/u01/oradata/orcl/auto01.dbf' SIZE 1G
EXTENT MANAGEMENT LOCAL AUTOALLOCATE;
```

---

## 11.2 Row Chaining and Row Migration

### Row Chaining
Occurs when a row is too large to fit in a single block. The row spans multiple blocks and is chained together. **Usually permanent** (for very wide rows).

### Row Migration
Occurs when an UPDATE makes a row too large for its current block. The row is moved to a new block, but a forwarding pointer remains in the original block. **Causes double I/O** (read original block + new block).

```sql
-- Detect row chaining/migration
ANALYZE TABLE employees COMPUTE STATISTICS;
SELECT chain_cnt FROM dba_tables WHERE owner = 'HR' AND table_name = 'EMPLOYEES';

-- Fix row migration (rebuild the table)
ALTER TABLE employees MOVE;           -- rebuilds table, invalidates indexes
ALTER INDEX emp_pk REBUILD;           -- rebuild invalidated index

-- Prevent row migration: increase PCTFREE for tables with heavy UPDATEs
ALTER TABLE employees PCTFREE 25;
```

---

# 12. TABLESPACES (MANAGEMENT)

## 12.1 Creating Tablespaces

```sql
-- Standard permanent tablespace
CREATE TABLESPACE app_data
  DATAFILE '/u01/oradata/orcl/app_data01.dbf' SIZE 1G AUTOEXTEND ON MAXSIZE 10G,
           '/u02/oradata/orcl/app_data02.dbf' SIZE 1G AUTOEXTEND ON MAXSIZE 10G
  EXTENT MANAGEMENT LOCAL AUTOALLOCATE
  SEGMENT SPACE MANAGEMENT AUTO
  DEFAULT COMPRESS FOR OLTP;  -- Table compression (12c+)

-- Bigfile tablespace (single very large file, up to 128TB with 8K blocks)
CREATE BIGFILE TABLESPACE bf_tablespace
  DATAFILE '/u01/oradata/orcl/bigfile01.dbf' SIZE 100G AUTOEXTEND ON;

-- Temporary tablespace
CREATE TEMPORARY TABLESPACE temp2
  TEMPFILE '/u01/oradata/orcl/temp02.dbf' SIZE 2G AUTOEXTEND ON
  EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M;

-- Undo tablespace
CREATE UNDO TABLESPACE undo2
  DATAFILE '/u01/oradata/orcl/undo02.dbf' SIZE 2G AUTOEXTEND ON;

-- Read-only tablespace (for historical data)
CREATE TABLESPACE hist_data
  DATAFILE '/u01/oradata/orcl/hist01.dbf' SIZE 10G;
ALTER TABLESPACE hist_data READ ONLY;
```

## 12.2 Modifying Tablespaces

```sql
-- Add a datafile to tablespace
ALTER TABLESPACE app_data 
  ADD DATAFILE '/u03/oradata/orcl/app_data03.dbf' SIZE 2G AUTOEXTEND ON MAXSIZE 20G;

-- Resize existing datafile
ALTER DATABASE DATAFILE '/u01/oradata/orcl/app_data01.dbf' RESIZE 2G;

-- Enable autoextend on existing datafile
ALTER DATABASE DATAFILE '/u01/oradata/orcl/app_data01.dbf' 
  AUTOEXTEND ON NEXT 100M MAXSIZE 10G;

-- Rename tablespace
ALTER TABLESPACE app_data RENAME TO application_data;

-- Take tablespace offline (normal)
ALTER TABLESPACE app_data OFFLINE;
-- Temporary: offline without checkpoint (if file is missing)
ALTER TABLESPACE app_data OFFLINE TEMPORARY;
-- Immediate (may need recovery)
ALTER TABLESPACE app_data OFFLINE IMMEDIATE;

-- Bring back online
ALTER TABLESPACE app_data ONLINE;

-- Make read-only
ALTER TABLESPACE hist_data READ ONLY;
-- Back to read-write
ALTER TABLESPACE hist_data READ WRITE;

-- Drop tablespace
DROP TABLESPACE app_data;                               -- Must be empty
DROP TABLESPACE app_data INCLUDING CONTENTS;            -- Drop even if has segments
DROP TABLESPACE app_data INCLUDING CONTENTS AND DATAFILES;  -- Also delete OS files
DROP TABLESPACE app_data INCLUDING CONTENTS AND DATAFILES CASCADE CONSTRAINTS;
```

## 12.3 Tablespace Monitoring

```sql
-- Tablespace usage summary
SELECT t.tablespace_name,
       t.status,
       t.contents,
       t.extent_management,
       t.segment_space_management,
       ROUND(NVL(free.free_mb, 0), 2) AS free_mb,
       ROUND(NVL(used.total_mb, 0), 2) AS total_mb,
       ROUND(100 * NVL(free.free_mb, 0) / NULLIF(NVL(used.total_mb, 0), 0), 2) AS pct_free
FROM dba_tablespaces t
LEFT JOIN (SELECT tablespace_name, SUM(bytes)/1024/1024 AS free_mb 
           FROM dba_free_space GROUP BY tablespace_name) free
       ON t.tablespace_name = free.tablespace_name
LEFT JOIN (SELECT tablespace_name, SUM(bytes)/1024/1024 AS total_mb 
           FROM dba_data_files GROUP BY tablespace_name) used
       ON t.tablespace_name = used.tablespace_name
ORDER BY t.tablespace_name;

-- Set threshold alerts (Oracle uses DBMS_SERVER_ALERT)
EXECUTE DBMS_SERVER_ALERT.SET_THRESHOLD(
  DBMS_SERVER_ALERT.TABLESPACE_PCT_FULL,  -- metric
  DBMS_SERVER_ALERT.OPERATOR_GE,          -- operator
  '85',                                   -- warning threshold (85%)
  DBMS_SERVER_ALERT.OPERATOR_GE,
  '97',                                   -- critical threshold (97%)
  1, 1,                                   -- observation period, consecutive occurrences
  NULL,                                   -- applies to all instances
  DBMS_SERVER_ALERT.OBJECT_TYPE_TABLESPACE,
  'APP_DATA'                              -- tablespace name
);
```

## 12.4 Segment Space Management

```sql
-- ASSM (Automatic Segment Space Management) — recommended
-- Manages free lists automatically using bitmaps
CREATE TABLESPACE auto_mgmt
  DATAFILE '/u01/oradata/orcl/auto01.dbf' SIZE 1G
  SEGMENT SPACE MANAGEMENT AUTO;

-- MSSM (Manual Segment Space Management) — legacy
-- Uses PCTFREE, PCTUSED, freelists
CREATE TABLESPACE manual_mgmt
  DATAFILE '/u01/oradata/orcl/manual01.dbf' SIZE 1G
  SEGMENT SPACE MANAGEMENT MANUAL;
```

---

# 13. UNDO TABLESPACE

## 13.1 What is Undo Data?

**Undo** (formerly called **rollback**) data stores the **before-image** of changed rows. It serves three purposes:

1. **Transaction Rollback** — `ROLLBACK` restores rows to original values
2. **Read Consistency** — long-running queries see a consistent snapshot (no dirty reads)
3. **Flashback Operations** — Flashback Query, Flashback Table use undo data

```
Transaction Timeline:
T1: BEGIN
T2: UPDATE employees SET salary = 5000 WHERE emp_id = 101;
    ┌─── Undo stores: "emp_id=101 had salary=4500"
T3: (Long running SELECT starts on another session)
T4: COMMIT  ← undo retained for UNDO_RETENTION period
T5: Another user's SELECT sees emp_id=101 with salary=4500 (not 5000!)
    ← Read consistency via undo!
```

## 13.2 Automatic Undo Management (AUM)

```sql
-- Enable AUM (recommended)
ALTER SYSTEM SET undo_management = AUTO SCOPE = SPFILE;
ALTER SYSTEM SET undo_tablespace = UNDOTBS1;
ALTER SYSTEM SET undo_retention  = 900;  -- Seconds to retain undo after commit

-- Check undo tablespace
SELECT tablespace_name, status, contents FROM dba_tablespaces 
WHERE contents = 'UNDO';

-- Current undo tablespace in use
SELECT value FROM v$parameter WHERE name = 'undo_tablespace';

-- Monitor undo usage
SELECT tablespace_name, status, SUM(bytes)/1024/1024 AS size_mb
FROM dba_undo_extents
GROUP BY tablespace_name, status;
/*
Status meanings:
ACTIVE   = currently used by running transactions
EXPIRED  = undo_retention period has passed, can be overwritten
UNEXPIRED= within undo_retention window, kept if space allows
*/

-- Undo advisor (estimate proper undo size)
SELECT d.undoblks, d.utl_begin, d.utl_end, d.maxquerylen, d.nospaceerrcnt,
       ROUND(d.undoblks * b.value / 1024 / 1024) AS est_mb
FROM v$undostat d, v$parameter b
WHERE b.name = 'db_block_size'
ORDER BY d.end_time DESC;
```

## 13.3 Undo Retention

```sql
-- Set undo retention (seconds)
ALTER SYSTEM SET undo_retention = 1800;  -- 30 minutes

-- Guarantee retention (don't overwrite unexpired undo even if tablespace full)
ALTER TABLESPACE undotbs1 RETENTION GUARANTEE;
ALTER TABLESPACE undotbs1 RETENTION NOGUARANTEE;  -- default

-- Check retention settings
SELECT tablespace_name, retention FROM dba_tablespaces WHERE contents = 'UNDO';
```

## 13.4 Undo Errors

```
ORA-01555: snapshot too old
  → Undo data overwritten before query completed
  → Fix: Increase UNDO_RETENTION or undo tablespace size
  → Workaround: Run query during off-peak hours or break into smaller batches

ORA-30036: unable to extend segment by N in undo tablespace
  → Undo tablespace is full and RETENTION GUARANTEE is set
  → Fix: Increase undo tablespace size or reduce UNDO_RETENTION
```

```sql
-- Add space to undo tablespace
ALTER TABLESPACE undotbs1 ADD DATAFILE '/u01/oradata/orcl/undo02.dbf' SIZE 2G;

-- Switch to a new undo tablespace
CREATE UNDO TABLESPACE undotbs2 DATAFILE '/u01/oradata/orcl/undo2_01.dbf' SIZE 4G;
ALTER SYSTEM SET undo_tablespace = UNDOTBS2;
-- Old tablespace stays until all its transactions are complete
DROP TABLESPACE undotbs1 INCLUDING CONTENTS AND DATAFILES;
```

---

# 14. DATA FILES MANAGEMENT

## 14.1 Data File States

| State | Meaning |
|---|---|
| `ONLINE` | Normal, in use |
| `OFFLINE` | Taken offline, not in use |
| `RECOVER` | Needs media recovery before can come online |
| `SYSOFF` | System tablespace offline (unusual, requires restart) |
| `MISSING` | File not found |

## 14.2 Adding and Resizing Data Files

```sql
-- Add datafile to tablespace
ALTER TABLESPACE app_data 
  ADD DATAFILE '/u01/oradata/orcl/app_data02.dbf' SIZE 2G 
  AUTOEXTEND ON NEXT 256M MAXSIZE UNLIMITED;

-- Resize datafile (can only shrink if space is available)
ALTER DATABASE DATAFILE '/u01/oradata/orcl/app_data01.dbf' RESIZE 5G;

-- Find the minimum size a datafile can be shrunk to
SELECT file_id, 
       CEIL( (NVL(hwm,1) * 8192) / 1024 / 1024) AS min_size_mb
FROM dba_data_files d,
     (SELECT file_id, MAX(block_id + blocks - 1) AS hwm 
      FROM dba_extents GROUP BY file_id) e
WHERE d.file_id = e.file_id (+)
ORDER BY file_id;
```

## 14.3 Moving / Renaming Data Files

### Method 1: Tablespace Offline (for non-SYSTEM tablespaces)

```sql
-- Step 1: Take tablespace offline
ALTER TABLESPACE app_data OFFLINE;

-- Step 2: Copy file at OS level
-- $ cp /u01/oradata/orcl/app_data01.dbf /u02/oradata/orcl/app_data01.dbf

-- Step 3: Rename in Oracle
ALTER TABLESPACE app_data 
  RENAME DATAFILE '/u01/oradata/orcl/app_data01.dbf' 
               TO '/u02/oradata/orcl/app_data01.dbf';

-- Step 4: Bring online
ALTER TABLESPACE app_data ONLINE;

-- Step 5: Remove old file at OS level (after confirming)
-- $ rm /u01/oradata/orcl/app_data01.dbf
```

### Method 2: ALTER DATABASE (for SYSTEM tablespace — needs MOUNT)

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;

-- Copy file at OS level
-- $ cp /old/path/system01.dbf /new/path/system01.dbf

ALTER DATABASE RENAME FILE '/old/path/system01.dbf' TO '/new/path/system01.dbf';
ALTER DATABASE OPEN;
```

### Method 3: Online Move (Oracle 12c+)

```sql
-- Move datafile while database is open (12c+)
ALTER DATABASE MOVE DATAFILE '/u01/oradata/orcl/app_data01.dbf' 
  TO '/u02/oradata/orcl/app_data01.dbf';

-- Keep old file as well (copy, don't move)
ALTER DATABASE MOVE DATAFILE '/u01/app_data01.dbf' 
  TO '/u02/app_data01.dbf' KEEP;
```

## 14.4 Taking Data Files Offline

```sql
-- Take individual datafile offline (requires recovery before ONLINE)
ALTER DATABASE DATAFILE '/u01/oradata/orcl/app_data01.dbf' OFFLINE;
ALTER DATABASE DATAFILE 7 OFFLINE;  -- By file number

-- Bring datafile online
ALTER DATABASE DATAFILE 7 ONLINE;   -- Oracle verifies consistency

-- If datafile needs recovery first:
RECOVER DATAFILE '/u01/oradata/orcl/app_data01.dbf';
-- Then:
ALTER DATABASE DATAFILE '/u01/oradata/orcl/app_data01.dbf' ONLINE;
```

---

# 15. MANAGING SPACE ALLOCATION

## 15.1 Extent Allocation

When a segment needs more space:
1. Oracle looks for free space in the current data file
2. If not enough, looks in other data files of same tablespace
3. If no space found, extends a data file (if AUTOEXTEND enabled)
4. If still no space: `ORA-01653: unable to extend table ... by N blocks`

```sql
-- Check free space in a tablespace (fragmented free blocks)
SELECT file_id, block_id, bytes/1024 AS free_kb, blocks
FROM dba_free_space
WHERE tablespace_name = 'APP_DATA'
ORDER BY file_id, block_id;

-- Coalesce free space (merge adjacent free extents)
ALTER TABLESPACE app_data COALESCE;

-- Segment shrink (release unused space within a segment back to tablespace)
-- Requires SEGMENT SPACE MANAGEMENT AUTO tablespace
ALTER TABLE employees ENABLE ROW MOVEMENT;  -- Required for shrink
ALTER TABLE employees SHRINK SPACE;          -- Compact data + reset HWM
ALTER TABLE employees SHRINK SPACE COMPACT;  -- Compact only, HWM stays
ALTER TABLE employees SHRINK SPACE CASCADE;  -- Also shrink dependent indexes
ALTER TABLE employees DISABLE ROW MOVEMENT;  -- Optional: disable after shrink
```

## 15.2 Segment Advisor

Oracle's built-in advisor recommends segments that can be shrunk:

```sql
-- Run Segment Advisor
DECLARE
  l_task_name VARCHAR2(100) := 'SEG_ADVISOR_' || TO_CHAR(SYSDATE,'YYYYMMDDHH24MISS');
  l_object_id NUMBER;
BEGIN
  DBMS_ADVISOR.CREATE_TASK('Segment Advisor', l_task_name);
  DBMS_ADVISOR.CREATE_OBJECT(l_task_name, 'SCHEMA', 'HR', NULL, NULL, NULL, l_object_id);
  DBMS_ADVISOR.SET_TASK_PARAMETER(l_task_name, 'recommend_all', 'TRUE');
  DBMS_ADVISOR.EXECUTE_TASK(l_task_name);
END;
/

-- View recommendations
SELECT f.task_name, o.attr1 AS object_name, f.message, f.more_info
FROM dba_advisor_findings f, dba_advisor_objects o
WHERE f.task_name = o.task_name AND f.object_id = o.object_id
ORDER BY f.task_name, f.object_id;
```

## 15.3 HWM — High Water Mark

The **High Water Mark (HWM)** is the boundary between used and never-used blocks in a segment. Blocks above the HWM have never been used. Full table scans read up to the HWM.

```
Block 1                        HWM                      Last block
│◄────────── Used blocks ──────►│◄── Never-used blocks ──►│
│ Row data  │ Row data  │ Empty │                         │
│ (below HWM)                   │ Above HWM: not scanned  │
└──────────────────────────────────────────────────────────┘
```

**Problem:** After mass DELETE, data is gone but HWM stays high → full scan still reads empty blocks!

```sql
-- Check HWM (approximate via blocks vs num_rows * avg_row_len)
SELECT table_name, num_rows, blocks, avg_row_len
FROM dba_tables WHERE owner = 'HR';

-- Reset HWM via TRUNCATE (removes all data)
TRUNCATE TABLE employees;  -- HWM reset to 0

-- Reset HWM via MOVE (preserves data, rebuilds segment)
ALTER TABLE employees MOVE;           -- Rebuilds, resets HWM, INVALIDATES indexes
ALTER INDEX emp_pk REBUILD;           -- Rebuild all indexes

-- Reset HWM via SHRINK (preserves data and indexes)
ALTER TABLE employees ENABLE ROW MOVEMENT;
ALTER TABLE employees SHRINK SPACE;   -- Resets HWM, indexes remain valid
ALTER TABLE employees DISABLE ROW MOVEMENT;
```

---

# 16. MANAGING DB SEGMENTS

## 16.1 Tables

```sql
-- Create table with storage clause
CREATE TABLE orders (
  order_id    NUMBER(10)   GENERATED ALWAYS AS IDENTITY,
  cust_id     NUMBER(10)   NOT NULL,
  order_date  DATE         DEFAULT SYSDATE NOT NULL,
  status      VARCHAR2(20) DEFAULT 'PENDING',
  total       NUMBER(12,2)
)
TABLESPACE app_data
PCTFREE 15
STORAGE (
  INITIAL     1M
  NEXT        1M
  MINEXTENTS  1
  MAXEXTENTS  UNLIMITED
)
COMPRESS FOR OLTP;   -- Advanced compression (licensed)

-- Rebuild table (move) to reclaim space or change tablespace
ALTER TABLE orders MOVE TABLESPACE new_tablespace;

-- Online table redefinition (12c+ for many operations)
EXECUTE DBMS_REDEFINITION.CAN_REDEF_TABLE('HR','EMPLOYEES', DBMS_REDEFINITION.CONS_USE_PK);

-- Table compression
ALTER TABLE orders COMPRESS;                    -- Basic compression (direct-path only)
ALTER TABLE orders COMPRESS FOR OLTP;           -- Advanced (DML-visible compression)
ALTER TABLE orders COMPRESS FOR QUERY LOW;      -- Hybrid Columnar Compression (Exadata)
ALTER TABLE orders NOCOMPRESS;                  -- Disable compression
```

## 16.2 Indexes

```sql
-- B-Tree index (default, most common)
CREATE INDEX emp_dept_idx ON employees(department_id);

-- Unique index
CREATE UNIQUE INDEX emp_email_idx ON employees(email);

-- Composite index (order matters for range scans)
CREATE INDEX emp_name_idx ON employees(last_name, first_name);

-- Function-based index
CREATE INDEX emp_upper_name_idx ON employees(UPPER(last_name));

-- Bitmap index (low cardinality columns, data warehouse)
CREATE BITMAP INDEX emp_gender_idx ON employees(gender);

-- Invisible index (won't be used by optimizer, but kept current)
CREATE INDEX emp_test_idx ON employees(hire_date) INVISIBLE;
ALTER INDEX emp_test_idx VISIBLE;

-- Reverse key index (prevents hot blocks in index — for sequence-based keys)
CREATE INDEX emp_seq_rev_idx ON orders(order_id) REVERSE;

-- Rebuild index
ALTER INDEX emp_dept_idx REBUILD;
ALTER INDEX emp_dept_idx REBUILD ONLINE;    -- No DML lock (12c+: always online)
ALTER INDEX emp_dept_idx REBUILD TABLESPACE new_ts;

-- Coalesce index (merge leaf blocks without reordering)
ALTER INDEX emp_dept_idx COALESCE;

-- Unusable → Rebuild
ALTER INDEX emp_dept_idx UNUSABLE;
ALTER INDEX emp_dept_idx REBUILD;

-- Index monitoring
ALTER INDEX emp_dept_idx MONITORING USAGE;
-- After some time:
SELECT index_name, used, start_monitoring FROM v$object_usage;
ALTER INDEX emp_dept_idx NOMONITORING USAGE;

-- Index statistics
ANALYZE INDEX emp_dept_idx COMPUTE STATISTICS;
SELECT index_name, blevel, leaf_blocks, clustering_factor, 
       num_rows, last_analyzed
FROM user_indexes WHERE index_name = 'EMP_DEPT_IDX';
```

## 16.3 LOB Segments

```sql
-- Table with LOB columns
CREATE TABLE documents (
  doc_id     NUMBER PRIMARY KEY,
  doc_name   VARCHAR2(200),
  doc_blob   BLOB,          -- Binary data (up to 4GB)
  doc_clob   CLOB,          -- Character data (up to 4GB)
  doc_nclob  NCLOB,         -- National character data
  doc_bfile  BFILE          -- External file reference
)
LOB (doc_blob) STORE AS BASICFILE (
  TABLESPACE lob_data
  CHUNK 8192
  ENABLE STORAGE IN ROW
)
LOB (doc_clob) STORE AS SECUREFILE (  -- SecureFile LOB (recommended)
  TABLESPACE lob_data
  DEDUPLICATE           -- Dedup identical LOBs
  COMPRESS HIGH         -- Compress LOB data
);
```

---

# 17. DATABASE USERS & SECURITY

## 17.1 Creating and Managing Users

```sql
-- Create user
CREATE USER hr_user IDENTIFIED BY "Secur3P@ss!"
  DEFAULT TABLESPACE app_data
  TEMPORARY TABLESPACE temp
  QUOTA 500M ON app_data
  QUOTA UNLIMITED ON temp
  PROFILE default
  PASSWORD EXPIRE;         -- Force password change on first login

-- Alter user
ALTER USER hr_user IDENTIFIED BY "N3wP@ss!";
ALTER USER hr_user QUOTA UNLIMITED ON app_data;
ALTER USER hr_user DEFAULT TABLESPACE new_ts;
ALTER USER hr_user ACCOUNT LOCK;     -- Lock account
ALTER USER hr_user ACCOUNT UNLOCK;   -- Unlock account
ALTER USER hr_user PASSWORD EXPIRE;  -- Force password change

-- Drop user
DROP USER hr_user;                   -- Only if user has no objects
DROP USER hr_user CASCADE;           -- Drop user + all their objects

-- Check user status
SELECT username, account_status, lock_date, expiry_date, 
       profile, default_tablespace, created
FROM dba_users WHERE username = 'HR_USER';
```

## 17.2 Privileges

### System Privileges:

```sql
-- Grant system privilege
GRANT CREATE SESSION    TO hr_user;        -- Allow login
GRANT CREATE TABLE      TO hr_user;        -- Create tables
GRANT CREATE VIEW       TO hr_user;        -- Create views
GRANT CREATE PROCEDURE  TO hr_user;        -- Create stored procedures
GRANT CREATE SEQUENCE   TO hr_user;        -- Create sequences
GRANT CREATE TRIGGER    TO hr_user;        -- Create triggers
GRANT CREATE ANY TABLE  TO hr_user;        -- Create tables in any schema
GRANT SELECT ANY TABLE  TO hr_user;        -- Query any table
GRANT DROP ANY TABLE    TO hr_user;        -- Drop any table (dangerous!)
GRANT ALTER ANY TABLE   TO hr_user;

-- Grant with ADMIN OPTION (can re-grant)
GRANT CREATE TABLE TO hr_user WITH ADMIN OPTION;

-- Revoke system privilege
REVOKE CREATE TABLE FROM hr_user;

-- View granted system privileges
SELECT grantee, privilege, admin_option
FROM dba_sys_privs WHERE grantee = 'HR_USER';
```

### Object Privileges:

```sql
-- Grant object privileges
GRANT SELECT ON hr.employees TO hr_user;
GRANT INSERT ON hr.employees TO hr_user;
GRANT UPDATE ON hr.employees TO hr_user;
GRANT DELETE ON hr.employees TO hr_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON hr.employees TO hr_user;  -- Multiple
GRANT ALL ON hr.employees TO hr_user;     -- All object privileges
GRANT EXECUTE ON hr.my_package TO hr_user;

-- Grant to all users
GRANT SELECT ON hr.employees TO PUBLIC;

-- Grant with GRANT OPTION (can re-grant object privilege)
GRANT SELECT ON hr.employees TO hr_user WITH GRANT OPTION;

-- Column-level privilege
GRANT UPDATE (salary, commission_pct) ON hr.employees TO hr_user;

-- Revoke object privilege
REVOKE SELECT ON hr.employees FROM hr_user;
REVOKE ALL ON hr.employees FROM hr_user;
```

### Roles:

```sql
-- Create a role
CREATE ROLE app_read_role;
CREATE ROLE app_write_role IDENTIFIED BY "rolepass";  -- Password-protected role

-- Grant privileges to role
GRANT SELECT ON hr.employees TO app_read_role;
GRANT SELECT ON hr.departments TO app_read_role;
GRANT INSERT, UPDATE ON hr.employees TO app_write_role;

-- Grant role to user
GRANT app_read_role TO hr_user;
GRANT app_write_role TO hr_user;

-- Grant role to role
GRANT app_read_role TO app_write_role;

-- Set default roles
ALTER USER hr_user DEFAULT ROLE ALL;
ALTER USER hr_user DEFAULT ROLE app_read_role;
ALTER USER hr_user DEFAULT ROLE ALL EXCEPT app_write_role;
ALTER USER hr_user DEFAULT ROLE NONE;

-- Enable role in session
SET ROLE app_write_role;
SET ROLE app_write_role IDENTIFIED BY "rolepass";
SET ROLE ALL;
SET ROLE NONE;

-- Predefined Oracle roles
GRANT CONNECT TO hr_user;              -- CREATE SESSION only
GRANT RESOURCE TO hr_user;            -- Create tables, procs, sequences, etc.
GRANT DBA TO hr_user;                 -- Almost unlimited power (DANGEROUS!)
GRANT SELECT_CATALOG_ROLE TO hr_user; -- Read access to data dictionary
GRANT EXECUTE_CATALOG_ROLE TO hr_user;-- Execute catalog packages

-- View roles granted to user
SELECT granted_role, admin_option, default_role
FROM dba_role_privs WHERE grantee = 'HR_USER';

-- View all privileges through roles
SELECT privilege FROM dba_sys_privs
WHERE grantee IN (SELECT granted_role FROM dba_role_privs WHERE grantee = 'HR_USER');
```

## 17.3 Profiles

Profiles enforce **password and resource policies**:

```sql
-- Create a profile
CREATE PROFILE app_profile LIMIT
  -- Password settings
  FAILED_LOGIN_ATTEMPTS    5        -- Lock after 5 failed logins
  PASSWORD_LIFE_TIME       90       -- Days before expiration
  PASSWORD_REUSE_TIME      365      -- Days before reuse allowed
  PASSWORD_REUSE_MAX       10       -- Min password changes before reuse
  PASSWORD_LOCK_TIME       1/24     -- Lock for 1 hour (1/24 of a day)
  PASSWORD_GRACE_TIME      7        -- Days of grace period after expiration
  PASSWORD_VERIFY_FUNCTION ora12c_strong_verify_function  -- Complexity function
  -- Resource settings (requires RESOURCE_LIMIT = TRUE)
  SESSIONS_PER_USER        5        -- Max concurrent sessions
  CPU_PER_SESSION          UNLIMITED
  CPU_PER_CALL             3000      -- CPU units per SQL call
  CONNECT_TIME             60        -- Max session duration (minutes)
  IDLE_TIME                30        -- Max idle time (minutes)
  LOGICAL_READS_PER_SESSION UNLIMITED
  LOGICAL_READS_PER_CALL   UNLIMITED;

-- Assign profile to user
ALTER USER hr_user PROFILE app_profile;

-- Enable resource limits
ALTER SYSTEM SET resource_limit = TRUE;

-- View profile details
SELECT profile, resource_name, limit 
FROM dba_profiles WHERE profile = 'APP_PROFILE'
ORDER BY resource_name;

-- Default profile
SELECT profile, resource_name, limit 
FROM dba_profiles WHERE profile = 'DEFAULT';
```

## 17.4 Fine-Grained Auditing (FGA)

```sql
-- Standard auditing
AUDIT SELECT TABLE BY hr_user;
AUDIT INSERT, UPDATE, DELETE ON hr.employees BY ACCESS;
-- BY ACCESS: one entry per statement
-- BY SESSION: one entry per session

-- View audit trail
SELECT os_username, username, action_name, obj_name, timestamp
FROM dba_audit_trail
WHERE username = 'HR_USER'
ORDER BY timestamp DESC;

-- Fine-grained auditing (row-level)
BEGIN
  DBMS_FGA.ADD_POLICY(
    object_schema  => 'HR',
    object_name    => 'EMPLOYEES',
    policy_name    => 'AUDIT_SALARY_ACCESS',
    audit_condition=> 'SALARY > 100000',    -- Only audit high-salary rows
    audit_column   => 'SALARY',             -- When salary column accessed
    handler_schema => NULL,
    handler_module => NULL,
    enable         => TRUE,
    statement_types=> 'SELECT, UPDATE'
  );
END;
/

-- View FGA audit trail
SELECT db_user, object_name, policy_name, sql_text, timestamp
FROM dba_fga_audit_trail
ORDER BY timestamp DESC;

-- Remove FGA policy
EXECUTE DBMS_FGA.DROP_POLICY('HR', 'EMPLOYEES', 'AUDIT_SALARY_ACCESS');
```

## 17.5 Virtual Private Database (VPD)

Row-level security — automatically appends WHERE clauses to queries:

```sql
-- Create policy function
CREATE OR REPLACE FUNCTION emp_access_policy(
  schema_name IN VARCHAR2,
  object_name IN VARCHAR2
) RETURN VARCHAR2 IS
BEGIN
  IF SYS_CONTEXT('USERENV','SESSION_USER') = 'HR' THEN
    RETURN NULL;  -- HR sees all rows
  ELSE
    -- Others only see their own department
    RETURN 'department_id = SYS_CONTEXT(''HR_CTX'',''DEPT_ID'')';
  END IF;
END;
/

-- Attach policy to table
BEGIN
  DBMS_RLS.ADD_POLICY(
    object_schema  => 'HR',
    object_name    => 'EMPLOYEES',
    policy_name    => 'EMP_SECURITY',
    function_schema=> 'HR',
    policy_function=> 'EMP_ACCESS_POLICY',
    statement_types=> 'SELECT,INSERT,UPDATE,DELETE'
  );
END;
/
```

---

# 18. MANAGING DB CONNECTIVITY

## 18.1 Oracle Net Architecture

```
CLIENT                              SERVER
────────────────────────────────────────────────────────
┌─────────────┐                    ┌──────────────────────┐
│ Application │                    │  Oracle Instance     │
│             │                    │                      │
│ Oracle Net  │◄──── Network ─────►│ Oracle Net (server)  │
│ (SQL*Net)   │    (TCP/IP)        │                      │
└─────────────┘                    └──────────────────────┘
      │                                     │
  tnsnames.ora                        listener.ora
  sqlnet.ora                          Oracle Listener
```

## 18.2 Oracle Listener

The **listener** is a server-side process that:
- Listens for incoming connection requests (default port 1521)
- Passes connections to the appropriate database service
- Can be administered with `lsnrctl`

```
listener.ora (server-side):
────────────────────────────
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = db-server)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME = ORCL)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
    )
  )
```

```bash
# Listener commands
lsnrctl start                  # Start listener
lsnrctl stop                   # Stop listener
lsnrctl status                 # Check status
lsnrctl reload                 # Reload without stopping (re-reads listener.ora)
lsnrctl services               # Detailed service info
lsnrctl set log_status on      # Enable logging
lsnrctl set password           # Set admin password

# Start named listener
lsnrctl start LISTENER2
```

## 18.3 tnsnames.ora

Client-side name resolution file:

```
# tnsnames.ora (client-side)
ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = db-server)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl)
    )
  )

# With failover (two listeners)
ORCL_HA =
  (DESCRIPTION =
    (FAILOVER = ON)
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL=TCP)(HOST=db1)(PORT=1521))
      (ADDRESS = (PROTOCOL=TCP)(HOST=db2)(PORT=1521))
    )
    (CONNECT_DATA = (SERVICE_NAME = orcl))
  )

# JDBC easy connect (no tnsnames needed)
# jdbc:oracle:thin:@//hostname:1521/service_name
# sqlplus user/pass@//hostname:1521/service_name
```

## 18.4 sqlnet.ora

Configuration file for Oracle Net behavior:

```
# sqlnet.ora
NAMES.DIRECTORY_PATH = (TNSNAMES, EZCONNECT, LDAP)  # Resolution order

# Encryption
SQLNET.ENCRYPTION_SERVER = REQUIRED
SQLNET.ENCRYPTION_TYPES_SERVER = (AES256, RC4_256)
SQLNET.CRYPTO_CHECKSUM_SERVER = REQUIRED

# Dead connection detection
SQLNET.EXPIRE_TIME = 10    # Check dead connections every 10 minutes

# Tracing
TRACE_LEVEL_SERVER = 16    # Detailed tracing (0=off, 4=user, 10=admin, 16=support)
TRACE_DIRECTORY_SERVER = /u01/trace
```

## 18.5 Database Services

Services are logical names for connecting to the database:

```sql
-- View existing services
SELECT name, network_name, pdb, enabled FROM dba_services;
SELECT name, inst_id FROM gv$services;  -- RAC: shows which instance each service is on

-- Create a service (via DBMS_SERVICE)
BEGIN
  DBMS_SERVICE.CREATE_SERVICE(
    service_name       => 'SALES_APP',
    network_name       => 'SALES_APP.example.com',
    failover_method    => 'BASIC',
    failover_type      => 'SELECT',
    failover_retries   => 10,
    failover_delay     => 5
  );
END;
/

-- Start/stop service
EXECUTE DBMS_SERVICE.START_SERVICE('SALES_APP');
EXECUTE DBMS_SERVICE.STOP_SERVICE('SALES_APP');

-- Delete service
EXECUTE DBMS_SERVICE.DELETE_SERVICE('SALES_APP');

-- SRVCTL (for RAC/CRS managed databases)
srvctl add service -d orcl -s sales_app -r "orcl1,orcl2" -a "orcl3"
srvctl start service -d orcl -s sales_app
srvctl stop  service -d orcl -s sales_app
srvctl status service -d orcl
```

---

# 19. SHARED SERVER ARCHITECTURE

## 19.1 Dedicated vs. Shared Server

| Feature | Dedicated Server | Shared Server |
|---|---|---|
| Server process | One per connection | Many users share few processes |
| Memory per user | High (own PGA stack) | Lower (UGA in SGA large pool) |
| Best for | Long-running queries, OLTP | Many short transactions |
| Session state | In PGA | In UGA (SGA Large Pool) |
| Performance | Faster for complex SQL | Better for connection-heavy OLTP |

```
DEDICATED SERVER:
Client → Listener → Server Process (dedicated) → Instance

SHARED SERVER (MTS - Multi-Threaded Server):
Client → Listener → Dispatcher → Request Queue → Shared Servers
                               ← Response Queue ←
```

## 19.2 Configuring Shared Server

```sql
-- Configure shared server
ALTER SYSTEM SET shared_servers = 5;         -- Initial shared servers
ALTER SYSTEM SET max_shared_servers = 20;    -- Maximum
ALTER SYSTEM SET shared_server_sessions = 200; -- Max shared server sessions
ALTER SYSTEM SET dispatchers = '(PROTOCOL=TCP)(DISPATCHERS=3)';
ALTER SYSTEM SET max_dispatchers = 10;

-- Large pool is important for shared server (stores UGA)
ALTER SYSTEM SET large_pool_size = 256M;

-- Connect forcing dedicated server (override default)
-- In tnsnames.ora:
ORCL_DED = (DESCRIPTION = (ADDRESS = ...) (CONNECT_DATA = (SERVER = DEDICATED) ...))

-- Or in connect string:
sqlplus user/pass@//hostname:1521/service_name:dedicated

-- Check shared server status
SELECT name, status, messages, bytes, breaks FROM v$dispatcher;
SELECT status, messages, bytes FROM v$shared_server;
SELECT requests, responses FROM v$queue WHERE type = 'COMMON';
SELECT requests, responses FROM v$queue WHERE type = 'DISPATCHER';
```

---

# 20. FAST RECOVERY AREA & REDO LOG MANAGEMENT

## 20.1 Fast Recovery Area (FRA)

The FRA (also called **Flash Recovery Area**) is a disk location where Oracle automatically stores recovery-related files:
- Online redo logs (optional)
- Archive log files
- Control file copies/autobackups
- Datafile copies (RMAN)
- RMAN backup sets
- Flashback logs

```sql
-- Configure FRA
ALTER SYSTEM SET db_recovery_file_dest = '/u01/fra' SCOPE = BOTH;
ALTER SYSTEM SET db_recovery_file_dest_size = 50G SCOPE = BOTH;  -- Must set this!

-- Check FRA usage
SELECT * FROM v$recovery_file_dest;
SELECT file_type, percent_space_used, percent_space_reclaimable, number_of_files
FROM v$flash_recovery_area_usage;

-- Check total FRA space and used
SELECT name, space_limit/1024/1024/1024 AS size_gb,
       space_used/1024/1024/1024       AS used_gb,
       space_reclaimable/1024/1024/1024 AS reclaimable_gb,
       number_of_files
FROM v$recovery_file_dest;

-- Delete obsolete files in FRA
RMAN> DELETE OBSOLETE;
RMAN> DELETE EXPIRED BACKUP;

-- Critical Alert: ORA-19809
-- db_recovery_file_dest_size reached!
-- Fix: Increase size, delete obsolete, add more space
ALTER SYSTEM SET db_recovery_file_dest_size = 100G;
```

## 20.2 Redo Log Management

### Redo Log Sizing:

```sql
-- Check log switch frequency (should be every 20-30 minutes ideally)
SELECT sequence#, first_time, next_time,
       ROUND((next_time - first_time) * 24 * 60) AS duration_minutes
FROM v$log_history
ORDER BY sequence# DESC
FETCH FIRST 20 ROWS ONLY;

-- If logs switch too frequently: increase log file size
ALTER DATABASE ADD LOGFILE GROUP 4 
  ('/u01/oradata/orcl/redo04a.log', '/u02/oradata/orcl/redo04b.log') 
  SIZE 500M;  -- Add larger groups

-- Drop smaller groups after switching past them
ALTER SYSTEM SWITCH LOGFILE;  -- Switch a few times to clear old groups
ALTER DATABASE DROP LOGFILE GROUP 1;  -- Drop old small groups
```

### Log Switch Operations:

```sql
-- Manual log switch (useful before backup/monitoring)
ALTER SYSTEM SWITCH LOGFILE;

-- Archive current log and switch
ALTER SYSTEM ARCHIVE LOG CURRENT;

-- Check current log sequence
SELECT thread#, sequence#, status FROM v$log WHERE status = 'CURRENT';

-- Log switch history (useful for sizing)
SELECT to_char(first_time,'YYYY-MM-DD HH24') AS hour, COUNT(*) AS switches
FROM v$log_history
WHERE first_time > SYSDATE - 7
GROUP BY to_char(first_time,'YYYY-MM-DD HH24')
ORDER BY 1;
```

### Redo Log Issues:

```
Problem: Log switch wait / "log file switch completion" wait
Cause: ARCn can't archive fast enough, or LGWR can't keep up
Fix:
1. Increase redo log size (bigger groups = less frequent switches)
2. Add more redo log groups
3. Speed up archive destination I/O
4. Add more ARCn processes: ALTER SYSTEM SET log_archive_max_processes = 4;

Problem: "log file switch (archiving needed)" wait
Cause: All redo logs are full and ARCn hasn't archived the oldest ones
Fix: Free up space in archive dest, add archive destinations

Problem: Corrupted redo log member
Fix: 
  ALTER DATABASE CLEAR LOGFILE GROUP 2;  -- if inactive
  -- If current/active: try SHUTDOWN ABORT then startup and recover
```

---

# 21. ORACLE BACKUPS

## 21.1 Backup Strategy Overview

```
ORACLE BACKUP TYPES:
──────────────────────────────────────────────────────────────────
Physical Backups
├── RMAN (Recovery Manager) — Oracle's recommended backup tool
│   ├── Full Backup         — All used blocks in a datafile
│   ├── Incremental Level 0 — Baseline (same as full for incrementals)
│   ├── Incremental Level 1 — Changed blocks since Level 0 or last Level 1
│   │   ├── Differential   — Changes since last backup of any level
│   │   └── Cumulative     — Changes since last Level 0
│   ├── Image Copy          — Block-for-block copy (not compressed, instantly usable)
│   └── Backup Set          — RMAN compressed format (proprietary)
│
└── OS-Level Backup (rarely used for Oracle)
    └── Cold backup (shutdown required for consistency)

Logical Backups
├── Data Pump Export (expdp) — Export schema/table data
└── Original Export (exp)   — Legacy, avoid for new work
```

## 21.2 RMAN Basics

```bash
# Connect to RMAN
rman target /                          # Local connection
rman target sys/pass@orcl              # Remote connection
rman target / catalog rman/rman@catdb  # With catalog database
rman target / nocatalog                # Without catalog (uses control file)
```

```sql
-- RMAN commands
RMAN> SHOW ALL;                        -- Show all RMAN config
RMAN> SHOW RETENTION POLICY;
RMAN> SHOW BACKUP;
RMAN> LIST BACKUP;                     -- List all backups
RMAN> LIST BACKUP SUMMARY;
RMAN> LIST COPY;                       -- List image copies
RMAN> REPORT NEED BACKUP;             -- Files needing backup
RMAN> REPORT OBSOLETE;                -- Backups beyond retention
RMAN> REPORT UNRECOVERABLE;           -- Unrecoverable operations

-- Full database backup
RMAN> BACKUP DATABASE;
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;     -- Include archive logs
RMAN> BACKUP DATABASE FORMAT '/u01/backup/db_%d_%T_%s_%p.bkp' TAG 'FULL_BKP';

-- Incremental backup strategy
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE;          -- Sunday: Level 0
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE; -- Daily: Cumulative Level 1
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;          -- Or: Differential Level 1

-- Tablespace backup
RMAN> BACKUP TABLESPACE app_data;
RMAN> BACKUP TABLESPACE system, sysaux, users;

-- Datafile backup
RMAN> BACKUP DATAFILE 4;
RMAN> BACKUP DATAFILE '/u01/oradata/orcl/app_data01.dbf';

-- Archive log backup
RMAN> BACKUP ARCHIVELOG ALL;
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT;  -- Backup then delete
RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 100 UNTIL SEQUENCE 200 THREAD 1;

-- Control file backup
RMAN> BACKUP CURRENT CONTROLFILE;

-- Image copy (instant restore capability)
RMAN> COPY DATAFILE 4 TO '/u01/backup/app_data01.dbf';
-- Or:
RMAN> BACKUP AS COPY DATAFILE 4 FORMAT '/u01/backup/app_data01.dbf';
```

## 21.3 RMAN Configuration

```sql
-- Configure retention policy
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;  -- Keep 7 days of backups
RMAN> CONFIGURE RETENTION POLICY TO REDUNDANCY 2;               -- Keep 2 copies
RMAN> CONFIGURE RETENTION POLICY TO NONE;                       -- No policy

-- Configure default backup destination
RMAN> CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '/u01/backup/%d/%T/%U';

-- Configure parallelism
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 4 BACKUP TYPE TO BACKUPSET;

-- Configure compression
RMAN> CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';  -- LOW, MEDIUM, HIGH, BASIC
RMAN> CONFIGURE BACKUP OPTIMIZATION ON;

-- Control file autobackup
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/u01/backup/cf_%F';

-- Channel configuration (tape via SBT)
RMAN> CONFIGURE CHANNEL DEVICE TYPE SBT PARMS 'ENV=(NB_ORA_SERV=media_server)';
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO SBT;

-- Encryption
RMAN> SET ENCRYPTION ON;
RMAN> SET ENCRYPTION ALGORITHM 'AES256';
RMAN> CONFIGURE ENCRYPTION FOR DATABASE ON;
```

## 21.4 RMAN Backup Script Example

```sql
-- Typical RMAN weekly backup script
RUN {
  ALLOCATE CHANNEL ch1 DEVICE TYPE DISK 
    FORMAT '/u01/backup/%d_%T_%s_%p.bkp' MAXPIECESIZE 2G;
  ALLOCATE CHANNEL ch2 DEVICE TYPE DISK 
    FORMAT '/u01/backup/%d_%T_%s_%p.bkp' MAXPIECESIZE 2G;
  
  BACKUP INCREMENTAL LEVEL 0 DATABASE
    TAG 'WEEKLY_LEVEL0'
    INCLUDE CURRENT CONTROLFILE
    PLUS ARCHIVELOG ALL DELETE INPUT;
  
  BACKUP SPFILE;
  
  DELETE OBSOLETE;
  
  RELEASE CHANNEL ch1;
  RELEASE CHANNEL ch2;
}
```

## 21.5 Data Pump Export/Import (Logical Backup)

```bash
# Data Pump Export (full database)
expdp system/pass FULL=Y DIRECTORY=DATA_PUMP_DIR DUMPFILE=fulldb_%date%.dmp LOGFILE=fulldb.log

# Schema export
expdp system/pass SCHEMAS=HR,SALES DIRECTORY=DATA_PUMP_DIR DUMPFILE=schema_hr_sales.dmp

# Table export
expdp hr/pass TABLES=EMPLOYEES,DEPARTMENTS DIRECTORY=DATA_PUMP_DIR DUMPFILE=tables.dmp

# Export with compression and parallel
expdp system/pass FULL=Y DIRECTORY=DATA_PUMP_DIR DUMPFILE=full_%U.dmp 
  COMPRESSION=ALL PARALLEL=4 FILESIZE=2G

# Data Pump Import
impdp system/pass FULL=Y DIRECTORY=DATA_PUMP_DIR DUMPFILE=fulldb.dmp LOGFILE=imp.log

# Import with remap schema and tablespace
impdp system/pass DIRECTORY=DATA_PUMP_DIR DUMPFILE=schema_hr.dmp \
  REMAP_SCHEMA=HR:HR_NEW REMAP_TABLESPACE=APP_DATA:NEW_DATA

# Import table to different name
impdp hr/pass DIRECTORY=DATA_PUMP_DIR DUMPFILE=tables.dmp \
  TABLES=EMPLOYEES REMAP_TABLE=EMPLOYEES:EMPLOYEES_BACKUP
```

```sql
-- Create directory object
CREATE OR REPLACE DIRECTORY data_pump_dir AS '/u01/dpump';
GRANT READ, WRITE ON DIRECTORY data_pump_dir TO hr;

-- Monitor Data Pump jobs
SELECT owner_name, job_name, state, degree, 
       attached_sessions, datapump_sessions
FROM dba_datapump_jobs WHERE state != 'NOT RUNNING';

-- Attach to running job
-- $ expdp system/pass ATTACH=job_name
```

---

# 22. DATABASE RECOVERY

## 22.1 Recovery Concepts

### Recovery Types:

| Type | Scenario | Data Loss |
|---|---|---|
| **Instance Recovery** | Crash/ABORT shutdown | None (uses redo) |
| **Crash Recovery** | Single instance failure | None |
| **Media Recovery** | Datafile corruption/loss | Depends on backup |
| **Complete Recovery** | All logs available | No data loss |
| **Incomplete (Point-in-Time)** | Logs missing or intentional rollback | Loses changes after target |

### Recovery Components:

```
RECOVERY PROCESS:
                    
 BACKUP           ARCHIVE LOGS          ONLINE REDO         "NOW"
   │                    │                    │                 │
   ▼                    ▼                    ▼                 │
[RESTORE] ──── ROLL FORWARD (apply changes) ──────────────────┤
                                                               │
                                          [ROLL BACK uncommitted]
                                                               │
                                                        DATABASE OPEN
```

## 22.2 Instance/Crash Recovery

Automatic — performed by SMON at startup:

```
1. Roll Forward (REDO):
   - Apply all changes from redo log (committed AND uncommitted)
   - Database reaches state at time of crash
   
2. Roll Back (UNDO):
   - Roll back all uncommitted transactions using undo data
   - Database reaches consistent state

Time depends on:
- Amount of redo since last checkpoint
- FAST_START_MTTR_TARGET (target mean time to recover)
```

```sql
-- Tune instance recovery time
ALTER SYSTEM SET fast_start_mttr_target = 60;  -- Target: 60 seconds recovery
-- Oracle adjusts checkpoint frequency automatically

-- Check estimated recovery time
SELECT estimated_mttr, target_mttr FROM v$instance_recovery;
```

## 22.3 RMAN Restore and Recovery

```sql
-- Complete recovery (all logs available, no data loss)
RMAN> SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN;

-- Recover a single tablespace (without bringing down whole DB)
RMAN> SQL 'ALTER TABLESPACE app_data OFFLINE IMMEDIATE';
RMAN> RESTORE TABLESPACE app_data;
RMAN> RECOVER TABLESPACE app_data;
RMAN> SQL 'ALTER TABLESPACE app_data ONLINE';

-- Recover a single datafile
RMAN> SQL 'ALTER DATABASE DATAFILE 7 OFFLINE';
RMAN> RESTORE DATAFILE 7;
RMAN> RECOVER DATAFILE 7;
RMAN> SQL 'ALTER DATABASE DATAFILE 7 ONLINE';

-- Incomplete (Point-in-Time) recovery
RMAN> SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE UNTIL TIME "TO_DATE('2026-04-01 10:00:00','YYYY-MM-DD HH24:MI:SS')";
RMAN> RECOVER DATABASE UNTIL TIME "TO_DATE('2026-04-01 10:00:00','YYYY-MM-DD HH24:MI:SS')";
RMAN> ALTER DATABASE OPEN RESETLOGS;  -- MUST use RESETLOGS after incomplete recovery!

-- Incomplete recovery by SCN
RMAN> RESTORE DATABASE UNTIL SCN 9876543;
RMAN> RECOVER DATABASE UNTIL SCN 9876543;
RMAN> ALTER DATABASE OPEN RESETLOGS;

-- Recover control file
RMAN> SHUTDOWN ABORT;
RMAN> STARTUP NOMOUNT;
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;
-- or: RESTORE CONTROLFILE FROM '/u01/backup/cf_xxxxx.bkp';
RMAN> ALTER DATABASE MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

## 22.4 Flashback Technology

```sql
-- Flashback Query (see data as it was)
SELECT * FROM employees AS OF TIMESTAMP (SYSDATE - INTERVAL '1' HOUR)
WHERE employee_id = 101;

SELECT * FROM employees AS OF SCN 9876543;

-- Flashback Table (undo table changes)
ALTER TABLE employees ENABLE ROW MOVEMENT;
FLASHBACK TABLE employees TO TIMESTAMP (SYSDATE - 1/24);
FLASHBACK TABLE employees TO SCN 9876543;
FLASHBACK TABLE employees TO BEFORE DROP;  -- Recover from Recycle Bin

-- Flashback Database (whole database point-in-time)
-- Requires: FLASHBACK ON, FRA configured
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
FLASHBACK DATABASE TO TIMESTAMP (SYSDATE - 1);
FLASHBACK DATABASE TO SCN 9876543;
ALTER DATABASE OPEN RESETLOGS;

-- Or flashback and check, then decide
FLASHBACK DATABASE TO TIMESTAMP ...;
ALTER DATABASE OPEN READ ONLY;    -- Check if this is the right point
-- If good:
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE OPEN RESETLOGS;   -- Commit the flashback

-- Enable Flashback Database
ALTER DATABASE FLASHBACK ON;
ALTER DATABASE FLASHBACK OFF;

-- Check flashback status
SELECT flashback_on, oldest_flashback_scn, oldest_flashback_time FROM v$database;
SELECT * FROM v$flashback_database_log;

-- Recycle Bin (Flashback Drop)
SELECT object_name, original_name, type, droptime FROM recyclebin;
FLASHBACK TABLE "BIN$xxxxx" TO BEFORE DROP;
FLASHBACK TABLE old_name TO BEFORE DROP RENAME TO new_name;
PURGE TABLE employees;         -- Remove from recycle bin
PURGE RECYCLEBIN;              -- Empty my recycle bin
PURGE DBA_RECYCLEBIN;          -- Empty all recycle bins (DBA)
PURGE TABLESPACE app_data;     -- Purge recycle bin for a tablespace
```

## 22.5 LogMiner

LogMiner reads redo/archive logs to reconstruct DML history:

```sql
-- Add log files
EXECUTE DBMS_LOGMNR.ADD_LOGFILE('/u01/arch/arch_1_100.arc', DBMS_LOGMNR.NEW);
EXECUTE DBMS_LOGMNR.ADD_LOGFILE('/u01/arch/arch_1_101.arc', DBMS_LOGMNR.ADDFILE);

-- Or automatically add logs for a time range
EXECUTE DBMS_LOGMNR.START_LOGMNR(
  STARTTIME => TO_DATE('2026-04-01 09:00:00','YYYY-MM-DD HH24:MI:SS'),
  ENDTIME   => TO_DATE('2026-04-01 10:00:00','YYYY-MM-DD HH24:MI:SS'),
  OPTIONS   => DBMS_LOGMNR.DICT_FROM_ONLINE_CATALOG + 
               DBMS_LOGMNR.CONTINUOUS_MINE
);

-- Query LogMiner output
SELECT timestamp, username, sql_redo, sql_undo, table_name, operation
FROM v$logmnr_contents
WHERE seg_name = 'EMPLOYEES'
AND operation IN ('INSERT','UPDATE','DELETE')
ORDER BY timestamp;

-- Find specific change
SELECT sql_undo FROM v$logmnr_contents
WHERE seg_owner = 'HR' AND seg_name = 'EMPLOYEES'
AND operation = 'DELETE'
AND timestamp BETWEEN TIMESTAMP '2026-04-01 09:30:00' AND TIMESTAMP '2026-04-01 09:35:00';

-- End LogMiner session
EXECUTE DBMS_LOGMNR.END_LOGMNR;
```

---

# 23. STORAGE AND SPACE MANAGEMENT

## 23.1 ASM — Automatic Storage Management

ASM is Oracle's built-in volume manager and file system for Oracle database files.

### ASM Architecture:

```
┌───────────────────────────────────────────────────────────┐
│                    ASM INSTANCE                           │
│         (separate instance, no user data)                │
│                                                           │
│  ┌──────────────────────────────────────────────────┐     │
│  │                DISK GROUPS                       │     │
│  │                                                  │     │
│  │  +DATA diskgroup     +FRA diskgroup              │     │
│  │  /dev/sdb  /dev/sdc  /dev/sdd  /dev/sde          │     │
│  │  (mirrored, striped automatically)               │     │
│  └──────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────┘
                              │
                    Database Instance uses
                    +DATA/orcl/datafile/
```

```sql
-- ASM SQL*Plus commands (connect to ASM instance)
-- sqlplus / as sysasm

-- View disk groups
SELECT name, state, type, total_mb, free_mb,
       ROUND(free_mb/total_mb*100,1) AS pct_free
FROM v$asm_diskgroup;

-- View ASM disks
SELECT group_number, disk_number, path, name, header_status, state,
       total_mb, free_mb
FROM v$asm_disk
ORDER BY group_number, disk_number;

-- Create disk group
CREATE DISKGROUP data_dg EXTERNAL REDUNDANCY
  DISK '/dev/sdb1', '/dev/sdc1', '/dev/sdd1';

CREATE DISKGROUP data_dg NORMAL REDUNDANCY     -- 2-way mirror
  FAILGROUP fg1 DISK '/dev/sdb1'
  FAILGROUP fg2 DISK '/dev/sdc1';

CREATE DISKGROUP data_dg HIGH REDUNDANCY       -- 3-way mirror
  FAILGROUP fg1 DISK '/dev/sdb1'
  FAILGROUP fg2 DISK '/dev/sdc1'
  FAILGROUP fg3 DISK '/dev/sdd1';

-- Add disk to disk group
ALTER DISKGROUP data_dg ADD DISK '/dev/sde1';

-- Drop disk from disk group (data rebalances automatically)
ALTER DISKGROUP data_dg DROP DISK DATA_0003;

-- Rebalance (manual, or automatic after add/drop)
ALTER DISKGROUP data_dg REBALANCE POWER 8;  -- Power 0-11, higher=faster

-- Check rebalance progress
SELECT group_number, operation, state, power, actual, sofar, est_work
FROM v$asm_operation;

-- ASM files
SELECT name, group_number, file_number, bytes/1024/1024 AS size_mb
FROM v$asm_file
WHERE group_number = 1;

-- ASMCMD utility (OS-level ASM file navigation)
-- asmcmd
-- ASMCMD> lsdg               -- List disk groups
-- ASMCMD> ls +DATA/orcl/     -- List files in ASM
-- ASMCMD> cp +DATA/orcl/datafile/users.dbf /u01/backup/  -- Copy file out
-- ASMCMD> md_backup /tmp/asmbackup.xml  -- Backup ASM metadata
```

## 23.2 Space Usage Monitoring and Management

```sql
-- Comprehensive tablespace space report
SELECT t.tablespace_name,
       t.status,
       t.contents,
       DECODE(t.bigfile,'YES','BIG','SMALL') AS ts_type,
       NVL(d.total_mb,0)   AS total_mb,
       NVL(f.free_mb,0)    AS free_mb,
       NVL(d.total_mb,0) - NVL(f.free_mb,0) AS used_mb,
       ROUND(100 * (NVL(d.total_mb,0) - NVL(f.free_mb,0)) / NULLIF(NVL(d.total_mb,0),0), 1) AS pct_used,
       NVL(d.max_mb,0)     AS max_mb     -- max if fully autoextended
FROM dba_tablespaces t
LEFT JOIN (SELECT tablespace_name,
                  SUM(bytes)/1048576 AS total_mb,
                  SUM(CASE WHEN autoextensible='YES' THEN maxbytes ELSE bytes END)/1048576 AS max_mb
           FROM dba_data_files GROUP BY tablespace_name) d
       ON t.tablespace_name = d.tablespace_name
LEFT JOIN (SELECT tablespace_name, SUM(bytes)/1048576 AS free_mb 
           FROM dba_free_space GROUP BY tablespace_name) f
       ON t.tablespace_name = f.tablespace_name
WHERE t.contents != 'TEMPORARY'
ORDER BY pct_used DESC NULLS LAST;

-- Top space consumers
SELECT owner, segment_name, segment_type, tablespace_name,
       ROUND(bytes/1024/1024/1024, 2) AS size_gb
FROM dba_segments
ORDER BY bytes DESC
FETCH FIRST 20 ROWS ONLY;

-- Tables that could benefit from compression
SELECT owner, table_name, num_rows, 
       ROUND(blocks * 8192 / 1024 / 1024, 1) AS approx_size_mb,
       compression, compress_for
FROM dba_tables
WHERE owner NOT IN ('SYS','SYSTEM','OUTLN','XDB')
AND num_rows > 100000
AND compression = 'DISABLED'
ORDER BY blocks DESC;

-- Space used by schema
SELECT owner, 
       ROUND(SUM(bytes)/1024/1024/1024, 2) AS size_gb,
       COUNT(*) AS num_segments
FROM dba_segments
WHERE owner NOT IN ('SYS','SYSTEM','OUTLN','XDB','ORACLE_OCM')
GROUP BY owner
ORDER BY SUM(bytes) DESC;

-- Find tablespace with autoextend that's close to MAXSIZE
SELECT d.file_name, d.tablespace_name,
       ROUND(d.bytes/1024/1024, 1) AS current_mb,
       ROUND(d.maxbytes/1024/1024, 1) AS max_mb,
       ROUND(100 * d.bytes / NULLIF(d.maxbytes, 0), 1) AS pct_of_max
FROM dba_data_files d
WHERE d.autoextensible = 'YES'
ORDER BY pct_of_max DESC NULLS LAST;
```

## 23.3 Automatic Space Management Features

### Automatic Segment Advisor:

```sql
-- Scheduled by Oracle via Automated Maintenance Tasks
-- Check advisor findings
SELECT attr1 AS object_name,
       attr2 AS object_type,
       attr3 AS partition_name,
       message,
       more_info
FROM dba_advisor_findings af, dba_advisor_objects ao
WHERE af.object_id = ao.object_id
AND af.task_name = ao.task_name
AND af.task_name LIKE 'SYS_AUTO_SPCADV%'
ORDER BY af.task_name DESC, af.id;
```

### Resumable Space Allocation:

```sql
-- Enable resumable for long-running DML
ALTER SESSION ENABLE RESUMABLE;
ALTER SESSION ENABLE RESUMABLE TIMEOUT 3600 NAME 'Long DML Job';

-- When space error occurs, session suspends (doesn't fail)
-- DBA gets an alert and can add space
-- Operation resumes automatically after space is added

-- Check suspended sessions
SELECT session_id, status, name, timeout, sql_text, error_msg
FROM dba_resumable;
```

### DBMS_SPACE Package:

```sql
-- Analyze free space in a segment
DECLARE
  v_total    NUMBER;
  v_used     NUMBER;
  v_expired  NUMBER;
  v_unexpired NUMBER;
BEGIN
  DBMS_SPACE.SPACE_USAGE(
    segment_owner    => 'HR',
    segment_name     => 'EMPLOYEES',
    segment_type     => 'TABLE',
    total_blocks     => v_total,
    total_bytes      => v_used,
    unused_blocks    => v_expired,
    unused_bytes     => v_unexpired,
    last_used_extent_file_id => NULL,
    last_used_extent_block_id => NULL,
    last_used_block  => NULL
  );
  DBMS_OUTPUT.PUT_LINE('Total blocks: ' || v_total);
END;
/

-- Predict object growth
SELECT DBMS_SPACE.OBJECT_GROWTH_TREND_FUTURE(
  'HR', 'EMPLOYEES', 'TABLE', NULL, 365
) AS growth_bytes_next_year
FROM dual;
```

---

## QUICK REFERENCE CARD

### Key V$ Views Summary

| View | Purpose |
|---|---|
| `V$INSTANCE` | Instance name, status, startup time |
| `V$DATABASE` | DB name, mode, SCN, archive mode |
| `V$SESSION` | All current sessions |
| `V$SQL` | SQL statements in shared pool |
| `V$SQLAREA` | Aggregated SQL statistics |
| `V$PROCESS` | OS processes |
| `V$LOCK` | Current locks |
| `V$LATCH` | Latch statistics |
| `V$BUFFER_POOL_STATISTICS` | Buffer cache hit ratios |
| `V$SGASTAT` | SGA component breakdown |
| `V$PGASTAT` | PGA statistics |
| `V$LOG` | Redo log group status |
| `V$LOGFILE` | Redo log member paths |
| `V$ARCHIVED_LOG` | Archive log history |
| `V$CONTROLFILE` | Control file paths |
| `V$DATAFILE` | Data file status/SCN |
| `V$PARAMETER` | Initialization parameters |
| `V$TABLESPACE` | Tablespace metadata |
| `V$UNDO_STAT` | Undo statistics |
| `V$RECOVERY_FILE_DEST` | FRA usage |
| `V$RMAN_BACKUP_JOB_DETAILS` | RMAN job history |
| `V$DIAG_ALERT_EXT` | Alert log contents |

### Key DBA Views Summary

| View | Purpose |
|---|---|
| `DBA_USERS` | All database users |
| `DBA_TABLES` | All tables |
| `DBA_INDEXES` | All indexes |
| `DBA_SEGMENTS` | All segments + sizes |
| `DBA_EXTENTS` | All extents |
| `DBA_FREE_SPACE` | Free space in tablespaces |
| `DBA_TABLESPACES` | Tablespace definitions |
| `DBA_DATA_FILES` | Data file details |
| `DBA_TEMP_FILES` | Temp file details |
| `DBA_SYS_PRIVS` | System privileges |
| `DBA_TAB_PRIVS` | Object privileges |
| `DBA_ROLE_PRIVS` | Role grants |
| `DBA_PROFILES` | Profile definitions |
| `DBA_AUDIT_TRAIL` | Standard audit records |
| `DBA_FGA_AUDIT_TRAIL` | Fine-grained audit records |
| `DBA_JOBS` | Job queue |
| `DBA_SCHEDULER_JOBS` | Scheduler jobs |
| `DBA_UNDO_EXTENTS` | Undo extent status |
| `DBA_HIST_*` | AWR historical data |

---

*End of Oracle Database Comprehensive Notes*
*Generated: April 2026*
