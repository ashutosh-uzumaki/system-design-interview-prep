# Chunk 24E — Request Lifecycle
### Phase 2 — Databases | SDE-2 Interview Prep
> "What happens when you type google.com and press Enter?"

---

## The 3 Phases of a Request

```
Phase 1 — Network Layer (before your servers):
→ DNS resolution
→ TCP connection
→ TLS handshake

Phase 2 — Your Infrastructure:
→ CDN
→ Load balancer
→ API Gateway
→ Microservice
→ Cache + Database

Phase 3 — Response Path:
→ Response assembled
→ Travels back same path
→ Rendered on screen
```

---

# Phase 1 — Network Layer

## Step 1 — DNS Resolution

```
You type: api.swiggy.com

Your device doesn't know the IP address.

DNS Resolution flow:
1. Check local DNS cache (device) → miss
2. Check router/ISP DNS → miss
3. Ask Root DNS server
4. Root says "ask .com DNS"
5. .com DNS says "ask swiggy.com DNS"
6. swiggy.com DNS returns: 52.14.23.45 ✅

Now device knows where to send request.

Cached DNS:
→ OS caches result for TTL duration
→ Next request = instant (0ms) ✅

DNS lookup time:
→ First time: ~20-100ms
→ Cached: ~0ms ✅
```

---

## Step 2 — TCP 3-Way Handshake

```
Device connects to 52.14.23.45:

Step 1: Client → Server: SYN
        "Hello, I want to connect"

Step 2: Server → Client: SYN-ACK
        "OK, I'm ready"

Step 3: Client → Server: ACK
        "Great, let's go"

Connection established ✅
Time: ~10-50ms (depends on network distance)

Why TCP?
→ Reliable delivery guaranteed
→ Packets arrive in order
→ Lost packets retransmitted ✅
```

---

## Step 3 — TLS Handshake (HTTPS)

```
Since modern apps use HTTPS:

1. Client → Server: "I support TLS 1.3, here's my hello"
2. Server → Client: Certificate + public key
3. Client verifies certificate ✅
4. Client + Server agree on encryption keys
5. Secure channel established ✅

All data encrypted from here ✅
Time: ~10-50ms

TLS 1.3 (latest):
→ Faster than TLS 1.2
→ 1 round trip instead of 2
→ Modern apps use TLS 1.3 ✅
```

---

# Phase 2 — Your Infrastructure

## Step 4 — CDN (Static Assets)

```
Request reaches CDN edge (nearest server):

Static assets (JS, CSS, images, fonts):
→ Served from CDN edge directly ✅
→ Never reaches origin server
→ Ultra fast (~5-20ms) ✅

API requests:
→ CDN passes through to origin ✅
→ Or CDN caches API responses (short TTL)

CDN benefits:
→ Mumbai user → Mumbai CDN edge
→ Not all the way to Bangalore servers
→ Lower latency ✅
```

---

## Step 5 — Load Balancer

```
Request arrives at Swiggy's infrastructure:

Load Balancer:
→ Multiple servers behind it
→ Picks healthiest/least busy server
→ Forwards request ✅

L4 Load Balancer:
→ Works at TCP level
→ Routes based on IP + port
→ Faster, no HTTP awareness

L7 Load Balancer:
→ Works at HTTP level
→ Reads headers, URLs, cookies
→ Routes based on path:
   /search → Search Service servers
   /order  → Order Service servers
   /pay    → Payment Service servers
→ Smarter routing ✅

Health checks:
→ LB pings servers every 5 seconds
→ Unhealthy server → removed from pool
→ Requests only go to healthy servers ✅
```

---

## Step 6 — API Gateway

```
Request hits API Gateway:

What API Gateway does:
1. Authentication
   → Validates JWT token
   → "Is this user logged in?" ✅

2. Authorization
   → "Does this user have permission?" ✅

3. Rate limiting
   → "Has this user exceeded 10 req/sec?" ✅
   → Redis INCR check

4. Request routing
   → Routes to correct microservice ✅

5. Request/Response transformation
   → Adds headers, modifies payload

6. Logging + monitoring
   → Every request logged ✅
   → Latency tracked ✅

7. SSL termination
   → Decrypts HTTPS at gateway
   → Internal traffic can be HTTP ✅

All in one place — services don't handle this ✅
```

---

## Step 7 — Microservice

```
Search request → Search Service:

1. Validate request parameters
   → q = "burger", city = "bangalore" ✅

2. Check Redis cache:
   "search:bangalore:burger" → cached? ✅

Cache HIT:
→ Return cached results immediately
→ ~1ms ✅
→ Done

Cache MISS → continue...

3. Query Elasticsearch:
   → Full text search for "burger"
   → Returns matching restaurant IDs ✅

4. Query PostgreSQL:
   → Metadata, ratings, filters
   → User preferences ✅

5. Combine results
   → Rank by relevance + distance

6. Store in Redis:
   "search:bangalore:burger" → results
   TTL: 5 minutes ✅

7. Return results
```

---

## Step 8 — Cache + Database Layer

