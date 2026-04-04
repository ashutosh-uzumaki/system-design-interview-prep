# Phase 2 — SQL Internals
### Chunks 17–20 | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Zerodha

---

## Chunk 17 — ACID

### What is ACID?
A set of 4 properties that guarantee database transactions are processed reliably.

| Property | Simple version | What breaks without it |
|---|---|---|
| **Atomicity** | All or nothing | Debit happens, credit doesn't |
| **Consistency** | DB always in valid state | Foreign key points to deleted row |
| **Isolation** | Concurrent txns don't interfere | Two users read same seat as available |
| **Durability** | Committed data survives crashes | Payment confirmed, then lost on restart |

### Real failure examples:

```
Without Atomicity (Razorpay):
→ Debit user ₹500 ✅
→ System crashes
→ Credit merchant ₹500 ❌ never happens
→ Money lost in transit 😬

Without Isolation (Zerodha):
→ User A reads stock price = ₹100
→ User B updates price to ₹150
→ User A's transaction uses stale ₹100
→ Wrong trade executed 😬

Without Durability:
→ Payment confirmed to user ✅
→ Server crashes before flush to disk
→ Payment record gone on restart 😬
```

### Interview framing:
> *"ACID is non-negotiable for financial systems. At Razorpay, Atomicity ensures a payment either fully completes or fully rolls back — there's no in-between state where money is debited but not credited."*

---

## Chunk 18 — The 4 Isolation Levels

### The problem: concurrent transactions interfere
Three types of interference exist:

| Problem | What happens |
|---|---|
| **Dirty Read** | Read uncommitted data from another transaction |
| **Non-repeatable Read** | Same row read twice, different values |
| **Phantom Read** | Same query run twice, different rows returned |

### The 4 isolation levels (weakest → strongest):

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| **Read Uncommitted** | ❌ Possible | ❌ Possible | ❌ Possible | Fastest |
| **Read Committed** | ✅ Prevented | ❌ Possible | ❌ Possible | Fast |
| **Repeatable Read** | ✅ Prevented | ✅ Prevented | ❌ Possible | Medium |
| **Serializable** | ✅ Prevented | ✅ Prevented | ✅ Prevented | Slowest |

### Real examples:

```
Read Committed (default PostgreSQL):
→ Razorpay dashboard reads only committed payments
→ No dirty reads ✅
→ But if you query balance twice in same txn = might differ ⚠️

Repeatable Read (default MySQL):
→ Zerodha: read stock price at txn start, stays same throughout
→ No surprise mid-transaction changes ✅

Serializable:
→ Seat booking (BookMyShow): only one user can book seat 4A
→ Strongest guarantee, highest lock contention 😬
```

### The tradeoff formula:
```
Higher isolation = safer data, more locks, slower throughput
Lower isolation  = faster, but risk of anomalies

Pick the lowest level that still prevents 
the anomalies your system cannot tolerate.
```

### Interview framing:
> *"For Razorpay payments I'd use Read Committed — prevents dirty reads without the performance hit of Serializable. For seat booking I'd use Serializable — phantom reads would let two users book the same seat."*

---

## Chunk 19 — Indexing

### Clustered vs Non-Clustered Index

| | Clustered | Non-Clustered |
|---|---|---|
| **What's stored** | Actual rows inside index | Pointer to actual row |
| **Separate structure?** | No — IS the table | Yes — separate B+ Tree on disk |
| **MySQL behavior** | Primary Key = clustered | All other indexes = non-clustered |
| **PostgreSQL behavior** | No native clustered index | All indexes point to heap |

```
Clustered (MySQL PK):
─────────────────────
[id=1 | alice | 25]   ← row lives inside index
[id=2 | bob   | 30]
[id=3 | carol | 22]

Non-Clustered (PostgreSQL / MySQL secondary):
─────────────────────────────────────────────
Index:                    Heap:
"alice@gmail" → ptr ────► [id=1, alice, 25]
"bob@gmail"   → ptr ────► [id=2, bob, 30]
```

---

### Heap Fetch (Double Lookup)

The two-step process for non-clustered index lookups:

```
Step 1 → Search index tree → find pointer
Step 2 → Follow pointer   → fetch row from heap

Step 2 = Heap Fetch = random disk I/O (expensive)
```

---

### MySQL vs PostgreSQL pointer difference

| | MySQL | PostgreSQL |
|---|---|---|
| **Pointer type** | Primary Key value | Physical heap address (file, page, slot) |
| **Row moves?** | Index still valid ✅ | Pointer goes stale ⚠️ |
| **Extra cost** | Double B+Tree traversal | Needs VACUUM to fix stale pointers |

