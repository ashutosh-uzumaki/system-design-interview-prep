# Chunk 23B — Cassandra Data Model + Tunable Consistency
### Phase 2 — Databases | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Hotstar, Uber

---

## The Fundamental Rule

```
PostgreSQL: schema drives queries
Cassandra:  queries drive schema

PostgreSQL:
→ Design tables first
→ Write any query later
→ JOINs handle relationships ✅

Cassandra:
→ Decide queries FIRST
→ Design table around queries
→ One query = one table (usually)
→ All data needed = in same table ✅
→ NO JOINs exist ❌
```

---

## Why No JOINs?

```
Cassandra = distributed across many nodes

JOIN requires:
→ Fetch data from Table A (maybe Node 1)
→ Fetch data from Table B (maybe Node 5)
→ Merge results

Cross-node operations = expensive network calls 😬
→ Cassandra avoids this entirely
→ Instead: store everything needed in one table ✅
→ Denormalization is intentional and encouraged
```

---

## Partition Key — Which Node

```
Partition Key = column that decides WHICH NODE stores data

All rows with same partition key:
→ stored on same node
→ fetched in single read ✅

How it maps to node:
node = hash(partition_key) % number_of_nodes

user_id = 123 → hash(123) % 6 = Node 3
user_id = 456 → hash(456) % 6 = Node 1

Always deterministic ✅
Always same node for same key ✅
```

### Why same node matters:

```
Without same-node storage:
user:123 data split across Node 1, 2, 3
→ Query needs 3 network calls 😬
→ Slow + expensive 😬

With same-node storage (partition key):
All user:123 data → Node 3
→ Single network call ✅
→ Fast ✅
```

---

## Clustering Key — Order Within Node

```
Clustering Key = column that sorts rows WITHIN a node

PRIMARY KEY (user_id, timestamp)
               ↑          ↑
         partition key   clustering key

Partition key  → decides WHICH node
Clustering key → decides ORDER within node
```

### Simple analogy:

```
user_id   = building number (which building/node)
timestamp = floor number (where within building)

Finding user:123's latest show:
→ Go to building user:123 (partition key → node)
→ Go to top floor (clustering key → latest timestamp)
→ Done ✅
```

### Why clustering key matters:

```
Without clustering key:
→ 1000 rows for user:123 stored randomly
→ "Get last 10 shows" → scan all 1000 rows 😬

With clustering key (timestamp DESC):
→ Rows sorted by timestamp on disk
→ "Get last 10 shows" → read last 10 directly ✅
→ Single sequential read ✅
```

---

## Real Table Design — Hotstar Watch History

```sql
CREATE TABLE watch_history (
    user_id    UUID,
    timestamp  TIMESTAMP,
    show_id    UUID,
    position   INT,
    PRIMARY KEY (user_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

```
Node layout:
─────────────────────────────────────────────────
Node 3 — ALL of user:123's data, sorted by time:

user_id | timestamp | show_id  | position
────────────────────────────────────────────
123     | 10:40PM   | show:999 | 8mins   ← latest
123     | 10:35PM   | show:789 | 12mins
123     | 10:30PM   | show:456 | 45mins
123     | 10:29PM   | show:234 | 22mins
```

```
This table efficiently answers:
✅ "Get all shows watched by user:123"
   WHERE user_id = 123

✅ "Get shows watched after 10PM"
   WHERE user_id = 123 AND timestamp > '10:00PM'

✅ "Get last 10 shows"
   WHERE user_id = 123 LIMIT 10

❌ "Get all users who watched show:456"
   WHERE show_id = 'show:456'
   → show_id not partition key → full cluster scan 😬
   → Need SEPARATE table for this query
```

---

## One Query = One Table

```
Same data stored in multiple tables for different queries:

