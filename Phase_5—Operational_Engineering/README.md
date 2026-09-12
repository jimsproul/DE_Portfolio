# Phase 5 — Operational Engineering

The final phase moves beyond simply making pipelines work.

## CI/CD Deployment Lifecycle

Jenkins becomes the deployment controller.

```text
Git
 ↓
Tests
 ↓
Terraform Plan
 ↓
Approval
 ↓
Terraform Apply
 ↓
AWS / Azure / GCP
 ↓
Validation
```

---

## Cost-Controlled Cloud Environments

Cloud environments should be temporary whenever practical.

```bash
terraform apply
```

Run the experiment.

Collect results.

Then:

```bash
terraform destroy
```

This teaches infrastructure lifecycle management while controlling cloud costs.

---

## Drift and Failure Exercises

The lab should deliberately introduce failures.

Examples:

1. Terraform creates infrastructure.
2. Manually modify the resource in the cloud console.
3. Run:

```bash
terraform plan
```

4. Detect the configuration drift.
5. Decide whether Terraform should restore the desired state or the IaC definition should change.

Repeat this exercise with:

* IAM policies
* Security groups
* Storage lifecycle rules
* Kubernetes configurations
* Networking
* Database settings

Also practice:

* Remote Terraform state
* State locking
* Infrastructure recovery
* Disaster recovery
* Secrets management
* Configuration rollback

---

# Observability and Data Reliability

Implement a common logical observability model across all three cloud platforms.

## Core Metrics

Track:

* Freshness SLA
* Quality Pass Rate
* Reconciliation Variance
* Rejected Records
* Pipeline Duration
* Data Volume
* Mean Time to Detect (MTTD)
* Mean Time to Recover (MTTR)
* Incident Frequency
* Repeat Incidents

Each cloud uses its native telemetry platform:

| Cloud | Monitoring       |
| ----- | ---------------- |
| AWS   | CloudWatch       |
| Azure | Azure Monitor    |
| GCP   | Cloud Monitoring |

The monitoring implementation changes, but the **logical reliability model remains consistent**.

---

# Engineering Roles Practiced

The completed lab provides practical exposure to several engineering disciplines.

| Role                    | Primary Technologies                     |
| ----------------------- | ---------------------------------------- |
| Data Engineer           | Python, PostgreSQL, PySpark, dbt         |
| Analytics Engineer      | SQL, dbt, PostgreSQL                     |
| Spark Engineer          | PySpark, Databricks, EMR, Dataproc       |
| Orchestration Engineer  | Airflow, MWAA, Composer                  |
| CI/CD Engineer          | Jenkins, Git, Docker                     |
| Kubernetes Engineer     | Kubernetes, k3d, kubectl, Helm           |
| Infrastructure Engineer | Terraform, Ansible, Packer               |
| AWS Platform Engineer   | Terraform, AWS CLI, CDK / CloudFormation |
| Azure Platform Engineer | Terraform, Azure CLI, Bicep              |
| GCP Platform Engineer   | Terraform, gcloud                        |
| DevSecOps Engineer      | OPA, Conftest, Secrets Management        |
| Platform Engineer       | IaC, CI/CD, Kubernetes, Observability    |

---

# Full Learning Progression

```text
Linux / Git
    ↓
Python / PostgreSQL
    ↓
Docker / Docker Compose
    ↓
dbt
    ↓
Airflow
    ↓
PySpark
    ↓
Databricks
    ↓
Jenkins
    ↓
Terraform
    ↓
Kubernetes / k3d
    ↓
Helm
    ↓
AWS
    ↓
Azure
    ↓
GCP
    ↓
Terraform State / Locking
    ↓
Ansible
    ↓
AWS CDK / Azure Bicep
    ↓
Packer
    ↓
Policy-as-Code
    ↓
Multi-Cloud CI/CD
    ↓
Drift / Disaster Recovery
```

---

# End-to-End Engineering Lifecycle

The project demonstrates the complete engineering lifecycle:

```text
Develop
   ↓
Ingest
   ↓
Transform
   ↓
Test
   ↓
Orchestrate
   ↓
Containerize
   ↓
Provision
   ↓
Deploy
   ↓
Observe
   ↓
Recover
   ↓
Destroy
```

---
