# Topic 3 — Vertical vs Horizontal Scaling + Stateful vs Stateless
## Complete Notes | Concepts + Interview Q&A + Technical Terms

---

## 📊 Topic 3 Stats
- **Chunks covered:** 9, 10, 10B
- **Status:** 🔄 In Progress (Chunks 11, 12 pending)
- **Running score:** 20.5/27

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

## Three Fundamental Problems

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

## Interview Questions + How to Answer

### Q: "When would you choose vertical scaling?"

> "Vertical scaling is my first choice for early-stage
> systems and databases — it's simple, requires no
> code changes, and works well up to a point.
> But it has three fundamental problems: exponential
> cost curve, a hard physical limit on machine size,
> and it remains a single point of failure.
> For anything needing high availability or
> internet scale, I'd plan for horizontal scaling
> from the start."

---

### Q: "Your DB is struggling at peak load — upgrade to bigger server?"

> "That's a short-term fix with three problems.
> First, 4× resources won't cost 4× — the pricing
> curve is exponential at higher tiers.
> Second, we still have a single point of failure —
> a bigger server crashing during a sale costs more.
> Third, resizing requires downtime — unacceptable
> during peak traffic.
> Short term: vertical scale as emergency fix.
> Long term: add read replicas + caching layer."

---

### Q: "Hotstar expects 5× IPL viewers — vertically scale?"

> "Vertical scaling won't work here for two reasons.
> First, 50M concurrent viewers likely exceeds what
> any single machine can handle — hard physical limit.
> Second, economics are terrible — paying for 5×
> capacity 365 days for a 4-hour event.
> That's massive over-provisioning.
> Real solution: horizontal auto-scaling on cloud —
> spin up thousands of servers as viewers join,
> tear them down after match ends.
> Pay only for those hours. This is cloud elasticity."

---

## Common Mistakes

```
MISTAKE 1: Recommending vertical scaling for HA systems
  → Vertical = SPOF always
  → HA needs redundancy = horizontal

MISTAKE 2: Saying 4× resources = 4× cost
  → Cost curve is exponential
  → Always mention this in interviews

MISTAKE 3: Forgetting downtime during upgrade
  → Vertical scaling requires maintenance window
  → Horizontal = zero downtime

MISTAKE 4: Not mentioning hard limit
  → Every machine has maximum size
  → After that → forced horizontal
```

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
User preferences    Server RAM ❌       DB + Cache ✅
Exchange rates      Server RAM ❌       Redis ✅
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

## Benefits and Challenges

```
BENEFITS OF HORIZONTAL SCALING
────────────────────────────────
✅ No hard limit — add as many as needed
✅ No SPOF — server dies, others continue
✅ Cost efficient — commodity servers cheap
✅ Auto-scaling — add/remove based on load
✅ Zero downtime scaling — add while running
✅ Different services scale independently

CHALLENGES OF HORIZONTAL SCALING
──────────────────────────────────
❌ Statelessness required — code must change
❌ Need load balancer — extra component
❌ Distributed systems complexity increases
❌ Data consistency harder across servers
❌ More servers = more to monitor/operate
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

## Interview Questions + How to Answer

### Q: "How do you horizontally scale your app?"

> "First I make sure our app servers are completely
> stateless — no session data in memory.
> Sessions go to Redis, authentication uses JWT.
> Then I put a load balancer in front that
> distributes requests across all server instances.
> Any server can handle any request.
> For auto-scaling, we set CPU/memory thresholds —
> when load spikes, new instances spin up automatically.
> This is how Hotstar handles IPL traffic spikes —
> thousands of instances spin up as viewers join,
> tear down after the match."

---

### Q: "Why is statelessness required for horizontal scaling?"

> "If servers store session state in memory,
> each server knows only about the users it's
> served. When a load balancer routes the same
> user to a different server, that server has
> no session — user appears logged out or
> loses their cart.
> Stateless servers offload all shared state
> to external stores like Redis — any server
> reads the same Redis, so any server can
> handle any request transparently."

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
Temp calculations       Server memory       Request scope only

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

## Technical Terms

| Plain English | Technical Term |
|---|---|
| Server stores connection in memory | Stateful server |
| Server stores nothing | Stateless server |
| Token carries its own state | JWT (JSON Web Token) |
| JWT can't be changed after creation | JWT Immutability |
| User sees old data in JWT | Stale Token Problem |
| Publish message to channel | Redis Pub-Sub (Publisher) |
| Listen to channel for messages | Redis Pub-Sub (Subscriber) |
| Named pipe servers subscribe to | Channel |
| Which server each user is on | User-Server routing table |
| Messages saved for offline users | Offline delivery / persistence |
| Technical term for cross-server chat problem | Cross-server message routing |

---

## Interview Questions + How to Answer

### Q: "How do you handle WebSocket connections in a horizontally scaled system?"

> "WebSocket connections are stateful by nature —
> each connection is pinned to one specific server.
> When User A on Server 3 sends to User B on Server 7,
> Server 3 can't directly access User B's connection.
> We solve this with Redis Pub-Sub — each server
> subscribes to its own channel. Server 3 looks up
> User B's server from a Redis routing table,
> publishes the message to that server's channel.
> Server 7 receives it instantly and delivers
> to User B's WebSocket. For offline users,
> messages are persisted to DB and delivered on reconnect."

---

### Q: "Should you store order status in JWT?"

> "No — JWT is immutable after creation and
> order status changes every few minutes.
> We can't push updated JWT to clients,
> so the token would show stale status.
> JWT is right for static data — userId, role,
> permissions — things that rarely change.
> For order status, Redis is correct —
> single source of truth, any server reads
> the same value, updated instantly when
> order service changes the status."

---

### Q: "You have 10 servers. Each stores exchange rate in local memory. What's wrong?"

> "This is a cache inconsistency problem.
> Each server refreshes independently so during
> the refresh window different servers show
> different rates — two users making simultaneous
> payments get charged different amounts.
> For Razorpay that's a direct financial and
> RBI compliance risk.
> Fix: centralise exchange rate in Redis —
> one source of truth all servers read from.
> Rate updated once in Redis → all servers
> see it instantly. Servers become truly stateless."

---

## Common Mistakes

```
MISTAKE 1: Storing dynamic data in JWT
  → JWT is immutable — can't update after issue
  → Use Redis for anything that changes
  → JWT only for userId, role, static claims

