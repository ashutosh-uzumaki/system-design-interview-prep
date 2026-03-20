# Topic 1 — How to Think About Scale
## Complete Notes | Concepts + Interview Q&A + Technical Terms

---

## 📊 Topic 1 Stats
- **Chunks covered:** 1, 2, 3
- **Drill score:** 18/25
- **Chunk scores:** 7.5/9
- **Status:** ✅ Complete

---

---

# CHUNK 1 — Reliability vs Scalability vs Maintainability

## Core Concept

Every system you design has 3 goals. Always.

```
SYSTEM GOALS
─────────────────────────────────────────────────────

RELIABILITY
  Simple:     System works correctly even when
              things go wrong
  Technical:  System continues to function correctly
              despite hardware failures, software bugs,
              or unexpected user behaviour
  Key word:   CORRECTLY — not just "up" but giving
              right answers

SCALABILITY
  Simple:     System handles more users/data without
              breaking or redesigning
  Technical:  System can handle growing load —
              measured in QPS, concurrent users,
              or data volume — without redesign
  Key word:   GROWING LOAD

MAINTAINABILITY
  Simple:     New engineers can understand, change,
              and operate the system safely
  Technical:  Three sub-properties:
              → Operability  — easy to run in production
              → Simplicity   — easy to understand
              → Evolvability — easy to change safely
  Key word:   CHANGE SAFELY

─────────────────────────────────────────────────────
```

---

## How to Identify Which Goal Matters Most

```
ASK THIS                        TELLS YOU
──────────────────────────────────────────────────
How many users?             →   Scalability concern
What if it gives wrong           Reliability concern
answer?                     →   (higher cost of wrong
                                 answer = higher priority)
How often does code change?      Maintainability concern
How many engineers?         →

──────────────────────────────────────────────────

EXAMPLES
─────────────────────────────────────────────────
Hospital records    →   Reliability ⭐⭐⭐⭐⭐
(1 user, life           Scalability ⭐
at stake)               Maintainability ⭐⭐

IPL live score      →   Reliability ⭐⭐⭐
(10M users)             Scalability ⭐⭐⭐⭐⭐
                        Maintainability ⭐⭐

Internal HR tool    →   Reliability ⭐⭐
(50 users, low          Scalability ⭐
stakes)                 Maintainability ⭐⭐⭐⭐⭐

Payments/Fintech    →   Reliability ⭐⭐⭐⭐⭐
                        Scalability ⭐⭐⭐⭐
                        Maintainability ⭐⭐⭐
─────────────────────────────────────────────────
```

---

## Technical Terms to Use in Interviews

| Plain English | Technical Term |
|---|---|
| System works even when things go wrong | Reliability |
| Handles more users without breaking | Scalability |
| Easy to change and operate | Maintainability |
| Easy to run in production | Operability |
| Easy to understand | Simplicity |
| Easy to change safely | Evolvability |
| How much work system does per second | Throughput / QPS |
| A bug introduced by a new engineer | Evolvability failure |

---

## Interview Questions + How to Answer

### Q: "Which is more important — reliability or scalability?"

> ❌ Wrong answer: "Both are equally important."
>
> ✅ Right answer: "It depends on the system.
> For payments, reliability is non-negotiable —
> a wrong answer costs money and trust.
> For a social feed, scalability dominates —
> showing slightly stale data is acceptable
> as long as the system stays up."

---

### Q: "Your app went down for 2 minutes. What failed?"

> Connect to the right goal:
> - Went down = Reliability failure (SPOF or no redundancy)
> - Went down under load = Scalability failure
> - Went down after a code change = Maintainability failure
>   (poor testing, no rollback strategy)

---

### Q: "A new engineer broke production. What should you have done?"

> "This is a Maintainability failure — specifically
> evolvability. We needed better automated test
> coverage, code review processes, and deployment
> safeguards like canary releases so a single
> engineer's change can't take down production."

---

### Q: "Design a system for X — where do you start?"

> "Before designing, I want to identify which
> goal matters most for this system.
> Is this reliability-critical like payments?
> Or scalability-critical like a social feed?
> Or a small internal tool where maintainability
> is the focus? The answer changes my entire
> architecture."

---

## Common Mistakes in Interviews

```
MISTAKE 1: Saying all three matter equally
  → Shows you can't prioritise
  → Fix: Always rank them for the specific system

MISTAKE 2: Confusing "system is down" with
           maintainability problem
  → System down = reliability failure
  → Hard to fix after down = maintainability failure

MISTAKE 3: Ignoring maintainability completely
  → Most candidates only talk reliability + scalability
  → Mentioning maintainability signals senior thinking

MISTAKE 4: Not connecting to business impact
  → Don't just say "reliability matters"
  → Say "reliability matters because wrong payment
    data = financial liability + user churn"
```

