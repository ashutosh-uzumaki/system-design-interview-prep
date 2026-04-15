# Chunks 33-34 — Why Replicate + Single-Leader Replication
### Phase 3 — Distributed Systems | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Hotstar

---

# Chunk 33 — Why Replicate

## 3 Problems Replication Solves

```
Problem 1 — Data loss on crash:
→ One server → crashes → data gone 😬
→ Replication → copy on another server ✅
→ Durability guaranteed ✅

Problem 2 — Read scaling:
→ 10M users reading simultaneously → slow 😬
→ Replicas handle read traffic ✅
→ Primary handles writes only ✅
→ Read load distributed ✅

Problem 3 — Availability:
→ Server down → system unavailable 😬
→ Replica promoted to primary ✅
→ System stays up ✅
→ High availability achieved ✅
```

## Simple Analogy:

```
Without replication:
→ One copy of document on laptop
→ Laptop stolen → everything gone 😬

With replication:
→ Copy on laptop + Google Drive + hard disk
→ Laptop stolen → still have data ✅
→ Multiple people read simultaneously ✅
→ One copy unavailable → use another ✅
```

## 3 Goals of Replication:

```
1. Durability    → data survives crashes
2. Read scaling  → distribute read load
3. Availability  → survive node failures
```

---

# Chunk 34 — Single-Leader Replication

## How It Works

```
Single-Leader Replication:
→ Primary (leader)    → handles ALL writes
→ Replicas (followers) → handle READ traffic

Write → always goes to primary ✅
Read  → can go to any replica ✅
```

## Step by Step:

```
Step 1: Client writes to primary
→ INSERT payment record

Step 2: Primary writes to WAL first
→ Crash recovery mechanism ✅

Step 3: Primary applies to storage ✅

Step 4: Primary streams WAL to replicas
→ "Hey replicas, here's my latest changes"

Step 5: Replicas apply the changes ✅

Step 6: All replicas eventually consistent ✅
```

## Visual:

```
Client
  ↓ write
Primary (leader)
  ↓ WAL streaming
  ├── Replica 1 ← reads
  ├── Replica 2 ← reads
  └── Replica 3 ← reads
```

## Real Examples:

```
PostgreSQL → streaming replication ✅
MySQL      → binlog replication ✅
Redis      → master-replica ✅
MongoDB    → replica set ✅
```

---

## WAL — Each Node Has Its Own

```
Primary WAL:
→ Every write → WAL first
→ Then applied to storage
→ Then streamed to replicas

Replica WAL:
→ Receives WAL entries from primary
→ Writes to its OWN WAL first ✅
→ Then applies to its own storage

PostgreSQL streaming replication:
Primary:
→ WAL writer → writes to WAL segment files
→ WAL sender process → streams to replicas

Replica:
→ WAL receiver → receives from primary
→ Writes to replica's own WAL ✅
→ WAL replayer → applies to storage
→ Confirms receipt (sync mode) ✅
```

## WAL protects vs doesn't protect:

```
WAL DOES protect:
✅ Primary crashing mid-write
   → Restarts → replays WAL → recovers
✅ Local data corruption
✅ Application crash before flush

WAL DOES NOT protect:
❌ Data not yet sent to replicas
   → Primary crashes before replication
   → Replicas promoted
   → Old WAL discarded → data LOST 😬
```

## Replication Offset = WAL Position:

```
PostgreSQL LSN (Log Sequence Number):
Primary LSN:   0/1000A50
Replica 1 LSN: 0/1000A50 ✅ (fully synced)
Replica 2 LSN: 0/1000900 ⚠️ (slightly behind)

Failover → Replica 1 elected (highest LSN) ✅
Same concept as Redis Sentinel offset ✅
```

---

## Sync vs Async vs Semi-Sync Replication

### Synchronous:

```
Primary writes
         ↓
Sends to Replica 1
         ↓
WAITS for confirmation
         ↓
Returns "success" to client ✅

Gain:
✅ Zero data loss
✅ Replica always has latest data

Lose:
❌ Slower writes (wait for replica)
❌ Replica slow → client waits 😬
❌ Replica down → writes blocked 😬
```

### Asynchronous:

