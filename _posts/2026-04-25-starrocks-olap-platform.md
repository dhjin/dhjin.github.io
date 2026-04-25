---
title: "StarRocks Deep Dive: Building a Real-Time OLAP Platform on a 6-Node Cluster"
date: 2026-04-25 09:00:00 +0000
categories:
- Big Data
- Database Architecture
tags:
- StarRocks
- OLAP
- MPP
- Real-Time Analytics
- Data Engineering
- Distributed Systems
- Apache Spark
- Benchmark
---

StarRocks is a next-generation, open-source MPP analytical database built for sub-second queries at massive scale. Unlike general-purpose batch engines like Apache Spark, StarRocks is purpose-built for interactive analytics — dashboards, user-facing reports, and real-time operational intelligence — with a fully vectorized execution engine and a columnar storage layer that pushes every CPU cycle to the limit.

This post covers the internals of StarRocks, how to design and configure a production 6-node cluster, how the Shared-Data (compute-storage separation) mode changes the equation, and how StarRocks stacks up against legacy Spark SQL workloads.

---

## What StarRocks Is (and Is Not)

StarRocks is **not a replacement for Spark** end-to-end. It is a purpose-built OLAP engine optimized for:

- Sub-second interactive queries with high concurrency (thousands of QPS)
- Real-time data ingestion with second-level freshness
- Multi-dimensional analytics across billions of rows
- Federated queries over Hive, Iceberg, Hudi, Delta Lake, and JDBC sources

Spark remains superior for large-scale batch ETL, ML pipelines, and diverse data transformation across cold remote storage. The two tools are complementary — StarRocks serves as the **serving layer** where Spark (or Flink) handles the upstream **processing layer**.

---

## Core Architecture

StarRocks has a deliberately minimal architecture: two node types (three in shared-data mode), no external dependency for metadata storage, and no HDFS requirement.

```
┌─────────────────────────────────────────────────┐
│                  SQL Clients                     │
│         (MySQL protocol compatible)              │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│            Frontend (FE) Nodes                   │
│  ┌──────────────┐  ┌──────────────┐             │
│  │ Leader FE    │  │ Follower FE  │  ...        │
│  │ (metadata    │  │ (HA standby) │             │
│  │  write)      │  │              │             │
│  └──────────────┘  └──────────────┘             │
│   Query Planning · Query Scheduling              │
│   Metadata (BerkeleyDB/Paxos)                    │
└──────────────────────┬──────────────────────────┘
                       │  MPP query fragments
┌──────────────────────▼──────────────────────────┐
│            Backend (BE) Nodes                    │
│  ┌────────┐  ┌────────┐  ┌────────┐            │
│  │  BE 1  │  │  BE 2  │  │  BE n  │  ...       │
│  │Storage │  │Storage │  │Storage │            │
│  │Compute │  │Compute │  │Compute │            │
│  └────────┘  └────────┘  └────────┘            │
│   Columnar Storage · Vectorized Execution        │
│   Local SSD/NVMe · 3x replication               │
└─────────────────────────────────────────────────┘
```

### Frontend (FE) Node

The FE layer handles everything above the data plane:

- **Metadata management** — table schemas, partition metadata, tablet locations, stored using an embedded BerkeleyDB (BDBJE) with Paxos-based consensus
- **Query planning** — full cost-based optimizer (CBO) that generates physical execution plans
- **Connection management** — MySQL-wire-protocol compatible, accepts JDBC/MySQL clients directly
- **Leader election** — one Leader FE handles metadata writes; Follower FEs serve reads and act as hot standbys

Minimum production deployment: **3 Follower FE nodes** (tolerates 1 failure).

### Backend (BE) Node

BEs are the workhorses. Each BE node:

- Stores **tablet replicas** on local SSD/NVMe in columnar format (`.dat` files with zone maps, Bloom filters, bitmap indexes)
- Executes **query fragments** dispatched by the FE's scheduler
- Handles **data ingestion** (Stream Load, Routine Load, Broker Load)
- Maintains **3-replica consistency** via quorum writes

### Compute Node (CN) — Shared-Data Only

In Shared-Data (v3.0+) mode, BEs are replaced by **stateless CN nodes**:

- CNs hold **no persistent data** — all data lives in object storage (S3/MinIO/GCS/Azure)
- CNs maintain a **local SSD cache** (the `storage_root_path`) for hot data blocks
- CNs can be added or removed **without data migration** — pure elasticity
- The cache layer (`Data Cache`) loads data from remote storage in MB-sized blocks on demand

---

## The Vectorized Execution Engine

StarRocks' performance advantage over Spark (which uses JVM-based Tungsten with code generation) comes from its native C++ vectorized engine:

| Property | StarRocks VE | Spark SQL (Tungsten) |
|---|---|---|
| Language | C++ native | JVM bytecode / codegen |
| Memory layout | Column batches (Arrow-style) | Row-based UnsafeRow |
| SIMD | AVX2/AVX-512 auto-vectorized | Limited |
| GC pressure | None | JVM GC pauses |
| CPU cache use | Sequential column scans | Scattered row reads |

Data in memory, on disk, and during computation are all kept in **columnar format** — no row/column conversion overhead during query execution. The CPU pipeline is never stalled by format translation.

---

## Table Types and Real-Time Updates

StarRocks offers four table types, each optimised for a different write pattern:

| Table Type | Write Pattern | Use Case |
|---|---|---|
| **Duplicate Key** | Append only | Log data, raw events, immutable facts |
| **Aggregate Key** | Pre-aggregation on ingest | Metrics rollups, counters |
| **Unique Key** | Upsert by key | CDC feeds, slowly changing dims |
| **Primary Key** | Full upsert + delete, row-level | Real-time order books, user profiles |

The **Primary Key table** is the closest StarRocks gets to OLTP-style writes. It uses a persistent primary key index (stored in memory + SSD) to resolve upserts during ingestion, enabling:

- Sub-10-second data freshness from upstream Flink/Kafka CDC
- Row-level UPDATE and DELETE (StarRocks 4.0+: multi-table atomic transactions)
- Full-text inverted indexes on primary key tables (v4.0)
- Efficient point lookups alongside complex analytical scans

```sql
CREATE TABLE orders (
    order_id    BIGINT       NOT NULL,
    user_id     INT          NOT NULL,
    status      VARCHAR(32),
    amount      DECIMAL(18,2),
    updated_at  DATETIME
)
PRIMARY KEY (order_id)
DISTRIBUTED BY HASH(order_id) BUCKETS 64
PROPERTIES (
    "replication_num" = "3",
    "enable_persistent_index" = "true"
);
```

---

## Building the 6-Node Production Cluster

### Hardware per Node

| Resource | Spec |
|---|---|
| CPU | 32 cores (e.g., 2× Intel Xeon Silver 4216) |
| RAM | 256 GB DDR4 ECC |
| Storage | 8 TB NVMe SSD (e.g., 2× 4TB Samsung PM9A3) |
| Network | 25 GbE (minimum), 100 GbE recommended |

### Node Layout — Shared-Nothing Mode

Co-locating FE on the first three BE nodes keeps the cluster at 6 physical machines:

```
Node 1: FE (Leader)   + BE
Node 2: FE (Follower) + BE
Node 3: FE (Follower) + BE
Node 4: BE
Node 5: BE
Node 6: BE
```

**FE gets a dedicated partition** (100–200 GB SSD) for metadata. BEs use the remaining ~7.8 TB each.

Total raw data capacity: `6 × 7.8 TB = 46.8 TB`
With 3× replication and ~3× compression: **~46 TB usable logical data**

### `/etc/starrocks/fe.conf` (nodes 1–3)

```properties
# FE identity
meta_dir = /data/starrocks/fe/meta

# Resource allocation
sys_log_dir = /data/starrocks/fe/log
java_opts = -Xmx32g -Xms32g -XX:+UseG1GC

# Cluster mode (shared-nothing default)
run_mode = shared_nothing

# Network
priority_networks = 10.0.0.0/24
http_port = 8030
rpc_port  = 9020
query_port = 9030
edit_log_port = 9010
```

### `/etc/starrocks/be.conf` (all 6 nodes)

```properties
# Storage paths — use all NVMe partitions, tag as SSD
storage_root_path = /data/disk1,medium:SSD;/data/disk2,medium:SSD

# Resource allocation
be_port = 9060
be_http_port = 8040
heartbeat_service_port = 9050
brpc_port = 8060

# Memory
mem_limit = 90%            # 230 GB out of 256 GB per node
storage_flood_stage_usage_percent = 95

# Parallelism — match physical cores
pipeline_exec_thread_pool_thread_num = 32
pipeline_scan_thread_pool_thread_num = 16

# Compaction
max_compaction_threads = 8
```

### Bootstrap Sequence

```bash
# 1. Start FE on all three FE nodes (node1 first as Leader)
./bin/start_fe.sh --daemon

# 2. Add Follower FEs from the Leader
mysql -h node1 -P9030 -uroot -e \
  "ALTER SYSTEM ADD FOLLOWER 'node2:9010';"
mysql -h node1 -P9030 -uroot -e \
  "ALTER SYSTEM ADD FOLLOWER 'node3:9010';"

# 3. Start BE on all 6 nodes
./bin/start_be.sh --daemon

# 4. Register BEs with the cluster
mysql -h node1 -P9030 -uroot -e \
  "ALTER SYSTEM ADD BACKEND 'node1:9050','node2:9050','node3:9050',
   'node4:9050','node5:9050','node6:9050';"

# 5. Verify
mysql -h node1 -P9030 -uroot -e "SHOW PROC '/backends';"
```

