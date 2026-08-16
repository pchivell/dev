# Domain: Data Platform

Applies to: data warehouses, data lakes, lakehouses, ETL/ELT pipelines,
analytics systems, ML feature stores, and any architecture where moving,
transforming, and storing data at scale is the primary concern.

---

## Domain Characteristics

- **Volume**: gigabytes to petabytes; row counts in millions to trillions
- **Variety**: structured (SQL), semi-structured (JSON/XML), unstructured (text/images)
- **Velocity**: batch (hourly/daily) vs streaming (sub-second) vs micro-batch (minutes)
- **Consumers**: BI dashboards, data scientists, ML models, operational APIs
- **Correctness**: late-arriving data, duplicates, and schema drift must be handled explicitly
- **Cost**: storage is cheap; compute is expensive; minimise scans and transformations

---

## Architecture Patterns

### Decide: Batch vs Streaming vs Lambda

| Pattern | Latency | Complexity | When |
|---------|---------|-----------|------|
| **Batch (ETL/ELT)** | Hours–days | Low | Reporting, daily aggregates, historical analysis |
| **Micro-batch** | Minutes | Medium | Near-real-time dashboards; Spark Structured Streaming |
| **Streaming** | Seconds | High | Fraud detection, live leaderboards, alerting |
| **Lambda Architecture** | Both | High | Serve both historical and live; dual complexity |
| **Kappa Architecture** | Streaming | Medium | Reprocess everything from event log; simpler than Lambda |

**Default: ELT batch pipeline** unless sub-minute latency is a stated requirement.

### Modern Data Stack (ELT)

```
Source Systems  →  Ingestion  →  Raw Layer  →  Transform  →  Serving Layer  →  Consumers
(DB, APIs, files)  (Fivetran/  (S3/GCS/    (dbt)         (DW / lakehouse)  (BI, ML, APIs)
                    Airbyte)    ADLS)
```

**Layers:**
- **Raw / Bronze**: exact copy of source data; immutable; never deleted; schema-on-read
- **Staged / Silver**: cleaned, typed, deduplicated; business-key resolved
- **Curated / Gold**: business-level aggregates and metrics; optimised for consumption

### Data Lakehouse (default for new greenfield)

```
Storage layer:   object store (S3/GCS) with open table format (Delta Lake / Iceberg / Hudi)
Compute layer:   Spark, DuckDB, Trino, or cloud-native (BigQuery, Snowflake, Redshift)
Catalog layer:   Unity Catalog / AWS Glue / Hive Metastore
```

Open table formats give ACID transactions, time travel, and schema evolution on cheap object storage.

---

## Technology Selection Framework

| Decision | Cloud (AWS) | Cloud (GCP) | On-Premise / Hybrid |
|----------|------------|------------|---------------------|
| Object store | S3 | GCS | MinIO |
| Data warehouse | Redshift / Athena | BigQuery | Trino + Iceberg |
| Stream processing | Kinesis / MSK (Kafka) | Pub/Sub + Dataflow | Apache Kafka + Flink |
| Transformation | **dbt** (all) | **dbt** | **dbt** |
| Orchestration | Airflow (MWAA) | Cloud Composer | **Apache Airflow** / Dagster |
| Ingestion | Fivetran / Airbyte | Fivetran / Airbyte | **Airbyte** (self-hosted) |
| BI layer | QuickSight / Superset | Looker | **Metabase** / Apache Superset |

**dbt is the default transformation tool** regardless of cloud — it provides SQL-based transforms,
lineage, testing, and documentation in one package.

---

## Data Quality Framework

Every pipeline must enforce:

1. **Schema contracts** — source schemas versioned; consumers notified of breaking changes
2. **Row count checks** — alert if daily load is 0 or > 3× yesterday
3. **Null checks** — not-null constraints on business keys
4. **Uniqueness checks** — primary key uniqueness per load
5. **Referential integrity** — FK relationships between tables
6. **Freshness** — alert if a table hasn't been updated within its SLA window

dbt tests cover items 2–5 natively. Integrate with **Great Expectations** or **Soda** for richer checks.

---

## Key Decision Checkpoints

1. **Batch vs streaming?** Only go streaming if the business requires sub-minute latency; streaming adds 5× complexity.
2. **Build vs buy ingestion?** Fivetran/Airbyte for standard sources; custom connectors for proprietary APIs.
3. **Single warehouse or lakehouse?** Lakehouse if ML workloads need raw data; warehouse if pure BI.
4. **Data mesh?** Domain ownership of data products — justified only for large orgs (50+ data engineers) with clear domain boundaries.
5. **PII / sensitive data?** Column-level encryption and masking from day one; retrofitting is expensive.
6. **Cost governance?** Tag every compute job with team/project; set budget alerts; partition tables by date to limit scan cost.

---

## Common Pitfalls

- **No raw layer** — transforming data in-place; no ability to reprocess; no audit of source data.
- **No data lineage** — unknown where a metric comes from; trust collapses.
- **Schema drift ignored** — source adds a column; pipeline silently drops it or breaks.
- **No SLA on freshness** — consumers don't know if data is stale; silent staleness is worse than an alert.
- **SELECT * in production transforms** — always project explicit columns; schema changes break pipelines.
- **No partitioning** — full table scans on billion-row tables; expensive and slow.
- **One giant DAG** — monolithic pipeline fails together; decompose into focused, independently schedulable DAGs.
