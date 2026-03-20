# 🔥 System Design Curriculum — Zero Rejection Edition
### SDE-2 Interview Prep | DDIA + Alex Xu + Real Interview Patterns
### Last Updated: Latest Version

---

## 📐 Every Chunk Format — Without Exception

```
1. Simple version       → plain English, zero jargon
2. Technical version    → proper terms, interview-ready language
3. Implementation       → how it actually works under the hood
4. Real example         → Indian/global companies — what they do exactly
5. Tradeoff             → gain X, lose Y, acceptable when Z
6. Interview framing    → exact sentence to say out loud
7. Quick Drill          → 2-3 questions immediately after chunk
```

> For consistency models, replication, consensus, caching, distributed locks,
> Kafka, OAuth, JWT, rate limiting — implementation is covered in full detail
> with real examples. Not just definitions.

---

## 🔁 Revision System

| Command | What happens |
|---|---|
| `revise chunk X` | 4-5 drill questions on that chunk |
| `revise topic X` | Full drill session — 5-8 questions |
| `revise phase X` | Phase review — 10-12 questions |
| `mock interview` | Full 45-min end-to-end simulation |

---

## 📊 Curriculum Stats

| | |
|---|---|
| **Phases** | 12 |
| **Topics** | 46 |
| **Total Chunks** | 190 |
| **Drill Sessions** | 46 |
| **Phase Reviews** | 12 |
| **Mock Interviews** | 3 |
| **Estimated Hours** | ~72h |
| **Total Questions** | 500+ |

---

---

# PHASE 1 — Foundations
**Estimated Time: 7 hours**

---

## Topic 1 — How to Think About Scale *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 1 | Reliability vs Scalability vs Maintainability | ✅ Done |
| Chunk 2 | Latency vs Throughput | ✅ Done |
| Chunk 3 | SPOF + Fault Tolerance vs High Availability | ✅ Done |
| Chunk 4 | Interview mental model — questions that signal seniority | ⬜ |
| 🔥 | **Topic 1 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 2 — Back-of-Envelope Estimation *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 5 | Key latency numbers every engineer must know | ⬜ |
| Chunk 6 | DAU → QPS → Storage formula | ⬜ |
| Chunk 7 | Bandwidth + Peak traffic estimation | ⬜ |
| Chunk 8 | Worked example end-to-end (Twitter scale) | ⬜ |
| 🔥 | **Topic 2 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 3 — Vertical vs Horizontal Scaling *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 9 | Vertical scaling — what it is, where it breaks, cost curve | ⬜ |
| Chunk 10 | Horizontal scaling + statelessness requirement | ⬜ |
| Chunk 10B | Stateful vs Stateless servers — tradeoffs, how to offload state, real examples | ⬜ |
| Chunk 11 | Load balancer role in horizontal scaling | ⬜ |
| Chunk 12 | Decision framework — when to scale up vs scale out | ⬜ |
| 🔥 | **Topic 3 Drill Session** — 5-8 questions | ⬜ |

> **Chunk 10B covers:**
> - Waiter analogy — remembers you vs doesn't
> - Where session/state lives in each model
> - Why stateless is required for horizontal scaling
> - How to handle unavoidable state → offload to Redis, DB, JWT
> - When stateful is unavoidable → WebSocket, game servers
> - Spring Boot session management — stateful vs stateless
> - Real example: Hotstar stateless app servers + Redis sessions
> - Tradeoffs + Interview framing

---

## Topic 4 — Concurrency Fundamentals *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 13 | Race conditions + dirty reads — simple to technical | ⬜ |
| Chunk 14 | Optimistic vs pessimistic locking | ⬜ |
| Chunk 15 | Distributed locks — intro only (deep dive in Phase 4) | ⬜ |
| Chunk 16 | Deadlocks — detection + prevention | ⬜ |
| 🔥 | **Topic 4 Drill Session** — 5-8 questions | ⬜ |

---

## 🧠 Phase 1 Review — 10-12 questions across all topics

---
---

# PHASE 2 — Databases Deep Dive
**Estimated Time: 8 hours**

---