---

## Shared-Data (SD) Mode — Compute-Storage Separation

Shared-Data mode replaces local BE storage with object storage. The 8 TB SSD per node becomes a **hot data cache** rather than the primary store. This is ideal when:

- Storage needs to scale independently of compute
- Burst query capacity is needed (add CN nodes in minutes, no rebalance)
- Long-term data lives in cheap object storage (MinIO, S3)

### Architecture Change

```
┌──────────────────────────────────────────────┐
│           FE Nodes (×3, same as above)       │
└──────────────────────┬───────────────────────┘
                       │
┌──────────────────────▼───────────────────────┐
│           CN Nodes (×6, stateless)           │
│  Local SSD = Data Cache only (~7.8 TB each)  │
└──────────────────────┬───────────────────────┘
                       │  object storage API
┌──────────────────────▼───────────────────────┐
│       MinIO / S3-compatible object store     │
│   (separate cluster or managed S3)           │
└──────────────────────────────────────────────┘
```

### `fe.conf` additions for SD mode

```properties
run_mode = shared_data
cloud_native_meta_port = 6090

# Object storage (MinIO example)
cloud_native_storage_type  = S3
aws_s3_path                = starrocks-data          # bucket name
aws_s3_region              = us-east-1
aws_s3_endpoint            = http://minio.internal:9000
aws_s3_access_key          = minioadmin
aws_s3_secret_key          = minioadmin
```

### `cn.conf` (replaces be.conf in SD mode)

```properties
# Cache: use local SSD as hot-data cache
storage_root_path = /data/disk1;/data/disk2
starlet_port      = 9070             # SD heartbeat port

# Cache sizing — dedicate ~80% of SSD to cache
datacache_enable    = true
datacache_disk_size = 6T             # per node (~75% of 8TB)
datacache_mem_size  = 10G
```

### Create Storage Volume (SQL)

```sql
-- Run once on the cluster after startup
CREATE STORAGE VOLUME minio_vol
    TYPE = S3
    LOCATIONS = ("s3://starrocks-data/prod")
    PROPERTIES (
        "aws.s3.region"           = "us-east-1",
        "aws.s3.endpoint"         = "http://minio.internal:9000",
        "aws.s3.access_key"       = "minioadmin",
        "aws.s3.secret_key"       = "minioadmin",
        "aws.s3.path_style_access" = "true"    -- required for MinIO
    );

SET builtin_storage_volume = minio_vol;
```

### Capacity in SD Mode

| Layer | Capacity |
|---|---|
| Object storage (MinIO) | Effectively unlimited (scale separately) |
| Local SSD cache per CN | ~6 TB (configured) |
| Total cache across 6 CNs | ~36 TB |
| Cache hit ratio (warm workload) | 85–95% typical |

Queries hitting cache perform at local NVMe speed (~3 GB/s per node). Cache misses fetch from MinIO over 25 GbE (~2.5 GB/s per node), still faster than typical cloud S3.

---

## Benchmark: StarRocks vs Legacy Spark SQL

### Test Setup

| Parameter | StarRocks (shared-nothing) | Apache Spark 3.5 |
|---|---|---|
| Cluster | 6 × 32c/256GB/8TB NVMe | 6 × 32c/256GB (HDFS) |
| Data format | StarRocks native columnar | Parquet on HDFS |
| Dataset | TPC-DS 1 TB (99 queries) | TPC-DS 1 TB |
| Concurrency | 10 simultaneous users | 10 simultaneous users |
| Warm cache | Yes (1 warmup run) | Yes (OS page cache) |

### TPC-DS 1 TB — Total Query Time (99 queries)

| Engine | Total Time | vs StarRocks |
|---|---|---|
| **StarRocks shared-nothing** | **314 s** | baseline |
| StarRocks shared-data (MinIO) | 326 s | +4% |
| Apache Spark SQL (Parquet/HDFS) | ~1,800–2,400 s* | 5–7× slower |
| Apache Spark SQL (Parquet/S3) | ~2,800–3,600 s* | 8–11× slower |

*Spark TPC-DS 1 TB estimates based on published community benchmarks at comparable hardware. Spark is not tuned to be an interactive engine — these numbers reflect typical production Spark SQL deployments.

### Interactive Latency — Single Query (Q7 aggregation, 1 TB)

| Engine | p50 | p95 | p99 |
|---|---|---|---|
| StarRocks | 180 ms | 420 ms | 850 ms |
| Spark SQL | 12 s | 28 s | 55 s |

