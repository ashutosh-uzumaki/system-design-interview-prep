# Topic 3 — Vertical vs Horizontal Scaling + Stateful vs Stateless + Load Balancing
## Complete Notes | Concepts + Interview Q&A + Technical Terms + Drill Q&A

---

## 📊 Topic 3 Stats
- **Chunks covered:** 9, 10, 10B, 11, 12
- **Drill score:** 30/40
- **Status:** ✅ Complete

---

---

# CHUNK 9 — Vertical Scaling (Scale-Up)

## Core Concept

```
VERTICAL SCALING — SIMPLE VERSION
────────────────────────────────────────────────────
Adding more resources to your EXISTING server

More CPU   → handles more parallel requests
More RAM   → fits more data in memory
Faster SSD → reads/writes data quicker

Same single machine — just more powerful
────────────────────────────────────────────────────
```

---

## The Cost Curve — Critical

```
VERTICAL SCALING COST CURVE
────────────────────────────────────────────────────

Cost
 ▲
 │                              ● (very expensive)
 │                         ●
 │                    ●
 │               ●
 │          ●
 │     ●
 │●
 └──────────────────────────────► Power

Doubling power does NOT double cost
It grows EXPONENTIALLY

AWS EC2 Example:
  4 cores,  16GB RAM  → ₹7,000/month
  8 cores,  32GB RAM  → ₹14,000/month  (2× cost, 2× power) ✅
  32 cores, 128GB RAM → ₹75,000/month  (10× cost, 8× power) ❌
  64 cores, 256GB RAM → ₹2,00,000/month (28× cost, 16× power) ❌

→ After a point you pay MORE for LESS gain
────────────────────────────────────────────────────
```

---

## Four Fundamental Problems

```
PROBLEM 1 — EXPONENTIAL COST
──────────────────────────────
4× resources ≠ 4× cost
Likely 8-10× cost at higher tiers
Not cost efficient at scale

PROBLEM 2 — HARD LIMIT
────────────────────────
Every machine has maximum size
AWS largest instance today:
  448 vCPUs, 24 TB RAM
After that → CANNOT scale up anymore
Forced to scale horizontally

PROBLEM 3 — SPOF REMAINS
──────────────────────────
One giant powerful server = one giant SPOF
Server crashes → everything down
No redundancy, no failover

PROBLEM 4 — DOWNTIME TO UPGRADE
──────────────────────────────────
Must shut down server to resize
= Planned maintenance window
= Unacceptable during peak traffic
Horizontal scaling = zero downtime scaling
```

---

## When to Use Vertical Scaling

```
USE VERTICAL SCALING WHEN          AVOID WHEN
──────────────────────────         ──────────
✅ Early stage startup             ❌ Need HA (24/7)
✅ Quick fix for traffic spike     ❌ Traffic > single machine
✅ Stateful legacy databases       ❌ Cost becomes exponential
✅ Small team, simple architecture ❌ Global distribution needed
✅ Monolith that can't distribute  ❌ Need auto-scaling
```

---

## Real World Example

```
EARLY FLIPKART — THE LESSON
────────────────────────────────────────────────────
Started on single powerful server
First Big Billion Day → server crashed
Couldn't scale up fast enough
→ Forced to move to horizontal scaling
→ Biggest lesson in Indian startup history

LESSON: Vertical scaling is a temporary fix
        Not a permanent solution
────────────────────────────────────────────────────
```

---

## Technical Terms

| Plain English | Technical Term |
|---|---|
| Bigger machine, same number | Vertical Scaling / Scale-Up |
| Cost grows faster than power | Exponential cost curve |
| Maximum machine size | Hard limit |
| Scheduled downtime for upgrade | Maintenance window |
| One component = system down | Single Point of Failure (SPOF) |
| Having more capacity than needed | Over-provisioning |

---
---

# CHUNK 10 — Horizontal Scaling (Scale-Out) + Statelessness

## Core Concept

```
HORIZONTAL SCALING — SIMPLE VERSION
────────────────────────────────────────────────────
Adding MORE machines to your pool
instead of making one machine bigger

Each server is IDENTICAL and INTERCHANGEABLE
Load balancer distributes requests across all
Any server can handle any request

KEY REQUIREMENT: Servers must be STATELESS
────────────────────────────────────────────────────
```

---

## The Setup