## Topic 5 — SQL Internals & ACID *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 17 | ACID — each property with a real failure example | ⬜ |
| Chunk 18 | The 4 isolation levels — what each prevents + performance cost | ⬜ |
| Chunk 19 | B-Tree indexes — composite index rules + over-indexing trap | ⬜ |
| Chunk 20 | When SQL wins — exact conditions + interview framing | ⬜ |
| 🔥 | **Topic 5 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 6 — NoSQL Types & Tradeoffs *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 21 | Key-Value stores — Redis, DynamoDB internals + tradeoffs vs SQL | ⬜ |
| Chunk 22 | Document stores — MongoDB, when flexible schema helps vs hurts | ⬜ |
| Chunk 23 | Wide-column stores — Cassandra write path + tunable consistency | ⬜ |
| Chunk 24 | Graph DBs + master decision framework — pick DB for any use case | ⬜ |
| 🔥 | **Topic 6 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 7 — CAP Theorem & PACELC *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 25 | CAP — what it actually says + what P really means | ⬜ |
| Chunk 26 | CP vs AP — real system examples + when to choose each | ⬜ |
| Chunk 27 | PACELC — latency vs consistency in normal operation | ⬜ |
| Chunk 28 | Tunable consistency — Cassandra ONE / QUORUM / ALL | ⬜ |
| 🔥 | **Topic 7 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 8 — Indexing: B-Trees vs LSM Trees *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 29 | B-Tree write path — in-place updates, WAL, why heavy writes hurt | ⬜ |
| Chunk 30 | LSM write path — MemTable → SSTable, sequential writes | ⬜ |
| Chunk 31 | Compaction, bloom filters, write amplification — LSM tradeoffs | ⬜ |
| Chunk 32 | Real DB mapping + how to bring this up naturally in interviews | ⬜ |
| 🔥 | **Topic 8 Drill Session** — 5-8 questions | ⬜ |

---

## 🧠 Phase 2 Review — 10-12 questions across all topics

---
---

# PHASE 3 — Distributed Systems
**Estimated Time: 10 hours**

---

