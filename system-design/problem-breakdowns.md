# Worked problem breakdowns

Twelve complete answer keys from hellointerview.com, each showing the delivery framework end-to-end with deep dives graded **bad → good → great**. Use the one closest to the system at hand; the grading language itself ("a good answer does X, a great answer does Y because Z") is the expert register to write in. Full verbatim notes: `research/hellointerview-system-design.md` Part 5.

## Ticketmaster (booking + contention)

Requirements: view/search/book events; **consistency for booking, availability for search**; 10M users on one event; search < 500ms; 100:1 reads.
Design: Event/Search/Booking services behind a gateway; Postgres; Elasticsearch (CDC-synced); Stripe webhooks; booking = seat → Redis lock (10-min TTL) → "in-progress" row → pay → webhook → transaction marks sold.
Deep dives:
- **No double booking.** Bad: `SELECT FOR UPDATE` held through checkout. Good: status field + expiry cron. Great: implicit status (transaction validates available-or-expired, stamps reservation+timestamp), or the recommended **Redis TTL lock** with DB optimistic concurrency as the backstop if Redis dies.
- **Extreme demand.** Great: **virtual waiting queue** — Redis sorted set gates entry to the booking page, SSE reports position, admitted sessions in a TTL'd set.
- **Search < 500ms.** Good: DB indexes. Great: Postgres tsvector+GIN, or recommended **Elasticsearch with fuzzy matching, CDC-synced**; cache via ES shard-level query cache + CDN.

## Uber (geospatial + matching)

Requirements: fare estimate, request ride, match nearby driver, accept/navigate; match < 1 min; **no driver double-assigned (strong consistency)**; 100k requests from one location.
Scale: 10M drivers × 5s pings ≈ 2M location updates/sec.
Design: Ride, Location, Matching, Notification services; driverId from JWT, never the body.
Deep dives:
- **2M updates/sec.** Bad: direct DB writes (~$200k/day DynamoDB). Great: **Redis GEOADD/GEOSEARCH in-memory** with TTL-expiry of stale drivers — loss self-heals in one 5s ping cycle. Plus adaptive client update intervals.
- **One driver, one request.** Great: **Redis lock keyed by driverId, TTL = the 10s acceptance window**.
- **Surge absorption.** Great: Kafka/SQS in front of matching, offset committed only after successful match, partitioned geographically, consumers auto-scaled on depth.
- **Driver non-response.** Great: **durable execution (Temporal/Step Functions)** modeling offer → timeout → next driver.
- **Further scale.** Geo-sharding + consistent hashing; scatter-gather only at region boundaries.

## Dropbox (large files)

Requirements: upload/download/share/sync; availability; files to 50GB; secure; low latency.
Design: client chunks (5-10MB) + uploads **directly to S3 via presigned URLs**; File Service issues URLs; DynamoDB metadata (chunks array, SharedFiles for ACL); S3 event flips status to "uploaded"; CDN with signed URLs.
Deep dives:
- **Large files.** Great: "trust but verify" — per-chunk presigned URLs, client PATCHes ETags, backend verifies via S3 ListParts. Resumability: SHA-256 fingerprints of file+chunks; retry skips uploaded chunks. Downloads: HTTP Range, no chunking needed.
- **Speed.** Parallel chunk uploads; content-defined chunking (Rabin) → delta sync on small edits; compress only when savings beat CPU (text yes, media no).

## FB News Feed (fan-out)

Requirements: post, follow, reverse-chron feed; availability, ≤1 min staleness; < 500ms; 2B users, unlimited followers.
Design: DynamoDB (Follows: PK following, SK followed, reverse GSI), PrecomputedFeed (~200 post IDs ≈ 2KB/user), SQS fan-out workers.
Deep dives:
- **Feed assembly.** Move it read-time → write-time: precompute; fall back to base tables past the 200-post window.
- **Celebrities.** Great: **hybrid fan-out** — per-account choice; celebrity posts skipped by workers and merged at read time. The 1-minute staleness budget is what buys the async architecture.
- **Viral hot keys.** Great: **replicated (not sharded) post cache** — N independent Redis instances, LB spreads reads, hot traffic ÷ N, "no coordination required." Replication beats sharding for hot reads.

## YouTube (media pipeline)