```
HORIZONTAL SCALING ARCHITECTURE
────────────────────────────────────────────────────

                    ┌─────────────┐
   Users ──────────►│ LOAD        │
                    │ BALANCER    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ SERVER 1 │ │ SERVER 2 │ │ SERVER 3 │
        │          │ │          │ │          │
        └──────────┘ └──────────┘ └──────────┘
              │            │            │
              └────────────┼────────────┘
                           ▼
                    ┌─────────────┐
                    │  DATABASE   │
                    │  + CACHE    │
                    └─────────────┘

Any server handles any request ✅
Server 2 dies → LB routes to 1 and 3 ✅
Add Server 4 → LB includes automatically ✅
No SPOF at app layer ✅
────────────────────────────────────────────────────
```

---

## The Statelessness Requirement

```
STATEFUL SERVER (BAD)                 STATELESS SERVER (GOOD)
──────────────────────                ───────────────────────
Session in server RAM                 Session in Redis

User → Server 1 → session stored      User → Server 1 → reads Redis
User → Server 2 → session missing ❌  User → Server 2 → reads Redis ✅
User logged out 😱                     User still logged in ✅

RULE: State must live OUTSIDE servers
      Never in server memory
```

---

## Where State Lives

```
STATE TYPE          WRONG (stateful)    RIGHT (stateless)
──────────          ────────────────    ─────────────────
Session data        Server RAM ❌       Redis ✅
Shopping cart       Server RAM ❌       Redis ✅
Auth token          Server RAM ❌       JWT ✅
Order status        Server RAM ❌       Redis ✅
Exchange rates      Server RAM ❌       Redis ✅
User preferences    Server RAM ❌       DB + Cache ✅
Temp calculations   Server RAM ✅       Server RAM ✅
(request scope)     (this is fine!)
```

---

## Cache Inconsistency Problem

```
CACHE INCONSISTENCY
────────────────────────────────────────────────────
Each server stores exchange rate in local memory
Refreshed every 5 minutes independently

10:00AM → All servers: rate = ₹83.20 ✅
10:02AM → Rate changes to ₹83.75
10:03AM → Server 1 refreshes: ₹83.75 ✅
           Servers 2-10: still ₹83.20 ❌

User A → Server 1 → charged ₹83.75
User B → Server 2 → charged ₹83.20
Same product, same time, different price 😱

Technical term: Cache Inconsistency
Fix: Centralise in Redis — single source of truth
────────────────────────────────────────────────────
```

---

## Vertical vs Horizontal — Final Comparison

```
                    VERTICAL        HORIZONTAL
                    ────────        ──────────
Approach            Bigger machine  More machines
Code change needed? ❌ None         ✅ Must be stateless
Cost curve          Exponential     Linear
Hard limit?         ✅ Yes          ❌ No
SPOF?               ✅ Yes          ❌ No (with LB)
Downtime to scale?  ✅ Yes          ❌ No
Auto-scaling?       ❌ Difficult    ✅ Easy
Best for            Early stage,    Internet scale,
                    DBs, quick fix  HA systems
```

---

## Spring Boot Fix — Java Specific

```java
// WRONG — Stateful (default Spring Boot)
HttpSession session;
session.setAttribute("cart", cartItems);
// Only THIS server knows this session ❌

// RIGHT — Stateless with Spring Session + Redis
// application.properties:
spring.session.store-type=redis
spring.redis.host=your-redis-host
// Now ALL sessions auto-stored in Redis ✅
// Any server reads same session ✅

// ALSO RIGHT — JWT for authentication
// Token carries user identity
// Server just validates signature
// No server-side storage needed ✅
```

---

## Technical Terms

| Plain English | Technical Term |
|---|---|
| More machines, same size | Horizontal Scaling / Scale-Out |
| Server stores nothing about users | Stateless server |
| Server remembers user data | Stateful server |
| Different servers have different data | Cache Inconsistency |
| One Redis as truth for all servers | Single source of truth |
| Scales up AND down automatically | Elasticity |
| More capacity than needed | Over-provisioning |
| Same user always hits same server | Sticky sessions (anti-pattern) |

---
---

# CHUNK 10B — Stateful vs Stateless Servers (Deep Dive)

## Core Concept

```
STATEFUL vs STATELESS — WAITER ANALOGY
────────────────────────────────────────────────────
Waiter A (Stateful):
  Remembers your preferences in his head
  You get a different waiter → starts from scratch
  Doesn't scale — each waiter must know all customers

Waiter B (Stateless):
  Remembers nothing
  All preferences in central customer book
  Any waiter checks book → serves you perfectly
  Add 100 waiters → all check same book → scales ✅

Central customer book = Redis / Database / JWT
────────────────────────────────────────────────────
```