**Why MySQL stores PK not disk address:**
> If a row moves (page split, compaction), raw disk addresses in all indexes become stale. Storing PK means the clustered index always has the current location — only one place to update.

---

### Covering Index

When ALL columns needed by a query (SELECT + WHERE) are in the index → no heap fetch needed.

```sql
-- Index on (email, name)
-- Query: SELECT name FROM users WHERE email = 'alice@gmail.com'

Normal lookup:          Covering index:
──────────────          ───────────────
Search index            Search index
    ↓                       ↓
Get pointer             Get name directly ✅
    ↓                   DONE. Never touched heap.
Heap fetch
    ↓
Get name
```

**Tradeoff:**
| Gain | Lose |
|---|---|
| Zero heap fetch | Larger index size |
| Faster reads (sequential I/O) | More memory consumed |
| Less disk I/O | Slower writes |

---

### Composite Index

One index on multiple columns — single B+ Tree built on combined columns.

```sql
CREATE INDEX idx_user_status ON orders(user_id, status);
```

```
Index tree sorted by user_id first, then status within each user_id:
────────────────────────────────────────────────────────────────────
(123, 'CANCELLED')  → ptr
(123, 'DELIVERED')  → ptr  ← single lookup ✅
(123, 'PENDING')    → ptr
(124, 'DELIVERED')  → ptr
```

**vs two separate indexes:**
```
INDEX(user_id) + INDEX(status) separately:
→ DB either picks one and filters manually
→ Or does index merge (two scans + intersection)
→ Both expensive 😬
```

---

### Left-Prefix Rule

The DB can only use an index from left to right. Cannot skip middle columns.

```
INDEX on (A, B, C):

WHERE A           → ✅ Fully efficient
WHERE A, B        → ✅ Fully efficient
WHERE A, B, C     → ✅ Fully efficient
WHERE B           → ❌ Full index scan
WHERE C           → ❌ Full index scan
WHERE B, C        → ❌ Full index scan
WHERE A, C        → ⚠️ Partially efficient (A used, C filtered manually)
```

**Real example (Razorpay):**
```sql
-- Query: WHERE merchant_id = 'M123' AND status = 'FAILED' AND created_at > '2024-01-01'

-- Good index:
(merchant_id, status, created_at)
      ↑            ↑         ↑
  equality      equality    range ← range always rightmost
```

**Column ordering rules:**
```
1. Most frequently queried column → leftmost
2. Equality columns → before range columns
3. Highest cardinality → leftmost
4. Range columns → rightmost
```

---

### Over-Indexing Trap

Every index = separate B+ Tree updated on every INSERT/UPDATE/DELETE.

```
Table with 7 indexes:
1 INSERT = 1 heap write + 7 index updates = 8 physical writes

Write amplification × number of indexes 😬
```

**How to identify indexes to remove:**
```
Ask 3 questions per index:
1. Is it redundant? (covered by another via left-prefix)
2. Is it low cardinality? (status with 4 values = useless alone)
3. Is it actually used by any query?
```

**Real example cleanup:**
```sql
-- Given these indexes on transactions:
INDEX(merchant_id)              ❌ Remove — redundant, covered by (merchant_id, status)
INDEX(user_id)                  ❌ Remove — redundant, covered by (user_id, status, amount)
INDEX(status)                   ❌ Remove — low cardinality (4 values only)
INDEX(merchant_id, status)      ✅ Keep — high traffic query
INDEX(merchant_id, created_at)  ✅ Keep — dashboard query
INDEX(user_id, status, amount)  ✅ Keep — covering index
INDEX(status, created_at)       ❌ Remove — low cardinality leftmost

7 indexes → 3 indexes ✅
```

**Rule of thumb:**
```
Read-heavy system  → more indexes acceptable
Write-heavy system → be very conservative
```

---

### PostgreSQL CLUSTER command

```sql
CLUSTER orders USING idx_user_id;
```

- Physically reorders heap to match index order — **one time only**
- New inserts go back to random heap positions
- Must re-run CLUSTER manually to maintain order
- NOT truly clustered like MySQL ❌

---

### UUID concern — MySQL vs PostgreSQL

| | MySQL | PostgreSQL |
|---|---|---|
| **Problem** | Page splits (rows live in PK index) | Heap fragmentation |
| **Root cause** | Random UUID = non-sequential inserts into clustered index | Related rows scattered across heap |
| **Fix** | UUIDv7 or ULID (time-ordered) | UUIDv7 or ULID |
| **Background process** | N/A | VACUUM cleans stale pointers |

