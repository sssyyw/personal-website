+++
date = '2026-05-11T22:52:17-04:00'
draft = false
title = 'Modern Real-Time OLAP Systems: ClickHouse is Winning'
+++

A comparison of ClickHouse, StarRocks, Apache Druid, and Apache Pinot — the four engines defining real-time analytics in 2026.

## Why Real-Time OLAP Matters Now

The old split was clean: OLTP for operations, batch warehouses for BI. Data freshness measured in hours was fine, because the workloads were retrospective.

That model has broken. User-facing analytics, fraud detection, ad bidding, and RAG-style AI applications all need sub-second queries over billions of rows, with data that's seconds old. Real-time OLAP databases close the gap between streaming ingestion (Kafka, Flink) and interactive analytical queries.

Four open-source engines lead the space:

- **ClickHouse** — vectorized scans, append-heavy workloads
- **StarRocks** — MPP joins, mutability, lakehouse federation
- **Apache Druid** — time-series segments, bitmap indexes
- **Apache Pinot** — extreme-concurrency point lookups

They're all columnar, distributed, and fast — but they diverge sharply on architecture, mutability, joins, and operational cost.

## Architecture

### ClickHouse: shared-nothing, vectorized C++

Built at Yandex for web analytics, ClickHouse is a monolithic, shared-nothing columnar engine written in C++. Each node stores data locally and executes queries against it.

The defining design choice is hardware-level optimization. Queries process columnar data in blocks, leveraging SIMD instructions to scan billions of rows per second per node. Data lives in the **MergeTree** family of engines: aggressive compression, async background merges, no read locks. Replication uses ZooKeeper or the lightweight ClickHouse Keeper. ClickHouse Cloud adds a decoupled compute-storage model.

### StarRocks: MPP with decoupled storage

StarRocks descends from Apache Doris but is ~90% rewritten. The architecture is unusually clean: two daemon types.

- **Frontend (FE)** — metadata, SQL parsing, query planning, client connections
- **Backend (BE)** — data storage and query execution

It runs in two modes: **shared-nothing** (BEs with local NVMe for maximum I/O) or **shared-data** (S3-compatible object storage with stateless Compute Nodes for cloud-native elasticity). The dual model lets compute and storage scale independently.

### Apache Druid: decoupled, complex

Druid was built at Metamarkets for programmatic ad analytics. It separates ingestion, storage, and query — which sounds great until you count the moving parts. A production cluster requires **six node types**: Coordinator, Overlord, Broker, Historical, MiddleManager, Router. Plus deep storage (S3/HDFS) and ZooKeeper.

Data is ingested into immutable, bitmap-indexed **segments**, pushed to deep storage, and pulled onto Historical nodes. Workload isolation is excellent. Operational burden is severe.

### Apache Pinot: scatter-gather for user-facing analytics

Pinot was built at LinkedIn to power "Who viewed your profile" — millions of users hitting analytical queries under 100ms. Architecture-wise it resembles Druid: Controllers, Brokers, Servers, Minions, with Helix and ZooKeeper coordinating cluster state.

Where ClickHouse brute-forces scans, Pinot leans entirely on pre-computation and layered indexes. It's optimized for predictable latency at extreme concurrency, at the cost of ad-hoc flexibility.

## Storage and Mutability

### ClickHouse: append-first

Data lands in compressed parts sorted by primary key. Background threads merge parts into larger segments. Columnar layout enables LZ4/Zstandard compression with 10x+ ratios — meaning less disk I/O and lower cloud storage costs.

Mutations are an afterthought. `ALTER TABLE ... UPDATE/DELETE` and `ReplacingMergeTree` write new versions to new parts; old rows linger until background merges purge them. Queries often need `FINAL` to deduplicate on the fly, which tanks latency under heavy CDC. Recent lightweight deletes and patch parts help, but the engine is fundamentally append-optimized.

Schema evolution, on the other hand, is excellent — `ALTER TABLE` is instant, no data rewrite required.

### StarRocks: real mutability

StarRocks offers four table types: Duplicate Key (append-only), Aggregate (pre-aggregation), Unique Key, and **Primary Key**. The Primary Key table is the differentiator.

When a row is upserted, StarRocks loads the tablet's primary key index into memory, evaluates the unique constraint at write time, and marks superseded rows as deleted immediately. Reads see only the latest authoritative row — no deduplication pass, no `FINAL` keyword, predictable latency.

This makes StarRocks the natural target for live CDC from PostgreSQL or MySQL. Online schema changes work without halting ingestion.

### Druid and Pinot: the immutability penalty