Table 1 — query by user:
CREATE TABLE watch_history_by_user (
    user_id   UUID,
    timestamp TIMESTAMP,
    show_id   UUID,
    position  INT,
    PRIMARY KEY (user_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);

Table 2 — query by show:
CREATE TABLE watch_history_by_show (
    show_id   UUID,
    timestamp TIMESTAMP,
    user_id   UUID,
    position  INT,
    PRIMARY KEY (show_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);

Same data written TWICE:
→ watch_history_by_user  → query by user ✅
→ watch_history_by_show  → query by show ✅

This is INTENTIONAL denormalization ✅
Storage is cheap. Network calls are expensive.
```

---

## Real Example — Zepto Order History

```sql
-- Query 1: "Get all orders for user:123"
CREATE TABLE orders_by_user (
    user_id     UUID,
    timestamp   TIMESTAMP,
    order_id    UUID,
    amount      DECIMAL,
    status      TEXT,
    PRIMARY KEY (user_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Query 2: "Get all orders for merchant:M456"
CREATE TABLE orders_by_merchant (
    merchant_id UUID,
    timestamp   TIMESTAMP,
    order_id    UUID,
    amount      DECIMAL,
    status      TEXT,
    PRIMARY KEY (merchant_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

---

## Handling Many Queries — Practical Reality

### Not 10,000 tables for 10,000 queries:

```
Reality check:

High frequency queries → dedicated table ✅
Low frequency queries  → acceptable to scan
Analytics queries      → BigQuery/Spark (not Cassandra)
Similar queries        → one table handles multiple
```

### Pattern 1 — Similar queries share one table:

```
3 queries → ONE table:
Query A: "Get last 10 shows for user:123"
Query B: "Get shows after Jan 1 for user:123"
Query C: "Get all shows for user:123"

PRIMARY KEY (user_id, timestamp)
→ All 3 use user_id as partition key ✅
→ All 3 use timestamp range ✅
→ One table, three queries ✅
```

### Pattern 2 — Analytics → not Cassandra:

```
"Get top 10 most watched shows globally"
→ Not a Cassandra query
→ Too expensive regardless of table design
→ Use BigQuery / Spark ✅

Cassandra  → operational queries (real-time, per entity)
BigQuery   → analytical queries (global aggregations)
```

### Pattern 3 — Materialized Views:

```
Write to one table → Cassandra auto-maintains secondary:

CREATE MATERIALIZED VIEW show_viewers AS
    SELECT * FROM watch_history
    WHERE show_id IS NOT NULL
    PRIMARY KEY (show_id, timestamp, user_id);

Write to watch_history → show_viewers updated ✅
No duplicate write logic in app code ✅

Tradeoffs:
❌ Higher write amplification
❌ Eventual consistency between tables
→ Use sparingly, only for high-frequency secondary queries
```

### Pattern 4 — Time Bucketing for Hot Partitions:

```
Problem:
user:123 watched 50,000 shows over 5 years
→ All 50,000 rows on same node = hot partition 😬

Solution — composite partition key with time bucket:
PRIMARY KEY ((user_id, year_month), timestamp)

user:123, 2024-01 → Node 3 (Jan data)
user:123, 2024-02 → Node 7 (Feb data)
user:123, 2024-03 → Node 1 (Mar data)

→ Data spread across nodes ✅
→ No single hot partition ✅
→ Query: WHERE user_id=123 AND year_month='2024-01' ✅
```

### Reality in production:

```
10,000 query types → maybe 50-100 Cassandra tables
→ Similar queries grouped → one table
→ Analytics → BigQuery
→ Secondary patterns → Materialized Views
→ Hot partitions → bucketing
```

---

## The Golden Rules of Cassandra Data Modeling

```
1. Start with your queries
2. One query = one table (usually)
3. Partition key = what you filter by (WHERE clause)
4. Clustering key = how you sort (ORDER BY)
5. Denormalize freely — storage is cheap
6. Never rely on JOINs or secondary indexes
7. Group similar queries → share one table
8. Analytics → BigQuery, not Cassandra
```

---

---

# Tunable Consistency — Cassandra's Superpower

## The Problem With Fixed Consistency

```
PostgreSQL → always CP (consistency first, always)
DynamoDB   → always AP (availability first, always)

Cassandra  → YOU choose per operation ✅
```

---

## The 3 Consistency Levels

```
Setup: 3 nodes, each has replica of data

ONE:
→ Only 1 node must confirm write/read
→ Fastest ✅
→ Least consistent ❌
→ Other 2 nodes sync in background

QUORUM:
→ Majority must confirm: (3/2)+1 = 2 nodes
→ Balanced speed + consistency ✅
→ Tolerates 1 node failure ✅

ALL:
→ Every node must confirm
→ Slowest ❌
→ Most consistent ✅
→ One node down = write fails 😬
```

---

## Visualized:

```
Write: user:123 → show:456

ONE:
Node 1 ✅ → "Success!" ← returned here
Node 2    → syncing...
Node 3    → syncing...

QUORUM:
Node 1 ✅
Node 2 ✅ → "Success!" ← returned here
Node 3    → syncing...

ALL:
Node 1 ✅
Node 2 ✅
Node 3 ✅ → "Success!" ← returned here
```

---

## Real Use Cases:

```
ONE → Hotstar watch history
→ User pauses at 45 mins
→ One node confirms → success
→ Other nodes sync shortly after
→ User switches device → reads 40 mins instead of 45
→ 5 min difference = minor UX issue ✅
→ Speed matters more than perfect consistency

QUORUM → Razorpay payment status
→ Payment:123 = SUCCESS
→ 2 nodes confirm → success
→ User checks status → reads from any 2 nodes
→ At least one has latest data ✅
→ No double charge risk ✅

ALL → Critical config updates
→ Rarely used in practice
→ When absolutely cannot afford any staleness
→ One node down = operation fails 😬
```

---

## Why QUORUM Beats ALL for Fintech:

```
ALL:
→ All 3 nodes must confirm
→ One node slow/down → entire write blocked 😬
→ Availability suffers 😬

QUORUM (2 out of 3):
→ Only majority needed
→ One node down? Still works ✅
→ 2 nodes confirm = success ✅
→ Faster than ALL ✅
→ Still strong enough for payments ✅
```

---

## The Math That Makes QUORUM Work:

```
Replication factor = 3
Write QUORUM = 2
Read QUORUM  = 2

Write QUORUM + Read QUORUM > Replication factor
     2       +      2      > 3
     4 > 3 ✅

Why this matters:
→ Write touches nodes: 1, 2
→ Read touches nodes:  2, 3
→ Node 2 = overlap ✅
→ Node 2 ALWAYS has latest write ✅
→ Read always returns latest data ✅
→ Strong consistency guaranteed ✅
```

---

## Tunable Consistency + CAP Theorem:

```
CAP says: pick 2 of 3 (fixed choice)

Cassandra says: pick per operation:
ONE    → favors Availability  (AP behavior)
ALL    → favors Consistency   (CP behavior)
QUORUM → balanced (strong consistency + some availability)

One cluster → different CAP behavior per query ✅
PostgreSQL can't do this ❌
DynamoDB can't do this ❌
Only Cassandra ✅
```

---

## What Happens During Node Failure:

```
3 nodes. Node 3 goes down.

Write with ONE:
→ Node 1 confirms ✅ → success
→ Node 3 misses write 😬
→ Available ✅ but potentially inconsistent ❌

Write with ALL:
→ Node 1 ✅, Node 2 ✅, Node 3 ❌ (down)
→ Write FAILS 😬
→ Consistent but unavailable 😬

Write with QUORUM:
→ Node 1 ✅, Node 2 ✅ → success ✅
→ Node 3 down — don't need it
→ Available ✅ AND consistent ✅
→ Best of both worlds ✅
```

---

## Read Repair — Fixing Stale Nodes:

```
Scenario:
→ Node 3 was down during write
→ Missed latest update
→ Came back online with stale data

Next QUORUM read:
→ Read from Node 1 + Node 2
→ Compare timestamps
→ Node 1 has latest ✅
→ Cassandra automatically pushes latest
  to Node 3 in background ✅
→ Called Read Repair ✅
→ Eventual consistency maintained automatically ✅
```

---

## Mix Consistency Levels — Real Architecture:

```
Same Cassandra cluster at Hotstar:

Watch history write  → ONE    (speed matters)
Watch history read   → ONE    (slight staleness ok)
Payment status write → QUORUM (consistency matters)
Payment status read  → QUORUM (must be accurate)
Config read          → ALL    (must be latest)

One cluster, different guarantees per operation ✅
```

---

## Cassandra vs PostgreSQL vs MongoDB — Full Comparison

| | PostgreSQL | MongoDB | Cassandra |
|---|---|---|---|
| **Write speed** | Medium | Medium | Very Fast ✅ |
| **Read speed** | Fast ✅ | Fast ✅ | Medium |
| **ACID** | Full ✅ | Partial ⚠️ | No ❌ |
| **Schema** | Fixed | Flexible | Query-driven |
| **JOINs** | Full ✅ | $lookup ⚠️ | None ❌ |
| **Consistency** | Strong (fixed) | Tunable ⚠️ | Tunable ✅ |
| **Scaling** | Vertical + sharding | Built-in sharding | Built-in ✅ |
| **Best for** | Financial, relational | Catalog, content | Time-series, write-heavy |

---

## Complete Decision Framework

```
Need ACID + relational data?          → PostgreSQL
Need fast temporary cache?            → Redis
Need flexible schema + nested data?   → MongoDB
Need massive write throughput?        → Cassandra ✅
Need strong + tunable consistency?    → Cassandra ✅
Need analytics + aggregations?        → BigQuery/Redshift
Need full-text search?                → Elasticsearch
```

---

## Interview Framing

**On Cassandra data modeling:**
> *"In Cassandra, queries drive schema design — not the other way around. You decide access patterns first, then design tables around them. The partition key determines which node stores the data, and the clustering key determines sort order within that node. Same data is often stored in multiple tables for different access patterns — storage is cheap, cross-node joins are not."*

**On tunable consistency:**
> *"Cassandra's tunable consistency is its real superpower. For Hotstar watch history we'd use ONE — fastest writes, slight staleness acceptable. For Razorpay payments we'd use QUORUM — majority confirmation gives strong consistency while tolerating one node failure. QUORUM works because write set plus read set must overlap, guaranteeing at least one node always has the latest write."*

**On CAP theorem:**
> *"Cassandra doesn't make a fixed CAP choice. ONE favors availability, ALL favors consistency, QUORUM balances both. One cluster can serve different CAP requirements per operation — something PostgreSQL and DynamoDB can't do."*

---

## Quick Drill Questions

```
Q1: What is the difference between 
    partition key and clustering key?

Q2: Why doesn't Cassandra support JOINs?

Q3: Hotstar needs two queries:
    a) Get watch history by user
    b) Get all viewers of a show
    How many tables? What are the primary keys?

Q4: What is tunable consistency and 
    why is it Cassandra's superpower?

Q5: Why does QUORUM beat ALL for fintech use cases?

Q6: Prove mathematically why QUORUM reads
    always return the latest write.

Q7: What is Read Repair?

Q8: What is time bucketing and when do you need it?
```

---

## Drill Answers

### Q1: Partition key vs clustering key
```
Partition key:
→ Decides WHICH NODE stores the data
→ All rows with same partition key = same node
→ hash(partition_key) % nodes = node number
→ Enables single-node reads ✅

Clustering key:
→ Decides ORDER of rows WITHIN a node
→ Rows sorted by clustering key on disk
→ Enables efficient range queries ✅
→ "Get last 10 rows" = read last 10 from sorted data

PRIMARY KEY (user_id, timestamp)
user_id   = partition key  → which node
timestamp = clustering key → sorted order within node
```

### Q2: Why no JOINs
```
Cassandra = distributed across many nodes

JOIN needs:
→ Data from Table A (Node 1)
→ Data from Table B (Node 5)
→ Cross-node network call = expensive 😬

Solution:
→ Denormalize → store all needed data in one table
→ Duplicate data across multiple tables
→ Storage cheap, network calls expensive ✅
```

### Q3: Hotstar two queries
```
Two tables needed:

Table 1 — query by user:
PRIMARY KEY (user_id, timestamp)
→ "Get watch history for user:123" ✅

Table 2 — query by show:
PRIMARY KEY (show_id, timestamp)
→ "Get all viewers of show:456" ✅

Same data written twice intentionally.
Alternatively: Materialized View on Table 1
→ Cassandra auto-maintains Table 2 ✅
```

### Q4: Tunable consistency
```
Most DBs make fixed CAP choice:
PostgreSQL → always CP
DynamoDB   → always AP

Cassandra → YOU choose per operation:
ONE    → 1 node confirms → fast, less consistent
QUORUM → majority confirms → balanced
ALL    → all nodes confirm → slow, most consistent

Superpower:
→ Same cluster, different guarantees per query
→ Watch history → ONE (speed)
→ Payments → QUORUM (consistency)
→ One cluster serves both ✅
```

### Q5: QUORUM beats ALL for fintech
```
ALL:
→ All 3 nodes must confirm
→ One node slow/down → write fails 😬
→ High availability cost 😬

QUORUM (2 of 3):
→ Tolerates 1 node failure ✅
→ Faster than ALL ✅
→ Still strong consistency ✅
→ Write set + Read set overlap = latest data always readable
→ Best balance for payment systems ✅
```

### Q6: QUORUM math proof
```
Replication factor N = 3
Write QUORUM W = 2
Read QUORUM  R = 2

W + R > N
2 + 2 > 3 ✅

Write touches nodes: {1, 2}
Read touches nodes:  {2, 3}
Overlap = {2} ✅

Node 2 always has latest write.
Read always includes Node 2.
Latest data always returned ✅
```

### Q7: Read Repair
```
Scenario:
→ Node 3 was down during write
→ Has stale data after coming back online

Next QUORUM read:
→ Reads Node 1 + Node 2
→ Compares timestamps
→ Finds Node 1 has latest version
→ Pushes latest to Node 3 in background ✅
→ Node 3 healed automatically ✅
→ No manual intervention needed ✅
```

### Q8: Time bucketing
```
Problem:
→ user:123 has 50,000 rows over 5 years
→ All on same node = hot partition 😬
→ One node overwhelmed while others idle 😬

Solution — composite partition key:
PRIMARY KEY ((user_id, year_month), timestamp)

user:123, 2024-01 → Node 3
user:123, 2024-02 → Node 7
user:123, 2024-03 → Node 1

→ Data spread across nodes ✅
→ No single hot partition ✅

When to use:
→ High-volume users/entities
→ Time-series data accumulated over years
→ Any scenario where single partition = too much data
```

---

## One-Line Summaries

```
Partition key    → which node stores data
Clustering key   → sort order within node
One query = table → design schema around access patterns
Denormalization  → intentional, storage is cheap
ONE              → fastest, least consistent
QUORUM           → majority, balanced (W+R>N)
ALL              → slowest, strongest, rarely used
Read repair      → stale nodes healed automatically
Time bucketing   → spread hot partition across nodes
Tunable consistency → same cluster, different CAP per query
```

---

*Chunk 23B complete*
*Cassandra coverage: Chunk 23 (LSM internals) + Chunk 23B (data model + consistency)*
*Next: Chunk 24 — Graph DBs + Master Decision Framework*
*Phase 2 progress: 10/13 chunks done*