MISTAKE 2: Thinking sticky sessions solves stateful problem
  → Sticky sessions = same user always same server
  → Server dies → user loses session anyway
  → Uneven load distribution
  → Use only as TEMPORARY workaround

MISTAKE 3: Not knowing Redis Pub-Sub for WebSocket
  → Most candidates can't explain cross-server chat
  → This is a senior-level answer
  → Know: publish → channel → subscribe flow

MISTAKE 4: Forgetting offline user handling
  → What if recipient is offline?
  → Must persist to DB
  → Deliver on reconnect
  → Always mention this in chat system design

MISTAKE 5: Saying stateless is always better
  → WebSocket, game servers are necessarily stateful
  → Know WHEN stateful is unavoidable
  → And how to handle it (Redis Pub-Sub)
```

---

# 🎯 Quick Reference — All Technical Terms (Topic 3)

| Concept | Simple Word | Technical Term | Use in Interview |
|---|---|---|---|
| Bigger machine | "upgrade server" | Vertical Scaling | "Vertical scaling has exponential cost" |
| More machines | "add servers" | Horizontal Scaling | "Horizontal scaling requires statelessness" |
| Cost grows faster than power | "expensive upgrade" | Exponential cost curve | "Cost curve is exponential at higher tiers" |
| Maximum machine size | "can't go bigger" | Hard limit | "Vertical scaling hits hard limit" |
| Add/remove servers automatically | "auto-scale" | Elasticity / Auto-scaling | "Cloud elasticity handles IPL spikes" |
| Pay for unused capacity | "wasted servers" | Over-provisioning | "Vertical scaling causes over-provisioning" |
| Server stores nothing | "no memory" | Stateless server | "App servers must be stateless" |
| Server remembers users | "has memory" | Stateful server | "WebSocket servers are stateful" |
| Different servers, different data | "inconsistent" | Cache Inconsistency | "Local memory causes cache inconsistency" |
| One store all servers read | "central data" | Single source of truth | "Redis is single source of truth" |
| Token carries its own data | "self-contained token" | JWT | "JWT makes auth stateless" |
| JWT can't change after issue | "fixed token" | JWT Immutability | "JWT immutability causes stale token problem" |
| Old data in token | "outdated token" | Stale Token Problem | "Order status in JWT causes stale token" |
| Send message to channel | "broadcast" | Publish (Redis Pub-Sub) | "Server 3 publishes to server-7 channel" |
| Listen to channel | "receive" | Subscribe (Redis Pub-Sub) | "Server 7 subscribes to its channel" |
| Planned server downtime | "maintenance" | Maintenance window | "Vertical scaling needs maintenance window" |
| Same user, same server | "pinned routing" | Sticky sessions | "Sticky sessions are an anti-pattern" |

---

# 📝 Things to Always Say in Interviews

1. **For vertical scaling:** *"Short-term fix — exponential cost curve and SPOF make it unsuitable for HA systems"*
2. **For horizontal scaling:** *"Requires statelessness — offload all state to Redis or use JWT"*
3. **For state decision:** *"Server RAM for request scope only — Redis for shared dynamic state — JWT for static auth"*
4. **For WebSocket:** *"Stateful by nature — use Redis Pub-Sub for cross-server routing"*
5. **For cache inconsistency:** *"Local memory caches cause inconsistency — centralise in Redis as single source of truth"*

---

# 🚫 Things to Never Say in Interviews

1. ~~"Just upgrade the server"~~ → Always mention cost curve + SPOF
2. ~~"Store sessions in server memory"~~ → Always say Redis
3. ~~"JWT for order status"~~ → JWT only for static data
4. ~~"Sticky sessions fix stateful problem"~~ → Anti-pattern, server death = same problem
5. ~~"Stateless is always better"~~ → WebSocket/games need stateful + mitigation

---

# ⏭️ Pending Chunks in Topic 3

| Chunk | Title | Status |
|---|---|---|
| Chunk 11 | Load Balancer Role | ⬜ Next |
| Chunk 12 | Decision Framework — Scale Up vs Scale Out | ⬜ Pending |
| 🔥 | Topic 3 Drill Session | ⬜ Pending |

---

*Topic 3 Partially Complete 🔄 | Chunks 11, 12 + Drill pending*
*Next: Chunk 11 — Load Balancer Role*