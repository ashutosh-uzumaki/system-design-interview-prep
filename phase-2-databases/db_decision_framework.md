# Master DB Decision Framework
### Phase 2 — Final Chunk | SDE-2 Interview Prep
> "Which database would you use here and why?"
> Target: Razorpay, PhonePe, Flipkart, Zerodha, Zepto

---

## The 6-Question Decision Framework

```
Q1: Do I need ACID?
→ Yes → PostgreSQL ✅
→ No  → consider NoSQL

Q2: Is the schema flexible/unpredictable?
→ Yes → MongoDB ✅
→ No  → continue

Q3: Is it write-heavy at massive scale?
→ Yes → Cassandra ✅
→ No  → continue

Q4: Is it read-heavy with same data
    accessed repeatedly?
→ Yes → Redis (cache) ✅
→ No  → continue

Q5: Do I need ranking, leaderboard,
    or counting?
→ Yes → Redis ZSet/String ✅
→ No  → continue

Q6: Do I need full-text search?
→ Yes → Elasticsearch ✅
→ No  → PostgreSQL (default) ✅
```

---

## Complete Decision Tree

```
START
  ↓
Need ACID + strong consistency?
  Yes → PostgreSQL ✅
  No  ↓

Flexible/unpredictable schema?
  Yes → MongoDB ✅
  No  ↓

Write-heavy (millions/sec)?
  Yes → Cassandra ✅
  No  ↓

Same data read millions of times?
  Yes → Redis (cache layer) ✅
  No  ↓

Need ranking/leaderboard/counting?
  Yes → Redis ZSet/INCR ✅
  No  ↓

Need full-text search?
  Yes → Elasticsearch ✅
  No  → PostgreSQL (default) ✅
```

---

## Each DB — One Line + Use Case

```
PostgreSQL:
→ ACID + relational + complex queries
→ Payments, orders, financial data
→ Default choice when unsure ✅

MongoDB:
→ Flexible schema + nested documents
→ Product catalog, content, user profiles

Cassandra:
→ Massive write throughput + time-series
→ Activity logs, watch history, events

Redis:
→ Ultra-fast reads + data structures
→ Cache, sessions, leaderboards, locks, rate limiting

Elasticsearch:
→ Full-text search + analytics
→ Product search, log analysis

BigQuery/Redshift:
→ Analytical queries on massive data
→ Business intelligence, reporting, ML features
```

---

## The 8 Key Questions For Any System

```
1. What is the read:write ratio?
   → Read-heavy  → Redis cache ✅
   → Write-heavy → Cassandra ✅
   → Balanced    → PostgreSQL ✅

2. Do I need ACID?
   → Financial data → always PostgreSQL ✅
   → Activity logs  → can skip ACID ✅

3. How large is the data?
   → Fits in RAM    → Redis ✅
   → Fits in disk   → PostgreSQL/MongoDB ✅
   → Massive (PBs)  → Cassandra/BigQuery ✅

4. How flexible is the schema?
   → Rigid and stable   → PostgreSQL ✅
   → Flexible/changing  → MongoDB ✅

5. What's the consistency requirement?
   → Strong     → PostgreSQL ✅
   → Tunable    → Cassandra (ONE/QUORUM/ALL) ✅
   → Eventual   → Redis/MongoDB ✅

6. What's the access pattern?
   → By primary key         → Redis/Cassandra ✅
   → Complex queries/joins  → PostgreSQL ✅
   → Full-text search       → Elasticsearch ✅
   → Analytics/aggregations → BigQuery ✅

7. Do I need persistence?
   → Yes, durable → PostgreSQL/Cassandra/MongoDB ✅
   → Temporary    → Redis (TTL) ✅

8. What's the scale?
   → Single machine → PostgreSQL ✅
   → Horizontal     → Cassandra/MongoDB ✅
   → In-memory      → Redis ✅
```

---

## DB Properties — Quick Reference

