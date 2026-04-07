# Chunk 22B — OLTP vs OLAP + Multi-tenancy
### Phase 2 — Databases | SDE-2 Interview Prep
> Real example: Ajio OTB Project (Trends, Tira, Ajio Luxe)

---

# PART 1 — OLTP vs OLAP

## The Core Problem

```
Two teams, one DB:

Team A (Payments/Orders — OLTP):
→ Process 10,000 orders/sec
→ Each query touches 1-2 rows
→ Needs millisecond latency
→ Write-heavy

Team B (Analytics — OLAP):
→ "Total sales in Q3 by category?"
→ Scans 500 MILLION rows
→ Takes minutes
→ Read-heavy, aggregation-heavy

What happens on same DB?
→ Team B's 500M row scan holds locks
→ Consumes all CPU + I/O
→ Team A's order inserts queue up
→ Payment processing crawls 😬
→ Cascading failure 😬
```

**OLTP and OLAP have fundamentally opposite requirements — they fight each other on the same DB.**

---

## OLTP vs OLAP — Full Comparison

| | OLTP | OLAP |
|---|---|---|
| **Full form** | Online Transaction Processing | Online Analytical Processing |
| **Query type** | Many small fast queries | Few large slow queries |
| **Rows touched** | 1-100 rows per query | Millions-billions of rows |
| **Operation** | INSERT, UPDATE, DELETE | SELECT, GROUP BY, SUM, AVG |
| **Latency** | Milliseconds | Seconds to minutes |
| **Optimization** | Low latency | High throughput |
| **ACID needed** | Yes ✅ | No ❌ |
| **Stale data ok?** | No ❌ | Yes ✅ |
| **Storage** | Row-based | Columnar |
| **Examples** | PostgreSQL, MySQL | BigQuery, Redshift, Snowflake |

---

## Why Storage Model Matters

### Row-based (OLTP):
```
Disk layout:
[order_id | customer | category | revenue | date | store | sku]
[order_id | customer | category | revenue | date | store | sku]
[order_id | customer | category | revenue | date | store | sku]

Query: SELECT category, SUM(revenue) GROUP BY category
→ Must read ENTIRE row to get category + revenue
→ date, store, sku read unnecessarily 😬
→ 10GB data → reads ALL 10GB from disk 😬
→ Single machine → limited CPU + RAM 😬
```

### Columnar (OLAP):
```
Disk layout — each column stored separately:
category.col → [phones, shoes, phones, TV...]
revenue.col  → [500, 1200, 800, 2400...]
date.col     → [Jan-1, Jan-1, Jan-2...]
store.col    → [BLR01, MUM02, DEL01...]

Same query:
→ Read ONLY category.col + revenue.col ✅
→ Skip ALL other columns ✅
→ 10GB data → reads maybe 2GB ✅
→ 5x less I/O immediately ✅
```

---

## How Columnar Storage Maintains Row Identity

```
Position = implicit row identifier:

Position:    [  1,      2,      3   ]
order_id:    [  1,      2,      3   ]
customer:    [Alice,   Bob,   Carol ]
category:    [phones, shoes,  phones]
revenue:     [  500,  1200,    800  ]

Position N in every column = same row ✅

Query: WHERE order_id = 2
→ Find position of id=2 → position 2
→ customer.col[2]  = Bob
→ category.col[2]  = shoes
→ revenue.col[2]   = 1200 ✅
```

---

## Why BigQuery is Faster Than PostgreSQL for Analytics

### Real example — Ajio OTB Project (10-15GB data):

```
PostgreSQL on 10-15GB:
→ Row-based → reads entire rows
→ Single machine → limited CPU + RAM
→ Sequential processing
→ AOP/predicted sales query → 5-10 minutes 😬
```

```
BigQuery on same data:

Reason 1 — Columnar storage:
→ Reads only needed columns
→ 10GB → maybe reads 2GB ✅

Reason 2 — Columnar compression:
→ Same column = similar values = compresses well
→ category: [shoes×1000, phones×500, TV×300]
→ Run-length encoding → 10x compression ✅
→ Even less data to read

Reason 3 — Massively parallel processing:
→ 10GB split across 1000 workers
→ Each worker processes 10MB
→ All run simultaneously in parallel
→ Results merged → returned in seconds ✅

Reason 4 — Serverless:
→ No server to manage
→ Auto-allocates compute per query
→ 10GB query → 1000 workers instantly
→ 1TB query → 10000 workers instantly ✅

Result:
PostgreSQL → 5-10 minutes 😬
BigQuery   → 3-10 seconds ✅
```

