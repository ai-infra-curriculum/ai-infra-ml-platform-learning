# Project 01 — Step-by-Step Build Guide

This guide walks you through building the platform core in
phases. Resist building everything at once; each phase
produces a working state you can demo.

## Phase 0 — Local cluster + project skeleton (1-2 hours)

1. **Bring up a local Kubernetes**.
   - `kind create cluster --name smartrecs`, or
   - `k3d cluster create smartrecs`.
2. **Verify** with `kubectl get nodes`.
3. **Project skeleton**:
   ```
   project-01-platform-core/
   ├── control-plane/        # FastAPI app
   ├── operator/             # The TrainingRun controller
   ├── sdk/                  # Python client library
   ├── cli/                  # CLI (Click or Typer)
   ├── crd/                  # CRD + RBAC manifests
   ├── deploy/               # Helm or kustomize for the platform
   ├── tests/                # Unit + integration tests
   ├── docs/
   └── Makefile
   ```
4. **Pin Python version + deps** in `pyproject.toml`.

## Phase 1 — The CRD (2-3 hours)

The CRD is the contract. Get it right first.

1. **Author the CRD YAML** following the schema in
   `architecture.md`. Use OpenAPI v3 schema validation.
2. **Apply** to the cluster: `kubectl apply -f crd/training-run-crd.yaml`.
3. **Verify** the schema rejects invalid specs:
   ```bash
   # Should be rejected (missing required fields)
   kubectl apply -f tests/fixtures/invalid-run.yaml
   # Should be accepted
   kubectl apply -f tests/fixtures/minimal-valid-run.yaml
   ```
4. **Write the conversion stub** for `v1alpha1 → v1beta1`. It
   does nothing yet but the structure must be in place.

## Phase 2 — The operator skeleton (5-7 hours)

Get reconciliation wired up before adding business logic.

1. **Set up kopf (Python) or controller-runtime (Go)**:
   ```python
   # operator/main.py (kopf version)
   import kopf

   @kopf.on.create('platform.smartrecs.io', 'v1alpha1', 'trainingruns')
   @kopf.on.update('platform.smartrecs.io', 'v1alpha1', 'trainingruns')
   def reconcile_training_run(spec, status, namespace, name, **_):
       logger.info(f"reconcile: {namespace}/{name}", extra={"spec": spec})
       # TODO: business logic
   ```
2. **Apply RBAC** so the operator can watch `TrainingRun`,
   create Jobs/ConfigMaps/ServiceAccounts, and update status.
3. **Deploy the operator** into the cluster (or run locally
   against the cluster for fast iteration).
4. **Apply a sample CR** and verify the log fires. You haven't
   done anything yet — but reconciliation works.

## Phase 3 — Operator reconciliation (8-10 hours)

Now the business logic.

1. **Implement `reconcile()`**:
   - If status.phase is empty → create Job + ConfigMap + SA.
   - If Job exists → read its status, update CR status.
   - If Job completed → set CR phase to Succeeded / Failed.
2. **Implement finalizers** so deletion cleans up resources.
3. **Implement retries**: on Job failure, recreate up to
   `spec.retries.max` times with backoff.
4. **Idempotency**: every reconcile() invocation must produce
   the same cluster state for the same input.

### Test cases to cover

- Happy path: create → run → succeed.
- Failure path: create → run → fail → retry → succeed.
- Cancellation: delete CR mid-run → Job cleanup.
- Operator restart mid-run: state recovers correctly.

## Phase 4 — Control plane API (6-8 hours)

The control plane is the user-facing surface.

1. **Scaffold FastAPI**:
   ```
   control-plane/
   ├── main.py
   ├── routers/
   │   ├── training_runs.py
   │   ├── tenants.py
   ├── models/         # Pydantic schemas
   ├── db/             # SQLAlchemy models + migrations
   ├── k8s/            # kubernetes client helpers
   ├── audit/          # audit-chain client
   └── auth/           # token validation
   ```
2. **Implement `POST /v1/training-runs`**:
   - Validate input.
   - Look up tenant + quota.
   - Insert into platform DB.
   - Apply CR to cluster.
   - Emit audit event.
   - Return 201.
3. **Implement `GET` endpoints** with filtering + pagination.
4. **Implement `DELETE`** (cancel) that updates DB + deletes
   the CR.
5. **Add `X-Request-Id` middleware** that propagates the ID
   into logs and audit entries.

### Test cases to cover

- Create → 201, find in DB, find in cluster.
- Create with quota exceeded → 429 with clear error.
- Create with invalid image format → 400.
- Get non-existent → 404.
- List with filters → correct subset.
- Cancel → CR is deleted, DB status is Cancelled.

