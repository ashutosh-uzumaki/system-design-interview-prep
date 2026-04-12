# Chunk 23C — Cassandra Remaining Concepts
### Token Ring | Vnodes | Gossip | Hot Partition | LWT | Anti-patterns
### Phase 2 — Databases | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Hotstar, Uber

---

# PART 1 — Data Distribution

## Token Ring — Consistent Hashing

```
Cassandra uses consistent hashing internally.
Hash output forms a TOKEN RING — a circle:

Range: 0 to 2^127 around the ring

6 nodes evenly distributed:
Node 1 → 0           to 21trillion
Node 2 → 21trillion  to 42trillion
Node 3 → 42trillion  to 63trillion
Node 4 → 63trillion  to 84trillion
Node 5 → 84trillion  to 105trillion
Node 6 → 105trillion to 2^127

Write: user:123
→ hash(user:123) = 55trillion
→ 55trillion falls in Node 3's range
→ Goes to Node 3 ✅
```

### Adding a new node:
```
Without consistent hashing:
→ Rehash everything → massive reshuffling 😬

With token ring:
→ New Node 7 inserted between Node 3 + Node 4
→ Only Node 3's data in Node 7's new range moves
→ Everything else stays ✅
→ Zero downtime migration ✅
```

---

## Virtual Nodes (vnodes)

### Problem with simple token ring:
```
Node 3 crashes:
→ Node 3 owned ONE big range
→ ALL data moves to Node 4 only 😬
→ Node 4 gets double load 😬
→ Other nodes idle 😬
```

### Solution — vnodes:
```
Without vnodes (simple token ring):
Node 1: |────────range1────────|
Node 2:                        |────────range2────────|

Node 3 crashes → entire range → Node 4 only 😬

With vnodes (256 tokens per node):
Node 1: |--| |--| |--|  scattered across ring
Node 2: |--| |--| |--|  scattered across ring
Node 3: |--| |--| |--|  scattered across ring
Node 4: |--| |--| |--|  scattered across ring

Node 3 crashes:
→ 256 small ranges spread across Node 1, 2, 4
→ Each absorbs small portion ✅
→ Load evenly distributed ✅
```

### Vnodes benefits:
```
✅ Even load distribution on failure
✅ New node → takes small chunks from ALL nodes
✅ Heterogeneous hardware:
   → Powerful node → 512 vnodes
   → Weak node    → 128 vnodes
   → Proportional load ✅
✅ Faster rebuilding (data from many nodes) ✅

Default: 256 vnodes per node
Config:  num_tokens in cassandra.yaml
```

---

# PART 2 — Query Routing

## Coordinator Node

```
Simple version:
→ Think of coordinator as receptionist
→ Client walks into ANY door
→ Receptionist figures out who you need
→ Routes you there ✅

Technical version:
Client → connects to Node 2 (any node)
              ↓
         Node 2 = Coordinator
              ↓
    Calculates: hash(partition_key) → token → node
              ↓
    Forwards request to correct node
              ↓
    Correct node returns data to Coordinator
              ↓
    Coordinator returns to Client ✅
```

### With Consistency Levels:
```
ONE:
→ Coordinator asks 1 node only ✅

QUORUM (RF=3):
→ Coordinator asks 2 nodes
→ Waits for 2 responses ✅

ALL:
→ Coordinator asks all 3 replicas
→ Waits for all 3 ✅
```

---

## Token-Aware Routing

```
Without Token Aware:
Client → Any Node (Coordinator) → Correct Node
→ 2 hops, extra network call 😬

With Token Aware (smart driver):
→ Driver downloads token ring on startup
→ Calculates correct node locally
→ Connects DIRECTLY to correct node ✅
→ 1 hop, coordinator eliminated ✅

Production always uses Token Aware routing ✅
Available in all Cassandra drivers (Java, Python etc.)
```

---

# PART 3 — Failure Detection

## Gossip Protocol

### Problem with naive approach:
```
100 nodes × 99 pings = 9,900 messages/sec 😬
1000 nodes × 999 pings = 999,000 messages/sec 😬
Network flooded 😬
```

### Gossip solution:
```
Every second, each node:
→ Picks 3 random nodes
→ Exchanges state information
→ "Here's what I know about every node"

After few rounds:
→ Every node knows state of every other node ✅
→ Only 3 messages per node per second ✅
→ 100 nodes = 300 messages/second ✅

Like office gossip:
→ Alice tells Bob "Carol is sick"
→ Bob tells Dave "Carol is sick"
→ Soon everyone knows ✅
→ No central announcement needed ✅
```

