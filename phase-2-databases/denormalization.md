# Chunk 24B — Denormalization
### Phase 2 — Databases | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Swiggy, Hotstar

---

## The Problem — JOINs at Scale

```
Swiggy normalized schema:

users:       [user_id, name, phone]
orders:      [order_id, user_id, restaurant_id, total]
restaurants: [restaurant_id, name, city]

Query: "Show order history for user:123"
→ Needs: order details + restaurant name

SELECT o.order_id, o.total, r.name
FROM orders o
JOIN users u ON o.user_id = u.user_id
JOIN restaurants r ON o.restaurant_id = r.restaurant_id
WHERE u.user_id = 123;

At Swiggy scale:
→ 10M users checking order history daily
→ 10M × 3 table JOINs
→ DB CPU maxes out 😬
→ Query latency spikes 😬
```

---

## What is Denormalization?

```
Normalization:
→ Each piece of data stored ONCE
→ No duplication
→ JOINs to combine data

Denormalization:
→ Deliberately duplicate data
→ Store related data together
→ No JOINs needed ✅
→ Faster reads ✅
```

---

## Swiggy Example — Before vs After:

```
Normalized (3 tables, JOIN required):
──────────────────────────────────────
users:       [user_id, name, phone]
restaurants: [restaurant_id, name, city]
orders:      [order_id, user_id, restaurant_id, total]
→ 3 table JOIN every query 😬

Denormalized (1 table, no JOIN):
─────────────────────────────────
orders: [
  order_id,
  user_id,
  user_name,         ← copied from users
  restaurant_id,
  restaurant_name,   ← copied from restaurants
  restaurant_city,   ← copied from restaurants
  total
]

Query: SELECT * FROM orders WHERE user_id = 123;
→ Single table read ✅
→ No JOIN needed ✅
→ 10M reads = fast ✅
```

---

## The Core Tradeoff

```
Restaurant changes name:
"Burger King" → "BK Restaurant"

Normalized:
→ Update ONE row in restaurants table ✅
→ All orders show new name automatically ✅

Denormalized:
→ Must update EVERY order row 😬
→ user:123 has 50 orders → 50 updates
→ 10M users × 50 orders = 500M updates 😬
→ Data inconsistency risk 😬
```

| | Normalized | Denormalized |
|---|---|---|
| **Data duplication** | None ✅ | Yes ❌ |
| **Updates** | Easy (one place) ✅ | Expensive (many places) ❌ |
| **Consistency** | Always ✅ | Risk of inconsistency ❌ |
| **Read performance** | Slower (JOINs) ❌ | Faster (no JOINs) ✅ |
| **Query complexity** | Higher ❌ | Lower ✅ |

---

## When to Denormalize — 4 Conditions

```
1. Read-heavy system:
→ 10+ reads per write ratio
→ Optimize for reads ✅
→ Swiggy order history = read-heavy ✅

2. Performance critical path:
→ Query runs millions of times/day
→ Every millisecond matters
→ JOINs becoming bottleneck ✅

3. Data rarely changes:
→ Restaurant name changes = rare
→ Order total = never changes
→ Safe to duplicate ✅

4. Denormalized data fits access pattern:
→ "Always show order + restaurant together"
→ Never need restaurant data alone
→ Makes sense to combine ✅
```

---

## When NOT to Denormalize

```
❌ Write-heavy system:
→ Updates expensive
→ Denormalization makes writes worse

❌ Data changes frequently:
→ Price changes every hour
→ Duplicated price = always stale 😬

❌ Strong consistency required:
→ Financial data
→ Can't afford any inconsistency

❌ Small scale:
→ JOINs on small tables = fast
→ No need to denormalize
→ Premature optimization 😬

❌ Complex relationships:
→ Many-to-many with frequent updates
→ Denormalization becomes unmaintainable
```

---

## 4 Denormalization Techniques

### Technique 1 — Duplicate Columns:

```
Copy frequently read columns into main table:

orders table adds:
→ restaurant_name (from restaurants)
→ seller_name (from sellers)
→ user_phone (from users)

Use when:
→ Column rarely changes ✅
→ Read together always ✅
→ JOIN cost too high ✅
```

### Technique 2 — Computed Columns:

```
Pre-calculate + store result:

orders table adds:
→ item_count (pre-counted, not counted on every read)
→ discount_amount (pre-calculated)
→ delivery_time_minutes (pre-calculated)

Without computed column:
SELECT COUNT(*) FROM order_items WHERE order_id = 123
→ Counts every read 😬

With computed column:
SELECT item_count FROM orders WHERE order_id = 123
→ Single column read ✅
```

### Technique 3 — Summary Tables:

```
Pre-aggregate data for frequent analytics:

Without summary table:
"Total revenue for merchant M123 today"
→ SELECT SUM(amount) FROM transactions
  WHERE merchant_id = 'M123' AND date = TODAY
→ Scans millions of rows 😬

With summary table:
CREATE TABLE daily_merchant_revenue (
  merchant_id VARCHAR,
  date        DATE,
  total       DECIMAL,
  PRIMARY KEY (merchant_id, date)
);

Query:
SELECT total FROM daily_merchant_revenue
WHERE merchant_id = 'M123' AND date = TODAY;
→ Single row read ✅
→ Milliseconds ✅

Real use: Razorpay merchant dashboard ✅
```

### Technique 4 — Materialized Views:

```
DB maintains pre-computed query result:
→ Auto-updated when source data changes ✅
→ Read = just select from view ✅

CREATE MATERIALIZED VIEW merchant_stats AS
SELECT 
  merchant_id,
  COUNT(*) as total_orders,
  SUM(amount) as total_revenue,
  AVG(amount) as avg_order_value
FROM transactions
GROUP BY merchant_id;

PostgreSQL: REFRESH MATERIALIZED VIEW merchant_stats;
Cassandra:  Native materialized views ✅

Use when:
→ Complex aggregation query
→ Run frequently
→ Source data changes infrequently
```

---

## Real Examples at Indian Companies

```
Swiggy order history:
→ Denormalize restaurant_name into orders ✅
→ Restaurant name rarely changes
→ Orders read 10x more than written ✅

Flipkart product catalog:
→ Denormalize seller_name into products ✅
→ "Sold by XYZ" without JOIN ✅
→ Seller name rarely changes

Razorpay transaction history:
→ Denormalize merchant_name into transactions ✅
→ Dashboard reads = fast ✅
→ Merchant name rarely changes

Hotstar watch history:
→ Denormalize show_name into watch_history ✅
→ "You watched Breaking Bad" without JOIN ✅
→ Show name never changes

Zepto order tracking:
→ Denormalize delivery_partner_name into orders ✅
→ Show rider name without JOIN ✅
```

---

## Denormalization vs NoSQL

```
MongoDB/Cassandra = denormalized by design:
→ Embed related data in document/row
→ No JOINs by design
→ Denormalization = default ✅
→ "Design by access pattern"

PostgreSQL = normalized by default:
→ JOINs expected
→ Denormalize SELECTIVELY
→ Only where performance demands ✅
→ Keep financial data normalized ✅

Rule:
→ NoSQL → always denormalize
→ SQL → denormalize only hot query paths
```

---

## Handling Denormalization Updates

```
Problem: restaurant changes name
→ Must update all order rows 😬

Solutions:

1. Accept eventual consistency:
→ Old orders show old name
→ New orders show new name
→ Usually acceptable for historical data ✅

2. Background job:
→ Update task runs async
→ Batch update all affected rows
→ Slight inconsistency window ✅

3. Event-driven update (Kafka):
→ Restaurant updates name
→ Publishes event to Kafka
→ Consumer updates all order rows
→ Eventual consistency ✅

4. Don't denormalize frequently changing data:
→ Only denormalize stable columns
→ restaurant_name = rarely changes = safe ✅
→ product_price = changes daily = don't denormalize ❌
```

---

## Decision Framework

```
Should I denormalize this column?

Ask:
1. How often is it read?
   → Millions/day → denormalize candidate ✅

2. How often does it change?
   → Rarely → safe to denormalize ✅
   → Frequently → don't denormalize ❌

3. Is it always read with parent entity?
   → Yes → denormalize ✅
   → Sometimes → keep normalized

4. Is consistency critical?
   → Financial data → keep normalized ✅
   → Display data → denormalize ok ✅

5. What's the read:write ratio?
   → 10:1 or higher → denormalize ✅
   → 1:1 → keep normalized
```

---

## Interview Framing

**On denormalizing order history:**
> *"I'd denormalize restaurant_name into the orders table for Swiggy's order history page. This eliminates a JOIN on the most frequently accessed query — users check order history 10x more than restaurants update their names. The tradeoff is update complexity, but restaurant name changes are rare enough that the read performance gain is worth it."*

**On summary tables:**
> *"For Razorpay's merchant dashboard showing daily revenue, I'd use a summary table pre-aggregated by merchant_id and date. Scanning millions of transactions on every dashboard load isn't sustainable at scale. The summary table is updated via a background job or Kafka consumer after each transaction."*

**On when NOT to denormalize:**
> *"I wouldn't denormalize product prices into order items — prices change frequently and financial data requires consistency. Instead I'd store price_at_purchase as a snapshot at order time, which is actually a form of intentional denormalization for immutable historical accuracy."*

---

## Quick Drill Questions

```
Q1: What is denormalization and why do we use it?

Q2: Flipkart stores product price in both
    products table and order_items table.
    Is this denormalization? Is it good or bad?

Q3: Swiggy wants to show "ordered from X restaurant"
    on order history. Normalize or denormalize?
    Why?

Q4: When should you NOT denormalize?
    Name 3 conditions.

Q5: What is a summary table?
    Give a real Razorpay example.

Q6: What is a materialized view?
    How is it different from a summary table?

Q7: Restaurant changes its name from 
    "Burger King" to "BK Restaurant".
    You've denormalized restaurant_name into orders.
    How do you handle the update?

Q8: Cassandra and MongoDB — are they 
    normalized or denormalized by design?
    Why?
```

