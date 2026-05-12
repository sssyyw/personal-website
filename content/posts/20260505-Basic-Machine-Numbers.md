+++
date = '2026-05-11T22:52:48-04:00'
draft = false
title = 'Back-of-the-Envelope Numbers Every System Designer Should Know'
+++

When you're sketching a system architecture on a whiteboard, you don't need precise benchmarks — you need to know whether your design is within an order of magnitude of feasible. Is one Postgres node enough? Do you need Kafka, or will RabbitMQ do? Should you reach for Cassandra, or is your workload nowhere near needing it?

Here are the numbers I keep in my head, calibrated against published benchmarks from Confluent, Instaclustr, Honeycomb, and others. Treat them as starting points for capacity planning, not SLAs.

## Databases

| Database | Reads/sec/node | Writes/sec/node | Notes |
|---|---|---|---|
| MySQL / Postgres | 25K–50K per core; 100K+ on tuned multi-core; up to 1M+ on very large hardware | 1K–5K durable single-row; 10K+ with batching; 100K+ with COPY/bulk load | Reads scale via replicas; writes bottleneck at the primary |
| Cassandra | 10K–50K | 30K–70K (commodity); 100K+ tuned | LSM-tree storage makes writes faster than reads — unusual |
| Redis | 80K–180K typical; 500K–1M+ with pipelining | Same as reads | In-memory; bounded by CPU and network |

A few things worth internalizing from this table:

**The Postgres write number surprises people.** Marketing benchmarks throw around "millions of writes per second," but those are bulk loads with COPY, not transactional writes with full durability. For real OLTP workloads with `fsync=on`, a single primary realistically handles 1K–5K writes per second. Reads are a different story — they scale to hundreds of thousands per node and millions across replicas.

**Cassandra inverts the usual read/write asymmetry.** Most databases are faster at reads than writes; Cassandra is the opposite because writes go straight to an append-only commit log with no read-before-write. If you're write-heavy, that matters.

**Redis is in a different performance class entirely.** It's not really comparable to the other two — it's a cache or session store, not a system of record. The pipelining number (500K–1M+ ops/sec) is the one that lets you do things like rate limiting at scale on a single node.

## Streaming and messaging

| System | Throughput | Notes |
|---|---|---|
| Kafka | 10K–50K msgs/sec per partition typical; ~20K/sec/partition in production at Honeycomb | Scales linearly by adding partitions and brokers |
| RabbitMQ | 20K–50K msgs/sec (transient); 1K–5K (persistent + confirmed) | Pays throughput cost for routing flexibility |
| ActiveMQ | 1K–10K msgs/sec (Classic); 50K+ (Artemis, non-persistent) | Newer Artemis engine meaningfully faster |
| Flink | 100K+ events/sec/node for keyed/windowed workloads | Dominated by state backend, checkpointing, serialization |

The mental model here: Kafka is a distributed log optimized for sequential append-and-replay; RabbitMQ and ActiveMQ are smart brokers doing per-message routing, filtering, and acknowledgment. You pay for the smarts in throughput, which is why teams pick Kafka for event streaming and analytics, and Rabbit or ActiveMQ for task queues with complex routing rules.

Flink is its own beast — throughput depends heavily on whether you're doing stateless transforms (fast), keyed aggregations with windowing (medium), or large joins with RocksDB state (slower because of disk I/O).

## How to use these numbers

When you sketch an architecture, run two quick checks:

1. **Is each component within 1–2 orders of magnitude of its ceiling?** If your design pushes Postgres to 100K writes/sec on a single primary, you have a sharding problem to solve, not a tuning problem.
2. **Where's the bottleneck likely to be?** Usually it's the write path of your primary datastore, or the network shuffle in your stream processor — rarely the read path or the message queue.

The specific numbers will shift with your hardware, your workload, and your durability requirements. But the rankings between systems stay stable, and that's what you actually need when you're deciding whether your design will work before you've written a line of code.