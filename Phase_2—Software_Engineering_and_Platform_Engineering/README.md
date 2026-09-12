# Phase 2 — Software Engineering and Platform Engineering

Once the data pipeline works, begin treating it as a production software project.

## CI/CD with Jenkins

Introduce Jenkins as the CI/CD engine.

A Git commit should trigger tasks such as:

```text
Git Commit
    ↓
 Jenkins
    ↓
Python Lint
    ↓
  pytest
    ↓
dbt Compile
    ↓
 dbt Test
    ↓
Rust Tests
    ↓
Docker Build
```

The objective is to automate validation before deployment.

---

## Infrastructure as Code with Terraform

Terraform becomes a first-class engineering discipline rather than merely a cloud deployment tool.

Learn:

* Providers
* Resources
* Variables
* Outputs
* Modules
* State
* Remote state
* State locking
* Imports
* Drift detection
* Lifecycle management

Start locally before moving into cloud environments.

A useful progression is:

```text
Docker Compose
      ↓
Terraform Docker Provider
      ↓
Kubernetes YAML
      ↓
Helm
```

---

## Kubernetes

Introduce Kubernetes locally using **k3d**.

Start with native Kubernetes objects before adding higher-level abstractions.

Learn:

* Pods
* Deployments
* Services
* ConfigMaps
* Secrets
* Jobs
* CronJobs
* Persistent Volumes
* Persistent Volume Claims
* Namespaces
* Resource requests
* Resource limits
* Health checks
* Ingress

---

## Helm

Once Kubernetes fundamentals are understood, package applications with Helm.

```text
Raw Kubernetes YAML
        ↓
      Helm
        ↓
Reusable Kubernetes Deployment
```

---

## Platform Tool Responsibilities

Each platform technology should have a clearly defined responsibility.

| Tool           | Responsibility                             |
| -------------- | ------------------------------------------ |
| Terraform      | Create infrastructure                      |
| Ansible        | Configure hosts                            |
| Helm           | Package and deploy Kubernetes applications |
| Jenkins        | Control CI/CD lifecycle                    |
| Packer         | Create immutable machine images            |
| OPA / Conftest | Policy-as-Code enforcement                 |

This separation reflects a production Platform Engineering model rather than treating IaC as a collection of deployment scripts.

---