## Topic 9 — Replication *(2.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 33 | Why replicate — durability vs read scaling | ⬜ |
| Chunk 34 | Single-leader — sync vs async + what you gain and lose | ⬜ |
| Chunk 35 | Replication lag — read-after-write, monotonic reads + fixes | ⬜ |
| Chunk 36 | Multi-leader + leaderless quorum — when and why | ⬜ |
| Chunk 37 | Conflict resolution — LWW vs vector clocks + tradeoffs | ⬜ |
| 🔥 | **Topic 9 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 10 — Sharding & Partitioning *(2.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 38 | Why sharding — signals that tell you it's time | ⬜ |
| Chunk 39 | Hash vs range partitioning — tradeoffs | ⬜ |
| Chunk 40 | Consistent hashing + virtual nodes — how it solves rebalancing | ⬜ |
| Chunk 41 | Hot spot + celebrity problem — 3 solutions with tradeoffs | ⬜ |
| Chunk 42 | Cross-shard limitations + Snowflake IDs | ⬜ |
| 🔥 | **Topic 10 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 11 — Consistency Models *(2h)*

> **Every consistency model follows this format:**
> real story → simple → technical → implementation →
> real system example → tradeoff → when to pick → interview framing

| Chunk | Title | Status |
|---|---|---|
| Chunk 43 | Linearizability — implementation via consensus, Zookeeper example | ⬜ |
| Chunk 44 | Causal consistency — vector clocks implementation, WhatsApp example | ⬜ |
| Chunk 45 | Read-your-writes + monotonic reads — implementation, Razorpay example | ⬜ |
| Chunk 46 | Eventual consistency — implementation via gossip, Cassandra example | ⬜ |
| 🔥 | **Topic 11 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 12 — Distributed Transactions & Saga *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 47 | Why transactions across services are fundamentally hard | ⬜ |
| Chunk 48 | Two-phase commit — mechanism + coordinator failure problem | ⬜ |
| Chunk 49 | Saga pattern — choreography vs orchestration with Swiggy example | ⬜ |
| Chunk 50 | Idempotency keys — implementation + real payment example | ⬜ |
| 🔥 | **Topic 12 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 13 — Leader Election & Failure Detection *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 51 | Why leader election — the split-brain problem | ⬜ |
| Chunk 52 | Raft algorithm — step by step, simple to technical | ⬜ |
| Chunk 53 | Zookeeper — distributed coordination + real use cases | ⬜ |
| Chunk 54 | Gossip protocol — failure detection at scale, Cassandra example | ⬜ |
| 🔥 | **Topic 13 Drill Session** — 5-8 questions | ⬜ |

---

## 🧠 Phase 3 Review — 10-12 questions across all topics

---
---

# PHASE 4 — Concurrency in Systems
**Estimated Time: 5 hours**

---

## Topic 14 — Concurrency at the Database Layer *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 55 | Lost update problem — two users editing same record | ⬜ |
| Chunk 56 | Optimistic locking — version numbers, implementation + when it breaks | ⬜ |
| Chunk 57 | Pessimistic locking — SELECT FOR UPDATE, deadlock risk, tradeoffs | ⬜ |
| Chunk 58 | MVCC — how Postgres handles concurrent reads without locking | ⬜ |
| 🔥 | **Topic 14 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 15 — Concurrency at Application & Distributed Layer *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 59 | Distributed locks — Redis SETNX + TTL + Redlock algorithm | ⬜ |
| Chunk 60 | Idempotency in APIs — implementation, why POST /pay called twice must not charge twice | ⬜ |
| Chunk 61 | CAS (Compare-And-Swap) — atomic counters, implementation | ⬜ |
| Chunk 62 | Rate limiting as concurrency control | ⬜ |
| 🔥 | **Topic 15 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 16 — Distributed Counting *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 63 | Why counting is hard at scale — the simple explanation | ⬜ |
| Chunk 64 | Approximate counting — HyperLogLog, Count-Min Sketch | ⬜ |
| Chunk 65 | Exact counting — Redis INCR + sharded counters | ⬜ |
| 🔥 | **Topic 16 Drill Session** — 5-8 questions | ⬜ |

---

## 🧠 Phase 4 Review — 10-12 questions across all topics

---
---

# PHASE 5 — API Design & Architecture Patterns
**Estimated Time: 5 hours**

---

## Topic 17 — API Design: REST vs GraphQL vs gRPC *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 66 | What an API is — simple to technical | ⬜ |
| Chunk 67 | REST — principles, HTTP verbs, status codes, idempotency | ⬜ |
| Chunk 68 | GraphQL — how it works, over-fetching problem it solves, tradeoffs | ⬜ |
| Chunk 69 | gRPC — protobuf, binary protocol, when it beats REST | ⬜ |
| Chunk 70 | Decision framework — REST vs GraphQL vs gRPC for any use case | ⬜ |
| 🔥 | **Topic 17 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 18 — Microservices vs Monolith *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 71 | Monolith — what it is, why companies start here, when it breaks | ⬜ |
| Chunk 72 | Microservices — what you gain, what you lose, hidden costs | ⬜ |
| Chunk 73 | Service decomposition — how to split a monolith, strangler fig pattern | ⬜ |
| Chunk 74 | When NOT to use microservices — the honest tradeoff | ⬜ |
| 🔥 | **Topic 18 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 19 — API Gateway & Service Discovery *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 75 | API Gateway — what it does, what lives there vs in services | ⬜ |
| Chunk 76 | Service discovery — how 50 microservices find each other | ⬜ |
| Chunk 77 | Client-side vs server-side discovery — tradeoffs | ⬜ |
| Chunk 78 | Service mesh — what it is, when you need it (Istio, Envoy) | ⬜ |
| 🔥 | **Topic 19 Drill Session** — 5-8 questions | ⬜ |

---

## 🧠 Phase 5 Review — 10-12 questions across all topics

---
---

# PHASE 6 — Performance Layer
**Estimated Time: 7 hours**

---

## Topic 20 — Caching Strategies *(2.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 79 | Why cache — what problem it solves + where it sits in the stack | ⬜ |
| Chunk 80 | Cache-aside — implementation, flow, consistency tradeoff | ⬜ |
| Chunk 81 | Write-through vs write-behind — consistency vs latency tradeoff | ⬜ |
| Chunk 82 | Cache stampede, penetration, avalanche — each with exact fix + tradeoff | ⬜ |
| Chunk 83 | Eviction policies — LRU vs LFU vs TTL + which use case needs which | ⬜ |
| 🔥 | **Topic 20 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 21 — CDN & Load Balancing *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 84 | CDN — what it caches, push vs pull tradeoffs | ⬜ |
| Chunk 85 | LB algorithms — round robin, least connections, IP hash | ⬜ |
| Chunk 86 | L4 vs L7 LB — tradeoffs + which to use when | ⬜ |
| Chunk 87 | Circuit breaker pattern — preventing cascade failures | ⬜ |
| 🔥 | **Topic 21 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 22 — Rate Limiting *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 88 | What rate limiting protects — abuse, DoS, cost control | ⬜ |
| Chunk 89 | Token bucket + leaky bucket — tradeoffs, burst handling | ⬜ |
| Chunk 90 | Fixed window problem + sliding window fix | ⬜ |
| Chunk 91 | Distributed rate limiting — Redis INCR implementation + race condition | ⬜ |
| 🔥 | **Topic 22 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 23 — High Traffic Patterns *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 92 | High Read Traffic — read replicas, caching layers, CDN strategy | ⬜ |
| Chunk 93 | High Write Traffic — write buffering, async writes, queue-based leveling | ⬜ |
| Chunk 94 | Traffic Spikes — autoscaling, load shedding, backpressure | ⬜ |
| Chunk 95 | Backpressure — what it is, how to implement it, tradeoffs | ⬜ |
| 🔥 | **Topic 23 Drill Session** — 5-8 questions | ⬜ |

---

## 🧠 Phase 6 Review — 10-12 questions across all topics

---
---

# PHASE 7 — Messaging & Async
**Estimated Time: 5 hours**

---

## Topic 24 — Message Queues vs Kafka *(2.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 96 | Why async — what it decouples, what it complicates | ⬜ |
| Chunk 97 | Traditional MQ (RabbitMQ, SQS) — internals + tradeoffs | ⬜ |
| Chunk 98 | Kafka internals — topics, partitions, offsets, consumer groups | ⬜ |
| Chunk 99 | Delivery guarantees — at-most, at-least, exactly-once + real cost | ⬜ |
| Chunk 100 | MQ vs Kafka decision framework — exact use case mapping | ⬜ |
| 🔥 | **Topic 24 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 25 — Async Patterns & Event-Driven Design *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 101 | Outbox pattern — dual-write problem + how outbox solves it | ⬜ |
| Chunk 102 | CQRS — why separate read/write models + consistency tradeoff | ⬜ |
| Chunk 103 | Event sourcing — append-only log, replay, tradeoffs vs CRUD | ⬜ |
| Chunk 104 | Dead letter queues — why non-negotiable in production | ⬜ |
| Chunk 105 | Deduplicating data — idempotency keys + bloom filters | ⬜ |
| 🔥 | **Topic 25 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 26 — Batch vs Stream Processing *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 106 | Batch processing — what it is, MapReduce, Spark basics | ⬜ |
| Chunk 107 | Stream processing — Flink, Kafka Streams, real-time aggregation | ⬜ |
| Chunk 108 | Lambda architecture — batch + speed layer + serving layer | ⬜ |
| Chunk 109 | Kappa architecture — stream-only, simpler, tradeoffs vs Lambda | ⬜ |
| 🔥 | **Topic 26 Drill Session** — 5-8 questions | ⬜ |

---

## 🧠 Phase 7 Review — 10-12 questions across all topics

---
---

# PHASE 8 — Storage & Data Systems
**Estimated Time: 4 hours**

---

## Topic 27 — Storage Types *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 110 | Block storage — what it is, EBS, when to use | ⬜ |
| Chunk 111 | Object storage — S3, GCS, how it works, tradeoffs vs block | ⬜ |
| Chunk 112 | File storage — EFS, NFS, shared file systems, tradeoffs | ⬜ |
| Chunk 113 | Decision framework — which storage for which use case | ⬜ |
| 🔥 | **Topic 27 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 28 — Search Systems *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 114 | How full-text search works — inverted index, simple to technical | ⬜ |
| Chunk 115 | Elasticsearch internals — shards, replicas, analyzer pipeline | ⬜ |
| Chunk 116 | Search vs DB query — when to add a search layer | ⬜ |
| Chunk 117 | Keeping search in sync with DB — CDC + Kafka pipeline | ⬜ |
| 🔥 | **Topic 28 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 29 — Data Modeling *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 118 | Normalization vs denormalization — tradeoffs | ⬜ |
| Chunk 119 | Schema design for read-heavy vs write-heavy systems | ⬜ |
| Chunk 120 | Designing for access patterns — NoSQL schema design rules | ⬜ |
| Chunk 121 | Database migrations — zero-downtime strategies | ⬜ |
| 🔥 | **Topic 29 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 30 — Connection Pooling *(0.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 122 | What connection pooling is — simple to technical | ⬜ |
| Chunk 123 | HikariCP (Java), PgBouncer — how they work + configuration | ⬜ |
| Chunk 124 | Connection pool exhaustion — what happens + how to prevent | ⬜ |
| 🔥 | **Topic 30 Drill Session** — 3-5 questions | ⬜ |

---

## 🧠 Phase 8 Review — 10-12 questions across all topics

---
---

# PHASE 9 — Classic HLD Problems
**Estimated Time: 8 hours**

> Every HLD problem follows this format:
> Requirements → Estimation → High-level design →
> Deep dive on 2 components → Tradeoffs → Full design drill

---

## Topic 31 — URL Shortener *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 125 | Requirements + estimation | ⬜ |
| Chunk 126 | ID generation — base62, Snowflake, tradeoffs of each | ⬜ |
| Chunk 127 | 301 vs 302 redirect — analytics vs performance tradeoff | ⬜ |
| 🔥 | **Full Design Drill** — 45-min simulation | ⬜ |

---

## Topic 32 — News Feed (Twitter/Instagram) *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 128 | Fan-out on write vs read — full tradeoff analysis | ⬜ |
| Chunk 129 | Celebrity problem — 3 solutions + tradeoffs of each | ⬜ |
| Chunk 130 | Feed ranking — chronological vs ML-ranked tradeoff | ⬜ |
| 🔥 | **Full Design Drill** — 45-min simulation | ⬜ |

---

## Topic 33 — Chat System + Notifications *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 131 | Short polling vs long polling vs SSE vs WebSocket — full comparison | ⬜ |
| Chunk 132 | Message ordering + offline delivery — consistency tradeoff | ⬜ |
| Chunk 133 | Notification pipeline — multi-channel, retry, DLQ | ⬜ |
| 🔥 | **Full Design Drill** — 45-min simulation | ⬜ |

---

## Topic 34 — Search Autocomplete + KV Store *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 134 | Trie + top-K precomputation — freshness vs performance tradeoff | ⬜ |
| Chunk 135 | KV store — consistent hashing + gossip protocol | ⬜ |
| 🔥 | **Full Design Drill** — 45-min simulation | ⬜ |

---

## Topic 35 — Handling Large Files + Media Streaming *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 136 | Chunked upload + resumable uploads — how S3 multipart works | ⬜ |
| Chunk 137 | Media streaming — HLS, adaptive bitrate streaming | ⬜ |
| Chunk 138 | CDN strategy for video — edge caching, cache invalidation | ⬜ |
| 🔥 | **Full Design Drill** — 45-min simulation | ⬜ |

---

## Topic 36 — Location Data + Recommendations *(0.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 139 | Geohashing + quadtrees — proximity search, Uber driver matching | ⬜ |
| Chunk 140 | Recommendation system — collaborative filtering vs content-based | ⬜ |
| 🔥 | **Full Design Drill** — 45-min simulation | ⬜ |

---

## 🧠 Phase 9 Review — 10-12 questions across all topics

---
---

# PHASE 10 — Observability, Reliability & Operations
**Estimated Time: 6 hours**

---

## Topic 37 — Monitoring, Alerting & Observability *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 141 | The 3 pillars — Metrics, Logs, Traces — simple to technical | ⬜ |
| Chunk 142 | RED method (Rate, Errors, Duration) + USE method | ⬜ |
| Chunk 143 | Alerting — P1 vs P2, alert fatigue, on-call design | ⬜ |
| Chunk 144 | SLI, SLO, SLA — definitions + how to set them | ⬜ |
| Chunk 145 | Dashboards — what good looks like vs vanity metrics | ⬜ |
| 🔥 | **Topic 37 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 38 — Deployment & Release Strategies *(2h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 146 | Why deployments fail — the simple explanation | ⬜ |
| Chunk 147 | Blue-green deployment — how it works + tradeoffs | ⬜ |
| Chunk 148 | Canary releases — gradual rollout + rollback strategy | ⬜ |
| Chunk 149 | Feature flags — what they are, branching strategy, tradeoffs | ⬜ |
| Chunk 150 | Git branching strategies — trunk-based vs GitFlow vs feature branching | ⬜ |
| Chunk 151 | Rolling deployments vs recreate — tradeoffs | ⬜ |
| 🔥 | **Topic 38 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 39 — Handling Failures & Resilience *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 152 | Retry strategies — when to retry, exponential backoff + jitter | ⬜ |
| Chunk 153 | Bulkhead pattern — isolating failures so they don't spread | ⬜ |
| Chunk 154 | Chaos engineering — intentionally breaking things, Netflix example | ⬜ |
| Chunk 155 | Removing SPOFs — active-active vs active-passive redundancy | ⬜ |
| 🔥 | **Topic 39 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 40 — Multi-Tenancy, Multi-Region & Cost *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 156 | Multi-tenancy — shared vs isolated, noisy neighbor problem | ⬜ |
| Chunk 157 | Tenant isolation strategies — DB per tenant vs schema vs row-level | ⬜ |
| Chunk 158 | Multi-region architecture — data residency, latency routing | ⬜ |
| Chunk 159 | Active-active vs active-passive globally — conflict resolution | ⬜ |
| Chunk 160 | Cost optimization — right-sizing, spot instances, tiered storage | ⬜ |
| 🔥 | **Topic 40 Drill Session** — 5-8 questions | ⬜ |

---

## 🧠 Phase 10 Review — 10-12 questions across all topics

---
---

# PHASE 11 — Security
**Estimated Time: 5 hours**

---

## Topic 41 — Authentication & Authorization *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 161 | Authentication vs Authorization — simple to technical | ⬜ |
| Chunk 162 | JWT vs Session tokens — internals, how signing works, tradeoffs | ⬜ |
| Chunk 163 | OAuth 2.0 + OpenID Connect — full flow step by step | ⬜ |
| Chunk 164 | HTTPS + TLS — what it protects, what it doesn't | ⬜ |
| 🔥 | **Topic 41 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 42 — Common Attack Vectors *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 165 | SQL Injection + XSS — how attacks work + prevention | ⬜ |
| Chunk 166 | CSRF attacks — how it works + prevention | ⬜ |
| Chunk 167 | DDoS — types + mitigation strategies | ⬜ |
| Chunk 168 | Man-in-the-middle attacks — prevention | ⬜ |
| 🔥 | **Topic 42 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 43 — Data Security *(1h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 169 | Encryption at rest vs in transit — tradeoffs | ⬜ |
| Chunk 170 | Hashing passwords — bcrypt, salt, why MD5 is dead | ⬜ |
| Chunk 171 | API security — rate limiting, API keys, secrets management | ⬜ |
| Chunk 172 | Zero trust architecture — never trust, always verify | ⬜ |
| 🔥 | **Topic 43 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 44 — Token Security & Compromise Handling *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 173 | What happens when a JWT is stolen — attack scenarios | ⬜ |
| Chunk 174 | Token revocation — blacklisting, short expiry + refresh tokens | ⬜ |
| Chunk 175 | Refresh token rotation — how it works + tradeoffs | ⬜ |
| Chunk 176 | API key compromise — detection, rotation, blast radius reduction | ⬜ |
| Chunk 177 | Secret management — Vault, AWS Secrets Manager, never hardcode | ⬜ |
| Chunk 178 | Detecting stolen tokens — anomaly detection, geo-velocity, device fingerprinting | ⬜ |
| Chunk 179 | PII data handling — masking, tokenization, GDPR basics | ⬜ |
| Chunk 180 | Audit logs — why they matter + how to design them | ⬜ |
| 🔥 | **Topic 44 Drill Session** — 5-8 questions | ⬜ |

---

## 🧠 Phase 11 Review — 10-12 questions across all topics

---
---

# PHASE 12 — Interview Execution
**Estimated Time: 3 hours**

---

## Topic 45 — How to Discuss Tradeoffs *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 181 | What interviewers actually score — the rubric they use | ⬜ |
| Chunk 182 | The tradeoff formula | ⬜ |
| Chunk 183 | The 8 universal tradeoff axes | ⬜ |
| Chunk 184 | First-principles comparison — two technologies you've never seen | ⬜ |
| Chunk 185 | Tradeoff mistakes that kill interviews | ⬜ |
| 🔥 | **Topic 45 Drill Session** — 5-8 questions | ⬜ |

---

## Topic 46 — The 45-Min HLD Framework *(1.5h)*

| Chunk | Title | Status |
|---|---|---|
| Chunk 186 | 5-phase structure with exact time splits | ⬜ |
| Chunk 187 | Clarification questions that signal seniority | ⬜ |
| Chunk 188 | How to recover when you don't know something | ⬜ |
| Chunk 189 | Red flags that instantly lower your score | ⬜ |
| 🔥 | **Topic 46 Full Mock Interview** — 45-min simulation | ⬜ |
| 🎯 | **Final Mock Interview #2** — different problem, full simulation | ⬜ |
| 🎯 | **Final Mock Interview #3** — interviewer challenges every decision | ⬜ |

---

## 🧠 Phase 12 Final Assessment — across ALL 12 phases

---
---

# 📋 Reference — The 8 Universal Tradeoff Axes

| # | Axis | What you're choosing between |
|---|---|---|
| 1 | Consistency vs Availability | Correct data vs system staying up |
| 2 | Latency vs Throughput | Fast for one vs fast for many |
| 3 | Cost vs Performance | Cheap vs fast |
| 4 | Simplicity vs Flexibility | Easy to understand vs easy to extend |
| 5 | Read optimization vs Write optimization | Fast reads vs fast writes |
| 6 | Coupling vs Scalability | Easy to build vs easy to scale |
| 7 | Durability vs Speed | Never lose data vs respond fast |
| 8 | Accuracy vs Performance | Exact answer vs approximate fast answer |

---

# 🗣️ The Tradeoff Formula

```
"We chose [X] over [Y].
 We gain [benefit A].
 We lose [cost B].
 That's acceptable because [reason C — specific to this system]."
```

**Example:**
> "We chose Cassandra over PostgreSQL.
>  We gain horizontal scalability and high write throughput.
>  We lose strong consistency and join support.
>  That's acceptable because our activity log is write-heavy
>  and eventual consistency is fine — no one needs a like count
>  accurate to the millisecond."

---

# ✅ Complete Checklist — tick as you go

## Phase 1 — Foundations
- [ ] Chunk 1 — Reliability vs Scalability vs Maintainability
- [ ] Chunk 2 — Latency vs Throughput
- [ ] Chunk 3 — SPOF + HA vs Fault Tolerance
- [ ] Chunk 4 — Interview mental model
- [ ] Chunk 5 — Key latency numbers
- [ ] Chunk 6 — DAU → QPS → Storage
- [ ] Chunk 7 — Bandwidth + Peak traffic
- [ ] Chunk 8 — Worked estimation example
- [ ] Chunk 9 — Vertical scaling
- [ ] Chunk 10 — Horizontal scaling
- [ ] Chunk 10B — Stateful vs Stateless servers
- [ ] Chunk 11 — Load balancer role
- [ ] Chunk 12 — Scale up vs scale out framework
- [ ] Chunk 13 — Race conditions + dirty reads
- [ ] Chunk 14 — Optimistic vs pessimistic locking
- [ ] Chunk 15 — Distributed locks intro
- [ ] Chunk 16 — Deadlocks

## Phase 2 — Databases
- [ ] Chunk 17 — ACID
- [ ] Chunk 18 — 4 isolation levels
- [ ] Chunk 19 — B-Tree indexes
- [ ] Chunk 20 — When SQL wins
- [ ] Chunk 21 — Key-Value stores
- [ ] Chunk 22 — Document stores
- [ ] Chunk 23 — Wide-column stores
- [ ] Chunk 24 — Graph DBs + decision framework
- [ ] Chunk 25 — CAP theorem
- [ ] Chunk 26 — CP vs AP
- [ ] Chunk 27 — PACELC
- [ ] Chunk 28 — Tunable consistency
- [ ] Chunk 29 — B-Tree write path
- [ ] Chunk 30 — LSM write path
- [ ] Chunk 31 — Compaction + bloom filters
- [ ] Chunk 32 — Real DB mapping

## Phase 3 — Distributed Systems
- [ ] Chunk 33 — Why replicate
- [ ] Chunk 34 — Single-leader replication
- [ ] Chunk 35 — Replication lag
- [ ] Chunk 36 — Multi-leader + leaderless
- [ ] Chunk 37 — Conflict resolution
- [ ] Chunk 38 — Why sharding
- [ ] Chunk 39 — Hash vs range partitioning
- [ ] Chunk 40 — Consistent hashing + vnodes
- [ ] Chunk 41 — Hot spot + celebrity problem
- [ ] Chunk 42 — Cross-shard + Snowflake IDs
- [ ] Chunk 43 — Linearizability
- [ ] Chunk 44 — Causal consistency
- [ ] Chunk 45 — Read-your-writes + monotonic reads
- [ ] Chunk 46 — Eventual consistency
- [ ] Chunk 47 — Why distributed txns are hard
- [ ] Chunk 48 — Two-phase commit
- [ ] Chunk 49 — Saga pattern
- [ ] Chunk 50 — Idempotency keys
- [ ] Chunk 51 — Leader election + split-brain
- [ ] Chunk 52 — Raft algorithm
- [ ] Chunk 53 — Zookeeper
- [ ] Chunk 54 — Gossip protocol

## Phase 4 — Concurrency
- [ ] Chunk 55 — Lost update problem
- [ ] Chunk 56 — Optimistic locking
- [ ] Chunk 57 — Pessimistic locking
- [ ] Chunk 58 — MVCC
- [ ] Chunk 59 — Distributed locks (Redis)
- [ ] Chunk 60 — Idempotency in APIs
- [ ] Chunk 61 — CAS operations
- [ ] Chunk 62 — Rate limiting as concurrency control
- [ ] Chunk 63 — Why counting is hard
- [ ] Chunk 64 — Approximate counting
- [ ] Chunk 65 — Exact counting

## Phase 5 — API & Architecture
- [ ] Chunk 66 — What an API is
- [ ] Chunk 67 — REST
- [ ] Chunk 68 — GraphQL
- [ ] Chunk 69 — gRPC
- [ ] Chunk 70 — API decision framework
- [ ] Chunk 71 — Monolith
- [ ] Chunk 72 — Microservices
- [ ] Chunk 73 — Service decomposition
- [ ] Chunk 74 — When NOT to use microservices
- [ ] Chunk 75 — API Gateway
- [ ] Chunk 76 — Service discovery
- [ ] Chunk 77 — Client vs server-side discovery
- [ ] Chunk 78 — Service mesh

## Phase 6 — Performance
- [ ] Chunk 79 — Why cache
- [ ] Chunk 80 — Cache-aside
- [ ] Chunk 81 — Write-through vs write-behind
- [ ] Chunk 82 — Cache failure modes
- [ ] Chunk 83 — Eviction policies
- [ ] Chunk 84 — CDN
- [ ] Chunk 85 — LB algorithms
- [ ] Chunk 86 — L4 vs L7 LB
- [ ] Chunk 87 — Circuit breaker
- [ ] Chunk 88 — What rate limiting protects
- [ ] Chunk 89 — Token bucket + leaky bucket
- [ ] Chunk 90 — Fixed + sliding window
- [ ] Chunk 91 — Distributed rate limiting
- [ ] Chunk 92 — High read traffic
- [ ] Chunk 93 — High write traffic
- [ ] Chunk 94 — Traffic spikes
- [ ] Chunk 95 — Backpressure

## Phase 7 — Messaging
- [ ] Chunk 96 — Why async
- [ ] Chunk 97 — Traditional MQ
- [ ] Chunk 98 — Kafka internals
- [ ] Chunk 99 — Delivery guarantees
- [ ] Chunk 100 — MQ vs Kafka framework
- [ ] Chunk 101 — Outbox pattern
- [ ] Chunk 102 — CQRS
- [ ] Chunk 103 — Event sourcing
- [ ] Chunk 104 — Dead letter queues
- [ ] Chunk 105 — Deduplication
- [ ] Chunk 106 — Batch processing
- [ ] Chunk 107 — Stream processing
- [ ] Chunk 108 — Lambda architecture
- [ ] Chunk 109 — Kappa architecture

## Phase 8 — Storage & Data
- [ ] Chunk 110 — Block storage
- [ ] Chunk 111 — Object storage
- [ ] Chunk 112 — File storage
- [ ] Chunk 113 — Storage decision framework
- [ ] Chunk 114 — Full-text search + inverted index
- [ ] Chunk 115 — Elasticsearch internals
- [ ] Chunk 116 — Search vs DB query
- [ ] Chunk 117 — DB to search sync
- [ ] Chunk 118 — Normalization vs denormalization
- [ ] Chunk 119 — Schema design
- [ ] Chunk 120 — Access pattern design
- [ ] Chunk 121 — Zero-downtime migrations
- [ ] Chunk 122 — Connection pooling
- [ ] Chunk 123 — HikariCP + PgBouncer
- [ ] Chunk 124 — Pool exhaustion

## Phase 9 — HLD Problems
- [ ] Chunk 125-127 — URL Shortener
- [ ] Chunk 128-130 — News Feed
- [ ] Chunk 131-133 — Chat + Notifications
- [ ] Chunk 134-135 — Autocomplete + KV Store
- [ ] Chunk 136-138 — Large Files + Streaming
- [ ] Chunk 139-140 — Location + Recommendations

## Phase 10 — Observability & Operations
- [ ] Chunk 141 — Metrics, Logs, Traces
- [ ] Chunk 142 — RED + USE methods
- [ ] Chunk 143 — Alerting
- [ ] Chunk 144 — SLI, SLO, SLA
- [ ] Chunk 145 — Dashboards
- [ ] Chunk 146 — Why deployments fail
- [ ] Chunk 147 — Blue-green
- [ ] Chunk 148 — Canary releases
- [ ] Chunk 149 — Feature flags
- [ ] Chunk 150 — Git branching strategies
- [ ] Chunk 151 — Rolling deployments
- [ ] Chunk 152 — Retry + backoff
- [ ] Chunk 153 — Bulkhead pattern
- [ ] Chunk 154 — Chaos engineering
- [ ] Chunk 155 — Removing SPOFs
- [ ] Chunk 156 — Multi-tenancy
- [ ] Chunk 157 — Tenant isolation
- [ ] Chunk 158 — Multi-region
- [ ] Chunk 159 — Active-active globally
- [ ] Chunk 160 — Cost optimization

## Phase 11 — Security
- [ ] Chunk 161 — AuthN vs AuthZ
- [ ] Chunk 162 — JWT vs Sessions
- [ ] Chunk 163 — OAuth 2.0 + OIDC
- [ ] Chunk 164 — HTTPS + TLS
- [ ] Chunk 165 — SQL injection + XSS
- [ ] Chunk 166 — CSRF
- [ ] Chunk 167 — DDoS
- [ ] Chunk 168 — MITM attacks
- [ ] Chunk 169 — Encryption at rest vs transit
- [ ] Chunk 170 — Password hashing
- [ ] Chunk 171 — API security
- [ ] Chunk 172 — Zero trust
- [ ] Chunk 173 — JWT stolen
- [ ] Chunk 174 — Token revocation
- [ ] Chunk 175 — Refresh token rotation
- [ ] Chunk 176 — API key compromise
- [ ] Chunk 177 — Secret management
- [ ] Chunk 178 — Stolen token detection
- [ ] Chunk 179 — PII handling
- [ ] Chunk 180 — Audit logs

## Phase 12 — Interview Execution
- [ ] Chunk 181 — What interviewers score
- [ ] Chunk 182 — Tradeoff formula
- [ ] Chunk 183 — 8 tradeoff axes
- [ ] Chunk 184 — First-principles comparison
- [ ] Chunk 185 — Tradeoff mistakes
- [ ] Chunk 186 — 45-min structure
- [ ] Chunk 187 — Clarification questions
- [ ] Chunk 188 — Recovery when stuck
- [ ] Chunk 189 — Red flags to avoid

---

# ⏱️ Time Allocation Summary

| Phase | Topics | Chunks | Hours |
|---|---|---|---|
| Phase 1 — Foundations | 4 | 17 | 7h |
| Phase 2 — Databases | 4 | 16 | 8h |
| Phase 3 — Distributed Systems | 5 | 22 | 10h |
| Phase 4 — Concurrency | 3 | 11 | 5h |
| Phase 5 — API & Architecture | 3 | 13 | 5h |
| Phase 6 — Performance Layer | 4 | 17 | 7h |
| Phase 7 — Messaging & Async | 3 | 14 | 5h |
| Phase 8 — Storage & Data | 4 | 15 | 4h |
| Phase 9 — Classic HLD Problems | 6 | 16 | 8h |
| Phase 10 — Observability & Ops | 4 | 20 | 6h |
| Phase 11 — Security | 4 | 20 | 5h |
| Phase 12 — Interview Execution | 2 | 9 | 3h |
| **TOTAL** | **46** | **190** | **~73h** |

---

*Target: SDE-2 at Razorpay, PhonePe, Flipkart, Uber, Microsoft, Google India*
*Strategy: Start lower-tier companies → mid-tier → top-tier*
*Revision: Use chunk numbers to drill any topic anytime*