---

## JWT — How It Makes Auth Stateless

```
TRADITIONAL SESSION (Stateful)
────────────────────────────────
User logs in
→ Server creates session ID "abc123"
→ Stores: { "abc123": { userId: 1, role: "admin" } }
→ Returns session ID to client
→ Next request: server looks up "abc123"
→ Problem: ONLY this server knows "abc123" ❌

JWT (Stateless)
────────────────
User logs in
→ Server creates token:
  { userId: 1, role: "admin", expiry: "1hr" }
→ Signs with secret key
→ Returns token to client
→ Next request: client sends token
→ ANY server validates signature ✅
→ Extracts user info from token itself ✅
→ No server-side storage needed ✅

JWT is RIGHT for STATIC data:
  ✅ userId
  ✅ user role (admin/user)
  ✅ email
  → These rarely change

JWT is WRONG for DYNAMIC data:
  ❌ Order status (changes every minute)
  ❌ Cart items (changes constantly)
  ❌ Account balance (changes per transaction)
```

---

## JWT Problems — Stale Token + Immutability

```
STALE TOKEN PROBLEM
────────────────────────────────────────────────────
JWT is IMMUTABLE after creation
Cannot be changed after issuing

Order status in JWT:
  JWT created: "Preparing"
  2 minutes later: "Out for delivery"
  JWT still says: "Preparing" ❌

Server can't PUSH new JWT to client
Client only gets new JWT on next request

→ User sees wrong order status
→ Could be minutes out of date

FIX: Never store dynamic data in JWT
     Use Redis for dynamic shared state
────────────────────────────────────────────────────
```

---

## When Stateful Is Unavoidable

```
THREE CASES WHERE STATEFUL IS NECESSARY
────────────────────────────────────────────────────

1. WebSocket connections (Chat apps)
   → Connection tied to ONE specific server
   → Can't move mid-conversation
   → Fix: Redis Pub-Sub to route between servers

2. Game servers
   → Game state (position, score) in one server
   → Too expensive to sync every frame
   → Fix: dedicated game server per session

3. Video streaming
   → Stream buffer tied to one server
   → Fix: CDN handles distribution
────────────────────────────────────────────────────
```

---

## Redis Pub-Sub for WebSocket Routing

```
PROBLEM: Chat across different servers
────────────────────────────────────────────────────
User A → connected to Server 3
User B → connected to Server 7
Server 3 cannot directly access User B's connection

SOLUTION: Redis Pub-Sub
────────────────────────────────────────────────────

SETUP:
  Server 3 → subscribes to "server-3" channel
  Server 7 → subscribes to "server-7" channel

  Redis routing table:
  { "UserB": "server-7" }
  { "UserA": "server-3" }

STEP BY STEP — User A sends "Hello" to User B:

STEP 1: User A sends via WebSocket to Server 3

STEP 2: Server 3 checks Redis routing table
        "Which server is User B on?" → "server-7"

STEP 3: Server 3 PUBLISHES to "server-7" channel
        { to: "UserB", message: "Hello" }

STEP 4: Server 7 is SUBSCRIBED to "server-7"
        Receives message instantly
        Delivers to User B's WebSocket ✅

STEP 5: Delivery receipt back via same mechanism
        User A sees ✓ (sent)
        User B reads → User A sees ✓✓ (read)

────────────────────────────────────────────────────

DIAGRAM:
                    REDIS PUB-SUB
                   ┌─────────────┐
User A             │ channel:    │             User B
  │                │ "server-7"  │                │
  │ "Hello"        └──────┬──────┘   "Hello"      │
  ▼                       │                       ▼
┌──────────┐  publish     │     deliver  ┌──────────┐
│ SERVER 3 │──────────────►──────────────►│ SERVER 7 │
│ User A   │              │              │ User B   │
│ WebSocket│              │              │ WebSocket│
└──────────┘              │              └──────────┘

────────────────────────────────────────────────────

OFFLINE USER HANDLING:
  User B not connected to any server
  → Message stored in database
  → User B comes online → connects to Server 2
  → Server 2 fetches all missed messages from DB
  → Delivers all at once
  → Updates last_seen timestamp
────────────────────────────────────────────────────
```

---

## State Decision Framework

