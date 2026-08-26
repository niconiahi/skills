# Core concepts

Reference for the fundamentals every design draws on. Source pages cited in `research/hellointerview-system-design.md` Part 2.

## Networking and protocol choice

- Default: HTTP over TCP; REST for ~90% of designs.
- Real-time updates, in escalation order: **plain/long polling** (the baseline — start here) → **SSE** (unidirectional server push over standard HTTP; live scores, notifications) → **WebSockets** (only for genuine bidirectional needs: chat, collaboration — not "because 'real-time' was mentioned").
- gRPC: internal perf-critical service-to-service only (binary, HTTP/2); browsers can't speak it, so never for public APIs.
- Load balancing: L7 routes on HTTP content; L4 is faster and required-in-practice for persistent connections (WebSockets).
- Physics floor: NY→London ≈ 80ms minimum from light-in-fiber alone.

## Data modeling and indexing

- Start with a **normalized relational model**; denormalize only with a stated reason.
- Index by query pattern: B-tree default (exact + range + sort), hash (exact only), full-text (Elasticsearch, or Postgres GIN), geospatial (PostGIS). Each index slows writes and costs disk — index the frequently-queried fields, not everything.
- A search index fed by CDC "will lag slightly behind the primary database" — name the lag as an accepted trade-off.

## Caching

- Redis hit ≈ 1ms vs 20-50ms for a typical DB query.
- Default pattern (~90% of cases): **cache-aside with TTL** — check cache, on miss query DB and store.
- Invalidation: immediate invalidation on write, short TTLs accepting staleness, or both.
- **Cache stampede**: a popular entry expires and concurrent requests all miss together. Mitigations: locking, early recomputation, staggered TTLs.
- Cache-tier outage: in-process fallback cache, circuit breakers, graceful degradation.
- Tiering: CDN for static assets; in-process for config/flags; Redis for application data.

## Sharding and consistent hashing

- Shard **only when numbers justify it**: TB-scale storage, tens-of-thousands writes/sec, or reads replicas can't absorb. State the shard key and its trade-off (Instagram by user_id: a user's data co-located, but "trending across all users" becomes a scatter-gather).
- Hash-based for even spread; range-based for natural partitions (multi-tenant). Costs: cross-shard transactions ~impossible, hot spots, painful resharding.
- **Consistent hashing** is why node changes are cheap: adding a server moves only the keys between it and its predecessor (vs ~90% reshuffle with modulo). Used by Redis Cluster, Cassandra, DynamoDB, CDNs. Mention it for elastic scaling; deep explanation rarely needed.

## CAP / PACELC

- Partitions are inevitable; during one you pick consistency or availability. **Availability is the default** — users tolerate a 2-second-stale feed, not a down app.
- Strong consistency where it's owed: **money, inventory, booking limited resources** (Ticketmaster seats, banking).
- Split it per feature: "different consistency requirements for different parts of the same application" is normal and expected.
- PACELC refines CAP: even without a partition, consistency costs latency.
- Safe interview answer: eventual consistency unless the problem involves money, inventory, or booking.

## Numbers to know

Latency ladder: memory ns → SSD µs → intra-datacenter 1-10ms (sub-1ms same-AZ, 1-2ms cross-AZ) → cross-region 50-150ms → cross-continent 100ms+.

Single-node ceilings (use these to kill premature distribution):

- **Cache (Redis)**: ~1ms, 100k+ ops/sec, up to ~1TB in RAM.
- **Database (Postgres)**: up to ~50k transactions/sec, sub-5ms cached reads, a few TB comfortably, 64 TiB+ storage possible.
- **App server**: 100k+ concurrent connections, 8-64 cores, 64-512GB RAM.
- **Big iron exists**: AWS instances go to 4TB and even 24TB RAM, 60TB local SSD, 336TB HDD; S3 is effectively unlimited.
- Datacenter network: 25 Gbps standard, 50-100 Gbps high-performance.
- Rule of thumb: "you don't need sharding until you're hitting tens or hundreds of terabytes."
