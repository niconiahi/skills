# Core concepts

Reference for the fundamentals every design draws on, sourced from hellointerview.com's core-concepts pages.

## Networking and protocol choice

The default is HTTP over TCP, with REST for ~90% of designs.

Real-time updates escalate in order: plain or long polling is the baseline and where you start, SSE is unidirectional server push over standard HTTP for live scores and notifications, and WebSockets are only for genuine bidirectional needs like chat and collaboration, never because "real-time" was mentioned.

gRPC is for internal perf-critical service-to-service only (binary, HTTP/2); browsers can't speak it, so it is never a public API.

L7 load balancing routes on HTTP content; L4 is faster and required in practice for persistent connections (WebSockets).

The physics floor is NY→London ≈ 80ms minimum from light-in-fiber alone.

## Data modeling and indexing

Start with a normalized relational model and denormalize only with a stated reason.

Index by query pattern: B-tree is the default (exact, range and sort), hash is exact only, full-text is Elasticsearch or Postgres GIN, geospatial is PostGIS.

Each index slows writes and costs disk, so index the frequently-queried fields, not everything.

A search index fed by CDC lags slightly behind the primary database; name the lag as an accepted trade-off.

## Caching

A Redis hit is ≈1ms against 20-50ms for a typical DB query.

The default pattern for ~90% of cases is cache-aside with TTL: check the cache, on a miss query the DB and store.

Invalidation is immediate on write, short TTLs accepting staleness, or both.

A cache stampede is a popular entry expiring so concurrent requests all miss together; the mitigations are locking, early recomputation and staggered TTLs.

A cache-tier outage is survived by an in-process fallback cache, circuit breakers and graceful degradation.

Tiering puts a CDN on static assets, in-process caches on config and flags, and Redis on application data.

## Sharding and consistent hashing

Shard only when the numbers justify it: TB-scale storage, tens of thousands of writes per second, or reads the replicas can't absorb.

State the shard key and its trade-off.

```
Instagram by user_id   a user's data co-located, but "trending across all users" becomes a scatter-gather
```

Hash-based sharding gives an even spread; range-based fits natural partitions (multi-tenant).

The costs are cross-shard transactions being near-impossible, hot spots, and painful resharding.

Consistent hashing is why node changes are cheap: adding a server moves only the keys between it and its predecessor, against a ~90% reshuffle with modulo.

Redis Cluster, Cassandra, DynamoDB and CDNs use it; mention it for elastic scaling, since a deep explanation is rarely needed.

## CAP / PACELC

Partitions are inevitable, and during one you pick consistency or availability.

Availability is the default, because users tolerate a two-second-stale feed but not a down app; strong consistency is owed where money, inventory or the booking of limited resources is involved (Ticketmaster seats, banking).

Splitting it per feature is normal and expected: different parts of one application have different consistency requirements.

PACELC refines CAP: even without a partition, consistency costs latency.

The safe interview answer is eventual consistency unless the problem involves money, inventory or booking.

## Numbers to know

The latency ladder runs memory ns → SSD µs → intra-datacenter 1-10ms (sub-1ms same-AZ, 1-2ms cross-AZ) → cross-region 50-150ms → cross-continent 100ms+.

Single-node ceilings kill premature distribution; the rule of thumb is that sharding is unnecessary until tens or hundreds of terabytes.

```
Cache (Redis)         ~1ms, 100k+ ops/sec, up to ~1TB in RAM
Database (Postgres)   up to ~50k transactions/sec, sub-5ms cached reads, a few TB comfortably, 64 TiB+ storage possible
App server            100k+ concurrent connections, 8-64 cores, 64-512GB RAM
Big iron              AWS instances to 4TB and even 24TB RAM, 60TB local SSD, 336TB HDD; S3 effectively unlimited
Datacenter network    25 Gbps standard, 50-100 Gbps high-performance
```