```
WHAT TO USE FOR EACH STATE TYPE
────────────────────────────────────────────────────

STATE TYPE              USE                 WHY
──────────              ───                 ───
Session data            Redis               Shared, dynamic
Shopping cart           Redis               Changes often
Auth token              JWT                 Static user info
Order status            Redis               Changes every min
Exchange rates          Redis               Single source
User preferences        DB + Cache          Persistent
Game state              Server memory       Too frequent sync
WebSocket connection    Server memory       Can't move
                        + Redis Pub-Sub     for routing
Files/images            S3/Object storage   Large, persistent
Temp calculations       Server RAM          Request scope only

────────────────────────────────────────────────────
GOLDEN RULE:
  Server RAM  → NEVER for shared state
  Redis       → dynamic shared state
  JWT         → static auth data only
  Database    → persistent permanent data
  S3          → files, media, large objects
────────────────────────────────────────────────────
```

---
---

# CHUNK 11 — Load Balancer

## What a Load Balancer Does

```
LOAD BALANCER — THREE JOBS
────────────────────────────────────────────────────
1. Distribute requests across servers
2. Monitor server health
3. Stop sending to dead/slow servers
────────────────────────────────────────────────────
```

---

## 5 Load Balancing Algorithms

```
ALGORITHM 1 — ROUND ROBIN
──────────────────────────────────────────────────
Requests distributed in order:
1 → Server 1, 2 → Server 2, 3 → Server 3,
4 → Server 1 (starts again)...

✅ Use when: all servers same size,
             all requests similar load
❌ Problem:  heavy request on one server
             = overloaded while others free

──────────────────────────────────────────────────

ALGORITHM 2 — WEIGHTED ROUND ROBIN
────────────────────────────────────
Assign weights based on server capacity

Example:
  Server 1 → weight 1 (4 cores)
  Server 2 → weight 4 (16 cores)
  Server 3 → weight 4 (16 cores)

Every 9 requests:
  1 → Server 1, 4 → Server 2, 4 → Server 3

✅ Use when: servers have different capacities
✅ Gradual migration — new servers get more traffic
❌ Problem:  doesn't consider real-time load

Technical term: Zero downtime migration

──────────────────────────────────────────────────

ALGORITHM 3 — LEAST CONNECTIONS
──────────────────────────────────
Send to server with FEWEST active connections

Server 1 → 10 connections
Server 2 → 45 connections (heavy request)
Server 3 → 12 connections
Next request → Server 1 ✅

✅ Use when: requests have different processing times
✅ File uploads, complex orders, varying API calls
❌ Problem:  slight overhead tracking connection count

──────────────────────────────────────────────────

ALGORITHM 4 — IP HASH
──────────────────────
hash(client IP) → always same server

✅ Use when: WebSocket connections,
             stateful sessions
❌ Problem:  office WiFi = all users same IP
             → ALL routed to same server
             → HOT SPOT PROBLEM

Fix: Use SESSION ID hash instead of IP hash
     Every user has unique session ID
     Even if same IP → different session IDs
     → Distributed evenly ✅

──────────────────────────────────────────────────

ALGORITHM 5 — RANDOM WITH 2 CHOICES
──────────────────────────────────────
Pick 2 servers randomly
Compare their load
Send to less loaded one

Why 2 is enough:
  Checking all 1000 servers = perfect choice
  Checking 2 random        = 99% as good
  Checking just 1          = could be terrible

✅ Use when: large server pools (1000+ servers)
✅ Best balance of speed and accuracy
If both equal load → pick randomly (both fine)
```

---

## Algorithm Decision Cheatsheet

```
USE CASE                    ALGORITHM
────────────────────────────────────────────────────
Simple REST API             → Round Robin
Different request sizes     → Least Connections
Different server capacities → Weighted Round Robin
WebSocket connections       → Session ID Hash
                              (NOT IP Hash)
1000+ servers               → Random with 2 Choices
Gradual server migration    → Weighted Round Robin
────────────────────────────────────────────────────
```

---

## L4 vs L7 Load Balancing