### What nodes gossip about:
```
Each node maintains state table:

Node 1: {status: UP,   load: 45%, token: 0-21T}
Node 2: {status: UP,   load: 52%, token: 21-42T}
Node 3: {status: DOWN, load: 0%,  token: 42-63T}

→ Spreads across all nodes ✅
→ No central registry ✅
→ No SPOF ✅
```

---

## Phi Accrual Failure Detection

```
Simple heartbeat (naive):
→ "Haven't heard in 5 seconds = dead"
→ Too rigid 😬
→ Slow network ≠ dead node 😬

Phi Accrual (Cassandra):
→ Tracks historical heartbeat intervals
→ Calculates PROBABILITY of failure
→ "Node 3 is 99.9% likely down"
→ Adaptive threshold ✅
→ Accounts for network jitter ✅

phi values:
→ phi < 8  → node probably alive
→ phi > 8  → node suspected dead
→ phi > 16 → node definitely dead
→ Threshold configurable ✅
```

---

## Seed Nodes

```
Problem:
→ New Node 7 wants to join cluster
→ Doesn't know anyone 😬

Solution:
→ 2-3 nodes designated as seeds
→ Listed in cassandra.yaml
→ New node contacts seed first
→ Seed introduces to cluster
→ Gossip spreads the news ✅

cassandra.yaml:
seed_provider:
  seeds: "192.168.1.1, 192.168.1.2"

Note:
→ Seeds not special after bootstrap
→ Just entry points for new nodes
→ Should always be available ✅
```

---

# PART 4 — Fault Tolerance

## Hinted Handoff

```
Scenario:
Write with QUORUM (RF=3)
→ Node 3 ✅, Node 4 ✅, Node 5 DOWN 😬
→ QUORUM met → write succeeds ✅
→ But Node 5 missed the write 😬

Solution — Hinted Handoff:
Step 1: Coordinator stores HINT locally:
{
  target_node: Node 5,
  partition_key: user:123,
  data: {show:456, timestamp:10:30PM}
}

Step 2: Node 5 comes back online
→ Gossip detects Node 5 is UP
→ Coordinator delivers hint ✅
→ Node 5 applies missed write ✅
→ Fully caught up ✅

Hint retention: 3 hours (default)
→ Node down > 3 hours → hints discarded 😬
→ Needs full repair instead
```

---

## Anti-Entropy Repair

```
Node 5 down for > 3 hours:
→ Hints discarded
→ Has stale data

Fix: nodetool repair
→ Uses Merkle trees
→ Compares data across replicas
→ Finds differences efficiently
→ Syncs missing data ✅

Run periodically in production:
→ Weekly repair recommended
→ Keeps replicas in sync ✅
```

## Hinted Handoff vs Read Repair vs Anti-Entropy:

```
Hinted Handoff:
→ Triggered on WRITE
→ Coordinator stores hint proactively
→ For short outages (< 3 hours) ✅

Read Repair:
→ Triggered on READ
→ Detects stale data during query
→ Fixes in background passively
→ Happens naturally ✅

Anti-Entropy Repair:
→ Manual: nodetool repair
→ For long outages (> 3 hours)
→ Uses Merkle trees for efficient diff ✅
→ Run periodically in production
```

---

# PART 5 — Read Performance

## Why Reads Are Slower in Cassandra

```
PostgreSQL (B-Tree):
→ Write = update IN PLACE
→ Data always in one location
→ Read = go directly there ✅
→ Fast reads ✅

Cassandra (LSM Tree):
→ Write = always APPEND
→ Never update in place
→ Same key can exist in:
   → MemTable (RAM)
   → SSTable 1 (disk)
   → SSTable 2 (disk)
   → SSTable 3 (disk)
→ Read = check ALL these places 😬
→ Read Amplification 😬
```

### Read path:
```
Read: user:123

Step 1 → MemTable (RAM) → found? return ✅
Step 2 → Bloom filter → eliminate 95% SSTables
Step 3 → Sparse index → find disk offset
Step 4 → Read from remaining SSTables
Step 5 → Pick latest timestamp ✅

1 logical read → 3-5 physical disk reads
vs PostgreSQL → 1 disk read ✅
```

