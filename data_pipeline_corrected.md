# Data Pipeline Architecture, Validation, and Storage Design

---

## Task 1 — Pipeline Architecture Diagram

### 1.1 End-to-End Architecture

```
        BATCH SOURCE                      STREAMING SOURCE
┌─────────────────────┐          ┌─────────────────────────┐
│  Historical Dataset │          │  Live Transaction Stream │
│  Online Retail .xlsx│          │  (API / Kafka topic)     │
└──────────┬──────────┘          └────────────┬────────────┘
           │                                  │
           ▼                                  ▼
┌─────────────────────┐          ┌─────────────────────────┐
│   Batch Ingestion   │          │  Streaming Ingestion     │
│   Scheduled daily   │          │  Row-by-row consumer     │
└──────────┬──────────┘          └────────────┬────────────┘
           │                                  │
           └─────────────┬────────────────────┘
                         │
            ┌────────────▼────────────┐
            │       LANDING ZONE      │
            │   Raw Storage (Parquet) │
            │   Partitioned by date   │
            └────────────┬────────────┘
                         │
                ┌────────▼────────┐
                │   Validation    │
                │ Schema + Rules  │
                └───────┬─────────┘
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
 ┌──────────────────┐   ┌──────────────────────┐
 │  Valid Records   │   │  Quarantine / DLQ    │
 │  Continue fwd    │   │  Failed + metadata   │
 └────────┬─────────┘   └──────────┬───────────┘
          │                        │
          ▼                        ▼
 ┌──────────────────┐   ┌──────────────────────┐
 │  Transformation  │   │  Error Monitoring    │
 │  Clean + Enrich  │   │  Alerts (Grafana /   │
 │  (Spark)         │   │  Airflow / PagerDuty)│
 └────────┬─────────┘   └──────────────────────┘
          │
  ┌───────▼────────┐
  │  Clean Layer   │
  │  Parquet tables│
  └───────┬────────┘
          │
  ┌───────▼────────┐
  │  Feature Layer │
  │  ML features   │
  └───────┬────────┘
          │
  ┌───────▼──────────┐
  │ Consumer Systems │
  │ BI Dashboard     │
  │ ML Model         │
  └──────────────────┘
```

---

### 1.2 Component Descriptions

**Data Sources**
Two sources feed the pipeline. The first is the historical Online Retail dataset (Excel file) with invoice numbers, product IDs, quantities, and prices. The second is a live transaction stream delivering new sales events via Apache Kafka.

**Ingestion Layer**
Batch ingestion runs daily via Apache Airflow, reads the Excel file, converts it to Parquet, and writes it to the landing zone. Streaming ingestion consumes Kafka events row-by-row using a Spark Structured Streaming job. Both paths write raw data without any modification.

**Landing Zone (Raw Storage)**
Stores data exactly as it arrives in Parquet format, partitioned by ingestion date (e.g. `year=2024/month=01/day=15`). Hosted on S3 or Azure Data Lake. Preserves the original data for reprocessing and debugging.

**Validation Stage**
Implemented using Great Expectations or a custom PySpark job. Checks data types, null constraints, value ranges, and cross-field logic. Valid records move forward; invalid records are routed to quarantine. Validation metrics (pass rate, failure counts) are logged.

**Quarantine / Dead Letter Area**
Stores rejected records in a dedicated Parquet partition alongside error metadata — the failed rule name, field value, and timestamp. Operators review the quarantine table via a BI dashboard. After fixing the upstream issue, records are reprocessed by re-running the validation job against the quarantine partition.

**Transformation Stage**
Apache Spark job that cleans and enriches validated data. Operations include deduplication, whitespace trimming, country name normalisation, line total calculation, and date component extraction. Also produces customer aggregations and ML-ready features.

**Storage Layers**
Three layers: raw (original data, indefinite retention), clean (standardised transactions for BI), and feature (aggregated customer metrics for ML). All use Parquet on cloud object storage with date partitioning.

**Consumer Interfaces**
BI dashboards (Tableau, Power BI, Metabase) query the clean layer via Athena or Databricks SQL. ML pipelines read the feature layer to train predictive models, optionally served through a feature store like Feast.

**Monitoring and Alerts**
Airflow tracks orchestration health. Great Expectations enforces data quality checks. Grafana or Datadog tracks key metrics: ingestion rate, validation pass/fail counts per rule, null rates per field, processing latency, and quarantine partition size. Threshold breaches trigger PagerDuty or email alerts.

---

## Task 2 — Validation and Error Handling Design

### 2.1 Validation Rules

**Schema Validations**
- InvoiceNo must be a non-empty string
- StockCode must be a non-empty string
- Description must be a string — nullable (null values accepted, record is not rejected)
- Quantity must be an integer
- InvoiceDate must be a valid parseable timestamp
- UnitPrice must be numeric
- CustomerID must be numeric when present — nullable (records without it are accepted but excluded from customer analytics)
- Country must be a non-empty string

**Value Range Validations**
- Quantity must not be zero
- UnitPrice must be greater than zero for non-cancellation records
- Quantity must not exceed 100,000
- UnitPrice must not exceed 50,000