```
L4 LOAD BALANCER (Layer 4 — Transport)
────────────────────────────────────────────────────
Reads: IP address + Port number ONLY
Doesn't open the request
Doesn't see URL, headers, body

Fast ✅
Cheap ✅
Can't do smart routing ❌

Use for:
→ Video streaming (raw speed needed)
→ Gaming (low latency needed)
→ TCP/UDP traffic
→ AWS NLB (Network Load Balancer)

────────────────────────────────────────────────────

L7 LOAD BALANCER (Layer 7 — Application)
────────────────────────────────────────────────────
Reads: URL path, headers, cookies, body
Opens and inspects the request
Smart routing based on content

Slightly slower ❌
More expensive ❌
Smart routing ✅
Microservices routing ✅

Use for:
→ REST APIs
→ Microservices (route by URL path)
→ HTTP traffic
→ AWS ALB (Application Load Balancer)

Example — Swiggy with L7:
  /api/restaurants → Restaurant servers
  /api/orders      → Order servers
  /api/payments    → Payment servers
  /static/images   → CDN servers

────────────────────────────────────────────────────

COMPARISON TABLE
─────────────────
           L4                    L7
           ──                    ──
Reads      IP + Port             URL, headers, body
Speed      Faster                Slightly slower
Smart?     ❌ No                 ✅ Yes
Cost       Cheaper               More expensive
Use when   Raw speed, TCP/UDP    REST APIs, HTTP
Examples   AWS NLB               AWS ALB
```

---

## Health Checks — Liveness vs Readiness

```
BASIC HEALTH CHECK (Liveness Probe)
────────────────────────────────────
"Is the server alive?"
→ Ping server → responds? ✅
→ Server responding ≠ server performing well
→ Too simple on its own ❌

SMART HEALTH CHECK (Readiness Probe)
──────────────────────────────────────
"Is the server ready to serve traffic?"

Checks:
→ Response time < 200ms threshold
→ DB connection available
→ Memory usage < 85%
→ Thread pool < 90% busy
→ Downstream services responding

If readiness probe fails:
→ Remove from LB pool temporarily
→ Don't restart (server is alive)
→ Just stop sending traffic

BOTH TOGETHER = Industry standard
  Kubernetes uses both:
  Liveness  → server crashed? restart it
  Readiness → server slow? remove from pool
```

---
---

# CHUNK 12 — Decision Framework: Scale Up vs Scale Out

## When to Scale Up (Vertical)

```
SIGNALS TO SCALE UP
──────────────────────────────────────────────────
→ Traffic under 10K QPS
→ Team is small (2-5 engineers)
→ No time to refactor for statelessness
→ Quick fix needed immediately
→ Database struggling first
  (easier to scale DB vertically)
→ Cost of horizontal complexity >
  cost of bigger machine

Example:
  Startup featured on Shark Tank
  → Sudden 10× traffic spike overnight
  → No time to add servers and refactor
  → Vertically scale immediately
  → Buy time to properly horizontally scale
──────────────────────────────────────────────────
```

---

## When to Scale Out (Horizontal)

```
SIGNALS TO SCALE OUT
──────────────────────────────────────────────────
→ Hitting vertical scaling cost ceiling
→ Need 99.99% availability (no SPOF)
→ Traffic is unpredictable / spiky
→ Different components need different scaling
→ Team large enough for distributed complexity
→ Already stateless or can be made stateless

Example:
  Swiggy before IPL season
  → Knows traffic will spike 10×
  → Can't vertically scale fast enough
  → Adds horizontal servers + auto-scaling
──────────────────────────────────────────────────
```

---

## The Complete Decision Framework

```
SCALE UP OR SCALE OUT?
──────────────────────────────────────────────────

START HERE
    │
    ▼
Is traffic under 10K QPS?
    │
    ├── YES → Vertical scaling fine for now
    │
    └── NO → Is HA (99.99%) required?
              │
              ├── YES → Must horizontal scale
              │
              └── NO → Is traffic spiky?
                        │
                        ├── YES → Horizontal + auto-scaling
                        │
                        └── NO → Is vertical cost exponential?
                                  │
                                  ├── YES → Switch to horizontal
                                  │
                                  └── NO → Vertical still OK

──────────────────────────────────────────────────
PRACTICAL RULE:
  Early stage    → Vertical first
  Growing stage  → Vertical + read replicas
                   + caching layer
  Scale stage    → Full horizontal
                   + auto-scaling + sharding
──────────────────────────────────────────────────
```

---

## Real Company Scaling Journeys

```
ZEPTO
  Day 1    → Single server, monolith
  Month 6  → Vertical scale DB + Redis cache
  Year 1   → Horizontal app servers + read replicas
  Year 2   → Microservices + auto-scaling + sharding

RAZORPAY
  Early    → Vertical scale
  Growth   → Horizontal app + vertical DB
  Scale    → Full horizontal + Kafka + Cassandra + Redis

HOTSTAR IPL
  Normal   → Moderate horizontal
  IPL      → Auto-scale to thousands of instances
  Post IPL → Scale back down, pay for hours only
```