```
Redis (cache):
→ Cache hit: ~1ms ✅
→ Checked BEFORE DB always

Elasticsearch:
→ Full text search
→ "burger" → [Burger King, Burger Singh...]
→ ~10-50ms

PostgreSQL:
→ Restaurant metadata, user data
→ Joins, filters
→ ~10-100ms

Connection pooling (PgBouncer):
→ Reuses DB connections
→ No new TCP handshake per query ✅
→ Much faster ✅
```

---

# Phase 3 — Response Path

```
Results flow back:

Search Service
      ↓
API Gateway (add response headers)
      ↓
Load Balancer
      ↓
Client (phone/browser)

Same path, opposite direction.
TLS encrypted throughout ✅
TCP ensures reliable delivery ✅

App receives JSON response:
→ Parses results
→ Renders UI
→ User sees restaurants ✅
```

---

# Complete Request Lifecycle

```
User searches "Burger" on Swiggy:

1. DNS Resolution (~20-100ms, ~0ms if cached)
   → api.swiggy.com → 52.14.23.45

2. TCP 3-way handshake (~10-50ms)
   → SYN → SYN-ACK → ACK

3. TLS handshake (~10-50ms)
   → Secure channel established

4. HTTP Request sent
   → GET /search?q=burger&city=bangalore

5. CDN check
   → Static assets: CDN edge ✅ (fast)
   → API request: passes through

6. Load Balancer (~1ms)
   → Routes to least busy Search server

7. API Gateway (~5ms)
   → JWT verified ✅
   → Rate limit OK ✅
   → Routed to Search Service ✅

8. Search Service
   → Request validated

9. Redis cache check (~1ms)
   → HIT → return immediately ✅
   → MISS → continue

10. Elasticsearch query (~10-50ms)
    → "burger" matches restaurants

11. PostgreSQL query (~10-100ms)
    → Metadata, filters

12. Results assembled + cached in Redis ✅

13. Response travels back:
    Service → Gateway → LB → Client

14. TLS decryption on client
    → App renders results ✅

Total time:
→ Cache HIT:  ~50-150ms  ✅
→ Cache MISS: ~150-400ms
```

---

## Where Time Is Spent

```
DNS resolution:         ~20-100ms  (cached = ~0ms)
TCP handshake:          ~10-50ms
TLS handshake:          ~10-50ms
Network travel:         ~10-30ms
CDN:                    ~5-20ms    (for static assets)
Load balancer:          ~1ms
API Gateway:            ~5ms
Redis cache hit:        ~1ms       ✅ fastest
Redis miss + DB query:  ~20-100ms
Response travel:        ~10-30ms

Optimization targets:
→ DNS prefetch → eliminate DNS time ✅
→ CDN → reduce network travel ✅
→ Redis cache → eliminate DB time ✅
→ Connection pooling → reduce TCP overhead ✅
→ HTTP/2 → reduce round trips ✅
```

---

## HTTP/1.1 vs HTTP/2

```
HTTP/1.1:
→ One request per TCP connection
→ 10 resources = 10 connections 😬
→ Head-of-line blocking
→ No header compression

HTTP/2:
→ Multiple requests on ONE TCP connection ✅
→ Multiplexing ✅
→ Header compression (HPACK) ✅
→ Server push ✅
→ Much faster for modern apps ✅

Swiggy loading 10 resources:
HTTP/1.1 → 10 TCP connections 😬
HTTP/2   → 1 TCP connection, 10 parallel requests ✅

HTTP/3 (latest):
→ Uses QUIC instead of TCP
→ Faster connection setup ✅
→ Better on mobile networks ✅
→ Google, Cloudflare use HTTP/3 ✅
```

---

## Latency Optimization — Interview Answer

```
"How would you reduce latency?"

Weak: "Add more servers"

Strong:
→ DNS prefetching → eliminate DNS lookup ✅
→ CDN → move content closer to user ✅
→ Redis cache → eliminate DB round trip (~100ms saved) ✅
→ Connection pooling → reduce TCP overhead ✅
→ HTTP/2 multiplexing → reduce round trips ✅
→ Keep-alive connections → reuse TCP connections ✅
→ Compress responses → reduce payload size ✅

Each optimization targets specific step ✅
```

---

## Real World Numbers

```
Google's target: < 200ms page load
Amazon: 100ms delay = 1% revenue loss
Swiggy: < 300ms search results

How they achieve it:
→ CDN edges globally ✅
→ Redis cache (1ms vs 100ms) ✅
→ DNS prefetching ✅
→ HTTP/2 ✅
→ Read replicas for DB ✅
→ Elasticsearch for search ✅
```

---

## Interview Framing

**On request lifecycle:**
> *"When a user searches on Swiggy, the request first goes through DNS resolution to find the server IP, then TCP + TLS handshake for secure connection. It hits CDN for static assets, then load balancer routes to least busy server. API Gateway validates JWT and rate limits before routing to Search Service. Service checks Redis first — cache hit returns in 1ms. Cache miss queries Elasticsearch for full-text search and PostgreSQL for metadata, stores result in Redis, and returns. Total time with cache hit: ~100-150ms."*

