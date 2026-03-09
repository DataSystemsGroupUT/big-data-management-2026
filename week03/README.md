# Week 03 Practice: Spark Structured Streaming + Kafka

One notebook covering Kafka infrastructure and Spark Structured Streaming end-to-end toy example.
Instructions and explanations are inline in the notebook.

## Learning outcomes

By the end of this session you should be able to:

- Start a Kafka broker (KRaft mode, no ZooKeeper) with Docker Compose
- Create and inspect topics, partitions, and offsets using CLI tools
- Understand why ordering holds within a partition but not across partitions
- Read consumer-group lag from `kafka-consumer-groups.sh --describe`
- Connect Spark Structured Streaming to Kafka with `readStream.format("kafka")`
- Interpret the raw Kafka DataFrame schema (`key`, `value`, `topic`, `partition`, `offset`, `timestamp`)
- Parse a JSON payload from the `value` column
- Write a streaming query to the `memory` sink and inspect results interactively
- Apply tumbling windows on event time
- Observe that late events update old windows when there is no watermark
- Add a watermark and see old windows dropped once the threshold is exceeded
- Compare `append`, `complete`, and `update` output modes
- Write to a persistent file sink with a checkpoint and verify that restarting the job does not produce duplicates
- Enrich a stream with a static lookup table using a streaming-static join

## Requirements

- Docker Desktop installed and running
- No dataset files needed — all data is produced inside the notebook

## Quick start

```bash
docker compose up -d
```

Then open:

- Jupyter: http://localhost:8888 — token: **bdm**
- Spark UI: http://localhost:4040 _(available once a SparkSession is started)_

Navigate to `work/week_03_practice.ipynb` and run the cells **in order**.

```bash
docker compose down   # when done
```

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Docker Compose network                         │
│                                                 │
│  ┌──────────────────┐    kafka:9092             │
│  │  kafka           │◄────────────────────────┐ │
│  │  KRaft broker    │    KRaft, single node   │ │
│  │                  │                         │ │
│  └──────────────────┘                         │ │
│                                               │ │
│  ┌────────────────────────────────────────┐   │ │
│  │  jupyter (pyspark-notebook)            │───┘ │
│  │  • kafka-python-ng  (CLI-like ops)     │     │
│  │  • PySpark + spark-sql-kafka connector │     │
│  └────────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘
         ▲                   ▲
    localhost:8888       localhost:4040
    (Jupyter UI)         (Spark UI)
```

## Kafka CLI reference

All CLI commands run **inside the `kafka` container** via `docker exec` from a **host terminal**.
The `apache/kafka` image keeps scripts at `/opt/kafka/bin/` — use the full path:

| What | Command |
|------|---------|
| Create topic | `docker exec kafka sh -c "/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic sensor-events --partitions 3 --replication-factor 1`" |
| List topics | `docker exec kafka sh -c "/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list`" |
| Describe topic | `docker exec kafka sh -c "/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic sensor-events`" |
| Console producer | `docker exec -it kafka sh -c "/opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic sensor-events`" |
| Consume from start | `docker exec kafka sh -c "/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic sensor-events --from-beginning`" |
| Keyed producer | `docker exec -it kafka sh -c "/opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic sensor-events --property parse.key=true --property key.separator=:`" |
| Consumer group lag | `docker exec kafka sh -c "/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group <group-id>`" |

## Reference

- [Structured Streaming + Kafka Integration](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- [Structured Streaming Programming Guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- [PySpark Streaming API](https://spark.apache.org/docs/latest/api/python/reference/pyspark.ss/index.html)
- [Kafka Documentation](https://kafka.apache.org/documentation/)