---

## Technical Terms

| Plain English | Technical Term |
|---|---|
| Add/remove servers automatically | Auto-scaling / Elasticity |
| Spin up servers before spike | Pre-warming / Proactive scaling |
| React to spike after it happens | Reactive scaling |
| AWS auto-scaling service | Auto Scaling Groups (ASG) |
| Kubernetes auto-scaling | Horizontal Pod Autoscaler (HPA) |
| Rent capacity, return when done | Cloud elasticity |
| Paying for unused capacity | Over-provisioning |

---
---

# 🎯 Complete Technical Terms Reference — Topic 3

| Concept | Simple Word | Technical Term |
|---|---|---|
| Bigger machine | "upgrade server" | Vertical Scaling |
| More machines | "add servers" | Horizontal Scaling |
| Cost grows faster than power | "expensive upgrade" | Exponential cost curve |
| Maximum machine size | "can't go bigger" | Hard limit |
| Add/remove automatically | "auto-scale" | Elasticity / Auto-scaling |
| Pay for unused capacity | "wasted servers" | Over-provisioning |
| Server stores nothing | "no memory" | Stateless server |
| Server remembers users | "has memory" | Stateful server |
| Different servers, different data | "inconsistent" | Cache Inconsistency |
| One store all servers read | "central data" | Single source of truth |
| Token carries its own data | "self-contained" | JWT |
| JWT can't change after issue | "fixed token" | JWT Immutability |
| Old data in token | "outdated token" | Stale Token Problem |
| Send message to channel | "broadcast" | Publish (Redis Pub-Sub) |
| Listen to channel | "receive" | Subscribe (Redis Pub-Sub) |
| Requests one by one in order | "take turns" | Round Robin |
| Different server weights | "more powerful gets more" | Weighted Round Robin |
| Send to least busy | "least queue" | Least Connections |
| Same IP same server | "pinned to server" | IP Hash |
| Too many requests one server | "bottleneck" | Hot Spot Problem |
| Pick 2, choose less loaded | "smart random" | Random with 2 Choices |
| Reads IP + Port only | "fast routing" | L4 Load Balancer |
| Reads URL + headers | "smart routing" | L7 Load Balancer |
| Is server alive? | "ping check" | Liveness Probe |
| Is server ready? | "performance check" | Readiness Probe |
| Scheduled server downtime | "maintenance" | Maintenance window |
| Spin up before traffic spike | "prepare ahead" | Pre-warming |
| AWS auto-scaling | "AWS scale out" | Auto Scaling Groups (ASG) |
| Kubernetes auto-scaling | "K8s scale out" | HPA |
| Save game state periodically | "state snapshot" | Checkpoint |
| Replica with live state | "always ready backup" | Hot Standby |

---

# 📝 Things to Always Say in Interviews

1. **For vertical scaling:** *"Short-term fix — exponential cost and SPOF make it unsuitable for HA"*
2. **For horizontal scaling:** *"Requires statelessness first — sessions to Redis, auth via JWT"*
3. **For state:** *"Server RAM for request scope only — Redis for shared dynamic state — JWT for static auth"*
4. **For WebSocket:** *"Stateful by nature — use Redis Pub-Sub for cross-server routing"*
5. **For load balancing:** *"Algorithm choice depends on request uniformity and server capacity"*
6. **For health checks:** *"Liveness probe detects crashes, readiness probe detects poor performance"*
7. **For IP Hash:** *"Use session ID hash not IP hash — offices share same IP causing hot spots"*
8. **For scaling decision:** *"Start vertical, switch to horizontal when you hit cost ceiling or need HA"*

---

# 🚫 Things to Never Say in Interviews

1. ~~"Just upgrade the server"~~ → Always mention cost curve + SPOF + downtime
2. ~~"Store sessions in server memory"~~ → Always say Redis
3. ~~"JWT for order status"~~ → JWT only for static data
4. ~~"Use IP Hash for all stateful connections"~~ → Breaks for office networks
5. ~~"Stateless is always better"~~ → WebSocket/games need stateful + mitigation
6. ~~"Health check just pings the server"~~ → Mention liveness vs readiness probes
7. ~~"Round Robin for everything"~~ → Depends on request uniformity

---
---

# 🔥 Drill Session — Questions & Answers

## Q1 — Zepto Vertical Scaling

