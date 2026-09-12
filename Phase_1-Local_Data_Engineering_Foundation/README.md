# Phase 1 — Local Data Engineering Foundation

Begin with the core technologies required to build and operate a basic data pipeline:

```text
Linux
  ↓
Git
  ↓
Python
  ↓
PostgreSQL
```

PostgreSQL runs inside Docker rather than being permanently installed as a host service.

Python performs ingestion and transformation while PostgreSQL initially serves as both the operational data source and analytical warehouse.

## Initial Pipeline

```text
API / Files
    ↓
  Python
    ↓
PostgreSQL RAW
    ↓
Transformation
    ↓
Curated Data
    ↓
Data Quality Tests
```

This establishes the fundamental Data Engineering workflow:

* Data ingestion
* SQL
* Transformation
* Testing
* Logging
* Repeatable execution
* Data quality validation

---

## Add dbt

Next, introduce **dbt** and organize PostgreSQL into warehouse-style layers:

```text
RAW
 ↓
STAGING
 ↓
INTERMEDIATE
 ↓
MARTS
```

Practice:

* Dimensional modeling
* Incremental models
* Snapshots
* Jinja
* Macros
* dbt tests
* Documentation
* Lineage
* Source freshness

---

## Add Airflow

Once individual pipeline components work correctly, introduce **Apache Airflow** as the orchestration layer.

Airflow coordinates:

```text
Python Ingestion
      ↓
Transformation
      ↓
     dbt
      ↓
Data Quality Validation
```

---

## Add PySpark

Introduce **PySpark** after the core pipeline is stable.

Initially use local Spark execution rather than attempting to simulate a large distributed Spark cluster.

Focus on:

* DataFrames
* Transformations
* Actions
* Joins
* Window functions
* Partitioning
* Shuffle behavior
* Predicate pushdown
* Partition pruning
* Parquet
* Spark SQL
* Explain plans

---

## Phase 1 Learning Progression

```text
Linux / Git
    ↓
Python / PostgreSQL
    ↓
Docker
    ↓
dbt
    ↓
Airflow
    ↓
PySpark
    ↓
Databricks
```

The same project evolves at every stage.

---