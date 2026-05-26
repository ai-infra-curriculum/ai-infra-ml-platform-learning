# Project 01: Self-Service ML Platform Core

> **Tier**: Capstone
> **Track**: AI/ML Platform Engineering
> **Estimated effort**: 80 hours
> **Complexity**: Advanced
> **Primary modules**: mod-001 (Platform Fundamentals), mod-002 (API Design), mod-003 (Multi-Tenancy & Resources), mod-007 (Developer Experience)
> **Secondary modules**: mod-008 (Observability), mod-009 (Security & Governance)

## 1. Overview

Build the **foundational platform layer** that turns Kubernetes
into a self-service ML platform. Data scientists submit
training jobs through a CLI or SDK that talks to a control
plane; the control plane translates platform intents into
Kubernetes objects, enforces multi-tenancy, and produces
auditable evidence of every action.

The deliverable is not a wrapper around `kubectl`. It is a
**platform product** with:

- A user-facing CRD (`TrainingRun`) and a Python SDK.
- A control-plane API that validates intents, applies quotas,
  and emits audit events.
- A reconciliation operator that materializes Kubernetes
  resources from intents.
- Per-tenant isolation via namespaces + workload identity +
  network policies.
- Metrics + structured logs from every component.

## 2. Why this project matters

Module 01's lecture notes spend significant time on the
distinction between "a Kubernetes cluster with extra steps"
and "a platform." This project forces you to build the
*difference*: the higher-order primitives, the contract layer,
and the operator pattern.

By the end you will have:

- Operated the operator pattern end-to-end (CRD → controller →
  status reconciliation).
- Designed and shipped a versioned API contract.
- Implemented per-tenant authorization with real evidence.
- Produced an audit trail that survives compliance review.

These are the skills the ML Platform Engineer track is
ultimately graded on.

## 3. What you will build

### The user-facing surface

```python
from smartrecs_platform import Client

client = Client()

run = client.create_training_run(
    name="recs-v17-experiment",
    image="ghcr.io/smartrecs/trainer:v3.2.1",
    resources={"cpu": "8", "memory": "32Gi", "gpu": "1"},
    dataset="recs-curated-2026-05",
    hyperparameters={"lr": 1e-3, "batch_size": 64, "epochs": 10},
    tenant="recs-team",
)

# Wait for completion + access metrics
run.wait()
print(run.status)        # Succeeded | Failed | Cancelled
print(run.metrics_uri)   # MLflow run URI
print(run.artifact_uri)  # Model registry URI
```

### The CRD (`TrainingRun`)

A custom resource that the operator watches. The user (or SDK)
applies this; the operator reconciles to materialize the run.

### The control plane

A FastAPI service that:
- Validates incoming `TrainingRun` requests against the
  schema and the tenant's quotas.
- Stores the request in the platform DB (PostgreSQL).
- Applies the corresponding CR to the cluster.
- Exposes a status API.
- Emits structured audit events.

### The operator

A Kubernetes controller (Python with kopf, or Go with
controller-runtime — choose one) that:
- Watches `TrainingRun` resources.
- Creates the underlying Job, ConfigMap, Service Account, and
  network policies.
- Updates `status` as the run progresses.
- Cleans up on completion or failure.

### Multi-tenancy

Per-tenant:
- Namespace.
- ResourceQuota and LimitRange.
- ServiceAccount with workload identity (SPIFFE-style or cloud
  IAM-bound).
- NetworkPolicy default-deny + tenant-internal allows.

### Observability

- Prometheus metrics from the control plane and operator
  (request rate, reconciliation latency, error rate).
- Structured JSON logs.
- Audit-chain entries (hash-chained, signed) for every
  significant action.

## 4. Architecture at a glance

```
┌─────────────────┐         ┌──────────────────┐
│ User / SDK      │ ──API──▶│ Control plane    │
│ (Python client) │         │ (FastAPI + DB)   │
└─────────────────┘         └────────┬─────────┘
                                     │ creates CR
                                     ▼
                            ┌──────────────────┐
                            │ Kubernetes API   │
                            └────────┬─────────┘
                                     │ watches CR
                                     ▼
                            ┌──────────────────┐
                            │ Operator         │
                            │ (reconciles)     │
                            └────────┬─────────┘
                                     │ creates
                                     ▼
                ┌─────────────┬─────────────┬─────────────┐
                │ Job         │ ConfigMap   │ ServiceAcct │
                │ NetPolicy   │ Quota       │ etc.        │
                └─────────────┴─────────────┴─────────────┘
```

See [`architecture.md`](./architecture.md) for the full design.

## 5. Skills you will demonstrate

By submitting this project you can defend:

- The operator pattern (informers, reconciliation, finalizers,
  status writes).
- CRD design that's versionable + extensible (v1alpha1 → v1
  migration story).
- Multi-tenant isolation across namespace, RBAC, network
  policy, and workload identity simultaneously.
- API design (versioning, error responses, pagination,
  filtering, sorting).
- Audit-chain integration that produces compliance-grade
  evidence.

## 6. Out of scope

- A web UI (CLI + SDK only).
- Model serving (covered separately in mod-006-model-management
  and project-04-model-registry).
- Feature store (project-02-feature-store).
- Workflow DAGs spanning many runs (project-03-workflow-
  orchestration).
- Production-grade HA for the control plane (single replica is
  acceptable for the capstone).

## 7. Time budget

| Phase | Hours |
|---|---|
| Design (CRD, API, schema) | 8 |
| Operator implementation | 25 |
| Control plane + DB | 15 |
| Multi-tenancy + isolation | 10 |
| Observability + audit | 8 |
| Testing + documentation | 14 |
| **Total** | **~80** |

## 8. Deliverables

See [`requirements.md`](./requirements.md) for the full
deliverable list and acceptance criteria.

## 9. How to start

1. Read all four files for this project: [`README.md`](./README.md),
   [`architecture.md`](./architecture.md),
   [`requirements.md`](./requirements.md),
   [`STEP_BY_STEP.md`](./STEP_BY_STEP.md).
2. Skim Module 01-03 lecture notes if you haven't recently.
3. Pick your operator framework (kopf for Python, controller-
   runtime for Go).
4. Bring up a kind / k3d cluster for development.
5. Start with the CRD + a minimal operator that just logs
   "saw a TrainingRun" — get the reconciliation loop wired up
   before you implement the business logic.

## 10. Cross-references

- [`ai-infra-ml-platform-solutions/SOLUTION_OVERVIEW.md`](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-solutions/blob/main/SOLUTION_OVERVIEW.md)
- [`engineer-solutions/mod-104` (operator pattern reference)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-104-kubernetes)
- [`senior-engineer-solutions/projects/project-204-k8s-operator`](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-solutions/tree/main/projects/project-204-k8s-operator) — production-grade `TrainingJob` operator