Requirements: upload + stream; availability; 10s-of-GB videos; low-latency adaptive streaming; 1M uploads/day; resumable uploads.
Design: presigned multipart to S3 → S3 event → **DAG pipeline under an orchestrator (Temporal)**: split (ffmpeg) → parallel per-segment transcodes + audio + transcripts → manifests; metadata in Cassandra by videoId; watch = manifest from CDN, client picks bitrate per network, streams segments.
Deep dives: seconds-long segments × formats (that's what enables adaptive bitrate); chunk fingerprints + S3 multipart for resume; hot video metadata replicated + distributed LRU cache; workers scale on queue depth; CDN serves all segments.

## Web Crawler (pipeline + politeness)

Requirements: crawl from seeds, extract text; fault-tolerant resume; **politeness (robots.txt, don't overload)**; 10B pages in < 5 days.
Design (system interface, no user API): frontier queue (SQS, carries URL IDs) → DNS → fetch → raw HTML to S3 → extract text+links → enqueue; DynamoDB tracks URL state and per-domain timing. Scale math: ~4 pages/sec/GBps realistic → ~8 machines for 5 days.
Deep dives:
- **Fault tolerance.** Great: **SQS visibility timeouts as exponential backoff** (ChangeMessageVisibility × ApproximateReceiveCount), DLQ after 5 tries; pipeline split into fetch/extract stages so a crash loses one stage's work.
- **Politeness.** robots.txt parsed once per domain into the DB; Redis `SET NX` per-domain lock + sliding-window 1 req/sec/domain + jitter.
- **Dedup.** URL: DB check before enqueue. Content: page hash + indexed column (recommended — exact) vs Bloom filter (compact but false positives skip real pages). Trap defense: max depth 15-20.

## Distributed Rate Limiter (infra)

Requirements: limit by user/API-key/IP, configurable rules, 429 + headers; **< 10ms per check**; 1M req/sec.
Design: lives in the API gateway; Redis Cluster holds **token bucket** state (steady refill + bursts; beats fixed window's boundary spike and sliding-log's memory).
Deep dives:
- **Atomicity.** Cross-gateway races fixed with **Redis Lua scripts** (read-calc-write as one op).
- **Scale.** ~50-100k checks/sec/node → ~10 shards, consistent-hashed by client ID (a client always hits one shard).
- **Availability.** **Fail-closed** — reject all when Redis is down; failing open invites cascade failure exactly during the viral spike. (Note: contextual — the opposite call is defensible where lockout hurts more than load.)
- **Latency.** Connection pooling + geo-distribution; explicitly no local caching (stale decisions).
- Rules: ~30s polling beats push (simpler ops).

## Bitly (read-heavy + uniqueness)

Requirements: shorten (custom alias, expiry), redirect < 100ms, 99.99% available, 1B URLs / 100M DAU, ~1000:1 reads.
Scale: up to ~600k reads/sec peak vs ~1 write/sec; 500GB total; 62⁶ ≈ 56B → 6 chars.
Design: `GET /{code}` returns **302, deliberately not 301** — 301 is browser-cached forever, surrendering control and analytics.
Deep dives:
- **Uniqueness.** Great: **global Redis counter → base62**, zero collisions; **batch 1000 counter values per instance** to cut coordination. Hash-and-truncate viable but needs collision retries. DB UNIQUE constraint as final net.
- **Redirect speed.** Redis/Memcached in front of the DB; or CDN edge compute (Cloudflare Workers) for hot codes.
- **Scale.** 500GB fits anything; split Read Service from Write Service; multi-region via disjoint counter ranges (A: 0-1B, B: 1B-2B).

## WhatsApp (connections + delivery)

Requirements: group chat (≤100), send/receive, **offline delivery (30 days)**, media; < 500ms; guaranteed deliverability; minimal server retention.
Design: **L4 LB** → chat servers with in-memory userId→socket maps; WebSocket command API (not REST); DynamoDB: Chat, ChatParticipant (reverse GSI), Message (**30-day TTL**), **Inbox of undelivered per recipient**, LastSeen; media direct to blob storage (30-day TTL). Send: **durable write first (Message + Inbox) → ack sender → Redis pub/sub to connected recipients → client ACKs clear Inbox**; reconnect drains Inbox.
Deep dives:
- **Routing at billions.** Bad: Kafka topic per user (~50KB/topic ≈ 50TB overhead for 1B users). Good: consistent-hash users to servers (ZooKeeper; delicate scaling). Great: **Redis pub/sub, channels per user** (1:1 chats dominate, so user channels beat chat channels).
- **Dead connections.** Great: **application-level heartbeats** (ping 10-30s, drop on missed pong); TCP keepalives are minutes-slow.
- **Pub/sub is at-most-once.** Great: **per-user global sequence number (Redis INCR) piggybacked on heartbeats** — clients detect gaps, re-sync from Inbox.
- **Ordering.** Accept server-timestamp order: "users would rather see new messages as quickly as possible than guarantee order."
- **Last seen.** Write only disconnect timestamps; "online now" answered live by the owning server.
- Multi-device: Clients table, per-client Inbox, device cap.

## Ad Click Aggregator (streaming analytics)

Requirements: click → redirect; advertiser metrics at 1-min granularity, sub-second queries; **no lost clicks; idempotent counting**; 10k clicks/sec peak.
Design: `/click` logs then **server-side 302** (client-side redirects let users bypass tracking). Bad: raw events + query-time GROUP BY. Good: Cassandra + periodic Spark → OLAP. Great: **Kafka/Kinesis → Flink windowed by AdId+minute (event time, watermarks) → OLAP**, flushing every few seconds.
Deep dives:
- **Hot shards.** Celebrity ad → salt the key (`AdId:0..N`); Flink strips and re-sums.
- **No loss.** Stream retention + replay covers Flink restarts (checkpointing judged overkill for 1-min windows). End-to-end truth: **lambda-architecture reconciliation** — raw events to S3, periodic Spark re-aggregation, batch wins discrepancies.
- **Idempotency.** **HMAC-signed impression IDs** verified by the processor + Redis dedup set (~1.6GB/day). Ordering rule: on miss, **write to the stream before the dedup cache** — a rare double count beats a lost click.
- **Query speed.** Pre-aggregation + nightly daily/weekly rollups.

## LeetCode (sandboxing + judged simplicity)

Requirements: browse problems, code, **submit and get feedback < 5s, isolated/secure execution**; live contest leaderboard (100k users).
Design: API server; DynamoDB (test cases nested as subdocuments — no complex queries needed); execution layer; Redis leaderboard.
Deep dives:
- **Isolation.** Bad: run in the API server. Good: VMs (slow, heavy). Great: **Docker containers** — read-only FS, CPU/mem limits, 5s timeout, no network, seccomp. Serverless rejected: cold starts threaten the SLA.
- **Leaderboard.** Great: **Redis sorted set updated in-band** (`ZADD`), clients poll every 5s — WebSockets explicitly rejected as unnecessary.
- **Scale.** Auto-scaled per-language container fleets; add SQS between API and runners only when spikes demand buffering.
- **Results.** Plain 1s polling of `GET /check/:id` — "matches what leetcode.com actually does."
This breakdown is the model for **justified simplicity**: polling over sockets, no queue until scale demands it, each with a named escalation path — that's the staff signal here.

## Tinder (feeds + pairwise consistency)

Requirements: profile + preferences, stack of matches by distance/preference, swipe, **mutual-swipe match notification (consistent, immediate)**; stack < 300ms; **never re-show a swiped profile**; 20M DAU, 2B swipes/day.
Design: Profile (relational + geo), Feed (Elasticsearch), Swipe (Cassandra — write-optimized), Notification (APNS/FCM).
Deep dives:
- **Match detection.** Great: partition key **`smaller_id:larger_id`** puts both swipe directions in one partition for atomic check; preferred: **Redis Lua** on a pair-keyed hash (atomic record-and-check, consistent-hashed), Cassandra as durable store.
- **Stack < 300ms.** Great: **hybrid** — serve cached pre-computed stack, refresh proactively when low, fall back to live ES (CDC-synced); TTL < 1h; pre-compute only for active users; force refresh on filter/location change.
- **No re-shows.** Client cache of recent swipes + backend Cassandra check; add a **Bloom filter** for heavy users — false positives only hide unseen profiles, never resurface swiped ones (the failure mode is the acceptable direction).

## Cross-problem regularities

Patterns that recur across the twelve — the things a top-tier answer reaches for unprompted:

- **Redis TTL locks** wherever a resource is held briefly (seat, driver, crawl domain) — always with a durable backstop.
- **Presigned URLs + direct client↔S3 transfer** for anything over ~10MB; app servers never proxy bytes.
- **Durable write → ack → async propagate** wherever delivery must survive crashes (WhatsApp Inbox, Uber queue-after-match-commit).
- **Consistency scoped to the one operation that needs it** (booking, matching, swiping) while everything else stays available/eventual.
- **Replication over sharding for hot reads**; salting/compound keys for hot writes.
- **Simplicity with an escalation path stated** (LeetCode's polling, Bitly's single DB) scores higher than reflexive distribution.
