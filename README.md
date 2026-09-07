# 🛠️ MLOps 

> Production-grade machine learning pipelines focused on zero-downtime deployment, lineage tracking, scalable serving, and drift-aware observability.

---

## 🎯 Core Focus Areas

* **Pipeline Automation (CI/CD/CT)**: Builds zero-downtime pipelines automating Continuous Training (CT), model packaging, and canary/shadow deployments triggered by Git commits or detected data drift.
* **Reproducibility & Lineage**: Guarantees end-to-end auditability across datasets, parameters, and compiled model weights using declarative versioning and centralized registries.
* **Scalable Serving**: Optimizes inference latency and throughput via distributed execution runtimes, dynamic auto-scaling policies, and hardware resource efficiency.
* **Observability & Reliability**: Safeguards inference pipelines against performance degradation through real-time drift detection (data & concept) and systems metric telemetry.

---

## 💻 Tech Stack

| Domain | Technologies |
| :--- | :--- |
| **Pipeline & Orchestration** | Apache Airflow |
| **Experiment Tracking & Registry** | MLflow |
| **Model Serving & Inference** | FastAPI |
| **Infra & Containerization** | Docker |
| **Observability & Monitoring** | Prometheus, Grafana, Evidently AI |
| **Languages & ML Frameworks** | Python, Bash, PyTorch, Scikit-learn, Ray |

---

## 🏗️ Architecture & Philosophy

* **Decoupled Architecture**: Training workflows, orchestration engines, and real-time serving clusters operate independently to isolate faults and enable horizontal autoscaling.
* **Immutable Artifacts & Strict Lineage**: Every dataset snapshot, training configuration, and compiled binary is treated as an immutable state for deterministic rollback guarantees.
* **Continuous Validation (Data & Model)**: Schema constraints, distribution baseline checks, and performance regression gates are enforced prior to production promotion.
* **Resilient Production Serving**: Minimizes API latency and maximizes uptime via progressive rollouts (canary/shadow deployments) and automated fallback mechanisms.
