# Chunk 23 — Cassandra Deep Dive
### Wide-Column Store | LSM Trees | Tunable Consistency
### Phase 2 — Databases | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Hotstar, Uber

---

## Why Cassandra Exists — The Problem

```
Hotstar IPL Final:
→ 500 million users simultaneously
→ Every user's watch history written every second
→ 500 million writes per second

PostgreSQL at this scale:
→ Write queue builds faster than DB processes
→ B-Tree index updated per write (random disk write)
→ Memory fills up
→ Disk I/O maxes out
→ CPU maxes out
→ Connection pool exhausted
→ Writes start failing 😬
→ DB crashes 😬

Even with 100 shards:
→ 5M writes/sec per shard
→ B-Tree random writes still too slow 😬

Root cause:
B-Tree = in-place updates = random disk writes = slow
```

---

## Sequential vs Random Writes

```
Random writes (B-Tree):
→ "Update row at position 7823 on disk"
→ Disk head physically moves to position
→ Writes there
→ Moves again to next position
→ Constant physical movement = slow 😬

Sequential writes (LSM):
→ "Just append to end of file"
→ No jumping around
→ Like writing in a notebook top to bottom
→ Always writing at end = fast ✅

Real numbers:
Random write   → ~100 writes/sec  (HDD)
Sequential write → ~1000 writes/sec (HDD)
→ 10x faster just from write pattern ✅
```

---

## Cassandra Write Path — Step by Step

### Component Overview:

```
RAM:  MemTable     → fast in-memory write buffer
DISK: CommitLog    → crash recovery (sequential append)
DISK: SSTables     → permanent immutable storage

Production disk layout:
/dev/sda → CommitLog (dedicated disk, sequential only)
/dev/sdb → SSTables  (dedicated disk, reads + compaction)
RAM      → MemTable  (lost on crash, rebuilt from CommitLog)

Why separate disks?
→ CommitLog writes compete with SSTable reads on same disk
→ Separate disks = no I/O contention ✅
→ Each gets full disk throughput ✅
```

---

### Step 1 — CommitLog (Crash Recovery):

```
Write arrives: "user:123 watched show:456 at 10:30PM"
         ↓
Written to CommitLog FIRST:
→ Sequential append only ✅
→ Never random writes ✅
→ Survives crashes ✅ (on disk)
→ Takes microseconds ✅

CommitLog on disk:
[write1: user:123, show:456, 10:30PM]
[write2: user:456, show:789, 10:31PM]
[write3: user:123, show:456, 10:32PM] ← just appended
```

---

### Step 2 — MemTable (Fast RAM Buffer):

```
After CommitLog → same write goes to MemTable:
→ Sorted in-memory data structure
→ Like a sorted hashmap in RAM
→ Extremely fast reads + writes ✅
→ Accumulates writes in memory

MemTable in RAM:
user:123 → {show:456, timestamp:10:30PM}
user:456 → {show:789, timestamp:10:31PM}
user:789 → {show:123, timestamp:10:29PM}
→ Sorted by key ✅

Write acknowledged to client HERE ✅
No waiting for full disk flush.
No B-Tree updates. No locks.
```

### Why both CommitLog AND MemTable?

```
MemTable = RAM = lost on crash 😬
CommitLog = disk = survives crash ✅

Server crashes:
→ MemTable gone 😬
→ CommitLog intact ✅
→ On restart → replay CommitLog
→ Rebuild MemTable ✅
→ No data lost ✅

CommitLog = crash recovery mechanism
MemTable  = fast in-memory write buffer
Both needed together ✅
```

---

### Step 3 — SSTable Flush:

```
MemTable fills up (RAM is limited)
         ↓
Flushed to disk as SSTable (Sorted String Table):
→ Immutable file on disk ✅
→ Sorted by key ✅
→ Sequential write (fast) ✅
→ Never modified after written ✅

Disk after multiple flushes:
SSTable 1: [user:100→show:456, user:123→show:789...]
SSTable 2: [user:123→show:999, user:456→show:123...]
SSTable 3: [user:234→show:456, user:567→show:789...]

Problem:
→ user:123 updated 50 times
→ Data scattered across 50 SSTables 😬
→ Read performance degrades 😬
→ This = Read Amplification
```

---

## Cassandra Read Path — 3 Mechanisms

### The Problem:

```
100 SSTables on disk.
user:123 could be in ANY of them.

Naive approach:
→ Search all 100 SSTables for user:123
→ Pick latest timestamp
→ 100 disk reads per query 😬
→ Terrible read performance 😬
```

---

### Mechanism 1 — MemTable Check:

```
Step 1: Check MemTable first (RAM)
→ user:123 in current MemTable?
→ Yes → return immediately ✅ (fastest path)
→ No  → continue to SSTables
```

---

### Mechanism 2 — Bloom Filter:

```
What is it:
→ Tiny probabilistic data structure per SSTable
→ Kept in RAM
→ Answers: "Does this SSTable POSSIBLY have user:123?"

How it works:
When SSTable written:
→ Each key run through 3 hash functions
→ Sets bits in a bit array

bit array: [0,0,1,0,1,1,0,0,1,0,1,0...]

Query: does user:123 exist in SSTable 1?
→ Run user:123 through same 3 hash functions
→ Check those bits:
   Any bit = 0 → DEFINITELY not here → skip ✅
   All bits = 1 → MAYBE exists → read SSTable

Critical property:
→ Never false negative ✅
  (if key IS there, bloom filter ALWAYS says yes)
→ Occasional false positive acceptable
  (says yes but key not there → wasted read, rare)

Memory efficiency:
→ 1M keys in SSTable = ~50MB to store all keys
→ Bloom filter for same keys = ~1MB ✅
→ 50x more memory efficient ✅
→ 100 SSTables × 1MB = 100MB RAM total ✅
```

---

### Mechanism 3 — Sparse Index:

```
Problem with binary search on disk:
→ 20 steps × random disk read = slow 😬

Sparse Index (kept in RAM):
→ Stores every Nth key with disk offset
→ Not every key — just every 128th key

Example:
user:100  → disk offset 0
user:1280 → disk offset 4096
user:2560 → disk offset 8192

Query: find user:123
→ Check sparse index in RAM
→ user:123 between user:100 and user:1280
→ Jump directly to offset 0
→ Scan only 128 entries from there ✅
→ 1 disk read instead of 20 ✅
```

---

### Full Read Path:

```
Read: user:123's watch history
         ↓
Step 1 — MemTable (RAM)
→ Found? Return immediately ✅
→ Not found? Continue
         ↓
Step 2 — Bloom Filter per SSTable (RAM)
→ NO  → skip SSTable ✅
→ YES → candidate SSTable
         ↓
Step 3 — Sparse Index (RAM)
→ Find approximate disk offset
→ Jump directly to offset ✅
         ↓
Step 4 — Read from disk
→ Small sequential scan
→ Find user:123's entry
→ Pick latest timestamp ✅

Read amplification comparison:
Naive:       100 disk reads 😬
With bloom:  ~3-5 disk reads ✅ (filters 95+ SSTables)
With index:  ~3 disk reads ✅  (direct offset jump)
```

---

## Compaction — Periodic Cleanup

### The Problem Without Compaction:

```
Every update = new SSTable entry
Never overwrites old entry

After 1 year:
→ Thousands of SSTables
→ user:123 has 10,000 entries
→ 9,999 are outdated
→ Disk fills up 😬
→ Even bloom + sparse index = slow 😬
```

### What Compaction Does:

```
Before compaction:
SSTable 1: [user:123→show:456 @10:30PM]
SSTable 2: [user:123→show:789 @10:35PM]
SSTable 3: [user:123→show:999 @10:40PM] ← latest
SSTable 4: [user:456→show:123 @10:32PM]
SSTable 5: [user:456→show:456 @10:38PM] ← latest

Compaction steps:
Step 1 → Pick SSTables to merge
Step 2 → Read all simultaneously
Step 3 → Per key → keep only latest timestamp
Step 4 → Write new merged SSTable
Step 5 → Delete old SSTables ✅

After compaction:
SSTable merged: [user:123→show:999 @10:40PM] ✅
                [user:456→show:456 @10:38PM] ✅

Result:
✅ Fewer SSTables
✅ Less disk space
✅ Faster reads
✅ Old versions cleaned up
```

---

### Tombstones — Distributed Deletes:

```
DELETE in Cassandra:
→ Never deletes immediately
→ Writes TOMBSTONE marker instead

SSTable: [user:123 → TOMBSTONE @11:00PM]

Why not delete immediately?
→ Cassandra = distributed across many nodes
→ Delete on node 1
→ Replication brings data back from node 2 😬

Tombstone approach:
→ All nodes see tombstone
→ All nodes delete during compaction ✅
→ Consistent deletion across cluster ✅

During compaction:
→ Sees tombstone for user:123
→ Removes ALL entries for user:123 ✅
→ Removes tombstone itself ✅
```

