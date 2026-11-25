---
title: "StarRocks Internals & Lakehouse Engine: CBO, Pipeline Execution, Delete Vectors, and Async Materialized Views"
date: 2026-11-25 09:00:00 +0000
categories:
- Big Data
- Database Architecture
tags:
- StarRocks
- OLAP
- MPP
- Query Optimizer
- Vectorized Execution
- Apache Iceberg
- Data Lakehouse
- Materialized Views
---

In the [previous StarRocks post](/big%20data/database%20architecture/2026/04/25/starrocks-olap-platform/), we covered how to size and deploy a 6-node StarRocks cluster, the trade-offs between Shared-Nothing and Shared-Data modes, and how StarRocks compares against Apache Spark SQL on TPC-DS benchmarks.

That post answered *what* StarRocks delivers. This post goes under the hood to answer *how* it delivers sub-second latency on multi-table joins, real-time CDC upserts, and open data lakehouse tables (Apache Iceberg, Paimon, Hudi, and Delta Lake) without requiring data movement.

We will dissect four core subsystems:

1. **The Cascades Cost-Based Optimizer (CBO)** and distributed join planning with Global Runtime Filters
2. **The Vectorized Pipeline Execution Engine** — SIMD columnar batches, Late Materialization, and low-Context-Switch cooperative scheduling
3. **Delete Vector (`DelVec`) Primary Key Storage** — why Merge-on-Write beats Merge-on-Read by 3–10× on real-time CDC workloads
4. **The Lakehouse Acceleration Layer** — multi-tier Data Cache and transparent query rewriting with Asynchronous Materialized Views

---

## 1. The Cost-Based Optimizer (CBO) & Distributed Join Strategies

Many first-generation real-time OLAP engines (such as ClickHouse or Apache Druid) were architected around wide, denormalized flat tables. While they scan single tables at blistering speeds, multi-table star or snowflake joins often require manual query tuning or pre-joined ETL pipelines.

StarRocks takes a fundamentally different approach: its Frontend (FE) embeds a native, Cascades-style **Cost-Based Optimizer (CBO)** designed from the ground up for distributed MPP execution.

```
SQL Query
   │
   ▼
┌───────────────────────────────────────────────────────────┐
│ 1. Parser & Analyzer → Logical Plan (AST)                 │
├───────────────────────────────────────────────────────────┤
│ 2. Rule-Based Transformations (RBO)                       │
│    - Predicate pushdown / Partition & Bucket pruning      │
│    - Subquery unnesting / Common Subexpression Elimination│
│    - Transparent Materialized View Query Rewrite (SPJG)   │
├───────────────────────────────────────────────────────────┤
│ 3. Cascades Cost-Based Search (CBO)                       │
│    - Column statistics (NDV, Min/Max, Nulls, Histograms)  │
│    - Join reordering (DP / Greedy / Left-deep & Bushy)    │
│    - Physical join selection (Broadcast/Shuffle/Colocate) │
├───────────────────────────────────────────────────────────┤
│ 4. Distributed Plan Fragment Generation + Runtime Filters │
└───────────────────────────────────────────────────────────┘
```

### Column Statistics and Cardinality Estimation

To choose the optimal join order across 5, 10, or 20 tables, the CBO relies on table and column statistics stored in `_statistics_.column_statistics`:

- **Row count & average column size**
- **NDV (Number of Distinct Values)** estimated via HyperLogLog (HLL)
- **Min / Max values** and **Null fraction**
- **Equi-height histograms** for skewed columns

With accurate NDV and histogram data, the CBO avoids the classic MPP pitfall where an underestimated intermediate join explodes across the network.

### The Four Physical Distributed Join Strategies

When joining a large fact table $L$ (`orders`, 10B rows) with a dimension or secondary table $R$, the CBO automatically selects one of four physical distribution strategies based on estimated byte cost:

