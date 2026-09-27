# Zomato AI Data Engineering Platform

An end-to-end data engineering and AI analytics platform for Zomato-style food delivery data.

The pipeline takes raw food-delivery data from **Amazon S3 → Snowflake → dbt → Airflow**, then adds an AI layer using **OpenAI** for review enrichment, RAG-based review analysis, and natural-language SQL.

## Architecture

```text
                    ┌──────────────────┐
                    │   Source Data    │
                    │   CSV Files      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │   Amazon S3      │
                    │    Data Lake     │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │    Snowflake     │
                    │                  │
                    │ RAW (Bronze)     │
                    │ STAGING (Silver) │
                    │ MARTS (Gold)     │
                    │ AI               │
                    └────────┬─────────┘
                             │
                             │ dbt
                             ▼
                    ┌──────────────────┐
                    │   Transformations│
                    │   + Tests        │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │     Airflow      │
                    │   Orchestration  │
                    └────────┬─────────┘
                             │
                  ┌──────────┴──────────┐
                  ▼                     ▼
          AI Review Enrichment      Analytics / AI
             OpenAI LLM             Streamlit Apps
```

## What the project does

The platform processes Zomato-style data through a modern data warehouse pipeline:

```text
CSV → S3 → Snowflake RAW
             ↓
          dbt STAGING
             ↓
          dbt MARTS
             ↓
      AI Review Enrichment
             ↓
         AI Analytics
```

The pipeline is designed to support **incremental processing**, data quality checks, orchestration, and AI-powered analytics.

## Data Layers

| Layer     | Technology              | Purpose                                          |
| --------- | ----------------------- | ------------------------------------------------ |
| Source    | CSV                     | Raw Zomato-style datasets                        |
| Data Lake | Amazon S3               | Centralized raw file storage                     |
| Bronze    | Snowflake RAW           | Raw data loaded from S3                          |
| Silver    | Snowflake STAGING + dbt | Cleaned and standardized data                    |
| Gold      | Snowflake MARTS + dbt   | Business-ready facts, dimensions, and aggregates |
| AI        | Snowflake AI + OpenAI   | Review enrichment and AI analytics               |

<img width="1113" height="662" alt="Screenshot 2026-08-28 180957" src="https://github.com/user-attachments/assets/535fdc86-e257-4890-ba85-d249461e1a3f" />


## Data Model

### Dimension tables

```text
dim_date
dim_customer
dim_restaurants
dim_food
```

### Fact tables

```text
fct_orders
fct_order_items
```

### Business marts

```text
mart_daily_city_revenue
mart_restaurant_performance
mart_delivery_sla
mart_review_insights
```

The Gold layer follows a **star-schema style design**, with fact tables containing measurable business events and dimensions providing descriptive context.

## Incremental Data Processing

The pipeline is designed so that new data can be processed without rebuilding the entire warehouse.

The processing flow is:

```text
New data arrives
      ↓
Check for new records/files
      ↓
Load new data into RAW
      ↓
dbt incremental MERGE
      ↓
Process only new/changed records
      ↓
AI-enrich only unprocessed reviews
```

For large fact tables such as orders and order items, dbt uses incremental models and `MERGE` logic instead of rebuilding millions of rows on every run.

AI review enrichment is also idempotent: reviews already present in `ZOMATO.AI.REVIEW_ENRICHED` are skipped.

This reduces:

* Snowflake compute
* unnecessary transformations
* duplicate processing
* OpenAI API calls

## Data Quality

dbt tests are used to validate the transformed data.

Examples include:

```text
unique
not_null
relationships
accepted_values
reconciliation checks
```

These tests run as part of the dbt build process and prevent invalid data from silently moving downstream.

## Airflow Orchestration

Apache Airflow runs the pipeline as a dependency-aware DAG.

```text
check_new_data
      ↓
reload_raw
      ↓
dbt_build_core
      ↓
enrich_reviews
      ↓
dbt_build_ai
```

The DAG coordinates:

1. Detection/loading of new data
2. Snowflake RAW ingestion
3. dbt transformations and tests
4. AI review enrichment
5. AI-specific dbt models

Airflow runs inside Docker for a reproducible orchestration environment.

## AI Layer

### 1. LLM Review Enrichment

Reviews are processed using OpenAI to generate structured information such as:

```text
sentiment_label
sentiment_score
topic
key_issue
```

