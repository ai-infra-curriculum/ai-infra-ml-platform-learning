# Project 04 — Step-by-Step Build Guide

## Phase 0 — Setup (1-2 hours)

Local cluster + PostgreSQL + S3-compatible storage (MinIO).
Project skeleton:

```
project-04-model-registry/
├── api/             # FastAPI registry service
├── sdk/             # Python client
├── cli/             # smartrecs CLI
├── gates/           # gate evaluator (reusable from project-03)
├── deploy/
├── tests/
└── Makefile
```

## Phase 1 — Data model + registry API (10-12 hours)

1. Define the schema per `architecture.md` §Data model.
2. Implement migrations (alembic or equivalent).
3. CRUD endpoints for `Model` and `ModelVersion`.
4. Search + filter on the list endpoint.
5. Versioning: registrations are immutable; attempts to update
   a `ModelVersion` return 405.

## Phase 2 — Signature integration (5-6 hours)

1. SDK signs on registration via Cosign keyless (OIDC).
2. Signature blob stored in S3 alongside the artifact; URI
   stored in DB.
3. Verification function: pulls the signature, verifies
   against Sigstore root, checks the OIDC subject matches the
   expected workflow.

## Phase 3 — Promotion gates (10-12 hours)

1. Gate definitions (YAML).
2. Programmatic gate evaluator (reuse the safe expression
   parser from project-03 if both are built).
3. Human approval gate:
   - Promotion API call returns 202 with the gate's pending
     state.
   - `POST /v1/promotions/{id}/approve` records the approval.
4. State machine: enforce valid transitions.
5. Failed-gate response includes the failing gate's name and
   the failing condition's value.

## Phase 4 — Deployment + rollout strategies (12-14 hours)

1. `Deployment` CRUD.
2. Rolling: update the Kubernetes Deployment object,
   `kubectl rollout status` semantics.
3. Blue-green: bring up a new ReplicaSet with full capacity;
   swap service selectors.
4. Canary: bring up new ReplicaSet; configure ingress to
   split traffic; expose ramp endpoint.
5. Shadow: bring up new ReplicaSet that receives mirrored
   requests; log comparison data; never reaches client.

For the capstone, you don't need to implement the actual
traffic-splitting layer (use Istio or a service mesh in the
deploy/ manifests). Focus on the registry's role: tracking
which strategy is in use + the current state.

## Phase 5 — Rollback (4-5 hours)

1. API endpoint + CLI command.
2. Logic: find the most recent `Production` version other
   than the current one; deploy that as a rollback strategy.
3. Audit-chain entry required (with reason field).

## Phase 6 — Lineage (7-8 hours)

1. `lineage_edges` table.
2. SDK auto-populates edges on registration (training-run
   ID, dataset versions, etc.).
3. Forward + reverse query endpoints.
4. Recursive CTE for multi-hop traversal.

## Phase 7 — Multi-tenancy (4-5 hours)

1. Tenant scope on all queries.
2. Cross-tenant share mechanism.
3. Test: tenant A's queries don't return tenant B's models.

## Phase 8 — Audit + observability (5-6 hours)

- Audit-chain integration on every state transition + share +
  rollback.
- Prometheus metrics per the requirements file.
- Grafana dashboard.

## Phase 9 — Testing + docs (6-8 hours)

Full acceptance demo + documentation.

## Time-budget recap

| Phase | Hours |
|---|---|
| 0 — Setup | 1-2 |
| 1 — Data model + API | 10-12 |
| 2 — Signature | 5-6 |
| 3 — Gates | 10-12 |
| 4 — Deployment + rollout | 12-14 |
| 5 — Rollback | 4-5 |
| 6 — Lineage | 7-8 |
| 7 — Multi-tenancy | 4-5 |
| 8 — Audit + observability | 5-6 |
| 9 — Testing + docs | 6-8 |
| **Total** | **~70** |

## Common pitfalls

- **Mutable `ModelVersion`**: never allow re-registration
  under the same version. Compliance pain.
- **Skipping signature verification at deployment time**: if
  you only verify at registration, an attacker who can write
  to the S3 bucket can swap a clean artifact for a poisoned
  one between registration and deployment.
- **No reason field on rollback**: every rollback should
  capture *why* — incident report, regression, etc.
- **Treating shadow as canary**: shadow's output never reaches
  clients. Confusing them produces silent customer impact.

## When you're done

The registry is the system of record for production models.
Every model in production has a documented lineage, a
verified signature, recorded promotion approvals, and a known
rollback target.

This is the management layer the rest of the platform stack
ties into.