---

## Drill Answers

### Q1: What is denormalization
```
Normalization = store each piece of data once,
                use JOINs to combine

Denormalization = deliberately duplicate data
                  across tables to avoid JOINs

Why use it:
→ JOINs expensive at scale
→ Read-heavy systems need fast queries
→ Duplicate stable data = no JOIN needed
→ Trade storage for read performance ✅
```

### Q2: Price in both products and order_items
```
Yes = denormalization ✅
Is it good? YES ✅

Reason:
→ product_price changes over time
→ Order must preserve price AT TIME OF PURCHASE
→ "price_at_purchase" in order_items = correct ✅
→ If order showed current price → wrong 😬
→ This is intentional denormalization
  for historical accuracy ✅

Called: Snapshot pattern
→ Capture value at point in time
→ Never affected by future changes ✅
```

### Q3: Swiggy restaurant name
```
Denormalize ✅

Reasons:
→ Order history = read-heavy (users check daily)
→ Restaurant name = rarely changes
→ Always shown together with order
→ JOIN cost too high at 10M users scale

restaurant_name copied into orders table:
→ No JOIN needed for order history ✅
→ Update risk = low (name rarely changes) ✅
```

### Q4: When NOT to denormalize
```
1. Data changes frequently:
   → product_price, stock_count
   → Duplicated data = always stale 😬

2. Strong consistency required:
   → Financial ledger, payment amounts
   → Can't afford any inconsistency

3. Write-heavy system:
   → Updates = expensive across all duplicates
   → Makes write performance worse 😬

Bonus:
4. Small scale (JOINs fast on small tables)
5. Complex many-to-many relationships
```

### Q5: Summary table
```
Pre-aggregated table for frequent analytics:

Without:
SELECT SUM(amount) FROM transactions
WHERE merchant_id = 'M123' AND date = TODAY
→ Scans millions of rows per dashboard load 😬

With summary table:
daily_merchant_revenue:
[merchant_id, date, total_revenue, order_count]

SELECT total_revenue FROM daily_merchant_revenue
WHERE merchant_id = 'M123' AND date = TODAY
→ Single row read ✅
→ Milliseconds ✅

Updated via: background job or Kafka consumer
after each transaction ✅
```

### Q6: Materialized view vs summary table
```
Summary table:
→ Manually maintained by application/job
→ You write the update logic
→ More control
→ Works in any DB

Materialized view:
→ DB maintains automatically ✅
→ Defined as SQL query
→ Refreshed on schedule or on demand
→ PostgreSQL: REFRESH MATERIALIZED VIEW
→ Less application code needed ✅

Both solve same problem:
→ Pre-compute expensive aggregations
→ Fast reads ✅

Choose based on:
→ Need auto-refresh? → Materialized view
→ Need custom logic? → Summary table
```

### Q7: Restaurant name change handling
```
Options:

1. Accept eventual consistency (simplest):
→ Old orders show old name
→ Historical accuracy preserved
→ Usually acceptable ✅

2. Background job:
→ Async batch update all affected rows
→ Slight inconsistency window
→ Acceptable for non-critical data ✅

3. Event-driven (Kafka):
→ Restaurant updates → publishes event
→ Consumer updates order rows
→ Eventual consistency ✅

4. Don't denormalize frequently changing data:
→ Only denormalize stable columns
→ restaurant_name = stable = ok ✅
→ restaurant_rating = changes = don't denormalize

Best answer in interview:
→ Accept historical accuracy (old orders = old name)
→ New orders use new name
→ Background job for recent orders ✅
```

### Q8: Cassandra and MongoDB
```
Both = denormalized by design ✅

MongoDB:
→ Document = self-contained JSON
→ Related data embedded in document
→ No JOINs exist
→ restaurant + menu in same document ✅

Cassandra:
→ "Design by access pattern"
→ Same data duplicated across multiple tables
→ One table per query pattern
→ watch_history_by_user + watch_history_by_show
→ Same data stored twice intentionally ✅

Why:
→ Distributed systems = cross-node JOINs expensive
→ Denormalization = single node read ✅
→ Storage cheap, network calls expensive
```

---

## One-Line Summaries

```
Denormalization  → duplicate data to avoid JOINs
Normalized       → one place per data, JOINs to combine
Summary table    → pre-aggregated for fast analytics
Materialized view → DB-maintained pre-computed query
Computed column  → pre-calculated stored result
Snapshot pattern → capture value at point in time
When to use      → read-heavy, data rarely changes
When not to use  → write-heavy, frequent changes, financial
```

---

*Chunk 24B complete*
*Next: Chunk 24C — Cache Invalidation*
*Phase 2 progress: 12/16 chunks done*