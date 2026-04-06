# Isolation Levels — Complete Revision Guide
### Chunk 18 | Phase 2 — Databases
> Target: Razorpay, PhonePe, Flipkart, Zerodha

---

## Memory Trick — Order of Levels

```
UC → RC → RR → S

"Uncommitted people never Commit,
 then they Repeat themselves,
 until finally they Serialize everything"

Read Uncommitted → Read Committed → Repeatable Read → Serializable
     weakest                                              strongest
```

---

## The 3 Problems First

Before isolation levels make sense, understand what they're solving.

---

### Problem 1 — Dirty Read

```
Transaction A starts
Transaction A writes row (NOT committed yet)
Transaction B reads that row           ← reads uncommitted data
Transaction A rolls back
Transaction B is now holding data that never existed 😬
```

**Real example:**
```
Razorpay:
→ Payment transaction starts, debits ₹500 (not committed)
→ Dashboard reads new balance
→ Payment rolls back
→ Dashboard showed wrong balance to merchant 😬
```

---

### Problem 2 — Non-Repeatable Read

```
Transaction A reads row X → value = 100
Transaction B updates row X → value = 200, commits
Transaction A reads row X again → value = 200

Same row, same transaction, different values 😬
```

**Real example:**
```
Zerodha trade execution:
→ Read stock price = ₹1000 at start of transaction
→ Another user's trade updates price to ₹1050
→ Read stock price again = ₹1050
→ Calculation uses two different prices 😬
```

---

### Problem 3 — Phantom Read

```
Transaction A runs query → returns 10 rows
Transaction B inserts 2 new rows that match query, commits
Transaction A runs same query → returns 12 rows

Same query, same transaction, different row count 😬
```

**Real example:**
```
BookMyShow:
→ Query available seats → 5 seats available
→ Another user books seat 4A
→ Query available seats again → 4 seats available
→ Seat 4A shown as available to both users simultaneously 😬
→ Double booking 😬
```

---

## The 4 Isolation Levels

### What each prevents:

```
                    Dirty    Non-Repeatable   Phantom    Performance
                    Read     Read             Read
─────────────────────────────────────────────────────────────────────
Read Uncommitted  │  ❌         ❌               ❌        Fastest
Read Committed    │  ✅         ❌               ❌        Fast
Repeatable Read   │  ✅         ✅               ❌        Medium
Serializable      │  ✅         ✅               ✅        Slowest
```

---

### Read Uncommitted — Level 1 (weakest)

```
What it does:
→ Transactions can read each other's uncommitted changes
→ Prevents nothing

When to use:
→ Almost never in production
→ Only when approximate data is acceptable
→ e.g. rough analytics where staleness doesn't matter

Real world:
→ Counting approximate active users on a dashboard
→ If count is off by a few = acceptable
```

---

### Read Committed — Level 2

```
What it does:
→ Only reads committed data
→ Prevents dirty reads ✅
→ But same row can change between two reads in same transaction

Default in:
→ PostgreSQL ✅
→ Oracle ✅

When to use:
→ Most read-heavy systems where one-time reads are sufficient
→ When slight staleness between reads is acceptable
```

**Real examples:**
```
✅ Razorpay dashboard
   → Merchant checks settlement total
   → Single read per page load
   → No need for consistency across multiple reads

✅ Zerodha price feed
   → Displaying live stock prices to users
   → High read volume
   → Price slightly stale = minor UX issue, not catastrophic

✅ Swiggy order status page
   → User refreshes to see order status
   → Single read, no cross-read consistency needed
```

---

### Repeatable Read — Level 3

```
What it does:
→ Snapshot of data taken at transaction start
→ Same row read twice = same value guaranteed ✅
→ Prevents dirty reads + non-repeatable reads ✅
→ But new rows inserted by others still visible ❌ (phantom reads possible)

Default in:
→ MySQL (InnoDB) ✅

When to use:
→ When same data must stay consistent throughout a transaction
→ Multi-step calculations using same rows
```

**Real examples:**
```
✅ Zerodha trade execution
   → Read stock price at transaction start
   → Price must stay same throughout entire trade calculation
   → Cannot have price change mid-transaction 😬

✅ Flipkart order total calculation
   → Read product prices at checkout start
   → Prices must stay consistent while order is being placed
   → Price change mid-checkout = wrong total charged 😬

✅ Razorpay settlement calculation
   → Summing transactions for a merchant
   → Values must stay consistent across multiple reads in same batch
```

