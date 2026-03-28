# Topic 4 — Concurrency Fundamentals
## Complete Notes | Concepts + Interview Q&A + Technical Terms + Drill Q&A

---

## 📊 Topic 4 Stats
- **Chunks covered:** 13, 14, 15, 16
- **Drill score:** 18.5/20 (92%)
- **Status:** ✅ Complete

---

---

# CHUNK 13 — Race Conditions + Dirty Reads

## Core Concept

```
SIMPLE VERSION
────────────────────────────────────────────────────
Bank with one counter, two customers arrive
at exact same time, both withdraw from
same account with ₹500 balance:

Customer A checks → ₹500 ✅
Customer B checks → ₹500 ✅
Customer A withdraws ₹500 → balance = ₹0
Customer B withdraws ₹500 → balance = -₹500 😱

Bank gave ₹1000 when only ₹500 existed
That's a concurrency problem
────────────────────────────────────────────────────
```

---

## Race Condition

```
TECHNICAL VERSION
────────────────────────────────────────────────────
Two or more operations race to read/modify
same data simultaneously → incorrect result

Thread A reads balance = 500
Thread B reads balance = 500
Thread A writes balance = 0  (500-500)
Thread B writes balance = 0  (500-500)
→ ₹1000 withdrawn, balance shows 0
→ Should show -500
→ Bank lost ₹500 😱

DIAGRAM:
Time →
Thread A: READ(500) ──────────────► WRITE(0)
Thread B: READ(500) ──────────────────────────► WRITE(0)
          Both read BEFORE either wrote ❌
────────────────────────────────────────────────────
```

---

## Dirty Read

```
TECHNICAL VERSION
────────────────────────────────────────────────────
One operation reads data that another
operation is in the middle of changing
— not yet committed

Thread A starts updating order status
→ "Processing" (not committed yet)
Thread B reads order status
→ sees "Processing" ❌
Thread A fails → rolls back
→ status back to "Pending"
Thread B already acted on wrong data 😱

DIAGRAM:
Time →
Thread A: START ──► UPDATE(Processing) ──► ROLLBACK
Thread B:                  READ(Processing) ❌
                           Acts on wrong data 😱
────────────────────────────────────────────────────
```

---

## Real World Examples

```
RACE CONDITIONS:
  → Two users book last seat (IRCTC)
  → Two users buy last item (Flipkart)
  → Two users transfer same money (PhonePe)
  → Two warehouse workers ship same item (Zepto)

DIRTY READS:
  → User sees order "Processing"
    but it gets cancelled
  → User sees balance updated
    but transaction rolls back
  → Delivery partner assigned to
    cancelled order
```

---

## Tradeoff

```
ALLOWING CONCURRENCY
──────────────────────────────────────────────────
✅ Higher throughput
✅ System feels fast
❌ Data integrity risk
❌ Race conditions possible
❌ Dirty reads possible

SERIAL EXECUTION (no concurrency)
──────────────────────────────────────────────────
✅ Always correct data
❌ Very slow
❌ All requests queue up
❌ Not scalable

SWEET SPOT:
  Allow concurrency
  BUT protect critical sections
  → Locking mechanisms
──────────────────────────────────────────────────
```

---

## Technical Terms

| Plain English | Technical Term |
|---|---|
| Two ops read before either writes | Race Condition |
| Reading uncommitted data | Dirty Read |
| Protecting shared resource | Critical Section |
| Multiple ops at same time | Concurrency |
| One at a time | Serial Execution |

---

## Interview Framing

> *"Concurrency problems occur when multiple operations access shared data simultaneously. Race conditions happen when two operations read before either writes — leading to incorrect results. Dirty reads happen when one operation reads uncommitted data from another. For a payment system these are critical — we need locking mechanisms to protect critical sections while still allowing concurrent operations on different data."*

---
---

# CHUNK 14 — Optimistic vs Pessimistic Locking

## Core Concept

