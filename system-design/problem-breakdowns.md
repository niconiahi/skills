# Worked problem breakdowns

Twelve complete answer keys from hellointerview.com, each showing the delivery framework end-to-end with deep dives graded bad → good → great.

Use the one closest to the system at hand.

The grading language itself ("a good answer does X, a great answer does Y because Z") is the expert register to write in.

## Ticketmaster (booking + contention)

Requirements: view, search and book events; consistency for booking, availability for search; 10M users on one event; search < 500ms; 100:1 reads.

Design: Event, Search and Booking services behind a gateway; Postgres; Elasticsearch (CDC-synced); Stripe webhooks.

```
booking: seat → Redis lock (10-min TTL) → "in-progress" row → pay → webhook → transaction marks sold
```

For no double booking, bad is `SELECT FOR UPDATE` held through checkout, good is a status field plus an expiry cron, and great is implicit status (the transaction validates available-or-expired and stamps reservation plus timestamp) or the recommended Redis TTL lock with DB optimistic concurrency as the backstop if Redis dies.

For extreme demand, great is a virtual waiting queue: a Redis sorted set gates entry to the booking page, SSE reports position, admitted sessions sit in a TTL'd set.

For search < 500ms, good is DB indexes and great is Postgres tsvector+GIN or the recommended Elasticsearch with fuzzy matching, CDC-synced, cached via the ES shard-level query cache plus a CDN.

## Uber (geospatial + matching)

Requirements: fare estimate, request ride, match a nearby driver, accept and navigate; match < 1 min; no driver double-assigned (strong consistency); 100k requests from one location.

Scale: 10M drivers × 5s pings ≈ 2M location updates/sec.

Design: Ride, Location, Matching and Notification services, with driverId from the JWT, never the body.

For 2M updates/sec, bad is direct DB writes (~$200k/day on DynamoDB) and great is Redis GEOADD/GEOSEARCH in memory with TTL-expiry of stale drivers, so a loss self-heals in one 5s ping cycle, plus adaptive client update intervals.

For one driver, one request, great is a Redis lock keyed by driverId with TTL equal to the 10s acceptance window.

For surge absorption, great is Kafka/SQS in front of matching, the offset committed only after a successful match, partitioned geographically, consumers auto-scaled on depth.

For driver non-response, great is durable execution (Temporal/Step Functions) modeling offer → timeout → next driver.

Further scale is geo-sharding plus consistent hashing, with scatter-gather only at region boundaries.

## Dropbox (large files)

Requirements: upload, download, share and sync; availability; files to 50GB; secure; low latency.

Design: the client chunks (5-10MB) and uploads directly to S3 via presigned URLs; a File Service issues the URLs; DynamoDB holds metadata (a chunks array, SharedFiles for ACL); an S3 event flips status to "uploaded"; the CDN serves signed URLs.

For large files, great is "trust but verify": per-chunk presigned URLs, the client PATCHes ETags, the backend verifies via S3 ListParts.

Resumability comes from SHA-256 fingerprints of file and chunks so a retry skips uploaded chunks; downloads use HTTP Range with no chunking needed.

For speed: parallel chunk uploads, content-defined chunking (Rabin) giving delta sync on small edits, and compression only when the savings beat the CPU (text yes, media no).

## FB News Feed (fan-out)

Requirements: post, follow, reverse-chron feed; availability with ≤1 min staleness; < 500ms; 2B users, unlimited followers.

Design: DynamoDB (Follows: PK following, SK followed, reverse GSI), PrecomputedFeed (~200 post IDs ≈ 2KB/user), SQS fan-out workers.

Feed assembly moves from read-time to write-time by precomputing, falling back to base tables past the 200-post window.

For celebrities, great is hybrid fan-out, a per-account choice where celebrity posts are skipped by workers and merged at read time; the 1-minute staleness budget is what buys the async architecture.

For viral hot keys, great is a replicated (not sharded) post cache: N independent Redis instances, the LB spreading reads, hot traffic ÷ N, "no coordination required." Replication beats sharding for hot reads.

## YouTube (media pipeline)

Requirements: upload and stream; availability; videos of tens of GB; low-latency adaptive streaming; 1M uploads/day; resumable uploads.

