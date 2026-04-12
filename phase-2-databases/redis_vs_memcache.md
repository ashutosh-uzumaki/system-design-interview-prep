# Chunk 24D — Redis vs Memcached
### Phase 2 — Databases | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Swiggy, Hotstar

---

## What is Memcached?

```
Memcached = pure cache, nothing else

→ In-memory key-value store
→ Strings ONLY (no data structures)
→ No persistence (data lost on restart)
→ No replication
→ No pub/sub
→ No distributed locks
→ No Lua scripting
→ Does ONE thing: cache strings fast
```

---

## Full Feature Comparison

| Feature | Redis | Memcached |
|---|---|---|
| **Data structures** | String, Hash, List, Set, ZSet ✅ | String only ❌ |
| **Persistence** | RDB + AOF ✅ | None ❌ |
| **Replication** | Master + Replica ✅ | None ❌ |
| **Clustering** | Redis Cluster ✅ | Basic ⚠️ |
| **TTL** | Per key ✅ | Per key ✅ |
| **Pub/Sub** | Yes ✅ | No ❌ |
| **Distributed locks** | SETNX ✅ | No ❌ |
| **Lua scripting** | Yes ✅ | No ❌ |
| **Threading** | Single + I/O threads | Multi-threaded ✅ |
| **Memory efficiency** | Good | Slightly better ✅ |
| **Max value size** | 512MB ✅ | 1MB ❌ |
| **HA (Sentinel)** | Yes ✅ | No ❌ |

---

## Where Memcached Wins (Rare Cases)

```
1. Raw cache throughput:
→ Multi-threaded → uses all CPU cores
→ Slightly faster for pure string caching
→ Simple use case: cache HTML fragments

2. Memory efficiency:
→ Simpler internals = less overhead per key
→ Slightly more keys per GB of RAM

3. Horizontal scaling (simple):
→ Add nodes, client distributes keys
→ No cluster coordination overhead
```

---

## Where Redis Wins (Almost Everything)

```
✅ Data structures:
→ Leaderboard? → ZSet
→ Queue? → List
→ Shopping cart? → Hash
→ Unique visitors? → Set
→ Memcached: strings only 😬

✅ Persistence:
→ Redis survives restart (RDB + AOF)
→ Memcached loses everything on restart 😬

✅ High availability:
→ Redis Sentinel → automatic failover
→ Memcached: no built-in HA 😬

✅ Distributed locks:
→ Redis SETNX → distributed coordination
→ Memcached: impossible 😬

✅ Pub/Sub:
→ Redis pub/sub → real-time messaging
→ Memcached: no pub/sub 😬

✅ Rate limiting:
→ Redis INCR atomic counter
→ Memcached: no atomic INCR 😬
```

---

## When to Pick Memcached

```
Very specific conditions:

1. Pure caching only:
→ No other Redis features needed
→ Just caching HTML/API responses
→ No queues, locks, leaderboards

2. Multi-threaded performance critical:
→ Need maximum throughput on multi-core
→ Squeezing every last bit of performance

3. Legacy system:
→ Already running Memcached
→ Migration cost > benefit
→ Keep it running

Reality: These cases are RARE in new systems
→ Redis handles all of these adequately
→ Memcached chosen mainly in legacy systems
```

---

## When to Pick Redis (Almost Always)

```
→ Need any data structure beyond strings ✅
→ Need persistence (survive restart) ✅
→ Need high availability (Sentinel) ✅
→ Need horizontal scaling (Cluster) ✅
→ Need distributed locks ✅
→ Need pub/sub messaging ✅
→ Building new system from scratch ✅

= almost every modern use case ✅
```

---

## Real World Usage

```
Companies using Redis:
→ Razorpay, PhonePe, Flipkart ✅
→ Hotstar, Swiggy, Zepto ✅
→ Twitter, GitHub, Stack Overflow ✅

Companies using Memcached:
→ Facebook (legacy, moving to Redis)
→ Wikipedia (legacy)
→ Most new systems = Redis ✅
```

---

## Real Use Case — LeetCode Code Judge

```
What needs caching:
1. Submission queue
2. Execution results
3. User session
4. Problem data
5. Leaderboard
6. Rate limiting

Redis handles ALL of these:

Submission queue → Redis List ✅
LPUSH new submission → worker RPOP to process

Execution results → Redis String + TTL ✅
"result:submission:123" → {status, output}
TTL: 24 hours

User session → Redis Hash ✅
HSET session:user:123 name "Ashutosh"

Leaderboard → Redis ZSet ✅
ZADD leaderboard score user:123
ZREVRANGE → top 100 instantly

Rate limiting → Redis INCR + TTL ✅
Max 10 submissions/minute per user

Memcached score: 0/6 use cases ❌
Redis score: 6/6 use cases ✅
```

### Full LeetCode architecture:
```
User submits code:
         ↓
Rate limit check (Redis INCR) ✅
         ↓
Push to queue (Redis List) ✅
         ↓
Worker pops from queue
         ↓
Executes in isolated Docker container
         ↓
Stores result (Redis String + TTL) ✅
         ↓
User polls → Redis ✅
         ↓
Update leaderboard (Redis ZSet) ✅
         ↓
Permanent storage → PostgreSQL ✅
```