| Join Strategy | Network Movement | When the CBO Selects It |
|---|---|---|
| **Broadcast Join** | Replicates the entire right table $R$ to every BE scanning $L$ | $R$ is small after filtering (typically $< 100\text{ MB}$); avoids shuffling $L$ |
| **Shuffle (Partitioned) Join** | Hash-partitions both $L$ and $R$ on the join key across the network | Both $L$ and $R$ are large and not co-partitioned |
| **Bucket Shuffle Join** | Hash-partitions only $R$ using $L$'s exact bucketing hash function | $L$ is already bucketed on the join key; saves $100\%$ of network I/O on $L$ |
| **Colocate Join** | **Zero network I/O** — join executes purely locally on each BE | $L$ and $R$ belong to the same `colocate_with` group and share bucket counts |

```sql
-- Example: Enabling zero-network Colocate Join between two large tables
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    amount DECIMAL(18, 2)
)
PRIMARY KEY (order_id, customer_id)
DISTRIBUTED BY HASH(customer_id) BUCKETS 32
PROPERTIES (
    "colocate_with" = "customer_group"
);

CREATE TABLE customer_profiles (
    customer_id BIGINT,
    segment VARCHAR(32),
    region VARCHAR(32)
)
PRIMARY KEY (customer_id)
DISTRIBUTED BY HASH(customer_id) BUCKETS 32
PROPERTIES (
    "colocate_with" = "customer_group"
);
```

### Global Runtime Filters (Dynamic Join Pushdown)

Even when a join is well-ordered, scanning billions of probe-side rows that will ultimately be discarded by an inner join wastes disk I/O and CPU. StarRocks solves this with **Global Runtime Filters**.

```
Probe Side (Fact Table: orders)            Build Side (Dim Table: customers)
┌───────────────────────────┐              ┌──────────────────────────────┐
│ 3. Storage Scan           │              │ 1. Scan & Filter             │
│    Apply RF1 at Segment   │              │    WHERE region = 'APAC'     │
│    & Page level           │              └──────────────┬───────────────┘
└─────────────┬─────────────┘                             │
              │ (95% rows skipped                         ▼
              │  before decoding)          ┌──────────────────────────────┐
              ▼                            │ 2. Build Hash Table          │
┌───────────────────────────┐              │    Emit Runtime Filter (RF1) │
│ 4. Hash Join Probe        │◄─────────────┤    (Bitset / Bloom / MinMax) │
└───────────────────────────┘   Broadcast  └──────────────────────────────┘
                                or Merge
```

1. While building the hash table on the right side (`customers WHERE region = 'APAC'`), the BE constructs a compact filter representation of the surviving join keys — a **Min/Max range**, an **IN-list**, a **Bloom Filter**, or a **Bitset**.
2. In distributed joins, local runtime filters are merged by a coordinator node into a **Global Runtime Filter** and pushed down to the scan operators on the left side (`orders`).
3. The storage engine evaluates the runtime filter against **ZoneMaps (page min/max)** and **Bloom indexes** before even decompressing data pages. Frequently, 80–99% of the fact table is skipped at the storage layer.

---

## 2. Vectorized Execution & The Pipeline Engine

Once the FE compiles a query into plan fragments and dispatches them to Backend (BE) or Compute Nodes (CN), execution is handed over to a C++ engine built around two principles: **columnar vectorization** and **event-driven pipeline scheduling**.

### Row-at-a-Time vs. Columnar Vectorized Chunks

Traditional Volcano-model engines (`next()` per row) suffer from virtual function call overhead, instruction cache misses, and poor CPU pipelining. StarRocks processes data in **Chunks** of up to **4,096 rows** stored in contiguous, cache-aligned columnar arrays.

```
Volcano (Row-oriented)                 StarRocks Vectorized Chunk (4096 rows)
┌────────────────────────┐             col_price: [ 19.99 | 45.00 | 12.50 | ... ] (contiguous float64[])
│ Row 0: [19.99, 2, 'A'] │             col_qty:   [   2   |   1   |   5   | ... ] (contiguous int32[])
│ Row 1: [45.00, 1, 'B'] │             filter:    [   1   |   0   |   1   | ... ] (uint8[] selection byte-map)
└────────────────────────┘
  Virtual call per row                   Single tight loop → SIMD (AVX2 / AVX-512 / NEON)
```