---

## Why You Used PostgreSQL for Computed Output

```
Your OTB system:
────────────────
Raw data (10-15GB):
→ BigQuery ✅
→ Sales history, SKU data, demand signals
→ Heavy aggregations: AOP, predicted sales
→ Columnar + parallel = fast ✅

Computed output (~50 rows):
→ PostgreSQL ✅
→ Store configurations
→ AOP targets per brand
→ Inventory buckets
→ Relational: stores ↔ AOP ↔ inventory buckets

Why PostgreSQL for computed data:
→ Only 50 rows → BigQuery overkill
→ Relational — 3 entities with JOINs needed
→ Row-level updates when store uploads new file
→ Referential integrity required
→ Transactional consistency needed
→ BigQuery charges per query on scanned data
   → 50 rows in PostgreSQL = free + instant ✅
```

---

## How to Explain This in Interviews

**Your answer:**
> *"We used a hybrid approach — BigQuery for raw analytical data since we were processing 10-15GB of sales history with heavy aggregations like AOP and predicted sales calculations. Columnar storage and parallel processing made that feasible in seconds. But the computed outputs were only ~50 rows of structured relational data — store configurations, AOP targets, inventory buckets — so we stored those in PostgreSQL where we needed referential integrity and JOINs across entities. Each database was chosen for what it does best."*

**If asked "why not BigQuery for everything?":**
> *"BigQuery doesn't support row-level updates well and charges per query on scanned data. Our store/AOP/inventory data needed JOINs across 3 entities, row-level updates when stores upload files, and referential integrity. 50 rows in PostgreSQL is simpler, faster, and cheaper for that use case."*

**If asked "why not PostgreSQL for everything?":**
> *"PostgreSQL on a single machine processing 10-15GB with GROUP BY + SUM across millions of rows would take minutes per query. Our users needed near real-time results. BigQuery's columnar storage and parallel processing gave us results in seconds. Right tool for right job."*

---

## Real Company Architecture:

```
Transactional layer (OLTP):
→ PostgreSQL / MySQL
→ Orders, payments, inventory
→ Real-time, ACID required

Analytics layer (OLAP):
→ BigQuery / Redshift / Snowflake
→ Copy of transactional data
→ Loaded via ETL pipeline (nightly or streaming)
→ Heavy aggregations, reporting, ML features

Your OTB project = exactly this pattern ✅
Also used at: Flipkart, Zepto, Swiggy, Myntra
```

---

## This Pattern Has a Name — Lambda Architecture (simplified):

```
Raw data layer  → BigQuery  (OLAP, heavy processing)
Serving layer   → PostgreSQL (OLTP, relational outputs)
```

---

---

# PART 2 — Multi-tenancy (Ajio: Trends, Tira, Ajio Luxe)

## The 3 Multi-tenancy Approaches

---

### Approach 1 — Shared DB + tenant_id column:

```
One table, all tenants mixed:

| tenant_id | store | sales | AOP |
|-----------|-------|-------|-----|
| trends    | BLR01 | 500   | 480 |
| tira      | MUM01 | 300   | 290 |
| trends    | DEL01 | 700   | 650 |

Every query must filter by tenant_id:
SELECT * FROM sales WHERE tenant_id = 'trends'
```

```
Gain:
✅ Simple — one DB to manage
✅ Cheap — shared infrastructure
✅ Easy to add new tenant

Lose:
❌ Noisy neighbor problem
   → Tira's heavy query slows Trends 😬
❌ Data leak risk
   → Miss one WHERE tenant_id filter
   → Tira sees Trends data 😬
❌ Compliance issues
   → Brands may not want data co-located
❌ Shared quota — one tenant exhausts BQ quota
   → All tenants affected 😬
```

---

### Approach 2 — Shared DB, separate schemas/datasets:

```
One BQ connection, different datasets:

trends_dataset.sales_table
tira_dataset.sales_table
luxe_dataset.sales_table
```

```
Gain:
✅ Better isolation than tenant_id column
✅ Still shared infrastructure
✅ Easier compliance than Approach 1
✅ Accidental cross-tenant query harder

Lose:
❌ Still noisy neighbor (shared compute)
❌ One BQ connection = shared quota 😬
❌ Not fully isolated
```