### How compaction helps reads:
```
Before compaction:
→ user:123 in 50 SSTables 😬
→ Read = check 50 locations 😬

After compaction:
→ user:123 in 1 merged SSTable ✅
→ Read = check 1 location ✅
→ Compaction directly improves read performance ✅
```

### Tombstone heavy reads:
```
Heavy deletes → many tombstones
Read must scan through tombstones:
→ "Is this key alive or deleted?"
→ Many tombstones = very slow reads 😬

Fix: Use TTL instead of manual deletes ✅
```

---

# PART 6 — Hot Partition Problem

## What Is It:

```
hash(partition_key) → always same node
→ One node gets ALL traffic
→ Other nodes idle
→ Hot node overwhelmed 😬

Example:
hash("iphone15") → always Node 3
10M users × same product → Node 3 only 😬
```

## 3 Scenarios:

```
1. Celebrity/Viral content:
→ iPhone launch, IPL final, viral tweet
→ One item = millions of requests
→ All hash to same node 😬

2. Time-based partition key:
→ Partition key = date ("2024-01-15")
→ ALL today's writes → same node 😬
→ Yesterday's node idle

3. Skewed user data:
→ Power user / celebrity account
→ Millions of rows in one partition
→ That node overwhelmed 😬
```

## How to Detect:

```
1. nodetool tpstats
   → One node has 10x more pending tasks 😬

2. nodetool cfstats
   → One partition key has 100x more requests

3. Grafana/Datadog:
   → One node at 95% CPU, others at 20% 😬

4. Application logs:
   → Timeouts on specific partition keys only
```

## 4 Solutions:

### Solution 1 — Redis Cache (read-heavy):
```
First request  → Cassandra Node 3
               → store in Redis ✅
Next 9,999,999 → Redis only ✅
               → Node 3 never touched ✅

Gain:
✅ Eliminates read hot partition
✅ Fastest to implement (production emergency)
✅ No schema changes

Lose:
❌ Cache invalidation complexity
❌ Stale data risk
❌ Redis must be clustered at scale

Best for: read-heavy, product pages, match scores
```

### Solution 2 — Write Salting (write-heavy):
```
Append random suffix to partition key:
"iphone15_1" → Node 3
"iphone15_2" → Node 7
"iphone15_3" → Node 1
"iphone15_4" → Node 5

On write: suffix = random(1 to N buckets)
On read:  query ALL N partitions → merge results

Salt range:
→ High write, low read  → 50-100 buckets
→ High write, high read → 5-10 buckets
→ NOT equal to node count
→ Hash distributes naturally ✅

Gain:
✅ Writes spread evenly
✅ No schema migration

Lose:
❌ Reads = scatter-gather (N queries)
❌ Application complexity

Best for: write-heavy, event logging, activity tracking
```

### Solution 3 — Time Bucketing (time-series):
```
Before:
PRIMARY KEY (product_id, timestamp)
→ All writes forever → same node 😬

After:
PRIMARY KEY ((product_id, hour_bucket), timestamp)

hour_bucket = "2024-01-15-10"

"iphone15", "2024-01-15-10" → Node 3
"iphone15", "2024-01-15-11" → Node 7
"iphone15", "2024-01-15-12" → Node 1

Bucket size guide:
→ Flash sale   → hour buckets ✅
→ Normal data  → day buckets ✅
→ Historical   → week/month buckets ✅

Gain:
✅ Natural time-based distribution
✅ Old buckets = cold data, easy to archive

Lose:
❌ Must query per bucket
❌ Current bucket still hot at peak
   → Combine with write salting ✅

Best for: time-series data, logs, events
```

### Solution 4 — Redesign Partition Key (long-term):
```
Before:
PRIMARY KEY (product_id, timestamp)
→ "iphone15" same for ALL users 😬

After:
PRIMARY KEY ((user_id, product_id), timestamp)

user:001 + iphone15 → Node 3
user:002 + iphone15 → Node 7
user:003 + iphone15 → Node 1
→ Perfect distribution ✅

Gain:
✅ Perfect write distribution
✅ No application changes
✅ Cleanest long-term solution

Lose:
❌ Can't query "all users who viewed iphone15"
❌ Schema redesign = data migration 😬
❌ Only when user_id in every query

Best for: new system design, can afford migration
```

## Which Solution When:

```
Production emergency (2AM):
→ Redis cache ✅ (fastest, no schema change)

Write-heavy hot partition:
→ Write salting ✅

Time-series data:
→ Time bucketing ✅

New system design:
→ Redesign partition key ✅

Extreme cases:
→ Combine: Redis cache + Write salting ✅
```