---

## Real Use Cases at Indian Companies

```
Razorpay:
→ Rate limiting (Redis INCR) ✅
→ Payment status cache (Redis String) ✅
→ Distributed locks (Redis SETNX) ✅
→ Memcached: can't do locks or INCR ❌

PhonePe:
→ OTP storage (Redis String + TTL) ✅
→ Session management (Redis Hash) ✅
→ Transaction deduplication ✅

Hotstar:
→ Match score cache (Redis String) ✅
→ User session (Redis Hash) ✅
→ Live leaderboard (Redis ZSet) ✅

Swiggy:
→ Restaurant cache (Redis Hash) ✅
→ Delivery tracking queue (Redis List) ✅
→ Surge pricing (Redis String) ✅

Zepto:
→ Cart (Redis Hash) ✅
→ Inventory cache (Redis String) ✅
→ Rate limiting (Redis INCR) ✅
```

---

## The Interview One-Liner

> *"Memcached is a simpler, slightly faster pure cache. Redis is a full data structure server that also happens to be a great cache. For any new system I'd choose Redis — it does everything Memcached does plus distributed locks, pub/sub, persistence, HA, and rich data structures, with minimal performance overhead."*

---

## Interview Framing

**On choosing between them:**
> *"I'd always choose Redis for new systems. Memcached only wins in very specific legacy scenarios where pure string caching throughput is the only requirement. The moment you need a queue, leaderboard, distributed lock, or persistent cache, Redis is the clear choice."*

**On LeetCode/coding judge:**
> *"Redis handles the entire caching layer — submission queue via List, rate limiting via INCR, leaderboard via ZSet, sessions via Hash, results via String with TTL. Memcached couldn't support this since it only handles strings. The actual execution runs in isolated Docker containers, permanent results in PostgreSQL."*

---

## Quick Drill Questions

```
Q1: Name 3 things Redis can do 
    that Memcached cannot.

Q2: When would you actually choose 
    Memcached over Redis?

Q3: Design the caching layer for 
    a coding judge (LeetCode style).
    Which Redis data structures for each need?

Q4: Razorpay needs: rate limiting, 
    payment status cache, distributed locks.
    Redis or Memcached? Why?

Q5: What happens to Memcached data 
    when the server restarts?
    How is Redis different?
```

---

## Drill Answers

### Q1: Redis over Memcached
```
1. Data structures:
   → Hash, List, Set, ZSet
   → Memcached: strings only ❌

2. Persistence:
   → RDB + AOF → survives restart ✅
   → Memcached: data lost on restart ❌

3. Distributed locks:
   → SETNX for distributed coordination ✅
   → Memcached: impossible ❌

Bonus:
4. Pub/Sub messaging
5. High availability (Sentinel)
6. Horizontal scaling (Cluster)
```

### Q2: When to choose Memcached
```
Only in these specific cases:

1. Pure string caching only:
   → No queues, locks, leaderboards needed
   → Just caching HTML/API responses

2. Multi-threaded performance critical:
   → Need absolute max throughput on multi-core
   → Difference is marginal in practice

3. Already in legacy system:
   → Migration cost > benefit

Reality: Almost never for new systems ✅
```

### Q3: LeetCode caching layer
```
Submission queue    → Redis List (LPUSH/RPOP) ✅
Rate limiting       → Redis String + INCR + TTL ✅
Leaderboard         → Redis ZSet (ZADD/ZREVRANGE) ✅
User session        → Redis Hash (HSET/HGETALL) ✅
Execution results   → Redis String + TTL ✅
Problem data cache  → Redis String + TTL ✅

Memcached: 0/6 ❌
Redis: 6/6 ✅
```

### Q4: Razorpay needs
```
Redis ✅

Rate limiting → Redis INCR + TTL ✅
→ Memcached: no atomic INCR ❌

Payment status → Redis String ✅
→ Memcached: can do this ⚠️ but...

Distributed locks → Redis SETNX ✅
→ Memcached: impossible ❌

Redis wins 3/3 use cases
Memcached wins 0/3 ❌
```

### Q5: Restart behavior
```
Memcached:
→ All data lost on restart 😬
→ No persistence
→ Cache warms up from scratch
→ DB hit for every key until warmed

Redis:
→ RDB: loads last snapshot on restart ✅
→ AOF: replays all operations on restart ✅
→ Data survives restart
→ Cache immediately available ✅

Impact:
→ Memcached restart = performance degradation
  until cache warms up 😬
→ Redis restart = minimal impact ✅
```

---

## One-Line Summaries

```
Memcached    → pure cache, strings only, fast, no extras
Redis        → data structure server + cache + much more
Persistence  → Redis survives restart, Memcached doesn't
Data structs → Redis has 5, Memcached has 1
Locks        → Redis SETNX, Memcached impossible
Threading    → Memcached multi-thread, Redis single+IO
Choose Redis → almost always for new systems ✅
Choose Memcached → legacy systems, pure cache only
```

---

*Chunk 24D complete*
*Next: Chunk 24E — Request Lifecycle*
*Phase 2 progress: 14/16 chunks done*