```
PESSIMISTIC LOCKING — "I don't trust anyone"
────────────────────────────────────────────────────
Like a toilet with a lock:
You go in → lock door immediately
Nobody else enters until you come out
Safe but only one person at a time

OPTIMISTIC LOCKING — "I trust, but verify"
────────────────────────────────────────────────────
Like a shared Google Doc:
Everyone edits simultaneously
When you save → checks if anyone
else changed it while you edited
If yes → your save fails → retry
If no → save succeeds ✅
────────────────────────────────────────────────────
```

---

## Pessimistic Locking — Implementation

```sql
-- PostgreSQL
BEGIN;
SELECT * FROM seats
WHERE seat_id = '34A'
FOR UPDATE;  -- locks this row only

-- Booking logic here
UPDATE seats SET status = 'booked'
WHERE seat_id = '34A';

COMMIT;  -- releases lock
```

```
WHAT HAPPENS:
──────────────────────────────────────────────────
User A: SELECT FOR UPDATE seat 34A
        → Row locked ✅

User B: SELECT FOR UPDATE seat 34A
        → WAITS... ⏳

User A: UPDATE + COMMIT → lock released

User B: Now reads seat 34A
        → Already booked → "unavailable" ✅
──────────────────────────────────────────────────
```

---

## What Gets Locked — IMPORTANT

```
NOT THE WHOLE TABLE!
──────────────────────────────────────────────────
Table lock  → entire table ❌ avoid
Page lock   → group of rows ⚠️ rare
Row lock    → specific row only ✅ use this

IRCTC has 500 seats:
  User A books 34A → LOCKS row 34A only
  User B books 12C → LOCKS row 12C only
  Both proceed simultaneously ✅

  User C also wants 34A:
  → Locked by User A → WAITS ⏳
  → User A commits → User C reads
  → Seat booked → "unavailable"

500 users can book 500 different seats
simultaneously ✅
Only conflict when 2 users want SAME seat
──────────────────────────────────────────────────
```

---

## SKIP LOCKED — Production Pattern

```sql
-- For high concurrency scenarios
SELECT * FROM inventory
WHERE product_id = 'iphone'
FOR UPDATE SKIP LOCKED;

-- If row is locked → SKIP immediately
-- Don't wait
-- Return "out of stock" instantly
-- Better user experience ✅
```

---

## Optimistic Locking — Implementation

```sql
-- Table has version column
CREATE TABLE seats (
  seat_id VARCHAR,
  status  VARCHAR,
  version INT  -- version number
);

-- Read with version
SELECT seat_id, status, version
FROM seats WHERE seat_id = '34A';
-- Returns: version = 5

-- Update only if version matches
UPDATE seats
SET status = 'booked', version = version + 1
WHERE seat_id = '34A'
AND version = 5;  -- check version

-- If 0 rows updated → conflict → retry
```

```
WHAT HAPPENS:
──────────────────────────────────────────────────
User A reads → version = 5
User B reads → version = 5

User A updates → version = 5 ✅ → becomes 6
User B updates → version = 5 ❌ → now 6!
→ 0 rows updated → CONFLICT → retry
──────────────────────────────────────────────────
```

---

## Reserve → Confirm Pattern

```
FLASH SALE — DON'T HOLD LOCK DURING PAYMENT
──────────────────────────────────────────────────
WRONG: Hold lock during payment processing
  → Payment takes 2-3 seconds
  → Lock held 2-3 seconds
  → 9,999 users waiting 😱

RIGHT: Reserve → Release → Confirm
  Step 1: Lock inventory row
  Step 2: Check stock > 0
  Step 3: Decrement stock → status = "reserved"
  Step 4: COMMIT → release lock immediately
  Step 5: Process payment (no lock held)

  Payment SUCCESS → status = "confirmed" ✅
  Payment FAILS   → run compensating transaction
                  → stock += 1
                  → status = "available"
──────────────────────────────────────────────────
```

---

## Comparison Table

