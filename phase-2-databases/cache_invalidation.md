# Chunk 24C — Cache Invalidation
### Phase 2 — Databases | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Swiggy, Zepto

---

## The Problem

```
Flipkart. iPhone 15 price = ₹79,999.

Cached in Redis:
"product:iphone15:price" → 79,999
TTL: 1 hour

Flash sale starts. Price drops to ₹59,999 in DB.

Result:
→ DB: ₹59,999 ✅
→ Cache: ₹79,999 😬
→ Users see wrong price for up to 1 hour 😬
→ Overcharged users 😬
→ Trust broken 😬

This = Cache Invalidation Problem
```

---

## Cache Invalidation vs Cache Eviction

```
Cache Invalidation:
→ YOU decide when to remove/update a key
→ Triggered by: data change in DB
→ "Price changed → remove old price"
→ About CORRECTNESS ✅

Cache Eviction:
→ CACHE decides what to remove
→ Triggered by: cache running out of memory
→ "Cache full → which key to delete?"
→ About MEMORY MANAGEMENT ✅

LRU, LFU, TTL = eviction policies (Phase 6)
Cache invalidation = this chunk ✅
```

---

## Why It's Hard

```
Phil Karlton:
"There are only two hard things in CS:
 cache invalidation and naming things."

Challenges:
→ DB updated → cache must know immediately
→ At scale: millions of cache keys
→ Can't check every key every second
→ Deleting key → stampede risk
→ Multiple cache layers (Redis + CDN + browser)
→ Distributed systems → eventual consistency
```

---

# The 4 Cache Invalidation Strategies

---

## Strategy 1 — TTL Expiry (Simplest)

```
Set TTL on every cache key:
"product:iphone15:price" → TTL 5 mins

DB updates price:
→ Cache shows old price for up to 5 mins
→ After TTL expires → next read fetches from DB
→ Cache repopulated with new price ✅
```

```
Gain:
✅ Dead simple to implement
✅ No extra infrastructure
✅ Works everywhere
✅ Automatic cleanup

Lose:
❌ Stale data window = TTL duration
❌ Flash sale price wrong for 5 mins 😬
❌ Can't control WHEN cache updates
❌ Short TTL = more DB hits
   Long TTL = more stale data
```

**TTL tuning:**
```
High consistency needed → short TTL (30 secs)
Tolerance for staleness → long TTL (1 hour)
Never stale → don't use TTL alone

Best for:
→ Restaurant listings (Swiggy)
→ Product descriptions
→ News articles
→ Any data tolerating staleness ✅
```

---

## Strategy 2 — Delete on Write (Cache-Aside)

```
When DB updates:
→ Application deletes cache key immediately
→ Next read = cache miss → fetch from DB
→ Cache repopulated with fresh data ✅

Flow:
Write to DB → delete cache key
Next read   → cache miss → DB read → cache set ✅
```

```
Gain:
✅ Near-zero stale data window
✅ Simple to implement
✅ Works for low-traffic keys

Lose:
❌ Cache stampede risk on popular keys
   → Millions hit DB simultaneously after delete 😬
❌ Every write = extra delete operation
❌ Race condition:
   Read 1:  cache miss → fetching from DB
   Write:   DB updated, cache key deleted
   Read 1:  populates cache with OLD value 😬
```

**Race condition fix:**
```
Use versioning + timestamp:
→ Read fetches DB + sets cache with timestamp
→ Write invalidates + sets newer timestamp
→ Cache only accepts newer timestamp ✅

Or use distributed lock:
→ Lock key during write
→ No concurrent stale writes ✅
```

**Best for:**
```
→ Low-traffic keys
→ Simple systems without Kafka
→ Data that must be fresh immediately ✅
```

---

## Strategy 3 — Event-Driven Invalidation (Kafka)

```
When DB updates:
→ Publish event to Kafka topic
→ Cache invalidation consumer reads event
→ Deletes/updates specific cache key ✅

Flow:
DB update → Kafka event → consumer → cache delete/update
```

