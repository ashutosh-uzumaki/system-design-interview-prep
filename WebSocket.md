# WebSockets — Complete HLD Interview Guide

---

## 1. What Are WebSockets?

WebSocket is a **full-duplex, persistent communication protocol** over a single TCP connection. Unlike HTTP's request-response model, WebSockets allow both client and server to send messages independently at any time after the initial handshake.

**Protocol Identifier:** `ws://` (unencrypted) / `wss://` (TLS encrypted)

**RFC:** 6455

---

## 2. Internal Working (How to Explain to an Interviewer)

### 2.1 The Handshake (Connection Upgrade)

WebSocket starts life as a normal HTTP request and then "upgrades" the protocol.

```
Client → Server (HTTP GET)
---------------------------------
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```
Server → Client (HTTP 101)
---------------------------------
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**Key Points to Mention:**
- The `Sec-WebSocket-Key` is a random Base64 nonce from the client.
- The server concatenates it with a magic GUID (`258EAFA5-E914-47DA-95CA-5AB0A964DE09`), SHA-1 hashes it, and Base64 encodes the result to produce `Sec-WebSocket-Accept`.
- This is **not for security** — it's to prevent accidental upgrade by misconfigured proxies/caches.
- After `101 Switching Protocols`, the TCP socket is held open and both sides speak the WebSocket **frame protocol**, not HTTP.

### 2.2 Frame Structure

Once upgraded, data is sent in **frames**, not raw bytes.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|     Masking-key (0 or 4 bytes)      |                         |
+-------------------------------------+                         |
|                     Payload Data                              |
+---------------------------------------------------------------+
```

**Key Opcodes:**
- `0x1` — Text frame
- `0x2` — Binary frame
- `0x8` — Connection close
- `0x9` — Ping
- `0xA` — Pong

**Interview Nugget:** Client-to-server frames MUST be masked (XOR with 4-byte key). Server-to-client frames are NOT masked. This is a security measure against cache poisoning attacks on intermediary proxies.

### 2.3 Connection Lifecycle

```
[Client]                              [Server]
   |--- HTTP GET (Upgrade) ------------>|
   |<-- HTTP 101 (Switching Protocols) -|
   |                                    |
   |<======= Full Duplex Channel =====>|
   |   (text/binary frames, ping/pong) |
   |                                    |
   |--- Close Frame (0x8) ------------>|
   |<-- Close Frame (0x8) -------------|
   |         TCP FIN handshake          |
```

---

## 3. When to Use WebSockets (and When NOT To)

### Use WebSockets When:
- **Low-latency, bidirectional** communication is needed (chat, gaming, collaborative editing)
- **Server-initiated pushes** are frequent (live dashboards, stock tickers, notifications)
- **High message frequency** — dozens to thousands of messages per second per connection
- You need a **persistent channel** to avoid repeated connection overhead

### Do NOT Use WebSockets When:
- You only need **server → client** push (use SSE instead — simpler)
- Communication is **infrequent** (standard HTTP with polling is fine)
- Your data is **request-response** natured (REST/gRPC is better)
- You need **strong caching** (HTTP has built-in caching; WebSockets don't)
- You need **broad compatibility** with firewalls/proxies (some corporate networks block WS)

---

## 4. Architecture Patterns for HLD

### 4.1 Single Server (Naive)

```
[Clients] ──ws──> [Single WS Server] ──> [DB]
```

All connections on one box. Simple, but dies at ~10K–50K concurrent connections depending on hardware and message throughput.

### 4.2 Scaled Architecture (Production)

```
                        ┌──────────────┐
[Clients] ──wss──>      │   L4/L7 LB   │
                        │ (sticky sess) │
                        └──────┬───────┘
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
        [WS Server 1]   [WS Server 2]    [WS Server N]
              │                │                 │
              └────────┬───────┘─────────────────┘
                       ▼
              ┌─────────────────┐
              │   Pub/Sub Bus   │
              │ (Redis / Kafka  │
              │  / NATS / RMQ)  │
              └────────┬────────┘
                       ▼
                 [Persistence]
                 [DB / Cache  ]
```

---

## 5. Scaling WebSockets — The Hard Parts

### 5.1 Sticky Sessions

A WebSocket connection is **stateful**. Once a client connects to Server-2, all subsequent frames go to Server-2. If the LB re-routes mid-connection, the connection breaks.

**Solutions:**
- **L4 (TCP-level) load balancing** — routes by source IP/port. No HTTP awareness needed after handshake.
- **L7 with sticky sessions** — use cookies or connection ID to pin to a server.
- **Consistent hashing** on user ID / room ID to deterministically route.

### 5.2 Cross-Server Messaging (The Fan-Out Problem)

User A is on Server-1. User B is on Server-3. A sends a message to B. Server-1 doesn't have B's connection.

**Solution: Pub/Sub Backplane**

Every WS server subscribes to a shared message bus.

```
Server-1 receives msg from A
  → Publishes to Redis Pub/Sub channel "room:123"
  → Server-3 (subscribed to "room:123") receives it
  → Server-3 pushes to User B's socket