```
              Pessimistic      Optimistic
              ───────────      ──────────
How           Lock before      Check version
              read             on write
Conflicts     Prevented        Detected after
When conflict Waiting          Retry
Performance   Slower           Faster
Deadlock risk YES ❌           NO ✅
Best for      High contention  Low contention
              IRCTC, inventory User profiles,
              bank accounts    settings
```

---

## Decision Framework

```
ASK: "Is this data shared between
      multiple users simultaneously?"
──────────────────────────────────────────────────
No → No locking needed
     OR optimistic at most
     (for multi-device scenarios)

Yes + High contention →
     Pessimistic locking
     (Flash sale, seat booking)

Yes + Low contention →
     Optimistic locking
     (User profiles, settings)

EXAMPLES:
  User's own profile    → No/Optimistic
  Shared seat (IRCTC)   → Pessimistic
  Shared inventory      → Pessimistic
  Flash sale (1 item)   → Pessimistic + SKIP LOCKED
──────────────────────────────────────────────────
```

---

## Tradeoff

```
PESSIMISTIC LOCKING
──────────────────────────────────────────────────
✅ Guaranteed no conflicts
✅ Simple to reason about
✅ Right for high contention
❌ Slower — threads wait
❌ Deadlock risk
❌ Reduces throughput

OPTIMISTIC LOCKING
──────────────────────────────────────────────────
✅ No waiting — faster
✅ Higher throughput
✅ No deadlock risk
❌ Retry logic needed
❌ Bad under high contention
   (many retries = retry storm)
```

---

## Interview Framing

> *"For seat booking I'd use pessimistic locking — SELECT FOR UPDATE locks the row before reading. High contention means optimistic locking would cause too many retries. For user profile updates I'd use optimistic locking with a version column — conflicts are rare so we avoid the overhead of locking. The tradeoff is pessimistic locking reduces throughput due to waiting, while optimistic locking requires retry logic but scales better under low contention."*

---
---

# CHUNK 15 — Distributed Locks

## Core Concept

```
SIMPLE VERSION — THE NOTICE BOARD
────────────────────────────────────────────────────
You have ONE job to run (send emails)
20 servers want to run it

Can't use server memory:
  Server 1 memory → Server 2 can't see ❌
  Server 5 memory → Server 12 can't see ❌

Need something ALL servers see → REDIS ✅

Redis = shared notice board:
  Server 1: "I AM RUNNING THE JOB" ✅
  Server 2: sees notice → goes away ✅
  Server 3-20: sees notice → go away ✅

Only ONE server runs job ✅
────────────────────────────────────────────────────

KEY INSIGHT:
  One server → in-memory lock fine
  Multiple servers → need Redis
  Redis = shared communication layer
  between all servers
────────────────────────────────────────────────────
```

---

## Redis SETNX — How It Works

```
SETNX = SET if Not eXists
────────────────────────────────────────────────────
SET lock:job "server1" NX EX 30

NX  = only set if key doesn't exist
EX  = expire after 30 seconds (safety)

Server 1:
  SETNX lock:job "server1" NX EX 30
  → Key doesn't exist → SET ✅
  → Returns 1 → got the lock!
  → Run the job
  → DEL lock:job (release when done)

Server 2:
  SETNX lock:job "server2" NX EX 30
  → Key already exists → SKIP ❌
  → Returns 0 → lock taken!
  → Back off

────────────────────────────────────────────────────
WHY EXPIRY IS CRITICAL:
  Server 1 acquires lock
  Server 1 CRASHES 😱
  Without expiry → lock stuck forever
  With EX 30 → auto-deletes after 30 sec
  Server 2 can acquire ✅
────────────────────────────────────────────────────
```

---

## Complete Flow Diagram

