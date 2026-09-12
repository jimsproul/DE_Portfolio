# Phase 3 — Move From Local POC to Cloud

Once the local environment reaches proof-of-concept maturity, the laptop becomes the **development workstation and control plane**.

The portable engineering core remains local:

```text
Git
Python
Rust
Docker
dbt
PySpark
Airflow
Jenkins
Kubernetes
Terraform
```

Cloud CLIs, SDKs, APIs, and Terraform providers connect the local environment to cloud-managed infrastructure.

The progression becomes:

```text
Local
  ↓
Containerized
  ↓
Kubernetes
  ↓
Cloud-Native
  ↓
Multi-Cloud
```

---

# AWS Data Engineering Track

AWS becomes the broad cloud Data Engineering implementation.

### Target Services

| Function       | AWS Service     |
| -------------- | --------------- |
| Object Storage | S3              |
| ETL            | Glue            |
| Spark          | Glue / EMR      |
| Data Warehouse | Redshift        |
| Orchestration  | MWAA            |
| Streaming      | MSK / Kinesis   |
| Containers     | EKS             |
| Secrets        | Secrets Manager |
| Monitoring     | CloudWatch      |
| Infrastructure | Terraform       |

Example architecture:

```text
API / Files / Events
        ↓
       S3
        ↓
      Glue
        ↓
    Redshift
        ↓
       dbt
        ↓
Analytics Mart
```

---

# Azure Data Engineering Track

Azure becomes the primary **Databricks and Lakehouse implementation**.

### Target Services

| Function       | Azure Service      |
| -------------- | ------------------ |
| Object Storage | ADLS Gen2          |
| Ingestion      | Data Factory       |
| Lakehouse      | Azure Databricks   |
| Processing     | Spark / Databricks |
| Table Format   | Delta Lake         |
| Governance     | Unity Catalog      |
| Streaming      | Event Hubs         |
| Analytics      | Microsoft Fabric   |
| Containers     | AKS                |
| Secrets        | Key Vault          |
| Monitoring     | Azure Monitor      |
| Infrastructure | Terraform          |

Example architecture:

```text
API / Files / Events
        ↓
Azure Data Factory
        ↓
    ADLS Gen2
        ↓
Azure Databricks
        ↓
     Spark
        ↓
   Delta Lake
        ↓
 Unity Catalog
        ↓
   dbt / Fabric
```

---

# GCP Data Engineering Track

GCP becomes the serverless analytics implementation.

### Target Services

| Function          | GCP Service      |
| ----------------- | ---------------- |
| Object Storage    | Cloud Storage    |
| Warehouse         | BigQuery         |
| Batch / Streaming | Dataflow         |
| Spark             | Dataproc         |
| Streaming         | Pub/Sub          |
| Orchestration     | Cloud Composer   |
| Containers        | GKE              |
| Secrets           | Secret Manager   |
| Monitoring        | Cloud Monitoring |
| Infrastructure    | Terraform        |

Example architecture:

```text
Events
  ↓
Pub/Sub
  ↓
Dataflow
  ↓
BigQuery
  ↓
 dbt
  ↓
Analytics Models
```

---