---
---

# CHUNK 2 — Latency vs Throughput

## Core Concept

```
LATENCY vs THROUGHPUT
─────────────────────────────────────────────────────

LATENCY
  Simple:     Time for ONE request to get a response
  Technical:  Time taken to complete a single request
              Measured in milliseconds (ms) or
              microseconds (µs)
  Also called: Response time

THROUGHPUT
  Simple:     How many requests the system handles
              per second
  Technical:  Number of requests processed per unit
              of time
  Measured in: QPS (queries per second)
               TPS (transactions per second)
               RPS (requests per second)

─────────────────────────────────────────────────────
THE HIGHWAY ANALOGY
────────────────────
  Latency   = time for ONE car to travel A → B
  Throughput = cars passing through per hour
─────────────────────────────────────────────────────
```

---

## Percentiles — Must Know for Interviews

```
PERCENTILES
────────────────────────────────────────────────────

p50  = 50% of requests finish within X ms
       (median — what most users experience)

p95  = 95% of requests finish within X ms

p99  = 99% of requests finish within X ms
       ← ALWAYS USE THIS IN INTERVIEWS

p999 = 99.9% of requests finish within X ms
       (tail latency — extreme cases)

────────────────────────────────────────────────────
WHY p99 MATTERS MORE THAN p50
───────────────────────────────
  p50 = 100ms, p99 = 4 seconds

  At Zepto scale (50,000 orders/day):
  1% of 50,000 = 500 users/day
  waiting 4+ seconds at checkout

  500 users × ₹400 avg order = ₹2,00,000/day
  = ₹7.3 crore/year in potential lost revenue

  p50 hides the pain of slow users
  p99 reveals the real user experience

────────────────────────────────────────────────────
INTERVIEW RULE:
  Never mention average latency
  Always talk about p99
  "p50 tells you how most users feel.
   p99 tells you how your worst-served
   users feel — and they're the ones who churn."
────────────────────────────────────────────────────
```

---

## The Tension Between Latency and Throughput

```
OPTIMISING ONE OFTEN HURTS THE OTHER
──────────────────────────────────────────────────────

Low Latency goal          High Throughput goal
────────────────          ────────────────────
Process each request      Process as many requests
as fast as possible       as possible per second

Sometimes means:          Sometimes means:
→ Do less work per        → Batch requests together
  request                 → Queue and process later
→ Serve from cache        → Accept higher per-request
→ Avoid DB round trips      latency

──────────────────────────────────────────────────────
BATCHING — Classic Tradeoff Example
─────────────────────────────────────
Without batching:
  1 request → 1 DB write
  Low latency ✅ Low throughput ❌

With batching:
  100 requests → 1 DB write
  Higher latency ❌ High throughput ✅

USE BATCHING WHEN:          DON'T USE BATCHING WHEN:
✅ Log ingestion            ❌ Payment confirmation
✅ Analytics events         ❌ Order placement
✅ Kafka message writes     ❌ OTP delivery
✅ Email campaigns          ❌ Anything user waits for

RULE: Batch when user is NOT waiting
      Don't batch when user IS waiting
──────────────────────────────────────────────────────
```

---

## Synchronous vs Asynchronous

```
SYNCHRONOUS vs ASYNCHRONOUS
────────────────────────────────────────────────────

SYNCHRONOUS
  User makes request → waits → gets response
  Low latency needed
  User is staring at loading screen
  Example: Payment confirmation, OTP, search results

ASYNCHRONOUS
  User makes request → gets immediate ack
  Processing happens in background
  User gets notified when done
  High latency acceptable
  Example: Instagram Year in Review,
           report generation, video processing

────────────────────────────────────────────────────
INTERVIEW QUESTION:
  "Is the user actively waiting for this result?"

  Yes → optimise for LATENCY → use SYNC
  No  → optimise for THROUGHPUT → use ASYNC
────────────────────────────────────────────────────
```

---

## Technical Terms to Use in Interviews

| Plain English | Technical Term |
|---|---|
| Time for one request | Latency / Response time |
| Requests per second | Throughput / QPS / TPS |
| Worst 1% of users' experience | p99 / Tail latency |
| Median user experience | p50 |
| Grouping multiple ops into one | Batching |
| User waits for response | Synchronous processing |
| Process in background, notify later | Asynchronous processing |
| User not blocked while waiting | Non-blocking operation |