| DB | ACID | Schema | Scale | Best For |
|---|---|---|---|---|
| **PostgreSQL** | Full ✅ | Fixed | Vertical + sharding | Finance, orders |
| **MongoDB** | Partial ⚠️ | Flexible ✅ | Horizontal | Catalog, content |
| **Cassandra** | No ❌ | Query-driven | Massive ✅ | Write-heavy, logs |
| **Redis** | No ❌ | Key-Value | In-memory | Cache, sessions |
| **Elasticsearch** | No ❌ | Inverted index | Horizontal | Search |
| **BigQuery** | No ❌ | Columnar | Massive ✅ | Analytics |

---

## Common Production Combinations

### Payment System (Razorpay):
```
PostgreSQL  → transactions, accounts (ACID) ✅
Redis       → idempotency keys, rate limiting,
              distributed locks, payment status cache ✅
Kafka       → async processing, write buffering ✅
Elasticsearch → transaction search, audit logs ✅
```

### E-commerce (Flipkart):
```
MongoDB      → product catalog (flexible schema) ✅
PostgreSQL   → orders, payments (ACID) ✅
Redis        → cart, sessions, search cache ✅
Elasticsearch → product search (full-text) ✅
Cassandra    → activity logs, browse history ✅
BigQuery     → analytics, reporting ✅
```

### Streaming (Hotstar):
```
Cassandra  → watch history (write-heavy) ✅
Redis      → live scores, sessions, cache ✅
PostgreSQL → subscriptions, payments ✅
CDN        → video streaming ✅
BigQuery   → viewing analytics ✅
```

### Food Delivery (Swiggy):
```
PostgreSQL → orders, payments (ACID) ✅
MongoDB    → restaurant profiles, menus ✅
Redis      → restaurant cache, sessions,
             delivery tracking ✅
Cassandra  → activity logs, order events ✅
Elasticsearch → restaurant search ✅
```

### Stock Trading (Zerodha):
```
PostgreSQL → trades, accounts (ACID) ✅
Redis      → live prices, top stocks (ZSet),
             session, rate limiting ✅
Cassandra  → trade history, market events ✅
Elasticsearch → trade search ✅
BigQuery   → portfolio analytics ✅
```

### Hyperlocal Delivery (Zepto):
```
PostgreSQL → inventory, orders (ACID) ✅
Redis      → inventory cache, cart, sessions ✅
Cassandra  → order history, delivery events ✅
Kafka      → inventory update pipeline ✅
```

---

## The Tradeoff Formula

```
"I'd use [DB] here because:
→ [specific requirement] needs [specific property]
→ We gain [benefit]
→ We lose [cost]
→ That's acceptable because [reason]"
```

### Examples:

```
Cassandra for watch history:
"I'd use Cassandra for Hotstar's watch history.
 500M concurrent writes needs LSM Tree throughput.
 We gain horizontal write scaling.
 We lose ACID and strong consistency.
 Acceptable because losing watch position
 is a minor UX issue, not a financial loss."

PostgreSQL for payments:
"I'd use PostgreSQL for Razorpay payments.
 Every payment needs Atomicity —
 debit and credit must happen together.
 We gain full ACID guarantees.
 We lose horizontal write scaling.
 Acceptable because correctness > scale here,
 and we handle scale via sharding + Kafka."

Redis for leaderboard:
"I'd use Redis ZSet for Zerodha's top stocks.
 ZSet gives us built-in ranking with O(log N) updates.
 We gain sub-millisecond reads for 1M users.
 We lose persistence (mitigated by AOF).
 Acceptable because stock rankings can be
 recomputed from source if Redis restarts."
```

---

## Interview Scenarios — Full Answers

### "Design Razorpay payment DB layer":
```
PostgreSQL → payment transactions (ACID) ✅
Redis      → idempotency keys + TTL ✅
Redis      → rate limiting (INCR) ✅
Redis      → distributed locks (SETNX) ✅
Kafka      → buffer write spikes ✅
```

