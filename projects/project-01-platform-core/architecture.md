# Project 01 — Architecture

## Component map

```
┌──────────────────────────────────────────────────────────────┐
│  USER LAYER                                                  │
│  ┌────────────────┐   ┌──────────────────┐                  │
│  │ Python SDK     │   │ CLI (smartrecs)  │                  │
│  └────────┬───────┘   └────────┬─────────┘                  │
│           │ HTTPS              │                            │
└───────────┼────────────────────┼────────────────────────────┘
            │                    │
┌───────────▼────────────────────▼────────────────────────────┐
│  CONTROL PLANE (in cluster)                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ FastAPI control-plane service                          │ │
│  │  • POST /v1/training-runs   (create)                   │ │
│  │  • GET  /v1/training-runs/{id}   (status)              │ │
│  │  • GET  /v1/training-runs   (list with filters)        │ │
│  │  • DELETE /v1/training-runs/{id}   (cancel)            │ │
│  └──────┬──────────────────────────────┬──────────────────┘ │
│         │ writes intent                │ emits audit event   │
│         ▼                              ▼                     │
│  ┌─────────────────┐         ┌─────────────────┐            │
│  │ Platform DB     │         │ Audit chain     │            │
│  │ (PostgreSQL)    │         │ (Postgres +     │            │
│  │  • runs         │         │  hash chain)    │            │
│  │  • tenants      │         └─────────────────┘            │
│  │  • quotas       │                                        │
│  └─────────────────┘                                        │
│         │                                                    │
│         │ applies CR                                         │
│         ▼                                                    │
│  ┌──────────────────────────┐                               │
│  │ Kubernetes API           │                               │
│  └──────────┬───────────────┘                               │
│             │ watches                                        │
│             ▼                                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ TrainingRun operator (controller)                    │  │
│  │  • reconcile()  — bring cluster state to spec        │  │
│  │  • on_failure() — capture status + emit audit event  │  │
│  │  • finalize()   — cleanup on delete                  │  │
│  └──────────┬───────────────────────────────────────────┘  │
│             │ creates / updates                              │
│             ▼                                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Workload resources (per tenant namespace)            │  │
│  │  Job · ConfigMap · ServiceAccount · NetworkPolicy    │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## Key design decisions

### 1. CRD is the source of truth, DB mirrors it

The Kubernetes `TrainingRun` CR is the **operational** source of
truth — the operator reconciles against it. The platform DB
mirrors the same data for **query** purposes (listing, filtering,
joining with tenant data) and for **audit** purposes (history
that survives CR deletion).

Trade-off considered: a database-only model is simpler but loses
Kubernetes-native semantics (RBAC, garbage collection,
finalizers). A CR-only model loses fast cross-tenant query.
The dual-store pattern accepts the consistency-management
overhead in exchange.

### 2. The control plane validates, the operator enforces

The control plane validates the request at admission time:
schema, tenant quota, image allowlist, dataset existence. It
applies the CR to the cluster.

The operator enforces invariants continuously: if a tenant's
quota is reduced *after* a run was admitted, the operator
detects the violation and prevents further pod creation; it
does not retroactively kill running pods (that's a separate
policy decision).

### 3. Multi-tenancy via namespace + workload identity

Each tenant gets:
- A dedicated namespace.
- A `ResourceQuota` and `LimitRange` in that namespace.
- A `ServiceAccount` per training-run pattern.
- A default-deny `NetworkPolicy` with allows only for the
  feature store, model registry, and the tenant's own data
  warehouse path.
- Workload-identity binding (SPIFFE SVID or cloud IRSA) so the
  training pod can read only its assigned dataset path.

### 4. CRD versioning

Ships at `v1alpha1`. Includes a `conversion` webhook to
support `v1beta1` and `v1` later without breaking existing
clients. The Kubernetes ConvertibleResource pattern.

### 5. Reconciliation idempotency

Every `reconcile()` invocation produces the same Kubernetes
state for the same `spec`. The operator is safe to restart at
any point.

## CRD spec

```yaml
apiVersion: platform.smartrecs.io/v1alpha1
kind: TrainingRun
metadata:
  name: recs-v17-experiment
  namespace: recs-team    # the tenant namespace