```
Gain:
✅ Decoupled — DB doesn't know about cache
✅ Multiple caches invalidated from one event
✅ Reliable — Kafka retries on failure
✅ Audit trail of all cache updates
✅ Multiple consumers from same event

Lose:
❌ Extra infrastructure (Kafka needed)
❌ Small delay (event processing time)
❌ More complex setup
❌ Eventual consistency window
```

**Real example — Flipkart price update:**
```
Product service:
→ Updates DB: price = ₹59,999
→ Publishes to Kafka:
  {product_id: "iphone15", new_price: 59999}
         ↓
Cache consumer:
→ Reads event
→ Deletes Redis key ✅
→ OR updates Redis key directly ✅
         ↓
CDN consumer:
→ Invalidates CDN cache ✅
         ↓
Search consumer:
→ Updates Elasticsearch price ✅

One event → ALL caches invalidated ✅
```

**Best for:**
```
→ Complex systems with multiple cache layers
→ When multiple consumers need same event
→ High reliability needed
→ Flipkart, Swiggy, Razorpay scale ✅
```

---

## Strategy 4 — CDC (Change Data Capture)

```
Most sophisticated approach.

CDC = watch DB transaction log (WAL)
    → detect every change automatically
    → propagate to cache without app code

Tools: Debezium + Kafka

How it works:
PostgreSQL writes to WAL
         ↓
Debezium reads WAL continuously
         ↓
Publishes to Kafka:
{
  table: "products",
  operation: "UPDATE",
  old: {price: 79999},
  new: {price: 59999}
}
         ↓
Cache consumer  → updates Redis ✅
Search consumer → updates Elasticsearch ✅
Analytics       → updates BigQuery ✅
```

```
Gain:
✅ Application doesn't handle invalidation
✅ Every DB change captured automatically
✅ Works across all consumers
✅ No code changes needed in app
✅ Most reliable approach
✅ Complete audit trail

Lose:
❌ Complex infrastructure
❌ Debezium + Kafka setup needed
❌ Still eventual consistency
❌ High operational overhead
❌ Overkill for small systems
```

**Best for:**
```
→ Large scale systems (Zepto, Flipkart)
→ Multiple downstream consumers
→ When app shouldn't own invalidation logic
→ Strong audit requirements ✅
```

---

## Strategy Comparison

| | TTL | Delete on Write | Kafka Events | CDC |
|---|---|---|---|---|
| **Complexity** | Low ✅ | Low ✅ | Medium | High ❌ |
| **Staleness** | TTL window | Near zero ✅ | Seconds | Seconds |
| **Consistency** | Eventual | Strong ✅ | Eventual | Eventual |
| **Stampede risk** | Yes ❌ | Yes ❌ | Low ✅ | Low ✅ |
| **Infrastructure** | None ✅ | None ✅ | Kafka | Kafka+Debezium |
| **App code needed** | No ✅ | Yes | Yes | No ✅ |
| **Best for** | Tolerant data | Simple systems | Complex | Large scale |

---

## Combining Strategies — Production Reality

```
Real systems use MULTIPLE strategies:

Flipkart product price:
→ TTL: 5 minutes (safety net) ✅
→ Kafka event on price change (immediate) ✅

Together:
→ Price change → Kafka invalidates immediately ✅
→ TTL = fallback if Kafka fails ✅
→ Best of both worlds ✅

Rule: TTL = safety net, Event = primary trigger
```

---

## Special Invalidation Patterns

### Pattern 1 — Write-Through (Always Consistent):
```
Every write → update DB + cache simultaneously

Flow:
App writes price = ₹59,999
→ Write to DB ✅
→ Write to Redis ✅
→ Both always in sync ✅

Gain: Cache never stale ✅
Lose: Every write costs extra (cache update) ❌

Best for:
→ User profiles (must be fresh)
→ Merchant settings (Razorpay)
→ Critical config data
```