```

**Common Pub/Sub Choices:**

| Technology | Throughput | Persistence | Best For |
|---|---|---|---|
| Redis Pub/Sub | Very High | None (fire-and-forget) | Chat, notifications |
| Redis Streams | High | Yes (consumer groups) | Ordered, replayable events |
| Kafka | Very High | Yes (log-based) | Event sourcing, audit trails |
| NATS | Extremely High | Optional (JetStream) | Low-latency microservices |
| RabbitMQ | Moderate | Yes | Task queues, routing logic |

### 5.3 Connection Limits

Each WebSocket connection holds a **TCP socket = file descriptor**.

| Resource | Default Limit | Tuned Limit |
|---|---|---|
| File descriptors per process | 1,024 | 1,000,000+ |
| Ephemeral ports | ~28K | ~60K |
| Memory per connection | ~2-10 KB idle | Depends on buffers |

**Tuning checklist:**
- Increase `ulimit -n` (file descriptors)
- Tune `net.ipv4.ip_local_port_range`
- Tune `net.core.somaxconn` (backlog queue)
- Use epoll (Linux) / kqueue (macOS) — event-driven I/O
- Bind multiple IPs on the server to multiply the port space

**Interview Line:** "A single modern server can handle 500K–1M+ idle WebSocket connections with epoll and proper tuning, but throughput (messages/sec) is usually the real bottleneck, not connection count."

### 5.4 Memory and CPU Pressure

- **Idle connections** cost ~2-10 KB each (kernel socket buffers).
- **Active connections** with JSON parsing, serialization, and business logic can cost 50-200 KB+ each.
- **CPU** is consumed per message for serialization/deserialization, routing logic, and auth checks.

**Mitigation:**
- Use binary protocols (MessagePack, Protobuf) over JSON to reduce parse cost.
- Offload heavy processing to worker queues.
- Use connection pooling / multiplexing at the application layer.

---

## 6. Failure Modes and Handling

### 6.1 Connection Drops

| Cause | Detection | Recovery |
|---|---|---|
| Network blip | Missed Pong to Ping | Client auto-reconnect with exponential backoff |
| Server crash | TCP RST / timeout | Client reconnects; LB routes to healthy node |
| Idle timeout (proxy/LB) | Sudden close | Periodic Ping/Pong heartbeats (every 30s) |
| Client goes to sleep (mobile) | No Pong | Server evicts after timeout; client reconnects on wake |

### 6.2 Reconnection Strategy (Client-Side)

```
function connect() {
    const ws = new WebSocket('wss://...');
    let retryDelay = 1000; // start at 1s

    ws.onclose = () => {
        setTimeout(() => {
            retryDelay = Math.min(retryDelay * 2, 30000); // cap at 30s
            retryDelay += Math.random() * 1000;            // jitter
            connect();
        }, retryDelay);
    };

    ws.onopen = () => {
        retryDelay = 1000; // reset on success
        ws.send(JSON.stringify({ type: 'RESUME', lastSeqId: '...' }));
    };
}
```

**Critical:** Always add **jitter** to backoff. Without it, a server restart causes a **thundering herd** of reconnections.

### 6.3 Message Ordering and Delivery Guarantees

WebSocket over TCP guarantees **in-order delivery per connection**. But in a distributed system:

- Messages via Pub/Sub may arrive **out of order** across servers.
- If a client disconnects and reconnects to a different server, it may **miss messages**.

**Solutions:**
- **Sequence IDs** on every message. Client tracks last received seq ID and requests a replay on reconnect.
- **Server-side buffer** — keep last N messages per room/channel in Redis or an in-memory ring buffer.
- **Persistent message log** (Kafka, Redis Streams) — client can seek to its last offset.

### 6.4 Split Brain / Partition

If the Pub/Sub bus becomes partitioned:
- Server-1 and Server-2 can still serve their local clients.
- Cross-server messages are lost / delayed.
- **Mitigation:** Health checks on pub/sub; degrade gracefully (show "reconnecting…" to users); use a pub/sub system with replication (Redis Sentinel/Cluster, Kafka with ISR).

---

## 7. Security Considerations

| Concern | Solution |
|---|---|
| Authentication | Authenticate during the HTTP handshake (cookie, token in query param, or first WS message). WS itself has no auth. |
| Authorization | Validate permissions on every inbound message, not just at connect time. |
| Origin validation | Check `Origin` header during handshake to prevent CSRF. |
| Rate limiting | Per-connection message rate limiter (token bucket). Disconnect abusive clients. |
| Payload validation | Validate and sanitize every message. Never trust client-sent data. |
| DoS — connection flood | Limit connections per IP. Use a connection admission controller. |
| DoS — slowloris | Set timeouts on handshake completion. Drop connections that don't upgrade in time. |
| Encryption | Always use `wss://` in production (TLS). |

