# Week 02 Practice: Spark Batch Processing with DataFrames

One notebook covering the PySpark DataFrame API end-to-end.
Instructions and explanations are inline in the notebook — no need to read anything else first.

## Learning outcomes

By the end of this session you should be able to:

- Define custom schemas using `StructType` and avoid costly schema inference
- Create DataFrames from files (CSV, JSON, Parquet), RDDs, SQL, and Row objects
- Transform columns (`withColumn`, `when`, `selectExpr`) and filter/sort rows
- Work with nested types: structs, arrays (`explode`, `element_at`), and maps (`getItem`)
- Aggregate data with `groupBy`, `agg`, and `collect_set`
- Reshape data into wide format with pivot tables
- Apply SQL window functions: ranking, lead/lag, rolling aggregations
- Join DataFrames and use broadcast hints to avoid shuffles
- Write and register UDFs with explicit return types

## Requirements

- Docker Desktop installed and running
- Dataset files placed in `work/data/`. Datasets can be downloaded from the link on Moodle

## Datasets

The notebook expects these files under `work/data/`:

| File | Used in |
|------|---------|
| `events-500k.json` | sections 1–12, exercises 1–2 |
| `sales.parquet` | sections 8, 12 |
| `users.parquet` | section 12 |
| `Chicago-Crimes-2018.csv` | section 1 |
| `countries.csv` | section 13 |
| `amsterdam-listings-2018-12-06.parquet` | section 10 (pivot) |
| `pageviews_by_second.parquet` | section 10 (pivot) |
| `health_profile_data.snappy.parquet` | section 11 (window functions) |
| `employees.csv` | exercise 3 |
| `zips.json` | exercise 2 |

## Quick start

```bash
docker compose up -d
```

Then open:

- Jupyter: http://localhost:8888 — token: **bdm**
- Spark UI: http://localhost:4040 _(available once a SparkSession is started)_

Navigate to `work/week_02_practice.ipynb` and run the cells.

```bash
docker compose down   # when done
```

## Reference

- [PySpark SQL Functions](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html)
- [PySpark Window](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/window.html)
- [Spark SQL Built-in Functions](https://spark.apache.org/docs/latest/api/sql/index.html)
