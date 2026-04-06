# Chunk 22 — Document Stores (MongoDB)
### Phase 2 — Databases | SDE-2 Interview Prep
> Target: Razorpay, PhonePe, Flipkart, Zomato, Swiggy

---

## The Problem MongoDB Solves

```
Flipkart product catalog in SQL:

iPhone 15:    color, storage, battery, camera, chip...
Nike Shoes:   size, color, material, sole, sport...
Samsung TV:   screen size, resolution, HDR, HDMI ports...

SQL approach:
→ Create separate table per category ❌ (100s of tables)
→ OR add nullable columns for every possible attribute

products table:
→ color, storage, battery, camera, chip  ← phone fields
→ size, material, sole, sport            ← shoe fields
→ screen_size, resolution, hdmi_ports    ← TV fields
→ 200+ nullable columns
→ 90% NULL for any given product 😬
→ ALTER TABLE on 500M rows = risk 😬

This is the SPARSE COLUMN PROBLEM
```

---

## What MongoDB Does Instead

Each product = self-contained JSON-like document with exactly the fields it needs:

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "iPhone 15",
  "category": "phone",
  "price": 79999,
  "specs": {
    "storage": "256GB",
    "camera": "48MP",
    "chip": "A16"
  },
  "tags": ["5G", "premium", "apple"],
  "in_stock": true
}

{
  "_id": ObjectId("507f1f77bcf86cd799439012"),
  "name": "Nike Air Max",
  "category": "shoes",
  "price": 8999,
  "specs": {
    "sizes": [7, 8, 9, 10],
    "material": "mesh",
    "sport": "running"
  }
}
```

Different fields. Same collection. No nulls. No schema change needed. ✅

---

## Core Terminology

```
SQL              MongoDB
────────────     ────────────
Database    →    Database
Table       →    Collection
Row         →    Document
Column      →    Field
Primary Key →    _id (ObjectId)
```

---

## BSON — How MongoDB Stores Documents

```
JSON:
→ Text based → slow to parse
→ No date type
→ No binary type
→ No integer vs float distinction

BSON (Binary JSON):
→ Binary encoded → faster to parse ✅
→ Extra types:
   → Date type       ✅
   → Binary data     ✅
   → Integer + float ✅
   → ObjectId        ✅ (MongoDB's primary key type)
```

### ObjectId — auto-generated _id:
```
ObjectId("507f1f77bcf86cd799439011")
→ 12 bytes total
→ 4 bytes  = timestamp
→ 5 bytes  = machine ID + process ID
→ 3 bytes  = random counter
→ Globally unique ✅
→ Sortable by creation time ✅
```

---

## Schema Flexibility

```
No fixed schema by default:
→ Document 1 can have 5 fields
→ Document 2 can have 20 fields
→ Document 3 can have completely different fields
→ All in same collection ✅

Missing field behavior:
Query: find products where discount > 10

Document 1 → discount: 20  → returned ✅
Document 2 → no discount   → NOT returned ✅
→ MongoDB treats missing field = null
→ null > 10 = false → filtered out gracefully
```

---

## Schema Drift — The Real Problem

```
No enforced schema → developers store inconsistently:

Developer A: "discount": 20        ← integer
Developer B: "discount": "20%"     ← string 😬
Developer C: forgets field entirely 😬

Query breaks because types inconsistent 😬
```

### Fix — Schema Validation:
```javascript
db.createCollection("products", {
  validator: {
    $jsonSchema: {
      required: ["name", "price"],
      properties: {
        price: { bsonType: "int" },
        name:  { bsonType: "string" }
      }
    }
  }
})
```

```
Result:
→ name and price required ✅
→ price must be integer ✅
→ Other fields still flexible ✅
→ Best of both worlds ✅
```

---

## Collection Scan vs Index

```
No index on price — query: price < 50000:
→ Open collection
→ Read document 1 → check price → no match
→ Read document 2 → check price → match ✅
→ Read document 3 → price missing → skip
→ Repeat for ALL 500M documents 😬
→ This = Collection Scan (MongoDB's full table scan)
```

---

## MongoDB Index Types

### Storage engine: WiredTiger
```
→ B-Tree for indexes (same concept as SQL) ✅
→ Document-level locking (not table-level)
→ Compression built-in
→ Left-prefix rule applies to compound indexes ✅
```

### 1. Single Field Index:
```javascript
db.products.createIndex({ price: 1 })
// 1 = ascending, -1 = descending
```

### 2. Compound Index (same as SQL composite):
```javascript
db.products.createIndex({ category: 1, price: 1 })

// Left prefix rule applies:
WHERE category = "phones"            → ✅ efficient
WHERE category = "phones" AND price  → ✅ efficient
WHERE price < 50000                  → ❌ collection scan
```

### 3. Multikey Index (unique to MongoDB):
```javascript
db.products.createIndex({ tags: 1 })

// Document:
{ "tags": ["5G", "premium", "apple"] }

// Creates separate index entry per array element:
"5G"      → document ptr
"apple"   → document ptr
"premium" → document ptr

// Query: find products where tags = "5G"
→ Single index lookup ✅
→ No array scanning needed ✅
// SQL can't do this natively (no array type)
```

### 4. Text Index:
```javascript
db.products.createIndex({ name: "text" })
// Full text search on string fields
// Powers search features
```

### 5. TTL Index (unique to MongoDB):
```javascript
db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 3600 }
)
// Auto-deletes documents after 1 hour
// Like Redis TTL but for entire documents ✅
```

### 6. Geospatial Index:
```javascript
db.restaurants.createIndex({ location: "2dsphere" })
// Powers location-based queries
// "Find restaurants within 5km" ✅
```

---

## When MongoDB Wins — 4 Exact Conditions

```
1. Flexible/unpredictable schema
   → Product catalogs (Flipkart, Myntra)
   → Fields vary completely per document
   → No nullable columns needed ✅

