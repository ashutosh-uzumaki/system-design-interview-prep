# Database Interview: Complete Answer Bank + Per-DB Questions

> Every answer is structured as: **Concept → How it works internally → Tradeoffs → What interviewers want to hear**

---

## Table of Contents

- [Part A — General Question Answers (60 Questions)](#part-a--general-question-answers)
- [Part B — PostgreSQL Questions (25)](#part-b--postgresql-questions)
- [Part C — MySQL / InnoDB Questions (25)](#part-c--mysql--innodb-questions)
- [Part D — MongoDB Questions (20)](#part-d--mongodb-questions)
- [Part E — Redis Questions (20)](#part-e--redis-questions)
- [Part F — Cassandra Questions (20)](#part-f--cassandra-questions)
- [Part G — Elasticsearch Questions (15)](#part-g--elasticsearch-questions)

---

# Part A — General Question Answers

---

## Beginner (1-2 YOE)

### Q1. What are ACID properties? Give a real-world failure example for each.

**Atomicity — All or nothing.**
A bank transfer debits Account A but the system crashes before crediting Account B. Without atomicity, money vanishes. The WAL and undo logs ensure either both operations complete or neither does.

**Consistency — Valid state to valid state.**
An INSERT violates a CHECK constraint (e.g., negative balance). The DB rejects the entire transaction, preventing an invalid state. Enforced by constraints, triggers, and application logic.

**Isolation — Concurrent transactions don't interfere.**
Two users book the last concert ticket simultaneously. Both read "1 seat available," both book it — oversold. Isolation levels (via MVCC or locking) prevent this by controlling what concurrent transactions can see.

**Durability — Committed data survives crashes.**
A transaction commits, the client gets "OK," then the server loses power. Without durability, committed data vanishes. The WAL is fsync'd to disk before acknowledging the commit, so the data can be recovered.

```
Interview tip: Don't just recite definitions.
Give the MECHANISM that implements each property:

  Atomicity   → WAL + Undo Logs
  Consistency → Constraints + Triggers + Application Logic
  Isolation   → MVCC or 2PL (locking)
  Durability  → WAL + fsync
```

---

### Q2. Explain clustered vs non-clustered index.

**Clustered Index:** The table data itself is physically sorted and stored by the index key. There can be only ONE clustered index per table because data can only be sorted one way on disk.

**Non-Clustered Index:** A separate structure containing the indexed columns + a pointer back to the actual row. You can have MANY non-clustered indexes.

```
CLUSTERED (InnoDB Primary Key):
─────────────────────────────
  B+ Tree leaves = ACTUAL ROW DATA (sorted by PK)

  PK=1 → [Alice, 25, NYC]     ← Data lives IN the index
  PK=2 → [Bob, 30, LA]
  PK=3 → [Carol, 28, SF]

NON-CLUSTERED (Secondary Index):
────────────────────────────────
  B+ Tree leaves = INDEXED COLUMN + POINTER TO ROW

  "Alice" → PK=1  ─┐
  "Bob"   → PK=2   ├── Must do SECOND lookup
  "Carol" → PK=3  ─┘   to get full row data

  InnoDB: pointer = Primary Key value (double lookup)
  PostgreSQL: pointer = (page, offset) tuple ID (direct, but HOT updates complicate this)
```

**Key insight for interviews:** InnoDB's secondary index stores the PK value, not a physical address. This means: (a) secondary index lookups require two B-tree traversals, and (b) wide PKs (like UUIDs) bloat every secondary index.

**PostgreSQL difference:** PostgreSQL has NO clustered index by default. The heap stores rows in insertion order. You can run `CLUSTER` to reorder once, but it's not maintained automatically.

---

### Q3. What is normalization (1NF → 3NF)? When would you denormalize?

**1NF:** Every column holds atomic (single) values. No repeating groups.
**2NF:** 1NF + every non-key column depends on the ENTIRE primary key (eliminates partial dependencies).
**3NF:** 2NF + no non-key column depends on another non-key column (eliminates transitive dependencies).

```
BEFORE NORMALIZATION:
┌─────────┬────────┬───────────┬────────────┬──────────────┐
│order_id │ name   │ product   │ prod_price │ customer_city│
├─────────┼────────┼───────────┼────────────┼──────────────┤
│ 1       │ Alice  │ Laptop    │ 999        │ NYC          │
│ 1       │ Alice  │ Mouse     │ 25         │ NYC          │
│ 2       │ Bob    │ Laptop    │ 999        │ LA           │
└─────────┴────────┴───────────┴────────────┴──────────────┘
Problems: customer_city depends on name (transitive),
          prod_price depends on product (partial dependency)

AFTER 3NF:
  Customers(customer_id, name, city)
  Products(product_id, name, price)
  Orders(order_id, customer_id)
  OrderItems(order_id, product_id, quantity)
```

**When to denormalize:**
- Read-heavy workloads where JOINs are expensive
- Reporting/analytics tables (pre-computed aggregates)
- Caching hot data (materialized views)
- NoSQL databases where JOINs don't exist (Cassandra requires denormalization by design)

**Interview tip:** Say "I normalize first for correctness, then selectively denormalize for performance — but I track the cost of maintaining consistency across denormalized copies."

---

### Q4. Difference between DELETE, TRUNCATE, and DROP?

```
┌───────────┬─────────────────┬───────────┬──────────┬───────────┐
│           │ What it does    │ Logged?   │Rollback? │ Speed     │
├───────────┼─────────────────┼───────────┼──────────┼───────────┤
│ DELETE    │ Removes rows    │ Row-by-row│ Yes      │ Slowest   │
│           │ (with WHERE)    │ in WAL    │          │           │
├───────────┼─────────────────┼───────────┼──────────┼───────────┤
│ TRUNCATE  │ Removes ALL rows│ Minimal   │ Depends* │ Fast      │
│           │ Resets identity │ logging   │          │           │
├───────────┼─────────────────┼───────────┼──────────┼───────────┤
│ DROP      │ Removes entire  │ Minimal   │ Depends* │ Fastest   │
│           │ table + schema  │ logging   │          │           │
└───────────┴─────────────────┴───────────┴──────────┴───────────┘

* PostgreSQL: TRUNCATE and DROP are transactional (can rollback!)
* MySQL: TRUNCATE is auto-committed (cannot rollback)
* DELETE fires triggers; TRUNCATE does not
* DELETE can be filtered with WHERE; TRUNCATE cannot
```

---

### Q5. Explain all JOIN types with examples.

```
Tables:
  Users:    [1,Alice], [2,Bob], [3,Carol]
  Orders:   [101,user_id=1], [102,user_id=1], [103,user_id=4]

INNER JOIN: Only matching rows from both tables
  Alice ↔ Order 101
  Alice ↔ Order 102
  (Bob, Carol excluded — no orders. Order 103 excluded — no user_id=4)

LEFT JOIN: All rows from left + matching from right (NULL if no match)
  Alice ↔ Order 101
  Alice ↔ Order 102
  Bob   ↔ NULL          ← Bob has no orders but still appears
  Carol ↔ NULL

RIGHT JOIN: All rows from right + matching from left
  Alice ↔ Order 101
  Alice ↔ Order 102
  NULL  ↔ Order 103     ← Order 103 has no matching user

FULL OUTER JOIN: All rows from both, NULL where no match
  Alice ↔ Order 101
  Alice ↔ Order 102
  Bob   ↔ NULL
  Carol ↔ NULL
  NULL  ↔ Order 103

CROSS JOIN: Every row × every row (Cartesian product)
  3 users × 3 orders = 9 rows

SELF JOIN: Table joined with itself
  Example: Find employees and their managers
  SELECT e.name, m.name AS manager
  FROM employees e JOIN employees m ON e.manager_id = m.id
```

---

### Q6. What is a transaction? What happens if the DB crashes mid-transaction?

A transaction is a group of operations that execute as a single atomic unit. Either ALL operations succeed (COMMIT) or NONE take effect (ROLLBACK).

**On crash mid-transaction:**

```
  BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;  ← executed
    -- CRASH HAPPENS HERE --
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;  ← never reached
  COMMIT;  ← never reached

  Recovery process:
  1. DB restarts
  2. Reads WAL from last checkpoint
  3. REDO phase: replays all WAL records (brings DB to crash state)
  4. UNDO phase: finds this transaction was NOT committed
     → rolls back the debit from account 1
  5. DB is now consistent — as if the transaction never happened
```

---

### Q7. What is an index? Why not index every column?

An index is a data structure (usually B+ tree) that provides fast lookup paths to rows without scanning the entire table. Like a book's index — instead of reading every page, jump directly to the right one.

**Why NOT index everything:**

```
Each index:
  ❌ Slows down writes (every INSERT/UPDATE/DELETE must update ALL indexes)
  ❌ Consumes disk space (index can be 10-30% of table size each)
  ❌ Consumes memory (indexes compete for buffer pool space)
  ❌ Increases VACUUM/maintenance time
  ❌ Optimizer might get confused with too many choices

  Example: Table with 10 indexes
  INSERT 1 row = 1 heap write + 10 index writes = 11 I/O operations!
```

**Rule of thumb:** Index columns used in WHERE, JOIN ON, ORDER BY, and GROUP BY. Use `EXPLAIN` to verify indexes are actually used. Drop unused indexes.

---

### Q8. Explain primary key vs unique key vs foreign key.

```
PRIMARY KEY:
  • Uniquely identifies each row
  • NOT NULL (always required)
  • Only ONE per table
  • In InnoDB: defines physical row order (clustered index)
  • Example: user_id SERIAL PRIMARY KEY

UNIQUE KEY:
  • Enforces uniqueness
  • CAN be NULL (one NULL per unique column in most DBs)
  • Can have MULTIPLE per table
  • Creates a non-clustered index
  • Example: email VARCHAR UNIQUE

FOREIGN KEY:
  • References PRIMARY KEY of another table
  • Enforces referential integrity
  • CAN be NULL (optional relationship)
  • Example: order.user_id REFERENCES users(user_id)
  • ON DELETE CASCADE / SET NULL / RESTRICT
```

---

### Q9. What is a view? What is a materialized view?

```
VIEW (Virtual Table):
─────────────────────
  CREATE VIEW active_users AS
    SELECT * FROM users WHERE status = 'active';

  • NOT stored on disk — just a saved query
  • Executed fresh every time you SELECT from it
  • Always up-to-date
  • No storage overhead
  • Can be slow if underlying query is complex

MATERIALIZED VIEW (Cached Result):
──────────────────────────────────
  CREATE MATERIALIZED VIEW monthly_sales AS
    SELECT month, SUM(amount) FROM orders GROUP BY month;

  • Result IS stored on disk (physical table)
  • Fast reads (pre-computed)
  • Must be REFRESHED to get new data:
      REFRESH MATERIALIZED VIEW monthly_sales;
  • Can become stale between refreshes
  • Uses disk space

  Tradeoff: Views = always fresh, slow
            Mat Views = possibly stale, fast
```

---

### Q10. What is connection pooling and why is it needed?

Each database connection consumes resources: memory (~10MB per connection in PostgreSQL), a process/thread, file descriptors, and kernel state.

```
Problem: 1000 microservice instances × 10 connections each
         = 10,000 connections → DB server runs out of memory!

Solution: Connection Pooler (PgBouncer, ProxySQL, HikariCP)

  1000 App Instances ──▶ PgBouncer ──▶ Database
  (10,000 logical         (50 real       (handles 50
   connections)            connections)    connections easily)

Modes (PgBouncer):
  • Session:     1:1 mapping while client connected (least savings)
  • Transaction: Connection returned to pool after each transaction
  • Statement:   Connection returned after each statement (most aggressive)
```

**Interview tip:** Know that connection pooling is CRITICAL in serverless/microservice architectures where thousands of short-lived instances connect simultaneously.

---

## Intermediate (2-5 YOE)

### Q11. How does MVCC work in PostgreSQL vs MySQL/InnoDB?

```
POSTGRESQL:
───────────
  • UPDATE creates a NEW physical tuple in the same heap page
  • Old tuple marked with xmax = updating transaction's ID
  • Both versions coexist in the heap
  • Visibility determined by xmin/xmax + snapshot
  • Dead tuples accumulate → VACUUM must clean them up

  Pros: No undo log overhead during write
  Cons: Table bloat, VACUUM overhead, full row copied on every update

MYSQL / InnoDB:
───────────────
  • UPDATE modifies the row IN-PLACE in the clustered index
  • Old version pushed to UNDO LOG (rollback segment)
  • roll_ptr in row points to undo chain
  • Readers follow undo chain to reconstruct old versions
  • Purge thread cleans undo logs when no active transaction needs them

  Pros: No heap bloat, no VACUUM needed
  Cons: Undo log can grow if long-running transactions exist,
        reconstructing old versions is CPU-intensive (undo chain traversal)

Key Difference:
  PostgreSQL: "append new, clean old later"  (VACUUM-dependent)
  InnoDB:     "update in place, keep old in undo log" (purge-dependent)
```

---

### Q12. What is write amplification? Compare B-tree vs LSM-tree.

Write amplification = (total bytes written to disk) / (bytes of actual data written by application).

```
B-TREE WRITE AMPLIFICATION:
───────────────────────────
  To insert 1 byte:
  1. Read the 16KB page containing the row
  2. Modify the byte
  3. Write the entire 16KB page back
  → Write amplification = 16KB / 1 byte = 16,384x (worst case)
  
  Also: WAL write (another copy)
  Mitigated by: batching writes to same page in buffer pool

LSM-TREE WRITE AMPLIFICATION:
─────────────────────────────
  For Leveled Compaction with size ratio T=10:
  Each entry is rewritten once per level during compaction
  With L levels: write amplification ≈ T × L ÷ 2
  Example: 10 × 5 ÷ 2 = 25x

  But each INDIVIDUAL write is just an append to memtable (very fast)
  The amplification happens in background compaction

  Summary:
    B-tree:   Immediate write amplification (random I/O per write)
    LSM-tree: Deferred write amplification (background compaction)
              But individual writes are much faster (sequential append)
```

---

### Q13. Explain sync vs async vs semi-sync replication tradeoffs.

```
SYNCHRONOUS:
  Client ──▶ Primary ──▶ Replica ACKs ──▶ Primary ACKs client
  
  ✅ Zero data loss (RPO = 0)
  ❌ Latency = primary write + network RTT + replica write
  ❌ Replica failure blocks ALL writes
  Use when: Financial systems, data you absolutely cannot lose

ASYNCHRONOUS:
  Client ──▶ Primary ACKs client ──▶ (Replica catches up later)
  
  ✅ Lowest latency (primary write only)
  ✅ Replica failure doesn't affect writes
  ❌ Data loss if primary crashes before replica catches up
  ❌ Replication lag → stale reads from replica
  Use when: Most applications, acceptable RPO > 0

SEMI-SYNCHRONOUS:
  Client ──▶ Primary ──▶ At least 1 replica ACKs ──▶ Primary ACKs client
  
  ✅ At most 1 transaction of data loss (the in-flight one)
  ✅ Better latency than full sync
  ⚠️ Degrades to async if all replicas are slow/down
  Use when: Most production databases (good balance)
```

---

### Q14. Walk through debugging a slow query.

```
STEP-BY-STEP PROCESS:
═════════════════════

1. IDENTIFY the slow query
   → pg_stat_statements (PostgreSQL), slow_query_log (MySQL)
   → Look for queries with high total_time or high calls

2. RUN EXPLAIN ANALYZE
   EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;

3. READ THE PLAN bottom-up, look for:

   🔴 RED FLAGS:
   ┌────────────────────────────────┬───────────────────────────┐
   │ What you see                   │ What it means             │
   ├────────────────────────────────┼───────────────────────────┤
   │ Seq Scan on large table        │ Missing index             │
   │ Nested Loop with large outer   │ Need hash/merge join      │
   │ Sort with "Disk: XXkB"         │ work_mem too low          │
   │ Rows estimated vs actual off   │ Stale statistics          │
   │ "Rows Removed by Filter: 99%" │ Index not selective enough │
   │ Bitmap Heap Scan + Recheck     │ Too many matching rows    │
   └────────────────────────────────┴───────────────────────────┘

4. FIX based on findings:
   → Missing index? CREATE INDEX
   → Stale stats? ANALYZE table_name
   → Sort spilling? SET work_mem = '256MB' (per-session)
   → Bad plan? Check for implicit type casts, parameter sniffing
   → Too many rows? Re-think the query / add pagination

5. VERIFY: Run EXPLAIN ANALYZE again to confirm improvement
```

---

### Q15. What happens during PostgreSQL VACUUM? Why is it needed?

```
WHY VACUUM IS NEEDED:
═════════════════════

  PostgreSQL's MVCC creates NEW tuples on UPDATE/DELETE.
  Old tuples become "dead" but still occupy space.

  Without VACUUM:
  ┌───────────────────────────────────────────────────────┐
  │ 1. TABLE BLOAT: Dead tuples waste disk space          │
  │    Table grows to 10x its actual data size            │
  │                                                        │
  │ 2. INDEX BLOAT: Dead index entries slow scans         │
  │                                                        │
  │ 3. TRANSACTION ID WRAPAROUND:                         │
  │    PostgreSQL uses 32-bit transaction IDs              │
  │    After ~2 billion transactions, IDs wrap around      │
  │    Without VACUUM freezing old tuples, ALL data        │
  │    would appear to be "in the future" = invisible!     │
  │    → CATASTROPHIC DATA LOSS (logically)                │
  │    PostgreSQL will FORCE SHUTDOWN to prevent this      │
  └───────────────────────────────────────────────────────┘

WHAT VACUUM DOES:
═════════════════

  Standard VACUUM:
  1. Scan heap pages for dead tuples
  2. Remove dead entries from indexes
  3. Mark heap space as REUSABLE (not returned to OS)
  4. Update Visibility Map (tracks all-visible pages)
  5. Update Free Space Map (tracks available space)
  6. Freeze old transaction IDs (prevents wraparound)
  → NO lock on table (runs concurrently)
  → Does NOT shrink the file on disk

  VACUUM FULL:
  1. Creates a NEW copy of the table with only live tuples
  2. Rebuilds all indexes
  3. Swaps old file with new file
  → REQUIRES EXCLUSIVE LOCK (table offline!)
  → Shrinks the file, returns space to OS
  → Use rarely, only when bloat is extreme
```

---

### Q16. Explain InnoDB's clustered index impact on secondary indexes.

```
InnoDB's CRITICAL DESIGN CHOICE:
════════════════════════════════

  Clustered index leaf nodes = actual row data (sorted by PK)
  Secondary index leaf nodes = indexed columns + PRIMARY KEY value

  Consequence: Secondary index lookup = TWO B-tree traversals

  SELECT * FROM users WHERE email = 'alice@example.com';

  Step 1: Traverse email index B-tree → find PK=42
  Step 2: Traverse clustered index B-tree with PK=42 → find full row

  This is called a "BOOKMARK LOOKUP" or "CLUSTERED INDEX LOOKUP"

IMPLICATIONS:
─────────────
  1. WIDE PRIMARY KEYS are expensive:
     If PK = UUID (16 bytes) vs INT (4 bytes):
     Every secondary index stores 16 bytes per entry instead of 4
     With 5 secondary indexes on 10M rows:
       UUID: 5 × 10M × 16 bytes = 800MB of extra index storage
       INT:  5 × 10M × 4 bytes  = 200MB
     → 4x difference!

  2. RANDOM PRIMARY KEYS cause fragmentation:
     UUID inserts scatter across all B-tree pages
     Sequential INT/BIGINT inserts append to the rightmost leaf
     → UUIDs = more page splits, more random I/O

  3. COVERING INDEX avoids the second lookup:
     CREATE INDEX idx ON users(email) INCLUDE (name);
     → email search returns name directly from secondary index
     → No clustered index lookup needed
```

---

### Q17. What are isolation levels? When does REPEATABLE READ still have issues?

```
REPEATABLE READ GOTCHAS:
════════════════════════

Standard SQL REPEATABLE READ allows PHANTOM READS:
  Txn A: SELECT COUNT(*) FROM orders WHERE status='pending'  → 5
  Txn B: INSERT INTO orders (status) VALUES ('pending')  → COMMIT
  Txn A: SELECT COUNT(*) FROM orders WHERE status='pending'  → 6!
  The new row is a "phantom"

MYSQL's REPEATABLE READ (InnoDB):
  Actually PREVENTS phantoms using next-key locks (record + gap)
  SELECT ... FOR UPDATE locks the range, preventing inserts
  But: "plain" SELECTs use snapshot (consistent read) so you
       won't see phantoms in non-locking reads either
  However: WRITE SKEW is still possible!

WRITE SKEW EXAMPLE:
  Table: doctors(id, name, on_call)
  Constraint: At least 1 doctor must be on call

  Txn A: SELECT COUNT(*) FROM doctors WHERE on_call = true → 2
         UPDATE doctors SET on_call = false WHERE id = 1  → COMMIT
  Txn B: SELECT COUNT(*) FROM doctors WHERE on_call = true → 2
         UPDATE doctors SET on_call = false WHERE id = 2  → COMMIT

  Both transactions saw 2 on-call doctors, each removed one.
  Result: 0 on-call doctors! Constraint violated!
  → Only SERIALIZABLE prevents this (SSI in PostgreSQL, S2PL in MySQL)
```

---

### Q18. Hash join vs nested loop join — when to use which?

```
NESTED LOOP:
  for each row in outer_table:         O(N × M) or O(N × log M) with index
    for each row in inner_table:
      if match → emit

  Best when:
  ✅ Outer table is SMALL (< 1000 rows)
  ✅ Inner table has INDEX on join column
  ✅ You need only first few rows (LIMIT)
  ❌ Terrible for two large tables without index

HASH JOIN:
  1. Build hash table on smaller input     O(N + M)
  2. Probe with larger input

  Best when:
  ✅ Large equi-joins (=)
  ✅ No useful index exists
  ✅ Enough memory for hash table (work_mem)
  ❌ Cannot do range joins (<, >, BETWEEN)
  ❌ Hash table spills to disk if too large

SORT-MERGE JOIN:                         O(N log N + M log M)
  1. Sort both inputs
  2. Merge sorted streams

  Best when:
  ✅ Both inputs already sorted (or have sorted indexes)
  ✅ Large equi-joins
  ✅ Need sorted output anyway (ORDER BY)
  ❌ Expensive if inputs need sorting from scratch
```

---

### Q19. What is the WAL? How does crash recovery (ARIES) work?

```
WAL = Write-Ahead Log
════════════════════
  Rule: EVERY change to a data page must FIRST be recorded in the WAL.
  The WAL record must reach disk BEFORE the dirty data page is flushed.
  This guarantees durability: if we crash, we can replay the WAL.

ARIES RECOVERY (3 Phases):
══════════════════════════

  Phase 1: ANALYSIS
  ─────────────────
  Start from last checkpoint. Scan WAL forward.
  Build two lists:
    • Dirty Page Table: pages modified but not yet flushed
    • Active Transaction Table: transactions that were in-progress

  Phase 2: REDO (Repeating history)
  ─────────────────────────────────
  Replay ALL WAL records from the earliest dirty page's LSN forward.
  Even replay changes from transactions that will be rolled back.
  This brings every page to its exact state at crash time.
  Why redo aborted txns? Because we don't know if their dirty pages
  were flushed to disk or not — redo is idempotent, so it's safe.

  Phase 3: UNDO (Rolling back losers)
  ────────────────────────────────────
  For each transaction in Active Transaction Table (uncommitted at crash):
  Walk their WAL records BACKWARD and undo each change.
  Write "Compensation Log Records" (CLR) so that if we crash DURING
  recovery, we don't undo the undo.

  After all 3 phases: DB is consistent and ready for new transactions.
```

---

### Q20. Explain optimistic vs pessimistic concurrency control.

```
PESSIMISTIC (Assume conflicts WILL happen):
════════════════════════════════════════════
  • Acquire locks BEFORE accessing data
  • Hold locks until transaction commits/aborts
  • Two-Phase Locking (2PL): growing phase → shrinking phase
  
  ✅ No wasted work (if conflict, other txn just waits)
  ❌ Deadlocks possible
  ❌ Reduced concurrency (readers block writers in strict 2PL)
  ❌ Lock overhead
  
  Used by: MySQL's SELECT ... FOR UPDATE, SQL Server default

OPTIMISTIC (Assume conflicts are RARE):
═══════════════════════════════════════
  • Read and write WITHOUT locks
  • At COMMIT time, check if any conflict occurred
  • If conflict detected → ABORT and retry the transaction

  ✅ No lock overhead during execution
  ✅ Readers never block writers
  ❌ Wasted work if conflict detected at commit (must retry)
  ❌ High abort rate under heavy contention
  
  Used by: PostgreSQL SSI, MVCC snapshot reads, OCC systems

  When to use which:
  • High contention (same rows updated often) → Pessimistic
  • Low contention (mostly reads, rare conflicts) → Optimistic
```

---

### Q21-Q30 (Condensed — key interview points)

**Q21. Covering Index:**
Contains all columns needed by a query, enabling index-only scans. The DB never touches the heap. In PostgreSQL, use `CREATE INDEX idx ON t(a) INCLUDE (b, c)`. In InnoDB, since secondary indexes already contain the PK, a query selecting only the indexed columns + PK is automatically a covering index scan.

**Q22. Redis sub-millisecond latency:**
Single-threaded event loop (no context switches), all data in RAM (no disk I/O for reads), efficient data structures (SDS strings, skip lists, ziplists), I/O multiplexing via epoll/kqueue, simple protocol (RESP). Since Redis 6.0, I/O threads handle network read/write while command execution remains single-threaded.

**Q23. MongoDB replica set failover:**
When a primary fails, secondaries detect via heartbeat timeout (~10s). An election begins using a Raft-like protocol. The secondary with the latest oplog entry is preferred. During election (~10-12s), no writes are accepted. In-flight writes with `w:1` may be lost. With `w:"majority"`, writes are only acknowledged after replicating to a majority — so surviving nodes have the data.

**Q24. Cassandra tunable consistency:**
Per-query consistency level. `ONE` = fastest (1 replica ACK). `QUORUM` = majority (⌊RF/2⌋+1). `ALL` = all replicas. Strong consistency requires `W + R > N`. Common pattern: `QUORUM` reads + `QUORUM` writes with RF=3. For cross-DC: `LOCAL_QUORUM` avoids cross-DC latency.

**Q25. InnoDB Change Buffer:**
Caches modifications to secondary index pages that aren't in the buffer pool. Instead of reading the index page from disk (random I/O), the change is buffered and merged later when the page is naturally read. Dramatically reduces random I/O for write-heavy workloads with many secondary indexes. Configured via `innodb_change_buffer_max_size`.

**Q26. Checkpoint tradeoff:**
Frequent checkpoints → less WAL to replay on crash (faster recovery) but more background I/O during normal operation. Infrequent checkpoints → less I/O overhead but longer recovery time and larger WAL accumulation. PostgreSQL: `checkpoint_timeout` (default 5 min), `max_wal_size`. InnoDB: redo log size determines effective checkpoint frequency.

**Q27. PostgreSQL SSI vs MySQL serializable:**
PostgreSQL SSI is optimistic — transactions run without blocking, and the system detects serialization anomalies at commit time by tracking read-write dependencies. If a cycle is found, one transaction is aborted. MySQL's serializable mode uses pessimistic locking (every SELECT becomes SELECT ... LOCK IN SHARE MODE) which blocks more but never wastes work. SSI has higher throughput under low contention; MySQL's approach is better under high contention.

**Q28. Sharding vs Partitioning:**
Partitioning splits a table into pieces that may live on the same server (PostgreSQL declarative partitioning, MySQL partitions). Sharding distributes those partitions across multiple servers. All sharding involves partitioning, but not all partitioning involves sharding.

**Q29. Read-your-writes consistency:**
After a write, ensure the same user reads from a source that reflects that write. Implementation: (a) route the user's post-write reads to the primary for a short window, (b) include the write's LSN/timestamp in the session and only read from replicas that have caught up past it, (c) use a sticky session to the same replica if it's guaranteed to be up-to-date.

**Q30. ALTER TABLE on 1TB table:**
In PostgreSQL, most ALTERs (add nullable column, rename) take only a metadata lock (instant). Adding a NOT NULL column with default rewrites the entire table in older versions but is instant in PG 11+. In MySQL, `ALTER TABLE` on InnoDB historically required a full table copy. Online DDL (MySQL 5.6+) can do many operations in-place but some still require a table rebuild. Tools like `pt-online-schema-change` or `gh-ost` do it with minimal locking by creating a shadow table and replaying changes.

---

## Advanced (5+ YOE) — Q31-Q50

### Q31. Sharding strategy for multi-tenant SaaS.

```
SHARD KEY: tenant_id
════════════════════

  Why tenant_id:
  ✅ Queries are almost always scoped to one tenant
  ✅ No cross-shard queries for single-tenant operations
  ✅ Natural data isolation
  ✅ Usually high cardinality (many tenants)
  ✅ Even distribution (if tenants are similar size)

  Handling "whale" tenants (10x-100x data of normal):
  ┌──────────────────────────────────────────────────┐
  │ Strategy 1: Dedicated shard per whale tenant     │
  │ Strategy 2: Sub-shard whale tenant by another    │
  │             key (e.g., tenant_id + user_id hash) │
  │ Strategy 3: Use consistent hashing with virtual  │
  │             nodes — whale gets more vnodes        │
  └──────────────────────────────────────────────────┘

  Cross-tenant queries (admin dashboards, analytics):
  → Scatter-gather across all shards (expensive but rare)
  → OR replicate to analytics DB (ClickHouse, BigQuery)

  Routing:
  App → Routing Layer (Vitess/ProxySQL/app logic) → Correct shard
  Mapping: tenant_id → shard can be hash-based or directory-based
```

---

### Q32. Zero-downtime migration to sharded architecture.

```
STEP-BY-STEP:
═════════════

  Phase 1: DUAL WRITES
  ─────────────────────
  App writes to BOTH old single DB and new sharded DB.
  Read from old DB (source of truth).

  Phase 2: BACKFILL
  ─────────────────
  Migrate historical data from old DB to new sharded DB.
  Use CDC (Change Data Capture) tool like Debezium to
  capture changes during migration.

  Phase 3: VERIFY
  ────────────────
  Run consistency checks: compare row counts, checksums,
  sample queries between old and new.

  Phase 4: SWITCH READS
  ─────────────────────
  Route reads to new sharded DB.
  Keep writes going to both (safety net).

  Phase 5: SWITCH WRITES
  ──────────────────────
  Route writes only to new sharded DB.
  Keep old DB as read-only backup temporarily.

  Phase 6: DECOMMISSION
  ─────────────────────
  Turn off old DB after confidence period.

  Alternative: Use Vitess (MySQL) or Citus (PostgreSQL)
  which handle online resharding transparently.
```

---

### Q33-Q50 (Key interview answer points)

**Q33. Cassandra hinted handoff + repair:**
When target node is down, coordinator stores the write as a "hint" (target node + mutation data). When the node returns, hints are replayed. Hints expire after `max_hint_window_in_ms` (default 3 hours). For longer outages, `nodetool repair` uses Merkle trees to compare data between replicas and reconcile differences. Run repair within `gc_grace_seconds` (default 10 days) to prevent resurrecting deleted data.

**Q34. PostgreSQL transaction ID wraparound:**
32-bit txn IDs (4 billion total, but circular comparison uses ~2 billion window). If a tuple's xmin is more than 2 billion txns in the past, it wraps and appears to be "in the future" → invisible. VACUUM freezes old tuples by marking them as "frozen" (visible to all transactions). `autovacuum_freeze_max_age` triggers aggressive VACUUM. If VACUUM can't keep up, PostgreSQL enters "emergency" mode and will SHUT DOWN to prevent data loss.

**Q35. InnoDB Doublewrite Buffer:**
Problem: a 16KB InnoDB page write may be interrupted by a crash mid-write ("torn page"). The WAL can't fix this because redo records assume the page base is valid. Solution: before flushing dirty pages, InnoDB writes them to a contiguous "doublewrite buffer" area on disk (sequential write). Then writes pages to their final locations (random write). On crash recovery, if a page is torn, restore it from the doublewrite buffer, then apply redo log.

**Q36. Real-time chat DB architecture:**

```
Messages:      Cassandra (write-heavy, partition by conversation_id,
               cluster by timestamp DESC)
User Presence: Redis (SET with TTL, pub/sub for status changes)
User Profiles: PostgreSQL (relational, ACID)
Search:        Elasticsearch (full-text message search)
Unread Counts: Redis (INCR per user per conversation, atomic)
Push Queue:    Kafka → consumer writes to push notification service
```

**Q37. Hot partition in Cassandra:**
Detect via `nodetool tablehistograms`. Fix: add a "bucket" to the partition key (e.g., `sensor_id:bucket` where `bucket = timestamp_hour % 12`). This spreads a single logical partition across 12 physical partitions. Tradeoff: reads must now scatter across 12 partitions and merge.

**Q38. LSM compaction strategies:**
Size-tiered: group SSTables by size, merge similar-sized ones. Lower write amplification, higher space amplification (up to 2x during compaction). Leveled: each level is 10x the previous, non-overlapping key ranges per level. Higher write amplification, but much better read performance and space usage. FIFO: drop oldest SSTables entirely. Only for TTL data where old data is expendable.

**Q39. Elasticsearch near-real-time:**
New documents go to an in-memory buffer. Every 1 second (configurable `refresh_interval`), the buffer is written as a new Lucene segment (searchable but not fsync'd). The transaction log (translog) ensures durability — it's fsync'd every 5 seconds or on explicit flush. Segments are immutable; deletes use `.del` bitmap files. Background merge combines segments. This 1-second refresh interval is why ES is "near-real-time" not "real-time."

**Q40. Split-brain prevention:**

```
PostgreSQL: Patroni uses etcd/ZooKeeper for leader election.
            Only node with the leader key can accept writes.
            Fencing: revoke old primary's access to storage.

MySQL:      Group Replication uses Paxos-based consensus.
            Node must be part of majority group to accept writes.

MongoDB:    Replica set election requires majority vote.
            Old primary steps down if it can't reach majority.

Redis:      Sentinel uses quorum for failover decision.
            min-replicas-to-write prevents isolated primary
            from accepting writes.

Cassandra:  No split-brain by design — no leader!
            All nodes are equal, consistency via quorum.
```

**Q41. CockroachDB/Spanner distributed transactions:**
Use a combination of Raft consensus (for replication within a range), MVCC with hybrid-logical clocks (HLC), and a parallel commit protocol. Spanner uses TrueTime (GPS + atomic clocks) for globally ordered timestamps. CockroachDB uses HLC which doesn't require special hardware. Both provide serializable isolation globally. Writes to a range go through the Raft leader; cross-range transactions use 2PC with the transaction coordinator.

**Q42. Global leaderboard with Redis:**

```
Design: Redis Sorted Set (ZSET)
  ZADD leaderboard 1500 "player:alice"    O(log N)
  ZADD leaderboard 2100 "player:bob"      O(log N)

  Top 10:     ZREVRANGE leaderboard 0 9 WITHSCORES    O(log N + 10)
  My rank:    ZREVRANK leaderboard "player:alice"      O(log N)
  My score:   ZSCORE leaderboard "player:alice"        O(1)
  Around me:  ZREVRANGE leaderboard (myrank-5) (myrank+5) O(log N + 10)

  For 10M users: All operations are O(log 10M) ≈ 23 steps. Sub-ms.

  Scaling: Partition by region/game-mode into separate ZSETs.
           Merge top-N from each partition for global leaderboard.
```

**Q43. CAP oversimplification (PACELC):**
CAP only describes behavior during partitions. PACELC extends this: Even when there's no partition, there's a latency vs consistency tradeoff. DynamoDB and Cassandra choose low latency normally (EL) and availability during partitions (PA). PostgreSQL and CockroachDB choose consistency both normally (EC) and during partitions (PC). Real systems are nuanced — MongoDB with `w:majority` is PC/EC, but with `w:1` behaves more like PA/EL.

**Q44. Cross-shard queries:**
Minimize by choosing shard key that aligns with query patterns. When unavoidable: (a) scatter-gather — send query to all shards, merge results at coordinator (expensive), (b) maintain global secondary indexes (e.g., Vitess VIndex), (c) replicate data to an analytics system (BigQuery, ClickHouse) for cross-shard analytics, (d) denormalize — copy frequently-needed cross-shard data.

**Q45. Survive full DC outage:**
Multi-region replication (sync or semi-sync to at least one remote DC). DNS-based failover (Route53 health checks → redirect traffic). Active-active setup with conflict resolution, or active-passive with promotion. Cassandra: set RF per DC (e.g., RF=3 in each of 2 DCs). CockroachDB: geo-partitioned with survival goal = region. Test regularly with chaos engineering.

---

# Part B — PostgreSQL Questions

### PG-1. How does PostgreSQL's process model work?

One OS process per client connection. The `postmaster` process accepts connections and forks a backend process for each. All backends share memory via `shared_buffers`, `WAL buffers`, and `CLOG`. Background processes include: `autovacuum launcher`, `WAL writer`, `background writer`, `checkpointer`, `stats collector`. This process-per-connection model is why connection pooling (PgBouncer) is critical — each process uses ~10MB of memory.

### PG-2. What is HOT (Heap Only Tuple)?

When an UPDATE doesn't change any indexed column AND there's free space on the same heap page, PostgreSQL creates a "HOT" update — the new tuple is placed on the same page and linked from the old tuple via a HOT chain. The index doesn't need updating. Benefit: dramatically reduces index bloat for workloads with frequent non-indexed column updates. Limitation: requires free space on the same page (controlled by `fillfactor`).

### PG-3. Explain the visibility map and free space map.

```
VISIBILITY MAP (VM):
  1 bit per heap page. Set = "all tuples on this page visible to all txns"
  → Index-only scans check VM: if page is all-visible, skip heap lookup
  → VACUUM only needs to scan pages NOT marked all-visible
  → Huge performance win for mostly-read tables

FREE SPACE MAP (FSM):
  Tracks available space per heap page
  → INSERT checks FSM to find a page with enough space
  → Avoids extending the file unnecessarily
  → VACUUM updates FSM after reclaiming dead tuples
```

### PG-4. What is the CLOG (Commit Log)?

Stores the commit status of every transaction ID: in-progress, committed, aborted, or sub-committed. Stored in `pg_xact/` directory. When a backend checks tuple visibility, it looks up the creating/deleting transaction's status in CLOG. Recently accessed CLOG pages are cached in shared memory.

### PG-5. How does PostgreSQL's cost-based optimizer work?

The optimizer generates multiple execution plans and estimates the cost of each based on: `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, `cpu_index_tuple_cost`, `effective_cache_size`. It uses table statistics from `pg_statistic` (maintained by ANALYZE): row counts, distinct values, most common values, histograms of value distribution. For queries with many joins (12+), it uses the Genetic Query Optimizer (GEQO) because exhaustive enumeration would be too slow.

### PG-6. What is TOAST (The Oversized-Attribute Storage Technique)?

When a row exceeds ~2KB, PostgreSQL compresses and/or stores large column values in a separate TOAST table. Strategies per column: `PLAIN` (no TOAST), `EXTENDED` (compress first, then external), `EXTERNAL` (external without compression), `MAIN` (try compression, avoid external). TOAST is transparent — queries see normal data. Fetching TOASTed values requires extra I/O.

### PG-7. How does streaming replication work internally?

Primary writes WAL records. `walsender` process on primary streams WAL to `walreceiver` on replica via a TCP connection. Replica replays WAL records to stay in sync. Sync replication: primary waits for replica to write WAL to disk before acknowledging commit. Async: primary doesn't wait. Replication slots prevent the primary from recycling WAL segments that the replica hasn't consumed yet (prevents replica from falling too far behind, but risks disk filling up).

### PG-8. Explain pg_stat_statements and how to use it.

Extension that tracks execution statistics for all SQL statements: call count, total/min/max/mean time, rows returned, buffer hits/reads, WAL bytes. Essential for identifying slow and frequent queries. Query: `SELECT query, calls, total_exec_time, mean_exec_time FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;`. Normalize queries (parameter values replaced with `$1, $2...`).

### PG-9. What is logical replication vs physical replication?

Physical: streams raw WAL bytes. Replica is an exact binary copy. Can't replicate selectively. Must be same major PG version. Used for failover and read replicas.

Logical: decodes WAL into logical changes (INSERT/UPDATE/DELETE on specific tables). Can replicate subset of tables, across different major versions, even to different database systems. Uses "publications" and "subscriptions." Higher overhead but much more flexible.

### PG-10. How do advisory locks work?

Application-defined locks not tied to any table or row. Two types: session-level (held until session ends) and transaction-level (released on commit/rollback). Obtained via `pg_advisory_lock(key)`. Use cases: preventing duplicate cron job execution, rate limiting, coordinating distributed workers. Unlike row locks, they don't cause VACUUM issues or table bloat.

### PG-11 to PG-25. (Quick-fire)

**PG-11.** `EXPLAIN (ANALYZE, BUFFERS)` — what do "shared hit" vs "shared read" mean? Hit = page found in buffer pool (cache). Read = page fetched from disk. High read count = cold cache or dataset larger than shared_buffers.

**PG-12.** Partial index: `CREATE INDEX idx ON orders(created_at) WHERE status = 'pending'`. Only indexes pending orders — much smaller, faster, and targeted.

**PG-13.** Expression index: `CREATE INDEX idx ON users(LOWER(email))`. Allows efficient lookups on function results.

**PG-14.** `work_mem` vs `shared_buffers`: shared_buffers is the global page cache; work_mem is per-operation memory for sorts and hashes. Setting work_mem too high with many concurrent queries can exhaust memory.

**PG-15.** Table inheritance vs partitioning: Modern PostgreSQL (10+) uses declarative partitioning (range/list/hash). Table inheritance is the legacy approach — partitioning is preferred.

**PG-16.** `pg_repack` vs `VACUUM FULL`: both compact tables, but pg_repack doesn't require an exclusive lock — it creates a new table, replays changes via triggers, and swaps. Much less downtime.

**PG-17.** Parallel query: PG can parallelize seq scans, joins, and aggregates. Controlled by `max_parallel_workers_per_gather`. Not used for: small tables, writes, cursors, or serializable isolation.

**PG-18.** JIT compilation (PG 11+): Just-In-Time compiles query expressions to native code using LLVM. Beneficial for CPU-bound analytical queries. Overhead for simple OLTP queries — `jit_above_cost` threshold controls activation.

**PG-19.** What is `pg_hba.conf`? Host-based authentication configuration. Controls: which users can connect, from which hosts, to which databases, using which auth method (md5, scram-sha-256, cert, trust, reject).

**PG-20.** Deadlock detection: PG maintains a wait-for graph. Checks every `deadlock_timeout` (default 1s). When a cycle is detected, aborts the youngest transaction (least work invested). Log message includes all involved queries.

**PG-21.** `SERIALIZABLE` in PG uses SSI (Serializable Snapshot Isolation): tracks rw-dependencies between concurrent transactions. If a "dangerous structure" (cycle of dependencies) is detected at commit, one transaction is aborted. No extra locking during execution — only detection at commit.

**PG-22.** How does `LISTEN/NOTIFY` work? Async inter-process messaging. A session does `LISTEN channel_name`. Another does `NOTIFY channel_name, 'payload'`. All listeners receive the message. Uses shared memory. Lightweight alternative to polling for cache invalidation, job queues.

**PG-23.** Foreign Data Wrappers (FDW): allow querying external data sources (other PostgreSQL instances, MySQL, CSV files, APIs) as if they were local tables. `postgres_fdw` for cross-server queries. Pushes down WHERE, JOIN, SORT when possible.

**PG-24.** `pg_basebackup` + WAL archiving for PITR: Take a full physical backup, continuously archive WAL segments to object storage. To restore to a specific point in time: restore the base backup, then replay WAL up to the target timestamp. Configured via `archive_command` and `recovery_target_time`.

**PG-25.** Extensions ecosystem: `pg_stat_statements` (query stats), `PostGIS` (geospatial), `pg_trgm` (trigram similarity search), `Citus` (distributed), `TimescaleDB` (time-series), `pgvector` (AI/ML vector similarity search), `pg_cron` (scheduled jobs). Extensions run inside the database process — native performance.

---

# Part C — MySQL / InnoDB Questions

### MY-1. Explain the MySQL server layer vs storage engine layer.

The server layer handles connections, parsing, optimization, caching, and cross-storage-engine features (replication binlog). The storage engine layer handles actual data storage, retrieval, indexing, and transactions. InnoDB is the default and most important engine. This separation means MySQL can swap storage engines per table, but it also means some features are duplicated (e.g., both redo log at InnoDB level and binlog at server level).

### MY-2. How does InnoDB's buffer pool work?

```
  ┌──────────────────────────────────────────────────────────┐
  │                    BUFFER POOL                            │
  │  Recommended: 70-80% of available RAM                    │
  │                                                           │
  │  Modified LRU with young/old sublist:                     │
  │  ┌──────────────────┬─────────────────────┐              │
  │  │   YOUNG (hot)    │    OLD (cold)        │              │
  │  │   sublist ~5/8   │    sublist ~3/8      │              │
  │  └──────────────────┴─────────────────────┘              │
  │                                                           │
  │  New pages enter at OLD head (not YOUNG)                  │
  │  Page moves to YOUNG head only if accessed again          │
  │  after innodb_old_blocks_time (default 1 second)          │
  │                                                           │
  │  WHY: Full table scans load many pages once — they        │
  │  enter OLD and get evicted quickly without polluting      │
  │  the YOUNG sublist where hot frequently-accessed          │
  │  pages live.                                              │
  └──────────────────────────────────────────────────────────┘
```

### MY-3. What is the Adaptive Hash Index?

InnoDB automatically builds in-memory hash indexes on frequently accessed B-tree index pages. It monitors access patterns — if a page is accessed repeatedly with the same search key, InnoDB builds a hash index for O(1) lookups. Fully automatic, no DBA intervention. Can be disabled with `innodb_adaptive_hash_index=OFF` if the overhead of maintaining it outweighs the benefit (high write workloads).

### MY-4. Binary log formats: STATEMENT vs ROW vs MIXED.

**STATEMENT:** Logs the SQL statement itself. Compact but non-deterministic functions (`NOW()`, `UUID()`, `RAND()`) can produce different results on replica. **ROW:** Logs actual row changes (before/after images). Deterministic and safe, but verbose (large binlog for bulk updates). **MIXED:** Uses STATEMENT by default, switches to ROW when the statement is non-deterministic. Recommended for most setups. Since MySQL 8.0, ROW-based is the default.

### MY-5. Explain GTID (Global Transaction ID) replication.

Each transaction gets a globally unique identifier: `server_uuid:transaction_id`. Replicas track which GTIDs they've executed. Benefits: (a) easy failover — a new replica knows exactly which transactions it's missing, (b) no need to track binlog file + position, (c) automatic duplicate detection. Essential for MySQL Group Replication and InnoDB Cluster.

### MY-6. What is the redo log and how does it differ from the binlog?

Redo log: InnoDB-specific, physical page-level changes, circular buffer (fixed size, overwritten), used for crash recovery. Binlog: server-level, logical changes (SQL or row images), append-only (files rotate), used for replication and PITR. Write order: redo log first (prepare), then binlog (commit). This two-phase relationship is the "internal XA" that keeps redo and binlog in sync.

### MY-7. How does InnoDB handle deadlocks?

InnoDB detects deadlocks using a wait-for graph. When a cycle is found, it immediately rolls back the transaction with the least undo work (smallest transaction). The error `ERROR 1213 (ER_LOCK_DEADLOCK)` is returned. Applications should catch this and retry. Minimize deadlocks by: accessing tables/rows in consistent order, keeping transactions short, and using appropriate isolation levels.

### MY-8. What is the doublewrite buffer? Why is it needed?

```
Problem: OS may write a 16KB InnoDB page in multiple 4KB chunks.
         Crash during write = "torn page" (partially written).
         Redo log can't fix this — it assumes page base is intact.

Solution: Doublewrite Buffer
  1. Write dirty pages to doublewrite buffer (sequential area on disk)
  2. fsync the doublewrite buffer
  3. Write pages to their actual locations (random I/O)
  4. If crash during step 3:
     → Recover intact page from doublewrite buffer
     → Then apply redo log
```

### MY-9. What is Online DDL in InnoDB?

Adding/modifying indexes, columns, etc. without blocking DML. Three modes: `INSTANT` (metadata only — add nullable column, rename), `INPLACE` (no table copy, but may need to rebuild index), `COPY` (creates new table, copies data). Not all DDL supports all modes. For operations requiring COPY on large tables, use `pt-online-schema-change` or `gh-ost` which create a shadow table, apply changes, replay concurrent DML via triggers/binlog, and swap.

### MY-10. Explain InnoDB's undo tablespace and purge system.

Undo tablespace stores old row versions for MVCC and rollback. Each transaction gets a rollback segment. After commit, undo records aren't immediately deleted — active transactions may need them. The purge system (background threads) removes undo records that no active transaction can see. `innodb_purge_threads` controls parallelism. Long-running transactions prevent purge → undo tablespace grows (monitor with `SHOW ENGINE INNODB STATUS`, look for "History list length").

### MY-11 to MY-25. (Quick-fire)

**MY-11.** InnoDB page structure: Header (checksum, page type, LSN), User Records (B-tree leaf/internal records), Page Directory (sparse index of records within page), Trailer (checksum for torn page detection).

**MY-12.** `innodb_flush_log_at_trx_commit`: `1` = fsync on every commit (safest, default). `2` = write to OS buffer on commit, fsync every 1s (small data loss risk). `0` = write and fsync every 1s (fastest, up to 1s data loss).

**MY-13.** MySQL Group Replication: built-in multi-primary or single-primary mode using Paxos-based consensus. Automatic failover, conflict detection. Foundation of InnoDB Cluster (Group Replication + MySQL Router + MySQL Shell).

**MY-14.** Partitioning in MySQL: RANGE, LIST, HASH, KEY. Partition pruning = optimizer skips irrelevant partitions. Useful for time-series data (`PARTITION BY RANGE (YEAR(created_at))`). Max 8192 partitions per table.

**MY-15.** `SELECT ... FOR UPDATE` vs `FOR SHARE`: FOR UPDATE takes exclusive (X) locks — blocks other writes AND reads-for-update. FOR SHARE takes shared (S) locks — blocks writes but allows other shared reads. Both prevent phantom reads under REPEATABLE READ via next-key locks.

**MY-16.** `innodb_file_per_table`: each table gets its own `.ibd` file (default ON since 5.6). Benefits: disk space reclaimed on DROP TABLE, easier backup/restore per table, less system tablespace bloat.

**MY-17.** Semi-sync replication: primary waits for at least one replica to ACK before returning commit to client. Ensures at least one replica has the data. Configured with `rpl_semi_sync_master_wait_for_slave_count`. Falls back to async if replicas are slow (configurable timeout).

**MY-18.** Query cache (removed in 8.0): cached full query results keyed by exact SQL text. Invalidated on ANY write to any table referenced by the query. Caused severe contention on write-heavy workloads. Use application-level caching (Redis) instead.

**MY-19.** `SHOW ENGINE INNODB STATUS`: Shows buffer pool hit rate, row operations/sec, redo log sequence, semaphore waits, deadlock info, active transactions, purge lag. Essential debugging tool.

**MY-20.** Invisible indexes (MySQL 8.0+): mark an index as invisible to the optimizer without dropping it. Test the impact of dropping an index safely. `ALTER TABLE t ALTER INDEX idx INVISIBLE/VISIBLE;`

**MY-21.** `innodb_io_capacity` and `innodb_io_capacity_max`: tell InnoDB how fast the storage system is (IOPS). Controls background flushing rate. SSD: set high (2000-20000). HDD: lower (200-400). Wrong values → either too aggressive flushing (perf overhead) or too lazy (redo log fills up, stalls).

**MY-22.** Multi-source replication: one replica pulls from multiple primaries. Use case: consolidating data from multiple shards into one reporting server. Configured with replication channels.

**MY-23.** `mysqldump` vs `xtrabackup`: mysqldump is logical (SQL statements), slow for large DBs, cross-version compatible. xtrabackup (Percona) is physical (copies data files + applies redo log), much faster for large DBs, hot backup (no lock for InnoDB tables).

**MY-24.** MySQL Router: lightweight proxy for InnoDB Cluster. Routes reads to secondaries, writes to primary. Automatic primary detection on failover. Pluggable routing strategies.

**MY-25.** Vitess for sharding: open-source horizontal scaling system (developed at YouTube). Handles connection pooling, query routing, resharding, schema management. Application talks to Vitess via MySQL protocol — transparent to the app. VSchema defines how tables are sharded.

---

# Part D — MongoDB Questions

### MG-1. How does WiredTiger's cache differ from the OS page cache?

WiredTiger maintains its own internal cache (default: 50% of RAM - 1GB, or 256MB minimum). It stores documents in an uncompressed, decoded format for fast access. The OS page cache holds the on-disk compressed format. This means data can exist in memory twice (WT cache + OS cache) in different forms. Best practice: size WT cache to fit the working set; let the OS cache handle the rest.

### MG-2. Explain the oplog and its implications.

The oplog (`local.oplog.rs`) is a capped collection on each replica set member. It stores idempotent operations (not SQL statements — actual inserts, field-level updates, deletes). Secondaries tail the oplog to replicate changes. The oplog size determines the "replication window" — how long a secondary can be offline and still catch up. If a secondary falls behind the oplog, it requires a full resync (expensive). Monitor with `rs.printReplicationInfo()`.

### MG-3. What is the aggregation pipeline vs MapReduce?

Aggregation pipeline: declarative stages (`$match`, `$group`, `$sort`, `$lookup`, `$project`, `$unwind`, etc.) processed in sequence. Executed in C++ within the engine — fast. MapReduce: user-defined JavaScript functions for map and reduce phases. Much slower (JavaScript execution overhead), deprecated since 4.4. Always prefer aggregation pipeline.

### MG-4. How do MongoDB transactions work internally?

Since 4.0: multi-document transactions within a replica set. Since 4.2: distributed transactions across shards. WiredTiger provides snapshot isolation — transaction sees a consistent point-in-time view. Writes are buffered; on commit, they're applied atomically. Internally uses a transaction timestamp and WiredTiger's MVCC. Limitations: 60-second time limit, 16MB size limit for oplog entry, performance overhead (reduced throughput). Design to minimize transaction scope.

### MG-5. Explain the difference between `$lookup` and a traditional SQL JOIN.

`$lookup` performs a left outer join by querying the joined collection for each input document. For each input document, it runs a query on the foreign collection. Not a hash join or sort-merge — more like a nested loop with index. Since 5.0: can use more complex `$lookup` with `pipeline` sub-stages. Performance: much slower than RDBMS joins for large datasets. Mitigation: embed related data in the same document (MongoDB's preferred pattern) or create indexes on the lookup field.

### MG-6 to MG-20. (Quick-fire)

**MG-6.** Shard key immutability: once set, the shard key field cannot be changed (the value can be updated since 4.2 with certain limitations, but the choice of which field is the shard key is permanent without migration).

**MG-7.** Chunk: a contiguous range of shard key values. Default size 128MB. The balancer moves chunks between shards for even distribution. Jumbo chunks (exceed max size, can't be split) are problematic.

**MG-8.** Read preference: `primary` (default, consistent), `primaryPreferred`, `secondary` (stale reads OK), `secondaryPreferred`, `nearest` (lowest latency). Choose based on consistency requirements.

**MG-9.** Write concern: `w:0` (fire-and-forget), `w:1` (primary ACK), `w:"majority"` (majority of replica set ACKs — recommended for durability), `w:N` (N members ACK). `j:true` requires journal sync on disk.

**MG-10.** Compound index vs single field index: MongoDB uses index intersection (combining results of multiple single-field indexes) but compound indexes are almost always more efficient for multi-field queries. Follow the ESR rule: Equality → Sort → Range for compound index field order.

**MG-11.** TTL indexes: automatically delete documents after a specified time. `createIndex({"createdAt": 1}, {expireAfterSeconds: 86400})`. Background thread checks every 60 seconds. Use for sessions, logs, temporary data.

**MG-12.** Change streams: real-time notification of data changes. Backed by the oplog. Can watch a collection, database, or entire deployment. Use for event-driven architectures, cache invalidation, real-time sync.

**MG-13.** `explain("executionStats")`: MongoDB's equivalent of EXPLAIN ANALYZE. Shows winning plan, rejected plans, index usage, documents examined vs returned, execution time. Key metric: ratio of `totalDocsExamined` to `nReturned` (ideally close to 1:1).

**MG-14.** Schema validation: `db.createCollection("users", {validator: {$jsonSchema: {...}}})`. Enforces document structure at write time. Validation levels: `strict` (all inserts/updates), `moderate` (only valid documents are validated on update).

**MG-15.** Capped collections: fixed-size, insertion-order collections. Old documents automatically deleted when full. No indexes except `_id`. Used for: oplog, logging, circular buffers.

**MG-16.** `$graphLookup`: recursive lookup for graph-like queries (e.g., organizational hierarchy, social connections). Performs breadth-first search on a collection. Limited to a single collection.

**MG-17.** mongodump/mongorestore vs filesystem snapshot: mongodump is logical (BSON export), works across versions, but slow for large DBs. Filesystem snapshots (LVM, EBS) are instant but require `db.fsyncLock()` or consistent snapshot support.

**MG-18.** Atlas Search: built on Apache Lucene, integrated into MongoDB Atlas. Supports full-text search, fuzzy matching, autocomplete, facets, vector search. Queries via `$search` aggregation stage. Alternative to running a separate Elasticsearch cluster.

**MG-19.** Retryable writes and idempotency: since 3.6, drivers automatically retry failed writes once. Each write carries a transaction number + session ID — the server detects and deduplicates retries. Inserts, updates, deletes are safe to retry.

**MG-20.** When NOT to use MongoDB: heavy joins/relationships (use RDBMS), strict ACID across many documents (overhead is high), complex aggregations with multiple table lookups (RDBMS optimizer is far better), small dataset that fits one server (PostgreSQL is simpler and more feature-rich).

---

# Part E — Redis Questions

### RD-1. Why is Redis single-threaded but still fast?

```
  1. ALL data in RAM — no disk I/O for reads
  2. Single-threaded command execution:
     → No context switching overhead
     → No locking/synchronization overhead
     → No race conditions
  3. I/O multiplexing (epoll/kqueue):
     → One thread handles thousands of connections
  4. Efficient data structures:
     → SDS strings (length-prefixed, O(1) length)
     → Skip lists for sorted sets (probabilistic balance)
     → Compact encodings (ziplist/listpack for small datasets)
  5. Simple protocol (RESP): easy to parse, low overhead
  6. Since Redis 6.0: I/O threads handle socket read/write
     → Command execution still single-threaded
     → Reduces network I/O bottleneck
```

### RD-2. Explain Redis eviction policies.

When `maxmemory` is reached, Redis must evict keys:

```
  noeviction:        Return error on writes (don't evict)
  allkeys-lru:       Evict least recently used key (any key)
  allkeys-lfu:       Evict least frequently used (Redis 4.0+)
  allkeys-random:    Evict random key
  volatile-lru:      LRU among keys with TTL set
  volatile-lfu:      LFU among keys with TTL set
  volatile-random:   Random among keys with TTL set
  volatile-ttl:      Evict key closest to expiration

  Best for caching: allkeys-lru or allkeys-lfu
  Best when mixing cache + persistent data: volatile-lru
```

### RD-3. How does Redis Cluster handle failover?

Each master node has one or more replicas. Nodes send PING/PONG heartbeats. If a master doesn't respond for `cluster-node-timeout` (default 15s), replicas start an election. The replica with the most up-to-date replication offset is preferred. Other masters vote. Promoted replica takes over the failed master's hash slots. Client libraries automatically redirect to the new master. During failover window: slot is unavailable (seconds).

### RD-4. Explain Redis Streams.

Append-only log structure (like Kafka topics). Each entry has an auto-generated ID (timestamp-based) and key-value pairs. Supports consumer groups: multiple consumers independently read from the stream, each processing different entries. `XADD` appends, `XREAD` reads, `XREADGROUP` reads within a consumer group with ACK support. Use cases: event sourcing, message queues, activity feeds. Advantage over pub/sub: persistence and consumer group semantics (don't lose messages if consumer is offline).

### RD-5. What is the Redis Sentinel?

Monitoring and automatic failover system for Redis master-replica setups (non-Cluster). Sentinel instances (run 3+ for quorum) monitor the master. If master fails, Sentinels agree on failover via quorum vote, promote a replica, and reconfigure other replicas. Clients query Sentinel for the current master address. Difference from Redis Cluster: Sentinel manages a single master-replica group; Cluster manages multiple shards.

### RD-6 to RD-20. (Quick-fire)

**RD-6.** Pub/Sub: fire-and-forget messaging. `SUBSCRIBE channel` / `PUBLISH channel message`. Messages are NOT persisted — if no subscriber is listening, message is lost. For persistent messaging, use Redis Streams instead.

**RD-7.** Pipelining: send multiple commands without waiting for each response. Reduces network round trips. `MULTI` (transactions) vs pipelining: MULTI guarantees atomicity; pipelining just batches for performance.

**RD-8.** Lua scripting: `EVAL` executes Lua scripts atomically on the server. The script runs to completion without interruption (single-threaded). Use for complex atomic operations that can't be expressed with built-in commands. Equivalent to stored procedures.

**RD-9.** `MULTI`/`EXEC` transactions: all commands between MULTI and EXEC are queued and executed atomically. No rollback on individual command failure — if one command errors, others still execute. Use `WATCH` for optimistic locking (CAS-like behavior).

**RD-10.** Memory optimization: use hashes for small objects (`HSET user:123 name Alice age 25`) instead of separate keys (string per field). Use `ziplist`/`listpack` encoding for small collections. Set `hash-max-ziplist-entries` and `hash-max-ziplist-value` appropriately.

**RD-11.** `SCAN` vs `KEYS`: `KEYS pattern` blocks the server while scanning ALL keys (O(N)). NEVER use in production. `SCAN cursor MATCH pattern COUNT hint` iterates incrementally with a cursor — non-blocking, safe for production.

**RD-12.** Redlock algorithm: distributed locking across multiple Redis instances. Acquire lock on majority (e.g., 3 of 5 independent Redis masters). If majority acquired within timeout → lock is valid. Controversial: Martin Kleppmann argued it's unsafe due to clock drift; Salvatore Sanfilippo (Redis creator) disagreed. For non-critical distributed locks, Redlock is practical. For safety-critical systems, use a consensus system (ZooKeeper, etcd).

**RD-13.** `OBJECT ENCODING key`: shows the internal encoding of a key. Useful for debugging memory usage. Examples: `embstr` (small string), `int` (numeric string), `ziplist` (small list/hash), `skiplist` (sorted set), `hashtable` (large hash/set).

**RD-14.** Key expiration internals: Redis uses two strategies: (a) lazy expiration — check TTL on access, delete if expired; (b) active expiration — periodically sample random keys with TTL, delete expired ones. This means expired keys may linger in memory briefly. `EXPIREAT` vs `EXPIRE`: absolute timestamp vs relative seconds.

**RD-15.** `INFO` command: returns server statistics in sections (server, clients, memory, stats, replication, cpu, keyspace). Key metrics: `used_memory`, `connected_clients`, `instantaneous_ops_per_sec`, `keyspace_hits/misses` (cache hit ratio).

**RD-16.** Redis on Flash / Redis on SSD: extends Redis to use SSDs for less-frequently accessed values while keeping hot data in RAM. Implemented in Redis Enterprise. Keys always in RAM, values tiered. Not available in open-source Redis.

**RD-17.** `WAIT numreplicas timeout`: blocks until the specified number of replicas have acknowledged all prior writes. Provides synchronous-like durability on an otherwise async replication setup. Use judiciously — adds latency.

**RD-18.** HyperLogLog: probabilistic data structure for cardinality estimation. `PFADD` adds elements, `PFCOUNT` returns approximate count. Uses only ~12KB per key regardless of cardinality. Standard error < 1%. Use for: unique visitor counts, distinct event counting.

**RD-19.** Geospatial commands: `GEOADD`, `GEODIST`, `GEORADIUS`, `GEOSEARCH`. Internally stored as sorted sets with geohash-encoded scores. Use for: nearby location queries, distance calculations. Limitation: Earth is treated as a perfect sphere (minor inaccuracy).

**RD-20.** When NOT to use Redis: dataset doesn't fit in RAM (expensive), need complex queries/joins (use RDBMS), need strong persistence guarantees (use PostgreSQL), need full-text search (use Elasticsearch), data relationships are complex (use a real database as primary store).

---

# Part F — Cassandra Questions

### CS-1. Explain the write path in detail.

```
Client → Coordinator (any node) → Determine replicas via partitioner
  │
  ├─ 1. Write to COMMIT LOG (sequential append, durability)
  ├─ 2. Write to MEMTABLE (in-memory sorted map per table)
  ├─ 3. Return ACK to coordinator
  │
  (Later, background:)
  ├─ 4. Memtable full → flush to SSTABLE on disk
  │     (immutable, sorted by partition key + clustering key)
  ├─ 5. Compaction merges SSTables, removes duplicates/tombstones
  └─ 6. Commit log segment discarded once its memtable is flushed
```

### CS-2. How does Cassandra's gossip protocol work?

Every second, each node randomly picks another node and exchanges state information (node status, load, schema version, token ranges). Information propagates exponentially — the whole cluster converges quickly. Gossip detects node failures (no heartbeat response), handles cluster membership, and distributes schema changes. Uses a versioned state with generation number and heartbeat counter to determine freshness.

### CS-3. Explain tombstones and their dangers.

A DELETE doesn't remove data — it writes a "tombstone" marker (a special record with a timestamp). The tombstone must persist for `gc_grace_seconds` (default 10 days) to propagate to all replicas (including ones that were temporarily down). After `gc_grace_seconds`, compaction can remove the tombstone and the original data. Danger: if you do many deletes, tombstones accumulate, causing slow reads (Cassandra must scan through tombstones). If you read more than `tombstone_warn_threshold` (default 1000) tombstones in one query, you get a warning; exceeding `tombstone_failure_threshold` (default 100,000) causes the query to fail.

### CS-4. What is the partition key vs clustering key?

```
PRIMARY KEY ((partition_key), clustering_key1, clustering_key2)

Partition Key:
  • Determines WHICH NODE stores the data (via consistent hashing)
  • All rows with same partition key are on the same node
  • Used for equality queries (WHERE partition_key = X)
  • Should have high cardinality for even distribution

Clustering Key:
  • Determines SORT ORDER within a partition
  • Enables range queries within a partition
  • WHERE partition_key = X AND clustering_key > Y works!

Example: Time-series sensor data
  PRIMARY KEY ((sensor_id), timestamp)
  
  sensor_id=A:
    timestamp=10:00 → reading=25.1
    timestamp=10:01 → reading=25.3
    timestamp=10:02 → reading=25.0
  
  Query: "Last 10 readings for sensor A"
  → Single partition read, data already sorted by timestamp ✅
```

### CS-5. Explain Lightweight Transactions (LWT).

Cassandra's version of compare-and-swap using Paxos consensus. Syntax: `INSERT ... IF NOT EXISTS` or `UPDATE ... IF column = value`. Provides linearizable consistency for a single partition. Internally: 4 round trips (prepare, promise, propose, commit) vs 1 for a normal write. ~4x slower than regular writes. Use sparingly: user registration (IF NOT EXISTS), inventory decrements (IF quantity >= X). Not a substitute for general ACID transactions.

### CS-6 to CS-20. (Quick-fire)

**CS-6.** Snitch: tells Cassandra the network topology (which nodes are in which rack/DC). Types: SimpleSnitch (single DC), GossipingPropertyFileSnitch (production, reads from local config), EC2Snitch (AWS). Critical for rack/DC-aware replication placement.

**CS-7.** Repair: `nodetool repair` synchronizes data between replicas using Merkle trees (hash trees). Compares tree hashes to find diverged ranges, then streams only the differing data. Must run within `gc_grace_seconds` to prevent resurrected deletes. Incremental repair (since 2.1) repairs only SSTables not yet repaired.

**CS-8.** Compaction strategies deep dive: STCS = merge similar-sized SSTables (write-optimized, space penalty). LCS = maintain sorted non-overlapping levels (read-optimized, write penalty). TWCS = group SSTables by time window, compact within windows (ideal for time-series with TTL). Choose based on workload: write-heavy → STCS, read-heavy → LCS, time-series → TWCS.

**CS-9.** `nodetool` essential commands: `status` (cluster overview), `info` (node details), `cfstats` (table statistics), `tpstats` (thread pool stats), `compactionstats` (active compactions), `proxyhistograms` (coordinator latency), `tablehistograms` (table-level latency).

**CS-10.** Secondary indexes in Cassandra: local indexes (each node indexes its own data). Query on secondary index without partition key → scatter-gather across ALL nodes (expensive!). Use only for: low-cardinality columns queried alongside partition key. Alternative: materialized views or manual denormalization.

**CS-11.** Materialized Views: automatically maintained denormalized tables with different partition keys. Write to base table → Cassandra updates the view. Limitations: eventually consistent (not atomic with base write), performance overhead, have been buggy historically. Many teams prefer manual denormalization.

**CS-12.** Batch statements: `BEGIN BATCH ... APPLY BATCH`. Logged batch (default): atomic across multiple partitions (uses a batch log). Unlogged batch: atomic only within a single partition. Counter batch: for counter updates only. Logged batches are expensive — use only when atomicity across partitions is needed. Not a performance optimization tool (common misconception).

**CS-13.** Hinted handoff details: when a write target is down, the coordinator stores the mutation as a "hint" on its own node. Hints are replayed when the target node comes back online. `max_hint_window_in_ms` (default 3 hours) limits how long hints are stored. Hints are best-effort, not guaranteed delivery — long outages still require repair.

**CS-14.** Read repair: on a read, coordinator requests data from R replicas. If responses differ, the most recent version (by timestamp) wins and is sent back to the stale replica. Types: blocking (synchronous, default for QUORUM) and background (probabilistic, `read_repair_chance` — deprecated in newer versions in favor of `dclocal_read_repair_chance`).

**CS-15.** Counters: special column type for distributed counters. Not idempotent — can't safely retry counter updates. Stored separately from regular data. Performance: slower than regular writes (requires read-before-write internally). Use for: page views, like counts. For exact counts, consider an RDBMS.

**CS-16.** Virtual nodes (vnodes): instead of each physical node owning one token range, each node owns many (default 256) small ranges. Benefits: more even data distribution, faster rebalancing when nodes join/leave. Drawback: repair is slower (more ranges to process). Newer recommendation: use fewer vnodes (16-32) or single-token assignments with careful capacity planning.

**CS-17.** SSTable structure: Data file (sorted key-value data), Primary Index (partition key → data file offset), Summary (sampled index entries for faster lookup), Bloom Filter (probabilistic membership), Compression Offset Map (for compressed SSTables), Statistics (min/max values, tombstone count).

**CS-18.** Multi-DC replication: configure `NetworkTopologyStrategy` with RF per DC. Example: `WITH REPLICATION = {'class': 'NetworkTopologyStrategy', 'us-east': 3, 'eu-west': 3}`. Use `LOCAL_QUORUM` for queries to avoid cross-DC latency. Data is asynchronously replicated across DCs. Useful for: disaster recovery, geo-locality, regulatory compliance.

**CS-19.** `SELECT * FROM table`: dangerous in Cassandra! Without a WHERE clause on partition key, this triggers a full cluster scan (every node, every SSTable). Always specify partition key in WHERE clause. For analytics across all data, use Spark-Cassandra connector.

**CS-20.** When NOT to use Cassandra: small dataset (< 100GB — use PostgreSQL), need complex queries/joins/aggregations, need strong multi-partition transactions, rapidly changing query patterns (data model is tied to queries), need to update/delete frequently (tombstone overhead).

---

# Part G — Elasticsearch Questions

### ES-1. How does indexing work internally?

Document → analyze text fields (tokenize, lowercase, stem) → add to inverted index per field → store in an in-memory buffer → refresh (default 1s) creates a new immutable Lucene segment → translog ensures durability between refreshes → background merge combines small segments into larger ones.

### ES-2. What is the difference between `text` and `keyword` types?

`text`: analyzed — broken into tokens by an analyzer. Used for full-text search. Not usable for exact matches, sorting, or aggregations (use `.keyword` sub-field). `keyword`: not analyzed — stored as-is. Used for exact matches, filtering, sorting, aggregations. Common pattern: map a field as both `text` (for search) and `keyword` (for filtering/sorting) using multi-fields.

### ES-3. Explain the relevance scoring model (BM25).

BM25 considers: (a) term frequency — how often the term appears in the document (more = more relevant, with diminishing returns), (b) inverse document frequency — how rare the term is across all documents (rarer = more relevant), (c) field length — shorter fields with the term score higher. Replaced TF-IDF as default in ES 5.0. `_score` is calculated per query per document. Use `explain: true` in the query to see the score breakdown.

### ES-4. How does shard allocation work?

Primary shards set at index creation (immutable without reindexing). Replica shards configurable anytime. ES distributes shards across nodes for balance, ensuring primary and replica of the same shard are on different nodes. Shard awareness: rack/zone-aware allocation prevents primary and replica from being in the same failure domain. Too many shards per node → overhead; too few → can't distribute load. Rule of thumb: 20-40GB per shard, <20 shards per GB of heap.

### ES-5 to ES-15. (Quick-fire)

**ES-5.** Segment merging: Lucene segments are immutable. New data → new segments. Background merge combines segments (like LSM compaction). Force merge (`_forcemerge`) reduces segments for read-heavy indexes. Don't force merge on actively written indexes.

**ES-6.** Index lifecycle management (ILM): automate index rollover (create new index when old reaches size/age/doc count), then transition through phases: hot → warm → cold → frozen → delete. Essential for time-series data (logs, metrics).

**ES-7.** Nested vs parent-child: Nested = stored in same Lucene document, queried atomically. Fast but expensive to update (entire document reindexed). Parent-child = separate documents linked by join field. Slower queries but independent updates. Use nested for small, rarely-changed arrays; parent-child for large, frequently-updated relationships.

**ES-8.** `_bulk` API: batch multiple index/update/delete operations in one request. Much faster than individual operations (reduces HTTP overhead, allows ES to optimize). Format: newline-delimited JSON (NDJSON). Recommended batch size: 5-15MB.

**ES-9.** Cluster states: GREEN = all shards allocated. YELLOW = all primaries allocated but some replicas unassigned (common with single-node clusters). RED = some primary shards unassigned (data loss risk). Monitor with `_cluster/health`.

**ES-10.** Cross-cluster search: query multiple ES clusters from a single request. Useful for geo-distributed setups. Configured via cluster settings, queried with `cluster_name:index` syntax.

**ES-11.** `_reindex` API: copy data from one index to another. Use for: changing mappings, changing shard count, index migration. Can transform documents with a script during reindex. Supports cross-cluster reindexing.

**ES-12.** Analyzer anatomy: Character Filters (strip HTML, replace chars) → Tokenizer (split into tokens) → Token Filters (lowercase, stemming, synonyms, stop words). Custom analyzers combine these components for domain-specific search.

**ES-13.** Doc values vs fielddata: Doc values = on-disk columnar storage for sorting/aggregations on keyword/numeric fields (default, efficient). Fielddata = in-memory data structure for text fields (disabled by default — expensive, can cause OOM). If you need to aggregate on text, use a `.keyword` sub-field.

**ES-14.** Snapshot and restore: backup indexes to a repository (S3, GCS, HDFS, Azure). Incremental snapshots (only changed segments). Restore specific indexes or entire cluster. Essential for disaster recovery and migration.

**ES-15.** When NOT to use Elasticsearch: primary data store (no ACID), frequent updates to existing documents (expensive delete+reindex), strong consistency requirements (eventual consistency between shards), small dataset with simple queries (PostgreSQL full-text search is simpler), complex relational data (use RDBMS + ES for search only).

---

## Final Cheat Sheet: What to Study Per Interview Type

```
┌──────────────────────┬────────────────────────────────────────────┐
│ Interview Type       │ Focus Areas                                │
├──────────────────────┼────────────────────────────────────────────┤
│ Backend Engineer     │ ACID, indexing, query optimization,        │
│                      │ connection pooling, caching patterns,      │
│                      │ PostgreSQL or MySQL internals              │
├──────────────────────┼────────────────────────────────────────────┤
│ System Design        │ Sharding, replication, CAP, consistency    │
│                      │ patterns, caching, polyglot persistence,   │
│                      │ DB selection decision framework            │
├──────────────────────┼────────────────────────────────────────────┤
│ Database Engineer    │ Storage engines (B-tree/LSM), WAL/ARIES,   │
│ / DBA               │ MVCC internals, VACUUM/purge, buffer pool, │
│                      │ backup/recovery, replication topologies    │
├──────────────────────┼────────────────────────────────────────────┤
│ Data Engineer        │ CDC, partitioning, ETL patterns, column    │
│                      │ stores, Elasticsearch, data modeling       │
├──────────────────────┼────────────────────────────────────────────┤
│ SRE / Platform       │ Failover, monitoring, connection pooling,  │
│                      │ backup strategies, capacity planning,      │
│                      │ incident response playbooks                │
└──────────────────────┴────────────────────────────────────────────┘
```

---

*Companion document to "Database Internals: 0 to 100 Interview Guide (with Diagrams)"*
*Last updated: March 2026*
