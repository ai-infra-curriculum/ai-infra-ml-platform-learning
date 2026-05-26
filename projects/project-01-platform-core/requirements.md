# Project 01 — Requirements

Each requirement below must be demonstrably met before the
project is considered complete. Acceptance criteria are
explicit; document any deviation in your submission.

## Functional requirements

### F1 — `TrainingRun` CRD ships

- [ ] CRD installed in the cluster via `kubectl apply`.
- [ ] OpenAPI schema enforces spec validity at admission.
- [ ] At least 5 spec fields validated (image format, resource
      shapes, dataset reference, hyperparameter types, retry
      config).
- [ ] CRD is versioned `v1alpha1` and includes structural
      schema for future conversion.

### F2 — Control plane API

- [ ] `POST /v1/training-runs` creates a run; returns 201 with
      the run ID and current status.
- [ ] `GET /v1/training-runs/{id}` returns the run + status.
- [ ] `GET /v1/training-runs` supports filters: `tenant`,
      `phase`, `created_after`, `created_before`.
- [ ] `DELETE /v1/training-runs/{id}` cancels (sets phase to
      Cancelled, deletes the Job).
- [ ] Errors return JSON with structure `{error, code,
      request_id}` — never stack traces.
- [ ] Per-request `X-Request-Id` header is propagated to logs
      and audit-chain entries.

### F3 — Operator reconciliation

- [ ] On `TrainingRun` creation: operator creates Job +
      ConfigMap + ServiceAccount in the tenant namespace.
- [ ] On `TrainingRun` deletion: operator cleans up associated
      resources (Finalizers used correctly).
- [ ] Operator restart: existing runs converge to correct state
      within 30 seconds.
- [ ] Status fields updated as the Job progresses
      (Pending → Running → Succeeded / Failed).
- [ ] Failed runs retry per `spec.retries.max`, with
      exponential backoff.

### F4 — Multi-tenancy

- [ ] Each tenant has its own namespace, created at tenant
      onboarding.
- [ ] `ResourceQuota` enforces per-tenant GPU + memory + CPU
      caps.
- [ ] `LimitRange` enforces per-pod defaults.
- [ ] `NetworkPolicy` default-deny in the tenant namespace
      with allows only for the platform's expected egress
      destinations (feature store, MLflow, object store).
- [ ] Workload identity (SPIFFE SVID or cloud IRSA) binds
      training pods to tenant-scoped credentials. A
      compromised pod in tenant A cannot read tenant B's data.

### F5 — Quota enforcement

- [ ] Control plane rejects new runs that would exceed
      tenant's monthly GPU-hour quota.
- [ ] Concurrent-run limit enforced per tenant.
- [ ] Quota usage is queryable via
      `GET /v1/tenants/{id}/quota-usage`.

### F6 — SDK + CLI

- [ ] Python SDK with the methods described in `README.md` §3.
- [ ] CLI (`smartrecs runs create | list | status | cancel`)
      wrapping the same operations.
- [ ] SDK + CLI authenticate to the control plane via a token
      (OIDC or static bearer for the capstone).
- [ ] SDK examples in `examples/` directory of the deliverable.

### F7 — Observability

- [ ] Prometheus metrics at `/metrics` on the control plane
      and operator:
      - `platform_runs_admitted_total{tenant, phase}`
      - `platform_runs_active{tenant}`
      - `platform_admission_latency_seconds`
      - `platform_reconcile_latency_seconds`
      - `platform_audit_chain_writes_total{result}`
- [ ] Structured JSON logs from all components.
- [ ] One Grafana dashboard JSON shipped in the deliverable.

### F8 — Audit chain

- [ ] Every significant action emits an audit event.
- [ ] Each entry includes: timestamp, signing identity,
      tenant, action, resource, payload hash, previous-entry
      hash.
- [ ] A `verify` command walks the chain and returns the first
      detected tampering (or "verified" if clean).
- [ ] The chain stores in PostgreSQL with a constraint that
      prevents updates to inserted rows (insert-only).

## Non-functional requirements

### NF1 — Reproducibility

- [ ] Project comes up with a single command: `make up`.
- [ ] All infrastructure declared as Kubernetes manifests
      (`deploy/` directory) — no manual `kubectl` setup steps
      required.
- [ ] Reasonable defaults; a `.env.example` documents required
      configuration.

### NF2 — Testing

- [ ] Unit tests for the control plane API (≥80% coverage).
- [ ] Operator tests using `envtest` (or kopf's test harness)
      covering reconciliation paths.
- [ ] End-to-end test: create a tenant, submit a run, wait for
      completion, verify outputs.

### NF3 — Security

- [ ] Container images run as non-root.
- [ ] No long-lived static credentials in the cluster.
- [ ] Secrets via External Secrets Operator (or equivalent),
      not raw `Secret` resources.
- [ ] All container images Cosign-signed (CI signing keyless).

### NF4 — Documentation

- [ ] `README.md` at the project root explaining what it is,
      what it does, and how to bring it up.
- [ ] `ARCHITECTURE.md` capturing the decisions in your
      implementation.
- [ ] API reference (OpenAPI spec) auto-generated from FastAPI.
- [ ] Onboarding guide for adding a new tenant.
- [ ] Operations runbook for common incidents (operator
      restart, DB recovery, stuck reconciliation).

## Acceptance demo

The grading exercise:

1. Spin up a fresh kind / k3d cluster.
2. `make up` the platform.
3. Create two tenants (`team-a`, `team-b`) via the CLI.
4. Submit one training run for each tenant.
5. While both are running, attempt to submit a run for
   `team-a` that exceeds its GPU-hour quota → must be
   rejected at admission with a clear error.
6. Attempt to read `team-b`'s output bucket from a pod
   running in `team-a` → must be denied at the IAM / network
   layer.
7. Run the audit-chain `verify` against the platform DB
   after the test run → must return "verified".
8. Restart the operator → in-flight runs must continue
   correctly.

## What's explicitly NOT graded

- UI / visual design.
- Multi-region or HA deployments.
- Distributed training across multiple pods (single-pod jobs
  are fine).
- Web frontend (CLI + SDK only).
- Production-grade quota algorithms (basic per-tenant caps are
  enough).

## Submission

Place under `deliverables/`:

- `code/` — your implementation (clearly organized:
  control-plane, operator, sdk, cli, deploy/).
- `docs/` — your architecture decisions, runbooks, ADRs.
- `tests/` — your test suite + a runnable demo script.
- `screenshots/` — Grafana dashboard, audit-chain verification
  output, denied-cross-tenant evidence.
- `SUBMISSION.md` — a 1-page summary of what you built, the
  decisions you made, the trade-offs accepted, and the gaps
  you're aware of.
