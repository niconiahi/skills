---
name: system-design
description: Hello Interview's system-design method — delivery framework, opinionated technology defaults, scaling patterns, and staff-level worked examples. Use when designing or architecting a backend system, practicing or conducting a system design interview, choosing between databases/caches/queues/search stores, or hardening a design for scale, consistency, contention, or real-time updates.
---

# System design (the Hello Interview method)

Distilled from [hellointerview.com](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction)'s "System Design in a Hurry" guide, its technology deep-dives, and its worked answer keys, which are the source of every claim here.

Everything reflects their opinionated, current-practice worldview, not generic 2015-era system design lore.

## Reference files

[core-concepts.md](./core-concepts.md) covers networking and protocol choice, indexing, caching, sharding, consistent hashing, CAP/PACELC and the hardware numbers for estimation; reach for it when justifying a consistency choice, picking a real-time protocol, or doing capacity math.

[key-technologies.md](./key-technologies.md) is the toolbox, Postgres, DynamoDB, Cassandra, Redis, Kafka, Elasticsearch, blob storage, API gateway, geospatial indexing and TSDBs, each with when-to-use, when-not, internals and interview-ready numbers; reach for it when choosing or defending a specific technology.

[patterns.md](./patterns.md) holds the eight recurring patterns, realtime updates, long-running tasks, contention, scaling reads, scaling writes, large blobs, multi-step processes and proximity; reach for it when the problem statement matches one of these shapes, and most problems compose two to four of them.

[problem-breakdowns.md](./problem-breakdowns.md) has twelve complete worked designs (Ticketmaster, Uber, Dropbox, News Feed, YouTube, Web Crawler, Rate Limiter, Bitly, WhatsApp, Ad Click Aggregator, LeetCode, Tinder) with every deep dive graded bad/good/great; reach for it when designing anything resembling one of these, or to see the method executed end-to-end.

## The delivery framework

A ~45-minute design runs this sequence in order; the framework exists to think linearly and avoid scope creep.

### Requirements (~5 min)

Functional requirements are "Users should be able to..." statements: interrogate the interviewer or stakeholder like a PM and keep only the top three features, since a long list hurts more than it helps.

Non-functional requirements are "The system should..." statements, always quantified and system-specific, naming which part needs the property.

```
low latency search, < 500ms
```

Consider consistency versus availability (per feature, not global), read/write ratio, burstiness, latency targets, durability, security and fault tolerance.

Skip upfront capacity estimation: compute a number only when it directly changes the design, because "it's a lot, got it" math is a known anti-signal.

### Core entities (~2 min)

Bullet the core nouns and iterate as the design reveals more.

```
Twitter: User, Tweet, Follow
```

### API (~5 min)

Default to REST and don't overthink it; GraphQL only for diverse clients needing different shapes, gRPC only for perf-critical internal service-to-service.

Resources are plural, and the current user derives from the auth token, never from request-body IDs.

```
POST /v1/tweets
GET  /v1/feed
```

Systems without a user API (a crawler) get a system interface, inputs → outputs, instead.

### Data flow (~5 min, data-processing systems only)

A short ordered list of transformations, skipped for everything else.

```
crawl: fetch → parse → extract → store → repeat
```

### High-level design (~10-15 min)

Draw the architecture that satisfies the API, one endpoint at a time, simple and complete first, because layering on complexity early is the classic way to never finish.

Trace data flow and state changes explicitly, since hand-waving is a scored failure.

Note future deep-dive areas verbally ("we'll want a cache here") and move on.

Put schema fields next to DB boxes, design-relevant columns only.

### Deep dives (~10 min)

Now harden: satisfy each non-functional requirement, attack bottlenecks, handle edge cases.

Grade your own options bad → good → great and say why, as the worked examples in [problem-breakdowns.md](./problem-breakdowns.md) model on every dive.

Leave the interviewer room to probe, because they have specific signals to collect.

## Opinionated defaults

One tool per slot, deviate only with a stated reason.

The SQL-versus-NoSQL debate itself is a pothole to avoid completely: pick one, justify with access patterns, move on.

| Slot | Default | Escape hatch |
|---|---|---|
| Relational DB | Postgres | MySQL |
| NoSQL | DynamoDB | Cassandra for write-heavy append-only |
| Cache | Redis (cache-aside + TTL) | Memcached |
| Search | Elasticsearch (synced via CDC, never primary store) | Postgres GIN for lighter needs |
| Queue/stream | Kafka (SQS when managed retries suffice) | — |
| Blob | S3 + metadata DB + presigned URLs + CDN | — |
| Real-time | Long polling first, then SSE, WebSockets only for true bidirectional | — |
| Load balancer | L7; L4 when connections are persistent (WebSockets) | — |

## Level calibration

The breadth/depth ratio is the leveling dial, and proactivity in deep dives is the seniority signal.

Mid (E4) runs 80/20: a clean API and data model, a complete working design, and at least the "good"-tier answer to the problem's central challenge, with the interviewer driving the deep dives and guidance expected.

Senior (E5) runs 60/40: fast through the basics, then proactively leading two or more deep dives with articulated trade-offs ("fan-out on write avoids read latency but requires async infrastructure") and spotting bottlenecks unprompted.

Staff+ (L6+) runs 40/60: two or three dives two or three levels deep into failure modes, grounded in production experience, with multi-region, observability, cost and cell-based architecture raised unprompted; the bar is that the interviewer only refocuses, never steers, and should learn something.

## Potholes

Each of these is a scored negative signal, with the positive habit on the right.

```
long requirement lists                          → top 3 features
ritual DAU/QPS math                             → compute only decision-driving numbers
premature sharding                              → a well-tuned single database with read replicas handles far more than most expect; shard at TB-scale or tens-of-thousands writes/sec, with the math said aloud
"NoSQL because scale"                           → justify by access patterns
reflexive WebSockets                            → polling/SSE first
caching everything                              → cache only read-heavy, rarely-changing data
queues on latency-bound sync paths (< 500ms)    → keep the sync path direct; queues are for async work
verbal hand-waving                              → trace data flow and state changes explicitly
monopolizing                                    → leave the interviewer room
memorized solutions                             → reason from requirements; probing collapses recitation
```
