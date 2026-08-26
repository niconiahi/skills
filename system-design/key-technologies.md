# Key technologies

The toolbox: one dependable option per slot covers ~90% of designs. Breadth before depth — knowing *a* queue matters more than knowing every queue. Source pages cited in `research/hellointerview-system-design.md` Part 3.

## Relational (Postgres/MySQL) — the product-design default

Transactions, strong consistency, joins, every index type (B-tree, hash, geospatial, full-text via GIN). B-trees handle exact match, range, and sort — the default index. Caveats: joins can bottleneck at scale; over-indexing slows writes; "I have relationships in my data" is a weak justification (so does NoSQL). Interview stance: informed architectural decisions, not MVCC/WAL internals. Stretch Postgres before reaching for anything specialized.

## DynamoDB — the NoSQL default

Managed, single-digit-ms, auto-scaling, transactional (`TransactWriteItems`: serializable across up to 100 items). Model: partition key + optional sort key (composite keys give range queries within a partition); 400KB item cap. Indexes: **GSI** (any partition key, eventually consistent *only*, own capacity) vs **LSI** (same partition key, 10GB cap, both consistency modes, creation-time only). Capacity: RCU = 4KB read/sec (eventual = half cost), WCU = 1KB write/sec; hard per-partition limits **3,000 RCU / 1,000 WCU** — this is where hot-partition math comes from. Streams feed Lambdas/Elasticsearch/analytics. DAX = managed write-through cache, µs reads, never caches strongly consistent reads. Avoid when: complex queries/joins/ad-hoc aggregation, or cost at very high write volume.

## Cassandra — write-heavy, availability-first