```
DISTRIBUTED LOCK WITH REDIS
──────────────────────────────────────────────────

WITHOUT DISTRIBUTED LOCK:
  Server 1  → Local memory (no lock info)
  Server 23 → Local memory (no lock info)
  Both process same resource 😱

WITH DISTRIBUTED LOCK:
  Server 1  → Redis: SETNX → ✅ GOT LOCK
  Server 23 → Redis: SETNX → ❌ WAIT

  ┌─────────┐     ┌─────────┐     ┌─────────┐
  │Server 1 │     │Server 23│     │Server 50│
  └────┬────┘     └────┬────┘     └────┬────┘
       │               │               │
       └───────────────┼───────────────┘
                       │
                       ▼
                  ┌─────────┐
                  │  REDIS  │
                  │lock:job │
                  │=server1 │
                  └─────────┘

  All servers check SAME Redis ✅
  Only ONE gets the lock ✅
──────────────────────────────────────────────────
```

---

## Idempotency Key — Bonus Concept

```
CLIENT GENERATES UNIQUE KEY
────────────────────────────────────────────────────
User clicks Pay → App generates:
idempotency-key = "550e8400-e29b-41d4"

Request 1 → Server 5:  carries "550e8400"
Request 2 → Server 18: carries "550e8400"
→ Same key! Lock works ✅

WHY CLIENT GENERATES (not server):
  Server generates → different servers
  generate different keys → lock fails ❌
  Client generates → same key always ✅

TWO LAYERS OF PROTECTION:
  Layer 1: Distributed lock
           (simultaneous requests)
  Layer 2: DB idempotency check
           (sequential requests)
           → Check if already processed
           → Return same result
────────────────────────────────────────────────────
```

---

## DB Lock vs Distributed Lock

```
WHEN TO USE WHICH
──────────────────────────────────────────────────
DB Lock (SELECT FOR UPDATE):
  ✅ Data is in the DB
  ✅ Simple, built-in
  ✅ Automatic rollback
  Use for: Booking seats, inventory,
           any DB row contention

Distributed Lock (Redis):
  ✅ Works across services
  ✅ Not tied to DB
  ✅ Fast (Redis in-memory)
  Use for: Cross-service operations,
           preventing duplicate jobs,
           cron job deduplication,
           rate limiting

SIMPLE RULE:
  Small job, must run ONCE → Distributed lock
  Large job, split work → Kafka/MapReduce
  DB rows → SELECT FOR UPDATE
──────────────────────────────────────────────────
```

---

## Tradeoff

```
DISTRIBUTED LOCK
──────────────────────────────────────────────────
✅ Works across multiple servers
✅ Simple — Redis SETNX
✅ Auto-expires on crash
✅ Fast (Redis in-memory)

❌ Redis = new dependency
❌ Expiry tuning is tricky
   Too short → lock expires mid-work
   Too long  → stuck if crash

EXPIRY RULE OF THUMB:
  Set to 3× expected operation time
  Payment takes 3 sec → EX 9
──────────────────────────────────────────────────
```

---

## Interview Framing

> *"For operations spanning multiple servers I use distributed locks via Redis SETNX. Each operation gets a unique lock key — only one server acquires it. I always set expiry to 3× expected processing time as safety net for crashes. In-memory locks don't work across servers because each server's memory is isolated — Redis acts as shared communication layer between all servers."*

---
---

# CHUNK 16 — Deadlocks

## Core Concept

```
SIMPLE VERSION — THE CORRIDOR
────────────────────────────────────────────────────
Two people in narrow corridor:
Person A walking right →
Person B walking left  ←

They meet in middle:
A says: "You move first"
B says: "No YOU move first"
Both waiting for other → nobody moves → forever 😄

In software:
Thread A holds Lock 1, wants Lock 2 → WAITING
Thread B holds Lock 2, wants Lock 1 → WAITING
Both stuck forever 😱
────────────────────────────────────────────────────
```

---

## How Deadlock Happens