Because values of the same data type sit back-to-back in memory:

- Arithmetic, comparisons, and hashing compile down to tight loops that use **SIMD instructions** (`AVX2` / `AVX-512` on x86_64, `NEON` on ARM).
- **Filter representation without copying**: When evaluating `WHERE price > 20`, StarRocks does not copy surviving rows into a new buffer immediately. Instead, it populates a `uint8_t` filter byte array (or selection vector) and passes it along the operator chain.
- **Low-Cardinality Dictionary Optimization**: For string columns encoded with dictionary compression on disk (`region = 'KR'`), StarRocks reads the integer dictionary codes directly into the vectorized Chunk. String comparisons, `GROUP BY`, and hash joins execute on 32-bit integers rather than variable-length string pointers, decoding back to strings only at the final result projection.

### Late Materialization at the Storage Layer

When a query filters on two columns and projects ten columns:

```sql
SELECT col1, col2, ..., col10
FROM user_events
WHERE event_type = 'PURCHASE' AND amount > 1000;
```

StarRocks reads and decodes **only** `event_type` and `amount` first. Using ordinal page offsets, it identifies the exact row numbers that satisfy the predicate, and then seeks directly to the matching pages of `col1..col10`. If the predicate selects $0.5\%$ of rows, $99.5\%$ of the pages for the other ten columns are never touched on disk.

### The Pipeline Execution Engine (Cooperative Multitasking)

Early MPP engines mapped one OS thread per plan fragment instance. Under high concurrency (e.g., 500 concurrent queries × 4 fragments × 16 parallel degree = 32,000 threads), the Linux kernel spends more CPU cycles context-switching and waiting on blocked network/disk I/O than executing instructions.

StarRocks replaces thread-per-fragment execution with a **Pipeline Engine**:

```
Plan Fragment
   │
   ▼  Decomposed into non-blocking Pipelines
┌──────────────────────────────────────────────────────────┐
│ Pipeline 1: [OlapScanSource] ──► [Filter] ──► [HashBuildSink]
│ Pipeline 2: [ExchangeSource] ──► [HashProbe] ──► [AggSink]
│ Pipeline 3: [AggSource]      ──► [Project]   ──► [ResultSink]
└─────────────────────────────┬────────────────────────────┘
                              │Split into Drivers (DOP = CPU cores)
                              ▼
┌──────────────────────────────────────────────────────────┐
│ User-Space Cooperative Scheduler (Fixed Thread Pool = N) │
│  ┌─────────────┐  ┌─────────────┐         ┌───────────┐  │
│  │ WorkerThd 1 │  │ WorkerThd 2 │   ...   │WorkerThd N│  │
│  └──────┬──────┘  └──────┬──────┘         └─────┬─────┘  │
│         └────────────────┼──────────────────────┘        │
│                          ▼                               │
│     Ready Queue: [Driver A1] [Driver B3] [Driver C2]     │
│     Blocked I/O Queue (Scan / Network Exchange)          │
└──────────────────────────────────────────────────────────┘
```

1. **Blocking Operators Split into Pipelines**: Every blocking operator (Hash Join Build, Aggregation, Sort, Network Exchange) is split into a `Sink` operator (which terminates one pipeline) and a `Source` operator (which starts the downstream pipeline).
2. **Fixed Worker Thread Pool**: A BE node with 32 CPU cores runs a fixed pool of 32 execution threads.
3. **Zero Blocking in Worker Threads**: Each pipeline is instantiated into multiple **Pipeline Drivers** according to the Degree of Parallelism (DOP). When a Driver's scan operator needs to wait for NVMe or object storage I/O, or when an exchange operator waits for network packets, the Driver yields immediately to a user-space blocked queue. The worker thread picks up the next ready Driver without an OS thread context switch.

