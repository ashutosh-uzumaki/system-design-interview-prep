# System Design (HLD) Interview — Deep Dive Reference Guide

---

## Table of Contents

1. [Realtime Updates](#1-realtime-updates)
2. [Fanout Pattern](#2-fanout-pattern)
3. [High Read Traffic](#3-high-read-traffic)
4. [High Write Traffic](#4-high-write-traffic)
5. [Handling Hot Keys](#5-handling-hot-keys)
6. [Handling Traffic Spikes](#6-handling-traffic-spikes)
7. [Handling Large Files](#7-handling-large-files)
8. [Media Streaming](#8-media-streaming)
9. [Handling Location Data](#9-handling-location-data)
10. [Generating Unique IDs](#10-generating-unique-ids)
11. [Distributed Counting](#11-distributed-counting)
12. [Leader Election](#12-leader-election)
13. [Failure Detection](#13-failure-detection)
14. [Handling Failures](#14-handling-failures)
15. [Recommendations](#15-recommendations)
16. [Multi-Tenancy](#16-multi-tenancy)
17. [Multi-Region Architecture](#17-multi-region-architecture)
18. [Deduplicating Data](#18-deduplicating-data)
19. [Distributed Transactions](#19-distributed-transactions)
20. [Removing Single Points of Failure](#20-removing-single-points-of-failure)

---

## 1. Realtime Updates

### Core Concept

Pushing data to clients the instant it changes on the server, rather than waiting for the client to ask. Think chat messages, live scores, stock tickers, collaborative editing.

### Techniques

| Technique | Direction | Latency | Complexity | Connection Cost |
|---|---|---|---|---|
| Short Polling | Client → Server (repeated) | High (poll interval) | Low | Low per request, high aggregate |
| Long Polling | Client → Server (held open) | Medium | Medium | Medium |
| Server-Sent Events (SSE) | Server → Client (unidirectional) | Low | Low | One persistent HTTP connection |
| WebSockets | Bidirectional | Very Low | High | One persistent TCP connection |

### Deep Dive: WebSockets vs SSE

**WebSockets** open a full-duplex TCP channel after an HTTP upgrade handshake. Ideal when both sides need to send data frequently (chat, multiplayer games). The trade-off is operational complexity: load balancers must support sticky sessions or Layer-4 routing, and you need a pub/sub backbone (Redis Pub/Sub, Kafka) so that any server in the fleet can push to any connected client.

**SSE** is simpler — it's just a long-lived HTTP response with `Content-Type: text/event-stream`. Because it's plain HTTP, it works naturally with HTTP/2 multiplexing and standard load balancers. The limitation is that the client can only receive, not send, so you'd pair it with normal REST calls for the upload path. Great fit for notifications, live dashboards, and feeds.

### Architecture Pattern

```
Client ←→ WebSocket Gateway (stateful)
                 ↓
          Message Broker (Redis Pub/Sub / Kafka)
                 ↓
          Backend Services (stateless)
```

The gateway maintains the connection map (userId → connectionId). When a backend service wants to push to user X, it publishes to the broker; the gateway instance holding X's connection picks it up and writes to the socket.

### Failure Modes & Mitigations

- **Connection drops**: Clients must implement exponential backoff reconnect with jitter. On reconnect, the client sends the last received event ID so the server can replay missed messages from a short-lived buffer (e.g., Redis Stream with a TTL).
- **Server crash**: All connections on that instance are lost. The connection registry should have TTLs; stale entries get cleaned. Clients reconnect to another instance.
- **Thundering herd on reconnect**: If a gateway goes down, thousands of clients reconnect simultaneously. Use jitter on the client side and admission control (connection rate limiting) on the server side.
- **Message ordering**: WebSockets guarantee order per connection, but if messages arrive from multiple backend producers, you need sequence numbers so the client can reorder.

### Trade-offs

| Pros | Cons |
|---|---|
| Sub-second latency | Stateful connections complicate scaling |
| Reduced server load vs polling | Requires sticky sessions or L4 LB |
| Efficient for high-frequency updates | Connection limits per server (~50K–100K) |

### Interview Questions

1. "Design a live comments feature for a streaming platform." → WebSocket gateway + Kafka topic per stream + fan-out to all viewers.
2. "How do you scale WebSocket servers?" → Horizontal scaling with shared pub/sub, sticky sessions at LB, connection draining on deploys.
3. "What happens if the user's network is flaky?" → Client-side heartbeat, reconnect with last event ID, server-side message buffer for replay.
4. "Why not just use polling?" → At scale, polling at 1s intervals with 10M users = 10M QPS to the server; WebSockets reduce this to near-zero idle cost.

---

## 2. Fanout Pattern

### Core Concept

When one event needs to reach many recipients. Classic example: a celebrity with 100M followers posts a tweet — how does it appear in every follower's feed?

### Two Strategies

**Fanout-on-Write (Push Model)**

When the author publishes, the system immediately writes the post into every follower's feed (typically a precomputed timeline in cache or a lightweight store).

- Pros: Reads are instant — just fetch the precomputed feed.
- Cons: Write amplification is extreme for high-follower users. A user with 100M followers triggers 100M writes. This is slow and expensive.

**Fanout-on-Read (Pull Model)**

Nothing is precomputed. When a user opens their feed, the system fetches the latest posts from all the people they follow, merges, ranks, and returns.

- Pros: No write amplification. Publishing is a single write.
- Cons: Read latency is high because you're doing a scatter-gather across many follow relationships at request time.

**Hybrid (What Twitter/X Actually Does)**

- For normal users (< some threshold of followers): fanout-on-write. Their posts are pushed into follower timelines.
- For celebrities (> threshold): fanout-on-read. Their posts are NOT pushed; instead, at read time, the system merges the precomputed timeline with a live fetch of celebrity posts.

This bounds the worst-case write amplification while keeping reads fast for the common case.

### Architecture

```
Post Service → [Is celebrity?]
                    │
        ┌───────────┴───────────┐
        No                      Yes
        ↓                       ↓
  Fanout Workers            Store in
  (write to each            celebrity
  follower's cache)         posts table
        ↓                       ↓
  Timeline Cache ←── merge at read time
```

### Failure Modes

- **Fanout lag**: If a user has 10M followers, the fanout workers may take minutes. Followers see a stale feed until their entry is written. Mitigation: prioritize active users — fans who opened the app in the last N hours get written first.
- **Worker crash mid-fanout**: Some followers get the post, others don't. Use a durable queue (Kafka) with consumer offsets so workers can resume.
- **Cache eviction**: If the timeline cache evicts entries, the read path must fall back to a reconstructed timeline from the database. Always have a "cold start" path.
- **Thundering herd on celebrity post**: If 50M users fetch the celebrity's latest post simultaneously, the celebrity posts table becomes a hot key. Put it in a distributed cache with replicas.

### Trade-offs Summary

| | Fanout-on-Write | Fanout-on-Read |
|---|---|---|
| Write cost | O(followers) | O(1) |
| Read cost | O(1) | O(following) |
| Feed freshness | Delayed by fanout lag | Always fresh |
| Best for | Users with few followers | Celebrities |
| Storage | High (duplicate entries) | Low |

### Interview Questions

1. "Design a news feed system." → Hybrid fanout, explain the celebrity threshold.
2. "A user with 50M followers posts — walk me through what happens." → Queue the post, skip fanout-on-write, mark as celebrity post. On read, merge.
3. "How do you handle deletes in fanout-on-write?" → Another fanout to remove entries, or lazy deletion (filter at read time and clean up async).

---

## 3. High Read Traffic

### Core Concept

Systems where reads vastly outnumber writes (100:1 or more). Examples: product catalogs, news articles, user profiles, configuration services.

### Strategy Stack (in order of impact)

1. **CDN (Edge Caching)**: Serve static and semi-static content from edge nodes closest to the user. This eliminates the request from ever reaching your origin. Use cache-control headers wisely: `max-age` for truly static assets, `stale-while-revalidate` for content that can be slightly stale while a background refresh happens.

2. **Application-Level Cache (Redis/Memcached)**: For dynamic but frequently accessed data. Use a cache-aside pattern: the application checks the cache first; on a miss, it reads from the DB and populates the cache. Alternatively, use read-through where the cache itself handles the DB fetch.

3. **Read Replicas**: Scale the database read path by adding replicas. Writes go to the primary; reads are distributed across replicas. Watch out for replication lag — a user who just wrote data might read from a replica that hasn't caught up yet. Mitigate with "read-your-own-writes" consistency: route the writing user's reads to the primary for a short window after a write.

4. **Materialized Views / Precomputation**: For expensive queries (joins, aggregations), precompute the result and store it. Update it on a schedule or via change data capture (CDC). This trades write-time computation for read-time speed.

5. **Database Query Optimization**: Proper indexing, query plan analysis, denormalization where appropriate. A covering index that satisfies a query entirely from the index without touching the table can be a 10x improvement.

### Caching Patterns

| Pattern | How It Works | Best For |
|---|---|---|
| Cache-Aside | App checks cache → miss → read DB → populate cache | General purpose |
| Read-Through | Cache itself fetches from DB on miss | Simplifies app code |
| Write-Through | Write to cache + DB simultaneously | Strong consistency needs |
| Write-Behind | Write to cache, async flush to DB | High write throughput |
| Refresh-Ahead | Proactively refresh before expiry | Predictable access patterns |

### Failure Modes

- **Cache stampede / thundering herd**: A popular key expires, and thousands of requests simultaneously hit the database. Mitigate with: (a) locking — only one request fetches from DB, others wait; (b) early expiration — refresh the cache before TTL expires; (c) stale-while-revalidate — serve stale data while refreshing in the background.
- **Cache avalanche**: Many keys expire at the same time (e.g., all set with the same TTL). Add random jitter to TTLs.
- **Cache penetration**: Requests for keys that will never exist in the DB bypass the cache every time. Use a bloom filter in front of the cache to reject impossible keys, or cache the null result with a short TTL.
- **Inconsistency between cache and DB**: After a write, the cache may hold stale data. Use cache invalidation (delete the key on write) rather than cache update to avoid race conditions. If two concurrent writes update the cache, the last one wins, which may not be the latest DB state. Delete + let next read repopulate is safer.
- **Replication lag on read replicas**: Mitigate with read-your-own-writes as described above. For critical reads (e.g., after payment), always read from primary.

### Trade-offs

| Pros | Cons |
|---|---|
| Sub-millisecond reads from cache | Cache invalidation complexity |
| Horizontal scaling via replicas | Replication lag / stale reads |
| CDN offloads origin completely | Cache warming on cold start |
| Reduced DB load by 90%+ | Memory cost of caching layer |

### Interview Questions

1. "Design a product page for an e-commerce site with 100M daily views." → CDN for images/static, Redis for product metadata, read replicas for long-tail queries.
2. "How do you ensure a user sees their own update immediately?" → Read-your-own-writes: sticky session to primary or version-token that routes to primary if the replica is behind.
3. "What's the difference between cache-aside and read-through?" → In cache-aside, the application is responsible for the DB fetch; in read-through, the cache library handles it transparently.

---

## 4. High Write Traffic

### Core Concept

Systems where ingestion rate is extreme: logging platforms (millions of events/sec), IoT telemetry, social media post ingestion, analytics event collection.

### Strategy Stack

1. **Write-Ahead Log (WAL) + Batching**: Instead of writing each event individually to the database, buffer writes in memory and flush in batches. This turns thousands of random I/O operations into a few sequential ones. Kafka inherently does this — producers batch messages, and Kafka appends them sequentially to a log.

2. **Message Queues / Event Streaming**: Decouple the write acceptance from the write processing. The API server accepts the write, puts it on a queue (Kafka, SQS), and returns 202 Accepted. Background consumers process the queue at their own pace. This absorbs spikes and prevents the database from being overwhelmed.

3. **Database Sharding**: Distribute writes across multiple database instances based on a shard key. Each shard handles a fraction of the total write load. Choose the shard key carefully — it should distribute writes evenly and align with your query patterns.

4. **LSM-Tree Based Storage**: Databases like Cassandra, RocksDB, and LevelDB use Log-Structured Merge Trees, which are optimized for writes. All writes go to an in-memory table (memtable), which is periodically flushed to sorted files on disk (SSTables). Reads are slower (must check multiple levels), but writes are extremely fast.

5. **CQRS (Command Query Responsibility Segregation)**: Separate the write model from the read model entirely. Writes go to a write-optimized store; an async process transforms and loads data into a read-optimized store. This lets you optimize each path independently.

### Failure Modes

- **Queue backlog**: If consumers can't keep up, the queue grows unboundedly. Set up monitoring and auto-scaling for consumers. Have a dead-letter queue for poison messages that repeatedly fail.
- **Data loss in buffering**: If the application crashes before flushing the in-memory buffer, data is lost. Use a WAL: write to a local append-only file before acknowledging, then flush from the WAL to the database.
- **Shard hotspots**: If the shard key is poorly chosen (e.g., by date — all writes go to "today's" shard), one shard gets all the traffic. See the "Hot Keys" section.
- **Write amplification in LSM trees**: Compaction (merging SSTables) consumes I/O and can compete with incoming writes. Tune compaction strategies (leveled vs. size-tiered) based on your read/write ratio.
- **Replication lag in CQRS**: The read model may be seconds or minutes behind the write model. Users may not see their own writes immediately. Design the UI to show optimistic updates.

### Trade-offs

| Approach | Write Throughput | Read Latency | Consistency | Complexity |
|---|---|---|---|---|
| Synchronous DB writes | Low | Low (strong consistency) | Strong | Low |
| Async via queue | Very High | Higher (eventual) | Eventual | Medium |
| CQRS | Very High | Low (optimized read model) | Eventual | High |
| Sharded DB | High | Low | Strong per shard | Medium-High |

### Interview Questions

1. "Design a logging/analytics platform ingesting 1M events/sec." → Kafka for ingestion, batch consumers writing to a columnar store (ClickHouse/Druid), separate read API.
2. "How do you prevent data loss when buffering writes?" → WAL on the producer side, acks from Kafka (acks=all for durability), replication factor ≥ 3.
3. "When would you use CQRS?" → When read and write patterns are fundamentally different. E.g., writes are append-only events, reads are complex aggregations.

---

## 5. Handling Hot Keys

### Core Concept

A "hot key" is a single key (in a cache, database partition, or message queue) that receives a disproportionate share of traffic. Examples: a viral tweet's ID, a celebrity's profile, a trending product, a popular hashtag.

### Why It's Dangerous

Even with sharding, all requests for a hot key go to the same shard. That shard becomes a bottleneck while other shards sit idle. This defeats the purpose of horizontal scaling.

### Mitigation Strategies

1. **Key Splitting / Suffixing**: Append a random suffix (e.g., `key_0`, `key_1`, ..., `key_N`) to distribute the hot key across N shards. On read, query all N suffixed keys and aggregate. This spreads the load at the cost of read fan-out.

2. **Local Caching**: Cache the hot key's value in each application server's local memory (with a very short TTL, e.g., 1–5 seconds). This absorbs the majority of reads without hitting the shared cache at all. The trade-off is slightly stale data.

3. **Replicated Cache Slots**: In Redis Cluster, you can create read replicas for specific slots. More replicas for the hot key's slot means more read capacity.

4. **Request Coalescing**: If 1000 requests arrive for the same key within a short window, only one actually fetches from the backend; the rest wait and share the result. Libraries like `singleflight` (Go) implement this.

5. **Adaptive Sharding / Consistent Hashing with Virtual Nodes**: If you detect a hot key, dynamically split its shard into sub-shards. This requires a more sophisticated routing layer.

### Detection

- Monitor per-key access rates in your cache (Redis `HOTKEYS` command, or custom instrumentation).
- Track partition-level throughput in your database (DynamoDB adaptive capacity, Cassandra per-partition metrics).
- Use sampling: log a fraction of requests and aggregate to find top keys.

### Failure Mode: Cascading Hot Key

A hot key overloads its shard → requests timeout → clients retry → more load → the shard crashes → traffic redistributes but another shard may now be overloaded (if using consistent hashing, the neighbor takes over the range). Mitigation: circuit breakers on the client side, rate limiting per key, and the strategies above.

### Interview Questions

1. "A product goes viral on your e-commerce site and the product page is timing out. What's happening and how do you fix it?" → Hot key in cache/DB. Local cache with short TTL, key splitting, request coalescing.
2. "How do you detect hot keys before they cause problems?" → Real-time monitoring, sampling, anomaly detection on per-partition metrics.

---

## 6. Handling Traffic Spikes

### Core Concept

Sudden, often unpredictable surges in traffic. Flash sales, breaking news, viral content, DDoS attacks, or simply organic growth spikes.

### Strategy Stack

1. **Auto-Scaling**: Configure auto-scaling groups (ASG) with metrics-based policies. Scale on CPU, request count, queue depth, or custom metrics. The problem: auto-scaling is reactive and takes minutes. You need a buffer for the gap.

2. **Over-Provisioning (Headroom)**: Run at 60–70% capacity normally so you have 30–40% headroom for spikes. More expensive, but provides instant absorption.

3. **Load Shedding**: When overwhelmed, intentionally drop low-priority requests. Return 503 with a Retry-After header. Prioritize: health checks > authenticated users > anonymous users > bots.

4. **Rate Limiting**: Token bucket or sliding window algorithms at the API gateway level. Per-user, per-IP, or per-API-key limits. This protects your backend from being overwhelmed.

5. **Queue-Based Buffering**: Accept requests into a queue and process them at a sustainable rate. The user gets an immediate acknowledgment and can poll for the result. Works for non-latency-sensitive operations (order processing, report generation).

6. **Circuit Breakers**: If a downstream service is failing, stop sending it traffic (open the circuit). Return a degraded response or a cached fallback. This prevents cascade failures.

7. **CDN / Static Fallback**: During extreme load, serve a static cached version of the page. Users see slightly stale content but the site stays up. "Stale is better than dead."

8. **Pre-scaling for Predicted Spikes**: If you know about an event (Black Friday, product launch), pre-scale infrastructure ahead of time. Warm up caches, provision extra capacity.

### Failure Modes

- **Auto-scale too slow**: The spike hits, new instances are spinning up, but the existing instances are overwhelmed and start dropping requests. Mitigation: headroom + load shedding during the gap.
- **Database doesn't scale**: You can auto-scale stateless app servers, but the database is often the bottleneck. Use connection pooling (PgBouncer), read replicas, and queue-based writes to protect the DB.
- **Cascade failure**: Service A is overloaded → it slows down → Service B (which depends on A) starts timing out → B's thread pool is exhausted → B fails → C fails, etc. Mitigation: circuit breakers, timeouts, bulkheads (isolate thread pools per dependency).

### Interview Questions

1. "Design a flash sale system." → Pre-scale, rate limit, queue-based order processing, inventory in Redis with atomic decrement, static fallback page if origin is overwhelmed.
2. "Your service is getting 10x normal traffic suddenly. Walk me through your response." → Load shedding → rate limiting → auto-scale triggers → investigate if it's organic or an attack → CDN absorbs what it can.
3. "How do you prevent one slow microservice from taking down the whole system?" → Circuit breakers, timeouts, bulkhead pattern, async communication where possible.

---

## 7. Handling Large Files

### Core Concept

Uploading, storing, processing, and serving files that are megabytes to gigabytes in size. Examples: video uploads, dataset ingestion, backup files, medical imaging.

### Upload Strategies

**Multipart / Chunked Upload**: Break the file into chunks (e.g., 5MB each). Upload each chunk independently. If a chunk fails, retry only that chunk. On the server side, reassemble. This is how S3 multipart upload works. The client gets an upload ID, uploads parts in parallel, and then calls "complete upload" to finalize.

**Pre-Signed URLs**: Instead of uploading through your API server (which would bottleneck your application), generate a pre-signed URL that lets the client upload directly to object storage (S3). Your server never touches the file bytes — it just orchestrates.

```
Client → Your API: "I want to upload a 2GB file"
Your API → S3: "Generate pre-signed upload URL"
Your API → Client: "Here's the URL, upload directly"
Client → S3: uploads directly
S3 → Your API (via event/webhook): "Upload complete"
Your API: triggers processing pipeline
```

**Resumable Uploads**: The tus protocol or Google's resumable upload protocol let clients resume interrupted uploads from where they left off. Essential for large files on unreliable networks.

### Storage

- **Object Storage (S3, GCS, Azure Blob)**: The standard for large file storage. Virtually unlimited capacity, 11 nines of durability, built-in replication. Store metadata (filename, size, owner, content type) in your database; store the actual bytes in object storage.
- **Tiered Storage**: Hot tier (frequent access, standard S3), warm tier (infrequent, S3-IA), cold tier (archival, Glacier). Lifecycle policies automatically move files between tiers based on age or access patterns.

### Processing

- **Async Processing Pipeline**: After upload, trigger a pipeline (via event) for transcoding, virus scanning, thumbnail generation, metadata extraction, etc. Use a queue (SQS/Kafka) to decouple upload from processing.
- **Streaming Processing**: For very large files, process them in a streaming fashion rather than loading the entire file into memory. E.g., parse a 10GB CSV line by line.

### Serving / Download

- **CDN**: Serve files through a CDN for low-latency downloads. Use signed URLs with expiration for access control.
- **Range Requests**: Support HTTP Range headers so clients can download portions of a file (essential for video seeking, PDF page loading, and resumable downloads).
- **Compression**: Gzip/Brotli for text-based files. For images, serve in modern formats (WebP/AVIF) with automatic format negotiation.

### Failure Modes

- **Incomplete upload**: Client disconnects mid-upload. With multipart upload, already-uploaded parts remain in S3 (and cost money). Set lifecycle policies to auto-delete incomplete multipart uploads after N days.
- **Processing failure**: A corrupted file crashes the transcoder. Use dead-letter queues, retry with backoff, and alerting. Validate file headers before processing.
- **Storage cost explosion**: Users upload unlimited files. Implement quotas, deduplication (content-addressable storage using file hash as key), and lifecycle policies.

### Interview Questions

1. "Design a file upload service for a cloud storage product like Dropbox." → Pre-signed URLs, chunked upload, dedup via content hashing, async processing, CDN for downloads.
2. "How do you handle a 10GB upload on a mobile network?" → Resumable chunked upload, each chunk retried independently, client tracks progress locally.

---

## 8. Media Streaming

### Core Concept

Delivering audio/video content to users in real time or near-real time. Two major categories: Video on Demand (VoD) and Live Streaming.

### Video on Demand (VoD)

**Adaptive Bitrate Streaming (ABR)**: The video is encoded at multiple quality levels (e.g., 240p, 480p, 720p, 1080p, 4K). Each quality level is split into small segments (2–10 seconds). The client's player monitors its bandwidth and buffer level, and dynamically switches between quality levels segment by segment. Protocols: HLS (Apple, most widely supported) and DASH (open standard).

**Pipeline**:
```
Raw Video → Transcoding (FFmpeg) → Multiple bitrate renditions
         → Segmenting (2-10s chunks)
         → Packaging (HLS .m3u8 manifest + .ts segments)
         → Upload to Object Storage
         → Serve via CDN
```

**Transcoding** is CPU-intensive. For a platform like YouTube, every uploaded video is transcoded into 10+ renditions. This is done asynchronously via a job queue, often on GPU-accelerated instances. A 1-hour 4K video might take several hours to transcode on a single machine, so you parallelize: split the video into segments, transcode segments in parallel on multiple workers, and stitch back together.

### Live Streaming

The additional challenge is latency. The pipeline is the same (ingest → transcode → segment → distribute via CDN), but it must happen in near-real time.

- **Ingest**: The broadcaster sends a stream via RTMP or SRT to an ingest server.
- **Transcoding**: Real-time transcoding into ABR renditions.
- **Distribution**: Segments are pushed to the CDN as they're produced. The CDN edge caches them.
- **Latency**: Standard HLS/DASH has 15–30 second latency. Low-latency HLS (LL-HLS) and CMAF with chunked transfer encoding reduce this to 2–5 seconds. For sub-second latency (e.g., auctions, live betting), you need WebRTC-based solutions.

### Failure Modes

- **CDN cache miss on live stream**: If a new segment isn't cached at the edge yet, the viewer's player stalls. Mitigation: push segments to CDN proactively (origin push) rather than waiting for pull.
- **Transcoder crash during live stream**: The stream goes down. Mitigation: hot standby transcoders that can take over immediately. Use redundant ingest points.
- **Buffering / rebuffering**: The player runs out of buffered segments because bandwidth dropped. ABR should detect this and switch to a lower bitrate. A good player algorithm (e.g., buffer-based like BBA or hybrid like MPC) is crucial.
- **Thundering herd on popular live event**: Millions of viewers request the same segment from the CDN. The CDN's edge is designed for this, but the origin must handle cache fill requests efficiently. Use request coalescing at the CDN origin shield.

### Trade-offs

| | VoD | Live |
|---|---|---|
| Latency | Not a concern (pre-encoded) | Critical (seconds matter) |
| Transcoding | Offline, can be thorough | Real-time, must be fast |
| Error tolerance | Can retry/re-encode | Must handle in real-time |
| CDN caching | Highly effective (static segments) | Effective but must refresh constantly |

### Interview Questions

1. "Design YouTube." → Upload pipeline (pre-signed URL → transcoding job queue → multi-bitrate encoding → CDN), metadata service, recommendation engine, search.
2. "Design a live streaming platform like Twitch." → RTMP ingest → real-time transcoder → HLS segmenter → CDN distribution. Chat via WebSocket separately.
3. "How do you minimize buffering for viewers in India on slow networks?" → ABR with low starting bitrate, short segment duration (2s for faster adaptation), CDN PoPs in India, preloading initial segments.

---

## 9. Handling Location Data

### Core Concept

Storing, indexing, and querying geospatial data efficiently. Use cases: ride-hailing (find nearby drivers), food delivery, store locators, geofencing.

### Data Structures & Indexing

**Geohash**: Encodes a (latitude, longitude) pair into a string (e.g., "tdr1y2"). Nearby locations share a common prefix. This lets you use a standard B-tree index on the geohash string and query with prefix matching. Limitation: points near a geohash cell boundary may be close in reality but have very different geohash prefixes. Solution: query the target cell and all 8 neighboring cells.

**Quadtree**: Recursively subdivides 2D space into four quadrants. Each node either holds a few points or is subdivided further. Good for in-memory spatial indexing with non-uniform point distributions (denser areas get finer subdivision). Used by Uber's H3-like systems internally.

**R-Tree**: A balanced tree of bounding rectangles. Used by PostGIS and most spatial databases. Efficient for range queries ("find all restaurants within this bounding box") and nearest-neighbor queries.

**S2 Geometry (Google) / H3 (Uber)**: Hierarchical spatial indexing systems that divide the Earth's surface into cells at multiple resolution levels. S2 uses a Hilbert curve to map 2D space to 1D, enabling efficient range queries on a single index. H3 uses hexagonal cells which have more uniform distance properties than square cells.

### "Find Nearby" Query Pattern

```
1. Convert user's (lat, lng) to a geohash / H3 cell at desired resolution
2. Determine the set of cells to search (target + neighbors)
3. Query the index: SELECT * FROM drivers WHERE geohash_cell IN (...)
4. For results, compute exact distance and filter by radius
5. Sort by distance, return top K
```

Step 4 is necessary because cell-based queries return a superset (the cell is a square/hexagon, not a circle).

### Tracking Moving Objects (e.g., Uber Drivers)

Drivers report their location every few seconds. You need a system that can handle millions of location updates per second and answer "nearest driver" queries with low latency.

- **In-Memory Geospatial Index**: Store driver locations in a Redis instance using GEOADD/GEOSEARCH, or in a custom in-memory quadtree.
- **Sharding by Geography**: Partition the world into regions (e.g., by city or by H3 cells at a coarse resolution). Each shard handles location updates and queries for its region.
- **Publish Location Updates**: Drivers publish to a Kafka topic partitioned by region. Consumers update the in-memory index.

### Failure Modes

- **Stale location data**: A driver's app loses connectivity; their last known location becomes stale. Mitigation: TTL on location entries. If not refreshed within N seconds, mark as unavailable.
- **Geohash boundary problem**: Two points 10 meters apart but in different geohash cells won't match a single-cell query. Always query neighboring cells.
- **Hotspot region**: Times Square has 10x the driver/rider density of a suburb. That geographic shard gets overloaded. Use finer-grained sharding in dense areas (adaptive resolution).

### Interview Questions

1. "Design Uber's matching system." → Driver location index (Redis GEOSEARCH or in-memory quadtree, sharded by city), rider request → query nearby drivers → rank by distance/ETA → match.
2. "How does geohashing work and what's its limitation?" → Encodes lat/lng into a string; nearby points share prefix. Limitation: boundary problem solved by querying neighbors.
3. "How do you handle millions of location updates per second?" → Kafka for ingestion, partitioned by region, consumers update in-memory geospatial index, short TTL for staleness.

---

## 10. Generating Unique IDs

### Core Concept

In a distributed system with no single database, generating globally unique, often sortable identifiers. Requirements vary: uniqueness (mandatory), sortability (often desired), low latency, no coordination.

### Approaches

| Approach | Format | Sortable? | Coordination | Collision Risk |
|---|---|---|---|---|
| UUID v4 | 128-bit random | No | None | Negligible (2^122 space) |
| UUID v7 | Timestamp + random | Yes | None | Negligible |
| Snowflake ID (Twitter) | 64-bit: timestamp + worker ID + sequence | Yes | Worker ID assignment | None (if worker IDs unique) |
| ULID | 128-bit: timestamp + random | Yes | None | Negligible |
| Database Auto-Increment | Sequential integer | Yes | Central DB | None |
| Ticket Server (Flickr) | Sequential from dedicated service | Yes | Ticket server | None |

### Snowflake ID Deep Dive

```
[1 bit unused][41 bits timestamp][10 bits machine ID][12 bits sequence]
```

- 41 bits of timestamp → ~69 years of millisecond precision.
- 10 bits of machine ID → 1024 unique workers.
- 12 bits of sequence → 4096 IDs per millisecond per worker.
- Total capacity: 4096 × 1024 = ~4M IDs per millisecond globally.

The beauty: IDs are roughly time-sorted (the high bits are the timestamp), which means they're index-friendly in databases (sequential inserts, no random page splits). No central coordinator needed at runtime — each worker generates IDs independently.

The challenge: assigning unique machine IDs. Options: ZooKeeper, etcd, or derive from IP/MAC address.

### Clock Skew Problem

Snowflake IDs depend on synchronized clocks. If a machine's clock goes backward (NTP adjustment), it could generate duplicate timestamps. Mitigation: if the clock goes backward, the worker either waits until the clock catches up or refuses to generate IDs (returns an error). Monitor clock skew across machines using NTP.

### UUID v4 vs Snowflake vs ULID

- **UUID v4**: Maximum simplicity, no coordination, but not sortable and 128 bits (larger storage, worse index performance).
- **Snowflake**: Compact (64 bits), sortable, but requires machine ID coordination.
- **ULID / UUID v7**: Best of both worlds — sortable, 128 bits, no coordination. ULID uses millisecond timestamp prefix + randomness. UUID v7 is the standardized version.

### Interview Questions

1. "Design a unique ID generation service for a distributed system." → Snowflake-style: describe bit layout, explain machine ID assignment via ZooKeeper, handle clock skew.
2. "Why not just use UUID v4?" → Works but: not sortable (bad for DB indexes), 128 bits (vs 64 for Snowflake), no embedded timestamp (can't extract creation time).
3. "What happens if the clock goes backward?" → Detect it, refuse to generate IDs until clock catches up, alert on monitoring.

---

## 11. Distributed Counting

### Core Concept

Counting things accurately at scale across multiple machines. Examples: view counts, like counts, unique visitors, rate limiting counters.

### The Challenge

A simple `counter++` doesn't work in distributed systems because it's not atomic across machines. Even with a single Redis instance, if you need exact counts under extreme throughput, you'll hit bottlenecks.

### Approaches

**Single Atomic Counter (Redis INCR)**: Simple and accurate. Redis INCR is atomic and fast (~100K ops/sec per key). Works until the counter itself becomes a hot key (millions of increments per second for a viral video).

**Sharded Counters**: Split the counter into N sub-counters (e.g., `views:video123:0`, `views:video123:1`, ..., `views:video123:9`). Each increment goes to a random sub-counter. To read the total, sum all sub-counters. This distributes write load at the cost of read fan-out.

**Local Aggregation + Periodic Flush**: Each app server maintains a local counter in memory. Periodically (every second or N increments), flush the local count to the central store. This dramatically reduces the write rate to the central store. Trade-off: the displayed count may lag by a few seconds.

**Probabilistic Counting (HyperLogLog)**: For counting unique items (e.g., unique visitors), HyperLogLog provides an approximate count with ~0.81% standard error using only 12KB of memory, regardless of the cardinality. Redis has built-in HLL support (PFADD, PFCOUNT). You can't get exact counts or list the items, but for analytics dashboards, the approximation is usually acceptable.

**CRDTs (Conflict-Free Replicated Data Types)**: G-Counter (grow-only counter) and PN-Counter (supports increment and decrement) are CRDTs that allow each replica to count independently and merge correctly. The merged count is the sum of all per-replica counts. Used in eventually consistent systems (Riak, Cassandra counters).

### Failure Modes

- **Lost counts with local aggregation**: If a server crashes before flushing its local buffer, those counts are lost. Accept this as a trade-off (it's just approximate counts) or use a WAL for the local buffer.
- **Double counting on retry**: A client increments, the server processes it but the ACK is lost, the client retries, the count is incremented twice. Solution: idempotency keys — each increment carries a unique ID; the server deduplicates.
- **Counter drift in CRDT**: If a node is partitioned for a long time, its local count diverges. Upon merge, the count is correct but there may be a sudden "jump" visible to users. Smooth it out in the UI.

### Interview Questions

1. "Design a view counter for YouTube that handles millions of increments per second." → Local aggregation on app servers + periodic batch flush to Redis sharded counters. Display is eventually consistent. Exact counts reconciled offline.
2. "How would you count unique visitors per day?" → HyperLogLog in Redis (PFADD per visitor, PFCOUNT for the total). 12KB per counter.
3. "What's the difference between sharded counters and CRDTs?" → Sharded counters split a single counter across slots in one system; CRDTs allow independent counters on separate replicas that merge correctly.

---

## 12. Leader Election

### Core Concept

In a distributed system, sometimes exactly one node must perform a particular task (e.g., run the scheduler, own a partition, coordinate a workflow). Leader election is the process by which nodes agree on who the leader is.

### Approaches

**ZooKeeper / etcd (Consensus-Based)**: Use a distributed consensus system. Nodes create an ephemeral sequential node in ZooKeeper (or acquire a lease in etcd). The node with the lowest sequence number is the leader. If the leader dies, its ephemeral node is deleted, and the next in line becomes leader. This relies on the consensus algorithm (ZAB for ZK, Raft for etcd) for correctness.

**Raft Consensus**: Nodes in a Raft cluster elect a leader via a voting protocol. The leader sends heartbeats; if followers don't receive one within a timeout, they start an election. A candidate needs a majority of votes. Raft guarantees at most one leader per term.

**Database-Based Locking**: Use a database row as a lock. `INSERT INTO leaders (service, node_id, expires_at) VALUES ('scheduler', 'node-3', NOW() + INTERVAL '30 seconds') ON CONFLICT DO NOTHING`. The node that succeeds is the leader. It must renew the lock before expiry. If it doesn't, another node can claim leadership.

**Lease-Based (Redis / DynamoDB)**: Acquire a lease (a lock with an expiration). Redis `SET lock_key node_id NX EX 30` — only succeeds if the key doesn't exist. The leader refreshes the lease periodically. If it fails to refresh (crash, network partition), the lease expires and another node acquires it.

### The Split-Brain Problem

The most dangerous failure: two nodes both believe they are the leader. This can happen if the old leader is partitioned but still alive. It continues acting as leader while a new leader is elected by the rest of the cluster.

Mitigations:
- **Fencing tokens**: Every time a new leader is elected, it gets a monotonically increasing token. Any operation the leader performs must include this token. The storage layer rejects operations with stale tokens.
- **Short leases with conservative timeouts**: The leader must stop acting before its lease could possibly expire. If the lease is 30 seconds, the leader considers itself no longer leader after 20 seconds and tries to renew.

### Failure Modes

- **Leader crash**: Detected by lease expiry or heartbeat timeout. New election starts. There's a brief period with no leader (availability gap).
- **Network partition**: If the leader is partitioned from the majority, it should step down (it can't renew its lease with the consensus quorum). A new leader is elected on the majority side.
- **GC pause**: A long garbage collection pause on the leader can look like a crash. The lease expires, a new leader is elected, then the old leader wakes up. Fencing tokens prevent the stale leader from causing harm.

### Interview Questions

1. "How do you ensure only one instance runs a cron job in a distributed system?" → Leader election via etcd lease. The leader runs the cron; others are standby. If the leader dies, another takes over.
2. "What's the split-brain problem?" → Two nodes think they're leader. Solve with fencing tokens and short leases.
3. "Why not just use a database lock?" → It works for simple cases, but database-based locks are slower and don't provide watches/notifications. ZooKeeper/etcd give you event-driven leader change notifications.

---

## 13. Failure Detection

### Core Concept

Determining whether a remote node in a distributed system is alive, dead, or unreachable. This is harder than it sounds because you can't distinguish between "the node is dead" and "the network between us is broken."

### Techniques

**Heartbeat**: Each node periodically sends a heartbeat (a small "I'm alive" message) to a monitor or to peer nodes. If no heartbeat is received within a timeout, the node is considered dead. Simple but requires careful timeout tuning: too short → false positives (declaring alive nodes as dead due to temporary network blips); too long → slow detection.

**Phi Accrual Failure Detector** (used in Cassandra): Instead of a binary alive/dead decision, it computes a "suspicion level" (phi) based on the statistical distribution of heartbeat arrival times. If heartbeats usually arrive every 1 second ± 200ms, and one is 5 seconds late, phi is very high → likely dead. This adapts to network conditions automatically.

**Gossip Protocol**: Nodes periodically exchange information about each other's health. Each node randomly picks a peer and shares its membership list with heartbeat timestamps. If multiple nodes agree that a peer's heartbeat is stale, it's marked as suspect, then dead. Used in Cassandra, Consul, SWIM protocol.

**Ping / ACK (SWIM Protocol)**: If node A suspects node B is dead (missed heartbeat), A asks K random other nodes to ping B on its behalf (indirect ping). If any of them get a response, B is alive. This reduces false positives from direct network issues between A and B.

### Failure Detection vs Failure Handling

Detection tells you *that* a node is down. Handling decides *what to do about it*. They're separate concerns. Detection should be fast and accurate; handling depends on the system's architecture (failover, rebalancing, alerting).

### Failure Modes (of the detector itself)

- **False positives**: Declaring a healthy node as dead. Causes unnecessary failovers, rebalancing, and potential instability. A node that's declared dead might still be processing requests, leading to split-brain.
- **False negatives**: Failing to detect a dead node. Requests continue to be routed to a dead node, causing errors.
- **Asymmetric partitions**: A can reach B, but B can't reach C, and A can reach C. Different nodes have different views of who's alive. Gossip helps converge these views.

### Interview Questions

1. "How does Cassandra detect node failures?" → Gossip-based failure detection with a phi accrual detector. Nodes exchange heartbeat state via gossip; the phi detector adapts thresholds to network conditions.
2. "Why not just use a simple timeout?" → Fixed timeouts don't adapt to varying network conditions. In a cross-datacenter setup, latencies vary widely. Phi accrual adapts automatically.
3. "What's the trade-off between fast detection and accuracy?" → Shorter timeout = faster detection but more false positives. Longer timeout = fewer false positives but slower detection. Phi accrual balances this.

---

## 14. Handling Failures

### Core Concept

Once a failure is detected, how does the system respond to maintain availability and correctness? This is the bread and butter of distributed systems design.

### Failure Categories

| Type | Example | Detection | Response |
|---|---|---|---|
| Crash failure | Process dies, server reboots | Heartbeat timeout | Failover to replica |
| Omission failure | Dropped messages/packets | Timeout on expected response | Retry + circuit breaker |
| Timing failure | Response is too slow | Deadline exceeded | Timeout + fallback |
| Byzantine failure | Node behaves arbitrarily (bug, hack) | Consistency checks | BFT consensus (rare) |

### Strategies

**Retries with Exponential Backoff and Jitter**: On transient failures, retry after increasing delays (1s, 2s, 4s, 8s, ...) with random jitter to prevent synchronized retries from many clients. Cap the number of retries and the max delay.

**Circuit Breaker Pattern**: Track the failure rate of calls to a dependency. If failures exceed a threshold, "open" the circuit — immediately return an error or fallback without calling the dependency. After a cool-down period, allow a few "probe" requests through (half-open state). If they succeed, close the circuit. States: Closed → Open → Half-Open → Closed.

**Bulkhead Pattern**: Isolate resources per dependency. Give each downstream service its own thread pool / connection pool. If service A is slow and exhausts its pool, it doesn't affect calls to service B. Named after ship bulkheads that contain flooding.

**Failover**: For stateful services with replicas, promote a replica to primary when the primary fails. Types: (a) hot standby — replica is always in sync, ready to serve immediately; (b) warm standby — replica is nearly in sync, needs brief catch-up; (c) cold standby — must be started and data restored from backup.

**Graceful Degradation**: When a non-critical component fails, continue serving a degraded experience. E.g., if the recommendation service is down, show popular items instead. If the comments service is down, show the article without comments.

**Idempotency**: Design operations so they can be safely retried without side effects. Use idempotency keys: the client sends a unique request ID, and the server deduplicates. Critical for payment processing and any state-changing operation.

**Saga Pattern for Distributed Transactions**: See the "Distributed Transactions" section below. In short: a chain of local transactions with compensating actions. If step 3 fails, run compensating actions for steps 2 and 1 to undo.

### Failure Mode: Cascading Failure

Service A calls B calls C. C is slow → B's threads are blocked waiting for C → B's thread pool is exhausted → A's calls to B fail → A fails. The entire system is down because of one slow service.

Prevention stack: timeouts (fail fast, don't wait forever) → circuit breakers (stop calling the failing service) → bulkheads (isolate the blast radius) → graceful degradation (serve without the failed dependency) → load shedding (drop excess load).

### Interview Questions

1. "A downstream payment service is intermittently failing. How do you handle it?" → Retry with backoff + circuit breaker. If open, return "payment pending" and process async when it recovers. Idempotency key to prevent double charges.
2. "Explain the circuit breaker pattern." → Three states (closed/open/half-open), failure threshold to open, probe requests to close. Like an electrical circuit breaker — protects the system from repeated failures.
3. "How do you prevent cascading failures?" → Timeouts, circuit breakers, bulkheads, async communication, graceful degradation.

---

## 15. Recommendations

### Core Concept

Suggesting relevant items to users based on their behavior, preferences, and item characteristics. Used in e-commerce, streaming, social media, news, and more.

### Approaches

**Collaborative Filtering**: "Users who liked X also liked Y." Two types:
- User-based: Find users similar to the target user (based on rating patterns), and recommend items those similar users liked.
- Item-based: Find items similar to items the user has liked (based on co-occurrence in user interactions). More stable and scalable than user-based.
- Matrix factorization (SVD, ALS): Decompose the user-item interaction matrix into latent factors. Each user and item is represented as a vector in latent space; the dot product predicts the interaction score.

**Content-Based Filtering**: Recommend items similar to what the user has interacted with, based on item features (genre, description, tags). Doesn't need other users' data. Limitation: can't discover serendipitous items that are dissimilar to past behavior.

**Hybrid**: Combine collaborative and content-based approaches. Most production systems are hybrids.

**Deep Learning / Embedding-Based**: Use neural networks to learn dense embeddings for users and items. Models like Two-Tower (separate user and item networks, trained on interaction data) allow efficient nearest-neighbor lookup in embedding space. YouTube, TikTok, and Spotify use variants of this.

### System Architecture (Two-Stage)

```
Stage 1: Candidate Generation (fast, broad)
  - Retrieve ~1000 candidates from millions of items
  - Methods: ANN (Approximate Nearest Neighbor) on embeddings, simple rules, popularity

Stage 2: Ranking (slow, precise)
  - Score each candidate with a complex ML model
  - Features: user history, item features, context (time, device), social signals
  - Return top K
```

This two-stage approach is necessary because running the complex ranking model on millions of items is too slow. Candidate generation is a fast prefilter.

### Cold Start Problem

- **New user**: No interaction history. Use content-based recommendations, popularity-based fallback, or ask onboarding questions.
- **New item**: No interaction data. Use content features, inject into a small fraction of users' feeds (exploration), and collect initial signals.

### Trade-offs

| Approach | Strengths | Weaknesses |
|---|---|---|
| Collaborative Filtering | Captures complex patterns, serendipity | Cold start, sparsity, popularity bias |
| Content-Based | Works for new users, explainable | Limited diversity, feature engineering |
| Deep Learning | State-of-the-art accuracy | Training cost, latency, black box |
| Hybrid | Best overall quality | Complexity |

### Interview Questions

1. "Design a movie recommendation system." → Two-stage: candidate generation (embedding ANN), ranking (ML model with user/item features). Collaborative filtering for warm users, content-based for cold start. Offline training pipeline + online serving.
2. "How do you handle the cold start problem?" → Onboarding survey, content-based initial recommendations, exploration (inject new items into feeds), bandits for balancing exploration/exploitation.
3. "How would you evaluate your recommendation system?" → Offline: precision@K, recall@K, NDCG. Online: A/B test measuring engagement (CTR, watch time, purchase rate). Also track diversity and novelty.

---

## 16. Multi-Tenancy

### Core Concept

A single instance of software serves multiple customers (tenants). Each tenant's data and experience must be isolated, but they share infrastructure for cost efficiency. SaaS products (Slack, Salesforce, AWS) are inherently multi-tenant.

### Isolation Models

| Model | Isolation | Cost | Complexity | Use Case |
|---|---|---|---|---|
| Shared Everything | Low | Lowest | Low | Small tenants, low risk |
| Shared DB, Separate Schema | Medium | Low-Medium | Medium | Medium tenants |
| Separate Database per Tenant | High | Medium-High | Medium | Compliance-sensitive |
| Separate Infrastructure | Highest | Highest | High | Enterprise / regulated |

**Shared Everything**: All tenants share the same database, tables, and application instances. Tenant isolation is enforced by including `tenant_id` in every query's WHERE clause. Risk: a bug that forgets the tenant filter leaks data across tenants.

**Separate Schema**: Each tenant gets their own database schema (or keyspace). Stronger isolation — a missing WHERE clause doesn't cross tenant boundaries. Migration overhead: schema changes must be applied to each tenant's schema.

**Separate Database**: Maximum data isolation. Each tenant has their own database instance. Expensive, but required by some regulations (data residency, HIPAA). Scaling means managing many database instances.

### Noisy Neighbor Problem

In shared infrastructure, one tenant's heavy usage degrades performance for others. A tenant running a massive report query can starve other tenants of database resources.

Mitigations:
- **Resource quotas**: CPU, memory, I/O limits per tenant (cgroups, database resource groups).
- **Rate limiting per tenant**: API rate limits and query concurrency limits.
- **Quality of Service (QoS) tiers**: Premium tenants get dedicated resources or higher limits.
- **Query timeouts**: Kill long-running queries after a threshold.
- **Tenant-aware scheduling**: Route heavy tenants to dedicated pools.

### Tenant-Aware Routing

The system must route each request to the correct data and resources. A request hits the API gateway, which extracts the tenant ID (from JWT token, subdomain, or API key), looks up the tenant's configuration (which database, which shard, which feature flags), and routes accordingly.

### Failure Modes

- **Data leak between tenants**: The most critical failure. A missing tenant_id filter or a cache key without the tenant prefix can expose one tenant's data to another. Mitigation: mandatory tenant_id in all queries (enforce at the ORM/framework level), tenant-scoped cache keys, regular security audits, penetration testing.
- **Tenant-wide outage**: A deployment breaks one tenant's custom configuration. Use canary deployments with tenant-aware canary selection.
- **Migration complexity**: Onboarding a new tenant or moving a tenant between shards requires careful data migration without downtime.

### Interview Questions

1. "Design a SaaS platform like Slack." → Multi-tenant architecture: shared infrastructure with tenant-aware routing, tenant_id in every data path, rate limiting per tenant, separate encryption keys per tenant.
2. "How do you prevent one tenant from affecting others?" → Noisy neighbor mitigations: resource quotas, rate limiting, QoS tiers, query timeouts.
3. "A bug exposed one tenant's data to another. How do you prevent this?" → Enforce tenant_id at framework level (middleware that injects tenant filter into every query), tenant-scoped cache keys, automated testing for cross-tenant access.

---

## 17. Multi-Region Architecture

### Core Concept

Deploying your system across geographically distributed data centers (regions) for lower latency (serve users from a nearby region), higher availability (survive a region-wide outage), and data residency compliance (keep EU data in EU).

### Patterns

**Active-Passive (Primary-Secondary)**: One region handles all writes (and optionally reads). Other regions are standby, receiving replicated data. On primary failure, a secondary is promoted. Simple but wastes standby resources and doesn't reduce latency for writes.

**Active-Active (Multi-Primary)**: All regions handle reads and writes for their local users. Data is replicated between regions. This is the holy grail but introduces the hardest problem: **conflict resolution**.

**Follow-the-Sun / Active-Active with Partitioned Writes**: Each user or piece of data has a "home region" (usually the one closest to them). Writes for that data always go to the home region. Other regions serve reads from replicated data. This avoids write conflicts while still distributing load.

### Data Replication

**Synchronous replication**: The write is only acknowledged after all replicas confirm. Strong consistency but high latency (cross-region round trip, 50–200ms+). Impractical for most use cases.

**Asynchronous replication**: The write is acknowledged after the local region confirms. Replicated to other regions in the background. Low latency but risk of data loss if the primary fails before replication completes. Also causes temporary inconsistency.

**Conflict Resolution (for multi-primary)**: When two regions modify the same record, you need a strategy: last-writer-wins (LWW) using timestamps (simple but can lose data), application-level merge logic (complex but correct), or CRDTs (data structures that merge automatically without conflicts).

### DNS & Traffic Routing

Use GeoDNS or a global load balancer (AWS Global Accelerator, Cloudflare) to route users to the nearest region. Implement health checks so that if a region fails, traffic is automatically rerouted.

### Failure Modes

- **Region-wide outage**: Entire region goes down (rare but happens — AWS us-east-1 has had several). If active-passive, failover to secondary. If active-active, other regions absorb the traffic. Key: ensure your DNS TTL is low enough for fast failover.
- **Replication lag**: A user writes in US-East, then reads from EU-West (e.g., on a different device or after a CDN redirect). The EU replica hasn't caught up yet, and the user doesn't see their data. Mitigate with sticky sessions to the home region, or "read-after-write" consistency using version vectors.
- **Split-brain between regions**: A network partition between regions causes both to accept writes independently. On reconnection, conflicts must be resolved. This is the CAP theorem in action — you can't have both consistency and availability during a partition.
- **Data residency violation**: An EU user's data is accidentally replicated to a US region, violating GDPR. Mitigation: strict data partitioning rules, region-tagged data, compliance audits.

### Trade-offs

| | Active-Passive | Active-Active |
|---|---|---|
| Write latency | Low (local writes) | Low (local writes) |
| Read latency | High for remote users | Low (local reads) |
| Consistency | Strong (single writer) | Eventual (multi-writer) |
| Availability | Requires failover | Naturally available |
| Complexity | Low | Very High |

### Interview Questions

1. "Design a globally distributed system." → Active-active with partitioned writes (user home region), async replication, GeoDNS for routing, conflict resolution strategy.
2. "How do you handle a full region outage?" → Failover: GeoDNS reroutes traffic, sessions are stateless (can be served anywhere), data is available from replicas. RPO depends on replication lag.
3. "How do you ensure GDPR compliance in a multi-region setup?" → Data partitioning by user region, no replication of EU data outside EU, tenant metadata tracks data residency requirements.

---

## 18. Deduplicating Data

### Core Concept

Ensuring that duplicate messages, events, or records don't cause duplicate processing or storage. In distributed systems, "exactly once" delivery is impossible in theory, but "effectively once" processing is achievable through idempotency.

### Why Duplicates Happen

- Network retries: A client sends a request, the server processes it, but the ACK is lost. The client retries, sending the same request again.
- At-least-once delivery: Message queues like Kafka guarantee at-least-once delivery, meaning the same message may be delivered more than once (e.g., after a consumer crash and rebalance).
- Producer retries: A Kafka producer retries a failed send, and the broker actually received the first one.

### Strategies

**Idempotency Keys**: The client generates a unique ID (UUID) for each logical operation and includes it in the request. The server stores processed IDs and checks before processing. If the ID is already seen, return the cached response without re-processing.

```
Client → Server: POST /payment {idempotency_key: "abc-123", amount: 100}
Server: Check if "abc-123" is in processed_keys table
  → Yes: return cached result
  → No: process payment, store "abc-123" → result, return result
```

**Kafka Exactly-Once Semantics (EOS)**: Kafka supports idempotent producers (assigns a producer ID and sequence number; the broker deduplicates) and transactional writes (atomic writes across multiple partitions). Combined with consumer offset management, this gives "effectively once" processing.

**Content-Based Deduplication**: Hash the content of the message/event. If two messages have the same hash, they're duplicates. Works well for file storage (content-addressable storage: the file's SHA-256 hash is its key) but less well for events with timestamps (same event at different times has a different hash).

**Deduplication Window**: You can't store all idempotency keys forever (unbounded storage). Define a window (e.g., 24 hours or 7 days). Within the window, duplicates are detected. After the window, a duplicate might slip through. Size the window based on the maximum expected retry delay.

**Bloom Filters for Approximate Dedup**: If you need to check billions of keys and can tolerate a small false positive rate (saying "seen" when it's new, but never saying "new" when it's seen), a bloom filter is space-efficient. Useful as a first-pass filter before checking the definitive store.

### Failure Modes

- **Dedup store unavailable**: If the idempotency key store is down, you have two choices: reject requests (safe but unavailable) or process without dedup (available but might double-process). Decision depends on the operation's criticality.
- **Key collision**: If using short or poorly generated keys, two different operations might hash to the same key, causing one to be falsely deduplicated. Use UUIDs or sufficiently long random keys.
- **Window too short**: If retries happen after the dedup window expires, duplicates slip through. Monitor retry patterns and size the window accordingly.

### Interview Questions

1. "How do you prevent double-charging in a payment system?" → Idempotency keys on the payment API. The client sends a unique key; the server checks before processing. Use a database unique constraint on the key.
2. "How does Kafka achieve exactly-once semantics?" → Idempotent producer (sequence numbers), transactional writes (atomic cross-partition), and consumer commits offsets transactionally with processing.
3. "How do you deduplicate events in a data pipeline?" → Content hash + dedup window in a fast store (Redis with TTL). Or use Kafka EOS end-to-end.

---

## 19. Distributed Transactions

### Core Concept

When a single business operation spans multiple services or databases, you need to ensure atomicity — either all operations succeed or none do. Traditional single-database ACID transactions don't work across services.

### Why It's Hard (CAP Theorem)

In a distributed system, during a network partition, you can only have Consistency or Availability, not both. Most systems choose availability (AP) and deal with eventual consistency. But some operations (financial transactions) absolutely require correctness, even at the cost of availability.

### Approaches

**Two-Phase Commit (2PC)**

A coordinator asks all participants to "prepare" (phase 1). If all agree, it sends "commit" (phase 2). If any says "no," it sends "abort."

- Pros: Strong consistency, correct.
- Cons: Blocking protocol — if the coordinator crashes after sending "prepare" but before sending "commit/abort," participants are stuck holding locks indefinitely. Low throughput due to lock contention. Poor fit for microservices.

**Three-Phase Commit (3PC)**: Adds a "pre-commit" phase to reduce the blocking window. In practice, rarely used because it doesn't fully solve the problem under network partitions.

**Saga Pattern**

The dominant approach for microservices. A saga is a sequence of local transactions, each in a separate service. If any step fails, compensating transactions are executed for all preceding steps to "undo" their effects.

Two coordination styles:
- **Choreography**: Each service listens for events and reacts. Service A completes → emits event → Service B listens and acts → emits event → Service C listens and acts. If C fails, it emits a failure event; B and A listen and run compensating actions. Simple, decoupled, but hard to track overall progress and debug.
- **Orchestration**: A central orchestrator tells each service what to do in sequence. If step 3 fails, the orchestrator triggers compensating actions for steps 2 and 1. Easier to understand and monitor, but the orchestrator is a potential single point of failure (mitigate with redundancy).

Example (E-commerce Order Saga):
```
1. Order Service: Create order (status: PENDING)
2. Payment Service: Charge customer
3. Inventory Service: Reserve items
4. Shipping Service: Schedule delivery
5. Order Service: Update order (status: CONFIRMED)

If step 3 fails:
  Compensate step 2: Refund customer
  Compensate step 1: Cancel order
```

**Outbox Pattern (Transactional Outbox)**

When a service needs to update its database AND publish an event atomically (e.g., save order + emit "order_created" event), you can't just do both because the database and the message broker are separate systems. The outbox pattern writes the event to an "outbox" table in the same database transaction as the business data. A separate process (CDC or poller) reads the outbox table and publishes events to the message broker.

```
Transaction:
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox (event_type, payload) VALUES ('order_created', '{...}');
COMMIT;

Background process:
  Read from outbox table → Publish to Kafka → Mark as published
```

This ensures the event is published if and only if the database transaction commits.

### Failure Modes

- **Saga partial completion**: Step 3 succeeds but step 4 fails. The compensating action for step 3 must be reliable. What if the compensation fails? → Retry with backoff. If compensation repeatedly fails, alert and manual intervention.
- **Non-compensatable actions**: Some actions can't be undone (sending an email, charging a credit card where the refund takes days). Design around this: put non-compensatable actions last in the saga, or use a "pending" state that's confirmed only after all steps succeed.
- **Outbox table grows unboundedly**: If the publisher fails, unprocessed events accumulate. Monitor outbox table size, alert on lag.
- **2PC coordinator crash**: Participants holding locks are blocked. Use timeout-based heuristics (auto-abort after N seconds) and persistent coordinator state for recovery.

### Trade-offs

| Approach | Consistency | Performance | Complexity | Best For |
|---|---|---|---|---|
| 2PC | Strong | Low (blocking) | Medium | Tightly coupled DBs |
| Saga (Choreography) | Eventual | High | High (debugging) | Loosely coupled services |
| Saga (Orchestration) | Eventual | High | Medium | Complex workflows |
| Outbox Pattern | Guarantees publish | High | Medium | DB + event atomicity |

### Interview Questions

1. "Design the order processing flow for an e-commerce platform." → Saga with orchestration: Order Orchestrator drives Payment → Inventory → Shipping. Each step has a compensating action. Use the outbox pattern for reliable event publishing.
2. "What's the problem with 2PC?" → Blocking: coordinator crash leaves participants stuck. Doesn't scale to many participants (microservices). High latency.
3. "How do you handle the case where a compensating action fails?" → Retry with exponential backoff. If it still fails, push to a dead-letter queue for manual resolution. Design compensations to be idempotent so retries are safe.

---

## 20. Removing Single Points of Failure

### Core Concept

A Single Point of Failure (SPOF) is any component whose failure brings down the entire system. The goal is to eliminate all SPOFs so no single component's death causes an outage.

### Common SPOFs and Solutions

| SPOF | Mitigation |
|---|---|
| Single database server | Primary-replica setup with automatic failover. For critical data, multi-AZ replicas |
| Single load balancer | Pair of LBs with VRRP / floating IP, or use a cloud-managed LB (inherently redundant) |
| Single DNS provider | Dual DNS providers (e.g., Route53 + Cloudflare) |
| Single region | Multi-region deployment (active-active or active-passive) |
| Single message broker node | Clustered broker (Kafka cluster with replication, RabbitMQ HA queues) |
| Single configuration server | ZooKeeper ensemble (3 or 5 nodes), etcd cluster with Raft |
| Single deployment pipeline | Multiple CI/CD runners, manual deploy fallback |
| Single network path | Multi-homed networking, multiple ISPs |

### Design Principles

1. **Redundancy**: Every critical component must have at least one backup. "N+1" means one more than needed; "N+2" means two more (can survive one failure while another is being repaired).

2. **Automatic Failover**: Redundancy is useless if failover requires manual intervention at 3 AM. Use health checks, leader election, and automated promotion. Test failover regularly (chaos engineering).

3. **Stateless Design**: Stateless services are trivially redundant — any instance can handle any request. Push state to dedicated stateful services (databases, caches) that have their own redundancy.

4. **Shared-Nothing Architecture**: Each node is self-sufficient and doesn't depend on shared state with other nodes. If one node fails, others continue independently.

5. **Avoid Hidden SPOFs**: The configuration management server, the monitoring system, the deployment pipeline, the TLS certificate renewal process — these are often overlooked SPOFs. Ask: "If this goes down, can we detect problems and deploy fixes?"

### Failure Modes

- **Correlated failures**: Redundant replicas on the same physical host, same rack, same power supply, or same AZ all fail together. Distribute replicas across failure domains (AZs, regions, racks).
- **Split-brain on failover**: Automatic failover promotes a replica, but the old primary comes back. Now you have two primaries. See the "Leader Election" section for fencing tokens.
- **Cascading failure from failover**: The primary database fails, traffic shifts to the replica, which is undersized and immediately overloaded. Mitigation: replicas should be provisioned to handle full load.
- **Dependency chains**: Service A → Service B → Service C. If C is a SPOF, then A and B effectively are too. Map your dependency graph and eliminate SPOFs at every level.

### Testing for SPOFs: Chaos Engineering

Intentionally inject failures (kill servers, drop network packets, fill disks) in production or staging and observe. Netflix's Chaos Monkey randomly kills instances; Chaos Kong simulates an entire region failure. The system should survive gracefully.

### Interview Questions

1. "How would you make this architecture highly available?" → Walk through each component, identify SPOFs, and add redundancy: load balancer (pair), app servers (multiple instances behind LB), database (primary + replicas in different AZs), cache (Redis Cluster or Sentinel), message broker (Kafka cluster).
2. "What's a hidden SPOF?" → The monitoring system, the certificate authority, the DNS provider, the deployment pipeline. If your monitoring is a SPOF, you won't know when other things fail.
3. "How do you test that your system has no SPOFs?" → Chaos engineering: randomly kill components and verify the system continues to operate. Game days: simulate outage scenarios with the team.

---

## Appendix: Quick Reference Cheat Sheet

| Concept | Key Technique | One-Line Trade-off |
|---|---|---|
| Realtime Updates | WebSocket + Pub/Sub | Low latency vs stateful connection management |
| Fanout | Hybrid push/pull | Write amplification vs read latency |
| High Read Traffic | Cache-aside + CDN + Read replicas | Speed vs staleness |
| High Write Traffic | Queue + Batch + LSM-tree | Throughput vs consistency lag |
| Hot Keys | Key splitting + local cache | Read fan-out vs write distribution |
| Traffic Spikes | Auto-scale + load shed + rate limit | Cost vs availability vs UX |
| Large Files | Chunked upload + pre-signed URL + CDN | Complexity vs reliability |
| Media Streaming | ABR (HLS/DASH) + CDN | Latency vs quality |
| Location Data | Geohash / H3 + in-memory index | Precision vs query speed |
| Unique IDs | Snowflake / ULID | Coordination vs sortability |
| Distributed Counting | Sharded counters / HyperLogLog | Accuracy vs throughput |
| Leader Election | Raft / etcd lease | Availability gap vs split-brain safety |
| Failure Detection | Phi accrual / gossip | Speed vs false positives |
| Handling Failures | Retry + circuit breaker + bulkhead | Availability vs complexity |
| Recommendations | Two-stage: candidate gen + ranking | Accuracy vs latency |
| Multi-Tenancy | Shared infra + tenant isolation | Cost efficiency vs isolation |
| Multi-Region | Active-active + async replication | Latency vs consistency |
| Deduplication | Idempotency keys + dedup window | Storage vs correctness |
| Distributed Transactions | Saga + outbox pattern | Eventual consistency vs simplicity |
| Remove SPOFs | Redundancy + auto-failover + chaos testing | Cost vs availability |

---

*Good luck with your HLD interviews. Remember: there's never one right answer — the interviewer wants to see you reason about trade-offs.*