```
REAL EXAMPLE — MONEY TRANSFER
────────────────────────────────────────────────────
User A sends ₹500 to User B
User B sends ₹500 to User A
Both simultaneously:

Transaction 1 (A→B):
  Step 1: Lock User A's account ✅
  Step 2: Lock User B's account → WAITING ⏳

Transaction 2 (B→A):
  Step 1: Lock User B's account ✅
  Step 2: Lock User A's account → WAITING ⏳

T1 waiting for B's lock (held by T2)
T2 waiting for A's lock (held by T1)
Circular wait → DEADLOCK 😱
────────────────────────────────────────────────────
```

---

## Three Fixes

```
FIX 1 — LOCK ORDERING ✅ (Best - prevent)
────────────────────────────────────────────────────
Always acquire locks in same predefined order

By user ID (smaller first):
  Transaction 1 (A→B): Lock A(123) → Lock B(456)
  Transaction 2 (B→A): Lock A(123) → Lock B(456)
  Same order → no circular wait ✅

Java implementation:
  int firstAccount  = id1 < id2 ? id1 : id2;
  int secondAccount = id1 < id2 ? id2 : id1;
  lock(firstAccount);
  lock(secondAccount);

────────────────────────────────────────────────────
FIX 2 — LOCK TIMEOUT (backup)
────────────────────────────────────────────────────
Don't wait forever for a lock

Transaction 1 waiting for Lock B
→ After 5 seconds → GIVE UP
→ Release Lock A
→ Retry after random delay

// PostgreSQL:
SET lock_timeout = '5s';

✅ Prevents infinite deadlock
❌ Transaction fails + retries
❌ Adds latency

────────────────────────────────────────────────────
FIX 3 — DEADLOCK DETECTION (automatic)
────────────────────────────────────────────────────
Let deadlock happen
PostgreSQL detects automatically
Kills one transaction (victim)
Other proceeds

// Error you'll see:
ERROR: deadlock detected
DETAIL: Process 1234 waits for
ShareLock on transaction 5678

Application handles:
→ Catch deadlock error
→ Retry transaction ✅

✅ Automatic — DB handles it
❌ One transaction always fails
❌ Retry logic needed
────────────────────────────────────────────────────
```

---

## Diagram

```
DEADLOCK — COMPLETE PICTURE
──────────────────────────────────────────────────

HOW IT HAPPENS:
  T1 holds A, wants B ──────┐
                             ↓ CIRCULAR WAIT 😱
  T2 holds B, wants A ──────┘

FIX 1 — Lock Ordering:
  Both always lock A before B
  T1: A → B ✅
  T2: A → WAIT for T1 → A → B ✅
  No circular wait ✅

FIX 2 — Timeout:
  T1 waiting for B
  After 5 sec → gives up → releases A
  T2 gets A → B ✅, T1 retries ✅

FIX 3 — Detection:
  DB detects circular wait
  Kills T1 (victim)
  T2 proceeds ✅, T1 retries ✅

BEST PRACTICE:
  Fix 1 in code (prevent)
  + Fix 3 by DB (auto-recover)
  + Fix 2 as last resort
──────────────────────────────────────────────────
```

---

## Java Lock Types Reference

```
JAVA LOCKS
──────────────────────────────────────────────────
Pessimistic (thread level):
  synchronized(this) { ... }
  ReentrantLock lock = new ReentrantLock();

Optimistic (thread level):
  AtomicInteger stock = new AtomicInteger(100);
  stock.compareAndSet(expected, update);

Distributed (server level):
  Redis SETNX (across multiple servers)

DB Row lock:
  SELECT FOR UPDATE (across all app servers)

RULE:
  Single server → Java synchronized
  Multiple servers → Redis distributed lock
  DB rows → SELECT FOR UPDATE
──────────────────────────────────────────────────
```

---

## Tradeoff

```
FIX 1 — LOCK ORDERING
──────────────────────────────────────────────────
✅ Prevents deadlock completely
✅ No performance impact
❌ Developers must follow order
❌ Easy to forget in complex systems

FIX 2 — TIMEOUT
──────────────────────────────────────────────────
✅ Always eventually resolves
❌ Failed transactions must retry
❌ Adds latency

FIX 3 — DETECTION
──────────────────────────────────────────────────
✅ Automatic — DB handles it
❌ One transaction always fails
❌ Retry logic needed
──────────────────────────────────────────────────
```