---

## Interview Questions + How to Answer

### Q: "How would you measure system performance?"

> "I'd look at p99 latency and QPS together.
> p99 latency tells me how my worst-served users
> feel — if p99 is high, I have a tail latency
> problem causing churn. QPS tells me how much
> load the system handles. I'd never rely on
> average latency alone — it hides outliers."

---

### Q: "We're processing 1M events per day but users say it's slow. What do you investigate?"

> "These are two different problems.
> 1M events/day is a throughput metric — sounds fine.
> But 'users say it's slow' is a latency complaint.
> I'd check p99 latency first — probably have a
> tail latency problem where most users are fine
> but 1% are experiencing very slow responses.
> I'd look at DB query times, cache hit rates,
> and whether any synchronous operations can
> be moved to async processing."

---

### Q: "Should we use batching for our order processing?"

> "Depends on whether the user is actively waiting.
> For order placement — no. Users expect immediate
> confirmation. Batching would add 500ms+ latency
> which increases cart abandonment.
> For order analytics and reporting — yes.
> Nobody is waiting for analytics data in real-time.
> Batching here increases throughput significantly
> with no user experience impact."

---

## Common Mistakes in Interviews

```
MISTAKE 1: Talking about average latency
  → Always use p99
  → "Average hides the tail"

MISTAKE 2: Confusing latency with throughput
  → High throughput ≠ low latency
  → System can handle 1M req/day (high throughput)
    but each request takes 4 seconds (bad latency)

MISTAKE 3: Not asking "is user waiting?"
  → Before any latency optimisation
  → Ask: is this sync or async?
  → Async = latency doesn't matter

MISTAKE 4: Recommending batching for user-facing flows
  → Batching = latency goes up
  → Never batch payment confirmation,
    OTP, order placement
```

---
---

# CHUNK 3 — SPOF + Fault Tolerance vs High Availability

## Core Concept

```
THREE CONCEPTS
─────────────────────────────────────────────────────

SINGLE POINT OF FAILURE (SPOF)
  Simple:     One component whose failure
              takes down the entire system
  Technical:  Any component in your system
              whose failure causes complete
              system unavailability
  Examples:
    → Single database with no replica
    → Single load balancer
    → Single Kafka broker (RF=1)
    → Single availability zone deployment
    → Single Redis cache instance

HIGH AVAILABILITY (HA)
  Simple:     System stays UP during failures
              May return stale data but responds
  Technical:  System remains operational even
              when components fail
              Measured as uptime percentage
  Key word:   OPERATIONAL

FAULT TOLERANCE (FT)
  Simple:     System stays CORRECT during failures
              Never returns wrong answer
  Technical:  System continues operating correctly
              despite component failures
              Requires consensus + coordination
  Key word:   CORRECTLY

─────────────────────────────────────────────────────
```

---

## Uptime Table — Memorise This

```
UPTIME TABLE
────────────────────────────────────────────────────
99%      → 3.65 days downtime/year   ❌ unacceptable
99.9%    → 8.7 hours downtime/year   ⚠️  okay internal
99.99%   → 52 minutes downtime/year  ✅ most apps
99.999%  → 5 minutes downtime/year   🔥 payments/banking
────────────────────────────────────────────────────
"Five nines" = 99.999% = what payments systems target
```

---

## HA vs Fault Tolerance Comparison

```
HIGH AVAILABILITY vs FAULT TOLERANCE
──────────────────────────────────────────────────────

                HA                    FT
                ──                    ──
Goal            Stay UP               Stay CORRECT
May return      Stale/approximate     Always correct
                data                  data
Cost            Lower                 Higher
Complexity      Medium                High
Mechanism       Redundancy +          Consensus +
                failover              coordination
Use for         Social feeds,         Payments,
                search, streaming     bank balances,
                                      leader election
Example DB      Cassandra (AP)        Zookeeper (CP)

──────────────────────────────────────────────────────
PRACTICAL TRUTH:
  Most systems aim for HA + Eventual Consistency
  Pure Fault Tolerance is expensive and slow
  Only use FT where correctness = non-negotiable
──────────────────────────────────────────────────────
```

---

## Active-Passive vs Active-Active

