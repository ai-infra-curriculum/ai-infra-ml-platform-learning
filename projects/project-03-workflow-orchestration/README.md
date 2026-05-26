# Project 03: ML Workflow Orchestration Engine

> **Tier**: Capstone
> **Track**: AI/ML Platform Engineering
> **Estimated effort**: 75 hours
> **Complexity**: Advanced
> **Primary modules**: mod-005 (Workflow Orchestration), mod-002 (API Design), mod-006 (Model Management)
> **Secondary modules**: mod-008 (Observability), mod-001 (Platform Fundamentals)

## 1. Overview

Build a **workflow orchestration engine** that lets data
scientists compose ML pipelines (data ingestion → feature
engineering → training → evaluation → promotion) declaratively
and run them with platform-grade scheduling, retry semantics,
and observability.

The deliverable is the orchestration layer the platform itself
provides. Not an Airflow wrapper — the actual engine.

## 2. Why this project matters

Production ML pipelines fail in three classes the platform must
handle: transient errors (retry with backoff), permanent errors
(fail loudly + run the right diagnostic playbook), and partial
successes (which downstream steps still run?). Most ML teams
re-discover this taxonomy the hard way. A workflow orchestration
engine that gets it right is what separates "shipping models"
from "shipping models reliably."

## 3. What you will build

### Declarative pipeline definition

```yaml
apiVersion: workflows.smartrecs.io/v1
kind: Pipeline
metadata:
  name: nightly-recs-retrain
  namespace: recs-team
spec:
  schedule: "0 2 * * *"
  steps:
    - name: ingest
      image: ghcr.io/smartrecs/ingest:v1.4
      env:
        SOURCE: warehouse.user_events
    - name: features
      image: ghcr.io/smartrecs/featurize:v2.1
      depends_on: [ingest]
    - name: train
      image: ghcr.io/smartrecs/trainer:v3.2
      depends_on: [features]
      resources:
        gpu: 1
    - name: evaluate
      image: ghcr.io/smartrecs/evaluator:v1.0
      depends_on: [train]
    - name: promote
      image: ghcr.io/smartrecs/promoter:v1.0
      depends_on: [evaluate]
      gates:
        - condition: "accuracy >= 0.85"
        - condition: "fairness_disparate_impact <= 1.25"
        - human_approval: true
  retries:
    default: 3
    backoff: exponential
  notification:
    on_failure: pagerduty/recs-team
    on_success: slack/#recs-deploys
```

### The engine

- **Scheduler**: cron-style + event-triggered.
- **Executor**: launches step pods, tracks status, handles
  retries.
- **DAG state machine**: tracks per-step state, enforces
  dependencies, handles partial failures.
- **Gates**: programmatic conditions + human approvals.
- **Observability**: per-pipeline + per-step metrics, logs,
  traces.

### Platform features

- Per-tenant isolation (workflows in tenant A don't see tenant
  B's resources).
- Audit-chain integration (every state transition emits an
  event).
- Pipeline-version-pinning + Git-style history.
- Re-run from a step (don't redo work that succeeded).

## 4. Out of scope

- Distributed dataflow (no Spark/Beam-style runners).
- Generic event sourcing (Kafka triggers are limited to
  workflow start, not mid-DAG).
- A web UI for designing pipelines (definition is YAML).

## 5. Time budget

| Phase | Hours |
|---|---|
| Pipeline CRD + parser | 8 |
| DAG state machine | 12 |
| Scheduler (cron + event) | 8 |
| Executor (pod runner) | 12 |
| Gates (programmatic + human) | 8 |
| Retry + backoff semantics | 6 |
| Re-run from step | 5 |
| Observability | 6 |
| Testing + documentation | 10 |
| **Total** | **~75** |

## 6. Cross-references

- [Module 05 lecture notes](../../lessons/mod-005-workflow-orchestration/).
- [Argo Workflows](https://argoproj.github.io/argo-workflows/) — read for inspiration; don't fork.
- [Kubeflow Pipelines](https://www.kubeflow.org/docs/components/pipelines/) — same.
- [`engineer-solutions/mod-105 exercise-01`](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-105-data-pipelines) — pipeline architecture reference.