2. Hierarchical/nested data
   → Data naturally fits as nested objects
   → Menu inside restaurant document
   → No JOINs needed ✅

3. Rapid iteration (early stage)
   → Schema changes frequently
   → No migration scripts needed
   → Add new field → just add it ✅

4. Document-centric access pattern
   → Always fetch entire document
   → Never need partial joins
   → "Get everything about restaurant X" ✅
```

---

## Real Example — Zomato Restaurant Profile:

```
SQL approach:
→ restaurants table + menus table + hours table + photos table
→ Complex JOINs every query 😬
→ New hour type (holiday special)?
   → ALTER TABLE 😬

MongoDB approach:
{
  "name": "Burger King",
  "location": { "lat": 12.97, "lng": 77.59 },
  "cuisine": "fast food",
  "menu": [
    { "item": "Whopper", "price": 250, "veg": false },
    { "item": "Fries",   "price": 99,  "veg": true }
  ],
  "hours": {
    "monday":  "9AM-11PM",
    "diwali":  "closed"    ← only added when needed ✅
  },
  "photos": ["url1", "url2", "url3"]
}

New restaurant with drone delivery field?
→ Just add it ✅
→ No schema change ✅
→ No ALTER TABLE ✅
```

---

## When MongoDB LOSES

```
❌ ACID transactions needed
   → MongoDB added multi-doc transactions in v4.0
   → Still slower + less mature than PostgreSQL
   → Financial data → use PostgreSQL

❌ Complex relationships
   → No true JOINs (has $lookup but expensive)
   → Many-to-many = painful
   → Social network → SQL or Graph DB

❌ Strong consistency required
   → Eventual consistency by default
   → Payment systems → PostgreSQL

❌ Complex analytics/aggregations
   → SQL more expressive for analytics
   → Heavy analytics → data warehouse (BigQuery)

❌ Schema drift risk at scale
   → Large teams + no schema = inconsistent data
   → Must enforce validation manually
   → Discipline required
```

---

## Real Company Usage

```
MongoDB (flexible schema needed):
→ Flipkart  — product catalog (varying attributes)
→ Zomato    — restaurant profiles + menus
→ Swiggy    — restaurant + menu data
→ Myntra    — fashion catalog (size, color, fabric vary)
→ Nykaa     — beauty product catalog

NOT MongoDB (ACID/consistency needed):
→ Razorpay  — payments        → PostgreSQL
→ Zerodha   — stock trades    → PostgreSQL
→ PhonePe   — transactions    → PostgreSQL
```

---

## Full Decision Framework

```
Need ACID + relational data?          → PostgreSQL
Need fast temporary cache + TTL?      → Redis
Need flexible schema + nested data?   → MongoDB
Need massive write throughput?        → Cassandra
Need full-text search?                → Elasticsearch
Need graph relationships?             → Neo4j
```

---

## MongoDB vs SQL — Side by Side

| | PostgreSQL | MongoDB |
|---|---|---|
| **Schema** | Fixed, rigid | Flexible, optional |
| **Data model** | Tables + rows | Collections + documents |
| **Relationships** | JOINs + foreign keys | Embedded docs / $lookup |
| **ACID** | Full support ✅ | Multi-doc (v4.0+) ⚠️ |
| **Indexes** | B-Tree | B-Tree + Multikey + Text + TTL |
| **Scaling** | Vertical + sharding | Built-in sharding ✅ |
| **Best for** | Financial, relational | Catalog, content, flexible |

---

## Interview Framing

**On choosing MongoDB:**
> *"For Flipkart product catalog I'd use MongoDB — each category has completely different attributes, and SQL would require hundreds of nullable columns or complex JOINs. MongoDB lets each product document carry exactly the fields it needs. Adding a new category requires zero schema changes."*

**On not using MongoDB for payments:**
> *"For payments I'd still use PostgreSQL — MongoDB's transaction support has improved since v4.0 but PostgreSQL is battle-tested for financial data with full ACID guarantees."*

**On schema drift:**
> *"The risk with MongoDB at scale is schema drift — without enforcement, different developers store the same field differently. I'd add JSON schema validation on critical fields while keeping flexibility for optional attributes."*

---

## Drill Questions

```
Q1: What is the sparse column problem and 
    how does MongoDB solve it?

