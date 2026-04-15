# Chunk 35 — Replication Lag Problems
### Phase 3 — Distributed Systems | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, WhatsApp, Swiggy

---

## What is Replication Lag?

```
Setup:
→ 1 Primary (handles writes)
→ 3 Replicas (handle reads)
→ Async replication (most common)

Async = primary doesn't wait for replicas
→ Replica might be 1-2 seconds behind
→ This gap = REPLICATION LAG

Root cause of ALL 3 problems:
→ Eventual consistency
→ Replicas eventually catch up
→ But in that 1-2 second window → problems 😬
```

---

## Problem 1 — Read-After-Write Inconsistency

```
Timeline:
t=0:  User writes review → Primary ✅
t=0:  Primary returns "success" ✅
t=0:  User immediately reads reviews
      → Request goes to Replica 1
      → Replica 1 hasn't synced yet 😬
      → Review missing 😬
t=2s: Replica 1 syncs → review appears ✅

User thinks: "My review was lost!" 😬
```

### Fix 1 — Time-Based Routing:

```
After any write by user:
→ Set flag: "read_from_primary = true, ttl = 60s"
→ For next 60 seconds → all reads → Primary ✅
→ After 60 seconds → back to replica ✅

Why 60 seconds?
→ Replication lag rarely exceeds 1-2 seconds
→ 60 seconds = very safe buffer ✅
→ Simple to implement ✅

Code logic:
if (user.lastWriteTime > now - 60seconds):
    readFrom = PRIMARY
else:
    readFrom = REPLICA
```

### Fix 2 — Replication Offset Tracking:

```
User writes review:
→ Primary returns:
  {success: true, offset: 1000}
→ Client stores offset 1000

User reads reviews:
→ Client sends: "I need data at offset >= 1000"
→ Load balancer checks replicas:
   Replica 1 offset = 998 → too stale ❌
   Replica 2 offset = 1001 → fresh enough ✅
→ Routes to Replica 2 ✅

PostgreSQL: uses LSN for this ✅
```

### Fix 3 — Read Own Writes from Primary:

```
After write → route that specific data to primary:
→ User's own profile/settings → primary ✅
→ Other users' data → replica ✅

Rule: Only route OWN writes to primary
      Not all reads → avoids overloading primary
```

### Real World Implementations:

```
Facebook:
→ After write → primary for 20 seconds ✅

Amazon DynamoDB:
→ "Strongly consistent reads" option
→ Always reads from primary ✅
→ Costs 2x read capacity 😬

Razorpay merchant settings:
→ After update → primary for 30 seconds ✅
→ Then back to replica ✅

MySQL ProxySQL:
→ Tracks write timestamp per session
→ Routes to primary until replica catches up ✅
```

---

## Problem 2 — Monotonic Reads

```
User refreshes Swiggy order status:

Refresh 1 → Replica 1 → "Order delivered" ✅
Refresh 2 → Replica 2 → "Order out for delivery" 😬
Refresh 3 → Replica 1 → "Order delivered" ✅

User sees order going BACKWARDS in time 😬

Why:
→ Replica 1 = more synced (has "delivered") ✅
→ Replica 2 = less synced (still "out for delivery") ⚠️
→ Different replicas at different sync points
→ User hits different replica each request
```

### Fix — Hash User to Same Replica:

```
Rule: Same user always reads from SAME replica

Implementation:
→ hash(user_id) % number_of_replicas = replica

user:123 → hash(123) % 3 = 1 → always Replica 1 ✅
user:456 → hash(456) % 3 = 2 → always Replica 2 ✅

Result:
→ user:123 always sees Replica 1's view
→ May be slightly stale BUT
→ Never goes backwards in time ✅
→ Monotonically increasing ✅

Failure handling:
→ Replica 1 goes down?
→ Re-hash to different replica
→ May see older data once → acceptable ✅
```

---

## Problem 3 — Consistent Prefix Reads

```
WhatsApp conversation:
Message 1: "Are you coming to office?" (written first)
Message 2: "Yes, see you at 10!" (written second)

User B reads:
→ Sees Message 2: "Yes, see you at 10!" 😬
→ Without Message 1
→ Makes no sense 😬

Why:
→ Message 2 synced to Replica 2 faster
→ Message 1 still in transit to Replica 2
→ Network: Message 2 packet arrived first 😬
→ OR different partitions:
   Message 1 → Shard A
   Message 2 → Shard B
   No ordering across shards 😬
```