```
Primary writes
         ↓
Returns "success" immediately ✅
         ↓
Sends to replicas in background

Gain:
✅ Fast writes (no waiting)
✅ Replica down → writes unaffected ✅

Lose:
❌ Data loss possible:
   → Primary confirms write ✅
   → Crashes before replica gets update
   → Replica promoted
   → Payment record GONE 😬
   → Called: replication lag data loss
```

### Semi-Synchronous (production default):

```
Primary writes
         ↓
Sends to Replica 1
         ↓
Waits for AT LEAST ONE replica to confirm
         ↓
Returns "success" to client ✅
         ↓
Other replicas sync in background

Gain:
✅ Zero data loss (one replica confirmed) ✅
✅ Faster than full sync ✅
✅ Available if one replica down ✅
→ Best of both worlds ✅

Used by:
→ MySQL (default) ✅
→ PostgreSQL (synchronous_standby_names) ✅
→ Razorpay payments ✅
```

## Real World Usage:

```
Razorpay payments:
→ Semi-sync ✅
→ Zero data loss non-negotiable
→ Acceptable latency overhead

Hotstar watch history:
→ Async ✅
→ Losing few seconds = acceptable
→ Speed matters more

Banking systems:
→ Sync ✅
→ Zero data loss non-negotiable
→ Latency acceptable tradeoff
```

---

## Failover Process

```
Step 1 — Detect failure:
→ Replicas stop receiving heartbeat
→ After timeout (30-60 seconds)
→ "Primary is down" 😬

Step 2 — Elect new primary:
→ Replica with HIGHEST offset elected ✅
→ Replica 1 (offset 1000) > Replica 2 (998)
→ Replica 1 wins ✅

Step 3 — Reconfigure:
→ Replica 1 promoted to primary ✅
→ Replica 2 + 3 now follow Replica 1
→ Clients redirected to new primary ✅

Step 4 — Old primary comes back:
→ Must become replica (not primary) ✅
→ Syncs missing writes from new primary ✅
```

---

## 4 Failover Problems + Fixes

### Problem 1 — Async Replication Data Loss:

```
Primary had writes #999 + #1000
→ Only #999 reached replicas
→ Primary crashes 😬
→ Replica promoted (only has #999)
→ Write #1000 LOST permanently 😬

Razorpay scenario:
→ Payment confirmed to user ✅
→ Primary crashes before replication
→ Payment record gone on new primary 😬

Fix: Semi-sync replication ✅
→ #1000 must reach at least one replica
→ Before confirming to client
```

### Problem 2 — Split Brain:

```
Primary doesn't crash — just network partition
Replicas can't reach primary
Replicas think primary is down
Elect new primary (Replica 1) 😬

Now TWO primaries:
→ Old primary accepts writes ❌
→ New primary accepts writes ❌
→ Data diverges 😬

Fix: Fencing / STONITH
→ "Shoot The Other Node In The Head"
→ Force old primary to shut down
→ Only one primary at a time ✅
```

### Problem 3 — Failover Timeout:

```
Primary is just SLOW (not dead)
Timeout = 10 seconds
Replicas elect new primary

Original primary recovers
Now two primaries 😬

Fix:
→ Longer timeout (30-60 seconds) ✅
→ Tradeoff: longer timeout = longer downtime
→ But fewer false positives ✅
```

### Problem 4 — Client Routing:

```
Client hardcoded to primary IP
Primary changes after failover
Client still sends to old IP 😬

Fix:
→ Virtual IP (VIP) ✅
→ DNS failover ✅
→ Load balancer handles routing ✅
→ Client talks to VIP, not direct IP
→ VIP always points to current primary ✅
```

---

## Production Failover Tools:

```
PostgreSQL:
→ Patroni ✅ (automatic failover)
→ Uses etcd/Zookeeper for consensus
→ Automatic promotion + reconfiguration

MySQL:
→ MHA (Master High Availability) ✅
→ Orchestrator ✅

Cloud:
→ AWS RDS Multi-AZ ✅
→ Automatic failover ~60-120 seconds
→ DNS updated automatically ✅
```

---

## Everything Connects:

```
Redis Sentinel:
→ Uses replication offset ✅
→ Same concept as PostgreSQL LSN

Cassandra CommitLog:
→ Same concept as WAL ✅
→ Crash recovery + replication

Debezium CDC:
→ Reads PostgreSQL WAL ✅
→ Same WAL used for replication

All distributed systems:
→ Same underlying pattern:
→ Log-based replication ✅
```

---

## Interview Framing