LSM storage (commit log → memtable → SSTables → compaction) makes writes extremely fast; leaderless ring with vnodes; tunable consistency ONE→ALL (QUORUM+QUORUM overlaps). Model **query-driven, not entity-driven**: `PRIMARY KEY ((partition_key), clustering_keys)` — partition key places, clustering keys sort. Denormalization is intentional. No joins, no ad-hoc aggregation; tombstones tax deletes; large partitions degrade (Discord's fix: time-bucketed partition keys). Pick it for high write throughput with known access patterns; skip it for strict consistency or flexible querying.

## Redis — the swiss-army knife, never the system of record

Single-threaded, all-RAM: ~100k writes/sec/node, sub-ms reads. **Durability is the catch**: RDB loses everything since last snapshot, default AOF loses up to 1s, replication is async — failover can vanish acknowledged writes. Cluster = 16,384 hash slots; multi-key ops require co-location (hash tags `{user:123}:...`).

Uses: cache (TTL + LRU) · distributed lock (`SET key token NX EX 30`, Lua-checked release) — "an efficiency tool that occasionally fails, not a correctness guarantee"; prefer `SELECT FOR UPDATE` when correctness matters · leaderboards (sorted sets) · rate limiting (counters / sorted sets / Lua for atomicity) · geospatial (GEOADD/GEOSEARCH) · streams with consumer groups (occasional double-processing) · pub/sub (**at-most-once, ephemeral** — offline subscribers miss messages).

Hot keys: client-side caching, duplicate the key across slots with random reads, or read replicas. Not for: system of record, working sets beyond RAM, cross-key queries, durable replayable streams (Kafka's job).

## Kafka — durable ordered log; "always available, sometimes consistent"

Topics → partitions (ordered immutable append-only logs); consumer groups give each partition to exactly one consumer; committed offsets resume after failure; leader-follower replication (factor 3, `acks=all` for durability). ~1TB and ~1M messages/sec per broker; messages under 1MB — **never blobs** (S3 + reference instead). Delivery: at-least-once default; exactly-once = idempotent producers + transactions. Consumer-side retry is DIY (retry topics + DLQ) — use SQS if you want managed retries. Hot partitions: better keys (compound e.g. ad-id+region), salting, or backpressure. Reliability discussion belongs on consumer failures and offsets, not broker downtime.

## Elasticsearch — search, never primary storage

Orchestrates Lucene: indices → shards → segments; inverted index for text (tokenization, stemming, fuzzy/edit-distance), doc values for sort/agg, BKD trees for geo. **Keep it a secondary index synced via CDC from the authoritative store** — eventually consistent, poor at write-heavy loads, overkill under ~100k docs, and sync drift is a classic bug source. Denormalize heavily; target results in 1-2 queries; paginate deep with `search_after` (+ PIT for consistency), not from/size. Be ready to justify it over Postgres GIN. Redis full-text search is "quite immature and bad."

## Blob storage (S3) — large files, paired with a metadata DB

Never the primary database. The canonical large-file architecture: **metadata DB for query/index + S3 for bytes + presigned URLs for direct client transfer + CDN for delivery**. Cost anchor: S3 ~$0.023/GB/mo vs DynamoDB ~$1.25/GB/mo. Know presigned URLs, multipart upload, chunking.

## API gateway and load balancer

Gateway = routing + cross-cutting middleware (auth/JWT, rate limiting, SSL termination, logging, CORS). In nearly every product design: say it "handles routing and middleware" and move on — interviewers rarely dig. Use one with microservices; skip for simple client-server. LB choice: L7 for flexibility; **L4 for persistent connections (WebSockets)**.

## Queues, streams, locks, CDN

- **Queue** (Kafka/SQS): buffer bursts, distribute work, decouple. Two warnings: a queue on a < 500ms synchronous path "nearly guarantees you'll break that latency constraint", and queues hide overload — pair with backpressure. Know FIFO, delayed retries, DLQs, partition keys.
- **Stream** (Kafka/Kinesis/Flink): retained + replayable, multiple consumers, windowed processing. Flink for continuous stateful processing and out-of-order/late events — first ask "do I really need real-time latencies?"; batch is often simpler.
- **Distributed lock**: Redis/ZooKeeper atomics with expiry, for cross-system holds (~minutes): cart holds, ride matching, cron exclusivity. DB locks suffice within one database.
- **CDN** (CloudFront/Cloudflare/Akamai): static assets, sometimes cached API responses; TTL + explicit invalidation; not for highly dynamic content.

## Geospatial indexing

Naive lat/lon indexes fail because "a one-dimensional sort order can't preserve two-dimensional closeness" — each index returns a strip, intersection degenerates to brute force. Two families:

- **Spatial trees** — quadtrees, BKD (Elasticsearch), R-trees (**PostGIS GiST — the standard choice**): best for shapes, containment, roads, delivery zones.
- **Encoded keys / space-filling curves** — geohash (5 chars ≈ 5km, 9 ≈ 5m; fix boundary misses with the 3×3 neighbor-cell trick), Google S2 (MongoDB), Uber H3 (hexagons, millions of writes/sec): best for moving points and high write volume. Redis GEO = geohash integers in sorted sets.

Every spatial index only narrows to candidates; **exact distance post-filtering always finishes the job**. Unsure → start with encoded keys. And only bother at all past ~hundreds of thousands of indexed items.

## Time-series databases

"Just because you have time series data doesn't mean you need a time-series database" — stretch Postgres/DynamoDB first. When justified (e.g. 50k writes/sec of metrics): append-only LSM writes, delta-of-delta + XOR compression, time-partitioning (retention = drop partition), downsampling rollups. **Cardinality is where TSDBs break** — tags must be low-cardinality (a user_id tag with 10M values explodes the series count). InfluxDB, TimescaleDB, Prometheus. Staff signal: knowing when the TSDB is overkill.

## Change data capture

Solves the dual-write problem (app writes DB and search index; one fails → drift): read the replication log, propagate changes. For: syncing derived indexes (Elasticsearch from Postgres), OLTP→warehouse, zero-downtime migration. Not for business reactions (emails on order shipped) or cache invalidation — those deserve explicit application events.