```
ACTIVE-PASSIVE
───────────────
  Mumbai (ACTIVE) ──────► Pune (PASSIVE/IDLE)
  All traffic → Mumbai    Backup only

  Failover time: 2 minutes
  Problems:
  ❌ Downtime during failover
  ❌ In-flight transactions lost
  ❌ Pune resources wasted while idle
  This is HIGH AVAILABILITY

ACTIVE-ACTIVE
──────────────
  Mumbai (ACTIVE) ◄──────► Pune (ACTIVE)
  Both handle traffic simultaneously

  If Mumbai goes down:
  → Pune already handling traffic ✅
  → Near-zero downtime ✅
  Problems:
  ❌ Write conflicts possible
  ❌ Harder to keep data consistent
  ❌ More expensive
  This is closer to FAULT TOLERANCE
```

---

## In-Flight Transactions — Fintech Critical

```
IN-FLIGHT TRANSACTIONS
────────────────────────────────────────────────────
Transactions being processed when crash happens

Without protection:
  Payment in progress → server crashes
  → Money debited but order not confirmed
  → User charged but gets nothing 😱

Fix 1: WAL (Write-Ahead Log)
  → Log transaction BEFORE executing
  → On recovery → replay log
  → No data loss

Fix 2: Idempotency Keys
  → Unique ID per transaction
  → If retried after failover
  → Same ID = no duplicate charge
  → Safe to retry always

Fix 3: 2PC (Two-Phase Commit)
  → Both DCs confirm before commit
  → Covered in Phase 3 in depth

────────────────────────────────────────────────────
FINTECH RULE:
  Fault tolerant WRITES (debit/credit)
  High Available READS (balance check)
  + Read-Your-Writes for balance after transaction
────────────────────────────────────────────────────
```

---

## Read-Your-Writes Consistency

```
READ-YOUR-WRITES
────────────────────────────────────────────────────
After YOU make a write,
YOU must always read your own latest data

Example:
  User pays ₹500
  Checks balance immediately
  Must see updated balance — not stale one

Implementation:
  Route THIS user's reads to PRIMARY
  for 30 seconds after their write
  Other users can read from replica

Why it matters in fintech:
  Without it → user sees old balance → panics
  → Tries to pay again → double charge
────────────────────────────────────────────────────
```

---

## Consensus — Quick Reference

```
CONSENSUS
────────────────────────────────────────────────────
Simple:     Majority must agree before
            anything is final

Technical:  Protocol where multiple nodes agree
            on a single value before committing
            No write is final until quorum confirms

Example:
  3 DB nodes, need 2/3 (quorum)
  Payment ₹500 arrives
  Node A writes ✅
  Node B writes ✅
  Node C crashes ❌
  Quorum reached (2/3) → Payment confirmed ✅
  Node C recovers → syncs from A or B

Used in: Zookeeper, etcd, Kafka leader election
Covered in full: Phase 3 — Topic 13
────────────────────────────────────────────────────
```

---

## Technical Terms to Use in Interviews

| Plain English | Technical Term |
|---|---|
| One component failure = system down | Single Point of Failure (SPOF) |
| System stays up during failures | High Availability (HA) |
| System stays correct during failures | Fault Tolerance (FT) |
| Two DCs, one idle backup | Active-Passive |
| Two DCs, both handling traffic | Active-Active |
| Majority must agree before commit | Consensus / Quorum |
| Transactions in progress during crash | In-flight transactions |
| Log before execute for recovery | Write-Ahead Log (WAL) |
| Unique ID to prevent duplicate ops | Idempotency Key |
| You always read your own writes | Read-Your-Writes Consistency |
| Five nines uptime | 99.999% availability |

---

## Interview Questions + How to Answer

### Q: "How do you eliminate single points of failure?"

> "I'd audit every component in the system and
> ask — what happens if this one thing goes down?
> For databases — primary + replica with auto-failover.
> For load balancers — active-active pair with
> health checks. For caches — Redis Sentinel or
> Redis Cluster. The goal is that no single
> component failure takes down the system."

---

### Q: "What's the difference between HA and Fault Tolerance?"

> "High Availability means the system stays
> operational during failures — it might return
> slightly stale data but it responds.
> Fault Tolerance means the system stays correct
> during failures — it never returns a wrong answer.
> For a social feed I'd choose HA — stale likes
> count is fine. For payments I'd choose Fault
> Tolerance — a wrong balance is unacceptable.
> In practice most systems combine both —
> fault tolerant writes with highly available reads."

---

### Q: "Your payment service has one database. What's the risk?"

> "That database is a Single Point of Failure.
> If it goes down — all payment processing stops.
> For a fintech, that means transaction history
> unavailable, UPI transfers failing, potential
> RBI compliance violation, and direct revenue loss.
> I'd immediately add a replica with automatic
> failover, WAL for crash recovery, and idempotency
> keys on every transaction so retries after
> failover never cause double charges."

---

