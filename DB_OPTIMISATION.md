# SQL Optimization — Complete Guide

> A ground-up reference covering every concept from the learning roadmap.  
> Read linearly or jump to any section. Every concept includes the *why*, not just the *what*.

---

## Table of Contents

1. [How Databases Read Data](#1-how-databases-read-data)
2. [Indexes](#2-indexes)
3. [Query Patterns — Slow vs Fast](#3-query-patterns--slow-vs-fast)
4. [Reading EXPLAIN / EXPLAIN ANALYZE](#4-reading-explain--explain-analyze)
5. [Composite Indexes](#5-composite-indexes)
6. [JOIN Strategies](#6-join-strategies)
7. [CTEs vs Subqueries vs Temp Tables](#7-ctes-vs-subqueries-vs-temp-tables)
8. [Table Partitioning](#8-table-partitioning)
9. [Caching Strategies](#9-caching-strategies)
10. [Deadlocks](#10-deadlocks)
11. [Window Functions](#11-window-functions)
12. [Database Statistics & ANALYZE](#12-database-statistics--analyze)
13. [Connection Pooling](#13-connection-pooling)
14. [Read Replicas & Sharding](#14-read-replicas--sharding)
15. [Transaction Isolation Levels](#15-transaction-isolation-levels)
16. [Full-Text Search](#16-full-text-search)
17. [Interview Scenarios — Real Stories](#17-interview-scenarios--real-stories)
18. [Interview Checklist](#18-interview-checklist)

---

## 1. How Databases Read Data

### Full table scan
Without an index, the DB reads **every row on disk** to find matches — even if only 1 row qualifies.

```sql
-- Reads ALL 12M rows to find one customer
SELECT * FROM transactions WHERE customer_id = 42;
```

### Pages and the buffer pool
- Data is stored in fixed-size **pages** (typically 8KB in PostgreSQL, 16KB in MySQL)
- The DB caches hot pages in memory — the **buffer pool**
- Hot pages = fast reads. Cold pages on disk = slow I/O
- More pages read = more cost. The query planner thinks in pages, not rows

### The query planner and cost model
The planner chooses between execution strategies by estimating **cost**:
- Number of rows to scan (selectivity)
- Whether an index exists
- Memory available for sorting/joining
- Statistics about data distribution (column histograms)

The planner can be wrong when statistics are stale — always run `ANALYZE` after large data loads.

---

## 2. Indexes

### What an index does
An index is a separate, sorted data structure that lets the DB find rows without scanning the whole table. Think of it as the index at the back of a book.

Without index: `O(n)` — scan every row  
With index: `O(log n)` — jump directly to the match

### B-tree index (default)
The standard index type. Works for equality (`=`), ranges (`>`, `<`, `BETWEEN`), and sorting (`ORDER BY`).

```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Now uses the index: 3 page reads vs 50,000
SELECT * FROM orders WHERE customer_id = 42;
```

### Covering index
If the index contains **all columns the query needs**, the DB never touches the actual table — called an **index-only scan**.

```sql
CREATE INDEX idx_orders_covering
  ON orders(customer_id)
  INCLUDE (order_date, total);

-- Index-only scan — never reads the table heap
SELECT order_date, total FROM orders WHERE customer_id = 42;
```

### Hash index
Only supports equality (`=`). Faster than B-tree for pure lookups, but can't handle ranges or sorting. Rarely the right choice — B-tree covers most cases.

### Partial index
Index only a subset of rows. Smaller, faster, and often more useful than a full index.

```sql
-- Only index active users — much smaller than indexing all users
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```

### When indexes hurt
- Slow down `INSERT`, `UPDATE`, `DELETE` — each write must update the index too
- Low-cardinality columns (boolean, status with 2–3 values) — DB often ignores the index and scans anyway
- Too many indexes waste disk and memory, and confuse the planner
- Functions applied to indexed columns break the index (see below)

### The function trap — extremely common mistake

```sql
-- BREAKS the index on email
WHERE LOWER(email) = 'user@example.com'

-- USES the index
WHERE email = 'user@example.com'   -- store data in consistent case

-- BREAKS the index on created_at
WHERE YEAR(created_at) = 2024

-- USES the index
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'
```

**Rule:** Never wrap an indexed column in a function in a `WHERE` clause. Transform the value you're comparing against instead.

---

## 3. Query Patterns — Slow vs Fast

### SELECT * — always avoid

```sql
-- Bad: transfers all columns, prevents covering index
SELECT * FROM users WHERE id = 1;

-- Good: fetch only what you need
SELECT id, name, email FROM users WHERE id = 1;
```

### N+1 query problem

```sql
-- Bad: 1 query to get orders + 1 query per order = N+1 total
SELECT id FROM orders;
-- then in application code for each order:
SELECT * FROM items WHERE order_id = ?;

-- Good: 1 query with a JOIN
SELECT o.id, i.name, i.quantity
FROM orders o
JOIN items i ON i.order_id = o.id;
```

### Pagination — OFFSET vs keyset

```sql
-- Bad: OFFSET 100000 reads and discards 100K rows every time
SELECT * FROM posts ORDER BY id LIMIT 20 OFFSET 100000;

-- Good: keyset pagination — jumps directly to the right place
SELECT * FROM posts
WHERE id > 100020          -- last seen id from previous page
ORDER BY id
LIMIT 20;
```

### EXISTS vs COUNT for existence checks

```sql
-- Bad: scans all matching rows to count them
SELECT COUNT(*) > 0 FROM orders WHERE user_id = 5;

-- Good: stops at the first match
SELECT EXISTS (
  SELECT 1 FROM orders WHERE user_id = 5
);
```

### OR on indexed columns

```sql
-- Bad: OR prevents index use on both columns
SELECT * FROM users WHERE first_name = 'John' OR last_name = 'Smith';

-- Good: UNION ALL uses index on each branch separately
SELECT * FROM users WHERE first_name = 'John'
UNION ALL
SELECT * FROM users WHERE last_name = 'Smith';
```

### Implicit type conversion

```sql
-- Bad: user_id is INTEGER but we pass a string — type cast breaks index
WHERE user_id = '42'

-- Good: match the column type
WHERE user_id = 42
```

### Wildcard LIKE patterns

```sql
-- Bad: leading wildcard = full table scan, index cannot be used
WHERE name LIKE '%john%'

-- Acceptable: trailing wildcard CAN use a B-tree index
WHERE name LIKE 'john%'

-- Best for arbitrary substring search: use trigram index (see Section 16)
```

---

## 4. Reading EXPLAIN / EXPLAIN ANALYZE

### Basic usage

```sql
-- EXPLAIN shows the plan the DB *would* use (doesn't run the query)
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- EXPLAIN ANALYZE actually runs the query and shows real timings
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;

-- Most useful form in PostgreSQL
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
```

### Key nodes to understand

| Node | Meaning |
|------|---------|
| `Seq Scan` | Full table scan — usually bad on large tables |
| `Index Scan` | Used an index, but still fetches from table heap |
| `Index Only Scan` | Used a covering index — best case |
| `Bitmap Heap Scan` | Used index to build a bitmap, then fetched pages — good for range queries |
| `Nested Loop` | For each row in outer, find matching row in inner — good for small datasets |
| `Hash Join` | Builds a hash table from one side, probes with the other — good for large datasets |
| `Merge Join` | Joins two pre-sorted streams — efficient when both sides are already sorted |
| `Sort` | Explicit sort — check if an index could avoid this |
| `Aggregate` | GROUP BY, COUNT, SUM etc. |

### Red flags in a query plan

- `Seq Scan` on a large table → missing index
- `rows=1` estimate but `actual rows=1000000` → stale statistics, run `ANALYZE`
- `Sort` with large memory usage → add index or increase `work_mem`
- Very high `actual time` on one node → that's your bottleneck
- `Loops: 10000` on a Nested Loop → consider restructuring the query

### Sample healthy plan output

```
Index Only Scan on orders  (cost=0.43..8.45 rows=3 width=24)
  Index Cond: (customer_id = 42)
  Heap Fetches: 0
  Actual rows: 3   Loops: 1
Planning time: 0.1 ms
Execution time: 0.08 ms
```

`Heap Fetches: 0` = covering index used, never touched the table.

---

## 5. Composite Indexes

### What they are
An index on **multiple columns**. Column order matters enormously.

```sql
CREATE INDEX idx_orders_status_date ON orders(status, created_at);
```

### The left-prefix rule
A composite index on `(a, b, c)` can be used for queries filtering on:
- `a`
- `a, b`
- `a, b, c`

It **cannot** be used for queries filtering only on `b` or `c` alone.

```sql
-- Index on (status, created_at)

-- USES the index (left prefix: status)
WHERE status = 'completed'

-- USES the index (both columns)
WHERE status = 'completed' AND created_at > '2024-01-01'

-- CANNOT use the index (skips status, the leftmost column)
WHERE created_at > '2024-01-01'
```

### Column ordering strategy
Put the most **selective** column first (the one that filters out the most rows), unless a leading equality filter makes another order better.

```sql
-- Good: status has high cardinality, filters heavily
CREATE INDEX ON orders(status, customer_id, created_at);

-- Query pattern that drives the decision
WHERE status = 'completed' AND customer_id = 42 ORDER BY created_at;
```

### Covering with composite indexes

```sql
-- Index covers the entire query — index-only scan
CREATE INDEX idx_covering
  ON orders(customer_id, status)
  INCLUDE (total, created_at);

SELECT total, created_at
FROM orders
WHERE customer_id = 42 AND status = 'completed';
```

---

## 6. JOIN Strategies

The query planner automatically picks a join strategy, but understanding them helps you write queries the planner can optimize.

### Nested Loop Join
For each row in the outer table, look up matching rows in the inner table.

- **Best when:** outer table is small, inner table has an index
- **Worst when:** both tables are large — degrades to O(n×m)

```sql
-- Works well: users is small (1K rows), orders has index on user_id
SELECT u.name, o.total
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.plan = 'enterprise';
```

### Hash Join
Build a hash table from the smaller table, then probe it with each row from the larger table.

- **Best when:** one table is significantly smaller, no useful index
- **Memory-intensive** — spills to disk if `work_mem` is exceeded

### Merge Join
Sort both tables on the join key, then merge them like a zipper.

- **Best when:** both tables are already sorted (e.g., indexed on the join key)
- Very efficient for large-to-large joins on sorted data

### Filter before you join — always

```sql
-- Bad: joins all rows, then filters
SELECT u.name, o.total
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed' AND o.created_at > '2024-01-01';

-- Good: filter in a subquery first, then join smaller result
SELECT u.name, recent.total
FROM users u
JOIN (
  SELECT user_id, total
  FROM orders
  WHERE status = 'completed' AND created_at > '2024-01-01'
) recent ON recent.user_id = u.id;
```

### JOIN order matters
The planner usually figures this out, but for complex queries: join smaller/more-filtered tables first. The first join should produce the smallest intermediate result.

---

## 7. CTEs vs Subqueries vs Temp Tables

### Subqueries
Inline, no name, used directly in the query.

```sql
SELECT name FROM users
WHERE id IN (
  SELECT user_id FROM orders WHERE total > 1000
);
```

**Good for:** simple filters, EXISTS checks  
**Watch out for:** correlated subqueries (runs once per outer row — very slow)

```sql
-- Correlated subquery: runs the inner SELECT for EVERY user row
SELECT name FROM users u
WHERE (SELECT COUNT(*) FROM orders WHERE user_id = u.id) > 5;

-- Better: JOIN + GROUP BY
SELECT u.name FROM users u
JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name
HAVING COUNT(*) > 5;
```

### CTEs (Common Table Expressions)
Named, readable, reusable within the query.

```sql
WITH high_value_orders AS (
  SELECT user_id, SUM(total) as revenue
  FROM orders
  WHERE created_at > '2024-01-01'
  GROUP BY user_id
),
top_users AS (
  SELECT user_id FROM high_value_orders WHERE revenue > 10000
)
SELECT name FROM users WHERE id IN (SELECT user_id FROM top_users);
```

**Good for:** breaking complex queries into readable steps, reusing results  
**PostgreSQL note:** CTEs are an **optimization fence** in older versions (pre-12) — the planner can't push filters into them. In PostgreSQL 12+, non-recursive CTEs are inlined by default.

### Temp tables
Actually materializes the result to disk/memory. Useful when you need to query the intermediate result multiple times.

```sql
CREATE TEMP TABLE recent_orders AS
  SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '30 days';

CREATE INDEX ON recent_orders(user_id);  -- can index a temp table!

-- Now use it multiple times
SELECT * FROM recent_orders WHERE user_id = 42;
SELECT COUNT(*) FROM recent_orders GROUP BY status;

DROP TABLE recent_orders;
```

**Good for:** complex multi-step queries, running multiple aggregations on the same filtered dataset  
**Watch out for:** overhead of materialization — only worth it for large intermediate results used multiple times

### Quick comparison

| | Subquery | CTE | Temp Table |
|---|---|---|---|
| Reusable in query | No | Yes | Yes |
| Indexed | No | No | Yes |
| Materialized | Sometimes | Sometimes | Always |
| Best for | Simple filters | Readable complex logic | Multi-step, reused data |

---

## 8. Table Partitioning

### What it is
Splitting a large table into smaller physical pieces (partitions) while keeping a single logical table interface. Queries that filter on the partition key only touch relevant partitions — called **partition pruning**.

### Range partitioning
Split by a range of values — most common for time-series data.

```sql
CREATE TABLE orders (
  id BIGINT,
  created_at TIMESTAMP,
  total NUMERIC
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2023 PARTITION OF orders
  FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- This query only scans orders_2024 — other partitions ignored
SELECT * FROM orders WHERE created_at > '2024-06-01';
```

### List partitioning
Split by discrete values.

```sql
CREATE TABLE users PARTITION BY LIST (region);

CREATE TABLE users_india  PARTITION OF users FOR VALUES IN ('IN');
CREATE TABLE users_us     PARTITION OF users FOR VALUES IN ('US');
CREATE TABLE users_europe PARTITION OF users FOR VALUES IN ('DE', 'FR', 'UK');
```

### Hash partitioning
Distribute rows evenly by hashing the partition key. Good for even load distribution without a natural range.

```sql
CREATE TABLE events PARTITION BY HASH (user_id);

CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... etc
```

### When to partition
- Tables over ~100M rows where queries always filter on one column (date, region, tenant_id)
- You need to drop old data quickly (`DROP TABLE orders_2020` is instant vs deleting millions of rows)
- You want to place hot and cold data on different storage tiers

### Partitioning pitfalls
- Queries that don't filter on the partition key scan **all** partitions — potentially worse than unpartitioned
- Joins across partitions are more complex for the planner
- Foreign keys to/from partitioned tables have limitations

---

## 9. Caching Strategies

### Query result cache
Some databases (MySQL) have a built-in query cache. PostgreSQL does not — you implement caching at the application layer.

### Materialized views
Pre-computed query results stored as a physical table. Fast to read, but must be refreshed.

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
  SELECT
    DATE_TRUNC('month', created_at) as month,
    SUM(total) as revenue
  FROM orders
  WHERE status = 'completed'
  GROUP BY 1;

-- Refresh (blocks reads in basic form)
REFRESH MATERIALIZED VIEW monthly_revenue;

-- Refresh without blocking reads (PostgreSQL)
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

**Good for:** dashboards, reports, expensive aggregations that don't need real-time data  
**Watch out for:** refresh frequency — stale data, and concurrent refresh requires a unique index

### Application-level caching (Redis)
Cache query results in Redis with a TTL. Pattern:

```
1. Check Redis for key
2. If hit: return cached result
3. If miss: run SQL query, store in Redis, return result
```

```python
cache_key = f"user:{user_id}:orders"
cached = redis.get(cache_key)
if cached:
    return json.loads(cached)

result = db.query("SELECT * FROM orders WHERE user_id = %s", user_id)
redis.setex(cache_key, 300, json.dumps(result))  # 300s TTL
return result
```

**Cache invalidation** is the hard part — invalidate on write, or use short TTLs for data that changes frequently.

### PostgreSQL buffer cache
Warm frequently accessed data into the buffer cache using `pg_prewarm`:

```sql
-- Pre-load a table's pages into the buffer pool
SELECT pg_prewarm('orders');
```

---

## 10. Deadlocks

### What is a deadlock
Two transactions each hold a lock the other needs, and both wait forever.

```
Transaction A: locks row 1, waits for row 2
Transaction B: locks row 2, waits for row 1
→ Deadlock
```

### How databases handle it
The DB detects the cycle and kills one transaction (the "victim"), returning an error to the application. The killed transaction must be retried.

### Detecting deadlocks in PostgreSQL

```sql
-- See current locks
SELECT * FROM pg_locks;

-- See blocking queries
SELECT
  blocked.pid,
  blocked.query,
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

### Preventing deadlocks

**1. Always acquire locks in the same order**

```sql
-- Bad: Transaction A locks user 1 then order 5
--      Transaction B locks order 5 then user 1  → deadlock
-- Good: always lock users before orders, consistently
```

**2. Keep transactions short**

```sql
-- Bad: long transaction holds locks for seconds
BEGIN;
  -- complex computation
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  -- ... more work ...
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Good: compute outside the transaction, lock briefly
-- (do computation here)
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**3. Use SELECT FOR UPDATE to declare intent early**

```sql
BEGIN;
-- Lock both rows upfront before doing any work
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Now safe to update — no other transaction can lock these rows
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**4. Use advisory locks for application-level coordination**

```sql
-- Acquire a named advisory lock (non-blocking version)
SELECT pg_try_advisory_lock(12345);
-- Do work
SELECT pg_advisory_unlock(12345);
```

---

## 11. Window Functions

### What they are
Perform calculations across a set of rows **related to the current row**, without collapsing rows like `GROUP BY` does.

```sql
SELECT
  user_id,
  order_date,
  total,
  SUM(total) OVER (PARTITION BY user_id ORDER BY order_date) AS running_total
FROM orders;
```

### Common window functions

```sql
-- Rank rows within a partition
ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)
RANK()        -- same rank for ties, gaps in sequence
DENSE_RANK()  -- same rank for ties, no gaps

-- Access other rows
LAG(salary, 1)  OVER (ORDER BY hire_date)  -- previous row's value
LEAD(salary, 1) OVER (ORDER BY hire_date)  -- next row's value

-- Aggregates as windows
SUM(total)   OVER (PARTITION BY user_id)
AVG(total)   OVER (PARTITION BY user_id ORDER BY created_at ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
COUNT(*)     OVER ()  -- count across all rows
```

### Performance tips for window functions

```sql
-- Bad: window function on entire table, then filter
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
  FROM orders
) t WHERE rn = 1;

-- Good: filter first in a CTE, then apply window function to smaller set
WITH recent AS (
  SELECT * FROM orders WHERE created_at > '2024-01-01'
)
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
  FROM recent
) t WHERE rn = 1;
```

An index on `(user_id, created_at DESC)` can make the `PARTITION BY ... ORDER BY` very fast.

---

## 12. Database Statistics & ANALYZE

### What statistics are
The query planner doesn't know the exact data — it uses **statistics** to estimate row counts and choose plans. Statistics include:
- Row count per table
- Column value histograms (distribution of values)
- Most common values
- NULL fraction

### When statistics go stale
After large `INSERT`, `UPDATE`, or `DELETE` operations, statistics may be outdated. The planner uses old row count estimates, picks wrong join strategies, and queries slow down.

```sql
-- Manually update statistics for a table
ANALYZE orders;

-- Update all tables in the database
ANALYZE;

-- See current statistics
SELECT * FROM pg_stats WHERE tablename = 'orders';
```

### Autovacuum and autoanalyze
PostgreSQL runs `ANALYZE` automatically via the **autovacuum** daemon. Default trigger: after 20% of rows change. For very large tables, 20% = millions of rows — statistics lag behind. Tune the thresholds:

```sql
-- For a 100M row table, trigger ANALYZE after 100K changes (0.1%)
ALTER TABLE orders SET (
  autovacuum_analyze_scale_factor = 0.001,
  autovacuum_analyze_threshold = 100000
);
```

### VACUUM and bloat
`DELETE` and `UPDATE` in PostgreSQL don't physically remove rows — they mark them as dead. `VACUUM` reclaims space. Without regular vacuuming, tables accumulate **bloat** — dead rows that slow down scans.

```sql
-- Reclaim space from dead rows
VACUUM orders;

-- Reclaim space AND update statistics
VACUUM ANALYZE orders;

-- Full vacuum — rewrites the table, reclaims all space (locks table!)
VACUUM FULL orders;
```

---

## 13. Connection Pooling

### The problem
Opening a new database connection is expensive — it takes 10–50ms and consumes server memory. At scale, thousands of concurrent connections overwhelm the DB.

### How pooling works
A pool maintains a set of pre-opened connections. Application requests borrow a connection, use it, and return it — no open/close overhead.

### PgBouncer (PostgreSQL)
The most popular PostgreSQL connection pooler. Three modes:

| Mode | Behaviour | Best for |
|------|-----------|---------|
| Session | Connection held for the entire client session | Apps that use session-level features |
| Transaction | Connection returned after each transaction | Most web apps — recommended default |
| Statement | Connection returned after each statement | Rare — breaks multi-statement transactions |

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20   # only 20 real DB connections needed
```

### Impact on query performance
With pooling you can handle 10,000 concurrent app connections with only 20–50 real DB connections — the DB is never overwhelmed. Without pooling, 1,000 concurrent connections each running a slow query = DB memory exhausted.

---

## 14. Read Replicas & Sharding

### Read replicas
A replica is a copy of the primary database that receives all writes via replication and serves read queries. Useful when reads dominate (typical ratio: 80% reads, 20% writes).

```
           ┌──────────────┐
Writes ──► │   Primary    │ ──replication──► Replica 1
           └──────────────┘                 Replica 2
                                            Replica 3
Reads ──────────────────────────────────► (any replica)
```

**Application pattern:**
```python
# Write to primary
primary_db.execute("INSERT INTO orders ...")

# Read from replica
replica_db.execute("SELECT * FROM orders WHERE user_id = 42")
```

**Watch out for:** replication lag — replicas are slightly behind the primary. Don't read your own writes from a replica immediately after writing.

### Sharding
Split data across multiple independent databases by a **shard key** (e.g., user_id, tenant_id). Each shard holds a subset of the data.

```
user_id 1–1M    → Shard 1 (its own DB server)
user_id 1M–2M   → Shard 2
user_id 2M–3M   → Shard 3
```

**When to shard:** only when a single primary + replicas can no longer handle write throughput or storage. Sharding adds massive complexity — exhaust all other options first.

**Sharding pitfalls:**
- Cross-shard queries are slow and complex
- Rebalancing shards when data grows unevenly is painful
- Transactions across shards require distributed transaction protocols

---

## 15. Transaction Isolation Levels

### The problem: concurrent transactions interfere
Without isolation, multiple concurrent transactions can cause anomalies:

| Anomaly | Description |
|---------|-------------|
| Dirty read | Reading uncommitted data from another transaction |
| Non-repeatable read | Reading the same row twice gets different values |
| Phantom read | A query returns different rows when run twice in the same transaction |

### Isolation levels (weakest to strongest)

```sql
-- Set isolation level for a transaction
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

| Level | Dirty reads | Non-repeatable reads | Phantom reads | Performance |
|-------|-------------|---------------------|---------------|-------------|
| Read Uncommitted | Possible | Possible | Possible | Fastest |
| Read Committed | Prevented | Possible | Possible | Default in PG/MySQL |
| Repeatable Read | Prevented | Prevented | Possible | Moderate |
| Serializable | Prevented | Prevented | Prevented | Slowest |

### PostgreSQL defaults
PostgreSQL's default is **Read Committed** — the right choice for most applications. Use **Repeatable Read** for financial calculations. Use **Serializable** only when correctness is more important than performance (e.g., generating unique invoice numbers).

### MVCC — how PostgreSQL avoids locks for reads
PostgreSQL uses **Multi-Version Concurrency Control** — reads never block writes, and writes never block reads. Each transaction sees a snapshot of the data as of when it started. This is why PostgreSQL reads are so fast even under write load.

---

## 16. Full-Text Search

### Why LIKE is insufficient

```sql
-- Can't use a B-tree index (leading wildcard)
WHERE body LIKE '%database optimization%'

-- Even trigram index struggles with very common words
```

### PostgreSQL full-text search

```sql
-- Create a tsvector column (pre-computed search vector)
ALTER TABLE articles ADD COLUMN search_vector tsvector;

UPDATE articles
SET search_vector = to_tsvector('english', title || ' ' || body);

-- Index it
CREATE INDEX idx_articles_fts ON articles USING gin(search_vector);

-- Search
SELECT title FROM articles
WHERE search_vector @@ to_tsquery('english', 'database & optimization');
```

**Ranking results:**

```sql
SELECT title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'database & optimization') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### Trigram index for fuzzy/substring search

```sql
-- Enable extension
CREATE EXTENSION pg_trgm;

-- Create trigram index
CREATE INDEX idx_users_name_trgm ON users USING gin(name gin_trgm_ops);

-- Now LIKE '%john%' uses the index
SELECT * FROM users WHERE name ILIKE '%john%';

-- Also supports similarity search
SELECT * FROM users WHERE similarity(name, 'johm') > 0.4;  -- typo-tolerant
```

### When to use Elasticsearch instead
Use Elasticsearch when you need:
- Relevance scoring with custom boosting
- Faceted search (filter by multiple dimensions simultaneously)
- Autocomplete / search-as-you-type
- Multi-language support
- Searching across millions of documents with sub-100ms response

Use PostgreSQL FTS when:
- Your data already lives in PostgreSQL
- Search is a secondary feature, not the core product
- You want to avoid maintaining a separate search service

---

## 17. Interview Scenarios — Real Stories

These are production stories following the structure: **problem → what I tried (including failures) → what worked → key lesson**.

---

### Scenario 1: Dashboard query timing out (12M rows)

**Context:** Admin dashboard, "total spend per user this month". Fine in dev (50K rows), 30s timeout in prod.

**Original query:**
```sql
SELECT user_id, SUM(amount) as total
FROM transactions
WHERE created_at >= '2024-01-01'
  AND status = 'completed'
GROUP BY user_id
ORDER BY total DESC;
```

**What I tried:**

1. `CREATE INDEX ON transactions(created_at)` → **18s. Not enough.** DB still scanned all January rows then filtered status.

2. `CREATE INDEX ON transactions(created_at, status)` → **8s. Better, but ORDER BY total still caused a filesort.**

3. Materialized view → **Refresh took 45s and blocked reads. Abandoned.**

4. Covering index + LIMIT:
```sql
CREATE INDEX idx_txn_covering
  ON transactions(created_at, status)
  INCLUDE (user_id, amount);

SELECT user_id, SUM(amount) AS total
FROM transactions
WHERE created_at >= '2024-01-01' AND status = 'completed'
GROUP BY user_id ORDER BY total DESC
LIMIT 100;  -- dashboard only shows top 100
```
→ **90ms. Index-only scan, never touched the table heap.**

**Key lesson:** A covering index eliminates table heap access entirely. Always ask "what columns does the query actually need to read?" — if the index holds them all, you get an index-only scan. Also: always add LIMIT if the consumer has a natural cap.

---

### Scenario 2: Search API at 4s response time

**Context:** Admin search endpoint — filter users by name, plan, signup date. 4–6s with all filters applied.

**Original query:**
```sql
SELECT u.*, p.*
FROM users u
JOIN user_profiles p ON p.user_id = u.id
WHERE LOWER(u.name) LIKE '%john%'
  AND u.plan IN ('pro', 'enterprise')
  AND u.created_at BETWEEN '2023-01-01' AND '2024-01-01';
```

**What I tried:**

1. `CREATE INDEX ON users(name)` → **Still 4s.** `LOWER()` wrapping + leading `%` both break index use independently.

2. Switched to `ILIKE` → **Still 4s.** ILIKE is case-insensitive natively, but leading `%` still prevents index use.

3. Trigram index:
```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_name_trgm ON users USING gin(name gin_trgm_ops);
```
→ **800ms.** Name search fixed, but `SELECT *` on the JOIN still pulling 50+ unused columns.

4. Trigram + filter-then-join + column selection:
```sql
SELECT u.id, u.name, u.email, u.plan, p.avatar_url, p.company
FROM (
  SELECT id, name, email, plan
  FROM users
  WHERE name ILIKE '%john%'
    AND plan IN ('pro', 'enterprise')
    AND created_at BETWEEN '2023-01-01' AND '2024-01-01'
  LIMIT 200
) u
JOIN user_profiles p ON p.user_id = u.id;
```
→ **120ms.**

**Key lesson:** Leading wildcards (`LIKE '%term%'`) need trigram indexes — B-trees can't help. Always filter before you join. `SELECT *` on a multi-table join is almost always wrong.

---

### Scenario 3: Nightly report running 45 minutes

**Context:** Revenue by product category for the last 30 days. Table sizes: orders (8M), order_items (40M), products (500K).

**What I tried:**

1. Added foreign key indexes (they were missing!) → **12 min.** Dropping from 45min to 12min just from missing FK indexes is sobering.

2. Replaced `COUNT(DISTINCT)` with CTE dedup → **9 min.** `COUNT(DISTINCT)` builds an in-memory hash of every value.

3. Pre-aggregate the large table before joining:
```sql
WITH order_revenue AS (
  SELECT order_id, SUM(quantity * unit_price) AS revenue
  FROM order_items
  GROUP BY order_id
),
recent AS (
  SELECT id FROM orders
  WHERE created_at >= NOW() - INTERVAL '30 days'
)
SELECT p.category, COUNT(r.id), SUM(or2.revenue)
FROM recent r
JOIN order_revenue or2 ON or2.order_id = r.id
JOIN order_items oi ON oi.order_id = r.id
JOIN products p ON p.id = oi.product_id
GROUP BY p.category;
```
→ **45 seconds.** 60× speedup.

**Key lesson:** Always aggregate the largest table as early as possible — before the join, not after. The original query joined 40M×8M rows before aggregating anything. Always check for missing FK indexes — a routine oversight with outsized impact.

---

### Scenario 4: Infinite scroll slowing down at depth

**Context:** Social app home feed. Page 1 = 80ms. Page 500 = 12s. Users noticed "the app gets slower as you scroll."

**Original:**
```sql
SELECT id, title, created_at FROM posts
WHERE user_id IN (SELECT followed_id FROM follows WHERE follower_id = 123)
ORDER BY created_at DESC
LIMIT 20 OFFSET 9980;
```

**What I tried:**

1. Composite index on `(user_id, created_at DESC)` → **Still slow at depth.** `OFFSET 9980` still discards 9,980 rows regardless of index.

2. Redis page cache → **Helped for hot pages, broke on new posts, complex to invalidate.**

3. Keyset pagination:
```sql
-- Client sends last_seen: created_at='2024-01-15 10:30:00', id=88234
SELECT id, title, created_at
FROM posts
WHERE user_id IN (SELECT followed_id FROM follows WHERE follower_id = 123)
AND (created_at, id) < ('2024-01-15 10:30:00', 88234)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```
→ **40ms at any scroll depth.** Constant time regardless of how deep you go.

**Key lesson:** `OFFSET N` always reads and discards N rows — it scales linearly with depth. Keyset pagination uses an indexed comparison to seek directly to the right position in `O(log n)`. The tradeoff: you lose the ability to jump to a specific page number, which is fine for infinite scroll.

---

## 18. Interview Checklist

Use this before any SQL/backend engineering interview.

### Concepts to be ready to explain

- [ ] What is a full table scan and when does it happen?
- [ ] How does a B-tree index work internally?
- [ ] What is a covering index / index-only scan?
- [ ] What breaks an index? (functions, leading wildcards, type mismatch)
- [ ] What is the left-prefix rule for composite indexes?
- [ ] How do you read an EXPLAIN ANALYZE output?
- [ ] What is keyset pagination and why is OFFSET slow?
- [ ] What is the N+1 problem and how do you fix it?
- [ ] What are the SQL JOIN strategies? When does each get chosen?
- [ ] What is a deadlock? How do you prevent one?
- [ ] What are transaction isolation levels? What's the default in PostgreSQL?
- [ ] What is partitioning? When would you use it?
- [ ] What are materialized views? Tradeoffs vs a cache?
- [ ] What is connection pooling? Why does it matter at scale?
- [ ] What is MVCC in PostgreSQL?
- [ ] When would you use Elasticsearch over PostgreSQL FTS?

### Things to mention unprompted (signals seniority)

- "The first thing I do is run `EXPLAIN ANALYZE` to see what the planner is actually doing"
- "I check for stale statistics — run `ANALYZE` after large data loads"
- "Indexes help reads but hurt writes — there's a tradeoff to weigh"
- "I always filter before I join — the join input should be as small as possible"
- "I look at the actual vs estimated rows in the plan — a big mismatch means stale stats"
- "For pagination, I prefer keyset over OFFSET for any table that grows"

### Narrative structure for scenario questions

When asked "tell me about a time you optimized a slow query":

1. **Context** — what the query did, table size, how slow
2. **First instinct** — what you tried first and why
3. **Why it failed or was insufficient**
4. **What you tried next** (repeat 2–3)
5. **The breakthrough** — what actually worked and the insight behind it
6. **The result** — specific numbers (30s → 90ms, 45min → 45s)
7. **What you learned** — the general principle you'd apply next time

---

*Generated as part of the SQL Optimization interactive course.*