**On replication:**
> *"Replication solves three problems: durability (data survives crashes), read scaling (replicas handle reads), and availability (failover when primary dies). PostgreSQL uses WAL streaming — the same log used for crash recovery is streamed to replicas."*

**On sync vs async:**
> *"For Razorpay payments I'd use semi-synchronous — at least one replica must confirm before returning success. This guarantees zero data loss while avoiding the availability problem of full sync. Hotstar watch history can use async — losing a few seconds of position is acceptable."*

**On WAL:**
> *"Each node maintains its own WAL. The primary streams WAL entries to replicas, which write to their own WAL first before applying to storage. WAL protects against local crashes but not replication lag data loss — if primary crashes before WAL reaches replicas, those writes are gone when the new primary takes over."*

**On failover:**
> *"During failover, the replica with the highest LSN offset is elected. Main risks are: data loss from async replication (fix: semi-sync), split brain from network partition (fix: fencing/STONITH), and client routing issues (fix: virtual IP). Tools like Patroni automate this using distributed consensus."*

---

## Quick Drill Questions

```
Q1: Name 3 problems replication solves.

Q2: What is WAL and what does it protect against?
    What does it NOT protect against?

Q3: Sync vs Async vs Semi-sync:
    When to use each? Give real examples.

Q4: What is split brain and how do you fix it?

Q5: Primary has writes #1-1000.
    Replica 1 has #1-999.
    Replica 2 has #1-995.
    Primary crashes. What happens?

Q6: How does PostgreSQL streaming 
    replication work step by step?
```

---

## Drill Answers

### Q1: 3 problems replication solves
```
1. Durability → copy survives if primary crashes
2. Read scaling → replicas handle read traffic
3. Availability → failover keeps system running
```

### Q2: WAL protects vs doesn't
```
Protects:
✅ Local crash mid-write (restart → replay WAL)
✅ Application crash before storage flush
✅ Local data corruption

Doesn't protect:
❌ Primary crashes before WAL reaches replicas
❌ Replica promoted → old WAL discarded
❌ Data lost permanently 😬
Fix: Semi-sync replication ✅
```

### Q3: Sync vs Async
```
Sync:
→ Zero data loss ✅
→ Blocked if replica down ❌
→ Use: banking, critical financial data

Async:
→ Fast writes ✅
→ Data loss possible ❌
→ Use: Hotstar watch history, activity logs

Semi-sync:
→ Zero data loss + available ✅
→ Use: Razorpay payments, most production systems
```

### Q4: Split brain
```
What: Two primaries accepting writes simultaneously
      Data diverges, no single source of truth 😬

Cause: Network partition → replicas think primary dead
       Elect new primary → old primary still running

Fix: Fencing/STONITH
→ Force old primary to shut down immediately
→ Only one primary at a time guaranteed ✅
```

### Q5: Primary crashes scenario
```
Replica 1 elected (highest offset #999) ✅
Write #1000 LOST permanently 😬
(Was only on primary's WAL, not replicated)

Fix: Semi-sync replication
→ Primary waits for Replica 1 to confirm #1000
→ Before returning success to client
→ Replica 1 has #1000 → promoted safely ✅
```

### Q6: PostgreSQL streaming replication
```
Primary side:
→ Write arrives → WAL writer logs to WAL file
→ Applied to storage
→ WAL sender process streams WAL entries

Replica side:
→ WAL receiver process receives entries
→ Writes to replica's own WAL ✅
→ WAL replayer applies to storage
→ Confirms receipt to primary (sync mode)

Offset tracking:
→ LSN (Log Sequence Number) = WAL position
→ Failover → highest LSN replica elected ✅
```

---

## One-Line Summaries

```
Replication     → durability + read scaling + availability
Primary         → handles all writes
Replica         → handles reads, backup for failover
WAL             → write-ahead log, crash recovery + replication
Each node WAL   → primary streams, replicas write own WAL
Sync            → wait for all replicas, zero data loss
Async           → immediate success, data loss risk
Semi-sync       → wait for one replica, best of both
Failover        → highest offset replica elected
Split brain     → two primaries, fix with fencing
STONITH         → force old primary to shut down
Patroni         → automatic PostgreSQL failover tool
```

---

*Chunks 33-34 complete*
*Next: Chunk 35 — Replication Lag Problems*
*Phase 3 progress: 2/22 chunks done*