spec:
  image: ghcr.io/smartrecs/trainer:v3.2.1@sha256:abc...
  command: ["python", "train.py"]
  resources:
    requests:
      cpu: "8"
      memory: 32Gi
      nvidia.com/gpu: 1
    limits:
      cpu: "8"
      memory: 32Gi
      nvidia.com/gpu: 1
  dataset:
    name: recs-curated-2026-05
    version: v1.3
  hyperparameters:
    lr: 0.001
    batch_size: 64
    epochs: 10
  outputs:
    artifact_uri: s3://smartrecs-models/recs/{run-id}/
    metrics_uri: mlflow://recs/{run-id}
  retries:
    max: 3
    backoff: exponential
status:
  phase: Running             # Pending|Running|Succeeded|Failed|Cancelled
  startedAt: "2026-05-26T10:00:00Z"
  completedAt: null
  conditions:
    - type: Admitted
      status: "True"
      lastTransition: "2026-05-26T09:59:55Z"
    - type: Running
      status: "True"
      lastTransition: "2026-05-26T10:00:00Z"
  metrics:
    podStatus: "Running"
    podName: training-run-recs-v17-xq2zd
```

## Data model (Platform DB)

```sql
-- Tenants
CREATE TABLE tenants (
    id UUID PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    namespace TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Quotas (per tenant)
CREATE TABLE quotas (
    tenant_id UUID REFERENCES tenants(id),
    resource_type TEXT NOT NULL,    -- 'gpu_hours_per_month', 'concurrent_runs', etc.
    limit_value NUMERIC NOT NULL,
    PRIMARY KEY (tenant_id, resource_type)
);

-- Training runs (mirror of the CR for query)
CREATE TABLE training_runs (
    id UUID PRIMARY KEY,
    tenant_id UUID REFERENCES tenants(id),
    name TEXT NOT NULL,
    spec JSONB NOT NULL,              -- the full CR spec
    status JSONB NOT NULL,            -- denormalized status
    phase TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_runs_tenant_phase ON training_runs(tenant_id, phase);
CREATE INDEX idx_runs_created ON training_runs(created_at);
```

## Audit chain

Every significant action emits an audit event into a hash-
chained log (per Module 03 §8 of the security track):

- `training_run_admitted` — control plane admitted the request.
- `training_run_started` — operator created the Job.
- `training_run_completed` — run finished (success or failure).
- `training_run_cancelled` — user cancelled.
- `quota_violation` — admission rejected due to quota.
- `policy_violation` — admission rejected due to policy.

Each entry: signing identity (the control plane's or operator's
workload identity), timestamp, payload, hash of previous entry.

## Non-functional requirements

| Property | Target |
|---|---|
| Control-plane p95 latency (admit) | < 500ms |
| Operator reconcile latency (p95) | < 2s |
| Audit-chain entry latency | < 100ms (async; failure must NOT block the user) |
| Control plane availability | Single replica acceptable for capstone; HA is out of scope |
| Operator availability | Leader-elected, but single replica acceptable for capstone |
| Failure recovery on restart | Reconciliation must converge in < 30s |

## What's deliberately out of scope

- **Web UI**: CLI + SDK only.
- **Distributed training across nodes**: single-job training
  only. Multi-pod training is a Module 04 / project-03 concern.
- **Cost-based scheduling**: simple FIFO is sufficient.
- **Production HA** for the control plane.

## Cross-references

- [`engineer-solutions/mod-104`](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-104-kubernetes) — Kubernetes patterns.
- [`senior-engineer-solutions/projects/project-204-k8s-operator`](https://github.com/ai-infra-curriculum/ai-infra-senior-engineer-solutions/tree/main/projects/project-204-k8s-operator) — production-grade operator example.
- Module 03 lecture notes (multi-tenancy patterns) for the tenant-isolation design.