---

## 8. Alternatives and Tradeoffs

### 8.1 Comparison Matrix

| Feature | WebSocket | SSE | HTTP Long Polling | HTTP Short Polling | gRPC Streaming |
|---|---|---|---|---|---|
| Direction | Bidirectional | Server → Client only | Server → Client (simulated) | Client → Server | Bidirectional |
| Protocol | WS (TCP) | HTTP/1.1 | HTTP/1.1 | HTTP/1.1 | HTTP/2 |
| Connection | Persistent | Persistent | Held open, recycled | New per request | Persistent |
| Overhead per msg | Very Low (2-14 bytes framing) | ~50 bytes (SSE format) | Full HTTP headers | Full HTTP headers | Low (HTTP/2 frames) |
| Browser support | All modern | All modern (no IE) | All | All | Needs grpc-web proxy |
| Proxy/firewall friendly | Sometimes blocked | Yes (it's HTTP) | Yes | Yes | Moderate |
| Auto-reconnect | Manual | Built-in (EventSource) | Manual | N/A | Manual |
| Scalability complexity | High | Moderate | Moderate | Low | High |
| Best for | Chat, gaming, collab editing | Live feeds, notifications | Legacy compat | Dashboards, low-freq updates | Microservice streaming |

### 8.2 When to Choose What

```
Need bidirectional + low latency?
├── Yes → WebSocket
│         (chat, multiplayer games, collaborative editing)
└── No
    ├── Server push only + browser client?
    │   └── SSE (Server-Sent Events)
    │       (notifications, live feeds, stock tickers)
    ├── Microservice-to-microservice streaming?
    │   └── gRPC Bidirectional Streaming
    │       (telemetry, log streaming, inter-service events)
    └── Simple, infrequent updates?
        └── HTTP Polling
            (dashboard refresh every 30s)
```

### 8.3 SSE as a Simpler Alternative (Deep Dive)

Many systems that use WebSockets actually only need **server → client** push. SSE is often the better choice:

- Works over standard HTTP — no proxy issues.
- Built-in reconnection with `Last-Event-ID` header.
- Natively supported by `EventSource` API.
- Uses text/event-stream MIME type.

**Limitation:** Client → server communication still requires separate HTTP requests (POST). For most apps (notifications, feeds), this is perfectly fine.

---

## 9. Real-World Architecture Examples

### 9.1 Chat System (e.g., Slack, Discord)

```
[Mobile/Web Clients]
        |
   [API Gateway] ── REST for auth, history, file uploads
        |
   [WS Gateway Layer] ── Handles persistent connections
        |                  Authenticates on handshake
        |                  Maintains user→server mapping in Redis
        |
   [Redis Pub/Sub] ── Channel per room/conversation
        |
   [Message Service] ── Writes to DB, triggers notifications
        |
   [Cassandra/Scylla] ── Message persistence (time-series)
   [Redis/Memcached]  ── Recent messages cache
   [S3]               ── File/media storage
```

**Key Design Decisions:**
- WS Gateway is **stateless** except for the socket map. Can be horizontally scaled.
- Message Service is a separate microservice — decoupled from connection handling.
- Presence (online/offline) tracked via heartbeats in Redis with TTL keys.

### 9.2 Live Dashboard / Stock Ticker

```
[Data Sources] → [Kafka] → [Stream Processor] → [WS Fan-out Service] → [Clients]
```

- Kafka ingests high-volume data.
- Stream processor (Flink/Spark) aggregates, filters, computes.
- WS fan-out service subscribes to processed topics and pushes to connected clients.
- Clients subscribe to specific "channels" (tickers, metrics).

### 9.3 Collaborative Editing (e.g., Google Docs)

```
[Clients] ←──ws──→ [WS Server] ←→ [CRDT/OT Engine] ←→ [Document Store]
```

- Uses **Operational Transformation (OT)** or **CRDTs** for conflict resolution.
- Every keystroke/operation is sent as a WS message.
- Server applies OT/CRDT logic to merge concurrent edits.
- Transformed operations are broadcast to all other clients in the document session.

---

## 10. Monitoring and Observability

### Key Metrics to Track

| Metric | Why |
|---|---|
| Active connections (per server, total) | Capacity planning, detect leaks |
| Connection rate (new/sec) | Detect thundering herd, DDoS |
| Message throughput (in/out per sec) | Performance baseline |
| Message latency (p50, p95, p99) | SLA compliance |
| Error rate (failed handshakes, abnormal closes) | Detect issues early |
| Pub/Sub lag | Cross-server delivery health |
| Memory / FD usage per server | Prevent OOMs and FD exhaustion |
| Reconnection rate | Detect instability |

### Health Check Design

WebSocket servers should expose an **HTTP health endpoint** (not WS) for the load balancer:
- `/health` → 200 OK if accepting new connections
- `/ready` → 200 OK if fully initialized and pub/sub connected
- Drain mode: return 503 on `/health` to stop new connections, but keep existing ones alive for graceful shutdown.

---

## 11. Common Interview Questions and Answers

### Q: "How would you scale WebSockets to 10M concurrent users?"

**Answer Framework:**
1. **Horizontal scaling** — N WebSocket gateway servers behind an L4 LB.
2. **Pub/Sub backplane** — Redis Cluster or Kafka for cross-server messaging.
3. **Connection routing** — Consistent hash on user/room ID for locality.
4. **Tiered architecture** — Separate WS gateways (stateful, lightweight) from business logic servers (stateless).
5. **Edge deployment** — WS gateways in multiple regions; inter-region pub/sub replication.
6. **Numbers:** ~500K connections per server × 20 servers = 10M. Add headroom for failover.

### Q: "What happens when a WebSocket server crashes?"

**Answer:**
- All connections on that server drop (TCP RST).
- Clients detect via `onclose` event and reconnect with exponential backoff + jitter.
- LB detects failed health check, removes server from pool.
- Clients reconnect to healthy servers.
- On reconnect, clients send their last sequence ID; server replays missed messages from the message log (Redis Streams / Kafka).
- No data loss if the message pipeline persists messages independently of the WS server.

### Q: "WebSocket vs gRPC streaming — when would you pick which?"

**Answer:**
- **WebSocket** — browser clients, wide compatibility, text-heavy protocols, simpler tooling.
- **gRPC streaming** — service-to-service, already in a gRPC ecosystem, need strong typing (Protobuf), need bidirectional streaming with backpressure, multiplexing over HTTP/2.
- **Hybrid** — gRPC internally between microservices, WebSocket at the edge for browser clients.

### Q: "How do you handle authentication with WebSockets?"

**Answer:**
- During the HTTP upgrade handshake — validate a JWT/session cookie.
- Alternatively, after connection, the first message must be an auth token; server validates and assigns identity.
- For token expiry: client sends a refresh message before expiry, or server closes the connection and client re-authenticates on reconnect.
- Never put sensitive tokens in the URL query string in production (they leak in logs).

### Q: "How do you handle backpressure?"

**Answer:**
- If the server produces messages faster than a client can consume, the send buffer fills up.
- Monitor the `bufferedAmount` property on the server-side socket.
- If buffer exceeds a threshold, either drop low-priority messages, batch/aggregate, or disconnect the slow client.
- On the pub/sub side: use consumer groups with acknowledgment (Redis Streams, Kafka) so slow consumers don't lose messages but also don't block fast ones.

---

## 12. Quick Reference — Protocol Cheat Sheet

| Aspect | Detail |
|---|---|
| Default Port | 80 (ws), 443 (wss) |
| Handshake | HTTP/1.1 Upgrade (GET → 101) |
| Framing overhead | 2 bytes (small msg) to 14 bytes (large + masked) |
| Max frame size | Protocol allows 2^63 bytes; practical limit set by server config |
| Ping/Pong | Opcode 0x9 / 0xA; used for keepalive |
| Close handshake | Both sides send close frame (0x8) with status code |
| Subprotocols | Negotiated via `Sec-WebSocket-Protocol` header |
| Extensions | e.g., `permessage-deflate` for compression |
| Masking | Client→Server: always masked. Server→Client: never masked. |

---

## 13. One-Page Summary for Quick Revision

```
WEBSOCKETS IN 60 SECONDS
─────────────────────────
WHAT:    Full-duplex, persistent TCP connection via HTTP upgrade.
WHEN:    Bidirectional, low-latency, high-frequency messaging.
WHEN NOT: Server-push only (use SSE), infrequent updates (use polling).

SCALING:
  • Horizontal WS servers + L4 LB with sticky sessions
  • Pub/Sub backplane (Redis/Kafka/NATS) for cross-server fan-out
  • Tune file descriptors, ports, kernel buffers
  • ~500K-1M idle connections per server; throughput is the real limit

FAILURES:
  • Client: exponential backoff + jitter on reconnect
  • Server: stateless gateways; message log for replay
  • Network: Ping/Pong heartbeats; proxy-friendly timeouts

SECURITY:
  • Auth on handshake (JWT/cookie), validate every message
  • Rate limit per connection, origin check, always wss://

TRADEOFFS:
  • Pro: lowest latency, true bidirectional, minimal overhead
  • Con: stateful (scaling complexity), no caching, proxy issues
```