Both rely on immutable segments. Any update — fixing pipeline errors, GDPR deletes, dimension changes — requires reingesting and reindexing segments, often via external Spark jobs.

Pinot has bolted on upsert support for real-time Kafka tables, but it's specialized and brittle. Schema evolution is rough: adding a column with a default value often requires a rolling restart of the server cluster to reload segments — a non-starter for the user-facing SLAs Pinot is supposed to serve.

## Query Execution and Joins

Joins are the cleanest dividing line between these engines.

### StarRocks: real distributed joins

StarRocks ships a mature **Cost-Based Optimizer** built on the Cascades/ORCA framework. It analyzes table statistics, cardinality, and distribution to pick join orders and algorithms.

The killer feature is **Colocate Joins**: fact and dimension tables that share a distribution key get joined locally on each backend, with zero network shuffle. When colocation isn't possible, StarRocks falls back to Broadcast Joins (small dimensions) or Shuffle Joins (large fact tables). Star and snowflake schemas work without compromise.

### ClickHouse: the denormalization tax

ClickHouse uses a rule-based planner. Multi-table joins with varied cardinalities hit catastrophic plans, network thrash, and OOMs.

The workaround: denormalize upstream. Engineers build wide flat tables via stateful Flink jobs before ingesting. This duplicates dimensions across billions of rows, inflates storage, and freezes the schema — if a stakeholder wants a new dimension, the whole Flink pipeline gets rebuilt. Parallel Hash and Grace Hash joins have improved things, but the optimizer is still well behind StarRocks.

### Druid and Pinot: limited joins

Both target flat event streams. Joins are restricted to broadcast/lookup against small dimensions. Pinot has no CBO. Distributed shuffle joins across large fact tables aren't viable in either. For normalized reporting schemas, both are structurally unsuitable.

## Indexing

| Engine | Strategy |
|---|---|
| ClickHouse | Sparse primary index (one entry per ~8,192 rows) + SIMD brute-force scan |
| Pinot | Forward, inverted, sorted, range, bloom, JSON, geospatial, text, **Star-Tree** |
| Druid | Dictionary + bitmap inverted indexes for fast boolean filtering |
| StarRocks | Sorted keys, bloom filters, bitmap, n-gram bloom, ZoneMap |

ClickHouse minimizes indexing — fewer index files, better compression, faster scans. Pinot maximizes it: its proprietary **Star-Tree** index pre-aggregates only up to a configurable threshold, balancing latency and storage bloat. Druid's bitmap indexes shine for high-cardinality time-series filtering but make ingestion notoriously slow.

## Performance

### Bulk Load and Storage

ClickBench on 100 GB (c6a.4xlarge, identical hardware):

| Metric | ClickHouse | StarRocks | Druid | Pinot |
|---|---|---|---|---|
| Load time | **269s** | ~400s | 19,620s (~5.5h) | Failed to fully load |
| Disk footprint | **14.26 GiB** | ~18 GiB | 42.09 GiB | N/A |
| Compression ratio | ~7x | ~5–6x | ~2.4x | ~3x |

ClickHouse wins ingestion and compression decisively. Druid is 3x larger and roughly 70x slower to load due to segment building, dictionary encoding, and bitmap construction. Pinot's general-purpose benchmark instability reflects its specialized tuning requirements.

### Streaming Ingestion Throughput

Per-node sustained ingestion under typical Kafka pipelines:

| Engine | Per-node throughput | Ingestion-to-query lag |
|---|---|---|
| ClickHouse | 500K–2M rows/sec (Buffer/Async inserts) | 1–10s |
| StarRocks | 100K–1M rows/sec (Stream Load, Primary Key 50–200K) | 1–5s |
| Druid | 10K–100K events/sec per ingestion task | 30s–2min (segment handoff) |
| Pinot | 100K–500K events/sec per server | 1–10s (real-time tables) |

Druid's segment handoff is the slowest path to query visibility; for sub-minute freshness, ClickHouse and StarRocks are the safer picks.

### Query Latency by Workload

Latency targets vary sharply by query shape. Approximate p50 / p99 on a properly sized cluster:

| Workload | ClickHouse | StarRocks | Druid | Pinot |
|---|---|---|---|---|
| Single-table aggregation | 50ms / 500ms | 80ms / 600ms | 200ms / 2s | 100ms / 1s |
| Star schema join (3–5 tables) | 1–5s / OOM risk | **150ms / 1s** | Not viable | Not viable |
| Time-series filter + group-by | 100ms / 800ms | 150ms / 1s | **80ms / 500ms** | 50ms / 400ms |
| Point lookup (indexed) | 200ms / 1s | 100ms / 500ms | 150ms / 800ms | **5ms / 50ms** |
| Ad-hoc TPC-H query | 5s+ | **1s** | Fails | Fails |

