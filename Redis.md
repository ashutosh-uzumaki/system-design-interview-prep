# Redis — Complete HLD Interview Deep Dive (0 → 100)

> **Why** every design decision was made, **how** it works under the hood, and the **tradeoffs** at every layer. Built for system design interviews.

---

## Table of Contents

1. [Why Redis Exists — The Problem It Solves](#1-why-redis-exists--the-problem-it-solves)
2. [Core Architecture — Single-Threaded But Blazing Fast](#2-core-architecture--single-threaded-but-blazing-fast)
3. [Data Structures — Redis's Superpower](#3-data-structures--rediss-superpower)
4. [Why Redis Is Fast — The Engineering Decisions](#4-why-redis-is-fast--the-engineering-decisions)
5. [Persistence — RDB vs AOF](#5-persistence--rdb-vs-aof)
6. [Replication — Primary-Replica Architecture](#6-replication--primary-replica-architecture)
7. [Redis Sentinel — High Availability](#7-redis-sentinel--high-availability)
8. [Redis Cluster — Sharding & Horizontal Scaling](#8-redis-cluster--sharding--horizontal-scaling)
9. [Eviction Policies — What Happens When Memory Is Full](#9-eviction-policies--what-happens-when-memory-is-full)
10. [Caching Patterns — How to Use Redis as a Cache](#10-caching-patterns--how-to-use-redis-as-a-cache)
11. [Cache Invalidation — The Hardest Problem](#11-cache-invalidation--the-hardest-problem)
12. [Pub/Sub & Streams](#12-pubsub--streams)
13. [Distributed Locking with Redis](#13-distributed-locking-with-redis)
14. [Transactions & Lua Scripting](#14-transactions--lua-scripting)
15. [Pipelining — Reducing Round Trips](#15-pipelining--reducing-round-trips)
16. [Common Redis Patterns in System Design](#16-common-redis-patterns-in-system-design)
17. [Redis vs Memcached — When to Use What](#17-redis-vs-memcached--when-to-use-what)
18. [Anti-Patterns & Common Mistakes](#18-anti-patterns--common-mistakes)
19. [Operational Best Practices](#19-operational-best-practices)
20. [Interview Questions — Deep Answers with Tradeoffs (50+)](#20-interview-questions--deep-answers-with-tradeoffs-50)

---

## 1. Why Redis Exists — The Problem It Solves

### The Problem

Databases (PostgreSQL, MySQL) store data on disk. Every read involves disk I/O, even with database caching (buffer pool). As applications scale to millions of requests per second, the database becomes the bottleneck — not because the query is complex, but because disk access is fundamentally slow (~1ms for SSD random read vs. ~100ns for RAM access — a 10,000x difference).

### What Redis Solved

Redis (REmote DIctionary Server) is an **in-memory data structure store**. All data lives in RAM. Every operation is a memory access. The result: sub-millisecond latency for most operations, and the ability to handle hundreds of thousands of operations per second on a single node.

**But it's not just fast — it's smart about data structures.** Unlike Memcached (which only stores strings), Redis exposes rich data structures (hashes, lists, sets, sorted sets, streams) as first-class primitives. This means you can do complex operations (sorted leaderboards, rate limiting, session management) in a single command instead of fetching data, transforming it in application code, and writing it back.

### Redis's Many Roles

| Role | How Redis Fills It | Why Not Use Something Else? |
|------|-------------------|---------------------------|
| **Cache** | Store hot data in front of a database | Faster than DB queries by 10-100x |
| **Session Store** | Store user sessions with TTL | Faster than DB; auto-expiry; shared across app instances |
| **Rate Limiter** | INCR + EXPIRE on request counters | Atomic operations; built-in TTL |
| **Leaderboard** | Sorted sets with scores | ZADD/ZRANGE is O(log N); no need to sort in app code |
| **Message Broker** | Pub/Sub or Streams | Lightweight alternative when Kafka is overkill |
| **Distributed Lock** | SET NX EX (Redlock for distributed) | Atomic set-if-not-exists with expiry |
| **Real-time Analytics** | HyperLogLog for cardinality, Bitmaps for flags | Memory-efficient probabilistic structures |
| **Queue** | Lists with LPUSH/BRPOP | Simple, reliable, blocking pop for consumers |
| **Geospatial Index** | GEOADD/GEORADIUS | Built-in geospatial distance calculations |

### When NOT to Use Redis

- **Primary database for critical data:** Redis is in-memory. Even with persistence (RDB/AOF), it's not as durable as PostgreSQL. A crash between persistence snapshots = data loss.
- **Large datasets that don't fit in RAM:** Redis stores everything in memory. If your dataset is 500GB, you need 500GB+ of RAM. Expensive.
- **Complex queries:** No SQL, no JOINs, no secondary indexes (natively). If you need to query data in flexible ways, use a real database.
- **Strong consistency requirements:** Redis replication is asynchronous by default. A failed primary may lose recently written data before replicas caught up.

---

## 2. Core Architecture — Single-Threaded But Blazing Fast

### The Single-Threaded Design

**The most surprising fact about Redis:** Command execution is single-threaded. One thread processes all commands sequentially. No locks, no mutexes, no race conditions.

**Why this works (and is actually faster):**

1. **No locking overhead:** Multi-threaded databases spend significant CPU time acquiring/releasing locks, handling contention, and managing shared state. Redis eliminates all of this.

2. **All operations are in-memory:** Since there's no disk I/O in the hot path, a single thread can process 100K+ operations/second. The bottleneck is the network, not the CPU.

3. **Operations are tiny:** Most Redis commands (GET, SET, INCR) are O(1) or O(log N). They complete in microseconds. A single thread can process millions of these per second.

4. **Simplicity:** Single-threaded code is easier to reason about, debug, and optimize. Every operation is atomic by nature — no need for transactions for single commands.

**The tradeoff:** A single slow command (e.g., `KEYS *` on a million-key database) blocks ALL other clients. Redis cannot utilize multiple CPU cores for command processing. To use multiple cores, you run multiple Redis instances on the same machine.

### What IS Multi-Threaded (Since Redis 6.0)

Redis 6.0+ added **I/O threads** — multiple threads handle network reading/writing (deserializing requests, serializing responses). But **command execution remains single-threaded.** This gives you the network throughput of multiple threads with the simplicity of single-threaded command processing.

```
Client Requests → [I/O Thread 1] → 
                  [I/O Thread 2] → [Single Command Thread] → [I/O Thread 1] → Responses
                  [I/O Thread 3] →                           [I/O Thread 2] →
                                                             [I/O Thread 3] →
```

### Event Loop Architecture

Redis uses an **event-driven, non-blocking I/O** model (like Node.js):

1. Single main thread runs an **event loop**
2. Uses OS I/O multiplexing (`epoll` on Linux, `kqueue` on macOS) to monitor thousands of sockets simultaneously
3. When data arrives on a socket, the event loop reads the command, executes it, and writes the response — all without blocking

**Why this matters for interviews:** When someone asks "How can Redis handle 100K ops/sec with a single thread?", the answer is: in-memory operations are microseconds, the event loop handles I/O multiplexing efficiently, and there's zero locking overhead. The network is usually the bottleneck, not the CPU.

---

## 3. Data Structures — Redis's Superpower

### Why Data Structures Matter

Most key-value stores give you `GET key` and `SET key value`. Redis gives you **server-side data structure operations.** Instead of fetching a list, modifying it in your app, and writing it back (3 round trips + serialization/deserialization), you do `LPUSH mylist value` (1 round trip, atomic).

### Core Data Structures

**Strings** — The simplest type. Can store text, numbers, or binary data (up to 512 MB).
```
SET user:123:name "Alice"          # Simple string
INCR request_count                 # Atomic increment (counter)
SETEX session:abc 3600 "data"      # Set with 1-hour expiry
SETNX lock:resource "owner"        # Set only if not exists (lock)
```
**Use cases:** Caching, counters, rate limiting, distributed locks, feature flags.

**Hashes** — A map of field-value pairs under a single key. Like a mini-object.
```
HSET user:123 name "Alice" age 30 email "alice@example.com"
HGET user:123 name                 # Get single field
HGETALL user:123                   # Get all fields
HINCRBY user:123 age 1             # Increment a field
```
**Use cases:** User profiles, shopping carts, configuration objects. More memory-efficient than separate string keys when storing related fields.

**Lists** — Ordered sequence of strings. Implemented as a linked list (actually a quicklist — a linked list of ziplists).
```
LPUSH queue:tasks "task1"          # Push to head
RPUSH queue:tasks "task2"          # Push to tail
BRPOP queue:tasks 30               # Blocking pop (wait up to 30s)
LRANGE queue:tasks 0 -1            # Get all elements
```
**Use cases:** Message queues, activity feeds, recent items, task queues.

**Sets** — Unordered collection of unique strings.
```
SADD online:users "user1" "user2"  # Add members
SISMEMBER online:users "user1"     # Check membership (O(1))
SINTER tag:python tag:redis        # Intersection of two sets
SCARD online:users                 # Count members
```
**Use cases:** Tags, unique visitors, mutual friends, deduplication.

**Sorted Sets (ZSets)** — Like sets, but each member has a score. Sorted by score.
```
ZADD leaderboard 1500 "player1"   # Add with score
ZADD leaderboard 2000 "player2"
ZRANGE leaderboard 0 -1 WITHSCORES  # Get all, sorted by score
ZRANK leaderboard "player1"       # Rank (0-based)
ZRANGEBYSCORE leaderboard 1000 2000  # Range query by score
```
**Use cases:** Leaderboards, priority queues, time-series (score = timestamp), rate limiting (sliding window).

**Internal implementation:** Skip list + hash table. ZADD is O(log N), ZRANK is O(log N), ZRANGE is O(log N + M) where M is the result size.

**Bitmaps** — Bit-level operations on strings.
```
SETBIT daily:active:2026-03-31 12345 1    # Mark user 12345 as active
GETBIT daily:active:2026-03-31 12345      # Check if active
BITCOUNT daily:active:2026-03-31          # Count active users
BITOP AND weekly:active daily:mon daily:tue daily:wed  # Intersection
```
**Use cases:** Daily active users, feature flags per user, bloom filter-like operations. Extremely memory-efficient: 1 billion users = 125 MB.

**HyperLogLog** — Probabilistic data structure for cardinality estimation.
```
PFADD unique:visitors "user1" "user2" "user3"
PFCOUNT unique:visitors            # Approximate count (0.81% error)
PFMERGE total:visitors set1 set2   # Merge multiple HLLs
```
**Use cases:** Counting unique visitors, unique search queries. Fixed 12 KB per HLL regardless of cardinality. **Tradeoff:** ~0.81% error rate, can't retrieve individual elements.

**Streams** — Append-only log data structure (similar to Kafka topics).
```
XADD events * sensor_id 1 temperature 22.5   # Append event
XREAD COUNT 10 STREAMS events 0               # Read events
XREADGROUP GROUP mygroup consumer1 COUNT 10 STREAMS events >  # Consumer group
```
**Use cases:** Event sourcing, message queues with consumer groups, activity logs. Unlike Pub/Sub, messages persist and support acknowledgment.

**Geospatial** — Coordinates stored as sorted set members.
```
GEOADD restaurants -73.935242 40.730610 "pizza_place"
GEORADIUS restaurants -73.935 40.730 5 km   # Find within 5km
GEODIST restaurants "pizza_place" "burger_joint" km  # Distance
```
**Use cases:** Nearby search, delivery tracking, location-based features.

---

## 4. Why Redis Is Fast — The Engineering Decisions

Every design decision in Redis trades something for speed:

### 1. In-Memory Storage
**How:** All data lives in RAM. No disk I/O in the read/write path.
**Speed gain:** RAM access ~100ns vs. SSD random read ~100μs (1000x faster).
**Tradeoff:** Data size is limited by available RAM. RAM is ~10-50x more expensive than SSD per GB. A crash without persistence means data loss.

### 2. Single-Threaded Command Execution
**How:** One thread processes all commands. No locks, no context switches.
**Speed gain:** Eliminates lock contention, cache line bouncing, and thread scheduling overhead.
**Tradeoff:** Can't use multiple CPU cores for commands. One slow command blocks everything. Solution: run multiple Redis instances per machine.

### 3. I/O Multiplexing (epoll/kqueue)
**How:** Single thread monitors thousands of sockets using OS-level event notification.
**Speed gain:** No thread-per-connection overhead. Can handle 10K+ concurrent connections with one thread.
**Tradeoff:** Complex event-driven programming model (but that's Redis's internal problem, not yours).

### 4. Efficient Data Structures with Compact Encodings
**How:** Redis uses memory-optimized internal encodings:
- Small hashes → `ziplist` (contiguous memory, cache-friendly)
- Small sets → `intset` (compact integer array)
- Small sorted sets → `ziplist` (linear scan is faster than skip list for tiny sets)
- As structures grow, Redis transparently upgrades to standard encodings (hash table, skip list)

**Speed gain:** Small objects use less memory and fewer cache misses.
**Tradeoff:** Encoding upgrades (e.g., ziplist → hashtable) are transparent but irreversible.

### 5. RESP Protocol (Redis Serialization Protocol)
**How:** Simple text-based protocol. Easy to parse, minimal overhead.
**Speed gain:** Parsing is trivial — no complex deserializer. Human-readable for debugging.
**Tradeoff:** Slightly more bandwidth than binary protocols (but network is rarely the bottleneck for typical payloads).

### 6. No Disk I/O in Hot Path
**How:** Persistence (RDB/AOF) happens in background processes (`fork()` for RDB, or buffered writes for AOF). The main thread never waits for disk.
**Speed gain:** Write latency is unaffected by persistence.
**Tradeoff:** Background persistence can cause latency spikes during `fork()` (due to copy-on-write memory pressure). More on this in the persistence section.

---

## 5. Persistence — RDB vs AOF

### Why Persistence Matters

Redis is in-memory. Without persistence, a restart = total data loss. Persistence writes data to disk so it can be recovered.

### RDB (Redis Database Backup) — Point-in-Time Snapshots

**How it works:**
1. Redis `fork()`s a child process
2. Child process writes the entire dataset to a temporary RDB file on disk
3. When complete, the temp file replaces the old RDB file (atomic rename)
4. Parent process continues serving clients uninterrupted

**Why fork()?** The child process gets a copy-on-write (COW) snapshot of memory. It can write to disk leisurely while the parent continues modifying data. Only modified memory pages are actually copied (OS-level COW).

**Configuration:**
```
save 900 1      # Snapshot if at least 1 key changed in 900 seconds
save 300 10     # Snapshot if at least 10 keys changed in 300 seconds
save 60 10000   # Snapshot if at least 10000 keys changed in 60 seconds
```

**Advantages:**
- Compact single-file backup (easy to transfer, archive)
- Fast restart (load binary file into memory)
- Lower disk I/O overhead during normal operation
- Good for backups and disaster recovery

**Disadvantages:**
- **Data loss window:** If Redis crashes between snapshots, all changes since the last snapshot are lost. With `save 300 10`, you could lose up to 5 minutes of data.
- **Fork overhead:** `fork()` copies the page table. For a 50GB dataset, this can take 1-2 seconds and cause a latency spike. During heavy writes, COW can temporarily double memory usage (because modified pages are copied).

### AOF (Append-Only File) — Write Log

**How it works:**
1. Every write command is appended to a log file (like a database WAL)
2. On restart, Redis replays the AOF file to reconstruct the dataset

**Sync options (the durability vs. performance tradeoff):**

| Policy | How | Data Loss | Performance |
|--------|-----|-----------|-------------|
| `appendfsync always` | fsync after every write | None | Slowest (disk I/O on every command) |
| `appendfsync everysec` | fsync every second | Up to 1 second | Good balance (recommended) |
| `appendfsync no` | Let OS decide when to flush | Depends on OS (typically 30s) | Fastest |

**AOF Rewrite:** Over time, the AOF file grows (it contains every write ever). Redis periodically rewrites it by creating a new, minimal AOF that represents the current state. This also uses `fork()`.

**Advantages:**
- Much smaller data loss window (1 second with `everysec`)
- Append-only — no corruption risk from partial writes
- Human-readable log (can inspect, edit, or replay)

**Disadvantages:**
- AOF files are larger than RDB files (text commands vs. binary snapshot)
- Slower restart (must replay all commands)
- Slightly lower write throughput (disk appends on every write)
- AOF rewrite still uses `fork()` with its memory overhead

### Hybrid Persistence (Redis 4.0+, Recommended)

Uses RDB for the base snapshot + AOF for incremental changes since the last snapshot. Combines fast restarts (RDB) with minimal data loss (AOF). This is the default in modern Redis.

### The Fork Problem — Why Large Datasets Hurt

**The issue:** `fork()` copies the process page table. For a 50GB dataset:
- Page table copy takes ~1-2 seconds (during which Redis is frozen)
- Copy-on-write means every modified page (4KB) is duplicated
- Under heavy writes, memory usage can temporarily spike to ~2x

**Mitigations:**
- Use enough RAM (at least 2x your dataset size for COW headroom)
- Avoid huge datasets on a single instance (use Redis Cluster to shard)
- Schedule RDB saves during low-traffic periods
- Some experts recommend disabling persistence on the primary and only persisting on replicas

### Persistence Decision Framework

| Scenario | Recommendation | Why |
|----------|---------------|-----|
| Pure cache (can reload from DB) | No persistence | Maximum performance, no fork overhead |
| Cache with warm-restart desire | RDB only | Fast restart, some data loss acceptable |
| Session store, moderate durability | AOF (`everysec`) | 1-second data loss window |
| Primary data store (not recommended, but...) | AOF (`always`) + RDB | Minimal data loss, but performance impact |
| Production best practice | Hybrid (RDB + AOF) | Fast restart + minimal data loss |

---

## 6. Replication — Primary-Replica Architecture

### Why Replication Exists

A single Redis instance is a single point of failure AND a read throughput bottleneck. Replication solves both.

### How It Works

```
┌──────────┐    async replication    ┌──────────┐
│  Primary  │ ─────────────────────▶ │ Replica 1│ ← Reads
│  (Master) │ ─────────────────────▶ │ Replica 2│ ← Reads
│           │                        │ Replica 3│ ← Reads
│ Reads +   │                        └──────────┘
│ Writes    │
└──────────┘
```

1. **Initial sync:** Replica connects to primary. Primary does a `BGSAVE` (RDB snapshot), sends the RDB file to the replica. Replica loads it.
2. **Ongoing replication:** Primary buffers all write commands in a **replication backlog** (ring buffer) and streams them to replicas in real-time.
3. **Partial resync:** If a replica briefly disconnects and reconnects, it can resume from where it left off in the backlog (if the backlog hasn't wrapped around). Otherwise, a full resync is needed.

### Replication Is Asynchronous

**Critical for interviews:** Redis replication is **asynchronous by default**. The primary sends the write acknowledgment to the client BEFORE the replica receives it. This means:

- A write can be acknowledged to the client and then **lost** if the primary crashes before the replica catches up
- Replicas may serve **stale reads** (they're always slightly behind the primary)

**The `WAIT` command:** You can request synchronous-like behavior: `WAIT 2 1000` blocks until at least 2 replicas confirm the write or 1000ms passes. But this isn't true synchronous replication — it doesn't prevent data loss if the primary crashes after WAIT returns but before replicas persist.

### Tradeoffs of Read Replicas

| Benefit | Tradeoff |
|---------|----------|
| Scale read throughput linearly | Reads may be stale (replication lag) |
| Failover target (promote replica to primary) | Promotion = brief downtime + potential data loss |
| Geographic distribution (replicas in different regions) | Higher replication lag for distant replicas |
| Persistence offloading (persist on replica, not primary) | Replica must handle fork() overhead instead |

---

## 7. Redis Sentinel — High Availability

### Why Sentinel Exists

Primary-replica setup doesn't auto-failover. If the primary dies, someone must manually promote a replica. Sentinel automates this.

### How Sentinel Works

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│Sentinel 1│     │Sentinel 2│     │Sentinel 3│
└─────┬────┘     └─────┬────┘     └─────┬────┘
      │                │                │
      ▼                ▼                ▼
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Primary  │────▶│ Replica 1│     │ Replica 2│
└──────────┘     └──────────┘     └──────────┘
```

1. **Monitoring:** Sentinels continuously ping the primary and replicas
2. **Subjective Down (SDOWN):** One sentinel thinks the primary is down (missed heartbeats)
3. **Objective Down (ODOWN):** A quorum of sentinels agree the primary is down
4. **Failover:** Sentinels elect a leader sentinel. The leader promotes the best replica to primary, reconfigures other replicas to follow the new primary, and notifies clients.
5. **Service discovery:** Clients ask Sentinel "who is the current primary?" instead of hardcoding IPs.

### Why You Need 3+ Sentinels

Sentinel uses quorum-based voting. With 2 sentinels, if one dies and the primary dies, the remaining sentinel can't reach quorum → no failover. With 3 sentinels, you can lose 1 and still failover.

### Sentinel Limitations

| Limitation | Why It Matters |
|-----------|---------------|
| **No sharding** | All data must fit on one primary. Can't scale writes. |
| **Asynchronous replication** | Promoted replica may be behind → data loss |
| **Client complexity** | Application needs a Sentinel-aware client library |
| **Failover time** | Seconds to minutes (detect failure + vote + promote) |
| **No multi-key atomic operations across nodes** | Because there's only one primary, this isn't an issue (but it is in Cluster) |

### When to Use Sentinel vs. Cluster

- **Dataset fits in one machine's RAM? → Sentinel** (simpler, no sharding complexity)
- **Dataset > single machine's RAM or need write scaling? → Redis Cluster**

---

## 8. Redis Cluster — Sharding & Horizontal Scaling

### Why Cluster Exists

Sentinel gives you HA but not horizontal scaling. If your dataset is 200GB and a single machine has 64GB RAM, Sentinel can't help. Redis Cluster shards data across multiple primary nodes.

### Hash Slots — How Sharding Works

Redis Cluster does NOT use consistent hashing. It uses **16,384 hash slots.**

```
Key → CRC16(key) % 16384 → Hash Slot → Node

Node A: Slots 0-5460
Node B: Slots 5461-10922
Node C: Slots 10923-16383
```

**Why 16,384 slots instead of consistent hashing?**
- Consistent hashing requires a hash ring and virtual nodes — more complex
- Hash slots make resharding simple: just move slot ranges from one node to another
- 16,384 is small enough to be represented as a bitmap (2KB) that every node can store
- Adding a new node = move some slots to it. Removing a node = move its slots away

### Cluster Topology

```
┌────────────┐  ┌────────────┐  ┌────────────┐
│ Primary A  │  │ Primary B  │  │ Primary C  │
│ Slots 0-5K │  │ Slots 5K-11K│ │ Slots 11K-16K│
│     │      │  │     │      │  │     │      │
│  Replica A │  │  Replica B │  │  Replica C │
└────────────┘  └────────────┘  └────────────┘
```

- Minimum: 3 primary nodes (each with 1 replica = 6 nodes total)
- Each primary owns a range of hash slots
- Each primary has 1+ replicas for HA
- Nodes communicate via **gossip protocol** (heartbeats, failure detection)

### Client Routing

1. Client sends command to any node
2. If the node owns the hash slot for that key → execute
3. If not → node returns `MOVED <slot> <ip:port>` redirect
4. Smart clients cache the slot map and route directly (avoiding the extra hop)

### Cross-Slot Operations (The Big Limitation)

**Operations touching multiple keys MUST be on the same hash slot.** Otherwise you get a `CROSSSLOT` error.

```
# This FAILS if key1 and key2 are on different slots:
MGET key1 key2

# Solution: Use hash tags to force keys to the same slot:
MGET {user:123}:name {user:123}:email
# {user:123} is the hash tag — only this part is hashed
```

**Why this limitation exists:** Redis Cluster has no distributed transaction protocol. Operations are atomic only within a single node. Cross-node atomicity would require 2PC (slow, complex).

**Tradeoff:** You must design your key schema carefully. Related keys should share a hash tag. But too many keys on one hash tag = hot slot.

### Resharding — Adding/Removing Nodes

1. Add new empty node to cluster
2. Move hash slots from existing nodes to the new node using `CLUSTER RESHARD`
3. During migration, if a client queries a key being migrated:
   - If key is still on source: `ASK` redirect (temporary)
   - If key already moved to target: `MOVED` redirect (permanent)
4. Clients experience no downtime during resharding

### Failover in Cluster

1. Nodes exchange heartbeats via gossip protocol
2. If a primary is unresponsive, other primaries vote on whether it's down
3. If majority agrees → the best replica is promoted to primary
4. The new primary takes over the hash slots
5. **No Sentinel needed** — Cluster has built-in failover

### Cluster Tradeoffs Summary

| Benefit | Tradeoff |
|---------|----------|
| Horizontal scaling (distribute data across nodes) | No cross-slot multi-key operations |
| Built-in HA (no Sentinel needed) | More nodes to manage (min 6) |
| Auto-resharding | Hash slot migration has a performance cost |
| Linear scalability up to 1000 nodes | Gossip protocol overhead increases with node count |
| Can handle datasets >> single machine RAM | Application must use hash tags for related keys |

---

## 9. Eviction Policies — What Happens When Memory Is Full

### Why Eviction Matters

Redis stores everything in RAM. When `maxmemory` is reached, Redis must decide: reject new writes or evict existing keys.

### Eviction Policies

| Policy | Evicts From | Algorithm | Best For |
|--------|------------|-----------|----------|
| `noeviction` | Nothing — returns errors on writes | N/A | Primary data store (no data loss) |
| `allkeys-lru` | All keys | Least Recently Used | **General caching (recommended default)** |
| `allkeys-lfu` | All keys | Least Frequently Used | Stable popular items (product catalog) |
| `volatile-lru` | Only keys with TTL | LRU | Mix of cache (with TTL) and permanent data (without TTL) |
| `volatile-lfu` | Only keys with TTL | LFU | Same as above, frequency-based |
| `volatile-ttl` | Only keys with TTL | Shortest TTL first | When TTL reflects priority |
| `allkeys-random` | All keys | Random | When all keys are equally likely to be accessed |
| `volatile-random` | Only keys with TTL | Random | Simple, low-overhead eviction |

### LRU vs. LFU — The Key Tradeoff

**LRU (Least Recently Used):**
- Evicts keys that haven't been accessed recently
- Good when recent access predicts future access
- **Weakness:** A rarely-used key that was just accessed once stays in cache, potentially evicting a frequently-used key that hasn't been accessed in the last few seconds

**LFU (Least Frequently Used):**
- Evicts keys with the lowest access frequency
- Good when some keys are consistently popular (product pages, user profiles)
- **Weakness:** Slow to adapt. A key that was popular yesterday but isn't today takes a long time to get evicted (mitigated by Redis's decay mechanism)

**Redis's LRU is approximated, not exact.** True LRU requires maintaining a global ordering of all keys by access time — expensive. Redis samples `maxmemory-samples` random keys (default 5) and evicts the least recently used among the sample. Increasing the sample size (e.g., 10) gives more accurate LRU at the cost of CPU.

**Redis's LFU uses a probabilistic counter (Morris counter)** stored in just 8 bits per key, with a 16-bit decay timestamp. This means it uses negligible extra memory but provides approximate frequency tracking.

### Practical Decision

```
Using Redis as a pure cache?       → allkeys-lru (safest default)
Have a mix of cache + persistent?  → volatile-lru (protect keys without TTL)
Some keys are always popular?      → allkeys-lfu (keeps popular keys)
Don't want ANY data loss?          → noeviction (but monitor memory!)
```

---

## 10. Caching Patterns — How to Use Redis as a Cache

### Cache-Aside (Lazy Loading) — The Most Common Pattern

```
Read:
1. App checks Redis for key
2. Cache HIT → return data
3. Cache MISS → query database → store in Redis (with TTL) → return data

Write:
1. App writes to database
2. App DELETES the Redis key (invalidate)
   (Don't update the cache — delete it. Next read will repopulate.)
```

**Why this is the most popular:**
- Application controls what gets cached and when
- Only requested data is cached (lazy — no wasted memory)
- Cache misses are self-healing (cache repopulates on next read)

**Tradeoff:**
- First request for a key is always a cache miss (cold start)
- If database is updated but cache isn't invalidated → stale data
- "Delete on write" can cause brief cache-miss storms after writes

### Write-Through

```
Write:
1. App writes to Redis AND database simultaneously
2. Redis always has fresh data

Read:
1. Always read from Redis
```

**Why use it:** No stale data (cache is always up to date). No cache miss penalty.
**Tradeoff:** Every write is slower (two writes). You cache data that may never be read (wasted memory).

### Write-Behind (Write-Back)

```
Write:
1. App writes to Redis only
2. A background process asynchronously writes to the database

Read:
1. Always read from Redis
```

**Why use it:** Fastest writes (only Redis). Batches database writes for efficiency.
**Tradeoff:** **Data loss risk** — if Redis crashes before the background process writes to DB, data is lost. Complex to implement correctly. Race conditions between background writer and direct DB reads.

### Read-Through

```
Read:
1. App asks Redis for data
2. If miss, Redis itself fetches from database, caches, and returns
```

**Why use it:** Encapsulates caching logic in the cache layer (clean separation).
**Tradeoff:** Redis doesn't natively support this — you need a proxy or middleware (like Redis Gears or application-level abstraction).

### Pattern Decision Framework

| Pattern | Cache Miss Penalty | Stale Data Risk | Write Latency | Complexity |
|---------|-------------------|-----------------|---------------|------------|
| Cache-Aside | High (first read) | Medium (until invalidation) | Low (DB only) | Low |
| Write-Through | None | None | High (Redis + DB) | Medium |
| Write-Behind | None | None | Lowest (Redis only) | High (data loss risk) |

---

## 11. Cache Invalidation — The Hardest Problem

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### Why It's Hard

When the source of truth (database) changes, the cache must be updated or invalidated. If you forget to invalidate, users see stale data. If you invalidate too aggressively, you lose cache benefits.

### Strategies

**TTL-Based Expiration (Simplest):**
- Set a TTL on every cached key (`SETEX key 300 value`)
- Data goes stale for at most TTL seconds
- **Tradeoff:** Stale data for up to TTL duration. Too short TTL = too many cache misses. Too long = too stale.

**Event-Driven Invalidation (Best for Consistency):**
- When the database changes, publish an event
- A subscriber invalidates (deletes) the relevant cache key
- **Tradeoff:** Requires event infrastructure (Kafka, DB triggers, CDC). More complex. But gives near-real-time consistency.

**Cache-Aside Delete on Write:**
- On every database write, delete the corresponding cache key
- Next read will repopulate from the database
- **Why delete instead of update?** Avoids race conditions. Consider: two concurrent writes (W1 then W2) where W1's cache update is slower. With "update cache," the cache could end up with W1's stale value. With "delete cache," the next read always gets the freshest data.

### The Thundering Herd Problem

When a popular cache key expires, hundreds of concurrent requests all miss the cache simultaneously and hammer the database.

**Solutions:**
1. **Locking:** The first request that misses acquires a distributed lock, fetches from DB, populates cache. Others wait.
2. **Stale-while-revalidate:** Serve slightly stale data while refreshing in the background. (Use TTL + a "soft TTL" — if soft TTL expired, return stale data AND trigger background refresh.)
3. **Pre-warming:** Refresh popular keys before their TTL expires.

---

## 12. Pub/Sub & Streams

### Pub/Sub

Fire-and-forget messaging. Publishers send messages to channels; subscribers receive them in real-time.

```
SUBSCRIBE news:sports          # Consumer subscribes
PUBLISH news:sports "Goal!"    # Producer publishes
```

**Limitations (critical for interviews):**
- **No persistence:** If no subscriber is listening when a message is published, it's lost forever.
- **No acknowledgment:** No way to know if a subscriber processed the message.
- **No consumer groups:** Every subscriber gets every message (no work distribution).
- **No replay:** Can't read old messages.

**When to use:** Real-time notifications, chat rooms, live updates — where losing occasional messages is acceptable.

### Streams (Redis 5.0+)

Persistent, append-only log with consumer groups. Think "lightweight Kafka."

**Advantages over Pub/Sub:**
- Messages persist (configurable retention)
- Consumer groups (distribute work across consumers)
- Acknowledgment (XACK)
- Message replay (read from any position)
- Blocking reads (XREADGROUP BLOCK)

**When Streams vs. Kafka:**
- Streams: Simple use cases, small scale, already using Redis, don't want to operate Kafka
- Kafka: High throughput (1M+ msg/s), long retention, complex stream processing, enterprise features

---

## 13. Distributed Locking with Redis

### Why Redis for Locks?

Distributed systems need mutual exclusion (only one process does X at a time). Redis's `SET NX EX` provides an atomic "set if not exists with expiry" — the basic building block.

### Simple Lock (Single Instance)

```
# Acquire
SET lock:resource <unique_token> NX EX 30  # Set only if not exists, expire in 30s

# Release (Lua script for atomicity)
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

**Why the unique token?** To prevent accidentally releasing someone else's lock. Only the holder (who knows the token) can release it.

**Why the expiry?** If the holder crashes, the lock auto-releases after 30 seconds instead of being held forever (deadlock prevention).

### Redlock (Distributed — Multiple Instances)

For a single Redis instance, the lock is a SPOF. Redlock uses N independent Redis instances (typically 5):

1. Try to acquire the lock on all N instances
2. If acquired on a majority (N/2 + 1) within a timeout, the lock is acquired
3. Release by deleting from all instances

**The controversy:** Martin Kleppmann argued that Redlock is unsafe due to clock skew, GC pauses, and network delays. The Redis creator (Salvatore Sanfilippo) disagreed. In practice, for most applications, a simple single-instance lock with TTL is sufficient.

**Tradeoff:** Simple lock (single Redis) = easy but SPOF. Redlock = more resilient but complex and controversial. For truly critical distributed locks, consider ZooKeeper or etcd (consensus-based, not timeout-based).

---

## 14. Transactions & Lua Scripting

### MULTI/EXEC Transactions

```
MULTI
SET user:123:balance 100
INCR user:123:login_count
EXEC
```

- Commands between MULTI and EXEC are queued and executed atomically
- **No rollback:** If one command fails, others still execute. This is NOT like database transactions.
- **Optimistic locking:** Use `WATCH` to monitor keys. If a watched key changes before EXEC, the transaction is aborted.

**Tradeoff:** No rollback means you must handle partial failures. No isolation levels (other clients can read intermediate state between commands in different transactions, but since execution is single-threaded, individual transactions are atomic).

### Lua Scripting (The Better Option)

```lua
-- Atomic check-and-set
local current = redis.call('GET', KEYS[1])
if current == ARGV[1] then
    redis.call('SET', KEYS[1], ARGV[2])
    return 1
else
    return 0
end
```

**Why Lua is preferred over MULTI/EXEC:**
- Full programming logic (conditionals, loops)
- Truly atomic (the script runs as a single command — no other commands can interleave)
- Can read and write in the same atomic operation
- Reduces round trips (one call vs. MULTI + N commands + EXEC)

**Tradeoff:** A long-running Lua script blocks all other clients (single-threaded). Keep scripts short.

---

## 15. Pipelining — Reducing Round Trips

### The Problem

Each Redis command requires a network round trip (~0.5-1ms on LAN). If you need 100 commands, that's 50-100ms of pure network latency.

### The Solution

Pipeline multiple commands in a single round trip:

```python
pipe = redis.pipeline()
pipe.set("key1", "value1")
pipe.set("key2", "value2")
pipe.get("key3")
results = pipe.execute()  # One round trip for all 3 commands
```

**Speed gain:** Instead of 100 round trips, you do 1. For batch operations, pipelining can give 5-10x throughput improvement.

**Tradeoff:** Commands in a pipeline are NOT atomic (unless wrapped in MULTI/EXEC). They're just batched for network efficiency. Results come back in order.

---

## 16. Common Redis Patterns in System Design

### 1. Rate Limiter (Sliding Window)
```
Use a sorted set where score = timestamp:
ZADD rate:user123 <now> <request_id>
ZREMRANGEBYSCORE rate:user123 0 <now - window>
ZCARD rate:user123  # Count requests in window
```
If count > limit → reject. All atomic with Lua script.

### 2. Leaderboard
```
ZADD leaderboard <score> <player_id>
ZREVRANGE leaderboard 0 9 WITHSCORES  # Top 10
ZRANK leaderboard <player_id>         # Player's rank
```
O(log N) for updates and rank queries. Scales to millions of players.

### 3. Session Store
```
SETEX session:<token> 3600 <serialized_session_data>
```
Auto-expiry via TTL. Shared across all app instances. Fast reads.

### 4. Distributed Counter
```
INCR page_views:article:456       # Atomic increment
INCRBY daily:revenue 2599         # Increment by amount
```

### 5. Pub/Sub for Real-Time Notifications
```
SUBSCRIBE user:123:notifications
PUBLISH user:123:notifications '{"type":"message","from":"user456"}'
```

### 6. Proximity Search
```
GEOADD drivers -73.935 40.730 "driver1"
GEORADIUS drivers -73.935 40.730 5 km COUNT 10 ASC
```
Find 10 nearest drivers within 5km.

### 7. Bloom Filter (via RedisBloom module)
```
BF.ADD seen_urls "https://example.com/page1"
BF.EXISTS seen_urls "https://example.com/page1"  # 1 (probably yes)
BF.EXISTS seen_urls "https://example.com/page2"  # 0 (definitely no)
```
Probabilistic: false positives possible, false negatives impossible. Memory-efficient deduplication.

---

## 17. Redis vs Memcached — When to Use What

| Dimension | Redis | Memcached |
|-----------|-------|-----------|
| Data structures | Strings, hashes, lists, sets, sorted sets, streams, geo... | Strings only |
| Persistence | RDB, AOF, hybrid | None |
| Replication | Built-in primary-replica | None (client-side) |
| Clustering | Redis Cluster (hash slots) | Client-side consistent hashing |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Transactions | MULTI/EXEC | No |
| Threading | Single-threaded commands, multi-threaded I/O | Multi-threaded |
| Memory efficiency | Higher overhead per key (object metadata) | Lower overhead (slab allocator) |
| Max value size | 512 MB | 1 MB (default) |

**Use Memcached when:** Simple key-value caching, multi-threaded performance is critical, minimal memory overhead per key matters, you don't need data structures or persistence.

**Use Redis when:** Basically everything else. Rich data structures, persistence, replication, Pub/Sub, scripting, sorted sets, and you want a single tool for many use cases.

**In practice:** Redis has largely replaced Memcached for new projects. Memcached still exists in legacy systems and specific high-throughput caching-only scenarios.

---

## 18. Anti-Patterns & Common Mistakes

### 1. Using `KEYS *` in Production
**Why it's bad:** Scans the entire keyspace. O(N) where N = total keys. Blocks the single thread for seconds on large databases.
**Fix:** Use `SCAN` (cursor-based, non-blocking iteration).

### 2. Storing Large Values (>100KB)
**Why it's bad:** Large values increase network latency, memory fragmentation, and slow down persistence.
**Fix:** Split large objects into smaller chunks or store in a blob store with Redis holding a reference.

### 3. Not Setting TTLs on Cache Keys
**Why it's bad:** Memory grows unbounded. Eventually, you hit `maxmemory` and either get evictions (unpredictable) or errors (`noeviction`).
**Fix:** Always set TTLs on cache data. Use eviction policies as a safety net, not a primary strategy.

### 4. Hot Key Problem
**Why it's bad:** One key receiving enormous traffic (e.g., a viral tweet's like count) saturates a single Redis node.
**Fix:** Client-side caching for hot keys, add local in-memory cache (L1 cache + Redis L2 cache), or shard the key (e.g., `counter:{tweet_id}:{shard_N}` across N keys, sum on read).

### 5. Using Redis as Primary Database Without Backups
**Why it's bad:** Memory is volatile. Even with AOF, you're one misconfiguration away from data loss.
**Fix:** Always have a persistent database as the source of truth. Use Redis as a performance layer, not the only storage.

### 6. Ignoring Memory Overhead
**Why it's bad:** A key with a 10-byte value actually uses ~80+ bytes (key metadata, encoding, pointers, allocator overhead). 1 million small keys can use 80+ MB beyond the actual data.
**Fix:** Use hashes to group small related keys (e.g., `HSET user:123 name "Alice" age 30` instead of `SET user:123:name "Alice"` and `SET user:123:age 30`). Hashes with <128 fields use ziplist encoding (very compact).

### 7. Running Long Lua Scripts
**Why it's bad:** Single-threaded. A 5-second Lua script blocks all clients for 5 seconds.
**Fix:** Keep scripts short and simple. Offload complex logic to the application.

---

## 19. Operational Best Practices

### Memory Management
- Set `maxmemory` explicitly (don't let Redis consume all RAM — leave room for OS, fork, etc.)
- Reserve 2x your dataset size for COW during persistence
- Monitor `used_memory` vs `maxmemory` with alerts at 80%

### Deployment Recommendations
- **Don't co-locate with memory-hungry apps** (page cache competition, OOM killer risk)
- **Disable THP (Transparent Huge Pages)** — causes latency spikes during fork
- **Use `vm.overcommit_memory=1`** — prevents fork failures when memory is tight
- **Production minimum:** 3 Sentinel nodes or 6 Cluster nodes (3 primary + 3 replica)

### Monitoring Key Metrics
- `used_memory` / `maxmemory` — memory pressure
- `evicted_keys` — if non-zero, you're running out of memory
- `keyspace_hits` / `keyspace_misses` — cache hit ratio (aim for >90%)
- `connected_clients` — connection count (watch for leaks)
- `instantaneous_ops_per_sec` — throughput
- `latest_fork_usec` — fork duration (should be <1 second)
- `repl_backlog_active` + replica lag — replication health

---

## 20. Interview Questions — Deep Answers with Tradeoffs (50+)

### 🟢 Level 1 — Fundamentals

**Q1: What is Redis? Why is it so fast?**

Redis is an in-memory data structure store. It's fast for five compounding reasons: (1) All data is in RAM — memory access is ~10,000x faster than disk. (2) Single-threaded command execution — no lock contention or context switching. (3) I/O multiplexing via epoll/kqueue — one thread efficiently handles thousands of connections. (4) Efficient data structures with compact encodings — small objects use cache-friendly contiguous memory (ziplist, intset). (5) Simple operations — most commands are O(1) or O(log N), completing in microseconds.

The tradeoff for all this speed: data size is limited by RAM (expensive), no complex queries, and weaker durability guarantees than disk-based databases.

---

**Q2: If Redis is single-threaded, how can it handle thousands of concurrent clients?**

It uses an event-driven I/O model with the OS's multiplexing facility (epoll on Linux). The single thread doesn't dedicate itself to one client at a time — it listens on all sockets simultaneously. When any socket has data ready, the event loop reads the command, executes it (microseconds), writes the response, and moves to the next ready socket.

Since each command executes in microseconds, one thread can process 100K+ commands per second. The bottleneck is network I/O, not CPU — which is why Redis 6.0 added I/O threads for network read/write while keeping command execution single-threaded.

---

**Q3: What data structures does Redis support? Give a real-world use case for each.**

Strings (session tokens, counters, feature flags), Hashes (user profiles — `HSET user:123 name "Alice" age 30`), Lists (task queues — LPUSH/BRPOP for producer/consumer), Sets (tracking unique visitors — SADD/SCARD), Sorted Sets (leaderboards — ZADD score, ZRANK), Bitmaps (daily active users — SETBIT user_id), HyperLogLog (approximate unique counts — 12KB for billions of elements), Streams (event logs with consumer groups), Geospatial (find nearby restaurants — GEORADIUS).

The key insight: these aren't just storage types — they're server-side operations. Instead of "fetch data, transform in app, write back" (3 round trips), you do "ZADD" or "LPUSH" (1 round trip, atomic).

---

**Q4: What's the difference between `SET key value EX 60` and `SETEX key 60 value`?**

Functionally identical — both set a key with a 60-second expiry. `SET` with options is the modern, preferred approach because it also supports `NX` (set if not exists), `XX` (set if exists), and `PX` (millisecond expiry) in one command. `SETEX` is the older, dedicated command.

**Interview insight:** This is often asked to check if you know about atomic expiry setting vs. doing `SET` then `EXPIRE` separately (which is NOT atomic — if you crash between the two, you have a key without expiry = memory leak).

---

**Q5: What happens when Redis runs out of memory?**

Depends on the `maxmemory-policy`:
- `noeviction` (default for non-cache use): Returns error on new writes. Reads still work.
- `allkeys-lru`: Evicts least recently used keys to make room.
- Other policies: Evict based on LFU, TTL, random, etc.

If no `maxmemory` is set, Redis will use all available system memory until the OOM killer terminates it. Always set `maxmemory` in production.

---

### 🟡 Level 2 — Intermediate

**Q6: Compare RDB and AOF persistence. When would you use each?**

**RDB:** Periodic point-in-time snapshots. Uses fork() + COW. Compact file, fast restart. But data loss between snapshots (minutes).

**AOF:** Append every write command to a log. Configurable sync (every second / every write / OS decides). Larger file, slower restart (must replay commands). But minimal data loss (1 second with `everysec`).

**The fork problem applies to both:** RDB does fork for snapshot. AOF does fork for rewrite (to compact the file). Both suffer from COW memory pressure on large datasets.

**Recommendation:** Use hybrid persistence (Redis 4.0+). RDB snapshot as base + AOF for incremental changes. Fast restart (load RDB) + minimal data loss (replay AOF tail).

**Tradeoff the interviewer wants:** "If you're using Redis purely as a cache that can be rebuilt from the database, disable persistence entirely — it's the fastest and simplest option."

---

**Q7: Explain Redis replication. What's the consistency model?**

Redis replication is asynchronous. The primary sends a write ack to the client BEFORE replicas receive it. This means:

1. A write can be acked and then lost if the primary crashes before replication
2. Replicas always serve slightly stale data
3. There's no built-in way to guarantee a read-after-write consistency

The `WAIT` command provides partial mitigation — it blocks until N replicas acknowledge, but it's not truly synchronous (the primary might crash after WAIT returns).

**Why async?** Synchronous replication would require waiting for the slowest replica on every write — destroying Redis's sub-millisecond latency. The design consciously trades consistency for speed.

**Interview insight:** "Redis provides eventual consistency. If you need strong consistency, Redis is not the right choice as a primary data store. Use it as a cache in front of a strongly consistent database."

---

**Q8: What's the difference between Redis Sentinel and Redis Cluster?**

| Aspect | Sentinel | Cluster |
|--------|----------|---------|
| Purpose | High availability (auto-failover) | HA + horizontal scaling (sharding) |
| Data distribution | All data on one primary | Data sharded across multiple primaries |
| Write scaling | No (single primary) | Yes (multiple primaries) |
| Multi-key operations | Full support | Only within same hash slot |
| Minimum nodes | 3 Sentinels + 1 primary + 1 replica | 6 (3 primary + 3 replica) |
| Complexity | Moderate | High |
| When to use | Data fits in one machine's RAM | Data exceeds one machine's RAM |

**The decision:** If your data fits in one machine (say, < 50-100 GB), use Sentinel. Simpler operations, full multi-key support, easier to reason about. Only go to Cluster when you genuinely need sharding.

---

**Q9: How does Redis Cluster shard data? Why hash slots instead of consistent hashing?**

Redis Cluster uses 16,384 hash slots. `CRC16(key) % 16384` maps each key to a slot. Each node owns a range of slots.

**Why not consistent hashing?**
1. **Resharding is simpler.** Move slot ranges from one node to another. No need to recompute the hash ring or manage virtual nodes.
2. **Metadata is compact.** The slot-to-node map is a 16KB bitmap. Every node can store it. Clients cache it for direct routing.
3. **Predictable distribution.** Each node gets a deterministic, configurable number of slots. No "virtual node" complexity.

**The tradeoff:** Cross-slot operations are forbidden. This means `MGET key1 key2` fails if key1 and key2 hash to different slots. You must use hash tags (`{user:123}:name`) to colocate related keys.

---

**Q10: Explain the hot key problem. How do you solve it?**

A "hot key" is a single key receiving disproportionate traffic. Since Redis Cluster assigns one key to one node, all traffic hits that one node — creating a bottleneck even if the rest of the cluster is idle.

**Examples:** A viral tweet's like count, a celebrity user's profile, a flash sale product page.

**Solutions:**

| Solution | How | Tradeoff |
|----------|-----|----------|
| **Client-side cache (L1)** | Cache hot key in application memory (with short TTL) | Stale data for TTL duration. Extra memory in app. |
| **Read replicas** | Route reads to replicas for this key | Stale reads. Sentinel/Cluster needed. |
| **Key sharding** | Split `counter:tweet:123` into `counter:tweet:123:{0..9}`. Sum on read. | Application complexity. Reads become 10 GETs + sum. |
| **Local caching proxy** | Use Redis-side client caching (Redis 6.0+ server-assisted client cache) | Added infrastructure. |

---

**Q11: What is the thundering herd problem? How do you prevent it?**

When a popular cache key expires, hundreds of concurrent requests all miss the cache simultaneously and slam the database.

**Prevention strategies:**

1. **Distributed lock on cache miss:** First thread to miss acquires a lock (`SET lock:key NX EX 5`), fetches from DB, populates cache. Other threads either wait (with backoff) or return a stale value.

2. **Stale-while-revalidate:** Cache has two TTLs — a "soft TTL" (e.g., 5 min) and a "hard TTL" (e.g., 10 min). After soft TTL, serve stale data AND trigger a background refresh. After hard TTL, mandatory refresh.

3. **Early expiration randomization:** Add jitter to TTLs (`TTL = base + random(0, 60s)`) so keys don't all expire simultaneously.

4. **Pre-warming:** Background process refreshes popular keys before they expire.

---

### 🟠 Level 3 — Advanced

**Q12: Walk me through what happens during Redis fork() for BGSAVE. Why is it problematic?**

1. Redis calls `fork()`. The OS creates a child process.
2. The child gets a copy of the parent's virtual memory space (same physical pages via copy-on-write).
3. The child starts writing the dataset to an RDB file on disk.
4. The parent continues serving clients.
5. When the parent modifies a memory page, the OS copies that page (COW) so the child still sees the original snapshot.

**Problems:**

- **Fork latency:** Copying the page table takes time proportional to the virtual memory size. For 50GB dataset, this can take 1-2 seconds. During this time, Redis is FROZEN — no commands processed.
- **Memory spike:** Under heavy writes, many pages are modified (COW copies). In the worst case, memory usage temporarily doubles.
- **Transparent Huge Pages (THP):** If enabled, the OS uses 2MB pages instead of 4KB. COW on a 2MB page for a single byte change wastes 2MB. Redis recommends disabling THP.

**Mitigations:**
- Disable THP
- Keep dataset size per instance reasonable (<25GB ideal)
- Use replicas for persistence (disable persistence on primary)
- Schedule BGSAVE during low-traffic periods
- Ensure enough RAM headroom (2x dataset for worst-case COW)

---

**Q13: How does Cluster failover work? Walk through the steps.**

1. **Failure detection:** Nodes exchange heartbeats via gossip protocol every second. If Node A doesn't receive a heartbeat from Node B for `cluster-node-timeout` ms, Node A marks B as "possibly failed" (PFAIL).

2. **Consensus:** Node A gossips B's PFAIL status to other nodes. If a majority of master nodes agree B is unreachable, B is marked as FAIL.

3. **Replica election:** If B was a master, its replica(s) start an election. The replica with the most up-to-date replication offset sends a vote request to all masters.

4. **Voting:** Masters that own hash slots vote. The replica needs a majority of votes.

5. **Promotion:** The winning replica promotes itself to master, takes over B's hash slots, and announces this to the cluster.

6. **Client redirect:** Clients sending commands for B's slots get `MOVED` redirects to the new master.

**Total failover time:** Typically 1-3 seconds. Configurable via `cluster-node-timeout`.

**Data loss risk:** Since replication is async, the promoted replica may be behind the failed master. Recently written data can be lost.

---

**Q14: How would you design Redis for an application that needs both caching AND persistent data?**

**Use `volatile-lru` eviction policy:**
- Cache keys: Set with TTL (`SETEX cache:user:123 300 "data"`)
- Persistent keys: Set without TTL (`SET config:feature_flag "enabled"`)
- When memory is full, only keys WITH TTL are evicted. Persistent keys are protected.

**Better approach: Separate Redis instances:**
- Instance 1: Cache (allkeys-lru, no persistence, can be flushed)
- Instance 2: Persistent data (noeviction, AOF persistence)

**Why separation is better:**
- Cache eviction doesn't accidentally delete persistent data
- Cache can be tuned for max performance (no persistence overhead)
- Persistent instance can have different HA requirements
- Clear operational boundaries

---

**Q15: You need to count unique visitors for a website with 100M daily users. How?**

**Option 1: Sets**
```
SADD unique:visitors:2026-03-31 "user_id_123"
SCARD unique:visitors:2026-03-31
```
Accurate but memory-hungry: 100M user IDs × ~50 bytes = ~5GB per day.

**Option 2: HyperLogLog**
```
PFADD unique:visitors:2026-03-31 "user_id_123"
PFCOUNT unique:visitors:2026-03-31
```
Fixed 12 KB per HyperLogLog regardless of cardinality. 100M users = still 12 KB.
Error rate: ~0.81%.

**Option 3: Bitmaps (if user IDs are sequential integers)**
```
SETBIT unique:visitors:2026-03-31 12345 1
BITCOUNT unique:visitors:2026-03-31
```
100M max user ID = 12.5 MB. Exact count. Very fast.

**Decision:**
- Need exact count AND have integer IDs → Bitmap
- Need approximate count, memory is critical → HyperLogLog
- Need to retrieve the actual members later → Set

---

**Q16: Design a rate limiter using Redis.**

**Fixed Window:**
```
key = "rate:user123:minute:202603311430"
INCR key
EXPIRE key 60  (only if new)
if count > limit: reject
```
**Problem:** Boundary spike. 100 requests at 14:30:59 + 100 requests at 14:31:01 = 200 requests in 2 seconds but passes both 1-minute windows.

**Sliding Window (Sorted Set):**
```lua
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local key = KEYS[1]

-- Remove entries outside window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
-- Count entries in window
local count = redis.call('ZCARD', key)
if count < limit then
    redis.call('ZADD', key, now, now .. math.random())
    redis.call('EXPIRE', key, window)
    return 1  -- allowed
else
    return 0  -- rejected
end
```
**More accurate** but higher memory per user (stores each request timestamp).

**Token Bucket (String + Lua):**
Store `{tokens, last_refill_time}`. On each request, calculate tokens since last refill, add to bucket (up to max), consume 1 token.

---

### 🔴 Level 4 — System Design Scenarios

**Q17: Design a real-time leaderboard for a game with 10M players.**

**Data structure:** Sorted Set.
```
ZADD leaderboard <score> <player_id>
```

**Operations (all O(log N)):**
- Update score: `ZADD leaderboard 1500 "player123"`
- Get rank: `ZREVRANK leaderboard "player123"` → 0-based rank
- Top 10: `ZREVRANGE leaderboard 0 9 WITHSCORES`
- Rank range: `ZREVRANGE leaderboard 100 109` → players ranked 101-110

**Scaling considerations:**
- 10M members in a sorted set ≈ ~1-2 GB RAM (player IDs + scores). Fits on one instance.
- For 100M+ players or global distribution, shard by region or game mode.
- For real-time score updates from game servers, use pipelining to batch updates.

**Why Redis over a database?** `ORDER BY score DESC LIMIT 10` on 10M rows requires an index scan. In Redis, `ZREVRANGE 0 9` is O(log N + 10) — sub-millisecond.

---

**Q18: Design a session store for a web application with 50M active sessions.**

**Design:**
```
SETEX session:<token> 3600 <serialized_session_data>
```

**Sizing:** 50M sessions × 1KB avg = 50 GB. Needs either a large single instance or Redis Cluster.

**Key decisions:**
- **TTL = session timeout** (e.g., 1 hour). Idle sessions auto-expire.
- **Partition by session token** in Cluster mode (natural even distribution — tokens are random).
- **Persistence:** AOF `everysec`. Sessions are important but can be regenerated (user logs in again).
- **HA:** Sentinel (if < ~100GB) or Cluster (if >100GB). Auto-failover prevents session loss.
- **Why not a database?** Session reads happen on every single HTTP request. Hitting a database for 50M concurrent sessions would require enormous read capacity. Redis handles this with sub-ms latency.

**Tradeoff:** If Redis crashes and AOF loses the last second, some users get logged out. Acceptable for most applications.

---

**Q19: Design a distributed cache for a microservices architecture.**

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Service A │     │ Service B │     │ Service C │
│ (L1 cache)│     │ (L1 cache)│     │ (L1 cache)│
└─────┬─────┘     └─────┬─────┘     └─────┬─────┘
      │                 │                 │
      └────────────┬────┴────────────────┘
                   │
           ┌───────▼───────┐
           │  Redis Cluster  │ (L2 cache)
           │  (Shared cache) │
           └───────┬───────┘
                   │
           ┌───────▼───────┐
           │   PostgreSQL    │ (Source of truth)
           └───────────────┘
```

**L1 (in-process):** Each service has a small in-memory cache (Caffeine/Guava). Short TTL (30s). Absorbs hot key traffic. No network round trip.

**L2 (Redis):** Shared across all service instances. Longer TTL (5 min). Prevents duplicate DB queries across services.

**Cache-aside pattern:** Check L1 → Miss → Check Redis → Miss → Query DB → Populate Redis → Populate L1 → Return.

**Invalidation:** On write to DB, delete from Redis. L1 expires naturally via short TTL. For tighter consistency, publish invalidation events via Redis Pub/Sub.

**Key design for Cluster:** Namespace keys by service: `svc-a:user:123`. Use hash tags for related keys: `{user:123}:profile`, `{user:123}:prefs`.

---

**Q20: How would you handle a cache stampede during a flash sale event?**

**Scenario:** 100K users simultaneously try to view a product that just went on sale. The product cache key expires at the worst possible moment.

**Solution (layered):**

1. **Pre-warm the cache:** Before the sale starts, populate the product cache with a long TTL.

2. **Stale-while-revalidate:** Use a soft TTL (background refresh) so the key never truly expires during the sale. Implementation: store `{data, refresh_at}` in the value. If `now > refresh_at`, return stale data and async refresh.

3. **Lock-based refresh:** If the key IS missing, use a distributed lock. First request acquires lock, fetches from DB, populates cache. Other requests either wait (with timeout) or get a "try again in 100ms" response.

4. **Request coalescing:** Application-level: if 100 concurrent requests all miss the same key, only one actually queries the DB. Others subscribe to the result. (Library-level feature, e.g., singleflight in Go.)

5. **Circuit breaker on DB:** If the DB is being overwhelmed, stop sending cache-miss queries and return an error/degraded response. Protect the database.

---

### ⚫ Level 5 — Deep Dive / Curveball

**Q21: Redis says it's single-threaded but uses I/O threads. Explain the threading model.**

The command execution pipeline has three phases: (1) Read request from socket, (2) Execute command, (3) Write response to socket.

Before Redis 6.0, all three happened in the single main thread. Since 6.0, phases (1) and (3) can be offloaded to I/O threads. Phase (2) — command execution — remains on the single main thread.

**Why not multi-thread execution?** Because Redis's atomicity guarantees are built on single-threaded execution. Making execution multi-threaded would require locks on every data structure, which would add overhead and complexity, likely making it slower for typical workloads (where operations are microseconds).

**When I/O threads help:** High connection count (10K+) with many small commands. The network parsing/serialization becomes the bottleneck, not execution.

---

**Q22: Explain the Redlock controversy. Why did Martin Kleppmann say it's unsafe?**

**Kleppmann's argument:** Redlock relies on timing assumptions. If a client acquires the lock, then experiences a long GC pause (or network delay), the lock's TTL might expire before the client finishes its work. Another client acquires the lock. Now two clients hold the lock simultaneously.

**Specifically:** Redlock doesn't use fencing tokens (monotonically increasing tokens that the resource being locked can use to reject stale requests). Without fencing, the system is unsafe under timing anomalies.

**Salvatore's counter-argument:** If you set the TTL much longer than expected processing time, the probability of this failure is negligible. Also, adding fencing tokens solves the issue but requires the protected resource to support them.

**Practical takeaway:** For most applications, a simple single-instance lock with TTL is sufficient. For truly critical mutual exclusion, use a consensus-based system (ZooKeeper, etcd) that doesn't depend on timing.

---

**Q23: How would you migrate from a single Redis instance to a Redis Cluster with zero downtime?**

1. **Set up the Cluster** alongside the existing instance (new hardware).
2. **Dual-write:** Application writes to BOTH the old instance and the new Cluster.
3. **Backfill:** Migrate existing data from old instance to Cluster (using `SCAN` + `RESTORE`, or a migration tool).
4. **Switch reads:** Once data is synced, point reads to the Cluster.
5. **Remove old writes:** Stop writing to the old instance.
6. **Decommission** the old instance.

**Challenges:**
- Hash tags: Your key schema must support Cluster's hash slot routing. Keys like `user:123:profile` and `user:123:prefs` need to use `{user:123}` as the hash tag if they're used in multi-key operations.
- Cross-key operations: `MGET key1 key2` works on a single instance but fails in Cluster if keys are on different slots. Must refactor application code.
- Lua scripts: Scripts accessing multiple keys must ensure all keys are on the same slot.

---

**Q24: What's the difference between Redis's approximate LRU and a true LRU?**

**True LRU:** Maintains a doubly-linked list of all keys ordered by access time. On every access, move the key to the head. On eviction, remove from the tail. O(1) per operation but requires a pointer per key (memory overhead) and lock contention in multi-threaded implementations.

**Redis's approximate LRU:** Doesn't maintain a global list. Instead, when eviction is needed, Redis randomly samples `maxmemory-samples` keys (default 5) and evicts the one with the oldest access time among the sample.

**Why approximate?** A global LRU list for millions of keys would add significant memory overhead (two pointers per key = 16 bytes × millions). The approximation uses zero extra memory per key — just a 24-bit timestamp field that already exists in every Redis object.

**How good is the approximation?** With `maxmemory-samples=5`, it's surprisingly close to true LRU. With `maxmemory-samples=10`, it's nearly identical. Redis published empirical comparisons showing the approximation works well in practice.

---

**Q25: Design a real-time chat system using Redis.**

**Architecture:**
- **Message routing:** Redis Pub/Sub for real-time delivery. Each user subscribes to their channel: `chat:user:123`.
- **Message persistence:** Redis Streams or an external database for message history.
- **Online status:** Redis Set (`SADD online_users "user123"`) with TTL heartbeat.
- **Typing indicators:** Redis Pub/Sub (fire-and-forget, no persistence needed).
- **Recent messages:** Redis Sorted Set (score = timestamp) or Stream per conversation, capped to last 100 messages.

**Scaling:**
- Pub/Sub doesn't work across Cluster shards natively. Use `SSUBSCRIBE` (shard channels in Redis 7.0+) or a dedicated messaging layer.
- For millions of concurrent users, shard by conversation or user ID across multiple Cluster nodes.
- Use Streams instead of Pub/Sub if you need message durability and consumer groups.

**Why Redis over Kafka for chat?** Chat needs sub-millisecond delivery. Redis Pub/Sub has lower latency than Kafka. But for persistent message history with replay, Kafka or a database is better.

---

### 💡 Rapid Fire

| Question | Answer |
|----------|--------|
| Max key size? | 512 MB (but keep keys short — every byte is stored in memory) |
| Max value size? | 512 MB |
| Is Redis CP or AP? | Neither cleanly. Single instance = CP-ish (no partition). Cluster with async replication = leans AP (favors availability over consistency). |
| What is the `WAIT` command? | Blocks until N replicas confirm a write. Partial synchronous replication. Doesn't prevent data loss if primary crashes after WAIT returns. |
| What is `OBJECT ENCODING key`? | Shows the internal encoding (ziplist, hashtable, skiplist, intset, etc.). Useful for memory optimization. |
| What is `SCAN`? | Cursor-based, non-blocking key iteration. Use instead of `KEYS *`. Guarantees eventual completion without blocking. |
| What is client-side caching? | Redis 6.0+ feature. Server tracks which keys a client has cached locally and sends invalidation messages when those keys change. Reduces network round trips. |
| What is `UNLINK` vs `DEL`? | `DEL` deletes synchronously (blocks if key is large). `UNLINK` deletes asynchronously in a background thread. Use `UNLINK` for large keys. |
| What is Redis Modules? | Extension system for adding custom data types and commands (RediSearch, RedisJSON, RedisBloom, RedisTimeSeries). |
| Single instance max throughput? | ~100K-200K ops/sec (simple commands, no persistence). Up to 500K+ with pipelining. |

---

### Quick Self-Test

1. Why is Redis single-threaded for commands, and what's the tradeoff?
2. Explain RDB vs AOF persistence — when would you choose each?
3. What happens during fork() and why is it problematic for large datasets?
4. How does Redis Cluster shard data? Why hash slots instead of consistent hashing?
5. What's the difference between Sentinel and Cluster?
6. Explain the hot key problem and 3 solutions.
7. How does cache-aside work? Why delete instead of update on write?
8. What is the thundering herd problem and how do you prevent it?
9. Compare LRU and LFU eviction — when to use each?
10. Design a rate limiter using Redis (explain the sliding window approach).

**If you can answer all 10 from memory with tradeoffs, you're ready.**

---

### Tradeoff Navigator

```
┌─────────────────────────────────────────────────────────────────┐
│                   REDIS PICK YOUR TRADEOFF                       │
│                                                                  │
│  Max speed           → No persistence, allkeys-lru, pipeline    │
│  Max durability      → AOF always + replicas (but slower)       │
│  Scale reads         → Add read replicas (stale reads possible) │
│  Scale writes        → Redis Cluster (cross-slot limitation)    │
│  Strong consistency  → Don't use Redis as primary store         │
│  Simple HA           → Sentinel (data fits one machine)         │
│  Horizontal scale    → Cluster (6+ nodes, hash tag discipline)  │
│  Memory efficiency   → Use hashes for small objects, HLL/bitmaps│
│  Zero data loss      → Probably not Redis. Use PostgreSQL.      │
│  Sub-ms latency      → Redis + client-side cache + pipeline     │
└─────────────────────────────────────────────────────────────────┘
```

---

*Last updated: March 2026. Covers Redis 7.x+ features including I/O threads, client-side caching, Streams consumer groups, and shard Pub/Sub.*