### Pattern 2 — Versioned Cache Keys:
```
Instead of invalidating, change the key version:

Before update:
"product:iphone15:v1" → ₹79,999

After update:
"product:iphone15:v2" → ₹59,999 ✅

App always reads latest version
"product:iphone15:v1" expires via TTL ✅

Gain:
→ No stampede (old key still serves during transition)
→ Gradual rollout ✅
→ Easy rollback ✅

Lose:
→ Need version tracking
→ Multiple keys in cache temporarily
```

### Pattern 3 — Soft Invalidation:
```
Don't delete cache key.
Mark as stale instead:

"product:iphone15:stale" → true

Next read:
→ Serve stale data immediately (no latency) ✅
→ Trigger async background refresh
→ Next request gets fresh data ✅

Gain:
→ Zero latency even on cache miss ✅
→ No stampede ✅
→ User never waits ✅

Lose:
→ Briefly serves stale data
→ More complex implementation
```

---

## Cache Stampede Prevention

```
Problem:
Popular key expires/deleted
→ 10,000 concurrent requests
→ All miss cache
→ All hit DB simultaneously 😬
→ DB overwhelmed 😬

Solutions:

1. Mutex Lock:
→ First request gets lock
→ Fetches from DB + populates cache
→ Other requests wait for lock
→ Lock released → others read from cache ✅

2. Probabilistic Early Expiry:
→ Before TTL expires, probabilistically refresh
→ Some requests refresh early
→ Cache never actually expires for all 😬

3. Soft Invalidation:
→ Serve stale data while refreshing async
→ Nobody hits DB directly ✅

4. Background refresh:
→ Separate job refreshes cache before TTL
→ Cache never actually expires ✅
→ Used for critical hot keys ✅
```

---

## Real Production Examples

```
Swiggy restaurant listings:
→ TTL: 5 minutes ✅
→ Menu update → Kafka event → immediate invalidation ✅

Razorpay merchant settings:
→ Write-through (always consistent) ✅
→ Settings critical → can't be stale

Flipkart product price:
→ TTL: 5 mins + Kafka on price change ✅
→ Flash sale → immediate invalidation ✅

Hotstar match score:
→ TTL: 10 seconds ✅
→ Score update → Redis updated directly ✅
→ Short TTL = always fresh enough

Zepto inventory count:
→ CDC with Debezium ✅
→ Every DB change → cache updated
→ Can't show wrong stock count ✅

PhonePe exchange rates:
→ TTL: 30 seconds ✅
→ Rate change → Kafka event ✅
→ Critical for payment calculations
```

---

## Interview Framing

**On basic invalidation:**
> *"For Flipkart product prices I'd combine TTL with event-driven invalidation. TTL acts as safety net, while Kafka events trigger immediate invalidation on price changes. Flash sale prices propagate instantly without waiting for TTL expiry."*

**On CDC:**
> *"For Zepto's inventory system I'd use CDC with Debezium reading PostgreSQL's WAL. Every inventory change automatically propagates to Redis without application code changes. The application just writes to DB and cache stays in sync automatically."*

**On stampede prevention:**
> *"When invalidating a hot key I'd use soft invalidation — mark as stale and serve existing data while refreshing async. This prevents cache stampede where millions of requests hit DB simultaneously after a cache miss."*

**On combining strategies:**
> *"In production I'd never rely on a single strategy. TTL as safety net, Kafka events for immediate propagation, and write-through for critical data that can never be stale. Each layer covers the failure mode of the previous."*

---

## Quick Drill Questions

```
Q1: What is the difference between 
    cache invalidation and cache eviction?

Q2: Flipkart flash sale starts. Price drops.
    You're using only TTL (1 hour).
    What happens? How do you fix it?

Q3: What is cache stampede?
    How do you prevent it?

Q4: What is CDC? When would you use it
    over Kafka events?

Q5: Razorpay merchant settings must never
    be stale. Which invalidation strategy?

Q6: What are the 4 cache invalidation strategies?
    One sentence each.

Q7: Why combine TTL + Kafka events together?

Q8: What is soft invalidation and why is it useful?
```

---

## Drill Answers

