# Chunk 21 — Drill Questions & Answers
### Redis Internals | SDE-2 Interview Prep

---

## Q1: Name 3 reasons Redis is faster than PostgreSQL.

**Answer:**
```
1. In-memory
   → No disk I/O
   → RAM access = nanoseconds vs disk = milliseconds
   → PostgreSQL hits disk even with buffer pool cache

2. Simple data model
   → No SQL parsing, no query planning, no index traversal
   → Just direct memory lookup
   → GET key → value. Done.

3. Single-threaded + Event loop
   → Zero lock contention
   → Zero context switching
   → Never blocks on I/O
   → Processes 100K ops/sec at sub-millisecond latency
```

**Interview framing:**
> *"Redis combines three things PostgreSQL can't match simultaneously — in-memory storage, a simple key-value model with no query overhead, and a single-threaded event loop with zero lock contention."*

---

## Q2: Which data structure for shopping cart? Why not JSON String?

**Answer:**
```
Use: Hash

Key:   cart:user123
Field: item:456  → Value: 2  (quantity)
Field: item:789  → Value: 1

Why not JSON String:
→ To update one item's quantity:
  JSON: GET → deserialize → update → serialize → SET
  Hash: HSET cart:user123 item:456 3 → done in one op ✅

→ JSON = read entire object for every single update 😬
→ Hash = update only the changed field ✅

Commands:
HSET cart:user123 "item:456" 2      ← add item
HSET cart:user123 "item:456" 3      ← update quantity
HGETALL cart:user123                ← get entire cart
HDEL cart:user123 "item:456"        ← remove item
```

---

## Q3: PhonePe OTP — which structure + which feature?

**Answer:**
```
Structure: String
Feature:   TTL (Time To Live)

Command:
SETEX otp:phone:9876543210 120 "482910"
→ Stores OTP "482910" for 2 minutes (120 seconds)
→ Redis auto-deletes after expiry ✅
→ No cleanup code needed in application ✅

Persistence: RDB
→ OTP loss on Redis restart = acceptable
→ User simply requests a new OTP
→ No need for AOF overhead here
```

**Interview framing:**
> *"I'd use Redis String with SETEX for OTP — 120 second TTL auto-expires the key so no cleanup logic needed. RDB persistence is sufficient since a lost OTP just means the user requests a new one."*

---

## Q4: What's the difference between RDB and AOF?

**Answer:**
```
RDB (Redis Database Snapshot):
→ Periodic full snapshot saved to disk
→ Configurable interval (every 5 mins, every 1 hr)
→ Fast restart (load one file)
→ Small file size
→ Data loss risk = everything since last snapshot

AOF (Append Only File):
→ Every write operation logged to disk immediately
→ On restart: replays all operations to reconstruct state
→ Slow restart (replay entire log)
→ Large file size
→ Data loss risk = ~1 second maximum
```

| | RDB | AOF |
|---|---|---|
| Data loss | Minutes | ~1 second |
| File size | Small | Large |
| Restart speed | Fast | Slow |
| Write performance | Better | Slightly slower |

```
Production recommendation:
→ Use BOTH together
→ AOF for durability
→ RDB for fast backups
→ On restart → Redis uses AOF (more recent) ✅
```

**When to use which:**
```
RDB → Hotstar watch history cache
      (losing 5 mins of watch position = acceptable)

AOF → Razorpay rate limiting counters
      (losing counter state = wrong rate limits = correctness issue)
```

---

## Q5: Swiggy restaurant cache — why split static vs dynamic data?

**Answer:**
```
Problem:
→ Restaurant data has two types of fields:

Static (rarely changes):
→ Name, location, cuisine type
→ Can cache for 24 hours

Dynamic (changes frequently):
→ Offers, delivery time, open/closed status, rating
→ Must refresh every 5 minutes

Why split:
→ If one key holds everything with 5min TTL:
  → Static data refreshed unnecessarily every 5 mins
  → Extra DB load for data that hasn't changed 😬

→ If one key holds everything with 24hr TTL:
  → Offers go stale, restaurant shows as open when closed 😬

Solution — Cache Segmentation by Volatility:
→ "restaurant:456:static"  → TTL 24 hours
→ "restaurant:456:dynamic" → TTL 5 minutes

Bonus — why store IDs not full details in location cache:
→ "restaurants:lat:12.97:lng:77.59" → [456, 789, 123...]
→ One restaurant updates offer?
→ Only invalidate "restaurant:456:dynamic" ✅
→ Location cache still valid ✅
→ Not the entire location listing
```