**On latency reduction:**
> *"I'd target the biggest time sinks first — Redis caching eliminates the 100ms DB round trip, CDN reduces network travel time, HTTP/2 multiplexing reduces connection overhead. DNS prefetching eliminates the 20-100ms DNS lookup on first load. Each optimization has a measurable impact on a specific step."*

---

## Quick Drill Questions

```
Q1: What happens before a request 
    reaches your load balancer?
    Name the steps.

Q2: What does API Gateway do?
    Name 5 responsibilities.

Q3: A user in Mumbai hits Swiggy.
    How does CDN help?

Q4: What is TCP 3-way handshake?
    Why is it needed?

Q5: Search results take 400ms.
    Walk through where time is spent
    and how to reduce it.

Q6: What is HTTP/2 and why is it 
    faster than HTTP/1.1?

Q7: Cache hit vs cache miss — 
    what's the latency difference?
    Why does it matter?
```

---

## Drill Answers

### Q1: Before load balancer
```
1. DNS resolution:
   → Domain name → IP address
   → Checks cache → ISP → Root → TLD → Domain DNS

2. TCP 3-way handshake:
   → SYN → SYN-ACK → ACK
   → Reliable connection established

3. TLS handshake:
   → Certificate verification
   → Encryption keys agreed
   → Secure channel established

Then request reaches load balancer ✅
```

### Q2: API Gateway responsibilities
```
1. Authentication → validate JWT token ✅
2. Authorization → check permissions ✅
3. Rate limiting → prevent abuse ✅
4. Request routing → to correct microservice ✅
5. SSL termination → decrypt HTTPS ✅
6. Logging + monitoring → every request tracked ✅
7. Request transformation → modify headers/payload ✅
```

### Q3: CDN for Mumbai user
```
Without CDN:
→ Mumbai user → request travels to Bangalore server
→ ~50ms network travel each way
→ 100ms just for network 😬

With CDN:
→ Mumbai user → Mumbai CDN edge server
→ ~5ms network travel ✅
→ Static assets (JS, CSS, images) served locally
→ 10-20x faster for static content ✅

API requests still go to origin,
but static assets = instant ✅
```

### Q4: TCP 3-way handshake
```
SYN → SYN-ACK → ACK

Why needed:
→ Establishes reliable connection
→ Both sides confirm ready to communicate
→ Sequence numbers synchronized
→ Without it → packets arrive out of order,
  no way to detect lost packets 😬

Cost: ~10-50ms (one round trip)
Optimization: Keep-alive connections reuse
existing TCP connection for multiple requests ✅
```

### Q5: 400ms search — reduce it
```
Break down 400ms:
→ DNS: ~50ms → fix: DNS prefetch (0ms)
→ TCP+TLS: ~100ms → fix: keep-alive, HTTP/2
→ Network: ~30ms → fix: CDN edge servers
→ Load balancer: ~1ms → acceptable
→ API Gateway: ~5ms → acceptable
→ Cache miss + DB: ~200ms → fix: Redis cache
→ Response: ~30ms → fix: compress response

After optimization:
→ DNS prefetch: 0ms ✅
→ HTTP/2: reuse connection ✅
→ Redis cache hit: 1ms (was 200ms) ✅
→ Total: ~50-100ms ✅
```

### Q6: HTTP/2 vs HTTP/1.1
```
HTTP/1.1:
→ One request per TCP connection
→ 10 resources = 10 TCP connections 😬
→ Head-of-line blocking

HTTP/2:
→ Multiple requests on ONE connection ✅
→ Multiplexing (parallel requests) ✅
→ Header compression (HPACK) ✅
→ ~2-3x faster for typical web pages ✅

Example:
Loading Swiggy app (10 JS/CSS files):
HTTP/1.1 → 10 connections × handshake = 500ms 😬
HTTP/2   → 1 connection, 10 parallel = 100ms ✅
```

### Q7: Cache hit vs miss latency
```
Redis cache hit:  ~1ms ✅
DB query (miss):  ~10-100ms

Difference: 10-100x faster ✅

Why it matters:
→ Swiggy: 10M searches/day
→ Each cache miss = 100ms extra
→ 10M × 100ms = 277 hours of extra wait per day 😬
→ Cache hit rate of 90% = massive improvement

Rule: Cache hit rate should be > 90% in production
→ Monitor and optimize if lower ✅
```

---

## One-Line Summaries

```
DNS         → domain name to IP address
TCP         → reliable connection via 3-way handshake
TLS         → encrypted channel via certificate
CDN         → serve static assets from nearest edge
Load balancer → distribute to healthy servers
API Gateway → auth, rate limit, route, log
Redis cache → 1ms vs 100ms DB query
HTTP/2      → multiple requests on one connection
Cache hit   → 1ms, cache miss → 100ms
Optimize    → target biggest latency contributor first
```

---

*Chunk 24E complete*
*Next: Master DB Decision Framework (Final Phase 2 chunk)*
*Phase 2 progress: 15/16 chunks done*