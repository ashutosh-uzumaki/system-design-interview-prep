# Chunk 21B — Drill Questions & Answers
### Redis Sentinel + Cluster | SDE-2 Interview Prep

---

## Q1: What is split brain and how does Sentinel prevent it?

**Answer:**
```
Split brain:
→ Primary crashes
→ Two replicas both think they should be primary
→ Both accept writes simultaneously
→ Data diverges → two sources of truth 😬
→ Impossible to reconcile which data is correct

Example:
→ Replica 1 accepts: payment:123 = SUCCESS
→ Replica 2 accepts: payment:123 = FAILED
→ Which is correct? No way to know 😬

How Sentinel prevents it:
→ Uses QUORUM VOTING
→ Majority of Sentinels must agree before promotion
→ Only ONE replica can win majority vote
→ Only ONE primary ever elected ✅

3 Sentinels → need 2 votes → only one candidate wins
→ Split brain impossible ✅
```

**Interview framing:**
> *"Split brain happens when two nodes both think they're primary and accept conflicting writes. Sentinel prevents this through quorum voting — a majority of Sentinels must agree on the promotion. Since only one candidate can win majority, only one primary is ever elected."*

---

## Q2: How does Sentinel pick which replica to promote?

**Answer:**
```
Scored in this priority order:

Priority 1 — Replication offset (most important)
→ Which replica received most commands from primary?
→ Highest offset = most up-to-date data
→ Minimizes data loss on failover

Example:
Primary wrote 1000 commands:
→ Replica 1: received 998 ← offset 998
→ Replica 2: received 1000 ← offset 1000 ✅
→ Sentinel picks Replica 2 ✅

Priority 2 — Replica priority config
→ Admin can manually assign priority per replica
→ Lower number = higher priority
→ Used to prefer replicas on better hardware

Priority 3 — Run ID (tiebreaker)
→ If everything else equal
→ Lexicographically smaller Run ID wins

Important caveat:
→ Redis replication is ASYNC by default
→ Even best replica may be missing last few commands
→ Small data loss window on failover is unavoidable
→ Acceptable for cache, not for financial ledger
```

---

## Q3: Why does Redis Cluster use 16,384 hash slots instead of hashing directly by node count?

**Answer:**
```
Problem with direct hashing (key % node_count):

6 nodes → key % 6
Add 7th node → key % 7
→ Almost ALL keys map to different nodes
→ Massive reshuffling required
→ System unavailable during reshuffle 😬

Example:
key "payment:123" % 6 = 3 → Node 3
key "payment:123" % 7 = 2 → Node 2
→ Key must move from Node 3 to Node 2
→ This happens for ~85% of all keys 😬

Hash slots solution:
→ 16,384 fixed slots (never changes)
→ slot = CRC16(key) % 16384 (always same formula)
→ Slots assigned to nodes (flexible)

Adding 7th node:
→ Just move ~390 slots from each existing node to Node 7
→ Only keys IN those slots move ✅
→ ~85% of keys stay exactly where they are ✅
→ System stays online ✅
→ Incremental, zero-downtime migration ✅
```

**Interview framing:**
> *"Direct hashing by node count means adding one node reshuffles almost all keys. Hash slots decouple the hashing from node count — 16,384 slots are fixed forever, only slot-to-node assignment changes. Adding a node just moves a subset of slots incrementally."*

---

## Q4: What is the difference between MOVED and ASK redirect?

**Answer:**
```
MOVED redirect:
→ Slot migration is COMPLETE
→ Slot permanently lives on new node
→ "Update your slot map"
→ Talk to new node for ALL future requests for this slot
→ Client updates local slot map ✅

ASK redirect:
→ Slot is CURRENTLY MIGRATING (in progress)
→ This specific key already moved to new node
→ "Just for THIS request, go ask new node"
→ Do NOT update your slot map yet
→ Migration still in progress
→ Other keys in same slot may still be on old node

Example:
Slot 1000 migrating from Node 1 → Node 7

Key A (already moved):
Node 1 → ASK → Node 7 → Client asks Node 7
Client does NOT update slot map ← migration in progress

Migration complete:
Node 1 → MOVED → Node 7 → Client asks Node 7
Client UPDATES slot map ← permanent now ✅
```

**One-line summary:**
```
MOVED = permanent, update your map
ASK   = temporary, don't update your map yet
```

---

## Q5: How does a Redis client know which node to talk to?