### "Design Flipkart product search":
```
MongoDB        → product catalog (flexible schema) ✅
Elasticsearch  → search (inverted index) ✅
Redis          → search results cache ✅
PostgreSQL     → orders (ACID) ✅
```

### "Design Hotstar for IPL":
```
Cassandra  → watch history (write-heavy) ✅
Redis      → live scores (read-heavy, ZSet) ✅
PostgreSQL → subscriptions (ACID) ✅
CDN        → video streaming ✅
BigQuery   → post-match analytics ✅
```

### "Design Zepto inventory":
```
PostgreSQL → inventory count (ACID, SELECT FOR UPDATE) ✅
Redis      → inventory cache (fast reads) ✅
Cassandra  → order history (write-heavy) ✅
Kafka      → inventory update events ✅
```

### "Design a coding judge (LeetCode)":
```
Redis      → submission queue (List) ✅
Redis      → rate limiting (INCR + TTL) ✅
Redis      → leaderboard (ZSet) ✅
Redis      → execution results (String + TTL) ✅
PostgreSQL → permanent submission history ✅
Docker     → isolated code execution ✅
```

---

## When Multiple DBs Are Right Answer

```
Interviewer: "Which DB for Swiggy?"

Wrong answer:
→ "PostgreSQL" (too simple, ignores scale)
→ "Cassandra" (misses ACID for payments)

Right answer:
→ "It depends on the component:
   Orders + payments → PostgreSQL (ACID)
   Restaurant catalog → MongoDB (flexible schema)
   Restaurant details → Redis cache (read-heavy)
   Activity logs → Cassandra (write-heavy)
   Search → Elasticsearch"

This shows SDE-2 thinking:
→ Different requirements → different DBs ✅
→ No one-size-fits-all ✅
→ Justify each choice ✅
```

---

## Red Flags to Avoid in Interviews

```
❌ "I'll use PostgreSQL for everything"
   → Shows no knowledge of distributed systems

❌ "I'll use MongoDB for everything"
   → Shows no knowledge of ACID requirements

❌ "I'll use Cassandra for payments"
   → Shows no understanding of ACID

❌ Choosing DB without explaining why
   → Always justify with specific requirement

❌ Not mentioning tradeoffs
   → Every choice has a cost → always mention it

❌ Not considering scale
   → "This will hit 10M users — PostgreSQL alone
      won't scale, we need sharding + Kafka"
```

---

## Quick Drill Questions

```
Q1: Which DB for each:
    a) Swiggy order history (1B orders)
    b) PhonePe OTP storage
    c) Myntra fashion catalog
    d) Zerodha live stock prices

Q2: Design the complete DB layer
    for a ride-sharing app like Ola.
    Name the DB for each component.

Q3: Interviewer asks "Use one DB for everything".
    How do you push back?

Q4: When would you use BOTH
    PostgreSQL AND Redis together?
    Give 2 real examples.

Q5: What's the difference between
    using Redis as primary DB vs as cache?
```

---

## Drill Answers

### Q1: DB for each
```
a) Swiggy order history (1B orders):
→ Cassandra ✅
→ Write-heavy, eventual consistency ok
→ Partition by user_id + time bucket

b) PhonePe OTP storage:
→ Redis ✅
→ String + TTL (120 seconds)
→ Auto-expires, no persistence needed

c) Myntra fashion catalog:
→ MongoDB ✅
→ Flexible schema (size, color, material vary)
→ Different attributes per category

d) Zerodha live stock prices:
→ Redis ZSet ✅
→ Top stocks by volume/price change
→ ZADD + ZREVRANGE = instant rankings
→ Updated every minute by background job
```

### Q2: Ola ride-sharing DB layer
```
Driver/rider profiles  → PostgreSQL ✅ (ACID, relational)
Active ride state      → Redis ✅ (fast, TTL when complete)
Trip history           → Cassandra ✅ (write-heavy, time-series)
Driver location        → Redis Geo ✅ (geospatial, real-time)
Payment transactions   → PostgreSQL ✅ (ACID non-negotiable)
Search/matching        → Redis + PostGIS ✅ (proximity search)
Analytics              → BigQuery ✅ (trip stats, ML features)
Notifications          → Kafka → FCM ✅ (async)
```

