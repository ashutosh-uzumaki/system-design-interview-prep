# Chunk 36 — Multi-Leader Replication
### Phase 3 — Distributed Systems | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Google Docs

---

## Why Multi-Leader Exists

```
Single-leader problem — cross-region latency:

Mumbai DC (Primary) + Singapore DC (Replica)

Indian user writes order:
→ Hits Mumbai primary ✅ → ~5ms latency

Singapore user writes order:
→ Must go to Mumbai primary 😬
→ ~150ms network latency 😬
→ Very slow for Singapore users 😬

Solution: Give Singapore its own primary ✅
```

---

## Multi-Leader Architecture

```
Mumbai DC:    Primary 1 ← accepts writes ✅
Singapore DC: Primary 2 ← accepts writes ✅

Both sync with each other asynchronously

Indian user    → Mumbai Primary   → ~5ms ✅
Singapore user → Singapore Primary → ~5ms ✅
```

---

## Consistency Model

```
Within one datacenter:
→ Strong consistency ✅
→ Your writes to Mumbai DC immediately consistent
  for other Mumbai users ✅

Across datacenters:
→ Eventual consistency ❌
→ Singapore user might see stale Mumbai data
  for a few seconds 😬
→ Async sync between DCs

One line:
Multi-leader = strong within DC + eventual across DCs
```

---

## Where Conflicts Come From

```
Scenario 1 — Mobile app offline:
→ User updates address offline → "Andheri, Mumbai"
→ Phone reconnects to Singapore DC ✅
→ User updates address on laptop → "Orchard, Singapore"
→ Laptop connects to Mumbai DC ✅
→ Both updates sync → conflict 😬

Scenario 2 — Two devices simultaneously:
→ Phone (Singapore DC): updates address
→ Laptop (Mumbai DC): updates same address
→ Both within milliseconds → conflict 😬

Scenario 3 — Collaborative editing:
→ User A + User B edit same paragraph
→ A connected to US server
→ B connected to EU server
→ Classic multi-leader conflict 😬

Scenario 4 — Inventory:
→ Mumbai DC: reduces stock by 1 (Indian buyer)
→ Singapore DC: reduces stock by 1 (Singapore buyer)
→ Only 1 item in stock 😬
→ Both succeed → oversell 😬
```

---

## 4 Conflict Resolution Strategies

### Strategy 1 — Last Write Wins (LWW):

```
Rule: Highest timestamp = winner

Mumbai:    address = "Andheri"  @ t=100
Singapore: address = "Orchard"  @ t=101

LWW: t=101 wins → "Orchard" ✅
"Andheri" discarded

Problems:
❌ Clock skew → wrong winner
   Mumbai clock:    10:00:00.100
   Singapore clock: 10:00:00.098 (2ms behind)
   "Andheri" written LATER but gets lower timestamp
   → Wrong winner due to clock skew 😬

❌ Silent data loss:
   → User B's write silently discarded
   → User B has no idea 😬

Used by:
→ Cassandra (default) ✅
→ DynamoDB ✅
→ Acceptable for: non-critical, independent writes
```

### Strategy 2 — Merge Values:

```
Don't pick winner — MERGE both:

Shopping cart conflict:
→ Mumbai DC:    cart = [iPhone, AirPods]
→ Singapore DC: cart = [iPhone, MacBook]

Merge → cart = [iPhone, AirPods, MacBook] ✅
→ Union of both sets
→ No data lost ✅

Works well for:
→ Additive data (sets, lists) ✅
→ Shopping carts ✅
→ Tags, likes ✅

Doesn't work for:
→ Non-additive data (address = one value) ❌
→ Numeric values (inventory count) ❌

Used by:
→ Amazon shopping cart (Dynamo paper) ✅
→ Google Docs (merge text changes) ✅
→ CRDTs (Conflict-free Replicated Data Types) ✅
```

### Strategy 3 — Application-Level Resolution:

```
Expose conflict to application:
→ Both values preserved:
{
  value1: "Andheri",  timestamp: 100, dc: Mumbai
  value2: "Orchard",  timestamp: 101, dc: Singapore
  conflict: true
}

→ Application decides:
  → Ask user which is correct ✅
  → Apply business rules ✅
  → Custom merge logic ✅

Used by:
→ CouchDB ✅
→ Riak ✅
→ When business logic must decide ✅

Benefit: No silent data loss ✅
Cost: More complex application code ❌
```

### Strategy 4 — Avoid Conflicts (Best Approach):

```
Best solution = don't let conflicts happen

Rule: Route all writes for same record
      to same datacenter

hash(user_id) → always Mumbai DC:
→ User:123 always writes to Mumbai ✅
→ No conflict possible ✅

Only conflict risk:
→ User:123's home DC goes down
→ Failover to Singapore DC
→ Temporary conflict window ✅

Used by: Most production multi-leader systems ✅
Simplest + safest approach ✅
```

---

## Conflict Resolution Comparison

| Strategy | Data Loss | Complexity | Best For |
|---|---|---|---|
| **LWW** | Possible ❌ | Low ✅ | Non-critical, independent writes |
| **Merge** | None ✅ | Medium | Carts, sets, collaborative docs |
| **App-level** | None ✅ | High ❌ | Complex business rules |
| **Avoid conflicts** | None ✅ | Low ✅ | User-specific data ✅ |

---

## When to Use Multi-Leader

```
✅ Multi-region low latency writes:
→ Users in multiple regions
→ Each region = local primary
→ Low write latency everywhere ✅
→ Example: Flipkart India + Singapore

✅ Offline-first apps:
→ Mobile app works offline
→ Each device = local leader
→ Syncs when back online ✅
→ Example: Google Docs offline mode

✅ Collaborative editing:
→ Multiple users edit simultaneously
→ CRDTs handle merging ✅
→ Example: Google Docs, Notion, Figma
```

---

## When NOT to Use Multi-Leader

```
❌ Financial transactions:
→ Conflict = double charge 😬
→ Use single-leader + sharding instead ✅

❌ Inventory management:
→ Conflict = oversell 😬
→ Use single-leader ✅

❌ Seat booking:
→ Same seat booked twice 😬
→ Use single-leader ✅

❌ Any system needing strong consistency:
→ Multi-leader = eventual across DCs
→ Data integrity risk 😬
```

---

## Consistency Spectrum

```
Strong consistency (single-leader + sync):
→ Every read sees latest write ✅
→ Slow writes ❌
→ Use: financial systems, payments

Eventual (single-leader + async):
→ Reads might be stale for 1-2 seconds
→ Fast writes ✅
→ Fixes: replication lag patterns

Eventual (multi-leader):
→ Strong within DC ✅
→ Eventual across DCs ❌
→ Fast cross-region writes ✅
→ Use: multi-region, offline, collaboration

Tunable (Cassandra leaderless):
→ ONE / QUORUM / ALL per operation ✅
→ Use: massive scale, time-series
```

---

## Real Production Examples

```
Google Docs:
→ Multi-leader (each device = leader)
→ Operational transformation for merging ✅
→ Eventual consistency acceptable ✅

CouchDB:
→ Multi-leader built-in
→ Application resolves conflicts ✅

Flipkart multi-region:
→ Route user writes to home DC ✅
→ Conflict avoidance by design ✅
→ Sync across DCs asynchronously ✅

GitHub multi-region:
→ Multi-region writes
→ Conflict avoidance via routing ✅

NOT multi-leader:
→ Razorpay payments → single-leader ✅
→ Zerodha trades → single-leader ✅
→ PhonePe transactions → single-leader ✅
```

---

## Interview Framing

**On why multi-leader:**
> *"Multi-leader solves the cross-region write latency problem. With single-leader in Mumbai, Singapore users face 150ms write latency. Adding a Singapore primary brings that down to 5ms. The tradeoff is conflict resolution — writes to same record from different DCs simultaneously create conflicts."*

**On conflict resolution:**
> *"I'd use conflict avoidance first — route all writes for a user to their home datacenter using hash(user_id). This eliminates most conflicts by design. For shopping carts, merge values (union). For critical financial data, I'd avoid multi-leader entirely and use single-leader with sharding instead."*