---

### Approach 3 — Separate DB per tenant (what you built):

```
Trends → own BQ project + own tables
Tira   → own BQ project + own tables
Luxe   → own BQ project + own tables
```

```
Gain:
✅ Complete data isolation
✅ No noisy neighbor problem
   → Tira's heavy query never affects Trends ✅
✅ Each tenant has own BQ quota ✅
✅ Compliance friendly
   → Brands own their data completely
✅ Custom configurations per tenant
   → Trends might need different table structure
✅ Security
   → Breach of one tenant = doesn't affect others

Lose:
❌ More expensive (separate infra per tenant)
❌ More operational overhead
   → Schema change = update every tenant's tables
   → New feature = deploy to every connection
❌ Harder cross-tenant analytics
   → "Compare Trends vs Tira performance"
   → Needs separate aggregation layer
```

---

## Why Silo Model Was RIGHT for Ajio:

```
Trends, Tira, Ajio Luxe = SEPARATE business units:
→ Each has own merchandising team
→ Each has own data sensitivity
→ Tira (beauty) ≠ Trends (fashion) ≠ Luxe (premium)
→ Different data models per brand
→ Complete isolation = correct call ✅

If tenant_id column approach:
→ Tira's team could see Trends data accidentally
→ One heavy Tira query slows Trends planning 😬
→ Reliance won't accept this for separate brands 😬
```

---

## Multi-tenancy Comparison Table:

| | Shared DB + tenant_id | Shared DB + schema | Silo (separate DB) |
|---|---|---|---|
| **Isolation** | Low ❌ | Medium ⚠️ | High ✅ |
| **Noisy neighbor** | High risk 😬 | Medium risk ⚠️ | None ✅ |
| **Cost** | Cheapest ✅ | Cheap ✅ | Expensive ❌ |
| **Ops overhead** | Low ✅ | Low ✅ | High ❌ |
| **Compliance** | Risky ❌ | OK ⚠️ | Best ✅ |
| **Best for** | Small SaaS | Mid-size | Enterprise/regulated |

---

## How to Explain Multi-tenancy in Interviews:

**Your answer:**
> *"We used a silo model — database-per-tenant approach. Each brand like Trends and Tira had their own BigQuery connection and separate tables. This gave us complete data isolation, no noisy neighbor problem, and each brand had their own BigQuery quota so heavy queries from one brand never impacted another. The tradeoff was more operational overhead — schema changes had to be applied per tenant — but for Reliance's separate business units, isolation was non-negotiable."*

**If asked "why not shared DB with tenant_id?":**
> *"Two reasons: noisy neighbor — Tira running a heavy quarterly analysis would slow down Trends' daily planning queries on shared infrastructure. And data sensitivity — these are separate business units with separate merchandising strategies. A missed WHERE tenant_id filter creates a data leak between brands. Isolation was safer."*

**If asked "how did you handle schema changes?":**
> *"Schema changes were applied per tenant connection via a migration script that ran against each BQ connection sequentially. Operational overhead — but acceptable given the small number of tenants."*

---

## This Maps to Our Curriculum:

```
Phase 10 — Observability & Operations:
→ Chunk 156 — Multi-tenancy models
   → Shared DB + tenant_id (considered)
   → Shared DB + separate schemas
   → Silo model (what you built) ✅

→ Chunk 157 — Tenant isolation strategies
   → Noisy neighbor problem
   → Compliance considerations
   → Operational tradeoffs
```

---

## Quick Drill Questions

```
Q1: What is the noisy neighbor problem in multi-tenancy?

Q2: Why can't you use same DB for OLTP and OLAP?

Q3: Why is BigQuery faster than PostgreSQL for 10GB analytics?

Q4: Your company has 3 enterprise clients who are competitors.
    Which multi-tenancy model? Why?

Q5: What is columnar storage and why does it compress better?

Q6: Ajio OTB — why BigQuery for raw data 
    but PostgreSQL for computed output?

Q7: What is the Lambda architecture pattern?

Q8: A startup has 100 small tenants with low data volume.
    Which multi-tenancy model? Why?
```

---

## Drill Answers