---

## 3. Primary Key Storage Internals: Delete Vectors (`DelVec`)

Real-time analytics frequently requires ingesting Change Data Capture (CDC) streams from transactional databases (MySQL, PostgreSQL, Oracle) via Kafka + Flink or Debezium. That means continuous `UPSERT` and `DELETE` operations alongside high-concurrency analytical `SELECT` queries.

Historically, OLAP engines forced an uncomfortable trade-off between two architectures:

- **Merge-on-Read (MoR)** *(used by traditional Unique Key tables and standard Iceberg/Hudi MoR)*: Writes are fast because updates simply append new delta files. However, every `SELECT` query must perform a multi-way sort-merge across base and delta files at read time, and cannot push down predicates on non-key columns cleanly. Query performance degrades by **3–10×** compared to append-only tables.
- **Copy-on-Write (CoW)**: Reads are fast because only one clean version exists, but every small batch of updates rewrites entire multi-hundred-megabyte data files, causing massive write amplification and limiting ingestion freshness to minutes or hours.

### How StarRocks Merge-on-Write (`DelVec`) Solves Both

StarRocks' **Primary Key Table** implements a **Merge-on-Write (Delete Vector)** architecture that keeps read performance identical to an immutable append-only table while supporting second-level CDC upserts.

```
Incoming Rowset (New Batch: PK=102 updated, PK=205 inserted)
   │
   ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. Lookup Primary Key Index (Hash Index in Memory / Disk)   │
│    - PK=102 found in Rowset #4, Segment #1, RowID = 8421    │
│    - PK=205 not found (new key)                             │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Update Delete Vector (Roaring Bitmap) for Rowset #4      │
│    DelVec[Rowset #4, Seg #1].set(8421)                      │
│    Write New Rowset #9 containing new version of PK=102     │
│    and new PK=205                                           │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Analytical SELECT Query                                  │
│    Scans Rowset #4 + Rowset #9 at full vectorized speed     │
│    Applies DelVec bitmap mask (skips RowID 8421 in O(1))    │
│    ► ZERO sort-merge overhead at read time!                 │
└─────────────────────────────────────────────────────────────┘
```

Under the hood, each Tablet (data partition shard) maintains three components:

1. **Immutable Rowsets & Segments**: Data arrives in micro-batches and is written as immutable columnar segments (`.dat` files) with full ZoneMap, BloomFilter, and Bitmap indexes.
2. **Persistent Primary Key Index**: An optimized hash index mapping `PrimaryKey -> (RowsetID, SegmentID, RowID)`. With `enable_persistent_index = true`, the index lives on local SSD/NVMe with a hot memory cache, keeping memory footprint minimal even for billion-row tables.
3. **Delete Vectors (`DelVec`)**: A compressed **Roaring Bitmap** attached to each segment. When a row is updated or deleted in a new transaction, StarRocks looks up the old `(SegmentID, RowID)` in the PK index and sets a single bit in that segment's `DelVec`.

When an analytical query scans the tablet, it does **no key deduplication or sort-merge**. It simply reads the segments using standard vectorized scans and masks out any row whose bit is set in the `DelVec` bitmap.

---

## 4. Lakehouse Engine & Asynchronous Materialized Views

In modern data architectures, petabytes of historical data live in open table formats (**Apache Iceberg**, **Apache Paimon**, **Delta Lake**, **Apache Hudi**) on S3, GCS, or MinIO. Moving all of that data into native OLAP storage is expensive and creates duplicate governance silos.

StarRocks acts as a direct **Lakehouse Query Engine** through External Catalogs:

```sql
-- 1. Register an Apache Iceberg Catalog (REST / Hive / Glue)
CREATE EXTERNAL CATALOG iceberg_lake
PROPERTIES (
    "type" = "iceberg",
    "iceberg.catalog.type" = "rest",
    "iceberg.catalog.uri" = "http://polaris-catalog:8181/api/catalog",
    "aws.s3.endpoint" = "http://minio.storage.svc:9000",
    "aws.s3.enable_path_style_access" = "true"
);

-- 2. Query Iceberg tables directly with zero ETL
SELECT
    date_trunc('hour', event_time) AS hr,
    count(*) AS events,
    sum(revenue) AS total_rev
FROM iceberg_lake.analytics.user_events
WHERE event_time >= '2026-11-01'
GROUP BY 1;
```

### Multi-Tier Data Cache for Remote Parquet/ORC

Querying Parquet files directly over object storage is normally bottlenecked by network throughput and S3 `GET` latency. StarRocks eliminates this gap with a block-level **Multi-Tier Data Cache**:

```
Iceberg / Paimon Parquet Files on Object Storage (S3 / MinIO)
   │
   │  (1 MB Block-Level Read + Split Metadata Cache)
   ▼
┌──────────────────────────────────────────────────────────┐
│ Compute Node (CN / BE) Local Cache Hierarchy             │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Tier 1: RAM Page Cache (Hot Column Pages)          │  │
│  └─────────────────────────┬──────────────────────────┘  │
│                            ▼                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Tier 2: Local NVMe Disk Cache (1 MB Parquet Blocks)│  │
│  │ - SLRU (Segmented LRU) eviction policy             │  │
│  │ - Immune to one-off full-table scan cache pollution│  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

- **Block-Level Caching**: Instead of caching entire Parquet files, StarRocks caches **1 MB byte ranges** corresponding to the exact column chunks and pages requested by queries.
- **Scan-Resistant SLRU Policy**: The cache uses a Segmented LRU (Probationary + Protected segments) eviction policy. If an analyst runs a one-time ad-hoc scan over 5 years of cold Iceberg data, those blocks enter the probationary segment and are evicted without evicting hot dashboard blocks in the protected segment.
- **Consistent Hashing Affinity**: The FE routes scans for a specific Parquet split to the same BE/CN node using consistent hashing, maximizing cache hit rates across queries.

### Asynchronous Materialized Views & Transparent Query Rewrite

For ultra-low-latency dashboards (e.g., $< 100\text{ ms}$ SLA at 1,000 QPS), even local NVMe cache scans over raw fact tables can be expensive if billions of rows must be aggregated on the fly.

StarRocks provides **Asynchronous Materialized Views (Async MV)** that can sit directly on top of native tables, **external Iceberg/Paimon tables**, or even federation joins across both:

```sql
-- Build an incremental Async Materialized View over an external Iceberg table
CREATE MATERIALIZED VIEW mv_hourly_sales_rollup
DISTRIBUTED BY HASH(store_id) BUCKETS 16
PARTITION BY (dt)
REFRESH ASYNC EVERY(INTERVAL 5 MINUTE)
AS
SELECT
    date_trunc('day', o.event_time) AS dt,
    o.store_id,
    s.region,
    count(o.order_id) AS order_cnt,
    sum(o.amount) AS total_amount,
    bitmap_union(to_bitmap(o.user_id)) AS uv_bitmap
FROM iceberg_lake.sales.fact_orders o
JOIN default_catalog.dim.stores s
  ON o.store_id = s.store_id