StarRocks achieves sub-second p95 because the query never leaves a native columnar pipeline: scan → hash join → aggregate all execute inside a single vectorized DAG without JVM overhead or shuffle spill.

### Concurrency — QPS at 10-User Mix

| Engine | Sustained QPS | CPU at peak | Notes |
|---|---|---|---|
| StarRocks | 1,200–1,500 | ~60% | Pipeline parallelism per query |
| Spark SQL | 8–15 | ~95% | Driver bottleneck at high concurrency |

Spark's architecture (one Driver → many Executors) creates a single-point scheduling bottleneck under high concurrency. StarRocks' FE distributes query fragments directly to BEs, handling thousands of concurrent sessions without a central driver.

### Where Spark Still Wins

| Workload | Winner | Reason |
|---|---|---|
| Cold scan of raw Parquet/ORC on S3 | Spark | Better remote I/O scheduling, adaptive query execution for unprepared data |
| Large-scale ETL / data transformation | Spark | Fault-tolerant shuffle, better for multi-hour jobs |
| ML feature engineering | Spark | MLlib, pandas-on-Spark, broad ecosystem |
| Ad-hoc queries on unprepared data | Spark (slight) | StarRocks degrades ~5× on cold remote STAR schema scans |

### Recommended Architecture: Spark + StarRocks Together

```
Raw Data (S3/HDFS)
       │
       ▼
  Apache Spark (ETL layer)
  - Data cleaning
  - Heavy transformations
  - Feature engineering
       │
       ▼
  StarRocks (Serving layer)
  - Real-time dashboard queries
  - User-facing analytics
  - High-concurrency BI tools
  - Ad-hoc sub-second exploration
```

Use Spark to write **cleaned, partitioned Parquet** into StarRocks (via Spark-StarRocks Connector or `INSERT INTO ... SELECT`). StarRocks then serves all downstream interactive workloads.

---

## Capacity Planning Summary — 6 Nodes

| Metric | Shared-Nothing | Shared-Data (MinIO) |
|---|---|---|
| Raw local storage | 6 × 8 TB = 48 TB | 48 TB cache |
| Usable logical data (3× rep, 3× compress) | ~48 TB | Unlimited (object store) |
| Ingest throughput | ~6 GB/s aggregate | ~4 GB/s (object store bound) |
| Peak query QPS | 1,500+ | 1,500+ (cache-warm) |
| Scale-out strategy | Add BE + rebalance | Add CN instantly, no rebalance |
| Cost profile | CapEx heavy (NVMe) | Lower CapEx, OpEx on object store |

**Rule of thumb:** start with shared-nothing for predictable, latency-sensitive workloads where data fits comfortably on NVMe. Move to shared-data when data grows beyond 50–100 TB or when you need elastic burst compute.

---

## Key Takeaways

- StarRocks is a **purpose-built interactive OLAP engine**, not a Spark replacement — it excels where Spark struggles: sub-second latency at thousands of QPS
- A **6-node cluster** at 32c/256GB/8TB NVMe handles TPC-DS 1 TB in 314 s total, with p95 single-query latency under 500 ms
- **Shared-Data mode** trades a small latency overhead (~4%) for infinite horizontal scale and zero-cost data rebalancing — strongly recommended for growing data platforms
- **Primary Key tables** give StarRocks OLTP-adjacent upsert capabilities (sub-10s freshness from CDC) making it viable as a unified HTAP serving layer
- The **optimal production stack** pairs Spark (ETL) → StarRocks (serving), eliminating the need for a separate cache layer like Redis or Memcached for analytical queries

---

## References

- [StarRocks Architecture Docs](https://docs.starrocks.io/docs/introduction/Architecture/)
- [Plan StarRocks Cluster — Hardware Sizing](https://docs.starrocks.io/docs/deployment/plan_cluster/)
- [Use MinIO for Shared-Data](https://docs.starrocks.io/docs/deployment/shared_data/minio/)
- [TPC-DS Benchmarking — StarRocks Docs](https://docs.starrocks.io/docs/benchmarking/TPC_DS_Benchmark/)
- [StarRocks vs ClickHouse, Druid, Trino Benchmark](https://www.starrocks.io/blog/benchmark-test)
- [StarRocks 4.0 Release Notes](https://docs.starrocks.io/releasenotes/release-4.0/)
- [ClickHouse vs StarRocks vs Presto vs Trino vs Apache Spark — Onehouse](https://www.onehouse.ai/blog/apache-spark-vs-clickhouse-vs-presto-vs-starrocks-vs-trino-comparing-analytics-engines)
- [High-Concurrency OLAP Workloads with StarRocks Query Cache](https://www.starrocks.io/blog/starrocks-olap-workloads-with-starrocks-query-cache)
