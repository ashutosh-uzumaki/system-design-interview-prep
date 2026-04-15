# Write Capacity Detection + Kafka Buffering
### Phase 3 Supplement | SDE-2 Interview Prep
> How to detect DB write capacity issues + how Kafka helps

---

# PART 1 — Detecting Write Capacity Issues

## 5 Signals Write Capacity Is Exceeded

### Signal 1 — Replication Lag Increasing:

```
Normal replication lag: ~10-100ms

Warning signs:
→ Lag growing: 1s → 5s → 30s → 5min 😬
→ Replicas falling further behind
→ Primary can't replicate fast enough

PostgreSQL monitoring:
SELECT now() - pg_last_xact_replay_timestamp()
AS replication_lag;

→ lag > 30 seconds = investigate
→ lag > 5 minutes = critical 😬
```

### Signal 2 — Primary CPU/IO Maxed Out:

```
Metrics to watch:
→ CPU usage > 80% consistently 😬
→ Disk I/O wait > 20% 😬
→ Write queue depth increasing 😬
→ WAL generation rate too high

Tools:
→ Datadog / Grafana dashboards
→ pg_stat_activity (PostgreSQL)
→ SHOW PROCESSLIST (MySQL)
```

### Signal 3 — Write Latency Increasing:

```
Normal write latency: ~5-10ms

Warning signs:
→ p99 write latency: 10ms → 50ms → 500ms 😬
→ Timeouts increasing
→ Connection pool exhaustion

Track:
→ p50, p95, p99 write latency
→ NOT just average (averages lie) ✅
```

### Signal 4 — Connection Pool Exhaustion:

```
PgBouncer / HikariCP metrics:
→ Waiting connections > 0 = warning
→ Waiting connections growing = critical 😬
→ "Too many connections" errors

Means:
→ Writes arriving faster than DB processes
→ Connections backing up 😬
```

### Signal 5 — Business Metrics (lagging indicator):

```
Indirect signals:
→ Payment failures increasing
→ Order creation timeouts
→ User complaints about slow checkout

These appear AFTER DB is struggling 😬
→ Always monitor DB BEFORE business impact ✅
```

---

## Alert Thresholds

```
GREEN (normal):
→ Write latency p99 < 20ms
→ Replication lag < 1 second
→ CPU < 70%
→ Connection pool wait = 0

YELLOW (warning → investigate):
→ Write latency p99 > 50ms
→ Replication lag > 5 seconds
→ CPU > 80%
→ Connection pool wait > 10

RED (critical → act now):
→ Write latency p99 > 200ms
→ Replication lag > 30 seconds
→ CPU > 90%
→ Connection pool exhausted
→ Write errors appearing
```

---

## What to Do When Capacity Exceeded

```
Short term (immediate relief):
→ Add read replicas → offload reads from primary
→ Add Redis cache → reduce DB reads
→ Kafka buffer → smooth write spikes ✅
→ Connection pooling tuning

Medium term:
→ Vertical scaling (bigger machine)
→ Query optimization
→ Index tuning

Long term (if nothing else works):
→ Sharding (distribute writes) ✅
→ Switch to Cassandra (if ACID not needed) ✅
→ Multi-leader (if multi-region needed) ✅
```

---

# PART 2 — Kafka Write Buffering

## The Core Problem — Spike vs Sustained Load

```
Without Kafka:
─────────────
Diwali sale starts:
→ 10,000 payments/second HIT DB simultaneously 😬
→ DB overwhelmed instantly
→ Write latency spikes to 500ms 😬
→ Timeouts + failures 😬

SPIKE = sudden burst DB can't handle 😬

With Kafka:
───────────
Diwali sale starts:
→ 10,000 payments/second → Kafka (~5ms) ✅
→ Kafka consumer writes at DB's comfortable pace
→ DB handles 2,000 writes/second comfortably
→ Consumer writes 2,000/second to DB ✅
→ Remaining 8,000 sit in Kafka queue
→ Processed over next few seconds ✅

SPIKE smoothed into steady stream ✅
```

---

## The Analogy

```
Without Kafka:
→ 10,000 people rush through ONE door simultaneously
→ Door jams 😬
→ Everyone stuck 😬

With Kafka:
→ 10,000 people enter waiting room (Kafka)
→ Door lets in 2,000 at a time (DB capacity)
→ Everyone gets through eventually ✅
→ Door never jams ✅
```

