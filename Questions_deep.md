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

---

# Part 2: HLD Interview Questions & How to Answer Them

> For each building block, you get **real interview questions** that test the concept, a **structured answer framework**, and the **key points the interviewer is listening for**.

---

## 1. Realtime Updates — Interview Questions

### Q1: "Design a live collaboration tool like Google Docs where multiple users edit simultaneously."

**How to Answer:**

Start by clarifying: how many concurrent editors per document? (Say ~50). How many total documents? (Millions). Does the user need to see keystrokes in real time or can there be a small delay?

**Step 1 — Communication Layer:**
Use WebSockets for bidirectional real-time communication. Each user opens a WebSocket to a collaboration gateway. The gateway maintains a mapping of documentId → set of connected clients.

**Step 2 — Conflict Resolution:**
This is the core challenge. Two approaches:
- **Operational Transformation (OT)**: Used by Google Docs. Each edit is an "operation" (insert char at position X, delete char at position Y). A central server transforms concurrent operations to maintain consistency. Requires a centralized server per document.
- **CRDTs (Conflict-Free Replicated Data Types)**: Used by Figma. Each character has a unique ID and position between neighbors. No central server needed — operations merge automatically. More complex data structure but enables peer-to-peer and offline editing.

**Step 3 — Architecture:**
```
Clients ←WebSocket→ Collaboration Gateway
                         ↓
                  Document Session Server (one per active document)
                         ↓
                  Persistent Storage (document snapshots + operation log)
```

The Document Session Server holds the authoritative state of the document in memory, applies OT/CRDT, and broadcasts transformed operations to all connected clients.

**Step 4 — Scaling:**
Partition by documentId — each document's session lives on one server. Use consistent hashing to route. If a server dies, another takes over by replaying the operation log from storage.

**What the interviewer is listening for:**
- You know WebSockets are needed (not polling)
- You mention OT or CRDTs and can explain one
- You address what happens when two users edit the same word simultaneously
- You think about persistence (operation log + periodic snapshots)
- You consider server failure and how to recover document state

---

### Q2: "Design a notification system for an app like Instagram (likes, comments, follows, live video starts)."

**How to Answer:**

**Clarify:** Push notifications (mobile) or in-app real-time notifications or both? (Both). How many users? (500M). How many notifications/day? (Billions).

**Step 1 — Ingestion:**
Events (like, comment, follow) flow through a message queue (Kafka). A notification service consumes these events, applies rules (does the user have notifications enabled? Are they muted? Aggregate: "X and 42 others liked your photo"), and produces a notification object.

**Step 2 — Delivery:**
- **In-app real-time**: If the user is online (has an active WebSocket/SSE connection), push immediately through a Notification Gateway.
- **Mobile push**: Send to APNs (Apple) or FCM (Google) via a push notification service.
- **Fallback**: If the user is offline, store in a notifications inbox (database/cache). When they open the app, fetch unread notifications.

**Step 3 — Architecture:**
```
Event Producers → Kafka → Notification Service (rules, aggregation, dedup)
                              ↓                    ↓
                     Online? → Push via WS     Offline? → Store in inbox
                              ↓                           + Queue for mobile push
                     Notification Gateway         Push Service → APNs/FCM
```

**Step 4 — Key considerations:**
- **Aggregation**: Don't send 100 separate "X liked your photo" notifications. Batch into "X, Y, and 98 others liked your photo." Use a short delay window (30–60 seconds) before sending.
- **Priority**: Live video start from someone you follow > a like on an old post. Priority queues for time-sensitive notifications.
- **Rate limiting**: Don't bombard a user. Cap at N notifications per hour.
- **Deduplication**: Same event retried from Kafka shouldn't produce duplicate notifications. Use event ID as idempotency key.

**What the interviewer is listening for:**
- Async architecture (Kafka, not synchronous)
- Distinction between online (WebSocket push) and offline (inbox + mobile push) paths
- Aggregation and dedup logic
- You consider the user experience (not just the backend)

---

### Q3: "How would you handle 10 million users receiving a score update for a cricket/football match simultaneously?"

**How to Answer:**

This is a broadcast problem. You can't open 10M individual WebSocket connections on one server.

**Step 1 — Connection Layer:**
A fleet of WebSocket gateway servers, each handling ~50K connections. That's ~200 servers. Each server maintains a mapping of which users are connected to it. When a user connects, they "subscribe" to a match.

**Step 2 — Score Update Flow:**
Score update comes from the match data source → published to a Kafka topic (e.g., `match-updates:{matchId}`) → every gateway server is a consumer of this topic → each gateway pushes the update to all locally connected subscribers.

**Step 3 — Why this works:**
Kafka does the fan-out to servers. Each server does the fan-out to its local clients. Total: one Kafka message → 200 servers → 10M clients. The score update message is small (a few bytes), so the bandwidth is manageable.

**Step 4 — Edge optimization:**
For even larger scale, use a CDN with edge compute (Cloudflare Workers, AWS CloudFront Functions) to push updates from edge PoPs. Or use SSE instead of WebSockets — simpler, works with HTTP/2 multiplexing, and CDN-friendly.

**What the interviewer is listening for:**
- You recognize this is a fan-out/broadcast problem
- You don't try to send 10M messages from one server
- You use a pub/sub backbone (Kafka/Redis Pub/Sub) to fan out to gateway servers
- You mention CDN/edge for extreme scale
- You consider that not all 10M are watching the same match — partition by matchId

---

## 2. Fanout Pattern — Interview Questions

### Q1: "Design Twitter's home timeline. A user opens the app and sees tweets from people they follow."

**How to Answer:**

**Clarify:** How many users? (500M). Average follows? (200). Celebrity follows? (Some accounts have 50M+ followers). Read:write ratio? (Very read-heavy; most users read, few tweet).

**Step 1 — Naive approach and why it fails:**
Pull model: when a user opens their feed, query "SELECT tweets FROM tweets WHERE author_id IN (user's following list) ORDER BY time DESC LIMIT 50." With 200 followees, this is a scatter-gather across potentially 200 shards. Latency is too high for 500M users opening the app.

**Step 2 — Fanout-on-write (push model):**
When user A tweets, a fanout service writes tweet_id into every follower's timeline cache (a sorted list in Redis, keyed by userId). When a follower opens the app, their feed is pre-computed — just read from cache. Fast reads, O(1).

**Step 3 — The celebrity problem:**
User with 50M followers tweets → 50M writes. This takes minutes and is expensive. Solution: hybrid approach.
- Classify users as "normal" (< 10K followers) and "celebrity" (> 10K followers).
- Normal user tweets → fanout-on-write to all followers.
- Celebrity tweets → NOT fanned out. Stored in a separate "celebrity tweets" table.
- On read: fetch precomputed timeline (from fanout-on-write) + fetch latest tweets from celebrities the user follows + merge and rank.

