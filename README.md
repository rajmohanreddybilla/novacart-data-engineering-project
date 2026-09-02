# Novacart Modern Data Platform

## 📌 Project Overview

Novacart is a growing e-commerce company that generates thousands of transactions through its online platform every day. The main operational data is stored in an Azure SQL Database and includes **orders, products, and payments**.

As the business grew, the analytics team started facing a few problems. Running analytical queries directly on the operational database was becoming inefficient and could affect the performance of the customer-facing application. The business also had limited historical visibility because updated records could overwrite previous values.

To address these challenges, I designed and built a **modern lakehouse data platform using Azure Databricks**.

The platform uses a **Medallion Architecture (Bronze → Silver → Gold)** to move data from the operational database into reliable, analytics-ready datasets.

---

## 🎯 Business Challenges

The existing operational database created several challenges:

* The transactional database was not designed for analytical workloads.
* Running reporting queries could impact the application database.
* Previous values could be overwritten when records were updated.
* Historical analysis was difficult.
* Reporting queries could become slow as data volumes increased.

### Goal

Build a scalable data platform that:

* Separates analytical workloads from the operational database.
* Ingests data incrementally instead of processing everything repeatedly.
* Cleans and validates incoming data.
* Keeps historical versions of important business data.
* Produces analytics-ready datasets.
* Supports dashboards and automated alerts.
* Provides monitoring and audit information for pipeline runs.

---

# 🏗️ Architecture

The solution follows a modern lakehouse architecture:

**Azure SQL → Lakehouse Federation → Bronze → Silver → Gold → Dashboard / Alerts**

The platform is orchestrated using **Databricks Jobs**.

### High-level flow

```text
                    ┌─────────────────────┐
                    │     Azure SQL       │
                    │                     │
                    │ Orders              │
                    │ Products            │
                    │ Payments            │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Lakehouse Federation│
                    │   + Unity Catalog   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       BRONZE        │
                    │                     │
                    │ Raw Delta Tables    │
                    │ Incremental Load    │
                    │ Run Tracking        │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       SILVER        │
                    │                     │
                    │ Cleaning             │
                    │ Validation           │
                    │ Deduplication        │
                    │ Quarantine           │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │        GOLD         │
                    │                     │
                    │ Business-ready data │
                    │ Aggregations         │
                    │ SCD Type 2           │
                    │ Reporting tables     │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
             ┌──────────────┐      ┌──────────────┐
             │  BI Dashboard│      │    Alerts    │
             │              │      │              │
             │ Sales        │      │ Threshold /  │
             │ Payments     │      │ Monitoring   │
             │ Categories   │      └──────────────┘
             └──────────────┘
```

---

# 🔹 Data Source

The source system is an **Azure SQL Database** containing three main tables:

| Table      | Purpose                          |
| ---------- | -------------------------------- |
| `orders`   | Customer order information       |
| `products` | Product and category information |
| `payments` | Payment information              |

Databricks connects to the Azure SQL source using **Lakehouse Federation and Unity Catalog**.

This allows Databricks to securely access the source without requiring the operational database to become the main reporting environment.

---

# 🥉 Bronze Layer

The Bronze layer is responsible for bringing data from the source system into the lakehouse with minimal transformation.

The data is stored as **Delta tables**.

### Bronze tables

```text
novacart_db.bronze_schema
│
├── orders_raw
├── products_raw
├── payments_raw
└── ingestion_control
```

### Incremental ingestion

Instead of loading the entire source table every day, the pipeline keeps track of the last successfully processed record using a **watermark**.

For example:

```text
Day 1
Azure SQL → 500 orders
              ↓
          Bronze = 500
```

The next day:

```text
Day 2
Azure SQL → 50 new orders
              ↓
       Only 50 are processed
              ↓
          Bronze = 550
```

The original 500 records remain in the Bronze Delta table.

This avoids repeatedly processing the same data and makes the pipeline more efficient as the source grows.

### Watermark logic

The pipeline tracks:

* Timestamp column
* Primary key
* Last successful timestamp
* Last successful primary key
* Run ID
* Number of rows written
* Run status
* Updated timestamp

A combination of timestamp and primary key is used to handle records that have the same timestamp.

---

# 🥈 Silver Layer

The Silver layer converts the raw Bronze data into clean and reliable datasets.

### Main processing steps

* Data cleaning
* Data type conversion
* Standardisation
* Deduplication
* Validation
* Business rule checks
* Invalid-record quarantine

### Silver tables

```text
novacart_db.silver_schema
│
├── orders_cleaned
├── orders_transformed
├── orders_quarantine
│
├── products_cleaned
├── products_transformed
├── products_quarantine
│
├── payments_cleaned
├── payments_transformed
├── payments_quarantine
│
└── processing_control
```

### Example data quality checks

For orders:

* Customer ID must exist.
* Product ID must exist.
* Order status must be valid.
* Order amount must be greater than zero.

For products:

* Product name must exist.
* Category must exist.
* Price must be valid and greater than zero.

For payments:

* Order ID must exist.
* Payment status must exist.
* Paid amount must be valid.

Records that fail validation are moved into **quarantine tables** rather than simply being deleted.

This allows the business or data team to investigate data-quality problems later.

---

# 🥇 Gold Layer

The Gold layer contains the business-ready data used for reporting and analytics.

The main Gold dataset combines:

```text
Orders + Products + Payments
```

to create a single analytics-ready view of order information.

### Gold tables

```text
novacart_db.gold_schema
│
├── orders_information
├── orders_information_scd2
├── category_performance
└── processing_control
```

