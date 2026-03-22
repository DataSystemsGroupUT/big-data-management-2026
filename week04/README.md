# Week 04 Practice: Lakehouse Design with Spark + Iceberg + MinIO

Hands-on practice covering column storage, object storage, encoding formats, schema evolution, and the medallion architecture — all on a local lakehouse built with Apache Iceberg and S3-compatible object storage.

## Learning outcomes

By the end of this session you should be able to:

- Explain why **column-oriented storage** (Parquet) outperforms row-oriented formats (CSV/JSON) for analytics
- Understand why **object storage** (S3/MinIO) is the foundation of lakehouse architectures
- Browse Parquet files and Iceberg metadata in **MinIO Console** (simulated S3)
- Create and query **Apache Iceberg** tables stored on object storage
- Understand how Iceberg provides **ACID transactions**, **snapshot isolation**, and **time travel**
- Perform **schema evolution** — add columns, rename columns — without rewriting data
- Build a **medallion architecture** (bronze → silver → gold) on Iceberg tables
- Inspect Iceberg **metadata**: snapshots, manifests, data files, and partition stats

## Requirements

- Docker Desktop installed and running
- No dataset files needed — all data is generated inside the notebook

## Quick start

```bash
docker compose up -d
```

Then open:

- **Jupyter:** [http://localhost:8888](http://localhost:8888) — token: `bdm`
- **MinIO Console:** [http://localhost:9001](http://localhost:9001) — login: `admin` / `password`
- **Spark UI:** [http://localhost:4040](http://localhost:4040) (available once a SparkSession is started)

Navigate to `work/week_04_practice.ipynb` and run the cells in order.

```bash
docker compose down   # when done
```

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Docker Compose network                                  │
│                                                          │
│  ┌──────────────────┐                                    │
│  │  minio           │  S3-compatible object storage      │
│  │  (S3 API :9000)  │  Bucket: warehouse/                │
│  │  (Console :9001) │  Stores Parquet files + metadata   │
│  └────────┬─────────┘                                    │
│           │  s3a://warehouse/...                         │
│  ┌────────▼─────────────────────────────────────────┐    │
│  │  jupyter (pyspark-notebook)                      │    │
│  │                                                  │    │
│  │  PySpark 4.1 + Iceberg 1.10.1 + AWS bundle       │    │
│  │  • Iceberg Hadoop catalog → s3a://warehouse      │    │
│  │  • Reads/writes Parquet on MinIO (not local disk)│    │
│  │                                                  │    │
│  │  Notebook: work/week_04_practice.ipynb           │    │
│  └──────────────────────────────────────────────────┘    │
│         ▲              ▲                                 │
│    localhost:8888  localhost:4040                        │
│    (Jupyter UI)    (Spark UI)                            │
└──────────────────────────────────────────────────────────┘
         ▲
    localhost:9001
    (MinIO Console — browse your S3 buckets!)
```

**Why MinIO?** In production, lakehouse data lives on cloud object storage (S3, GCS, Azure Blob). MinIO simulates this locally so you can see the separation of storage and compute — Spark reads/writes Parquet files to MinIO over the S3 API, just as it would in the cloud. You can browse the raw files in the MinIO Console at [localhost:9001](http://localhost:9001).

## Notebook structure

| Part | Topic | What you'll do |
|------|-------|----------------|
| **1** | Storage formats | Generate synthetic data, write as CSV / JSON / Parquet to MinIO, compare file sizes and query speed, browse files in MinIO Console |
| **2** | Apache Iceberg basics | Create Iceberg tables on S3 (MinIO), insert data, inspect snapshots, time travel |
| **3** | Schema evolution | Add columns, rename columns, widen types — Iceberg handles it without rewriting files |
| **4** | Medallion architecture | Build bronze (raw) → silver (cleaned) → gold (aggregated) Iceberg tables on MinIO |
| **Exercises** | Two self-guided tasks | Partitioned table + snapshot rollback; compression comparison |

## Exploring data in MinIO Console

After running Part 1, open [http://localhost:9001](http://localhost:9001) and navigate to the `warehouse` bucket. You'll see:

```
warehouse/
├── format_comparison/
│   ├── csv/           ← row-oriented, large files
│   ├── json/          ← row-oriented, even larger
│   └── parquet/       ← columnar, much smaller!
└── practice.db/
    ├── sales/
    │   ├── metadata/  ← Iceberg metadata (snapshots, manifests)
    │   └── data/      ← Parquet data files
    ├── bronze_sensors/
    ├── silver_sensors/
    └── gold_hourly_sensors/
```

This is exactly what an S3 bucket looks like in a production lakehouse.

## Reference

- [Apache Iceberg — Spark Quickstart](https://iceberg.apache.org/spark-quickstart/)
- [Iceberg Spark DDL reference](https://iceberg.apache.org/docs/latest/spark-ddl/)
- [Iceberg Schema Evolution](https://iceberg.apache.org/docs/latest/evolution/)
- [MinIO Documentation](https://min.io/docs/minio/container/index.html)
- [Armbrust et al. (2021) — Lakehouse: A New Generation of Open Platforms](https://www.cidrdb.org/cidr2021/papers/cidr2021_paper17.pdf)
- *Designing Data-Intensive Applications*, 2nd Ed. — Ch. 4 (Storage) & Ch. 5 (Encoding)