**Question:**
Zepto has 1 powerful server at 70% CPU handling 8K QPS. Expecting 80K QPS in 3 months. Manager says upgrade to AWS's biggest server (448 cores, 24TB RAM).

1. Will this solve the problem?
2. What are the risks?
3. What would you recommend?

**Answer:**
```
1. WILL IT SOLVE THE PROBLEM?
   Partially — but risky
   Currently 8K QPS at 70% CPU
   80K QPS = 10× traffic
   Even 448 cores might not handle it
   Hard limit is a real concern

2. RISKS
   → Exponential cost (100×+)
   → SPOF remains — crash = Zepto down
   → Maintenance window = downtime to upgrade
   → Temporary fix — next spike same problem

3. RECOMMENDATION
   Short term:
   → Vertical scale one level (buy 2-3 weeks)
   → Add Redis cache (reduce QPS needed)
   
   Medium term:
   → Make stateless → Redis sessions
   → Add second server + LB
   → Active-active setup
   → Auto-scaling at 70% CPU threshold
```

---

## Q2 — Razorpay Load Balancing

**Question:**
Razorpay has 20 servers. Engineer says use Round Robin.
Traffic: /api/payment = 800ms, /api/balance = 10ms, /api/webhook = 100ms.
Is Round Robin right? What would you use?

**Answer:**
```
Round Robin is WRONG here.

Requests vary 80× in processing time.
Round Robin blindly takes turns.
→ Server handling /api/payment overwhelmed
→ Other servers sitting idle

Use LEAST CONNECTIONS:
→ Server processing heavy payment request
→ High active connections
→ LB stops sending there
→ New requests go to free servers
→ No server overwhelmed ✅
```

---

## Q3 — PhonePe Session Problem

**Question:**
PhonePe stores payment session in server memory.
User starts payment on Server 1, next request goes to Server 6.
1. What happens?
2. Technical term?
3. Two fixes?

**Answer:**
```
1. WHAT HAPPENS?
   Server 6 has empty session
   User's payment state lost
   Must restart payment 😱
   Worst case: money debited, order not confirmed

2. TECHNICAL TERM
   Stateful Server Problem /
   Session Affinity Problem

3. TWO FIXES
   Fix 1: Redis for payment session
   → Key: "payment:{userId}:{txnId}"
   → Any server reads same Redis ✅

   Fix 2: JWT for authentication only
   → NOT for payment session (dynamic)
   → JWT for userId, role (static) only
   → Combined: JWT = who, Redis = where payment is
```

---

## Q4 — Hotstar IP Hash Hot Spot

**Question:**
Hotstar uses IP Hash. IPL Final metrics:
Server 1 → 40M connections, Server 2 → 5M, Server 3 → 3M, Server 4 → 2M.
1. What happened?
2. Technical term?
3. Fix for next IPL?

**Answer:**
```
1. WHAT HAPPENED?
   Millions of office/university users
   share same public IP address
   IP Hash → all mapped to Server 1
   Server 1 overwhelmed 😱

2. TECHNICAL TERM
   Hot Spot Problem
   Too many requests on one server
   due to shared property (same IP)

3. FIX FOR NEXT IPL
   Hash on USER ID or SESSION ID
   instead of IP address
   → Even if 1000 users share same IP
   → All have different user IDs
   → Distributed evenly ✅
```

---

## Q5 — PUBG Game Server

**Question:**
Gaming company with 3 servers, 100 players per game, state updates 60×/sec.
1. Stateful or stateless?
2. Which LB algorithm?
3. Server crashes mid-game?

**Answer:**
```
1. STATEFUL
   Game state changes 60×/second
   100 players × 60 = 6,000 updates/sec
   Stateless = 6M Redis writes/sec per game
   → Too slow, too expensive
   Server memory = nanosecond access ✅

2. SESSION ID HASH
   All 100 players in same game
   share unique Game Session ID
   hash(gameSessionId) → same server
   All players → same server ✅

3. SERVER CRASH HANDLING
   CHECKPOINT + REPLAY:
   → Save state snapshot to Redis every 5 sec
   → Crash → restore from checkpoint
   → Players reconnect → resume game
   → Last 5 seconds lost (acceptable)
   
   OR HOT STANDBY (expensive):
   → Mirror state to standby server
   → Instant failover
   → Double cost
   
   Most games use Checkpoint + player compensation
```

---

## Q6 — Swiggy Health Check