**Step 4 — Architecture:**
```
Tweet Service → Kafka → Fanout Service (checks follower count)
                              ↓                       ↓
                    Normal: write to          Celebrity: write to
                    follower timelines       celebrity tweets store
                    (Redis sorted sets)

Read Path:
  User opens app → Timeline Service
    → Fetch from Redis (precomputed timeline)
    → Fetch celebrity tweets for this user's celeb follows
    → Merge, rank, return top 50
```

**Step 5 — Additional considerations:**
- **Delete/edit**: On delete, either fan out a delete (expensive) or lazy-filter at read time (check if tweet still exists before returning).
- **New follower**: If you follow someone, you don't see their past tweets in your precomputed timeline. Backfill the last N tweets on follow.
- **Inactive users**: Don't fanout to users who haven't opened the app in 30 days. On their next login, do a cold-start pull.

**What the interviewer is listening for:**
- You propose fanout-on-write and identify the celebrity problem without prompting
- You arrive at the hybrid solution
- You discuss the threshold and trade-offs
- You mention deletes, new follows, and inactive users

---

### Q2: "A notification needs to reach 1 million subscribers of a topic. How do you design this?"

**How to Answer:**

This is a pure fan-out problem. Unlike the timeline problem, there's no celebrity/normal split — every notification goes to all subscribers.

**Step 1 — Partitioned fanout workers:**
The subscriber list for a topic is stored in a database/cache. When a notification is triggered, a coordinator splits the subscriber list into chunks (e.g., 10K each) and enqueues a "send notification to chunk X" job for each chunk.

**Step 2 — Worker fleet:**
A pool of workers picks up chunk jobs from the queue and sends notifications (via WebSocket push, mobile push, or email). Each worker processes its chunk independently. Parallelism = number of workers.

**Step 3 — Delivery guarantees:**
Use an at-least-once delivery model. Each chunk job is retried if the worker crashes. Individual failed deliveries (e.g., APNs returns a device-not-registered error) are logged and the device token is invalidated.

**Step 4 — Scaling math:**
1M subscribers / 10K per chunk = 100 jobs. If each job takes 5 seconds to process, with 20 workers, all 100 jobs complete in ~25 seconds. Scale workers to meet latency SLA.

**What the interviewer is listening for:**
- You chunk the work for parallelism
- You use a durable queue for reliability
- You handle individual delivery failures gracefully
- You can do the back-of-envelope math

---

## 3. High Read Traffic — Interview Questions

### Q1: "Design the product detail page for Amazon. It gets 100M views per day."

**How to Answer:**

**Clarify:** What data is on the page? Product info (title, description, price, images), reviews, seller info, stock availability, recommendations. How often does the data change? Product info: rarely. Price: occasionally. Stock: frequently. Reviews: frequently.

**Step 1 — CDN for static assets:**
Images, CSS, JS served from CDN. Product images rarely change — long cache TTL (days). Use content hashing for cache busting on updates.

**Step 2 — Tiered caching for dynamic data:**
- **L1 — Local in-memory cache (on app servers)**: Product metadata that changes rarely. TTL: 60 seconds. Absorbs the hottest traffic.
- **L2 — Distributed cache (Redis)**: Product details, price, seller info. TTL: 5 minutes for most, 30 seconds for price. Cache-aside pattern.
- **L3 — Database (read replicas)**: Cache miss goes to a read replica. Only ~1% of requests should reach this layer.