---

### Compaction Tradeoffs:

```
Gain:
✅ Fewer SSTables → faster reads
✅ Less disk space
✅ Removes stale data + tombstones
✅ Smaller bloom filters + sparse indexes

Lose:
❌ Write amplification
   → Data written once by app
   → Rewritten during compaction (2-3x)
   → Maybe rewritten again next compaction (3-4x)

❌ Compaction uses disk I/O + CPU
   → Can slow reads/writes temporarily
   → Schedule during low-traffic periods
   → Hotstar: compaction at 3AM, not during IPL 😬

Write amplification factor = 3-4x
Acceptable for 500M writes/sec ✅
Much better than B-Tree random writes 😬
```

---

## Full LSM Tree Picture

```
WRITE PATH:
───────────────────────────────────────────────
Write arrives
     ↓
CommitLog (disk, sequential append) ← crash recovery
     ↓
MemTable (RAM, sorted)              ← fast buffer
     ↓ (when full)
SSTable flush (disk, immutable)     ← persistent storage
     ↓ (periodic)
Compaction                          ← merge + cleanup
     ↓
Fewer, larger SSTables ✅

READ PATH:
───────────────────────────────────────────────
Read arrives
     ↓
MemTable (RAM)         ← fastest, check first
     ↓ miss
Bloom Filter (RAM)     ← skip irrelevant SSTables
     ↓ candidates
Sparse Index (RAM)     ← find disk offset
     ↓
Disk read              ← small sequential scan
     ↓
Latest timestamp wins ✅
```

---

## LSM Tree vs B-Tree — Direct Comparison

| | B-Tree | LSM Tree |
|---|---|---|
| **Write type** | Random (in-place) | Sequential (append) |
| **Write speed** | Slower | Faster ✅ |
| **Read speed** | Faster ✅ | Slower (multiple SSTables) |
| **Space efficiency** | Better | Worse (compaction needed) |
| **Write amplification** | Lower | Higher (compaction) |
| **Read amplification** | Lower | Higher (multiple SSTables) |
| **Used by** | PostgreSQL, MySQL, MongoDB | Cassandra, RocksDB, HBase |
| **Best for** | Read-heavy | Write-heavy ✅ |

---

## Real Production Use Cases

```
Hotstar:
→ Watch history: user:X watched show:Y till Z seconds
→ 500M concurrent users during IPL
→ Write-heavy, eventual consistency ok ✅

Uber:
→ Trip data: every GPS location update
→ Millions of rides × updates per second
→ Write-heavy ✅

WhatsApp:
→ Message storage
→ Billions of messages per day
→ Write-heavy, eventual consistency ok ✅

Flipkart:
→ Activity feeds, catalog events
→ High write throughput needed ✅

NOT Cassandra:
→ Razorpay payments  → PostgreSQL (ACID needed)
→ Zerodha trades     → PostgreSQL (ACID needed)
```

---

## Interview Framing

**On why Cassandra:**
> *"Hotstar needs to handle 500M concurrent write operations during IPL. PostgreSQL's B-Tree does random in-place updates — it simply can't sustain that throughput. Cassandra uses LSM Trees — writes go to CommitLog for crash recovery and MemTable for fast buffering, then flush to immutable SSTables sequentially. Sequential writes are 10x faster than random writes."*

**On read performance:**
> *"Cassandra's read path uses three mechanisms to stay fast despite multiple SSTables — bloom filters eliminate irrelevant SSTables entirely, sparse indexes allow direct disk offset jumps, and MemTable serves the hottest data from RAM. Together they reduce 100 potential disk reads to 3-5."*

**On compaction:**
> *"Compaction periodically merges SSTables, removes stale versions and tombstones. The tradeoff is write amplification — data gets rewritten 3-4x — but that's acceptable compared to B-Tree random writes at 500M writes/sec. In production, compaction is scheduled during low-traffic windows."*

---

## Quick Drill Questions

```
Q1: Why can't PostgreSQL handle 500M writes/sec?

Q2: Why are sequential writes faster than random writes?

Q3: What is the role of CommitLog vs MemTable?
    Why do we need both?

Q4: What is a Bloom filter and what is its
    critical property?

Q5: What is Read Amplification and how does
    Cassandra solve it?

Q6: What is a Tombstone and why does Cassandra
    use it instead of immediate deletes?

Q7: What is Write Amplification?
    What causes it in Cassandra?

Q8: Hotstar watch history vs Razorpay payments.
    Cassandra or PostgreSQL for each? Why?
```