GROUP BY 1, 2, 3;
```

What makes StarRocks' Async MV uniquely powerful are three capabilities:

1. **Partition-Level Incremental Refresh**: When new partitions or snapshots land in the upstream Iceberg table, StarRocks detects only the modified partitions and refreshes just those slices of the MV.
2. **SPJG Transparent Query Rewrite**: Application code and BI tools (Superset, Tableau, Grafana) **do not need to query `mv_hourly_sales_rollup` directly**. Users continue querying `iceberg_lake.sales.fact_orders`. The CBO's SPJG (Select-Project-Join-GroupBy) rewriter automatically detects that the query can be satisfied by the MV—even if the query requests a higher rollup (e.g., `GROUP BY month, region`) or adds extra `WHERE` predicates—and rewrites the execution plan transparently.
3. **Real-Time + Historical Union Rewrite**: If the MV is refreshed up to 5 minutes ago, and the query asks for data up to `NOW()`, StarRocks can rewrite the plan into a `UNION` that reads historical partitions from the pre-aggregated MV and scans only the last 5 minutes of delta partitions from the base Iceberg/native table.

---

## 5. Architecture Comparison: StarRocks vs. ClickHouse vs. Trino

When designing an analytical data platform, engineers typically evaluate **StarRocks**, **ClickHouse**, and **Trino**. While all three are fast SQL engines, their internal architectures optimize for very different workloads:

| Architectural Dimension | StarRocks | ClickHouse | Trino |
|---|---|---|---|
| **Primary Design Center** | Unified Real-Time OLAP + Lakehouse MPP | Ultra-fast Single-Table Log/Event Analytics | Stateless Federated Lakehouse Query Engine |
| **Cost-Based Optimizer (CBO)** | Full Cascades CBO (join reordering, histograms, MV rewrite) | Limited (historically rule-based; manual join ordering often needed) | Mature CBO (depends on Hive/Iceberg table stats) |
| **Multi-Table Join Execution** | Distributed Shuffle / Broadcast / Bucket / Colocate + Global Runtime Filters | Memory-heavy hash joins; struggles with large-to-large distributed shuffles | Distributed MPP hash joins; high throughput, higher latency floor |
| **Real-Time Upserts / CDC** | Native Primary Key (`DelVec` Merge-on-Write, sub-second reads) | `ReplacingMergeTree` (eventual background merge; `FINAL` penalty at read) | Not a storage engine (relies on Iceberg/Delta compaction) |
| **Lakehouse Querying (Iceberg/Hudi)** | Native C++ vectorized scanner + NVMe block cache + Async MV rewrite | Supported via table functions, secondary to MergeTree | First-class Java connectors across 30+ data sources |
| **High-Concurrency QPS** | 1,000–5,000+ QPS (C++ Pipeline engine + Query Cache + MV) | 1,000+ QPS on single-table queries | Typically 10–100 QPS (JVM coordinator overhead) |
| **Runtime Language** | C++ (BE/CN) + Java (FE) | C++ | Java |

### When to Choose Which

- **Choose ClickHouse** when your workload is $95\%$ append-only observability, telemetry, or clickstream logs stored in a single wide table, and you want maximum compression and raw scan speed per node without complex multi-table joins or strict primary-key upserts.
- **Choose Trino** when your primary goal is ad-hoc federated querying and ETL across dozens of heterogeneous enterprise systems (Iceberg, PostgreSQL, Kafka, Elasticsearch, Snowflake) where query latency of 1–10 seconds at moderate concurrency is acceptable.
- **Choose StarRocks** when you need **both** sub-second high-concurrency BI/user-facing analytics **and** multi-table star/snowflake joins, real-time CDC upserts (`DelVec`), or high-speed querying directly over an Apache Iceberg / Paimon data lakehouse.

---

## Summary

StarRocks achieves its performance profile not through a single trick, but through the tight integration of four layers:

1. **Cascades CBO + Global Runtime Filters** eliminate unnecessary network shuffles and push join selectivity all the way down to storage page indexes.
2. **Vectorized Chunks + Cooperative Pipeline Drivers** saturate modern CPU SIMD units while avoiding OS thread context-switch thrashing under high QPS.
3. **Delete Vector (`DelVec`) Primary Key storage** resolves the classic OLAP upsert dilemma, delivering real-time CDC ingestion with zero Merge-on-Read penalty.
4. **NVMe Data Caching + Async Materialized Views** allow StarRocks to sit directly on top of Apache Iceberg and Paimon, uniting the low cost of an open data lakehouse with the sub-second speed of a dedicated MPP warehouse.