## Phase 5 — Multi-tenancy (5-7 hours)

Tenant isolation is where the platform either works or doesn't.

1. **Tenant onboarding**:
   - `POST /v1/tenants` creates the tenant record.
   - Operator (or a separate provisioner) creates the
     tenant's namespace, ResourceQuota, LimitRange,
     ServiceAccount, NetworkPolicy.
2. **Workload identity binding**:
   - Each tenant ServiceAccount maps to a SPIFFE ID or cloud
     IAM role.
   - The IAM/SPIFFE policy permits access only to that
     tenant's bucket prefix.
3. **NetworkPolicy default-deny**:
   - Tenant namespace gets a default-deny policy.
   - Allow egress to feature-store, model-registry, audit-log
     destinations.
   - Deny cross-tenant access.

### Test case to cover

The acceptance criterion from `requirements.md` §F4:
- Pod in tenant A attempts to read tenant B's bucket prefix
  → fails at IAM.
- Pod in tenant A attempts to call a service in tenant B's
  namespace → fails at NetworkPolicy.

## Phase 6 — Observability (3-4 hours)

1. **Add `prometheus_client` to the control plane and operator**.
2. **Expose `/metrics`** on both.
3. **Wire structured JSON logging**.
4. **Author a Grafana dashboard** as JSON (rendered live
   against a kind cluster's metrics; commit the JSON for
   replay).

## Phase 7 — Audit chain (3-4 hours)

The audit chain is the compliance backbone.

1. **PostgreSQL `audit_log` table** with insert-only constraint
   (no UPDATE/DELETE; enforced at SQL level via trigger).
2. **Each row**: id (sequential), timestamp, identity,
   payload (JSONB), payload_hash (SHA-256), prev_hash.
3. **Per-write**: compute hash of (payload || prev_hash);
   write into the row.
4. **`verify()` function** (CLI or script): walks the table
   in order, recomputes hashes, returns the first mismatch
   or "verified".

## Phase 8 — SDK + CLI (3-4 hours)

1. **Python SDK** wrapping the control plane API.
2. **CLI** using Click or Typer.
3. **Examples** in `examples/`:
   - `submit_run.py` — submit a run via SDK.
   - `cli_walkthrough.sh` — end-to-end CLI demo.

## Phase 9 — Testing + acceptance (4-6 hours)

1. **Unit tests** on control-plane API.
2. **Operator tests** with envtest / kopf test harness.
3. **End-to-end test** that:
   - Brings up the platform.
   - Creates a tenant.
   - Submits a run.
   - Waits for completion.
   - Verifies status, audit chain, output artifacts.
4. **Acceptance demo script** matching the criteria in
   `requirements.md`.

## Phase 10 — Documentation (3-4 hours)

1. **README** at project root.
2. **ARCHITECTURE.md** capturing your decisions.
3. **OpenAPI spec** auto-generated from FastAPI.
4. **Onboarding guide** for adding a tenant.
5. **Runbook** for common ops issues.

## Time-budget recap

| Phase | Hours |
|---|---|
| 0 — Setup | 1-2 |
| 1 — CRD | 2-3 |
| 2 — Operator skeleton | 5-7 |
| 3 — Reconciliation | 8-10 |
| 4 — Control plane | 6-8 |
| 5 — Multi-tenancy | 5-7 |
| 6 — Observability | 3-4 |
| 7 — Audit chain | 3-4 |
| 8 — SDK + CLI | 3-4 |
| 9 — Testing | 4-6 |
| 10 — Documentation | 3-4 |
| **Buffer** | 10 |
| **Total** | **~80** |

## Common pitfalls

- **Reaching for full distributed-training support**. Stay
  single-pod; this is a platform project, not a training
  project.
- **Skipping finalizers**. They prevent dangling resources.
- **Treating the CRD as immutable**. Plan for `v1` from day
  one; the conversion webhook is part of the deliverable.
- **Skipping the audit-chain insert-only constraint**. The DB
  must reject UPDATE/DELETE on the audit table; this is the
  thing an auditor checks.
- **Hardcoding tenant lists**. Tenants are first-class
  resources, not config.

## When you're done

You have a real ML platform with:
- A working operator.
- A user-facing API + SDK + CLI.
- Multi-tenant isolation verifiable in tests.
- Audit-chain evidence of every action.
- Observability for the team that operates the platform.

The skills demonstrated here are the foundation for the rest
of the ML platform track. Projects 02-05 build out specific
layers on top.