**On consistency:**
> *"Multi-leader gives strong consistency within a datacenter but eventual consistency across them. That's acceptable for product catalog or user activity, but not for payments or inventory where conflicts cause real financial damage."*

---

## Quick Drill Questions

```
Q1: Why does multi-leader exist?
    What problem does it solve?

Q2: User updates address on offline phone,
    then same address on laptop (online).
    Both sync. What happens? How to resolve?

Q3: Name 4 conflict resolution strategies
    with tradeoffs.

Q4: Flipkart has users in India + Singapore.
    Should they use multi-leader?
    How do they avoid conflicts?

Q5: When would you NEVER use multi-leader?
    Name 3 cases.

Q6: What's the consistency model of
    multi-leader replication?
```

---

## Drill Answers

### Q1: Why multi-leader
```
Problem: Single-leader cross-region write latency
→ Primary in Mumbai
→ Singapore user writes → 150ms to Mumbai 😬

Solution: Each DC has own primary
→ Singapore user → Singapore primary → 5ms ✅
→ Indian user → Mumbai primary → 5ms ✅
→ Both sync asynchronously ✅
```

### Q2: Address conflict resolution
```
Conflict: Two values for same field
→ "Andheri" @ t=100 (phone, offline)
→ "Orchard" @ t=101 (laptop, online)

Options:
1. LWW → "Orchard" wins (latest timestamp)
   → Risk: clock skew → wrong winner 😬

2. Application-level → expose both to user
   → "Which address is correct?" ✅
   → No silent data loss ✅

3. Avoid conflict → route user writes to one DC
   → hash(user_id) → always Mumbai ✅
   → No conflict in first place ✅

Best: Avoid conflicts by routing ✅
```

### Q3: 4 conflict resolution strategies
```
1. LWW: highest timestamp wins
   → Simple ✅, data loss possible ❌

2. Merge: union of both values
   → No data loss ✅, only for additive data ❌

3. Application-level: expose conflict to app
   → Full control ✅, more complex ❌

4. Avoid conflicts: route to same DC
   → No conflicts ✅, not always possible ❌
```

### Q4: Flipkart multi-leader
```
Yes, multi-leader makes sense:
→ India + Singapore users need low latency ✅
→ Product catalog, user activity = eventual ok ✅

Conflict avoidance:
→ hash(user_id) → always same DC for that user
→ User:123 always → Mumbai DC ✅
→ No conflict possible ✅

NOT multi-leader for:
→ Inventory → oversell risk 😬 → single-leader
→ Payments → double charge risk 😬 → single-leader
```

### Q5: Never use multi-leader
```
1. Financial transactions:
   → Conflict = double charge 😬
   → Use single-leader + sharding ✅

2. Inventory management:
   → Conflict = oversell 😬
   → Single-leader with row locking ✅

3. Seat/ticket booking:
   → Same seat booked twice 😬
   → Single-leader with pessimistic locking ✅
```

### Q6: Multi-leader consistency
```
Within datacenter:
→ Strong consistency ✅
→ Writes immediately visible to same DC users

Across datacenters:
→ Eventual consistency ❌
→ 1-2 second sync delay between DCs
→ Users in different DCs may see stale data

One line:
Strong within DC + eventual across DCs
```

---

## One-Line Summaries

```
Multi-leader      → multiple primaries, each accepts writes
Why               → cross-region low latency writes
Conflict          → same record updated on two primaries simultaneously
LWW               → latest timestamp wins, data loss risk
Merge             → union of values, good for additive data
App-level         → expose conflict to application, full control
Avoid conflicts   → route same record to same DC always ✅
Within DC         → strong consistency ✅
Across DCs        → eventual consistency ❌
Never use for     → payments, inventory, seat booking
Use for           → multi-region, offline apps, collaboration
```

---

*Chunk 36 complete*
*Next: Chunk 37 — Conflict Resolution Deep Dive + Leaderless Replication*
*Phase 3 progress: 4/22 chunks done*