## Real Examples:

```
Hotstar IPL final:
→ match:IPL_final → write path: Cassandra (events)
→ Read path: Redis cache (500M users reading score)
→ CDN for video stream

Flipkart Big Billion Day:
→ product:iphone15 → Redis cache for reads
→ Write salting for inventory updates

Power user (skewed data):
→ VIP user with 2M orders
→ Fix: Time bucketing by year_month
PRIMARY KEY ((user_id, year_month), timestamp)
```

---

# PART 7 — Advanced Features

## TTL (Time To Live)

```
Like Redis TTL but for Cassandra rows:

INSERT INTO messages (id, content, user_id)
VALUES ('msg123', 'Hello', 'user:456')
USING TTL 2592000;  ← 30 days in seconds

→ Row auto-deleted after expiry ✅
→ No manual delete needed ✅
→ Cassandra writes tombstone internally
→ Cleaned up after gc_grace_seconds ✅

Use cases:
→ Message expiry (WhatsApp 30 days)
→ OTP storage (2 minutes)
→ Session data (1 hour)
→ Log retention (90 days)
→ Temporary promotions

Why TTL over manual deletes:
→ Manual delete = tombstone accumulation 😬
→ TTL = controlled expiry, clean tombstone handling ✅
```

---

## Counter Columns

```
Special column type for atomic counting:

CREATE TABLE page_views (
    page_id  TEXT,
    views    COUNTER,
    PRIMARY KEY (page_id)
);

UPDATE page_views
SET views = views + 1
WHERE page_id = 'iphone15';

Why special counter type?
→ Regular write = Last Write Wins (LWW)
→ Two concurrent increments:
   LWW: both read 100, both write 101 → loses one ❌
   Counter: atomic increment → 102 ✅

Use cases:
→ Page view counts (Flipkart)
→ Like counts (Instagram)
→ Download counts
→ API call counts

Limitations:
❌ Counter table = ONLY counter columns
❌ Can't mix counter + regular columns
❌ Can't use TTL with counters
❌ Slightly slower than regular writes
```

---

## Lightweight Transactions (LWT)

```
Cassandra's limited ACID-like feature:

Normal write = Last Write Wins (no condition check)
LWT = IF condition check before write

Examples:
-- Username uniqueness:
INSERT INTO usernames (username, user_id)
VALUES ('ashutosh', 'user:123')
IF NOT EXISTS;

-- Inventory check:
UPDATE inventory
SET count = count - 1
WHERE product_id = 'iphone15'
IF count > 0;

-- Only update if value matches:
UPDATE users
SET email = 'new@email.com'
WHERE user_id = 'user:123'
IF email = 'old@email.com';
```

### How LWT works:
```
Uses Paxos consensus algorithm:
→ 4 round trips between nodes
→ Guarantees conditional atomicity ✅

Performance cost:
Normal write → 1 round trip
LWT write    → 4 round trips
→ ~4x slower 😬
```

### When to use LWT:
```
✅ Username/email uniqueness check
✅ Simple inventory reservation
✅ "Check then set" operations
✅ Preventing duplicate inserts

When NOT to use:
❌ High throughput writes (too slow)
❌ Complex multi-row transactions
❌ Financial transactions → PostgreSQL ✅
❌ When eventual consistency is acceptable
```

---

## Replication Strategies

### SimpleStrategy:
```
→ Single datacenter only
→ Replicas on next N nodes in ring
→ Development/testing ONLY

CREATE KEYSPACE hotstar
WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 3
};
```

### NetworkTopologyStrategy:
```
→ Multiple datacenters ✅
→ Rack + datacenter aware
→ ALWAYS use in production

CREATE KEYSPACE hotstar
WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'DC1': 3,  ← 3 replicas in DC1
    'DC2': 2   ← 2 replicas in DC2
};

Benefits:
✅ Rack awareness (replicas on different racks)
✅ Datacenter awareness (survive DC failure)
✅ Multi-region support
✅ Local reads (read from nearest DC)
✅ Production grade ✅
```

---

# PART 8 — Anti-Patterns