### Orders Information

The `orders_information` table contains information such as:

* Order
* Customer
* Product
* Category
* Order amount
* Payment amount
* Payment status
* Order date
* Payment completion ratio
* Payment state

A payment completion ratio is calculated as:

```text
Paid Amount / Order Amount
```

This can be used to identify:

* Paid orders
* Unpaid orders
* Partially paid orders
* Overpaid orders

---

# 🕒 Historical Tracking with SCD Type 2

One of the business requirements was to keep historical information when important order attributes change.

For this, I implemented **Slowly Changing Dimension Type 2 (SCD Type 2)**.

The table:

```text
orders_information_scd2
```

keeps previous versions of records instead of simply overwriting them.

It uses:

```text
valid_from_ts
valid_to_ts
is_current
```

For example:

```text
Order 200002

Version 1
Status: Pending
is_current: false
valid_to_ts: 2026-09-02 11:31

Version 2
Status: Failed
is_current: true
valid_to_ts: NULL
```

This allows the business to answer questions such as:

> What was the order status before it changed?

and

> When did the order status change?

---

# 📊 Category Performance

The Gold layer also creates category-level business metrics.

The `category_performance` table includes metrics such as:

* Total orders
* Gross Merchandise Value (GMV)
* Total paid amount
* Average payment completion ratio
* Payment failure rate

This provides a simple analytical layer for understanding product-category performance.

---

# 📦 Gold Snapshots

The project also creates snapshots of Gold datasets.

Latest datasets are stored in a Databricks Volume:

```text
gold_snapshots_vol/
│
├── gold_latest/
│   ├── orders_information
│   └── category_performance
│
└── gold_snapshots/
    ├── orders_information/
    └── category_performance/
```

Historical snapshots are stored using run date and run timestamp, allowing previous outputs to be retained.

---

# 🔄 Pipeline Orchestration

The notebooks are orchestrated using **Databricks Jobs**.

The main flow is:

```text
Bronze Notebook
       ↓
Silver Notebook
       ↓
Gold Notebook
       ↓
Dashboard / Reporting
       ↓
Alert
```

The pipeline can be scheduled to run automatically, for example, once per day.

Each layer generates a unique **Run ID**, which helps with:

* Pipeline monitoring
* Troubleshooting
* Auditing
* Identifying which run processed a record

Control tables are also maintained for Bronze, Silver, and Gold processing.

---

# 🚨 Alerts

A Databricks SQL alert is used to monitor a business or data-quality condition.

When a predefined condition or threshold is reached, the alert can notify the relevant team so that the issue can be investigated.

This adds a basic monitoring layer on top of the analytical platform.

---

# 🔧 GitHub Integration

The Databricks project is connected with GitHub so that notebooks and transformation logic can be version controlled.

This provides:

* Version tracking
* Collaboration
* Code history
* Reproducibility
* Safer development

The project logic is therefore not only stored inside Databricks but can also be maintained through a Git-based workflow.

---

# 🛠️ Technologies Used

| Technology                 | Purpose                             |
| -------------------------- | ----------------------------------- |
| **Azure SQL Database**     | Operational source system           |
| **Azure Databricks**       | Data engineering and processing     |
| **Apache Spark / PySpark** | Large-scale data processing         |
| **Delta Lake**             | Reliable lakehouse storage          |
| **Unity Catalog**          | Data governance and access          |
| **Lakehouse Federation**   | Access to Azure SQL from Databricks |
| **Databricks Jobs**        | Pipeline orchestration              |
| **Databricks SQL Alerts**  | Monitoring and notifications        |
| **GitHub**                 | Version control                     |
| **BI Dashboard**           | Business reporting                  |

---

# 📁 Project Structure

```text
Novacart-Modern-Data-Platform/
│
├── README.md
│
├── notebooks/
│   ├── bronze_ingestion
│   ├── silver_transformation
│   └── gold_transformation
│
├── architecture/
│   └── architecture.png
│
├── screenshots/
│   ├── azure_sql.png
│   ├── bronze_pipeline.png
│   ├── silver_pipeline.png
│   ├── gold_pipeline.png
│   ├── databricks_job.png
│   ├── dashboard.png
│   └── alert.png
│
└── documentation/
    └── data_flow.md
```

---

# 🚀 Key Outcomes

This project demonstrates how an operational transactional database can be transformed into a more scalable analytics platform.

The solution provides:

✅ Incremental data ingestion
✅ Bronze, Silver and Gold architecture
✅ Delta Lake tables
✅ Data cleaning and validation
✅ Data-quality quarantine
✅ Historical tracking using SCD Type 2
✅ Business-ready aggregations
✅ Pipeline audit/control tables
✅ Databricks Job orchestration
✅ Dashboard-ready datasets
✅ Automated alerting
✅ GitHub version control

---

# 💡 What I Learned

Through this project, I gained practical experience in designing a data engineering pipeline rather than simply transforming a dataset.

The main concepts I worked with were **incremental ingestion, watermarking, Delta Lake, PySpark transformations, data quality, SCD Type 2, orchestration, monitoring and analytics-ready data modelling**.

The project also helped me understand how different layers of a modern data platform work together to separate **raw data ingestion, data quality processing and business analytics**.

---

## 👤 Author

**Raj Mohan Reddy Billa**

MSc Data Science | Graduate Data Engineer

Interested in Data Engineering, Cloud Data Platforms, Databricks, Microsoft Fabric, PySpark and SQL.

