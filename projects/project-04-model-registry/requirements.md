# Project 04 — Requirements

## Functional requirements

### F1 — Registry CRUD + search

- [ ] `POST /v1/models` registers a model.
- [ ] `POST /v1/models/{id}/versions` registers a version.
- [ ] `GET /v1/models?tenant=...&status=...&accuracy_gte=...`
      lists with filters.
- [ ] `GET /v1/models/{id}/versions/{version}` returns full
      metadata + lineage.
- [ ] All metadata immutable post-registration.

### F2 — Signature verification

- [ ] Cosign keyless signing on registration.
- [ ] Verification at promotion (gate).
- [ ] Verification at deployment (gate).
- [ ] Verification fails fast with clear message on bad sigs.

### F3 — Promotion gates

- [ ] Default gates per `architecture.md` §Promotion gates.
- [ ] Configurable per model.
- [ ] Human approval gate records approver identity + signed
      approval.
- [ ] Failed gates halt promotion with the failed gate's name.

### F4 — Deployment + rollout

- [ ] Rolling, blue-green, canary, shadow strategies all
      implemented.
- [ ] Canary supports traffic-share parameter; ramp via API.
- [ ] Shadow doesn't affect customer-visible outputs but logs
      comparison data.
- [ ] Deployment recorded with target, strategy, deployer
      identity, timestamp.

### F5 — Rollback

- [ ] `POST /v1/deployments/{id}/rollback` returns target to
      prior `Production` version.
- [ ] Rollback emits an audit event with reason (required
      field).
- [ ] Rollback under 5 minutes from API call to traffic
      restored (single-cluster setup).

### F6 — Lineage

- [ ] Forward + reverse lineage queries.
- [ ] Edges populated automatically by the training pipeline
      via SDK on registration.
- [ ] Reverse query returns all affected models for a given
      data version.

### F7 — Multi-tenancy + audit

- [ ] Tenant scope enforced on list + read.
- [ ] Every promotion, deployment, rollback, share action
      emits an audit-chain event.
- [ ] Audit-chain `verify` clean.

### F8 — Observability

- [ ] Metrics: `registry_models_total`, `registry_promotions_total`,
      `registry_deployments_active`, `registry_rollouts_in_flight`.
- [ ] Per-model deployment timeline (visualizable in Grafana).
- [ ] Grafana dashboard JSON included.

## Non-functional requirements

- [ ] `make up` brings up registry + dependencies.
- [ ] Unit + integration tests.
- [ ] All container images Cosign-signed + non-root.
- [ ] OpenAPI spec auto-generated.

## Acceptance demo

1. Register 2 models with 2 versions each.
2. Promote one model `Registered → Staging → Production` with
   passing gates.
3. Attempt to promote with failing gate (low accuracy): denied.
4. Deploy with canary 5% strategy.
5. Ramp canary to 50%, then full.
6. Rollback to prior version.
7. Reverse-lineage query: "what models trained on dataset
   v3.2?" — correct list.
8. Cross-tenant access attempt: 403.
9. Audit-chain `verify`: clean.

## Submission

`deliverables/` contains:
- `code/`
- `docs/`
- `tests/`
- `screenshots/` — Grafana dashboard, gate failure example,
  canary ramp, rollback, audit-chain verification.
- `SUBMISSION.md`.
