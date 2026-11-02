# CreditLens

Intelligent SME credit underwriting platform — alternative scoring with XGBoost/SHAP, compliance RAG over BCT/Basel docs, and full loan lifecycle management via Spring Cloud microservices.

## What it does

- **ML credit scoring** — scores SMEs on transaction behavior (cash flow, payment patterns, revenue stability) using XGBoost/LightGBM with SHAP explainability per decision
- **Compliance RAG** — loan officers query BCT circulars, Basel IV norms, and KYC/AML policy documents in natural language with cited answers
- **Loan lifecycle management** — end-to-end state machine from application intake to post-disbursement drift monitoring
- **Audit trail** — every decision produces an immutable snapshot: model version, feature values, SHAP output, threshold, compliance citations, officer ID, timestamp

## Stack

| Layer | Technologies |
|---|---|
| Backend | Java 21, Spring Boot 4.1, Spring Cloud 2025, Keycloak, RabbitMQ, PostgreSQL |
| Frontend | Angular 18 |
| MLOps | Python, FastAPI, Airflow, MLflow, DVC, XGBoost, LightGBM, SHAP |
| RAG | FastAPI, pgvector, RAGAS |
| Data | MinIO, Apache Iceberg, Trino, PySpark |
| DevOps | Docker, Kubernetes, Kustomize, GitHub Actions, Prometheus, Grafana, OpenTelemetry |

## Repository structure

creditlens/
├── services/
│ ├── api-gateway/
│ ├── application-service/
│ ├── scoring-service/
│ ├── document-service/
│ ├── rag-service/
│ ├── notification-service/
│ ├── eureka-server/
│ └── config-server/
├── ml-engine/
├── rag-engine/
├── airflow/
├── frontend/
├── k8s/
├── docs/
└── config-repo/



## Getting started

### Prerequisites

- Docker Desktop
- Java 21
- Python 3.11+
- Node 20+

### Run infrastructure

```bash
docker-compose up -d
```

This starts: PostgreSQL ×3, RabbitMQ, Keycloak, MinIO, pgvector.

Note: docker-compose.yml is not yet in the repo — it lands in Sprint 1 Day 2 (CL-2). The instructions above will work once CL-2 is merged.

### Config Server

The `config-repo/` directory is the git-backed configuration store for Spring Cloud Config Server. It is tracked as regular files in this repository.

### Services

Each service under `services/` is an independent Spring Boot application. See the README inside each service directory for individual setup instructions.

## Sprints

| Sprint | Goal |
|---|---|
| S1 | Platform Foundation + Loan Application Lifecycle |
| S2 | ML Scoring Engine + Notifications |
| S3 | Auditability + Role-keyed RBAC |
| S4 | Compliance RAG |
| S5 | Feature Pipeline + MLOps Loop |
| S6 | Shadow Mode Infrastructure |
| S7 | Observability + Kubernetes |
| S8 | SME Portal + Post-disbursement |

## License

MIT