**Business Rule Validations**
- If InvoiceNo starts with "C", the record is a cancellation and Quantity must be negative
- InvoiceDate cannot be in the future relative to the pipeline run timestamp
- CustomerID, when present, should match a known customer in the customer dimension table

---

### 2.2 Error Handling Flow

**Schema Validation Failures**
The record is rejected entirely and written to the quarantine partition with the rule name, offending field, and raw value. If the schema failure rate exceeds 1% of the batch, an alert fires. Recovery: after the source is corrected, the quarantine partition is resubmitted to the validation job via a reprocessing Airflow DAG.

**Value Range Failures**
Records are written to quarantine for review. Minor cases (e.g. UnitPrice = 0 on a clearly valid transaction) may be auto-corrected by setting the value to null and flagging it as imputed, if approved by the business. Alerts fire when daily range failure counts exceed a defined threshold. Recovery: operators review, apply corrections, and re-run the reprocessing DAG.

**Business Rule Failures**
Records go to the dead letter queue — a separate quarantine partition for records that cannot be auto-corrected. Operators are alerted immediately, investigate the root cause, and resubmit corrected records through the pipeline.

**Partial Acceptance**
Records with a null Description or null CustomerID are not rejected. They pass into the clean layer with the null value preserved. Null CustomerID records are excluded from customer aggregations and feature engineering but remain in the transaction-level dataset.

---

## Task 3 — Transformation and Storage Design

### 3.1 Transformations

**Cleaning Operations**
Input: validated transaction records. Output: standardised dataset.

- Trim whitespace from text fields (Description, Country, StockCode). *Idempotent: yes — trimming the same string always produces the same result.*
- Normalise Country names via a lookup table (e.g. "EIRE" → "Ireland"). *Idempotent: yes — applying the same map repeatedly gives the same output.*
- Remove exact duplicate rows (same InvoiceNo, StockCode, Quantity, UnitPrice, InvoiceDate). *Idempotent: yes — the clean layer partition is overwritten (INSERT OVERWRITE), not appended.*

**Derived Columns**
Input: cleaned transaction data. Output: enriched dataset.

- LineTotal = Quantity × UnitPrice. *Idempotent: yes — deterministic arithmetic.*
- CancellationFlag = TRUE if InvoiceNo starts with "C". *Idempotent: yes.*
- OrderYear, OrderMonth, OrderDay extracted from InvoiceDate. *Idempotent: yes — date extraction from a fixed timestamp is stable.*

**Customer Aggregations**
Input: clean transaction data. Output: customer-level metrics.

- Total revenue per customer (SUM of LineTotal). *Idempotent: yes — result partition is overwritten on each run.*
- Number of distinct orders (COUNT DISTINCT InvoiceNo). *Idempotent: yes.*
- Unique products purchased (COUNT DISTINCT StockCode). *Idempotent: yes.*
- Recency: days since last purchase. *Idempotent: yes — computed relative to a fixed observation window end date, not the live system clock.*

**Feature Engineering for ML**
Input: clean transactions within the observation window (default: last 90 days). Output: model-ready features.

- Total spending in the last 90 days. *Idempotent: yes — window bounds are fixed parameters.*
- Purchase frequency (orders per week). *Idempotent: yes.*
- Average order value. *Idempotent: yes.*
- Product diversity score (unique StockCodes / total line items). *Idempotent: yes.*

> All feature jobs overwrite the date-stamped partition on re-run, ensuring idempotency regardless of how many times the job executes for a given date.

---

### 3.2 Storage Layers

| Layer   | Contents                              | Format  | Update Frequency | Retention        |
|---------|---------------------------------------|---------|------------------|------------------|
| Raw     | Original ingested data (batch+stream) | Parquet | Continuous/daily | Indefinite       |
| Clean   | Validated, transformed transactions   | Parquet | Daily            | Long-term (3yr+) |
| Feature | Aggregated customer ML features       | Parquet | Daily/weekly     | Medium (1 year)  |

**Why Parquet?**
Parquet is columnar, which means query engines only read the columns needed for the query. This improves performance when working with large datasets.It compresses efficiently (Snappy/Zstd) and is natively supported by Spark, Athena, Databricks, and Pandas. The clean layer is partitioned by date so BI tools prune irrelevant partitions. The feature layer stores one file per extraction date, enabling time-travel by reading historical snapshots.

---

### 3.3 Incremental Updates

**Tracking Processed Data**
A high-water mark is stored in a metadata table (e.g. a small DynamoDB table or Parquet metadata file). After each successful run the pipeline records the latest ingestion timestamp processed. On the next run only records newer than the high-water mark are loaded from the landing zone.

**Late-Arriving Records**
Late records are still accepted. The pipeline detects the correct date partition and writes the record there using "INSERT OVERWRITE". Downstream aggregations for the affected date range are flagged and recomputed on the next feature refresh.

**Feature Refresh**
Customer features are refreshed daily. The feature job reads all clean-layer transactions within the observation window and overwrites the feature layer partition for that date. This ensures late-arriving records and corrections are always reflected in the next feature snapshot. ML models are retrained after each refresh.