---

## Multiple Consumers — Parallelism

```
One consumer = one write at a time → slow 😬

Production: MULTIPLE consumers in parallel:

Kafka topic: payments
Partition 1 → Consumer 1 → DB writes ✅
Partition 2 → Consumer 2 → DB writes ✅
Partition 3 → Consumer 3 → DB writes ✅
Partition 4 → Consumer 4 → DB writes ✅

4 consumers × 500 writes/sec each
= 2,000 writes/sec total ✅

Scale consumers = scale write throughput ✅
Add more partitions = add more consumers ✅
```

---

## The Real Benefit — Decoupling Latency

```
Without Kafka:
→ User waits for DB write (slow during spike)
→ p99 latency = DB write latency
→ 500ms during Diwali spike 😬

With Kafka:
→ User waits for Kafka write (always fast)
→ p99 latency = Kafka write latency = ~5ms ✅
→ DB processes in background ✅
→ User already got "payment processing" ✅
```

---

## Handling Consistency — Redis + Kafka

```
Problem:
User pays → Kafka confirms ✅
User checks payment status immediately:
→ DB hasn't processed yet 😬
→ Status shows "not found" 😬

Fix — Redis status cache:

Step 1: API writes to Kafka (~5ms) ✅
Step 2: API sets Redis:
        "payment:123:status" → "PROCESSING"
        TTL: 10 minutes ✅
Step 3: User reads status → Redis ✅
        Gets "PROCESSING" immediately ✅
Step 4: Kafka consumer processes payment
Step 5: Consumer updates DB ✅
Step 6: Consumer updates Redis:
        "payment:123:status" → "SUCCESS" ✅
Step 7: User notified via webhook/push ✅
```

---

## Full Kafka Buffering Flow

```
User clicks Pay:
         ↓
API writes to Kafka (~5ms) ✅
         ↓
API writes to Redis: status = "PROCESSING" ✅
         ↓
Returns to user: "Payment processing" ✅
         ↓ (async, background)
Kafka Consumer 1 reads payment event
         ↓
Writes to PostgreSQL ✅
         ↓
Updates Redis: status = "SUCCESS" ✅
         ↓
Notifies user via webhook/push notification ✅

User experience:
→ Instant acknowledgment (~5ms) ✅
→ DB processes at comfortable pace ✅
→ No timeouts, no failures ✅
```

---

## When Kafka Buffering DOESN'T Help

```
Sustained high load (not just spikes):
→ 10,000 writes/sec FOREVER
→ Kafka fills up 😬
→ Consumer can never catch up
→ Queue grows indefinitely 😬

Kafka helps most with:
✅ Sudden spikes (Diwali sale, IPL final)
✅ Bursty traffic patterns
✅ Smoothing write load

For sustained load → also need:
→ More consumers (parallelism) ✅
→ DB sharding ✅
→ Both together ✅
```

---

## Real Example — Razorpay Diwali

```
Normal day: 1,000 payments/second
Diwali sale: 10,000 payments/second (10x spike)

Without Kafka:
→ DB overwhelmed 😬
→ Write latency: 5ms → 500ms 😬
→ Payments failing 😬

With Kafka:
→ API writes to Kafka: ~5ms ✅
→ 4 consumers × 2,000 writes = 8,000/sec capacity
→ Spike absorbed ✅
→ Queue clears within seconds ✅
→ User experience: instant ✅
→ DB: never exceeds comfortable rate ✅

Long term fix:
→ Shard by merchant_id ✅
→ Each shard handles subset of merchants
→ Writes distributed permanently ✅
```

---

## Kafka vs Direct DB Write

| | Direct DB Write | Kafka Buffered |
|---|---|---|
| **Write latency** | 5ms normal, 500ms spike | ~5ms always ✅ |
| **DB load** | Matches traffic spikes | Smoothed ✅ |
| **Failure handling** | DB failure = request failure | Kafka retains messages ✅ |
| **Consistency** | Immediate ✅ | Eventual (seconds) ⚠️ |
| **Complexity** | Simple ✅ | More complex ❌ |
| **Best for** | Low-medium traffic | High traffic, spikes ✅ |

---

## Interview Framing