Design: presigned multipart to S3 → S3 event → a DAG pipeline under an orchestrator (Temporal): split (ffmpeg) → parallel per-segment transcodes plus audio and transcripts → manifests; metadata in Cassandra by videoId; watch fetches the manifest from the CDN, the client picks a bitrate per network and streams segments.

The deep dives: seconds-long segments × formats are what enable adaptive bitrate; chunk fingerprints plus S3 multipart give resume; hot video metadata is replicated with a distributed LRU cache; workers scale on queue depth; the CDN serves all segments.

## Web Crawler (pipeline + politeness)

Requirements: crawl from seeds and extract text; fault-tolerant resume; politeness (robots.txt, don't overload); 10B pages in < 5 days.

Design is a system interface with no user API: frontier queue (SQS, carrying URL IDs) → DNS → fetch → raw HTML to S3 → extract text and links → enqueue, with DynamoDB tracking URL state and per-domain timing.

Scale math: ~4 pages/sec/GBps realistic → ~8 machines for 5 days.

For fault tolerance, great is SQS visibility timeouts as exponential backoff (ChangeMessageVisibility × ApproximateReceiveCount), a DLQ after 5 tries, and the pipeline split into fetch and extract stages so a crash loses one stage's work.

Politeness parses robots.txt once per domain into the DB and uses a Redis `SET NX` per-domain lock plus a sliding-window 1 req/sec/domain with jitter.

Dedup checks URLs in the DB before enqueue and content by page hash plus an indexed column (recommended, exact) against a Bloom filter (compact, but false positives skip real pages).

Trap defense is a max depth of 15-20.

## Distributed Rate Limiter (infra)

Requirements: limit by user, API key or IP, configurable rules, 429 plus headers; < 10ms per check; 1M req/sec.

Design: it lives in the API gateway, and Redis Cluster holds token bucket state (steady refill plus bursts, beating fixed window's boundary spike and sliding-log's memory).

Atomicity fixes cross-gateway races with Redis Lua scripts (read-calc-write as one op).

Scale is ~50-100k checks/sec/node → ~10 shards, consistent-hashed by client ID so a client always hits one shard.

Availability is fail-closed, rejecting all when Redis is down, because failing open invites cascade failure exactly during the viral spike (contextual: the opposite call is defensible where lockout hurts more than load).

Latency uses connection pooling and geo-distribution, with explicitly no local caching, since decisions go stale.

Rules are polled every ~30s rather than pushed, for simpler ops.

## Bitly (read-heavy + uniqueness)

Requirements: shorten (custom alias, expiry), redirect < 100ms, 99.99% available, 1B URLs / 100M DAU, ~1000:1 reads.

Scale: up to ~600k reads/sec peak against ~1 write/sec; 500GB total; 62⁶ ≈ 56B → 6 chars.

Design: `GET /{code}` returns a 302, deliberately not a 301, because a 301 is browser-cached forever and surrenders control and analytics.

For uniqueness, great is a global Redis counter → base62 with zero collisions, batching 1000 counter values per instance to cut coordination; hash-and-truncate is viable but needs collision retries, and a DB UNIQUE constraint is the final net.

Redirect speed comes from Redis/Memcached in front of the DB, or CDN edge compute (Cloudflare Workers) for hot codes.

At scale 500GB fits anything; split the Read Service from the Write Service, and go multi-region via disjoint counter ranges (A: 0-1B, B: 1B-2B).

## WhatsApp (connections + delivery)

Requirements: group chat (≤100), send and receive, offline delivery (30 days), media; < 500ms; guaranteed deliverability; minimal server retention.

Design: an L4 LB → chat servers with in-memory userId→socket maps; a WebSocket command API (not REST); DynamoDB tables Chat, ChatParticipant (reverse GSI), Message (30-day TTL), an Inbox of undelivered messages per recipient, and LastSeen; media goes direct to blob storage (30-day TTL).

```
send: durable write first (Message + Inbox) → ack sender → Redis pub/sub to connected recipients → client ACKs clear Inbox; reconnect drains Inbox
```

For routing at billions, bad is a Kafka topic per user (~50KB/topic ≈ 50TB overhead for 1B users), good is consistent-hashing users to servers (ZooKeeper; delicate scaling), and great is Redis pub/sub with channels per user, since 1:1 chats dominate so user channels beat chat channels.

For dead connections, great is application-level heartbeats (ping 10-30s, drop on a missed pong), because TCP keepalives are minutes-slow.

Pub/sub is at-most-once, so great is a per-user global sequence number (Redis INCR) piggybacked on heartbeats: clients detect gaps and re-sync from the Inbox.

Ordering accepts server-timestamp order, because "users would rather see new messages as quickly as possible than guarantee order."

Last seen writes only disconnect timestamps, with "online now" answered live by the owning server.

Multi-device adds a Clients table, a per-client Inbox and a device cap.

## Ad Click Aggregator (streaming analytics)

Requirements: click → redirect; advertiser metrics at 1-min granularity with sub-second queries; no lost clicks and idempotent counting; 10k clicks/sec peak.

Design: `/click` logs then does a server-side 302, since client-side redirects let users bypass tracking.

Bad is raw events plus query-time GROUP BY, good is Cassandra plus periodic Spark → OLAP, and great is Kafka/Kinesis → Flink windowed by AdId+minute (event time, watermarks) → OLAP, flushing every few seconds.

A celebrity ad makes a hot shard, so salt the key (`AdId:0..N`) and let Flink strip and re-sum.

No loss comes from stream retention plus replay covering Flink restarts (checkpointing judged overkill for 1-min windows), with end-to-end truth from lambda-architecture reconciliation: raw events to S3, periodic Spark re-aggregation, batch winning discrepancies.

Idempotency uses HMAC-signed impression IDs verified by the processor plus a Redis dedup set (~1.6GB/day), with the ordering rule that on a miss you write to the stream before the dedup cache, because a rare double count beats a lost click.

Query speed comes from pre-aggregation plus nightly daily and weekly rollups.

## LeetCode (sandboxing + judged simplicity)

Requirements: browse problems, code, submit and get feedback < 5s with isolated, secure execution; a live contest leaderboard (100k users).

Design: API server; DynamoDB (test cases nested as subdocuments, no complex queries needed); an execution layer; a Redis leaderboard.

For isolation, bad is running in the API server, good is VMs (slow, heavy), and great is Docker containers with a read-only FS, CPU and memory limits, a 5s timeout, no network and seccomp; serverless is rejected because cold starts threaten the SLA.

For the leaderboard, great is a Redis sorted set updated in-band (`ZADD`) with clients polling every 5s, WebSockets explicitly rejected as unnecessary.

Scale is auto-scaled per-language container fleets, adding SQS between API and runners only when spikes demand buffering.

Results are plain 1s polling of `GET /check/:id`, which "matches what leetcode.com actually does."

This breakdown is the model for justified simplicity: polling over sockets, no queue until scale demands it, each with a named escalation path; that is the staff signal here.

## Tinder (feeds + pairwise consistency)

Requirements: profile and preferences, a stack of matches by distance and preference, swipe, mutual-swipe match notification (consistent, immediate); stack < 300ms; never re-show a swiped profile; 20M DAU, 2B swipes/day.

Design: Profile (relational plus geo), Feed (Elasticsearch), Swipe (Cassandra, write-optimized), Notification (APNS/FCM).

For match detection, great is the partition key `smaller_id:larger_id`, which puts both swipe directions in one partition for an atomic check; preferred is Redis Lua on a pair-keyed hash (atomic record-and-check, consistent-hashed) with Cassandra as the durable store.

For the stack < 300ms, great is hybrid: serve a cached pre-computed stack, refresh proactively when low, fall back to live ES (CDC-synced), TTL < 1h, pre-compute only for active users, force a refresh on filter or location change.

No re-shows come from a client cache of recent swipes plus a backend Cassandra check, adding a Bloom filter for heavy users, since its false positives only hide unseen profiles and never resurface swiped ones, the acceptable direction of failure.

## Cross-problem regularities

The things a top-tier answer reaches for unprompted recur across the twelve.

Redis TTL locks appear wherever a resource is held briefly (a seat, a driver, a crawl domain), always with a durable backstop.

Presigned URLs with direct client↔S3 transfer handle anything over ~10MB, and app servers never proxy bytes.

Durable write → ack → async propagate appears wherever delivery must survive crashes (the WhatsApp Inbox, Uber's queue-after-match-commit).

Consistency is scoped to the one operation that needs it (booking, matching, swiping) while everything else stays available and eventual.

Replication beats sharding for hot reads, and salting or compound keys handle hot writes.

Simplicity with a stated escalation path (LeetCode's polling, Bitly's single DB) scores higher than reflexive distribution.
