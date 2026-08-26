---
name: system-design
description: Hello Interview's system-design method — delivery framework, opinionated technology defaults, scaling patterns, and staff-level worked examples. Use when designing or architecting a backend system, practicing or conducting a system design interview, choosing between databases/caches/queues/search stores, or hardening a design for scale, consistency, contention, or real-time updates.
---

# System design (the Hello Interview method)

Distilled from hellointerview.com's "System Design in a Hurry" guide, technology deep-dives, and their worked answer keys. Full research notes with per-claim URLs: `research/hellointerview-system-design.md` (repo root). Everything here reflects their opinionated, current-practice worldview — not generic 2015-era system design lore.

## Reference files

- [`core-concepts.md`](core-concepts.md) — networking/protocol choice, indexing, caching, sharding, consistent hashing, CAP/PACELC, hardware numbers for estimation. Reach for it when justifying a consistency choice, picking a real-time protocol, or doing capacity math.
- [`key-technologies.md`](key-technologies.md) — the toolbox: Postgres, DynamoDB, Cassandra, Redis, Kafka, Elasticsearch, blob storage, API gateway, geospatial indexing, TSDBs — each with when-to-use, when-not, internals, and interview-ready numbers. Reach for it when choosing or defending a specific technology.
- [`patterns.md`](patterns.md) — the eight recurring patterns: realtime updates, long-running tasks, contention, scaling reads, scaling writes, large blobs, multi-step processes, proximity. Reach for it when the problem statement matches one of these shapes (most problems compose 2-4 of them).
- [`problem-breakdowns.md`](problem-breakdowns.md) — twelve complete worked designs (Ticketmaster, Uber, Dropbox, News Feed, YouTube, Web Crawler, Rate Limiter, Bitly, WhatsApp, Ad Click Aggregator, LeetCode, Tinder) with every deep dive graded bad/good/great. Reach for it when designing anything resembling one of these, or to see the method executed end-to-end.

## The delivery framework

A ~45-minute design runs this sequence. The framework exists to "think linearly and avoid scope creep" — follow it in order.

**1. Requirements (~5 min).** Functional requirements are "Users should be able to..." statements — interrogate the interviewer/stakeholder like a PM and keep only the **top 3 features**; a long list hurts more than it helps. Non-functional requirements are "The system should..." statements, always **quantified and system-specific** ("low latency search, < 500ms" — it names which part needs the latency). Consider: consistency vs availability (per-feature, not global), read/write ratio, burstiness, latency targets, durability, security, fault tolerance. **Skip upfront capacity estimation** — compute a number only when it will directly change the design; "it's a lot, got it" math is a known anti-signal.

**2. Core entities (~2 min).** Bullet the core nouns (Twitter: User, Tweet, Follow). Start small, iterate as the design reveals more.

**3. API (~5 min).** **Default to REST** — "don't overthink this"; GraphQL only for diverse clients needing different shapes, gRPC only for perf-critical internal service-to-service. Plural resources (`POST /v1/tweets`, `GET /v1/feed`). Derive the current user from the **auth token, never from request-body IDs**. Systems without a user API (crawler) get a system interface (inputs → outputs) instead.

**4. Data flow (~5 min, only for data-processing systems).** A short ordered list of transformations (crawl: fetch → parse → extract → store → repeat). Skip it for everything else.

**5. High-level design (~10-15 min).** Draw the architecture that satisfies the API, one endpoint at a time. Simple and complete first — layering on complexity early is the classic way to never finish. Trace data flow and state changes explicitly (hand-waving is a scored failure). Note future deep-dive areas verbally ("we'll want a cache here") and move on. Schema fields next to DB boxes, design-relevant columns only.

**6. Deep dives (~10 min).** Now harden: satisfy each non-functional requirement, attack bottlenecks, handle edge cases. Grade your own options **bad → good → great** and say why (the worked examples in `problem-breakdowns.md` model this on every dive). Leave the interviewer room to probe — they have specific signals to collect.

## Opinionated defaults

One tool per slot, deviate only with a stated reason:

| Slot | Default | Escape hatch |
|---|---|---|
| Relational DB | **Postgres** | MySQL |
| NoSQL | **DynamoDB** | Cassandra for write-heavy append-only |
| Cache | **Redis** (cache-aside + TTL) | Memcached |
| Search | **Elasticsearch** (synced via CDC, never primary store) | Postgres GIN for lighter needs |
| Queue/stream | **Kafka** (SQS when managed retries suffice) | — |
| Blob | **S3** + metadata DB + presigned URLs + CDN | — |
| Real-time | **Long polling first**, then SSE, WebSockets only for true bidirectional | — |
| Load balancer | L7; **L4 when connections are persistent (WebSockets)** | — |

The SQL-vs-NoSQL debate itself is "a pothole you should completely avoid" — pick one, justify with access patterns, move on.

## Level calibration

Breadth/depth ratio is the leveling dial; proactivity in deep dives is the seniority signal.

- **Mid (E4), 80/20**: clean API + data model, complete working design, at least the "good"-tier answer to the problem's central challenge. Interviewer drives deep dives; needing guidance is expected.
- **Senior (E5), 60/40**: fast through basics, then proactively lead 2+ deep dives with articulated trade-offs ("fan-out on write avoids read latency but requires async infrastructure"), spot bottlenecks unprompted.
- **Staff+ (L6+), 40/60**: 2-3 dives "2 or 3 levels deep" into failure modes, grounded in production experience — multi-region, observability, cost, cell-based architecture, raised unprompted. Bar: the interviewer only refocuses, never steers, "and should learn something."

## Potholes

Each of these is a scored negative signal; the positive habit is on the right.

- Long requirement lists → top 3 features.
- Ritual DAU/QPS math → compute only decision-driving numbers.
- Premature sharding → "a well-tuned single database with read replicas can handle way more than most candidates think"; shard at TB-scale or tens-of-thousands writes/sec, with the math said aloud.
- "NoSQL because scale" → justify by access patterns.
- Reflexive WebSockets → polling/SSE first.
- Caching everything → cache only read-heavy, rarely-changing data.
- Queues on latency-bound synchronous paths (< 500ms budgets) → keep the sync path direct; queues are for async work.
- Verbal hand-waving → trace data flow and state changes explicitly.
- Monopolizing → leave the interviewer room.
- Memorized solutions → reason from requirements; probing collapses recitation.