### Q1: Noisy neighbor problem
```
One tenant's heavy workload consumes shared resources
→ Slows down ALL other tenants on same infrastructure

Example:
Tira runs quarterly analysis → scans 15GB
→ Consumes all shared BQ compute
→ Trends' daily planning queries queue up
→ Trends team can't work 😬

Solution: Silo model → each tenant own compute ✅
```

### Q2: OLTP and OLAP on same DB
```
OLTP query: SELECT * FROM orders WHERE id = 123
→ Touches 1 row, done in milliseconds ✅

OLAP query: SELECT SUM(revenue) FROM orders WHERE year = 2024
→ Scans 500M rows, takes minutes
→ Holds locks, consumes all CPU + I/O
→ OLTP inserts queue up
→ Payment processing slows to crawl 😬

Fundamentally opposite requirements:
OLTP → low latency, few rows, write-heavy
OLAP → high throughput, millions of rows, read-heavy
```

### Q3: BigQuery faster for analytics
```
3 reasons:
1. Columnar storage → reads only needed columns
   → 10GB query might only read 2GB ✅

2. Columnar compression → similar values compress well
   → Even less data to read ✅

3. Massively parallel → 1000s of workers simultaneously
   → 10GB split across 1000 machines
   → Each processes 10MB in parallel ✅

Combined: minutes → seconds ✅
```

### Q4: 3 enterprise competitor clients
```
Silo model (separate DB per tenant) ✅

Reasons:
→ Competitors → cannot risk data leaks
→ Compliance → each client owns their data
→ No noisy neighbor → each has own quota
→ Enterprise = willing to pay for isolation
→ tenant_id approach = unacceptable risk
  (missed filter = competitor sees your data 😬)
```

### Q5: Columnar storage + compression
```
Columnar storage:
→ Each column stored separately on disk
→ Query only reads needed columns
→10 column table, need 2 columns → read 20% of data ✅

Why it compresses better:
→ Same column = same data type = similar values
→ category: [shoes, shoes, shoes, phones, phones...]
→ Run-length encoding: shoes×1000, phones×500
→ Compresses 10x easily ✅

Row-based:
→ Mixed types per row → poor compression 😬
```

### Q6: Ajio OTB — BigQuery + PostgreSQL
```
BigQuery for raw data (10-15GB):
→ Sales history, SKU data = massive
→ Heavy aggregations: AOP, predicted sales
→ Columnar + parallel = seconds ✅
→ PostgreSQL would take minutes 😬

PostgreSQL for computed output (~50 rows):
→ Store configs, AOP targets, inventory buckets
→ Relational: 3 entities with JOINs
→ Row-level updates when stores upload
→ Referential integrity needed
→ 50 rows → BigQuery overkill + expensive
→ PostgreSQL = simpler, faster, free ✅
```

### Q7: Lambda architecture
```
Pattern for systems needing both:
→ Historical batch processing
→ Real-time stream processing

Three layers:
Batch layer  → processes all historical data (BigQuery/Spark)
Speed layer  → processes real-time data (Kafka/Flink)
Serving layer → merges results, serves queries (PostgreSQL/Redis)

Simplified version (what Ajio OTB used):
Raw layer    → BigQuery (heavy processing)
Serving layer → PostgreSQL (relational outputs)
```

### Q8: 100 small tenants, low data volume
```
Shared DB + tenant_id column ✅

Reasons:
→ 100 small tenants = low data per tenant
→ Low volume = noisy neighbor risk minimal
→ Shared infrastructure = much cheaper
→ Operational simplicity = one DB to manage
→ Adding new tenant = just new tenant_id value
→ Silo model for 100 tenants = 
  100 DBs to manage = too expensive + complex 😬

Tradeoff accepted:
→ Some isolation risk
→ But startup needs to keep costs low
→ Can migrate to silo model for enterprise clients later
```

---

## One-Line Summaries

```
OLTP         → many small fast queries, row-based, ACID
OLAP         → few large slow queries, columnar, analytics
Columnar     → store by column, read only needed columns
BigQuery     → columnar + parallel = analytics in seconds
Lambda arch  → batch layer + speed layer + serving layer
Silo model   → DB per tenant, max isolation, high cost
Shared DB    → one DB, tenant_id column, cheap, noisy neighbor risk
Noisy neighbor → one tenant's workload hurts others
```

---

*Chunk 22B complete*
*Next: Chunk 23 — Wide-Column Stores (Cassandra)*
*Phase 2 progress: 8/13 chunks done*