**Step 3 — Stock availability (real-time, can't be stale):**
Stock is read from a dedicated inventory service backed by Redis (not a cache — it's the source of truth for available-to-promise count). This is updated on every purchase via atomic decrement.

**Step 4 — Reviews:**
Read from a separate reviews service. Cached aggressively (new reviews appearing a few minutes late is acceptable). Average rating is precomputed and cached.

**Step 5 — Recommendations:**
Fetched from the recommendation service asynchronously. If the rec service is slow/down, show "popular in this category" as a fallback (graceful degradation).

**Architecture:**
```
CDN → Load Balancer → App Server
                         ├→ L1 local cache → L2 Redis → L3 Read Replica (product data)
                         ├→ Inventory Service (stock, real-time)
                         ├→ Reviews Service (cached, eventual consistency OK)
                         └→ Recommendation Service (async, fallback to popular items)
```

**What the interviewer is listening for:**
- Multi-layer caching strategy with different TTLs for different data freshness needs
- You separate real-time data (stock) from cacheable data (product info, reviews)
- You use graceful degradation for non-critical services (recommendations)
- You mention CDN for static assets

---

### Q2: "Your cache goes down. What happens and how do you handle it?"

**How to Answer:**

This is a thundering herd / cache avalanche scenario.

**Immediate impact:** All traffic hits the database directly. If the cache was absorbing 95% of reads, the DB now gets 20x its normal load. It will likely be overwhelmed and either slow to a crawl or crash.

**Short-term mitigation:**
1. **Rate limiting / load shedding**: Protect the DB by rejecting excess requests with 503 + Retry-After.
2. **Circuit breaker**: If DB response times spike, trip the circuit breaker and serve stale data or a degraded response.
3. **Local cache on app servers**: Even a 10-second in-memory cache on each app server dramatically reduces DB load during cache recovery.

**Cache recovery:**
- Don't let all app servers hit the DB simultaneously to repopulate the cache (cache stampede). Use request coalescing: for each key, only one request fetches from DB; others wait for the result.
- Warm the cache gradually: use a cache warming script that pre-populates the top N most accessed keys from the DB before routing traffic.

**Prevention:**
- Run the cache as a cluster (Redis Cluster, 3+ nodes) so a single node failure doesn't take out the whole cache.
- Use Redis Sentinel or Cluster for automatic failover.
- Multi-AZ cache replicas.
- Monitor cache hit rate — a sudden drop is an early warning.

**What the interviewer is listening for:**
- You understand the cascading impact (cache miss → DB overload)
- You mention thundering herd and request coalescing
- You think about both immediate response and prevention
- You mention cache warming and cluster redundancy

---

## 4. High Write Traffic — Interview Questions

### Q1: "Design a system that ingests 1 million events per second from IoT devices."

**How to Answer:**

**Clarify:** What kind of events? (Sensor readings: device_id, timestamp, value, type). How long must data be retained? (Raw: 30 days. Aggregated: years). What are the query patterns? (Real-time dashboards + historical analysis).

**Step 1 — Ingestion layer:**
IoT devices send data to an API gateway (HTTP or MQTT for constrained devices). The gateway validates and forwards to Kafka. Why Kafka? It handles 1M+ messages/sec per cluster, provides durability via replication, and decouples ingestion from processing.

**Kafka sizing:** At 1M events/sec with ~200 bytes per event, that's ~200MB/sec. With replication factor 3, ~600MB/sec of disk I/O. A Kafka cluster of 10–15 brokers handles this comfortably. Partition by device_id for ordering per device.

**Step 2 — Stream processing:**
Kafka consumers (Flink / Spark Streaming) process the stream:
- **Real-time path**: Compute rolling aggregates (avg temperature per region per minute) and push to a real-time dashboard store (Redis or a time-series DB like InfluxDB/TimescaleDB).
- **Batch path**: Write raw events to a data lake (S3 in Parquet format) for historical analysis.

**Step 3 — Storage:**
- **Hot (last 24h)**: Time-series database (InfluxDB, TimescaleDB) for real-time dashboards and alerting.
- **Warm (last 30 days)**: Columnar store (ClickHouse) for interactive queries.
- **Cold (archival)**: S3 in Parquet, queryable via Athena/Presto.

**Step 4 — Backpressure handling:**
If downstream processing can't keep up, Kafka acts as a buffer (data is retained for the configured retention period). Consumers process at their own pace. Add more consumer instances to scale processing.

**Architecture:**
```
IoT Devices → API Gateway → Kafka (partitioned by device_id)
                                ├→ Flink (real-time aggregation) → InfluxDB → Dashboard
                                ├→ Flink (alerting rules) → Alert Service
                                └→ S3 Sink Connector → S3 (Parquet) → Athena
```

**What the interviewer is listening for:**
- Kafka as the central ingestion buffer
- Separation of real-time and batch processing paths (Lambda/Kappa architecture)
- Tiered storage with different latency/cost trade-offs
- Back-of-envelope sizing for Kafka

---

### Q2: "Design the write path for a social media platform where 50K posts are created per second."

**How to Answer:**

**Step 1 — API layer:**
POST /posts endpoint behind a load balancer. Stateless app servers accept the request, validate, and produce a message to Kafka.

**Step 2 — Async processing:**
The app server does NOT write to the database synchronously (that would bottleneck the DB). Instead:
- Write to Kafka (topic: `new-posts`). Return 202 Accepted to the client.
- The client gets an optimistic response ("your post is live") even though it's still being processed.

**Step 3 — Consumer pipeline:**
- **Post Storage Consumer**: Writes the post to the database (sharded by userId). Batch inserts for throughput.
- **Fanout Consumer**: Triggers the fanout-on-write pipeline (see Fanout section).
- **Search Indexer Consumer**: Indexes the post in Elasticsearch for search.
- **Media Processor Consumer**: If the post has images/video, triggers the media processing pipeline.

**Step 4 — Database choice:**
For the posts table, use a write-optimized store. Cassandra (LSM-tree) handles 50K writes/sec easily with horizontal scaling. Shard by userId. The trade-off: reads require knowing the userId (no efficient global secondary indexes). Pair with Elasticsearch for search queries.

**What the interviewer is listening for:**
- Async write path (Kafka between API and DB)
- Multiple consumers for different concerns (storage, fanout, search, media)
- Write-optimized database choice (Cassandra/DynamoDB) with justification
- Optimistic response to the user

---

## 5. Handling Hot Keys — Interview Questions

### Q1: "Elon Musk tweets and the tweet_id becomes a hot key. Your cache server for that key is at 100% CPU. What do you do?"

**How to Answer:**

**Immediate triage (runtime):**
1. **Replicate the hot key across multiple cache nodes**: Instead of one Redis node serving all reads for that key, replicate it to N nodes (e.g., 5). Route reads randomly across them. This gives 5x read throughput immediately.
2. **Local caching on app servers**: Each app server caches the tweet in local memory with a 1–2 second TTL. Millions of requests are served from local memory without ever hitting Redis.

**Medium-term (design-level):**
3. **Request coalescing**: If 10K requests arrive for the same key within a 10ms window, only one fetches from Redis; the rest wait and share the result.
4. **Preemptive hot key detection**: Monitor per-key access rates in real-time. When a key's rate exceeds a threshold, automatically promote it to "hot key mode" (local cache + replicated cache).

**Architecture for hot key handling:**
```
Request → App Server
              ├→ Check local cache (1s TTL) → HIT → return
              └→ MISS → Request coalescer
                           ├→ First request for this key → fetch from Redis (randomly pick from N replicas)
                           └→ Duplicate request → wait for first request's result
```

**What the interviewer is listening for:**
- You identify local caching as the most impactful immediate fix
- You mention request coalescing
- You think about both reactive (fix now) and proactive (detect early) approaches
- You quantify the improvement (local cache absorbs 99%+ of requests)

---

### Q2: "Your DynamoDB table has a hot partition. How do you fix it?"

**How to Answer:**

DynamoDB partitions data by the partition key. If one partition key gets disproportionate traffic, that partition is throttled even if the table has overall capacity.

**Solutions:**
1. **Write sharding**: Append a random suffix (0-9) to the partition key. Instead of `pk=celebrity_123`, write to `pk=celebrity_123#7`. Reads must query all 10 suffixed keys and merge. This spreads writes across 10 partitions.

2. **DynamoDB DAX (in-memory cache)**: Put DAX in front of DynamoDB. Hot reads are served from DAX's in-memory cache. Doesn't help with hot writes though.

3. **Redesign the partition key**: If the hot key is due to poor key design (e.g., partitioning by date — all today's writes go to one partition), redesign with a more granular key (e.g., date + userId hash).

4. **DynamoDB adaptive capacity**: DynamoDB automatically detects hot partitions and redistributes capacity. But it has limits — truly extreme hotspots still require sharding.

**What the interviewer is listening for:**
- You understand DynamoDB's partition key model
- You propose write sharding with concrete implementation
- You discuss the read fan-out trade-off
- You mention DAX for read-heavy hot keys

---

## 6. Handling Traffic Spikes — Interview Questions

### Q1: "Design a flash sale system where 1 million users try to buy 1,000 items at exactly 12:00 PM."

**How to Answer:**

**The core challenge:** 1M concurrent requests for 1K items. 999K users will be disappointed. The system must not oversell, crash, or be unfair.

**Step 1 — Pre-sale preparation:**
- Pre-scale everything: web servers, load balancers, cache.
- Put the product page on CDN. The dynamic "Buy Now" button can be controlled by a feature flag that enables at 12:00.
- Pre-load inventory count into Redis: `SET inventory:item123 1000`.

**Step 2 — The purchase flow:**
```
User clicks "Buy" → API Gateway (rate limiter: 1 request per user)
                         ↓
                    Purchase Service
                         ↓
                    Redis: DECR inventory:item123
                         ↓
                    Result ≥ 0? → Enqueue order to Kafka → Return "Order placed!"
                    Result < 0? → INCR inventory:item123 (rollback) → Return "Sold out"
```

Redis DECR is atomic and handles ~100K+ ops/sec. This ensures no overselling.

**Step 3 — Order processing (async):**
The actual order creation, payment, and fulfillment happen asynchronously via Kafka consumers. The user gets immediate feedback ("Order placed, processing...") and can check status later.

**Step 4 — Handling the 999K rejections:**
Most users get "Sold out" within milliseconds. The system is designed to say "no" very fast, which is the key to handling the spike.

**Step 5 — Fairness:**
- Use a virtual queue: Instead of letting everyone hit the purchase endpoint, put users in a queue at 12:00. Process the queue in order. Users see "You are #4521 in line." Only dequeue at the rate you can process.
- CAPTCHA or proof-of-work to prevent bots.

**What the interviewer is listening for:**
- Atomic inventory management (Redis DECR)
- Async order processing (don't do payment synchronously)
- Rate limiting per user
- "Saying no fast" as a design principle
- Virtual queue for fairness
- Pre-scaling

---

## 7. Handling Large Files — Interview Questions

### Q1: "Design Dropbox's file sync service."

**How to Answer:**

**Clarify:** Focus on: upload, storage, sync across devices, deduplication. How large are files? (Up to 50GB). How many users? (500M).

**Step 1 — Chunking:**
Split every file into fixed-size blocks (e.g., 4MB). Each block is identified by its SHA-256 hash (content-addressable). This enables:
- **Deduplication**: Two users uploading the same file → same block hashes → blocks already stored. No re-upload needed.
- **Delta sync**: When a file is edited, only changed blocks are re-uploaded (not the entire file).
- **Resumable upload**: If upload is interrupted, only re-upload the missing blocks.

**Step 2 — Upload flow:**
```
Client: Split file into blocks → Compute hash of each block
Client → Server: "I want to upload file X with blocks [hash1, hash2, hash3, ...]"
Server: Check which blocks already exist in storage
Server → Client: "I already have hash1 and hash3. Upload hash2 only."
Client → Object Storage (S3): Upload block hash2 via pre-signed URL
Client → Server: "Upload complete"
Server: Update file metadata (file → ordered list of block hashes)
```

**Step 3 — Sync across devices:**
When a file changes on device A:
- Device A uploads changed blocks and updates metadata.
- The metadata service emits a "file_changed" event.
- Device B has a long-polling or WebSocket connection to the notification service.
- Device B receives the notification, fetches new metadata, downloads only the changed blocks, and reconstructs the file locally.

**Step 4 — Storage:**
- **Block store**: S3 (or equivalent). Blocks are immutable (content-addressable). This simplifies caching and replication.
- **Metadata store**: A database mapping fileId → [ordered list of block hashes], plus versioning info (for conflict resolution and version history).

**Step 5 — Conflict resolution:**
If two devices edit the same file simultaneously offline, both upload different versions. The server keeps both versions and presents a conflict to the user ("conflicted copy"), similar to Dropbox's actual behavior.

**What the interviewer is listening for:**
- Content-addressable block storage (dedup + delta sync)
- Pre-signed URLs for direct-to-S3 upload
- Notification-based sync (not polling)
- Conflict resolution strategy
- Chunking enables resume, dedup, and delta sync simultaneously

---

## 8. Media Streaming — Interview Questions

### Q1: "Design YouTube."

**How to Answer:**

This is a massive system. Focus on the video pipeline (upload → process → serve) since that's the streaming-specific part.

**Upload Pipeline:**
1. User uploads via chunked upload (pre-signed URL to S3).
2. Upload completion triggers a message to the processing queue.

**Processing Pipeline (async):**
3. **Transcoding**: The raw video is transcoded into multiple bitrate/resolution renditions (240p through 4K). Use a distributed transcoding fleet. Split the video into segments, transcode in parallel across workers, stitch results.
4. **Thumbnail generation**: Extract frames at intervals, allow user to choose or auto-select.
5. **Content moderation**: ML models scan for policy violations.
6. **Metadata extraction**: Duration, codec info, aspect ratio.

**Storage:**
7. Transcoded segments stored in S3. Manifest files (.m3u8 for HLS) generated for each rendition.

**Serving:**
8. Video served via CDN. The client's player loads the manifest, then fetches segments. ABR algorithm switches quality based on bandwidth.

**Scaling considerations:**
- **Transcoding**: Most expensive part. Use spot instances for cost savings. Queue-based, so naturally handles backpressure.
- **CDN**: Handles most of the serving. Popular videos are cached at edge nodes. Long-tail videos may need origin fetches.
- **Upload spike**: A creator with 100M subscribers uploads → millions of viewers try to watch immediately. Pre-warm CDN for popular creators' content (predictive caching).

**What the interviewer is listening for:**
- Async processing pipeline with queue
- Multi-bitrate transcoding + ABR
- CDN-first serving architecture
- Parallel transcoding for speed
- Content moderation in the pipeline

---

### Q2: "How would you reduce latency for a live streaming platform like Twitch to under 5 seconds?"

**How to Answer:**

Standard HLS has 15–30 second latency because it buffers several segments (each 6–10 seconds) before playback starts.

**Reducing latency:**
1. **Shorter segments**: Use 2-second segments instead of 6-second. The player needs fewer segments buffered before starting. Trade-off: more HTTP requests, more CDN load.
2. **Low-Latency HLS (LL-HLS)**: Uses "partial segments" (sub-second chunks) that can be pushed to the client before the full segment is complete. The server sends the manifest with "preload hints" so the player knows to request the next partial before it's fully ready.
3. **CMAF with Chunked Transfer Encoding**: Send segments as chunked HTTP responses. The player can start decoding before the full segment arrives.
4. **CDN tuning**: Reduce cache TTL for live segments. Use origin shield to prevent thundering herd at origin.
5. **Ingest optimization**: Use SRT (Secure Reliable Transport) instead of RTMP for the broadcaster-to-server link — better performance over lossy networks.

**For sub-second latency (< 1 second):**
You need to move away from segment-based protocols entirely and use WebRTC. Trade-off: WebRTC doesn't scale as well for one-to-many (it's designed for peer-to-peer), so you need a Selective Forwarding Unit (SFU) architecture where the server receives the WebRTC stream and forwards it to viewers.

**What the interviewer is listening for:**
- You understand why HLS has high latency (segment buffering)
- You know about LL-HLS and CMAF
- You mention the trade-off between latency and scalability
- For sub-second, you pivot to WebRTC + SFU

---

## 9. Handling Location Data — Interview Questions

### Q1: "Design Uber's ride matching system."

**How to Answer:**

**Core problem:** When a rider requests a ride, find the nearest available drivers and match one to the rider.

**Step 1 — Driver location tracking:**
Drivers send GPS coordinates every 3–5 seconds. These are published to Kafka (partitioned by city/region). Consumers update an in-memory geospatial index.

**Step 2 — Geospatial index:**
Use Redis GEOADD to store driver locations: `GEOADD drivers:bangalore lng lat driver_id`. When a ride is requested, use `GEOSEARCH drivers:bangalore FROMLONLAT lng lat BYRADIUS 5 km COUNT 20 ASC` to find the 20 nearest drivers.

**Step 3 — Matching logic:**
For each nearby driver candidate:
- Check availability (not already on a ride).
- Compute ETA (via a routing service like OSRM or Google Maps API).
- Rank by ETA (not just straight-line distance — a driver 2km away across a river is worse than one 3km away on the same road).
- Send ride offer to the top-ranked driver. If they decline/timeout, move to the next.

**Step 4 — Sharding by geography:**
The world is divided into regions (cities or H3 cells at a coarse resolution). Each region has its own Redis instance holding driver locations. A rider in Bangalore only queries the Bangalore shard. This ensures scalability and locality.

**Step 5 — Supply-demand balancing:**
If demand exceeds supply in a region (surge pricing scenario), the system tracks request density and driver density per H3 cell to compute a surge multiplier.

**Architecture:**
```
Driver App → Kafka (location updates) → Location Consumer → Redis GEO (per city)
Rider App → Ride Service → Query Redis GEO (nearest drivers)
                         → ETA Service (routing)
                         → Matching Engine (rank + offer)
                         → Driver App (push ride offer via WebSocket)
```

**What the interviewer is listening for:**
- Geospatial indexing (Redis GEO or similar)
- ETA-based ranking, not just distance
- Geographic sharding
- Real-time location updates via streaming
- You address driver state (available, busy, offline)

---

## 10. Generating Unique IDs — Interview Questions

### Q1: "Design a distributed ID generation service for a microservices architecture."

**How to Answer:**

**Requirements:** Globally unique, roughly time-sortable (for database index efficiency), low latency (< 1ms), high throughput (100K+ IDs/sec), no single point of failure.

**Approach: Snowflake-style ID**

```
[1 bit unused][41 bits timestamp (ms)][10 bits machine ID][12 bits sequence]
= 64-bit integer
```

**How it works:**
Each ID generation server has a unique machine ID (0–1023). When it receives a request:
1. Get current timestamp in milliseconds.
2. If same millisecond as last ID, increment the 12-bit sequence counter (0–4095).
3. If sequence overflows (4096 IDs in 1ms), wait until the next millisecond.
4. Combine: `timestamp << 22 | machineId << 12 | sequence`.

**Machine ID assignment:**
Use ZooKeeper or etcd. Each server, on startup, acquires a unique machine ID (ephemeral sequential node in ZK). If the server dies, the ID is released. Alternatively, derive from the IP address or k8s pod metadata.

**Deployment:**
Run as a library embedded in each service (no network call needed, lowest latency) or as a dedicated microservice (easier to manage machine IDs, slight network overhead).

**Failure handling:**
- **Clock goes backward (NTP adjustment)**: Detect it. Refuse to generate IDs until the clock catches up. Log an alert. This is rare but critical.
- **Machine ID collision**: If two servers get the same machine ID (e.g., ZK bug), IDs could collide. Use a startup check: generate a few IDs and verify uniqueness against a recent sample.
- **All sequences exhausted in one millisecond**: Wait. In practice, 4096 IDs per ms per server is rarely a bottleneck.

**Why not UUID v4?** 128 bits (wastes space and index efficiency), not sortable (random inserts fragment B-tree indexes), no embedded timestamp (can't extract creation time).

**Why not auto-increment from a database?** Single point of failure, limited throughput, doesn't work across shards without coordination.

**What the interviewer is listening for:**
- You can explain the bit layout of Snowflake IDs
- You address machine ID assignment (ZK/etcd)
- You handle the clock skew problem
- You compare with UUID and auto-increment and justify the choice

---

## 11. Distributed Counting — Interview Questions

### Q1: "Design a view counter for YouTube videos. Some videos get millions of views per second."

**How to Answer:**

**Step 1 — Why naive approach fails:**
A single Redis key per video: `INCR views:videoId`. Redis INCR handles ~100K ops/sec per key. A viral video getting 5M views/sec overwhelms a single key.

**Step 2 — Local aggregation:**
Each app server maintains an in-memory counter per video. Every second (or every 1000 increments), flush the local count to Redis in a batch: `INCRBY views:videoId 1247`. If you have 500 app servers, this reduces Redis traffic from 5M/sec to 500/sec per key.

**Step 3 — Sharded counters for the hottest videos:**
For extremely viral videos, even 500 INCRBY/sec might be a lot on one key. Shard the counter: `views:videoId:0` through `views:videoId:9`. Each flush goes to a random shard. To read, SUM all 10 shards. This is only needed for the top 0.01% of videos.

**Step 4 — Display count:**
The displayed count doesn't need to be real-time. Cache the SUM for 30 seconds. "1.2M views" is fine even if the true number is 1,247,832 — nobody notices the difference.

**Step 5 — Durable counting (offline reconciliation):**
The Redis counters are the real-time approximate path. For accurate billing/analytics, log every view event to Kafka → batch process hourly into a data warehouse → reconcile with the Redis count.

**Architecture:**
```
View Event → App Server (local counter++)
                  ↓ (every 1s)
              Redis INCRBY (sharded for hot videos)
                  ↓ (cached, 30s TTL)
              Display Count API

View Event → Kafka → Data Warehouse (hourly batch for exact count)
```

**What the interviewer is listening for:**
- Local aggregation as the primary throughput optimization
- Sharded counters for extreme hot keys
- Approximate display is acceptable (eventual consistency)
- Exact counting happens offline in a batch pipeline

---

## 12. Leader Election — Interview Questions

### Q1: "You have a scheduled job that should run on exactly one server. How do you ensure only one instance runs?"

**How to Answer:**

**Option 1 — Database lock:**
```sql
INSERT INTO cron_locks (job_name, node_id, expires_at)
VALUES ('daily_report', 'node-3', NOW() + INTERVAL '5 minutes')
ON CONFLICT (job_name) DO NOTHING;
```
If the insert succeeds, this node is the leader. It must renew the lock before expiry. If it crashes, the lock expires, and another node can claim it.

Pros: Simple, no new infrastructure. Cons: Database becomes a dependency for leader election. Clock skew between DB and app servers can cause issues.

**Option 2 — etcd/ZooKeeper lease:**
Acquire a lease in etcd with a TTL. The lease holder is the leader. etcd handles distributed consensus (Raft), so this is correct even under network partitions. The leader sends keepalives to refresh the lease.

Pros: Purpose-built for this, correct under partitions. Cons: Requires etcd infrastructure.

**Option 3 — Redis RedLock:**
`SET cron:daily_report node-3 NX EX 30`. If it succeeds, this node is the leader. For higher correctness, use RedLock across multiple Redis instances.

Pros: Fast, simple. Cons: Redis is not a consensus system — RedLock has known correctness issues under clock drift and network partitions (see Martin Kleppmann's analysis).

**Which to recommend?**
- For correctness-critical systems (financial, exactly-once processing): etcd or ZooKeeper.
- For best-effort (it's OK if the job occasionally runs twice): database lock or Redis.

**What the interviewer is listening for:**
- You present multiple options with trade-offs
- You mention the split-brain risk and fencing tokens
- You differentiate between correctness-critical and best-effort scenarios
- You mention lease renewal and what happens on crash

---

## 13. Failure Detection — Interview Questions

### Q1: "How does a distributed database like Cassandra know when a node is down?"

**How to Answer:**

Cassandra uses a **gossip protocol** combined with a **phi accrual failure detector**.

**Gossip:**
Every second, each node randomly selects 1–3 peers and exchanges a "gossip digest" — a compact summary of the health state of all known nodes (including heartbeat generation numbers and timestamps). Through this random peer-to-peer exchange, information propagates to all nodes exponentially fast (like an epidemic).

**Phi Accrual Failure Detector:**
Instead of a fixed timeout ("no heartbeat in 5 seconds = dead"), the phi detector maintains a statistical model of inter-heartbeat arrival times for each peer. It computes a "phi" value representing the confidence that the node has failed. A phi of 1 means there's a 10% chance the node is alive; phi of 8 means there's a 0.00000001% chance. The operator configures a threshold (typically phi > 8).

**Why this is better than a fixed timeout:**
- In a cross-datacenter deployment, heartbeat intervals are naturally longer and more variable. A fixed timeout that works for intra-DC would be too aggressive for cross-DC (many false positives). Phi accrual adapts automatically.
- During a GC pause, heartbeats are delayed. Phi accrual considers recent variability instead of a hardcoded threshold.

**What happens after detection:**
When a node is suspected dead (phi > threshold), Cassandra marks it as DOWN. Reads and writes are routed to other replicas. Hinted handoff stores writes destined for the down node. When it comes back, repair processes reconcile data.

**What the interviewer is listening for:**
- You know gossip protocols and can explain convergence
- You understand phi accrual and why it's better than fixed timeouts
- You explain what happens after detection (hinted handoff, repair)

---

## 14. Handling Failures — Interview Questions

### Q1: "A payment service your app depends on starts returning errors for 30% of requests. How do you handle this?"

**How to Answer:**

**Layer 1 — Retries with backoff:**
For the 30% that fail, retry once with a 500ms delay. Many transient errors (network blip, temporary overload) resolve on retry. Use an idempotency key to prevent double-charging. Cap retries at 2–3 to avoid amplifying load.

**Layer 2 — Circuit breaker:**
If the error rate stays at 30%, the circuit breaker trips to OPEN. For the next 30 seconds, all calls immediately return a fallback response WITHOUT calling the payment service. This gives the payment service time to recover.

After 30 seconds, the circuit moves to HALF-OPEN: allow 10% of requests through as probes. If probes succeed, close the circuit (resume normal calls). If probes fail, re-open.

**Layer 3 — Graceful degradation:**
While the circuit is open, what does the user experience?
- Option A: "Payment is temporarily unavailable. Your order has been saved and we'll process payment shortly." Queue the order and process when the payment service recovers.
- Option B: For lower-value orders, approve optimistically and charge later (accept business risk).
- Option C: Show the error and let the user retry later (worst UX but simplest).

**Layer 4 — Alerting:**
The circuit breaker opening triggers a PagerDuty alert. The on-call engineer investigates the payment service.

**What the interviewer is listening for:**
- Layered approach: retries → circuit breaker → degradation
- Idempotency key for safe retries on payments
- You think about the user experience during degradation
- Alerting and observability

---

### Q2: "Walk me through how you'd design a system to survive the failure of an entire availability zone."

**How to Answer:**

**Step 1 — Stateless services across AZs:**
Run at least 3 instances of every stateless service, distributed across 3 AZs. The load balancer health-checks each instance and routes away from unhealthy ones. If an entire AZ goes down, 2/3 of instances remain.

**Step 2 — Databases:**
Run the primary in AZ-1 with synchronous replicas in AZ-2 and AZ-3. On AZ-1 failure, promote the AZ-2 replica. RDS Multi-AZ handles this automatically. For Cassandra/DynamoDB, replication factor 3 with rack-awareness (rack = AZ) ensures data exists in all 3 AZs.

**Step 3 — Caches:**
Redis Cluster with replicas in different AZs. If an AZ dies, Redis Sentinel promotes replicas in other AZs. Accept a brief cache miss period during failover.

**Step 4 — Message queues:**
Kafka with brokers across 3 AZs, replication factor 3, min.insync.replicas 2. An AZ failure loses 1/3 of brokers but all data is safe on the remaining 2 AZs. Partitions are re-balanced to surviving brokers.

**Step 5 — Capacity planning:**
Each AZ must be provisioned to handle 50% of total traffic (not 33%), because when one AZ fails, the remaining two must absorb 100% between them. This is the "N+1 AZ" principle.

**What the interviewer is listening for:**
- You address every layer (compute, storage, cache, messaging)
- You mention capacity planning (the 50% rule)
- You know that stateless services are easy; stateful services need careful replication
- You mention automated failover (not manual)

---

## 15. Recommendations — Interview Questions

### Q1: "Design a recommendation engine for an e-commerce platform."

**How to Answer:**

**Step 1 — Data signals:**
- Explicit: purchases, ratings, wishlist additions.
- Implicit: views, time spent on product page, search queries, add-to-cart (even without purchase).
- Context: time of day, device, season, user location.

**Step 2 — Offline pipeline (train models):**
- **Collaborative filtering (item-based)**: Compute item-to-item similarity based on co-purchase / co-view patterns. "Users who bought X also bought Y." Store the top-K similar items per item.
- **Embedding model**: Train a two-tower neural network. User tower: encodes user history into a vector. Item tower: encodes item features into a vector. Train so that the dot product of user and item vectors predicts interaction. Store item embeddings in a vector database (Pinecone, Milvus, pgvector).

**Step 3 — Online serving (two-stage):**
- **Candidate generation**: Given the user, retrieve ~1000 candidates. Sources: (a) ANN search on item embeddings using the user's embedding as query. (b) Items similar to the user's recent views/purchases (from item-item similarity table). (c) Popular items in the user's browsing category. (d) Trending items.
- **Ranking**: Score each of the ~1000 candidates with a ranking model (gradient-boosted trees or a neural ranker) using rich features: user demographics, item metadata, user-item cross features, contextual features. Return top 20.

**Step 4 — Cold start:**
- New user: Show popularity-based recommendations. After a few interactions, start personalizing. Use onboarding quiz ("What categories interest you?").
- New item: Use content-based features (title, description, category embedding) to find similar items and bootstrap.

**Step 5 — Evaluation:**
- Offline: Precision@K, Recall@K, NDCG on held-out data.
- Online: A/B test measuring CTR, add-to-cart rate, purchase conversion, revenue per user.
- Monitor for filter bubbles: track recommendation diversity and novelty.

**What the interviewer is listening for:**
- Two-stage architecture (candidate gen + ranking)
- Multiple signal types (explicit + implicit)
- Cold start handling
- Evaluation strategy (offline metrics + online A/B test)

---

## 16. Multi-Tenancy — Interview Questions

### Q1: "Design a SaaS analytics platform that serves thousands of companies, each with different data volumes."

**How to Answer:**

**Step 1 — Tenant isolation model:**
- Small tenants (< 1GB data, low query volume): Shared database, shared schema, `tenant_id` column on every table. Cost-efficient.
- Large tenants (> 100GB, high query volume): Dedicated database instance. Prevents noisy neighbor issues and meets enterprise compliance requirements.
- Medium tenants: Shared database, separate schema per tenant.

A "tenant metadata" service stores each tenant's configuration: which isolation model, which database host, resource limits, feature flags.

**Step 2 — Routing:**
Every API request includes a tenant identifier (from JWT, subdomain, or API key). The API gateway extracts it and enriches the request with the tenant's configuration from the metadata service (cached). The backend uses this to route to the correct database.

**Step 3 — Noisy neighbor prevention:**
- Per-tenant rate limiting at the API gateway.
- Per-tenant query concurrency limits at the database layer.
- Per-tenant resource quotas (CPU, memory) using container resource limits (k8s resource quotas per namespace, or cgroups).
- Long-running queries are killed after a timeout.

**Step 4 — Data isolation and security:**
- Every database query includes `WHERE tenant_id = ?` enforced at the ORM/middleware level (not relying on developers to remember).
- Cache keys are prefixed with tenant_id: `cache:tenant_123:dashboard:456`.
- Encryption keys per tenant (envelope encryption: each tenant has a data encryption key, wrapped by a master key).

**Step 5 — Scaling:**
- Horizontal scaling of shared infrastructure for small tenants.
- Vertical or dedicated scaling for large tenants.
- Tenant migration tool: move a growing tenant from shared to dedicated without downtime (dual-write, then switch, then backfill).

**What the interviewer is listening for:**
- Tiered isolation based on tenant size/needs
- Tenant-aware routing
- Noisy neighbor prevention (rate limits, quotas, timeouts)
- Data leak prevention (tenant_id enforcement)
- Tenant migration strategy

---

## 17. Multi-Region Architecture — Interview Questions

### Q1: "Design a globally distributed messaging app like WhatsApp."

**How to Answer:**

**Step 1 — User home region:**
Each user is assigned a "home region" based on their phone number's country code or signup location. The user's data (messages, contacts, profile) lives in the home region. This avoids write conflicts — each user's data has a single writer.

**Step 2 — Message routing:**
When user A (home: US-East) sends a message to user B (home: EU-West):
1. A's client sends the message to US-East (A's home region).
2. US-East writes the message to A's outbox.
3. US-East looks up B's home region (EU-West) from a global user registry (cached).
4. US-East forwards the message to EU-West.
5. EU-West writes to B's inbox.
6. If B is online and connected to EU-West, push via WebSocket immediately. Otherwise, store for later delivery + send mobile push notification.

**Step 3 — Global user registry:**
A lightweight table mapping userId → home region. Replicated across all regions (read-heavy, rarely changes). Use DynamoDB Global Tables or a similar globally replicated store.

**Step 4 — Handling region failure:**
If US-East goes down:
- Users homed to US-East can't send/receive messages temporarily.
- Users in other regions are unaffected (they can still message each other).
- For higher availability: replicate each user's data to a secondary region. On failure, promote the secondary. Accept that some recent messages might be lost (RPO = replication lag).

**Step 5 — End-to-end encryption:**
Messages are encrypted client-side. The server stores ciphertext only. This simplifies multi-region because the server doesn't need to read/process message content — just route encrypted blobs.

**What the interviewer is listening for:**
- Home region concept to avoid write conflicts
- Cross-region message routing
- Global user registry with low-latency lookups
- Region failure handling and RPO trade-offs
- You mention E2E encryption's impact on architecture

---

## 18. Deduplicating Data — Interview Questions

### Q1: "How do you ensure exactly-once processing in a Kafka-based event pipeline?"

**How to Answer:**

**The problem:** Kafka guarantees at-least-once delivery. After a consumer crash and rebalance, some events are redelivered. If the consumer processed an event but didn't commit the offset before crashing, it processes the event again on restart.

**Approach 1 — Idempotent consumers:**
Design consumers so that processing an event twice has the same effect as processing it once. For example, if the event is "set user balance to $100," processing it twice is fine. If the event is "add $10 to user balance," processing it twice gives $20 instead of $10 — not idempotent.

For non-naturally-idempotent operations, use a deduplication table:
```
1. Read event (event_id: abc-123).
2. Check: SELECT 1 FROM processed_events WHERE event_id = 'abc-123'.
3. If exists → skip (already processed).
4. If not exists → process + INSERT INTO processed_events (event_id) VALUES ('abc-123') in the SAME transaction as the business logic.
```

The transaction ensures that if the business logic succeeds, the event is marked as processed atomically. If the consumer crashes before the transaction commits, both the business logic and the dedup marker are rolled back, and the event will be reprocessed correctly on retry.

**Approach 2 — Kafka Exactly-Once Semantics (EOS):**
Kafka supports transactional writes: the consumer reads from an input topic, processes, writes to an output topic, and commits the consumer offset — all atomically. If any step fails, everything is rolled back. This provides effectively-once processing within the Kafka ecosystem.

**Approach 3 — Outbox + CDC:**
For writes to external databases, use the outbox pattern. The consumer writes business data + outbox event in one DB transaction. A CDC connector (Debezium) reads the outbox table and publishes to the next Kafka topic. This chains exactly-once guarantees across service boundaries.

**What the interviewer is listening for:**
- You explain why at-least-once leads to duplicates
- You mention idempotent consumers as the primary strategy
- You describe the dedup table + transaction pattern
- You know Kafka EOS exists and when to use it

---

## 19. Distributed Transactions — Interview Questions

### Q1: "Design the checkout flow for an e-commerce platform that spans Order, Payment, Inventory, and Shipping services."

**How to Answer:**

**Why not a single database transaction?** Each service has its own database. You can't do a cross-database ACID transaction in a microservices architecture (no shared transaction coordinator).

**Approach: Saga with Orchestration**

An Order Orchestrator drives the workflow:

```
Step 1: Order Service → Create order (PENDING)
Step 2: Inventory Service → Reserve items
Step 3: Payment Service → Charge customer
Step 4: Shipping Service → Schedule delivery
Step 5: Order Service → Update order (CONFIRMED)
```

**Compensating actions (if a step fails):**
```
If Step 4 fails (shipping unavailable):
  → Compensate Step 3: Refund payment
  → Compensate Step 2: Release inventory reservation
  → Compensate Step 1: Cancel order (CANCELLED)

If Step 3 fails (payment declined):
  → Compensate Step 2: Release inventory reservation
  → Compensate Step 1: Cancel order (PAYMENT_FAILED)
```

**Orchestrator implementation:**
The orchestrator is a state machine:
```
PENDING → INVENTORY_RESERVED → PAYMENT_CHARGED → SHIPPING_SCHEDULED → CONFIRMED
    ↓              ↓                    ↓                   ↓
 FAILED     COMPENSATING_PAYMENT  COMPENSATING_INV   COMPENSATING_ALL
```

Store the saga state in a database. On crash, the orchestrator recovers by reading its last state and resuming.

**Reliability — Outbox Pattern:**
Each service uses the outbox pattern to atomically update its database AND emit an event. The orchestrator listens for these events.

```
Inventory Service transaction:
  UPDATE inventory SET reserved = reserved + 1 WHERE item_id = X;
  INSERT INTO outbox (event_type, payload) VALUES ('items_reserved', '{...}');
COMMIT;
```

A CDC process (Debezium) reads the outbox and publishes to Kafka. The orchestrator consumes from Kafka.

**Key design decisions:**
- **Payment last (or near-last)**: Minimize the time money is charged without fulfillment. Some teams put payment before shipping; others put it after. If payment is before shipping and shipping fails, you must refund (annoying for the user).
- **Compensations must be idempotent**: The "refund payment" compensation might be retried. Use the original transaction ID as the refund reference.
- **Timeout handling**: If a service doesn't respond within 30 seconds, assume failure and compensate. The service might still process it later — but the compensation (refund, release) will reconcile.

**What the interviewer is listening for:**
- You choose Saga (not 2PC) for microservices and explain why
- You define compensating actions for each step
- You draw the state machine
- You use the outbox pattern for reliability
- You address failure scenarios: what if compensation fails? (Retry + dead-letter + manual resolution)

---

## 20. Removing Single Points of Failure — Interview Questions

### Q1: "Here's an architecture diagram. Identify all single points of failure and fix them."

**How to Answer (General Framework):**

Walk through every component systematically:

**1. DNS:**
"Is there a single DNS provider? If Route53 goes down (it happened in 2016), the entire system is unreachable. → Add a secondary DNS provider (Cloudflare) with health checks."

**2. Load Balancer:**
"Is there a single LB? → Use a cloud-managed LB (AWS ALB/NLB — inherently redundant across AZs) or a pair of HAProxy instances with VRRP failover."

**3. Application Servers:**
"Are there at least 2 instances per service? Are they in multiple AZs? → Run N+1 instances across AZs. Ensure auto-scaling is configured."

**4. Database:**
"Is there a single database? → Primary-replica with automatic failover (RDS Multi-AZ). For critical data, cross-region replica."

**5. Cache:**
"Is there a single Redis node? → Redis Sentinel (automatic failover with replicas) or Redis Cluster (sharding + replication)."

**6. Message Queue:**
"Is there a single Kafka broker? → Kafka cluster with 3+ brokers, replication factor 3, min.insync.replicas 2."

**7. External dependencies:**
"Is the system dependent on a single third-party API (e.g., payment processor)? → Integrate a backup processor. Use the circuit breaker pattern: if the primary fails, route to the backup."

**8. Configuration / Service Discovery:**
"Is there a single config server (etcd, Consul)? → Run as a 3-node or 5-node cluster for quorum-based consensus."

**9. Deployment / CI/CD:**
"Can you deploy if the CI/CD system is down? → Have a manual deploy fallback (script that can push from a developer's machine)."

**10. Monitoring / Alerting:**
"If your monitoring system goes down, will you know other things are failing? → Use a separate monitoring stack or a third-party service (Datadog, PagerDuty) that's independent of your infrastructure."

**What the interviewer is listening for:**
- Systematic approach (you don't miss components)
- You know the right redundancy mechanism for each component type
- You mention non-obvious SPOFs (DNS, monitoring, CI/CD)
- You consider correlated failures (same AZ, same rack)
- You mention chaos engineering as validation

---

### Q2: "How would you convince your engineering manager to invest in removing SPOFs when 'it's been working fine'?"

**How to Answer:**

This is a soft-skills + technical judgment question.

**Frame it in business terms:**
- Calculate the cost of downtime: revenue per hour × expected hours of downtime per incident × probability of incident per year.
- A single-region, single-database architecture has a historical MTBF (mean time between failures) of X months. AWS itself publishes availability data.

**Use concrete scenarios:**
- "Our database has no replica. If the disk fails at 2 AM on a Saturday, our RTO (recovery time objective) is however long it takes to restore from backup — likely 2–4 hours. During that time, the product is completely down."
- "Our Redis instance is a single node. It handles all session data. If it dies, every user is logged out simultaneously."

**Propose incremental improvements:**
Don't ask for a full multi-region active-active rewrite. Start with the highest-impact, lowest-effort wins:
1. Add a database read replica with automatic failover (~2 hours of work, prevents the scariest scenario).
2. Move to Redis Sentinel (3 nodes) — modest cost increase, eliminates the cache SPOF.
3. Run at least 2 instances of every service in different AZs.

**What the interviewer is listening for:**
- Business-level reasoning (cost of downtime, not just technical purity)
- Pragmatic, incremental approach
- Specific, concrete risks (not abstract "we should be more resilient")

---

## Appendix B: Universal Answer Framework for ANY HLD Question

When faced with a system design question, follow this framework:

### Step 1 — Clarify Requirements (2 minutes)
- Functional requirements: What does the system DO?
- Non-functional requirements: Scale (users, data volume, QPS), latency, availability, consistency.
- Constraints: Budget, timeline, existing tech stack.

### Step 2 — Back-of-Envelope Estimation (2 minutes)
- DAU, peak QPS, storage per year, bandwidth.
- This determines whether you need sharding, caching, CDN, etc.

### Step 3 — High-Level Design (10 minutes)
- Draw the major components (clients, API gateway, services, databases, caches, queues).
- Show data flow for the primary use case.
- Identify which "building blocks" from this guide apply.

### Step 4 — Deep Dive (15 minutes)
- The interviewer will pick 1–2 components to dive deep on.
- Use the detailed patterns from this guide: caching strategies, sharding, fan-out model, etc.
- Discuss trade-offs at every decision point.

### Step 5 — Address Failure Modes (5 minutes)
- What happens when X goes down?
- How do you detect it? How do you recover?
- What data is lost? What's the blast radius?

### Step 6 — Scaling and Future Evolution (3 minutes)
- How would this change at 10x scale?
- What would you change if the requirements evolved (e.g., add multi-region)?

---

*Good luck with your HLD interviews. Remember: there's never one right answer — the interviewer wants to see you reason about trade-offs, make decisions, and handle failure scenarios gracefully.*