Example:

```text
Review:
"Far too expensive for what you get."

Result:
sentiment_label  → negative
sentiment_score  → -0.7
topic            → pricing
key_issue        → too expensive for the value
```

The enriched records are stored in:

```text
ZOMATO.AI.REVIEW_ENRICHED
```

### 2. RAG — Chat with Reviews

The RAG application:

```text
User Question
      ↓
Generate embedding
      ↓
Find similar reviews
      ↓
Retrieve top-k reviews
      ↓
Send context to LLM
      ↓
Grounded answer
```

This allows users to ask questions such as:

```text
"What are the most common complaints about delivery?"
```

and receive answers based on actual customer reviews.

### 3. Text-to-SQL

The text-to-SQL application allows users to query warehouse data using natural language.

```text
Natural Language Question
          ↓
        OpenAI
          ↓
      SQL Generation
          ↓
    SQL Validation
          ↓
       Snowflake
          ↓
        Result
```

The generated queries are restricted to safe read-only operations before execution.

## Tech Stack

```text
Python
Pandas
NumPy

Amazon S3
Snowflake

dbt
dbt-snowflake

Apache Airflow 3
Docker

OpenAI
GPT-4o-mini
text-embedding-3-small

Streamlit
```

## Repository Structure

```text
zomato-ai-data-analytics/
│
├── airflow/
│   ├── dags/
│   │   └── zomato_batch.py
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── ai/
│   ├── enrich_reviews.py
│   ├── rag_chat.py
│   └── text_to_sql.py
│
├── zomato/
│   ├── dbt_project.yml
│   ├── models/
│   │   ├── staging/
│   │   └── marts/
│   ├── macros/
│   ├── tests/
│   └── snapshots/
│
├── snowflake/
│   ├── 01_setup.sql
│   ├── 02_storage_integration.sql
│   ├── 03_stage_and_formats.sql
│   ├── 04_raw_tables.sql
│   └── 05_copy_into.sql
│
├── aws/
│   └── iam/
│
├── docs/
│   └── architecture.png
│
├── .gitignore
└── README.md
```

## Running the Project

### 1. Snowflake setup

Run the SQL scripts in the `snowflake/` directory in order to create:

* warehouse
* database
* schemas
* storage integration
* external stage
* RAW tables

### 2. dbt

```bash
cd zomato

dbt debug
dbt build --exclude tag:ai
```

### 3. Airflow

```bash
cd airflow

docker compose build
docker compose up -d
```

Open:

```text
http://localhost:8080
```

Trigger the `zomato_batch` DAG.

### 4. AI Applications

From the `ai/` environment:

```bash
python enrich_reviews.py
```

Run the RAG application:

```bash
streamlit run rag_chat.py
```

Run the text-to-SQL application:

```bash
streamlit run text_to_sql.py
```

## Security

Secrets are kept outside the source code using environment variables.

Do not commit:

```text
.env
profiles.yml
credentials
API keys
passwords
CSV datasets
virtual environments
```

The dataset and generated artifacts are intentionally excluded from Git.

## Key Engineering Concepts Demonstrated

```text
Data Lake
Data Warehouse
Medallion Architecture
Dimensional Modeling
Fact & Dimension Tables
Incremental Processing
MERGE Strategy
Data Quality Testing
SCD2 Snapshot
Airflow DAG Orchestration
Dockerized Airflow
LLM-based Data Enrichment
RAG
Text-to-SQL
```

## Result

This project combines **modern data engineering and generative AI** into a single pipeline:

```text
S3
 ↓
Snowflake RAW
 ↓
dbt STAGING
 ↓
dbt MARTS
 ↓
Airflow Orchestration
 ↓
AI Enrichment
 ↓
RAG / Text-to-SQL / Analytics
```

The result is an end-to-end platform capable of ingesting, transforming, validating, incrementally processing, and intelligently analyzing large-scale food-delivery data.

## Future Improvements

* Implement robust new-data detection using file metadata, load timestamps, or control tables before ingestion.
* Improve incremental processing with watermarking and change-data tracking.
* Add automated retries, alerting, and failure notifications in Airflow.
* Scale AI review enrichment with batching, rate-limit handling, and stronger caching.
* Add monitoring for pipeline health, data quality, model performance, and API usage.
* Deploy the platform on AWS with production-grade CI/CD and infrastructure automation.