```
1. Too many tombstones:
→ Heavy manual deletes = tombstone accumulation
→ Reads scan through tombstones = slow 😬
→ Fix: Use TTL instead of manual deletes ✅

2. Unbounded partitions:
→ Partition grows forever
→ user:123 has 50M rows in one partition 😬
→ Fix: Time bucketing ✅

3. Secondary index abuse:
→ CREATE INDEX on low-cardinality column
→ status = DELIVERED → millions of rows per value
→ Cassandra secondary indexes = full cluster scan 😬
→ Fix: Create separate table for that query ✅

4. Using Cassandra like SQL:
→ Designing schema first, queries later 😬
→ Expecting JOINs to work 😬
→ Fix: Always design by access pattern ✅

5. Large objects in rows:
→ Storing video/image blobs in Cassandra 😬
→ Fix: Store in S3, store URL in Cassandra ✅

6. Too many tables for same data:
→ Duplicate data without considering write cost
→ 100 tables for same entity = 100x write amplification 😬
→ Fix: Group similar queries → one table ✅

7. Ignoring compaction:
→ Not monitoring SSTable count
→ Reads degrade over time 😬
→ Fix: Monitor + tune compaction strategy ✅
```

---

# PART 9 — Complete Cassandra Mental Model

```
Write path:
───────────
Write → CommitLog (crash recovery, sequential)
      → MemTable (RAM, fast buffer, ~128MB)
      → SSTable flush (immutable, sorted, on disk)
      → Compaction (merge + LWW cleanup)

Read path:
──────────
Read → MemTable (fastest)
     → Bloom filter (skip irrelevant SSTables)
     → Sparse index (find disk offset)
     → Disk read (pick latest timestamp) ✅

Data distribution:
──────────────────
Consistent hashing → Token Ring ✅
256 vnodes per node → even distribution ✅
New node → small chunks from all nodes ✅

Query routing:
──────────────
Any node → Coordinator → routes to correct node
Token-aware driver → skip coordinator → direct ✅

Failure detection:
──────────────────
Gossip → 3 random nodes/second
Phi Accrual → probability-based detection
Hinted Handoff → short outage recovery (< 3hrs)
Anti-Entropy Repair → long outage recovery ✅

Consistency:
────────────
ONE    → fastest, least consistent
QUORUM → balanced (W+R>N math) ✅
ALL    → slowest, strongest
Tunable per operation ✅
```

---

# Interview Framing

**On vnodes:**
> *"Cassandra uses 256 virtual nodes per physical node by default. When a node fails, its 256 small token ranges spread across all remaining nodes — no single node absorbs all the load. This also makes adding new nodes seamless — they take proportional chunks from everyone."*

**On gossip:**
> *"Cassandra uses gossip for failure detection — each node gossips with 3 random nodes per second, sharing cluster state. Information propagates exponentially, so failures are detected cluster-wide in seconds. Phi Accrual failure detection uses probability rather than fixed timeouts, adapting to network conditions."*

**On hot partition:**
> *"Hot partition happens when one partition key gets disproportionate traffic. In a production emergency I'd add Redis caching immediately — no schema changes needed. Long-term fix depends on the pattern: write salting for write-heavy, time bucketing for time-series, partition key redesign for permanent fix."*

**On LWT:**
> *"Cassandra's Lightweight Transactions give us conditional writes using Paxos consensus. They're 4x slower than regular writes, so I'd use them only for simple check-then-set operations like username uniqueness — not for high-throughput payment flows where PostgreSQL is more appropriate."*

---

# Quick Drill Questions

```
Q1: What problem do vnodes solve over simple token ring?

Q2: How does gossip protocol work?
    Why not ping every node?

Q3: What is hinted handoff?
    What happens if node is down > 3 hours?

Q4: Name 4 solutions to hot partition
    with one real example each.

Q5: When would you use LWT vs PostgreSQL
    for a conditional write?

Q6: What is the difference between TTL and
    manual delete in Cassandra?

Q7: SimpleStrategy vs NetworkTopologyStrategy
    — when to use each?

Q8: Name 3 Cassandra anti-patterns and fixes.
```

---

# Drill Answers

### Q1: Vnodes vs simple token ring
```
Simple token ring:
→ Each node owns ONE large range
→ Node fails → entire range → ONE neighbor 😬
→ Uneven load distribution 😬

Vnodes:
→ Each node owns 256 SMALL scattered ranges
→ Node fails → 256 ranges spread across ALL nodes ✅
→ Even load distribution ✅
→ Heterogeneous hardware supported ✅
→ New node joins → takes small chunks from everyone ✅
```