Q2: What is BSON and why does MongoDB use it over JSON?

Q3: Zomato needs to store restaurant profiles with 
    varying menus and special hours. SQL or MongoDB? Why?

Q4: What is schema drift and how do you fix it in MongoDB?

Q5: What is a Multikey index and why doesn't SQL have it?

Q6: What is a TTL index in MongoDB? 
    Give a real use case.

Q7: When would you NOT use MongoDB?
    Name 3 conditions.

Q8: Flipkart wants to store both product catalog 
    AND payment transactions.
    Which DB for each and why?
```

---

## Drill Answers

### Q1: Sparse column problem + MongoDB solution
```
Sparse column problem:
→ SQL table needs fixed columns for all possible attributes
→ iPhone needs: storage, camera, chip
→ Nike shoes needs: size, material, sport
→ Same table = hundreds of nullable columns
→ 90% NULL for any given product 😬
→ Wastes storage, complicates queries

MongoDB solution:
→ Each product = document with only its own fields
→ iPhone doc has storage, camera, chip
→ Shoes doc has size, material, sport
→ No nulls, no wasted storage ✅
→ No schema change when new category added ✅
```

### Q2: BSON vs JSON
```
JSON → text based, slow to parse, limited types
BSON → binary encoded, faster to parse ✅
     → extra types: Date, Binary, ObjectId, Integer ✅
     → ObjectId = auto-generated unique _id
       (timestamp + machine ID + counter)
     → Globally unique + sortable by creation time ✅
```

### Q3: Zomato restaurant profiles
```
MongoDB ✅

Reasons:
1. Flexible schema
   → Each restaurant has different menu structure
   → Different special hours (Diwali, Christmas)
   → SQL = sparse columns or complex JOINs 😬

2. Nested data
   → Menu naturally lives inside restaurant document
   → No JOIN needed to fetch restaurant + menu ✅

3. Rapid changes
   → New field needed (drone delivery zone)?
   → Just add to document ✅
   → No ALTER TABLE on 500M rows 😬

4. Document-centric access
   → "Get everything about this restaurant"
   → One document fetch = all data ✅
```

### Q4: Schema drift + fix
```
Schema drift:
→ No enforced schema → developers inconsistent
→ "discount": 20 vs "discount": "20%" vs missing
→ Queries break, data unreliable 😬

Fix — JSON Schema Validation:
db.createCollection("products", {
  validator: {
    $jsonSchema: {
      required: ["name", "price"],
      properties: {
        price: { bsonType: "int" },
        name:  { bsonType: "string" }
      }
    }
  }
})

→ Enforce critical fields ✅
→ Keep optional fields flexible ✅
```

### Q5: Multikey index
```
Multikey index:
→ Indexes each element of an array separately
→ Document: { tags: ["5G", "premium", "apple"] }
→ Creates: "5G" → ptr, "apple" → ptr, "premium" → ptr
→ Query: find products where tags = "5G" → single lookup ✅

Why SQL doesn't have it:
→ SQL has no native array column type
→ Arrays stored in separate junction tables
→ Requires JOIN to query array elements 😬
→ MongoDB arrays are first-class citizens ✅
```

### Q6: TTL index + use case
```
TTL index:
→ Auto-deletes documents after specified time
→ No application-level cleanup needed

db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 3600 }
)

Real use cases:
→ User sessions → expire after 1 hour
→ OTP documents → expire after 2 minutes
→ Temporary cart → expire after 24 hours
→ Log documents → expire after 30 days

Similar to Redis TTL but for entire MongoDB documents ✅
```

### Q7: When NOT to use MongoDB
```
1. ACID transactions needed
   → Multi-doc transactions slower than PostgreSQL
   → Financial data → PostgreSQL

2. Complex relationships
   → No true JOINs, $lookup is expensive
   → Many-to-many relationships = painful
   → Use SQL or Graph DB

3. Strong consistency required
   → Eventual consistency by default
   → Payment systems → PostgreSQL

Bonus:
4. Complex analytics → data warehouse (BigQuery)
5. Schema discipline critical → SQL enforces it
```

### Q8: Flipkart product catalog + payments
```
Product catalog → MongoDB
→ Varying attributes per category
→ No JOINs needed
→ Flexible schema
→ Rapid iterations on catalog structure

Payment transactions → PostgreSQL
→ ACID non-negotiable
→ Atomicity: debit + credit must happen together
→ Relational: payment links to user, merchant, order
→ Strong consistency required
→ Battle-tested for financial data

Two DBs, two different requirements. ✅
```

---

## One-Line Summaries

```
Document    → JSON-like object with flexible fields
Collection  → group of documents (like SQL table)
BSON        → binary JSON, faster + more types
ObjectId    → auto-generated unique _id with timestamp
Schema drift → inconsistent data without enforcement
Multikey    → indexes each array element separately
TTL index   → auto-deletes documents after time
$lookup     → MongoDB's expensive JOIN equivalent
```

---

*Chunk 22 complete*
*Next: Chunk 22B — OLTP vs OLAP*
*Phase 2 progress: 7/13 chunks done*