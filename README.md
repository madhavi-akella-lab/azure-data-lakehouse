# Azure Data Lakehouse — Retail Analytics

**Stack:** Azure Databricks · ADLS Gen2 · Azure Data Factory · PySpark · Delta Lake  
**Pattern:** Medallion Architecture (Bronze → Silver → Gold)  
**Type:** Cloud Data Engineering · Freelance Engagement

---

A retail client needed a single unified analytics platform across fragmented source systems. Data was scattered across flat files, relational databases, and semi-structured JSON feeds — inconsistent formats, no governance, no single source of truth.

This project delivered a production-grade Azure Lakehouse that consolidated everything.

---

## The Problem

| Before | After |
|--------|--------|
| Data siloed across 4+ source systems | Single unified Lakehouse layer |
| No consistent data quality standards | Bronze/Silver/Gold quality enforcement |
| Manual reporting taking hours | Analytics-ready Gold datasets on demand |
| No scalability for growing data volume | Auto-scaling Databricks clusters |
| Legacy batch jobs, no lineage | ADF-orchestrated pipelines with monitoring |

---

## How It Was Built

### Ingestion — Bronze Layer
Raw data lands exactly as received from source systems. No transformations. Full history preserved. ADF pipelines handle scheduling, retries, and failure alerting.

```python
# Example: Reading mixed retail data into Bronze
df_raw = (spark.read
    .option("inferSchema", "true")
    .option("multiLine", "true")
    .json(f"{adls_path}/raw/transactions/")
)
df_raw.write.format("delta").mode("append").save(bronze_path)
```

### Transformation — Silver Layer
Business rules applied. Nulls handled. Duplicates removed. Schema enforced. This is where raw data becomes trusted data.

```python
# Example: Silver transformation with quality rules
df_silver = (df_bronze
    .dropDuplicates(["transaction_id"])
    .filter(col("amount").isNotNull() & (col("amount") > 0))
    .withColumn("transaction_date", to_date(col("transaction_ts")))
    .withColumn("ingestion_ts", current_timestamp())
)
df_silver.write.format("delta").mode("overwrite").partitionBy("transaction_date").save(silver_path)
```

### Aggregation — Gold Layer
Analytics-ready. Pre-aggregated for BI consumption. Partitioned for fast query performance. This is what the dashboards and reports read from.

```python
# Example: Gold aggregation for daily sales reporting
df_gold = (df_silver
    .groupBy("store_id", "transaction_date", "product_category")
    .agg(
        sum("amount").alias("daily_revenue"),
        count("transaction_id").alias("transaction_count"),
        avg("amount").alias("avg_transaction_value")
    )
)
df_gold.write.format("delta").mode("overwrite").save(gold_path)
```

---

## Performance Tuning Applied

**Partitioning** — Silver and Gold tables partitioned by date. Eliminates full table scans for time-range queries.

**Broadcast joins** — Small dimension tables (products, stores) broadcast to avoid shuffle. Significant runtime reduction on join-heavy aggregations.

**Caching** — Frequently referenced Silver datasets cached in memory during multi-step Gold transformations.

**Z-ordering** — Applied on high-cardinality filter columns (product_id, store_id) in Gold layer to improve query skip rates.

---

## Pipeline Orchestration — ADF

Each layer runs as an independent ADF pipeline with:
- Dependency chains between Bronze → Silver → Gold
- Parameterized for incremental vs full loads
- Email alerts on failure with error details
- Run history and monitoring via ADF Monitor

---

## What I Would Add at Larger Scale

- **Data quality framework** — Great Expectations or Databricks built-in quality checks at Silver layer
- **Data catalog** — Unity Catalog for column-level lineage and access control
- **Streaming layer** — Delta Live Tables for real-time Bronze ingestion alongside batch
- **CI/CD** — GitHub Actions deploying notebook changes through dev/staging/prod

---

*Built by [Madhavi Akella](https://madhavi-akella.netlify.app) · Data & AI Engineer · Cincinnati, OH*
