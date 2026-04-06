# Week 05 Practice: Change Data Capture with Debezium + Kafka + Iceberg

Hands-on practice covering log-based CDC with Debezium, Kafka-based event streaming, and applying change events to a lakehouse using Apache Iceberg's MERGE INTO — building a complete Bronze → Silver pipeline from a live PostgreSQL database.

## Learning outcomes

By the end of this session you should be able to:

- Explain why **log-based CDC** (reading the WAL) is preferred over timestamp-based or trigger-based approaches
- Configure **PostgreSQL** for logical replication (`wal_level = logical`)
- Deploy a **Debezium PostgreSQL connector** via the Kafka Connect REST API
- Observe CDC events flowing into **Kafka topics** (one topic per source table)
- Understand the **Debezium event envelope**: `before`, `after`, `op`, `source`, `ts_ms`
- Consume CDC events from Kafka using **Spark Structured Streaming**
- Build a **Bronze layer** — append-only Iceberg table storing raw CDC events
- Build a **Silver layer** — current-state mirror using **MERGE INTO** with deduplication
- Understand **idempotent processing** — why MERGE INTO is safe to replay
- Use Iceberg **time travel** and **snapshot rollback** to debug and recover from bad data
- Perform **schema evolution** end-to-end: alter source → Debezium detects → Bronze stores → Silver evolves

## Requirements

- Docker Desktop installed and running (4 GB+ RAM recommended for this stack)
- No dataset files needed — all data is generated via SQL against the live PostgreSQL instance

## Quick start

```bash
docker compose up -d
```

Wait ~30 seconds for all services to stabilise, then open:

- **Jupyter:** [http://localhost:8888](http://localhost:8888) — token: `bdm`
- **MinIO Console:** [http://localhost:9001](http://localhost:9001) — login: `admin` / `password`
- **Kafka Connect REST API:** [http://localhost:8083](http://localhost:8083)
- **Spark UI:** [http://localhost:4040](http://localhost:4040) (available once a SparkSession is started)

Navigate to `work/week_05_practice.ipynb` and run the cells in order.

```bash
docker compose down -v   # when done (-v removes volumes for clean restart)
```

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  Docker Compose network                                              │
│                                                                      │
│  ┌──────────────┐         ┌──────────────────┐                       │
│  │  postgres     │  WAL    │  connect         │                       │
│  │  (source DB)  │───────►│  (Debezium on     │                       │
│  │  :5432        │         │   Kafka Connect)  │                       │
│  │  wal_level=   │         │  :8083 REST API   │                       │
│  │  logical      │         └────────┬─────────┘                       │
│  └──────────────┘                  │ produces CDC events              │
│                                    ▼                                  │
│  ┌──────────────────────────────────────────┐                         │
│  │  kafka (KRaft, single node)              │                         │
│  │  :9092                                   │                         │
│  │  Topics: dbserver1.public.customers, ... │                         │
│  └────────────────────┬─────────────────────┘                         │
│                       │ consumed by                                   │
│  ┌────────────────────▼─────────────────────────────────────────┐     │
│  │  jupyter (pyspark-notebook)                                  │     │
│  │  PySpark 4.1 + Iceberg 1.10.1 + Kafka connector             │     │
│  │  • Reads CDC events from Kafka                               │     │
│  │  • Writes Bronze Iceberg table (append-only) on MinIO        │     │
│  │  • Writes Silver Iceberg table (MERGE INTO) on MinIO         │     │
│  │  Notebook: work/week_05_practice.ipynb                       │     │
│  └──────────────────────────────────────────────────────────────┘     │
│                                                                      │
│  ┌──────────────────┐                                                │
│  │  minio           │  S3-compatible object storage                  │
│  │  :9000 (S3 API)  │  Bucket: warehouse/                            │
│  │  :9001 (Console) │  Stores Parquet + Iceberg metadata             │
│  └──────────────────┘                                                │
└──────────────────────────────────────────────────────────────────────┘
         ▲         ▲           ▲            ▲
    :8888     :4040       :9001        :8083
   Jupyter   Spark UI   MinIO Console  Connect API
```

**What's new compared to Seminar 3 and 4?**

| Component | Seminar 3 | Seminar 4 | Seminar 5 (this one) |
|-----------|-----------|-----------|---------------------|
| Kafka | ✓ | — | ✓ (reused) |
| MinIO + Iceberg | — | ✓ | ✓ (reused) |
| PostgreSQL | — | — | ✓ **new** |
| Debezium (Kafka Connect) | — | — | ✓ **new** |

We are combining the Kafka stack from Seminar 3 with the lakehouse stack from Seminar 4, and adding PostgreSQL + Debezium on top.

## Notebook structure

| Part | Topic | What you'll do |
|------|-------|----------------|
| **1** | PostgreSQL + Debezium setup | Connect to PostgreSQL, create source tables, register the Debezium connector via REST API, verify CDC events appear in Kafka |
| **2** | Exploring CDC events | Consume events from Kafka, inspect the Debezium envelope (before/after/op), observe INSERT/UPDATE/DELETE behaviour |
| **3** | Bronze layer | Read CDC stream into an append-only Iceberg table on MinIO — every event preserved |
| **4** | Silver layer (MERGE INTO) | Deduplicate and apply CDC events using MERGE INTO to build a current-state table |
| **5** | Schema evolution | ALTER TABLE at the source, observe Debezium's auto-detection, evolve Iceberg schema |
| **Exercises** | Two self-guided tasks | Snapshot rollback after bad data; add a second source table and build its pipeline |

## Exploring data

After running the notebook, browse MinIO at [http://localhost:9001](http://localhost:9001):

```
warehouse/
└── cdc.db/
    ├── bronze_customers/
    │   ├── metadata/    ← Iceberg snapshots, manifests
    │   └── data/        ← Parquet files (every CDC event)
    └── silver_customers/
        ├── metadata/
        └── data/        ← Parquet files (current state, after MERGE)
```

## Useful commands

```bash
# Check all containers are running
docker compose ps

# View Debezium Connect logs
docker compose logs connect --tail 50

# View PostgreSQL logs
docker compose logs postgres --tail 20

# List Kafka topics (after connector is registered)
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --list

# Consume CDC events from a topic (Ctrl+C to stop)
docker compose exec kafka /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic dbserver1.public.customers \
  --from-beginning

# Connect to PostgreSQL directly
docker compose exec postgres psql -U cdc_user -d sourcedb
```

## Reference

- [Debezium PostgreSQL Connector Documentation](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)
- [Debezium Tutorial](https://debezium.io/documentation/reference/stable/tutorial.html)
- [Kafka Connect REST API](https://kafka.apache.org/documentation/#connect_rest)
- [Apache Iceberg — MERGE INTO](https://iceberg.apache.org/docs/latest/spark-writes/#merge-into)
- [Spark Structured Streaming + Kafka](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- *Designing Data-Intensive Applications*, 2nd Ed. — Ch. 3 (Storage), Ch. 5 (Encoding), Ch. 12 (Stream Processing), Ch. 13 (Data Integration)
