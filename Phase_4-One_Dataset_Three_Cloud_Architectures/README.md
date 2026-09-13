# Phase 4 — One Dataset, Three Cloud Architectures

Avoid artificial `Hello World` projects.

Create one synthetic business dataset containing:

* Customers
* Accounts
* Products
* Orders
* Transactions
* Events

Generate multiple ingestion patterns:

* Batch files
* APIs
* CDC-like events
* Streaming events

Then implement the same business requirements independently in each cloud.

## AWS

```text
Events / Files
      ↓
Kinesis / MSK / S3
      ↓
     Glue
      ↓
   Redshift
      ↓
     dbt
```

## Azure

```text
Events / Files
      ↓
Event Hubs / ADLS
      ↓
  Databricks
      ↓
 Delta Lake
      ↓
     dbt
```

## GCP

```text
Events / Files
      ↓
 Pub/Sub / GCS
      ↓
   Dataflow
      ↓
   BigQuery
      ↓
     dbt
```

The objective is to demonstrate both:

* **Portable Data Engineering principles**
* **Legitimate cloud architectural differences**

Portable components should include:

```text
Python
SQL
dbt
PySpark
pytest
Rust
Docker
Git
Terraform
```

Cloud-specific storage, messaging, security, monitoring, and infrastructure implementations remain provider-specific.

---