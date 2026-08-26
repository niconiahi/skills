# Common patterns

"The ability to identify and apply patterns is a skill that often separates senior engineers from more junior engineers." Patterns compose: a video platform = large blobs + long-running tasks + realtime updates + multi-step processes. Name the patterns a problem contains, then apply each. (Hello Interview's full pattern articles are premium; this captures their framework pages and free portions — see `research/hellointerview-system-design.md` Part 4 for what's behind the paywall.)

## Realtime updates

Two hops: server↔client channel, and source→server propagation.

- Client channel, in escalation order: simple polling (baseline) → long polling ("the easy solution") → SSE ("the efficient one-way street") → WebSockets ("the full-duplex champion", true bidirectional only) → WebRTC (peer-to-peer). Start with polling "until it no longer serves your needs." Even the site admits SSE networking edge cases cost them "mind boggling" time — simplicity has real value.
- Server side, two architectures: **pub/sub routing** (Redis pub/sub; the WhatsApp approach — decoupled, connection servers subscribe per user) vs **stateful servers in a consistent-hash ring** (the Google-Docs approach — better when per-connection processing is heavy).

## Long-running tasks

Two-phase: web server validates, enqueues a job, returns a job ID in milliseconds; workers poll, execute, update job status; DLQs catch poison messages. Counter-caveat before reaching for it: "If you have short-running jobs, returning the status of the job synchronously... simplifies your architecture dramatically providing clearer back-pressure and better user experience." Sync handling only breaks past 30-60s server/LB timeouts (PDF generation, transcoding).

## Dealing with contention

Double-booking shape: two readers see the last seat, both commit. Escalate **inside the database first**: conditional writes → pessimistic locking (`SELECT FOR UPDATE`) → optimistic concurrency control → isolation levels — and only then distributed coordination. The warning that anchors this: "Databases are built around problems of contention. When you separate your data into multiple databases, you're taking on all of the challenges that the database systems were originally designed to solve."

## Scaling reads

Mature apps run 100:1+ read:write (Instagram: hundreds of reads to open the app, ~one post/day). Three-tier escalation, cheapest first: **in-database** (indexes, denormalization) → **read replicas** → **external caching** (Redis, CDN). Costs to name at each tier: replication lag, cache invalidation, stampedes, hot keys.

## Scaling writes

Order of moves: vertical scaling + right DB choice (LSM stores for write-heavy) → sharding vs vertical partitioning — pick partition keys "that distribute load evenly while keeping related data together" → write queues + load shedding for bursts → batching / hierarchical aggregation. Hot partitions: split keys across shards, dynamic hot-key splitting.

## Handling large blobs

A 100MB BLOB column "kills query performance, backup times, and replication." Threshold: files over ~10MB that don't need SQL go to object storage. Canonical shape: API hands out **presigned URLs**; clients transfer **directly** to/from blob storage/CDN; app servers never touch the gigabyte path; multipart + resumable uploads for when it fails at 99%; metadata DB tracks state.

## Multi-step processes

Even simple sequences get hard distributed: partial failures, external waits (payment gateways), crashes mid-process, evolving steps. Four families, simplest first: single-server primitives → saga (compensating transactions) → event-driven choreography → **workflow orchestration / durable execution** (Temporal, AWS Step Functions) — "declarative workflow definitions where the system guarantees exactly-once execution." Fits payments, order fulfillment, notifications, agent pipelines; skip for simple systems.

## Proximity

PostgreSQL+PostGIS or Redis geospatial; regional subdivision; Elasticsearch geo-queries. "Geospatial indexes are only really necessary when you need to index hundreds of thousands or millions of items" — most queries are local. Full treatment: geospatial section of [`key-technologies.md`](key-technologies.md).