### Q3: Push back on one DB
```
"I understand the constraint, but using
one DB creates significant tradeoffs:

If PostgreSQL only:
→ Watch history at 500M writes/sec
→ Single primary overwhelmed 😬
→ B-Tree can't sustain this throughput

If Cassandra only:
→ Payment transactions lose ACID
→ Double charges possible 😬

Best approach: right tool for right job.
PostgreSQL for ACID-critical data.
Cassandra for write-heavy time-series.
Redis for caching + sessions.

If truly forced to one DB:
→ PostgreSQL with aggressive caching
→ Add Redis as mandatory cache layer
→ Accept limitations on write throughput"
```

### Q4: PostgreSQL + Redis together
```
Example 1 — Razorpay payments:
→ PostgreSQL: store transaction (ACID) ✅
→ Redis: cache payment status for dashboard ✅
→ Redis: rate limiting (INCR) ✅
→ Redis: distributed lock (SETNX) ✅
→ DB = source of truth, Redis = speed layer

Example 2 — Flipkart product page:
→ PostgreSQL: product data, orders (ACID) ✅
→ Redis: cache product details (TTL 5min) ✅
→ 10M reads → Redis (1ms)
→ Write/update → PostgreSQL (source of truth)
→ Redis invalidated on price change ✅
```

### Q5: Redis as primary vs cache
```
Redis as CACHE (most common):
→ Sits in front of PostgreSQL/MongoDB
→ Stores copy of DB data
→ TTL set (data can be stale)
→ If Redis down → read from DB ✅
→ DB = source of truth

Redis as PRIMARY DB (limited cases):
→ No backing store
→ Data lives ONLY in Redis
→ AOF persistence for durability
→ If Redis down → data at risk 😬
→ Use when: sessions, OTPs, leaderboards
→ Data can be reconstructed if lost ✅

Key difference:
→ Cache: DB is source of truth ✅
→ Primary: Redis is source of truth ✅
→ Most production systems = Redis as cache ✅
```

---

## One-Line Summaries

```
PostgreSQL    → ACID + relational, default for financial data
MongoDB       → flexible schema, product catalogs
Cassandra     → massive writes, time-series, eventual consistency
Redis         → ultra-fast, cache + sessions + leaderboards + locks
Elasticsearch → full-text search, log analytics
BigQuery      → columnar analytics, reporting, ML features
Combination   → right tool for right component, always justify
Tradeoff formula → gain X, lose Y, acceptable because Z
Red flag      → one DB for everything without justification
```

---

# 🎉 PHASE 2 COMPLETE

```
✅ Chunk 17  — ACID
✅ Chunk 18  — Isolation Levels
✅ Chunk 19  — Indexing (B+Tree, clustered, covering, composite)
✅ Chunk 20  — When SQL Wins
✅ Chunk 21  — Redis Internals + Data Structures
✅ Chunk 21B — Redis Sentinel + Cluster
✅ Chunk 22  — MongoDB Document Stores
✅ Chunk 22B — OLTP vs OLAP + Multi-tenancy
✅ Chunk 23  — Cassandra LSM Internals
✅ Chunk 23B — Cassandra Data Model + Consistency
✅ Chunk 23C — Cassandra Advanced (vnodes, gossip, hot partition)
✅ Chunk 24B — Denormalization
✅ Chunk 24C — Cache Invalidation
✅ Chunk 24D — Redis vs Memcached
✅ Chunk 24E — Request Lifecycle
✅ Master DB Decision Framework ← this file

Next: Phase 3 — Distributed Systems
```

---

*Phase 2 complete — 16/16 chunks done*
*Next: Phase 3 — Replication, Sharding, Consistency Models*
*Target: Razorpay, PhonePe, Flipkart, Zerodha, Zepto*