Standardized benchmark winners:

| Benchmark | Winner | Runner-up |
|---|---|---|
| ClickBench (flat, aggregations) | ClickHouse | StarRocks (close) |
| SSB (star schema, joins) | StarRocks (1.87x faster than CH) | ClickHouse |
| TPC-H (complex joins) | StarRocks (3–5x faster than CH) | ClickHouse |

### Concurrency and SLA Profiles

Latency is meaningless without knowing how many concurrent queries each engine can hold to that latency. The four engines target very different SLAs:

| Engine | Target QPS per cluster | p99 SLA | Typical use |
|---|---|---|---|
| ClickHouse | 100–1,000 | 100ms–2s | Internal BI, observability, ad-hoc exploration |
| StarRocks | 1,000–10,000 | 100ms–500ms | Mixed workload: BI + customer-facing dashboards |
| Druid | 100–1,000 | 200ms–1s | Operational dashboards, time-series slice-and-dice |
| Pinot | **10,000–100,000+** | **10–100ms** | User-facing analytics at consumer scale |

Pinot is the only engine designed from day one for tens of thousands of concurrent queries with tight tail latency — LinkedIn runs it at >250K QPS for "Who viewed your profile." ClickHouse and Druid degrade noticeably above ~1K concurrent users without aggressive caching layers. StarRocks sits comfortably in the middle and can serve customer-facing workloads if the query patterns aren't extreme.

### What Breaks Each Engine

- **ClickHouse**: heavy CDC (mutations trigger `FINAL` deduplication), complex joins (RBO picks bad plans → OOM), >1K concurrent users.
- **StarRocks**: very high cardinality Primary Key tables (memory pressure on PK index), >10K QPS without careful tuning.
- **Druid**: any join beyond broadcast, mutable data, ingestion freshness below ~30s.
- **Pinot**: ad-hoc queries that bypass the Star-Tree, multi-table relationships, schema changes under live traffic.

## Streaming and Lakehouse Federation

Kafka integration is table stakes for all four. The interesting divergence is upstream complexity and data lake support.

**StarRocks** is the strongest federation engine. It queries Iceberg, Hudi, and Delta directly on object storage via its CBO, hitting performance close to native SSD queries at ~5% of the storage cost. Native joins mean less Flink preprocessing.

**ClickHouse** offers external table integrations, but data lake federation is shallow — performance generally requires ingesting into native MergeTree tables.

**Druid and Pinot** can't perform ad-hoc queries against open table formats. Both need separate batch pipelines to load data into proprietary segments.

## Ecosystem (2026)

| Engine | Stars | Governance | Position |
|---|---|---|---|
| ClickHouse | 46,700+ | ClickHouse Inc. | Market leader, ~1,300 commits/week |
| Apache Druid | 12,700+ | Apache | Mature, slowing adoption |
| StarRocks | 11,500+ | Linux Foundation | Explosive growth, 500+ contributors |
| Apache Pinot | 6,100+ | Apache (StarTree commercial) | Niche, elite engineering shops |

ClickHouse owns the broad market — Cloudflare, eBay, and Lyft run it at petabyte scale, many having migrated off Druid for 90% infrastructure reduction. StarRocks is gaining fast at ClickHouse's expense, particularly in shops that need joins and mutability. Druid has visible momentum loss as teams tire of its operational footprint. Pinot stays niche, serving LinkedIn, Uber, and Stripe where its concurrency profile justifies the complexity.

## Choosing an Engine

There's no "best" — only fit. The decision turns on four questions:

**Pick ClickHouse if:** your data is append-only event streams (logs, telemetry, web analytics), queries are mostly single-table aggregations, and you can flatten upstream. You'll get the smallest storage footprint and the fastest scans on the market.

**Pick StarRocks if:** you need real upserts from CDC, run star/snowflake schemas with complex joins, or want to query an Iceberg lakehouse directly. It's the closest thing to a real-time data warehouse.

**Pick Druid if:** you're doing high-cardinality time-series dashboarding (ad-tech, network monitoring) and can afford the operational overhead. Bitmap indexes still win for slice-and-dice.

**Pick Pinot if:** you're serving user-facing analytics to millions of concurrent users with predictable query patterns, and the Star-Tree fits your aggregation shape. Accept the join limitations and operational complexity as the cost of admission.

The biggest practical shift in 2026 is that StarRocks has eroded ClickHouse's "single best default" status. If your workload has any meaningful join or mutability requirement, StarRocks is now the safer bet — and ClickHouse should be reserved for the pure scan-heavy workloads where it's still untouchable.