### Q: "We want 99.999% availability. How do you achieve it?"

> "Five nines means only 5 minutes of downtime
> per year. To achieve this I'd need:
> Active-active setup across two regions —
> if one goes down, the other handles all traffic
> with zero failover time.
> Zero SPOFs — every component redundant.
> Automated health checks and failover —
> no manual intervention required.
> The tradeoff is cost and complexity —
> active-active is significantly more expensive
> than active-passive, and handling write
> conflicts across regions adds complexity."

---

## Common Mistakes in Interviews

```
MISTAKE 1: Saying "we have 3 app servers
           so we're highly available"
  → HA requires ALL components to be redundant
  → 1 DB + 3 app servers = SPOF at DB level
  → System goes down if DB fails

MISTAKE 2: Confusing HA with Fault Tolerance
  → HA = stays up (may be stale)
  → FT = stays correct (never wrong)
  → Most interviews expect you to know the difference

MISTAKE 3: Recommending 2PC/SAGA for concurrency
  → 2PC/SAGA = cross-service distributed transactions
  → Concurrency (double booking) = distributed locks
  → Different problems, different solutions

MISTAKE 4: Not mentioning fintech consequences
  → Always connect SPOF to domain impact
  → Fintech: regulatory risk, financial liability
  → Healthcare: patient safety risk
  → E-commerce: revenue loss, reputation damage

MISTAKE 5: Forgetting Read-Your-Writes
  → Users must always see their own transactions
  → Route user's reads to primary after their write
  → Otherwise user panics and retries → double charge
```

---
---

# 🎯 Quick Reference — All Technical Terms (Topic 1)

| Concept | Simple Word | Technical Term | Use in Interview |
|---|---|---|---|
| System works even when broken | "works correctly" | Reliability | "Reliability is non-negotiable for payments" |
| Handles more users | "scales" | Scalability | "At 10M DAU we need horizontal scalability" |
| Easy to change | "easy to maintain" | Maintainability | "3 engineers can't maintain 15 microservices" |
| Time for one request | "how fast" | Latency (p99) | "Our p99 latency must be under 200ms" |
| Requests per second | "how many" | Throughput / QPS | "We handle 50K QPS at peak" |
| Worst 1% users | "slow users" | Tail latency / p99 | "p99 reveals tail latency issues" |
| One component = system down | "no backup" | SPOF | "The DB is a single point of failure" |
| System stays up | "always responds" | High Availability | "We target 99.99% availability" |
| System stays correct | "never wrong" | Fault Tolerance | "Payments need fault tolerance" |
| Both DCs handle traffic | "both active" | Active-Active | "Active-active gives us near-zero failover" |
| One DC is backup | "one standby" | Active-Passive | "Active-passive has 2min failover risk" |
| Majority must agree | "everyone agrees" | Consensus / Quorum | "Quorum ensures no data loss" |
| Unique ID per operation | "no duplicates" | Idempotency Key | "Idempotency prevents double charges" |
| Log before execute | "safe writes" | Write-Ahead Log (WAL) | "WAL enables crash recovery" |
| You read your own writes | "see my changes" | Read-Your-Writes | "Read-Your-Writes prevents balance panic" |

---

# 🗣️ The Tradeoff Formula — Apply to Every Answer

```
"We chose [X] over [Y].
 We gain [benefit A].
 We lose [cost B].
 That's acceptable because [reason C]."

Example:
"We chose High Availability over Fault Tolerance
 for our social feed.
 We gain lower latency and simpler architecture.
 We lose strong consistency — feed may be slightly stale.
 That's acceptable because showing a like count
 that's 2 seconds old has zero business impact."
```

---

# 📝 Things to Always Say in Interviews

1. **Before choosing HA or FT:** *"What's the cost of a wrong answer in this system?"*
2. **Before any latency discussion:** *"I always look at p99, not average"*
3. **When drawing architecture:** *"Let me make sure there's no SPOF here"*
4. **For any failure discussion:** *"What are the domain-specific consequences?"*
5. **For fintech systems:** *"We need fault tolerant writes + highly available reads"*

---

# 🚫 Things to Never Say in Interviews

1. ~~"All three goals are equally important"~~ → Always prioritise
2. ~~"Average latency is fine"~~ → Always use p99
3. ~~"We have 3 servers so we're highly available"~~ → Check all components
4. ~~"Use 2PC for double booking"~~ → Use distributed locks
5. ~~"Both HA and FT are impossible together"~~ → Say "possible with tradeoffs"

---

*Topic 1 Complete ✅ | Next: Topic 2 — Back-of-Envelope Estimation*