**On detecting write capacity issues:**
> *"We monitor three key metrics: write latency p99, replication lag, and connection pool wait time. When p99 exceeds 50ms or replication lag grows beyond 5 seconds, we investigate. Business metrics like payment failures are lagging indicators — we want to catch issues before they impact users."*

**On Kafka buffering:**
> *"Kafka buffering helps with write spikes, not necessarily sustained load. During Diwali, 10,000 payments/second hit Kafka instead of DB directly. Multiple consumers process at DB's comfortable rate — 2,000/second across 4 partitions. Users get instant acknowledgment from Kafka while DB processes asynchronously. For sustained high load, we also need sharding."*

**On consistency with Kafka:**
> *"The consistency challenge with async Kafka processing is that DB hasn't updated when user checks status. We solve this by writing initial status to Redis immediately — user reads 'processing' from Redis, then consumer updates both DB and Redis when done. User gets consistent view without hitting DB."*

---

## Quick Drill Questions

```
Q1: Name 3 signals that write capacity
    is being exceeded.

Q2: Why does Kafka help with write spikes
    but not necessarily sustained load?

Q3: User pays via Kafka-buffered system.
    Immediately checks payment status.
    What do they see? How do you fix it?

Q4: How do multiple Kafka consumers
    increase write throughput?

Q5: When would you use Kafka buffering
    vs just sharding the DB?
```

---

## Drill Answers

### Q1: Signals of write capacity exceeded
```
1. Write latency p99 increasing:
   → 10ms → 50ms → 200ms+ 😬

2. Replication lag growing:
   → Replicas falling further behind
   → Primary can't keep up with replication

3. Connection pool exhaustion:
   → Waiting connections > 0
   → "Too many connections" errors
   → Writes queuing up
```

### Q2: Kafka spikes vs sustained
```
Spikes:
→ Kafka absorbs burst → releases at DB pace ✅
→ Diwali: 10x for 1 hour → queue clears ✅

Sustained load:
→ 10,000 writes/sec forever
→ Consumer writes 2,000/sec
→ Queue grows 8,000/sec indefinitely 😬
→ Never catches up 😬

Fix for sustained:
→ More consumers + partitions ✅
→ DB sharding ✅
→ Both together ✅
```

### Q3: Payment status consistency
```
Problem:
→ User pays → Kafka ✅
→ User checks status → DB not updated yet 😬

Fix:
→ Write Redis immediately: "payment:123" → "PROCESSING"
→ User reads Redis → sees "PROCESSING" ✅
→ Kafka consumer processes → updates DB + Redis
→ Redis: "payment:123" → "SUCCESS" ✅
→ User notified async ✅
```

### Q4: Multiple consumers
```
One consumer: 500 writes/sec
Four consumers: 4 × 500 = 2,000 writes/sec ✅

Each consumer reads from own partition:
Partition 1 → Consumer 1
Partition 2 → Consumer 2
Partition 3 → Consumer 3
Partition 4 → Consumer 4

All write to DB in parallel ✅
More partitions = more consumers = more throughput ✅
```

### Q5: Kafka vs sharding
```
Use Kafka buffering when:
→ Traffic is spiky (Diwali, IPL) ✅
→ Async processing acceptable ✅
→ Need to decouple write speed from DB speed ✅
→ Quick fix needed ✅

Use sharding when:
→ Sustained high write load ✅
→ Data too large for one machine ✅
→ Long-term architectural solution ✅
→ Strong consistency still needed ✅

Best answer: BOTH together
→ Kafka: handles spikes ✅
→ Sharding: handles sustained load ✅
→ Kafka buys time while sharding is implemented ✅
```

---

## One-Line Summaries

```
Write capacity signals → latency p99, replication lag, connection pool
Kafka buffering       → absorbs spikes, smooths into steady stream
Multiple consumers    → parallel writes, more throughput
Consistency fix       → Redis for immediate status, DB async
Kafka vs sharding     → Kafka for spikes, sharding for sustained
Spike                 → sudden burst, Kafka helps ✅
Sustained load        → forever high, need sharding ✅
p99 latency           → track percentiles not averages
```

---

*Write Capacity + Kafka Buffering supplement complete*
*Next: Chunk 36 — Multi-Leader + Leaderless Replication*
*Phase 3 progress: 3/22 chunks done*