### Q1: Invalidation vs Eviction
```
Cache Invalidation:
→ YOU trigger removal
→ Because data changed in DB
→ About correctness
→ Examples: price change, name update

Cache Eviction:
→ CACHE triggers removal
→ Because memory is full
→ About memory management
→ Examples: LRU, LFU, TTL-based removal

Both remove keys — different reasons ✅
```

### Q2: Flash sale + TTL only
```
Problem:
→ Price drops in DB immediately
→ Cache shows old price for up to 1 hour 😬
→ Users overcharged 😬

Fix:
→ Add event-driven invalidation
→ Price change → publish Kafka event
→ Cache consumer → deletes/updates Redis key
→ Price propagates immediately ✅
→ Keep TTL as safety net ✅
```

### Q3: Cache stampede
```
What it is:
→ Popular key expires/deleted
→ Thousands of concurrent requests miss cache
→ All hit DB simultaneously 😬
→ DB overwhelmed 😬

Prevention:
1. Mutex lock → only first request fetches from DB
2. Soft invalidation → serve stale, refresh async
3. Background refresh → refresh before TTL expires
4. Probabilistic early expiry → some requests refresh early
```

### Q4: CDC vs Kafka events
```
Kafka events:
→ Application publishes event after DB write
→ App must know to publish
→ Code change needed per service
→ Medium complexity

CDC (Debezium):
→ Reads DB transaction log (WAL) automatically
→ Application doesn't know about cache
→ No code changes needed
→ Every DB change captured automatically
→ More complex infrastructure

Use CDC when:
→ Multiple downstream consumers ✅
→ App shouldn't own invalidation logic ✅
→ Complete audit trail needed ✅
→ Large scale (Zepto, Flipkart) ✅

Use Kafka events when:
→ Specific events matter only ✅
→ Simpler infrastructure ✅
→ Medium scale ✅
```

### Q5: Merchant settings never stale
```
Write-Through ✅

Every settings update:
→ Write to DB ✅
→ Write to Redis ✅
→ Both always in sync ✅
→ Cache never stale ✅

Why not TTL: settings could be stale during TTL window
Why not delete-on-write: race condition risk
Why not Kafka: small delay → briefly stale
Write-Through: zero staleness ✅
```

### Q6: 4 strategies
```
1. TTL Expiry:
   → Key auto-expires after time, repopulated on next read

2. Delete on Write:
   → App deletes key after DB update, repopulated on miss

3. Event-Driven (Kafka):
   → DB update → Kafka event → consumer invalidates cache

4. CDC:
   → DB WAL → Debezium → Kafka → consumers update cache
```

### Q7: TTL + Kafka combined
```
TTL alone:
→ Stale for TTL duration on price change 😬

Kafka alone:
→ If Kafka fails → cache never invalidated 😬

Together:
→ Kafka: immediate invalidation on change ✅
→ TTL: safety net if Kafka fails ✅
→ Worst case: stale for TTL duration ✅
→ Best case: immediate update ✅

Rule: Event = primary trigger, TTL = fallback ✅
```

### Q8: Soft invalidation
```
Instead of deleting key:
→ Mark key as stale ("product:iphone15:stale" = true)
→ Serve stale data immediately (zero latency) ✅
→ Trigger async background refresh
→ Next request gets fresh data ✅

Why useful:
→ No cache stampede (stale data served during refresh) ✅
→ Zero latency for user (always served from cache) ✅
→ DB never overwhelmed ✅
→ Perfect for high-traffic hot keys ✅
```

---

## One-Line Summaries

```
Invalidation    → remove because data is wrong
Eviction        → remove because cache is full
TTL             → auto-expire, simple, stale window risk
Delete on write → immediate, stampede risk
Kafka events    → decoupled, reliable, slight delay
CDC             → automatic, no app code, complex infra
Write-through   → always consistent, extra write cost
Soft invalidation → serve stale, refresh async, no stampede
Cache stampede  → mass DB hit after popular key expires
```

---

*Chunk 24C complete*
*Next: Chunk 24D — Redis vs Memcached*
*Phase 2 progress: 13/16 chunks done*