**Answer:**
```
Every Redis client maintains a LOCAL slot map:

slot 0     to 2730  → Node 1 (192.168.1.1:6379)
slot 2731  to 5461  → Node 2 (192.168.1.2:6379)
slot 5462  to 8192  → Node 3 (192.168.1.3:6379)
...

How client gets slot map initially:
Step 1 → Connect to ANY node in cluster
Step 2 → Send CLUSTER SLOTS command
Step 3 → Get full slot map back
Step 4 → Cache locally

How client routes a request:
Step 1 → Calculate slot: CRC16("payment:123") % 16384 = 7823
Step 2 → Look up slot map: slot 7823 → Node 3
Step 3 → Talk directly to Node 3 ✅

No middleman. No routing proxy. Client routes itself.

How slot map stays updated:
→ Node returns MOVED redirect when slot has moved
→ Client updates its map automatically
→ Self-healing ✅
```

**Interview framing:**
> *"Redis clients maintain a local slot map cached from CLUSTER SLOTS command. They calculate the slot via CRC16, look up the correct node locally, and talk directly — no proxy needed. MOVED redirects keep the map up to date automatically."*

---

## Q6: What happens to requests during slot migration?

**Answer:**
```
During migration of slot 1000 from Node 1 → Node 7:

Scenario A — Key not yet migrated (still on Node 1):
Client → Node 1 → Node 1 serves directly ✅

Scenario B — Key already migrated to Node 7:
Client → Node 1
Node 1: "This key moved, ASK Node 7"
Client → Node 7 → Node 7 serves directly ✅

At NO point is the key unavailable:
→ Either Node 1 serves it
→ Or Node 1 redirects to Node 7
→ Client always gets the data ✅
→ Zero downtime during reshuffling ✅

After migration complete:
→ Node 1 sends MOVED (not ASK)
→ Client updates slot map
→ Future requests go directly to Node 7 ✅
```

---

## Q7: Razorpay needs Redis for rate limiting at 10M txns/day. Sentinel or Cluster? Why?

**Answer:**
```
Answer: Redis Cluster

Why not Sentinel:
→ Sentinel = one primary handling all writes
→ 10M transactions/day = very high write throughput
→ Rate limiting = INCR on every API call
→ Single primary becomes bottleneck 😬
→ Data may exceed single node RAM

Why Cluster:
→ Horizontal write scaling
   → Rate limiting counters distributed across nodes
   → Each node handles subset of merchants
   → No single bottleneck ✅

→ Data capacity
   → Multiple nodes × RAM per node
   → Handles massive key count ✅

→ Built-in HA
   → Each shard has replica
   → Automatic failover per shard ✅

→ Hash by merchant_id
   → merchant:M123:ratelimit → always same node
   → Consistent routing ✅

Specific setup:
→ 6 nodes minimum
→ Each with replica
→ Shard key = merchant_id
→ AOF persistence (losing counters = wrong rate limits)
```

**Interview framing:**
> *"I'd use Redis Cluster for Razorpay rate limiting at this scale. 10M transactions/day means Sentinel's single primary becomes a write bottleneck. Cluster distributes rate limiting counters across nodes via hash slots, giving us horizontal write scaling plus built-in HA per shard."*

---

## Q8: What does Redis Cluster NOT support that single Redis does?

**Answer:**
```
❌ Cross-slot multi-key operations:
   MGET key1 key2
   → If key1 and key2 on different nodes → fails 😬

   Workaround: Hash tags
   → {user}.cart and {user}.session
   → {} forces same slot → same node ✅

❌ Cross-slot transactions:
   MULTI
   SET key1 val1   ← Node 1
   SET key2 val2   ← Node 3 (different node)
   EXEC
   → Fails in Cluster mode 😬

❌ Some Lua scripts:
   → Scripts touching keys on multiple nodes fail

❌ Database selection:
   → Single Redis has 16 DBs (SELECT 0-15)
   → Cluster only supports DB 0

❌ Operational simplicity:
   → 6+ nodes to manage vs 1
   → More complex monitoring
   → More complex backup/restore

Summary:
→ Any operation touching multiple keys
   on different nodes = not supported
→ Design schemas so related keys
   land on same node (hash tags) ✅
```

---

## Summary — One-Line Answers

```
Q1: Split brain = two primaries → quorum voting prevents it
Q2: Highest replication offset = most up-to-date replica wins
Q3: Hash slots decouple hashing from node count → incremental migration
Q4: MOVED = permanent (update map), ASK = temporary (don't update)
Q5: Client holds local slot map → CRC16(key) → node lookup → direct talk
Q6: Key still on old node → served directly, moved → ASK redirect → zero downtime
Q7: Cluster → single primary write bottleneck at 10M txns/day
Q8: Cross-slot operations, cross-node transactions, multiple DBs
```

---

*Chunk 21B Drill complete*
*Next: Chunk 22 — Document Stores (MongoDB)*
*Phase 2 progress: 6/8 chunks done*