### Fix 1 — Sequence Numbers:

```
Assign sequence number to every write:
Message 1 → seq: 1001
Message 2 → seq: 1002

Replicas:
→ Never apply seq 1002 before 1001 ✅
→ Buffer 1002 until 1001 received ✅
→ Order preserved ✅
```

### Fix 2 — Same Partition for Related Writes:

```
All messages in same conversation:
→ Partition key = conversation_id
→ All go to same replica/partition ✅
→ Order guaranteed within partition ✅

WhatsApp:
→ Partition by conversation_id ✅
→ Messages always in order ✅

Kafka:
→ Same principle
→ Order guaranteed within partition ✅
→ No order guarantee across partitions ❌
```

### Fix 3 — Causal Dependency Tracking:

```
Message 2 depends on Message 1:
→ Track this dependency
→ Don't serve Message 2 until Message 1 visible ✅
→ Vector clocks implement this ✅
→ Covered in Chunk 44 — Causal Consistency
```

---

## Quorum and These Problems

```
Quorum (Cassandra/DynamoDB) DOES help:

W + R > N:
Write: W=2 nodes must confirm
Read:  R=2 nodes must respond
→ Overlap node always has latest write ✅
→ Read-after-write solved at DB layer ✅

BUT Quorum only works in leaderless systems:
→ Cassandra ✅
→ DynamoDB ✅

Single-leader (PostgreSQL):
→ Only primary accepts writes
→ No quorum concept
→ Need application-level fixes ✅

So:
→ Cassandra → Quorum solves it ✅
→ PostgreSQL → application routing fixes ✅
```

---

## All 3 Problems — Summary Table

| Problem | What Happens | Root Cause | Fix |
|---|---|---|---|
| **Read-After-Write** | User reads own stale data | Replica not synced yet | Route to primary for 60s after write |
| **Monotonic Reads** | Data appears to go backwards | Different replicas, different sync | Hash user_id → always same replica |
| **Consistent Prefix** | Messages out of order | Related writes arrive out of order | Sequence numbers / same partition |

---

## How They Relate:

```
All 3 problems:
→ Root cause = async replication = eventual consistency
→ Replicas eventually catch up
→ Problems occur in the sync window

Severity:
→ Read-After-Write: confusing UX 😬
→ Monotonic Reads: very confusing UX 😬
→ Consistent Prefix: broken UX (nonsense conversation) 😬

Fix philosophy:
→ Strong consistency → read from primary always
   → Eliminates all 3 ✅
   → But high latency + primary overloaded 😬

→ Eventual consistency + smart routing
   → Fixes all 3 ✅
   → Better performance ✅
   → More complex application logic ❌
```

---

## Real Production Examples

```
Flipkart product review:
→ User posts review → writes to primary
→ User immediately views reviews
→ Route to primary for 60 seconds ✅
→ Review always visible ✅

Swiggy order status:
→ hash(user_id) → same replica ✅
→ Order status never goes backwards ✅

WhatsApp messages:
→ Partition by conversation_id ✅
→ Messages always in order ✅
→ Sequence numbers as backup ✅

Razorpay merchant settings:
→ After update → primary for 30 seconds ✅
→ Merchant always sees latest settings ✅
```

---

## Interview Framing

**On replication lag:**
> *"Async replication creates an eventual consistency window where replicas lag 1-2 seconds behind primary. This causes three problems: read-after-write inconsistency, monotonic read violations, and consistent prefix violations."*

**On read-after-write:**
> *"After a user writes, their subsequent reads should go to primary for ~60 seconds. This ensures they always see their own writes even if replicas haven't caught up. Other users can still read from replicas."*

**On monotonic reads:**
> *"Hash the user_id to always route to the same replica. Even if that replica is slightly stale, the user will never see data go backwards in time — they always see the same replica's monotonically increasing view."*

**On consistent prefix:**
> *"For ordered data like chat messages, partition by conversation_id so all related writes go to the same partition. Order is guaranteed within a partition. Sequence numbers provide an additional safety net."*