---

### Serializable — Level 4 (strongest)

```
What it does:
→ Transactions execute as if they were serial (one after another)
→ Prevents ALL anomalies ✅
→ Dirty reads ✅ Non-repeatable reads ✅ Phantom reads ✅

How it works:
→ Range locks on queries (not just row locks)
→ New inserts that match your WHERE clause are blocked
→ Until your transaction commits

Cost:
→ Highest lock contention
→ Slowest throughput
→ Most deadlock risk
```

**Real examples:**
```
✅ BookMyShow seat booking
   → Two users query available seats simultaneously
   → Without Serializable → both see seat 4A available → double booking 😬
   → With Serializable → second user blocked until first commits ✅

✅ Bank account transfer
   → Read balance → debit → credit
   → Cannot afford ANY anomaly
   → Financial data integrity non-negotiable

✅ Airline ticket booking
   → Last seat scenario
   → Two users cannot both book the last seat
```

---

## Real System Mapping — Quick Reference

| System | Isolation Level | Key Reason |
|---|---|---|
| Razorpay dashboard | Read Committed | Single read, staleness acceptable |
| Zerodha price feed | Read Committed | High volume reads, minor staleness ok |
| Zerodha trade execution | Repeatable Read | Same price across entire transaction |
| Flipkart checkout | Repeatable Read | Prices consistent during order placement |
| BookMyShow seat booking | Serializable | Phantom read = double booking 😬 |
| Bank transfer | Serializable | Zero tolerance for any anomaly |
| Swiggy order status | Read Committed | Single read per request |
| IRCTC ticket booking | Serializable | Last ticket scenario = phantom read risk |

---

## The Key Interview Insight

```
Isolation level choice = driven by CONSEQUENCE of anomaly
                         not just data type

Ask yourself:
"What is the worst case if this anomaly happens?"

Catastrophic (double booking, double charge, wrong trade)
→ Serializable

Calculation errors (wrong total, wrong price mid-txn)
→ Repeatable Read

Minor UX issue (stale count, stale price display)
→ Read Committed

Approximate data acceptable (rough analytics)
→ Read Uncommitted (rare)
```

---

## Defaults — Know This Cold

```
PostgreSQL default → Read Committed
MySQL default      → Repeatable Read
Oracle default     → Read Committed
SQL Server default → Read Committed
```

---

## The Tradeoff Formula

```
Higher isolation → safer data → more locks → lower throughput
Lower isolation  → faster    → fewer locks → anomaly risk

Never use Serializable everywhere:
→ Lock contention kills throughput at scale
→ Razorpay doing 10M transactions/day cannot afford 
  full serialization on every dashboard read

Pick the LOWEST level that prevents 
the anomalies your system cannot tolerate.
```

---

## Interview Framing

**For payment systems:**
> *"I'd use Read Committed for the dashboard reads — merchants are doing single reads per request, so cross-read consistency isn't needed. For the actual payment transaction I'd use Serializable — a partial debit with no credit is catastrophic and unacceptable."*

**For booking systems:**
> *"Serializable is non-negotiable for seat booking. A phantom read means two users simultaneously see the same seat as available — resulting in a double booking. The performance cost is acceptable given the low write volume on any single seat."*

**For trade execution:**
> *"Repeatable Read for trade execution — the stock price read at transaction start must stay consistent through the entire calculation. If the price changes mid-transaction, the trade executes at the wrong price."*

---

## Drill Questions — Use These for Self-Revision

```
Q1: Name the 4 isolation levels weakest to strongest.

Q2: Which level prevents dirty reads but not phantom reads?

Q3: BookMyShow seat booking — which level and why?

Q4: Zerodha price feed — which level and why?

Q5: What's the difference between non-repeatable read 
    and phantom read?

Q6: What are the defaults for PostgreSQL and MySQL?

Q7: Why not use Serializable everywhere?

Q8: Razorpay has two operations:
    a) Dashboard showing merchant settlements
    b) Actual payment transaction
    Different isolation levels for each? Why?
```

---

## One-Line Summaries — Memorize These

```
Read Uncommitted → reads anything, even garbage
Read Committed   → only reads real, committed data
Repeatable Read  → what I read stays same all transaction
Serializable     → I'm the only one in the room
```

---

*Chunk 18 complete | Score: 90% average*
*Next: Chunk 19 — Indexing | Chunk 21 — Redis internals*