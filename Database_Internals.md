# Database Internals: From 0 to 100 — The Complete Interview Guide

> A deep-dive reference covering what, why, how, tradeoffs, internal workings, scaling strategies, and failure handling for every major database system.

---

## Table of Contents

1. [Foundational Concepts](#1-foundational-concepts)
2. [Storage Engine Internals](#2-storage-engine-internals)
3. [Indexing Deep Dive](#3-indexing-deep-dive)
4. [Concurrency Control & MVCC](#4-concurrency-control--mvcc)
5. [Transaction Management & ACID](#5-transaction-management--acid)
6. [Write-Ahead Logging (WAL)](#6-write-ahead-logging-wal)
7. [Query Processing Pipeline](#7-query-processing-pipeline)
8. [Replication](#8-replication)
9. [Sharding & Partitioning](#9-sharding--partitioning)
10. [Database-Specific Internals](#10-database-specific-internals)
11. [When to Use Which Database](#11-when-to-use-which-database)
12. [Scaling Strategies](#12-scaling-strategies)
13. [Failure Scenarios & Recovery](#13-failure-scenarios--recovery)
14. [CAP Theorem & Distributed Tradeoffs](#14-cap-theorem--distributed-tradeoffs)
15. [Interview Question Bank](#15-interview-question-bank)

---

## 1. Foundational Concepts

### How a Database Works End-to-End

```
Client Query
    │
    ▼
┌──────────────────────────────┐
│     SQL INTERFACE             │
│  Parser → Planner → Optimizer │
│         → Executor            │
└──────────────────────────────┘
    │
    ▼
┌──────────────────────────────┐
│     STORAGE ENGINE            │
│  Buffer Pool ↔ Txn Manager   │
│  Index Mgr  ↔ Lock Manager   │
└──────────────────────────────┘
    │
    ▼
┌──────────────────────────────┐
│          DISK                 │
│  Data Files | Index Files     │
│       | WAL Files             │
└──────────────────────────────┘
```

### Data Models

| Model | Structure | Example DBs | Best For |
|-------|-----------|-------------|----------|
| Relational | Tables with rows/columns | PostgreSQL, MySQL | Structured data, complex queries, ACID |
| Document | JSON/BSON documents | MongoDB, CouchDB | Flexible schemas, rapid iteration |
| Wide-Column | Column families | Cassandra, HBase | Write-heavy, time-series, IoT |
| Key-Value | Simple key→value pairs | Redis, DynamoDB | Caching, sessions, low-latency lookups |
| Graph | Nodes + Edges | Neo4j, Amazon Neptune | Relationships, social networks |
| Time-Series | Timestamp-indexed | TimescaleDB, InfluxDB | Metrics, monitoring, IoT |

### Row Store vs Column Store

**Row Store (OLTP):** Stores entire rows contiguously. Great for transactional workloads where you read/write full records (e.g., banking, e-commerce). Examples: PostgreSQL, MySQL.

**Column Store (OLAP):** Stores each column contiguously. Ideal for analytics scanning few columns across millions of rows. Enables better compression since similar values sit together. Examples: ClickHouse, Amazon Redshift, Apache Parquet.

**Tradeoff:** Row stores win on single-record CRUD; column stores win on aggregate queries over large datasets. You can't optimize for both simultaneously — this is why many architectures separate OLTP and OLAP (the "HTAP" problem).

---

## 2. Storage Engine Internals

### B-Tree / B+ Tree

**What:** A self-balancing tree where each node can have multiple children. B+ trees (the variant used in most databases) store all data in leaf nodes, with internal nodes holding only keys and pointers. Leaf nodes are linked together for efficient range scans.

**Why:** Optimized for read-heavy workloads and disk-based storage. The high branching factor means a tree holding millions of keys is only 3-4 levels deep, so a point query requires at most 3-4 disk reads.

**How it works internally:**
1. Data is organized in fixed-size **pages** (typically 4KB-16KB)
2. Internal nodes contain keys and pointers to child pages
3. Leaf nodes contain actual data (or pointers to data) and are doubly-linked
4. Insertions that overflow a page trigger a **page split** — the page divides into two and the parent gets a new key
5. Deletions that underflow trigger a **merge** with a sibling page

**Key properties:**
- All leaves are at the same depth (perfectly balanced)
- Point query: O(log_B N) disk reads
- Range query: O(log_B N + K/B) where K = result count
- In-place updates (mutable structure)

**Tradeoff:** Excellent read performance but writes require random I/O (updating pages in place). Write amplification is proportional to page size B in worst case (rewriting an entire page for a single-byte change).

### LSM Tree (Log-Structured Merge Tree)

**What:** A write-optimized data structure that batches all writes in memory before flushing them to disk as sorted, immutable files (SSTables).

**Why:** Eliminates random writes entirely. All disk I/O is sequential, which is dramatically faster on both HDDs and SSDs.

**How it works internally:**
1. Writes go to an in-memory buffer called the **memtable** (typically a skip list or red-black tree)
2. When the memtable reaches a threshold, it's flushed to disk as an immutable **SSTable** (Sorted String Table)
3. Multiple SSTables accumulate on disk across **levels** (L0, L1, L2...)
4. A background **compaction** process merges overlapping SSTables, removing duplicates and tombstones
5. Reads must check the memtable + all SSTable levels, aided by **Bloom filters** to skip SSTables that don't contain the key

**Compaction strategies:**
- **Size-Tiered (STCS):** Merge SSTables of similar size. Better write throughput, worse space amplification.
- **Leveled (LCS):** Each level is 10x larger than the previous. Better read performance and space usage, more write amplification.
- **FIFO:** Simply drop oldest SSTables. Used for time-series data with TTL.

**Tradeoff:** Great write throughput but reads are more expensive (must check multiple levels). The three-way tradeoff: a data structure can optimize at most two of read amplification, write amplification, and space amplification.

### B-Tree vs LSM-Tree Comparison

| Dimension | B-Tree | LSM-Tree |
|-----------|--------|----------|
| Write pattern | Random I/O (in-place update) | Sequential I/O (append-only) |
| Read pattern | 1 disk seek per query (warm cache) | Multiple levels to check |
| Write amplification | Higher (page rewrites) | Lower (sequential flushes) |
| Read amplification | Lower | Higher (multiple levels) |
| Space amplification | Lower (one copy of data) | Higher (duplicates across levels) |
| Best for | Read-heavy OLTP | Write-heavy workloads |
| Used by | PostgreSQL, MySQL/InnoDB, SQL Server | Cassandra, RocksDB, LevelDB, HBase |

---

## 3. Indexing Deep Dive

### B+ Tree Index (Default in most RDBMS)

Supports: equality (`=`), range (`<`, `>`, `BETWEEN`), prefix `LIKE`, `ORDER BY`, `MIN/MAX`.

### Hash Index

Supports: only exact equality (`=`). O(1) average lookup. Cannot do range scans. Used internally by hash joins and in-memory stores like Redis.

### Composite Index

An index on multiple columns `(a, b, c)`. Follows the **leftmost prefix rule** — it can satisfy queries on `(a)`, `(a, b)`, or `(a, b, c)` but NOT on `(b)` or `(c)` alone.

### Covering Index

An index that contains all columns needed by a query. The DB can satisfy the query entirely from the index without touching the table — called an **index-only scan**. Huge performance win.

### Specialized Index Types

| Type | What | Used For | Database |
|------|------|----------|----------|
| GIN (Generalized Inverted) | Maps values → rows (inverted index) | Full-text search, JSONB, arrays | PostgreSQL |
| GiST (Generalized Search Tree) | Balanced tree for complex data | Geometric, range, nearest-neighbor | PostgreSQL |
| BRIN (Block Range INdex) | Stores min/max per block range | Naturally ordered data (timestamps) | PostgreSQL |
| R-Tree | Spatial index for rectangles | Geospatial queries | MySQL, SQLite |
| Bloom Filter | Probabilistic membership test | LSM reads (skip unnecessary SSTables) | Cassandra, RocksDB |

### Indexing Tradeoffs

- Every index speeds up reads but **slows down writes** (index must be updated on every INSERT/UPDATE/DELETE)
- Indexes consume **disk space** and **memory** (buffer pool)
- Too many indexes → write amplification, longer vacuum/compaction
- Too few indexes → table scans, slow queries
- **Rule of thumb:** Index columns used in WHERE, JOIN, ORDER BY. Drop unused indexes.

---

## 4. Concurrency Control & MVCC

### Why Concurrency Control Matters

Without it, concurrent transactions produce anomalies: dirty reads, non-repeatable reads, phantom reads, lost updates, and write skew.

### Locking-Based Approaches

**Two-Phase Locking (2PL):**
- Growing phase: acquire locks, never release
- Shrinking phase: release locks, never acquire
- Guarantees serializability but prone to deadlocks
- Pessimistic — assumes conflicts will happen

**Deadlock handling:** Either detect (wait-for graph → abort youngest txn) or prevent (wound-wait / wait-die schemes).

### MVCC (Multi-Version Concurrency Control)

**What:** Instead of locking, maintain multiple versions of each row. Readers see a consistent snapshot without blocking writers.

**Why:** Readers never block writers, writers never block readers. Dramatically improves concurrency.

**How it works (PostgreSQL model):**
1. Each row version has `xmin` (creating txn ID) and `xmax` (deleting txn ID)
2. UPDATE creates a **new** row version (new `xmin`) and marks the old version with `xmax`
3. A transaction sees only versions where `xmin < current_txn_id` and `xmax` is either empty or from an uncommitted/aborted txn
4. Dead row versions accumulate → **VACUUM** process reclaims space

**How it works (MySQL/InnoDB model):**
1. InnoDB stores the latest version in-place in the clustered index
2. Old versions are kept in the **undo log** (rollback segment)
3. To read an old version, InnoDB follows the undo chain backward
4. **Purge thread** cleans up undo logs when no active txn needs them

**Tradeoff:** MVCC avoids lock contention but introduces: version bloat (PostgreSQL), undo log overhead (InnoDB), and the need for background cleanup (VACUUM / purge).

### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|------------|---------------------|--------------|-------------|
| READ UNCOMMITTED | Possible | Possible | Possible | Fastest |
| READ COMMITTED | No | Possible | Possible | Good (PG default) |
| REPEATABLE READ | No | No | Possible* | Moderate (MySQL default) |
| SERIALIZABLE | No | No | No | Slowest |

*MySQL's REPEATABLE READ actually prevents phantoms via **gap locks** and **next-key locks**, making it closer to serializable in practice.

**Interview gotcha:** PostgreSQL's SERIALIZABLE uses **Serializable Snapshot Isolation (SSI)** — an optimistic approach that detects conflicts at commit time rather than acquiring locks upfront. This means it has higher throughput than traditional 2PL-based serializable but may abort more transactions.

---

## 5. Transaction Management & ACID

### ACID Deep Dive

**Atomicity** — All or nothing. Implemented via WAL + undo logs. If a transaction fails mid-way, the undo log rolls back partial changes.

**Consistency** — The database moves from one valid state to another. Enforced by constraints (PK, FK, CHECK, UNIQUE) and triggers.

**Isolation** — Concurrent transactions don't interfere. Implemented via MVCC or locking (see above).

**Durability** — Once committed, data survives crashes. Implemented via WAL — the commit record hits disk before the client gets acknowledgment.

### Distributed Transactions

**Two-Phase Commit (2PC):**
1. **Prepare phase:** Coordinator asks all participants "Can you commit?" Each participant writes prepare record to its WAL and responds yes/no.
2. **Commit phase:** If all said yes, coordinator writes commit record and tells all participants to commit. If any said no, everyone aborts.

**Problem:** If the coordinator crashes after prepare but before commit, participants are **blocked** — they hold locks and can't decide independently. This is the classic "blocking" problem of 2PC.

**Three-Phase Commit (3PC):** Adds a pre-commit phase to avoid blocking, but is impractical in real networks with partitions.

**Saga Pattern:** Break the distributed transaction into a sequence of local transactions. Each has a compensating action for rollback. Used in microservices (e.g., book flight → book hotel → charge card; if charge fails → cancel hotel → cancel flight).

**Tradeoff:** 2PC gives strong consistency but is slow and blocking. Sagas give availability but only eventual consistency and complex compensation logic.

---

## 6. Write-Ahead Logging (WAL)

### What & Why

Before any data page is modified on disk, the change is first written to a sequential, append-only log file. This ensures durability (committed changes survive crashes) and atomicity (incomplete changes can be undone).

### How WAL Works

1. Transaction modifies pages in the **buffer pool** (memory)
2. Each modification generates a **WAL record** (redo log entry)
3. WAL records are written to disk **before** the dirty data pages are flushed
4. On commit, the WAL is `fsync`'d to disk — only then does the client get "commit OK"
5. Dirty pages are flushed lazily in the background by the **checkpoint** process

### Crash Recovery (ARIES Protocol)

1. **Analysis phase:** Scan WAL forward from last checkpoint to identify dirty pages and active transactions at crash time
2. **Redo phase:** Replay all WAL records from the earliest dirty page LSN forward — brings the database to the exact state at crash time
3. **Undo phase:** Roll back any transactions that were active (uncommitted) at crash time using undo records

### Checkpointing

Periodically, the DB writes all dirty pages to disk and records a checkpoint in the WAL. This limits how far back recovery must replay. Without checkpoints, recovery would replay the entire WAL history.

**Tradeoff:** Frequent checkpoints → faster recovery but more I/O during normal operation. Infrequent checkpoints → less overhead but longer recovery time.

---

## 7. Query Processing Pipeline

### The Journey of a SQL Query

```
SQL String
   │
   ▼
[1] PARSER        → Syntax check, produce AST (Abstract Syntax Tree)
   │
   ▼
[2] ANALYZER      → Semantic check (tables/columns exist?), resolve names
   │
   ▼
[3] REWRITER      → Apply views, rules, query rewrites
   │
   ▼
[4] OPTIMIZER     → Generate candidate plans, estimate costs, pick cheapest
   │
   ▼
[5] EXECUTOR      → Execute the physical plan, return results
```

### Query Optimizer Internals

The optimizer estimates the **cost** of each plan based on: table statistics (row counts, distinct values, histograms), I/O cost (sequential vs random reads), CPU cost, and memory available for sorts/hashes.

**Join algorithms:**

| Algorithm | How | Best When |
|-----------|-----|-----------|
| Nested Loop | For each row in outer, scan inner | Small outer table, indexed inner |
| Hash Join | Build hash table on smaller input, probe with larger | Equi-joins, no useful index |
| Sort-Merge Join | Sort both inputs, merge | Both inputs already sorted or large equi-joins |

**Interview tip:** Use `EXPLAIN ANALYZE` to see the actual execution plan. Look for sequential scans on large tables (usually means a missing index), nested loop joins with large outer tables, and sort operations spilling to disk.

---

## 8. Replication

### Why Replicate?

1. **High availability** — failover if primary dies
2. **Read scaling** — distribute read traffic across replicas
3. **Geographic distribution** — serve reads from nearest datacenter

### Replication Topologies

**Single-Leader (Master-Slave):**
- All writes go to one leader; replicas consume the leader's WAL/binlog
- Simple, consistent writes
- Replication lag means stale reads on replicas
- Used by: PostgreSQL, MySQL, MongoDB (replica sets)

**Multi-Leader (Master-Master):**
- Multiple nodes accept writes; changes are propagated bi-directionally
- Better write availability across regions
- Conflict resolution is hard (last-write-wins, merge, custom logic)
- Used by: MySQL Group Replication, CockroachDB

**Leaderless (Peer-to-Peer):**
- Any node accepts reads and writes
- Uses quorum: W + R > N for consistency (W = write replicas, R = read replicas, N = total)
- No single point of failure
- Used by: Cassandra, DynamoDB, Riak

### Sync vs Async Replication

| | Synchronous | Asynchronous |
|---|---|---|
| Commit waits for | Replica acknowledgment | Only local disk |
| Data loss risk | Zero (if replica is up) | Possible (replication lag) |
| Latency | Higher | Lower |
| Availability | Lower (replica failure blocks writes) | Higher |

**Semi-synchronous:** Wait for at least one replica before acknowledging. Balances durability and performance. Used by MySQL's semisync replication.

---

## 9. Sharding & Partitioning

### Terminology

- **Partitioning:** Splitting a table into pieces (can be on same or different machines)
- **Sharding:** Distributing partitions across multiple machines (horizontal scaling)

### Partitioning Strategies

**Range-Based:** Partition by value ranges (e.g., dates, IDs 1-1M, 1M-2M). Good for range queries. Risk of **hotspots** if access patterns are skewed (e.g., recent dates get all traffic).

**Hash-Based:** Apply hash function to shard key, distribute by hash value. Even distribution, no hotspots. But range queries must scatter to all shards.

**Consistent Hashing:** Nodes are placed on a hash ring. Data maps to the nearest node clockwise. Adding/removing a node only reassigns keys from the adjacent node. Used by Cassandra and DynamoDB.

**Directory-Based:** A lookup table maps keys to shards. Maximum flexibility but the directory is a single point of failure and bottleneck.

### Shard Key Selection (Critical!)

A good shard key has: high cardinality (many distinct values), even distribution, and is present in most queries (to avoid scatter-gather). A bad shard key leads to hot shards, cross-shard queries, and painful resharding.

**Example:** For a multi-tenant SaaS app, `tenant_id` is often ideal — queries are tenant-scoped, distribution is usually even, and data isolation is natural.

### Resharding

Adding/removing shards requires redistributing data. Strategies: consistent hashing (minimal redistribution), virtual nodes / vnodes (finer granularity), or online schema migration tools (e.g., Vitess for MySQL, pg_partman for PostgreSQL).

---

## 10. Database-Specific Internals

### PostgreSQL

**Storage engine:** Heap-based storage with B-tree indexes. No clustered index by default — the heap stores rows in insertion order.

**MVCC implementation:** Append-only with `xmin`/`xmax` transaction IDs on each tuple. Old tuple versions live in the same heap pages alongside current versions. VACUUM reclaims dead tuples.

**WAL:** 16MB WAL segment files by default. Supports streaming replication (continuous WAL shipping to replicas). WAL is used for both crash recovery and replication.

**Buffer pool:** Shared buffers (configurable, typically 25% of RAM). Uses a **clock-sweep** eviction algorithm (variant of LRU).

**Query optimizer:** Cost-based optimizer with support for genetic query optimization (GEQO) for complex joins (12+ tables). Maintains per-column statistics with histograms.

**Unique features:** Rich type system (JSONB, arrays, hstore, range types), GIN/GiST/BRIN indexes, table inheritance, CTEs, window functions, SSI for true serializable isolation.

**Weaknesses:** Write amplification from full-tuple MVCC (every UPDATE writes a complete new row version). VACUUM overhead can be significant on write-heavy tables. No native built-in sharding (rely on Citus or manual partitioning).

**Scaling strategy:** Vertical scaling + read replicas + Citus extension for sharding + table partitioning (native declarative partitioning since v10).

### MySQL (InnoDB)

**Storage engine:** InnoDB is the default. Uses a **clustered index** (primary key defines physical row order). Secondary indexes store the PK value as a pointer back to the clustered index.

**MVCC implementation:** In-place updates in the clustered index + **undo logs** for old versions. Reads reconstruct old versions by following the undo chain. Purge thread cleans up old undo records.

**WAL:** Called the **redo log** (ib_logfile). Fixed-size circular buffer. Also has a **binary log (binlog)** used for replication and point-in-time recovery (separate from redo log).

**Buffer pool:** InnoDB buffer pool (typically 70-80% of RAM). Uses a **modified LRU** with a young/old sublist to prevent full-table scans from evicting hot pages.

**Locking:** Uses **next-key locks** (lock the index record + gap before it) under REPEATABLE READ. This prevents phantom reads but can cause unexpected lock contention.

**Unique features:** Extremely mature replication ecosystem (async, semi-sync, group replication), wide hosting/tooling support, Vitess for horizontal sharding.

**Weaknesses:** The undo log can bloat if long-running transactions exist. Clustered index means secondary index lookups require a double lookup (secondary → PK → data). Online DDL is improving but still painful for large tables.

**Scaling strategy:** Read replicas + ProxySQL/MySQL Router for load balancing + Vitess or MySQL Cluster for sharding.

### MongoDB

**Storage engine:** **WiredTiger** (default since 3.2). Uses a B-tree variant with document-level compression (snappy or zstd). Supports both B-tree and LSM configurations internally.

**Data model:** Documents (BSON format) stored in collections. No enforced schema by default (schema validation available since 3.6).

**Concurrency:** Document-level locking (WiredTiger). MVCC with snapshot isolation for multi-document transactions (since v4.0).

**Replication:** **Replica sets** — one primary + multiple secondaries. Automatic failover via election (Raft-like consensus). Oplog (operation log) streams changes to secondaries.

**Sharding:** Built-in. A **mongos** router directs queries to the right shard. Config servers store metadata. Supports hash-based and range-based shard keys. Chunks auto-balance across shards.

**Unique features:** Flexible schema, aggregation pipeline, change streams, Atlas (managed service), full-text search (Atlas Search / Lucene-based).

**Weaknesses:** Multi-document transactions have performance overhead and limitations. Memory-mapped I/O means it benefits from large RAM. Shard key choice is immutable (changing requires data migration). No JOINs (use `$lookup` which is essentially a left outer join but not as performant).

**Scaling strategy:** Replica sets for HA → sharded clusters for write/storage scaling. Atlas auto-scaling for managed deployments.

### Redis

**Storage engine:** Entirely **in-memory**. Data structures stored in a hash table. Single-threaded event loop for commands (I/O multiplexing with epoll/kqueue). Since Redis 6.0, I/O threads handle network read/write but command execution remains single-threaded.

**Data structures:** Strings, Lists (linked list / ziplist), Sets, Sorted Sets (skip list + hash table), Hashes, Streams, Bitmaps, HyperLogLog, Geospatial indexes.

**Persistence:**
- **RDB (snapshotting):** Periodic full dump to disk. Fast recovery, possible data loss between snapshots.
- **AOF (Append Only File):** Logs every write command. Can be replayed for recovery. Configurable fsync (every second / every write / never).
- **RDB + AOF:** Use both for best durability + fast restart.

**Replication:** Async master-replica. Replicas receive a full RDB snapshot on first sync, then continuous command stream.

**Clustering:** Redis Cluster — hash slots (0-16383) distributed across nodes. Each key hashes to a slot. Automatic resharding and failover. Limitation: multi-key operations must target the same slot (use hash tags `{user:123}`).

**Weaknesses:** Dataset must fit in memory (expensive at scale). Single-threaded command execution can bottleneck on CPU-intensive operations. Persistence options all have tradeoffs (data loss vs performance). No built-in complex querying.

**Scaling strategy:** Redis Cluster for horizontal scaling + read replicas per shard. Use as a caching layer in front of a primary database, not as sole data store for critical data.

### Cassandra

**Storage engine:** **LSM-tree** based. Writes go to an in-memory **memtable** → flushed to immutable **SSTables** on disk. Background compaction merges SSTables.

**Architecture:** **Peer-to-peer** ring with consistent hashing. No master node — every node is equal. Data is automatically distributed and replicated across the ring.

**Data model:** Wide-column store. Tables have a **partition key** (determines which node stores the data) and optional **clustering columns** (determines sort order within a partition).

**Consistency:** **Tunable consistency** — choose consistency level per query: ONE, QUORUM, ALL, LOCAL_QUORUM, etc. Formula: W + R > N for strong consistency (e.g., QUORUM reads + QUORUM writes with RF=3).

**Compaction strategies:** Size-Tiered (default, write-optimized), Leveled (read-optimized), Time-Window (time-series data).

**Unique features:** Linear horizontal scalability, multi-datacenter replication built-in, tunable consistency, no single point of failure, TTL on data.

**Weaknesses:** No JOINs, no subqueries, limited aggregation. Must model data around queries (denormalization required). Tombstones from deletes can cause read performance issues. Repair process is complex. No ACID transactions across partitions (lightweight transactions use Paxos but are slow).

**Scaling strategy:** Add nodes to the ring — data automatically rebalances. Cassandra scales linearly: 2x nodes ≈ 2x throughput.

### Elasticsearch

**Storage engine:** Built on Apache Lucene. Uses an **inverted index** (term → document IDs) for full-text search. Segments are immutable (LSM-like); new documents go to an in-memory buffer, periodically flushed to new segments. Background merge combines segments.

**Architecture:** Distributed cluster with shards and replicas. An index is split into primary shards (distributed across nodes). Each primary shard has replica shards on different nodes.

**Unique features:** Near-real-time full-text search, aggregations, nested/parent-child documents, relevance scoring (BM25), geo queries.

**Weaknesses:** Not a primary data store (no ACID transactions). Updates are expensive (delete + re-index). Schema changes on large indices are painful. Resource-heavy (RAM, CPU, disk I/O).

**When to use:** Search functionality, log analytics (ELK stack), metrics aggregation, e-commerce product search.

---

## 11. When to Use Which Database

| Scenario | Recommended DB | Why |
|----------|---------------|-----|
| General-purpose web app with complex queries | **PostgreSQL** | Rich SQL, JSONB support, strong consistency, extensions |
| High-traffic web app, need simple scaling | **MySQL** | Mature replication, Vitess sharding, huge ecosystem |
| Flexible schema, rapid prototyping | **MongoDB** | Schema-less documents, easy horizontal scaling |
| Caching, sessions, leaderboards | **Redis** | In-memory speed, rich data structures |
| Write-heavy IoT / time-series at massive scale | **Cassandra** | LSM-tree writes, linear scalability, multi-DC |
| Full-text search, log analytics | **Elasticsearch** | Inverted index, aggregations, near-real-time |
| Social network / recommendation engine | **Neo4j** | Graph traversals are O(1) per hop vs O(n) JOINs |
| Global-scale with strong consistency | **CockroachDB / Spanner** | Distributed SQL, serializable, geo-partitioning |
| Serverless, managed NoSQL | **DynamoDB** | Zero ops, auto-scaling, single-digit ms latency |
| Analytics / data warehouse | **ClickHouse / BigQuery** | Columnar storage, vectorized execution |

### Decision Framework

```
Need ACID transactions?
├── Yes → Need horizontal write scaling?
│         ├── Yes → CockroachDB / Spanner / YugabyteDB
│         └── No  → PostgreSQL / MySQL
└── No  → What's the access pattern?
          ├── Key-value lookups → Redis / DynamoDB
          ├── Wide-column / time-series → Cassandra / ScyllaDB
          ├── Document / flexible schema → MongoDB
          ├── Full-text search → Elasticsearch
          └── Graph traversals → Neo4j
```

---

## 12. Scaling Strategies

### Vertical Scaling (Scale Up)

Add more CPU, RAM, faster disks to a single machine. Simple but has a ceiling. Good first step before introducing distributed complexity.

### Horizontal Scaling (Scale Out)

Add more machines. Requires partitioning/sharding. Introduces complexity: distributed transactions, rebalancing, cross-shard queries.

### Read Scaling

Add **read replicas**. Route reads to replicas, writes to primary. Works well when read:write ratio is high (common — most apps are 90%+ reads). Watch for replication lag causing stale reads.

### Write Scaling

**Sharding** is the primary strategy. Partition data across multiple write nodes. Each shard handles a subset of the data independently. Choose shard key carefully.

### Caching Layers

```
Application
    │
    ▼
┌────────────┐    Cache Miss    ┌────────────┐
│   Redis /   │ ──────────────► │  Database   │
│  Memcached  │ ◄────────────── │             │
└────────────┘    Fill Cache     └────────────┘
```

**Patterns:**
- **Cache-Aside:** App checks cache first; on miss, reads DB and populates cache. Most common.
- **Write-Through:** App writes to cache; cache synchronously writes to DB. Consistent but slower writes.
- **Write-Behind (Write-Back):** App writes to cache; cache asynchronously flushes to DB. Fast writes but risk of data loss.
- **Read-Through:** Cache itself fetches from DB on miss. Simplifies application code.

**Cache invalidation** is one of the hardest problems in CS. Strategies: TTL-based expiry, event-driven invalidation (publish change events), versioned keys.

### Connection Pooling

Databases have a limit on concurrent connections. Use a connection pooler (PgBouncer for PostgreSQL, ProxySQL for MySQL) to multiplex thousands of application connections onto a smaller number of DB connections.

---

## 13. Failure Scenarios & Recovery

### Node Failure

**Single-node DB:** Crash recovery via WAL replay (ARIES protocol). DB restarts, replays redo log, undoes incomplete transactions. Typically seconds to minutes of downtime.

**Replicated DB:** Automatic failover to a replica. The replica is promoted to primary. Client connections are redirected (via DNS update, VIP, or proxy).

**Data loss risk:** With async replication, the replica may be behind. Committed transactions on the old primary that weren't replicated are lost. Mitigate with synchronous or semi-synchronous replication.

### Split Brain

When a network partition causes two nodes to both think they're the primary. Both accept writes → data divergence.

**Prevention:** Use a **quorum** — a node can only be primary if it has support from a majority of nodes. Fencing mechanisms (STONITH — "Shoot The Other Node In The Head") forcibly shut down the old primary.

### Replication Lag

Replicas fall behind the primary. Reads from replicas return stale data.

**Solutions:**
- **Read-your-writes consistency:** After a write, route that user's reads to the primary (or wait for the replica to catch up)
- **Monotonic reads:** Ensure a user always reads from the same replica (session affinity)
- **Causal consistency:** Track dependencies between operations

### Disk Failure

**Single disk:** Data loss if no replication. Mitigate with RAID, replicas, or cloud storage (EBS, persistent disks).

**Entire node storage failure:** Rebuild from replica or backup. Point-in-time recovery: restore last full backup + replay WAL/binlog up to desired timestamp.

### Corruption

Silent data corruption (bit rot) detected by checksums on pages (PostgreSQL has data checksums, InnoDB has page checksums). Recovery: restore from a replica or backup.

### Hot Shard

One shard receives disproportionate traffic. Causes: celebrity user, viral content, bad shard key.

**Solutions:** Re-shard with better key, add salt/jitter to keys, split the hot shard, or add a caching layer in front of the hot shard.

### Long-Running Transactions

Hold locks or prevent VACUUM/undo cleanup. Cause bloated tables (PostgreSQL), undo log growth (InnoDB), and lock contention.

**Solutions:** Set `statement_timeout` and `idle_in_transaction_session_timeout`. Monitor and kill long transactions. Design for short transactions.

### Backup Strategies

| Type | What | Recovery Speed | Storage Cost | Frequency |
|------|------|---------------|--------------|-----------|
| Full Backup | Entire DB snapshot | Fastest restore | Highest | Weekly |
| Incremental | Changes since last backup (any type) | Slowest (chain) | Lowest | Hourly/Daily |
| Differential | Changes since last full backup | Moderate | Moderate | Daily |
| Continuous WAL Archiving | Stream WAL to object storage | Point-in-time recovery | Low overhead | Continuous |

**The 3-2-1 Rule:** Keep 3 copies of data, on 2 different media types, with 1 offsite copy.

---

## 14. CAP Theorem & Distributed Tradeoffs

### CAP Theorem

In a distributed system, you can guarantee at most **two of three** properties during a network partition:

- **Consistency (C):** Every read returns the most recent write
- **Availability (A):** Every request receives a response (no timeouts)
- **Partition Tolerance (P):** System continues operating despite network partitions

Since network partitions are inevitable in distributed systems, the real choice is between **CP** (consistent but may be unavailable during partition) and **AP** (available but may return stale data during partition).

| Category | Databases | Behavior During Partition |
|----------|-----------|--------------------------|
| CP | PostgreSQL, MySQL, MongoDB, CockroachDB, HBase | Reject writes or become unavailable to maintain consistency |
| AP | Cassandra, DynamoDB, CouchDB, Riak | Accept writes on all available nodes, resolve conflicts later |

### PACELC Theorem (Extension of CAP)

If there is a **P**artition, choose **A**vailability or **C**onsistency. **E**lse (normal operation), choose **L**atency or **C**onsistency.

| DB | During Partition | Normal Operation |
|----|-----------------|------------------|
| PostgreSQL | PC (consistent) | EC (consistent, higher latency) |
| Cassandra | PA (available) | EL (low latency) |
| MongoDB | PC (consistent) | EC (consistent) |
| DynamoDB | PA (available) | EL (low latency) |
| CockroachDB | PC (consistent) | EC (consistent) |

---

## 15. Interview Question Bank

### Beginner Level

1. **What are the ACID properties? Give a real-world violation example for each.**
2. **Explain the difference between a clustered and non-clustered index.** (InnoDB PK is clustered — physical row order. Non-clustered index stores a pointer/PK value.)
3. **What is normalization? When would you denormalize?** (Normalize for write consistency; denormalize for read performance when JOINs are too expensive.)
4. **What's the difference between DELETE, TRUNCATE, and DROP?**
5. **Explain the different types of JOINs with examples.**

### Intermediate Level

6. **How does MVCC work in PostgreSQL vs MySQL?** (PG: append-only tuples with xmin/xmax + VACUUM. MySQL: in-place update + undo log + purge thread.)
7. **What is write amplification and how does it differ between B-trees and LSM-trees?**
8. **Explain the tradeoffs between synchronous and asynchronous replication.**
9. **How would you handle a slow query? Walk through your debugging process.** (Check `EXPLAIN ANALYZE` → look for seq scans, high row estimates, sorts spilling to disk → add indexes, rewrite query, check statistics, increase work_mem.)
10. **What happens during a PostgreSQL checkpoint?** (All dirty pages in shared buffers are flushed to disk. A checkpoint record is written to WAL. This limits recovery replay time.)

### Advanced Level

11. **Design the sharding strategy for a multi-tenant SaaS platform.** (Shard by `tenant_id`. Ensures tenant isolation, even distribution, no cross-shard queries for single-tenant operations. Handle whale tenants with dedicated shards.)
12. **How would you migrate from a single DB to a sharded architecture with zero downtime?** (Dual-write approach: write to both old and new systems. Backfill historical data. Verify consistency. Switch reads. Remove old writes. OR: use a CDC tool like Debezium.)
13. **Explain Serializable Snapshot Isolation (SSI) in PostgreSQL.** (Tracks read/write dependencies between concurrent transactions. If a cycle is detected at commit time, one transaction is aborted. Optimistic — no locking overhead during execution.)
14. **How does Cassandra handle a write when the target node is down?** (Hinted handoff: the coordinator stores the write as a "hint" and replays it when the node comes back. If hints expire, the periodic repair process resolves inconsistencies via Merkle trees.)
15. **Explain the InnoDB change buffer.** (Caches changes to secondary index pages that aren't in the buffer pool. Instead of doing random I/O to read the index page, the change is buffered and merged later when the page is read. Dramatically reduces random I/O for write-heavy workloads.)

### System Design Crossover

16. **Design a URL shortener's database layer.** (Key considerations: high write throughput, simple key-value lookups, TTL support. Options: DynamoDB for managed scaling, Redis for hot URLs + PostgreSQL for persistence.)
17. **Design the database architecture for a real-time chat application.** (Messages: Cassandra (write-heavy, time-ordered partitions by conversation_id). User presence: Redis (in-memory, pub/sub). User profiles: PostgreSQL (relational, ACID). Search: Elasticsearch.)
18. **How would you build a leaderboard that updates in real-time for 10M users?** (Redis Sorted Sets: `ZADD` for updates, `ZREVRANGE` for top-N, `ZRANK` for user's position — all O(log N).)

---

## Appendix: Key Formulas & Numbers to Remember

| Metric | Value |
|--------|-------|
| B-tree point query | O(log_B N) disk reads, typically 3-4 for millions of rows |
| LSM read | Must check memtable + L0 + ... + Ln (Bloom filters skip most) |
| Quorum formula | W + R > N for strong consistency |
| Replication factor | Typically 3 (tolerates 1 node failure with quorum) |
| Buffer pool sizing | PostgreSQL: ~25% RAM. InnoDB: ~70-80% RAM |
| WAL segment size | PostgreSQL: 16MB default |
| Redis max throughput | ~100K-200K ops/sec single-threaded (simple commands) |
| Cassandra write path | Client → Coordinator → Commit Log → Memtable → Ack |
| Page size | PostgreSQL: 8KB. InnoDB: 16KB. MongoDB: varies |

---

## Appendix: Recommended Reading

- *Designing Data-Intensive Applications* — Martin Kleppmann (the bible)
- *Database Internals* — Alex Petrov (deep dive into storage engines & distributed systems)
- *High Performance MySQL* — Schwartz, Zaitsev, Tkachenko
- *PostgreSQL 14 Internals* — Egor Rogov
- *The Art of PostgreSQL* — Dimitri Fontaine
- Use The Index, Luke (use-the-index-luke.com) — SQL indexing and tuning

---

*Last updated: March 2026. Covers PostgreSQL 17, MySQL 8.4, MongoDB 8.0, Redis 7.x, Cassandra 5.0, Elasticsearch 8.x.*