---

## Interview Framing

> *"Deadlocks happen when two transactions wait for each other's locks in a circular pattern. I prevent them primarily through lock ordering — always acquiring locks in the same predefined order, like smaller user ID first. PostgreSQL automatically detects deadlocks and kills one transaction, so the application needs retry logic. Lock timeouts are a backup — if waiting more than 5 seconds, give up and retry."*

---
---

# 🎯 Complete Technical Terms — Topic 4

| Concept | Simple Word | Technical Term |
|---|---|---|
| Two ops read before either writes | "both read same data" | Race Condition |
| Reading uncommitted data | "reading mid-change" | Dirty Read |
| Lock before reading | "lock first" | Pessimistic Locking |
| Check version on write | "verify before save" | Optimistic Locking |
| Lock specific DB row | "row lock" | SELECT FOR UPDATE |
| Skip locked rows | "skip busy rows" | SKIP LOCKED |
| Row version for conflict detection | "version number" | Optimistic Version Column |
| Lock across multiple servers | "shared lock" | Distributed Lock |
| Redis lock command | "set if not exists" | SETNX |
| Auto-delete lock on crash | "lock expiry" | TTL on lock |
| Unique ID per operation | "request ID" | Idempotency Key |
| Same op twice = same result | "safe to retry" | Idempotency |
| Two transactions waiting for each other | "stuck forever" | Deadlock |
| Always lock in same sequence | "consistent order" | Lock Ordering |
| Give up waiting after time | "stop waiting" | Lock Timeout |
| DB finds and resolves deadlock | "auto-detect" | Deadlock Detection |
| Manual undo of committed transaction | "reverse action" | Compensating Transaction |
| Ops run simultaneously | "at same time" | Concurrency |
| One op at a time | "queue up" | Serial Execution |

---

# 📝 Things to Always Say in Interviews

1. **For high contention:** *"Pessimistic locking with SELECT FOR UPDATE — conflicts are too frequent for optimistic to work"*
2. **For low contention:** *"Optimistic locking with version column — conflicts are rare, avoid locking overhead"*
3. **For multiple servers:** *"Distributed lock via Redis SETNX — server memory is isolated, Redis is the shared communication layer"*
4. **For deadlock prevention:** *"Lock ordering — always acquire locks in same predefined order, smaller ID first"*
5. **For payment duplication:** *"Idempotency key generated by client — same key means same operation, prevents double charging"*
6. **For flash sale:** *"Pessimistic lock + SKIP LOCKED — failed users see out of stock immediately, no waiting"*
7. **For lock expiry:** *"Set to 3× expected operation time — safety net for crashes"*

---

# 🚫 Things to Never Say in Interviews

1. ~~"Use pessimistic locking for everything"~~ → Kills throughput at scale
2. ~~"Lock the whole table"~~ → Table locks = never in production
3. ~~"Store lock in server memory"~~ → Invisible to other servers
4. ~~"Use SAGA for dirty reads"~~ → SAGA is for cross-service, use isolation level
5. ~~"Optimistic locking for flash sales"~~ → Retry storm at high contention
6. ~~"Hold lock during payment processing"~~ → Reserve → Release → Confirm instead

---

# 🔥 Drill Session — Questions & Answers

## Q1 — Zepto Warehouse Race Condition

**Question:**
Two warehouse workers scan same item simultaneously. Worker A ships 3, Worker B ships 5. Stock was 10. What's wrong?

**Answer:**
```
RACE CONDITION
──────────────────────────────────────────────────
Worker A reads stock = 10 → ships 3 → writes 7
Worker B reads stock = 10 → ships 5 → writes 5

System shows: 5 items remaining
Actual stock: 10 - 3 - 5 = 2 items remaining
System thinks 3 extra items exist → overselling! 😱

Fix: Pessimistic locking on inventory row
  SELECT stock FOR UPDATE
  → Only one worker at a time
  → Correct stock always
──────────────────────────────────────────────────
```