---

## Chunk 20 — When SQL Wins

### 4 exact conditions where SQL wins:

```
1. ACID required
   → Financial transactions, stock trades
   → Partial writes = catastrophic

2. Relational data
   → Foreign keys, joins needed
   → Data integrity at DB level, not app level

3. Flexible querying
   → Queries not known upfront
   → Ad-hoc analytics, dashboards

4. Predictable schema
   → Structure doesn't change often
   → SQL rigid schema = feature, not bug
```

### When SQL loses:

```
❌ Massive write throughput (10M+ writes/sec)
❌ Flexible/unpredictable schema (product catalogs)
❌ Hierarchical/graph data (social networks)
❌ Global low-latency reads (multi-region)
```

### Real company choices:

| Company | DB | Why |
|---|---|---|
| Razorpay | PostgreSQL | ACID for payments |
| Zerodha | PostgreSQL | ACID for trades |
| PhonePe | MySQL | Consistency for transactions |
| Flipkart | SQL | Relational orders data |
| Hotstar | Cassandra | 500M users, write-heavy, no ACID needed |

---

### How Razorpay scales SQL at 10M transactions:

SQL doesn't scale on its own at this level. These layers are added on top:

```
Layer 1 — Sharding
→ Partition by merchant_id across multiple PostgreSQL instances
→ Writes distributed ✅
→ But: no cross-shard joins, no cross-shard transactions ⚠️

Layer 2 — Read Replicas
→ Primary handles writes only
→ 3+ replicas handle read traffic
→ 80% of traffic offloaded ✅

Layer 3 — Redis Cache
→ Cache payment status, merchant data
→ Repeated reads never touch DB ✅

Layer 4 — Kafka (async writes)
→ API writes to Kafka (fast)
→ Consumer writes to PostgreSQL (rate controlled)
→ DB never hit with sudden burst ✅

Layer 5 — PgBouncer (connection pooling)
→ 10000 app connections → 500 DB connections
→ Connections reused efficiently ✅
```

```
User Request
     ↓
API Layer (stateless, horizontally scaled)
     ↓
Redis Cache ──────────────────────────────► return if hit ✅
     ↓ miss
Kafka (buffer + smooth spikes)
     ↓
PgBouncer (connection pooling)
     ↓
PostgreSQL Shard (correct shard via merchant_id)
     ↓
Read Replicas (handle read traffic)
```

> **Key insight:** SQL is not inherently unscalable. NoSQL gives scale out of the box. SQL gives ACID — and you engineer the scale on top. Razorpay chose correctness first, then built scale around it.

---

### The decision framework:

```
Ask these 4 questions:

1. Do I need ACID?           → Yes → SQL
2. Is data relational?       → Yes → SQL
3. Is schema predictable?    → Yes → SQL
4. Massive scale + flexible? → Evaluate NoSQL

All 4 yes → SQL is almost always right.
```

---

### Interview framing:

> *"I'd use PostgreSQL here. We need ACID — specifically Atomicity, since a stock trade must debit one account and credit another atomically. The data is relational with foreign keys between trades, accounts, and instruments. Schema is stable and predictable. These are exactly the conditions where SQL wins."*

---

## Quick Reference — The Tradeoff Formula

```
"We chose [X] over [Y].
 We gain [benefit A].
 We lose [cost B].
 That's acceptable because [reason C]."

Example:
"We chose Cassandra over PostgreSQL for watch history.
 We gain horizontal scalability and high write throughput.
 We lose ACID guarantees.
 That's acceptable because losing a watch position 
 is a minor UX issue, not a financial loss."
```

---

## Phase 2 Progress

```
✅ Chunk 17 — ACID
✅ Chunk 18 — 4 Isolation Levels
✅ Chunk 19 — Indexing (B+Tree, Clustered, Non-Clustered,
              Covering, Composite, Left-Prefix, Over-indexing)
✅ Chunk 20 — When SQL Wins
⬜ Chunk 21 — Key-Value Stores (Redis, DynamoDB)
⬜ Chunk 22 — Document Stores (MongoDB)
⬜ Chunk 23 — Wide-Column Stores (Cassandra)
⬜ Chunk 24 — Graph DBs + Decision Framework
```

---

*Score so far: 90% average*
*Target: SDE-2 at Razorpay, PhonePe, Flipkart*