**Question:**
All 10 servers show "healthy" but users complain app takes 8 seconds.
1. Is engineer right that it's user-side?
2. What could be wrong?
3. What should health check include?

**Answer:**
```
1. ENGINEER IS WRONG
   "Healthy" = server alive, not server fast
   Server responding to ping ≠ performing well

2. WHAT COULD BE WRONG
   → DB connection pool exhausted
     (requests waiting for DB connection)
   → Memory leak (RAM filling up)
   → Thread pool exhausted
     (can't accept new requests)
   → Downstream dependency slow
     (DB/Redis/external API slow)

3. SMART HEALTH CHECK SHOULD INCLUDE
   Liveness Probe: Is server alive?
   Readiness Probe: Is server ready?
   → Response time < 200ms
   → DB connection available
   → Memory < 85%
   → Thread pool < 90% busy
   → Downstream services responding

   Liveness → server crashed? restart
   Readiness → server slow? remove from pool
```

---

## Q7 — Flipkart Big Billion Day

**Question:**
Normal: 10K QPS. Big Billion Day: 500K QPS for 24 hours.
1. Vertical or horizontal?
2. Which scaling strategy?
3. Cost after sale ends?

**Answer:**
```
1. HORIZONTAL SCALING
   500K QPS = 50× normal
   No single machine handles this
   Hard limit of vertical would be hit

2. SCALING STRATEGY
   Step 1: Make stateless
   → Sessions → Redis
   → Auth → JWT

   Step 2: Auto-scaling at 70% CPU
   → AWS Auto Scaling Groups
   → New servers spin up automatically
   → LB notified automatically

   Step 3: Pre-warm before sale
   → Spin up 50× servers at 11:50pm
   → Ready before traffic hits
   → Proactive not reactive scaling ✅

   Step 4: Cache aggressively
   → Product pages cached in Redis
   → Reduces DB load dramatically

3. COST AFTER SALE
   Auto-scaling scales IN too
   → Traffic drops after 24 hours
   → Extra servers automatically removed
   → Pay only for 24 hours of scale
   → Cloud elasticity ✅
```

---

## Q8 — QuickMed Healthcare App

**Question:**
QuickMed: 1000 users/day, doctor-patient video calls, stores medical records.
1. Scaling type right now?
2. What makes it unique?
3. Special components?

**Answer:**
```
1. VERTICAL SCALING RIGHT NOW
   Only 1000 users/day
   Small team, building product
   No need for distributed complexity
   Scale when you feel the pain
   Monitor at 70% CPU/RAM

2. THREE UNIQUE THINGS
   Unique 1: PHI (Protected Health Information)
   → Medical data = most sensitive data
   → Diagnosis, prescriptions, allergies
   → Data leak = privacy disaster
   → DPDP Act 2023 compliance required
   → Data must stay in India
   → Encrypt everything always

   Unique 2: Video call = stateful
   → Doctor + patient on same server
   → Session ID hash for LB
   → Medical consultation can't be interrupted

   Unique 3: Real-time critical data
   → Doctor needs allergies instantly
   → Wrong/slow data = wrong diagnosis
   → Cache patient records in Redis

3. SPECIAL COMPONENTS
   Database (most critical):
   → Add DB replica EVEN AT 1000 users
   → SPOF here = life threatening
   → Automatic failover < 30 seconds
   → WAL enabled, daily encrypted backups

   Encryption:
   → At rest: AES-256
   → In transit: TLS 1.3
   → Keys in Vault/KMS, never hardcoded

   Access Control (RBAC):
   → Patient sees own records only
   → Doctor sees assigned patients only
   → Admin sees nothing medical
   → Every access = audit log entry

   Video call reliability:
   → WebRTC for peer-to-peer video
   → TURN server as fallback
   → Rejoin link if call drops
```

---

## 📊 Drill Scores

| Question | Score |
|---|---|
| Q1 — Zepto vertical scaling | 3.5/5 |
| Q2 — Razorpay LB algorithm | 5/5 |
| Q3 — PhonePe session problem | 3/5 |
| Q4 — Hotstar IP Hash hot spot | 4.5/5 |
| Q5 — PUBG game server | 2.5/5 |
| Q6 — Swiggy health check | 3/5 |
| Q7 — Flipkart Big Billion Day | 5/5 |
| Q8 — QuickMed scaling | 3.5/5 |
| **Total** | **30/40 = 75%** |

---

*Topic 3 Complete ✅ | Next: Topic 4 — Concurrency Fundamentals*