---

## Drill Answers

### Q1: PostgreSQL at 500M writes/sec
```
→ B-Tree = random in-place updates = slow disk I/O
→ Write queue builds faster than DB processes
→ Memory fills up, CPU maxes out
→ Connection pool exhausted
→ Writes fail, DB crashes 😬
Even with 100 shards → B-Tree random writes = bottleneck
```

### Q2: Sequential vs random writes
```
Random: disk head physically moves to different positions
→ Constant mechanical movement = slow 😬

Sequential: always append to end of file
→ No movement, just keep writing forward
→ 10x faster than random on HDD ✅
```

### Q3: CommitLog vs MemTable
```
CommitLog (disk):
→ Sequential append for crash recovery
→ If server crashes → replay CommitLog → rebuild MemTable
→ Persistent, survives restart ✅

MemTable (RAM):
→ Fast in-memory sorted buffer
→ Lost on crash → recovered from CommitLog
→ Write acknowledged here (fast) ✅

Need both:
→ MemTable alone = data loss on crash 😬
→ CommitLog alone = slow (disk read for every write)
→ Together = fast writes + crash safety ✅
```

### Q4: Bloom Filter
```
Probabilistic data structure per SSTable in RAM
→ Answers: "Does this key POSSIBLY exist here?"

Critical property:
→ Never false negative ✅
  (key IS there → bloom ALWAYS says yes)
→ Occasional false positive ok
  (says yes but key not there → rare wasted read)

Memory: 1M keys → ~1MB bloom filter
vs storing all keys → ~50MB
→ 50x more efficient ✅
```

### Q5: Read Amplification + solution
```
Read Amplification:
→ 1 logical read → many physical disk reads
→ user:123 updated 50 times → 50 SSTables
→ Naive read = check all 50 SSTables 😬

Solution — 3 mechanisms:
1. Bloom filter → skip irrelevant SSTables (RAM check)
2. Sparse index → direct disk offset jump (RAM lookup)
3. MemTable → serve hottest data from RAM

Result: 100 potential reads → 3-5 actual reads ✅
```

### Q6: Tombstones
```
Cassandra = distributed across many nodes
Immediate delete on one node:
→ Replication brings data back from other nodes 😬

Tombstone = special "deleted" marker written to SSTable
→ All nodes see tombstone via replication
→ All nodes delete during compaction ✅
→ Consistent deletion across cluster ✅

Removed during compaction along with old versions
```

### Q7: Write Amplification
```
1 logical write by app:
→ Written to CommitLog     (1x)
→ Written to MemTable      (memory)
→ Flushed to SSTable       (2x)
→ Rewritten at compaction  (3x)
→ Maybe again at next compaction (4x)

Write amplification factor = 3-4x
Caused by: compaction merging SSTables

Acceptable because:
→ Sequential writes still faster than
  B-Tree random writes even with amplification ✅
```

### Q8: Hotstar vs Razorpay
```
Hotstar watch history → Cassandra ✅
→ 500M concurrent writes during IPL
→ No ACID needed (losing watch position = minor)
→ Eventual consistency acceptable
→ Write-heavy → LSM Tree = perfect fit

Razorpay payments → PostgreSQL ✅
→ ACID non-negotiable (Atomicity critical)
→ Debit + credit must happen together or not at all
→ Strong consistency required
→ Write volume manageable with sharding + Kafka
→ Correctness > write throughput
```

---

## One-Line Summaries

```
CommitLog    → sequential append, crash recovery
MemTable     → sorted RAM buffer, fast writes
SSTable      → immutable sorted disk file
Bloom filter → probabilistic skip check, never false negative
Sparse index → every Nth key + offset, direct disk jump
Compaction   → merge SSTables, remove stale + tombstones
Tombstone    → distributed delete marker
Write amp    → 1 logical write → 3-4 physical writes
Read amp     → 1 logical read → many disk reads (mitigated)
LSM Tree     → sequential writes, write-heavy systems
```

---

*Chunk 23 — LSM Tree internals complete*
*Next: Chunk 23B — Cassandra Data Model + Tunable Consistency*
*Phase 2 progress: 9/13 chunks done*