**Interview framing:**
> *"I'd segment the cache by data volatility. Static restaurant data like name and location gets 24hr TTL. Dynamic data like offers and delivery time gets 5min TTL. The location cache stores only IDs — so a single restaurant update only invalidates that restaurant's cache, not the entire location listing."*

---

## Q6: When would you NOT use Redis?

**Answer:**
```
❌ Data too large for memory
   → Redis is memory-only
   → 1TB dataset = extremely expensive RAM
   → Use PostgreSQL or Cassandra

❌ Complex queries needed
   → No joins, no aggregations, no GROUP BY
   → "Find all users who ordered X and Y" → PostgreSQL

❌ Strong durability required
   → Even AOF loses ~1 second of data
   → Financial ledger, audit logs → PostgreSQL
   → Cannot afford any data loss

❌ Relational data
   → No foreign keys, no relationships
   → Use PostgreSQL

❌ Large values
   → Redis optimized for small values
   → Storing large files/blobs → S3/object storage
```

---

## Q7: Zerodha wants top 10 trending stocks updated every minute. Which Redis data structure?

**Answer:**
```
Use: ZSet (Sorted Set)

Why ZSet:
→ Stores items with a score
→ Auto-sorted by score
→ Can retrieve top-K in O(log N)
→ Perfect for rankings

Commands:
ZADD trending:stocks 5000 "RELIANCE"
ZADD trending:stocks 3200 "INFY"
ZADD trending:stocks 4800 "TCS"

ZREVRANGE trending:stocks 0 9 WITHSCORES
→ returns top 10 in descending order ✅

TTL: 60 seconds
→ EXPIRE trending:stocks 60
→ Refreshed every minute by background job
```

**Why not other structures:**
```
String → can't store multiple items with scores
Hash   → no built-in sorting
List   → no scoring, manual sort needed
Set    → no scores, just unique items
ZSet   → scores + sorting built-in ✅
```

---

## Q8: What is TTL and name 3 real use cases?

**Answer:**
```
TTL (Time To Live):
→ A duration set on a Redis key
→ Redis automatically deletes the key after expiry
→ No application-level cleanup code needed
→ Check remaining TTL: TTL key → seconds remaining
→ Already expired: TTL key → -2

Setting TTL:
EXPIRE key 120              ← set TTL on existing key
SETEX key 120 "value"       ← set key + TTL together
```

**3 real use cases:**

```
1. OTP codes (PhonePe):
   SETEX otp:9876543210 120 "482910"
   → Auto-expires in 2 minutes
   → Expired OTP cannot be reused ✅

2. Session tokens (Hotstar):
   SETEX session:user123 3600 "token_xyz"
   → Auto-expires in 1 hour
   → User must re-login after session expires ✅

3. Rate limiting window (Razorpay):
   INCR api:merchant:M123:count
   EXPIRE api:merchant:M123:count 60
   → Counter resets every 60 seconds
   → Max N requests per minute enforced ✅
```

**Bonus use cases:**
```
Password reset links → TTL 600s  (10 minutes)
Cache entries        → TTL 300s  (5 minutes)
Distributed locks    → TTL 30s   (prevent deadlocks)
```

---

## Summary — One-Line Answers

```
Q1: Fast because → in-memory + simple model + single-threaded event loop
Q2: Hash → update one field without reading entire object
Q3: String + TTL → SETEX with 120s expiry, RDB persistence
Q4: RDB = periodic snapshot (fast restart, some loss)
    AOF = every op logged (minimal loss, slow restart)
Q5: Split by volatility → static 24hr TTL, dynamic 5min TTL
Q6: Not Redis when → data too large, complex queries, strong durability needed
Q7: ZSet → scores + auto-sorting + top-K retrieval built-in
Q8: TTL = auto-delete after time → OTP, sessions, rate limiting
```

---

*Chunk 21 Drill complete*
*Next: Chunk 21B — Redis Sentinel + Cluster*