---

## Q2 — PhonePe Dirty Read

**Question:**
Service reads transaction status "PROCESSING" but another thread rolled it back to "PENDING". What happens?

**Answer:**
```
DIRTY READ
──────────────────────────────────────────────────
Problem: Read uncommitted data
Thread reads "PROCESSING" (not committed)
Acts on it → sends money to merchant
Original transaction rolls back → "PENDING"
Money sent but marked failed → money loss 😱

Fix: Use READ COMMITTED isolation level
@Transactional(isolation = Isolation.READ_COMMITTED)
→ Only reads committed data
→ Never reads mid-transaction data

SAGA is for cross-service transactions
Isolation level fixes dirty reads in same DB
──────────────────────────────────────────────────
```

---

## Q3 — Swiggy Cron Job

**Question:**
10 servers, weekly payout cron job at 9am Monday. What goes wrong? Which lock? Where does lock live?

**Answer:**
```
WITHOUT PROTECTION:
  10 servers × 1 payout job
  = 10 payouts per delivery partner 😱
  Swiggy loses crores!

LOCK TYPE: Distributed Lock
LIVES IN: Redis (not server memory)

WHY NOT SERVER MEMORY:
  Server 1 memory invisible to Server 2-10
  Each server thinks no lock → all run 😱

REDIS SOLUTION:
  SET lock:weekly-payout "server1" NX EX 3600
  → Only one server gets it ✅
  → EX 3600 = 1 hour expiry (job takes ~30 min)
  → Other servers see lock → skip job ✅
──────────────────────────────────────────────────
```

---

## Q4 — IRCTC Deadlock

**Question:**
Op1 locks order then driver. Op2 locks driver then order. Deadlock?

**Answer:**
```
YES — DEADLOCK
──────────────────────────────────────────────────
Op1: locks order #123 → wants driver #456
Op2: locks driver #456 → wants order #123
→ Circular wait 😱

FIX — Lock Ordering:
  Always lock ORDER first, then DRIVER
  Op1: order → driver ✅
  Op2: order → driver ✅ (waits for Op1)
  No circular wait ✅

Note: Two users can't share a wallet
→ Wallet lock per user = no conflict
→ Deadlock only happens on truly shared resources
──────────────────────────────────────────────────
```

---

## Q5 — Razorpay Pessimistic Lock at Scale

**Question:**
Razorpay 50,000 payments/sec. Engineer says use pessimistic locking on every payment. What's wrong?

**Answer:**
```
WRONG ASSUMPTION
──────────────────────────────────────────────────
Each payment = unique row
No contention between different payments
Pessimistic lock = unnecessary overhead

REAL PROBLEM:
  Merchant balance row IS shared
  All payments to same merchant → same row
  Pessimistic lock here = bottleneck at 50K QPS

WHAT RAZORPAY ACTUALLY DOES:
  1. Idempotency keys → prevent duplicate processing
  2. Distributed lock on payment ID only
     → Not on balance rows
  3. Append-only payment log
     → No row contention
  4. Update merchant balance async via Kafka
     → Background aggregation
     → No locking needed

Result: 50K QPS with no locking overhead ✅
──────────────────────────────────────────────────
```

---

## 📊 Drill Scores

| Question | Score |
|---|---|
| Q1 — Zepto race condition | 4/5 |
| Q2 — PhonePe dirty read | 4.5/5 |
| Q3 — Swiggy cron job | 5/5 |
| Q4 — IRCTC deadlock | 5/5 (bonus!) |
| Q5 — Razorpay pessimistic lock | Taught |
| **Total** | **18.5/20 = 92%** |

---

*Topic 4 Complete ✅ | Next: Phase 1 Review → Then Phase 2*