**On quorum:**
> *"In leaderless systems like Cassandra, QUORUM reads solve read-after-write at the DB layer — the overlap between write and read quorum guarantees the latest write is always visible. In single-leader systems like PostgreSQL, we need application-level routing fixes instead."*

---

## Quick Drill Questions

```
Q1: What is replication lag?
    What causes it?

Q2: User posts comment, immediately reads it,
    doesn't see it. Which problem is this?
    How do you fix it?

Q3: User refreshes page and sees older data
    than before. Which problem?
    Exact fix?

Q4: WhatsApp message B appears before
    message A even though A was sent first.
    Which problem? 3 fixes?

Q5: Does quorum solve all 3 problems?
    When does it work and when doesn't it?

Q6: What's the difference between
    read-after-write and monotonic reads?
```

---

## Drill Answers

### Q1: Replication lag
```
What: Gap between primary and replica data
      Replica is 1-2 seconds behind primary

Cause: Async replication
→ Primary returns success immediately
→ Sends to replicas in background
→ Replica hasn't applied write yet
→ Gap = replication lag

Root cause: Eventual consistency
→ Replicas "eventually" catch up
→ Problems occur in that window
```

### Q2: Read-After-Write
```
Problem: Read-After-Write Inconsistency

User writes → primary ✅
User reads → replica (not synced yet) 😬
Own write not visible 😬

Fix options:
1. Time-based: read from primary for 60 seconds ✅
2. Offset tracking: read replica with offset ≥ write offset ✅
3. Sticky routing: all user's reads → primary after write ✅
```

### Q3: Monotonic reads violation
```
Problem: Monotonic Reads Violation

Cause:
→ Different requests hit different replicas
→ One replica more synced than other
→ User sees "newer" then "older" data 😬

Fix:
→ hash(user_id) % num_replicas = replica_number
→ Same user ALWAYS hits same replica ✅
→ May be stale but never backwards ✅
```

### Q4: Consistent prefix reads
```
Problem: Consistent Prefix Reads

Cause:
→ Message B reached replica faster than A
→ Network conditions / different partitions
→ User reads B without A 😬

Fix 1: Sequence numbers
→ Assign seq to each write
→ Replica applies in order ✅

Fix 2: Same partition
→ Partition by conversation_id
→ All messages → same partition → same order ✅

Fix 3: Causal dependency tracking
→ B depends on A → don't serve B until A visible
→ Vector clocks ✅
```

### Q5: Quorum and the 3 problems
```
Quorum WORKS for leaderless systems:
→ Cassandra, DynamoDB ✅
→ W + R > N → overlap guarantees latest data
→ Read-after-write solved at DB layer ✅

Quorum DOESN'T apply to single-leader:
→ PostgreSQL, MySQL ❌
→ Only primary accepts writes
→ No quorum concept
→ Need application-level fixes ✅

Quorum for monotonic reads:
→ Partially helps (consistent per request)
→ But doesn't guarantee same replica per user
→ Still need hash routing for true monotonic ✅
```

### Q6: Read-after-write vs Monotonic reads
```
Read-After-Write:
→ Specific to: user reading THEIR OWN writes
→ Problem: my write not visible to me
→ Fix: route MY reads to primary

Monotonic Reads:
→ Any user seeing data go backwards
→ Problem: newer state then older state
→ Fix: always route to same replica

Key difference:
→ Read-after-write = missing your own write
→ Monotonic reads = seeing older data after newer
→ Different problems, different fixes ✅
```

---

## One-Line Summaries

```
Replication lag    → replica 1-2s behind primary
Eventual consistency → root cause of all 3 problems
Read-after-write   → own write not visible → route to primary 60s
Monotonic reads    → data goes backwards → hash user to same replica
Consistent prefix  → writes out of order → sequence numbers / partition
Quorum fix         → works in leaderless (Cassandra), not single-leader
Time-based routing → simplest fix, 60 second window
Offset tracking    → precise fix, route to replica with matching offset
Same partition     → order guaranteed within partition (Kafka/Cassandra)
```

---

*Chunk 35 complete*
*Next: Chunk 36 — Multi-Leader + Leaderless Replication*
*Phase 3 progress: 3/22 chunks done*