### Q2: Gossip protocol
```
Naive approach: every node pings every other
→ 100 nodes = 9,900 messages/sec 😬

Gossip:
→ Each node picks 3 random nodes/second
→ Exchanges complete cluster state
→ Information propagates exponentially
→ 100 nodes = 300 messages/sec ✅
→ No central registry, no SPOF ✅
→ Phi Accrual: probability-based detection
   adapts to network conditions ✅
```

### Q3: Hinted handoff
```
Node 5 down during write (QUORUM met without it):
→ Coordinator stores HINT:
  {target: Node5, data: missed_write}
→ Node 5 comes back → hint delivered ✅
→ Node 5 catches up ✅

Node down > 3 hours:
→ Hints discarded (disk space)
→ Run: nodetool repair
→ Uses Merkle trees to find + sync differences ✅
```

### Q4: Hot partition solutions
```
1. Redis Cache (read-heavy):
   → iPhone 15 page views
   → First read → cache, next 9.9M → Redis ✅

2. Write Salting (write-heavy):
   → Activity logging for viral event
   → "event_123_1" through "event_123_10"
   → Spread across 10 nodes ✅

3. Time Bucketing (time-series):
   → IPL match events per over
   → PRIMARY KEY ((match_id, hour_bucket), timestamp)
   → Different hours → different nodes ✅

4. Redesign Partition Key (long-term):
   → Add user_id to partition key
   → PRIMARY KEY ((user_id, product_id), timestamp)
   → Each user hashes differently ✅
```

### Q5: LWT vs PostgreSQL
```
Use LWT when:
→ Simple check-then-set ✅
→ Username uniqueness check ✅
→ Low-throughput conditional insert ✅
→ Already using Cassandra for that data

Use PostgreSQL when:
→ Financial transactions (ACID required) ✅
→ Multi-row conditional logic ✅
→ High-throughput writes (LWT = 4x slower) ✅
→ Complex business rules
```

### Q6: TTL vs manual delete
```
Manual delete:
→ Writes tombstone immediately
→ Tombstones accumulate over time 😬
→ Reads must scan through tombstones 😬
→ Heavy deletes = serious read degradation

TTL:
→ Row auto-expires after set time
→ Cassandra handles tombstone cleanup internally
→ gc_grace_seconds controls cleanup window
→ Clean, predictable expiry ✅
→ No tombstone accumulation 😬

Rule: Always prefer TTL over manual deletes
      when data has natural expiry ✅
```

### Q7: Replication strategies
```
SimpleStrategy:
→ Single datacenter only
→ Development/testing
→ Never in production 😬

NetworkTopologyStrategy:
→ Multiple datacenters ✅
→ Rack + DC aware
→ Survives rack/DC failure ✅
→ Local reads from nearest DC ✅
→ ALWAYS in production ✅
```

### Q8: Anti-patterns + fixes
```
1. Too many tombstones:
   → Heavy manual deletes 😬
   → Fix: Use TTL ✅

2. Secondary index abuse:
   → INDEX on status = DELIVERED 😬
   → Full cluster scan 😬
   → Fix: Separate table for that query ✅

3. Using Cassandra like SQL:
   → Schema first, queries later 😬
   → Expecting JOINs 😬
   → Fix: Design by access pattern ✅
```

---

# One-Line Summaries

```
Token ring      → consistent hashing, 0 to 2^127
Vnodes          → 256 small ranges, even load on failure
Coordinator     → any node routes request to correct node
Token aware     → client routes directly, skip coordinator
Gossip          → 3 random nodes/sec, no central registry
Phi Accrual     → probability-based failure detection
Seed nodes      → entry point for new nodes joining
Hinted handoff  → coordinator stores missed writes, delivers on recovery
Anti-entropy    → nodetool repair, Merkle tree sync
Hot partition   → one node overwhelmed, others idle
Write salting   → random suffix, spread writes across buckets
Time bucketing  → time unit in partition key, spread over time
TTL             → auto-expire rows, no tombstone accumulation
Counter         → atomic increment, separate table type
LWT             → IF condition write, Paxos, 4x slower
SimpleStrategy  → dev/test only, single DC
NetworkTopology → production always, multi-DC, rack aware
```

---

*Chunk 23C complete — Full Cassandra coverage done*
*Chunks 23 + 23B + 23C = Complete Cassandra reference*
*Next: Chunk 24 — Graph DBs + Master Decision Framework*
*Phase 2 progress: 11/13 chunks done*