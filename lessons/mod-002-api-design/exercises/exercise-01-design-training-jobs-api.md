# Exercise 01: Design a Training-Jobs API

## Objective

Produce a complete OpenAPI 3.1 spec for a training-jobs submission API.

## Scenario

ML platform users submit training jobs via:
- `POST /v1/training-jobs` — submit
- `GET /v1/training-jobs/{id}` — status
- `GET /v1/training-jobs` — list (with filters + pagination)
- `DELETE /v1/training-jobs/{id}` — cancel
- `POST /v1/training-jobs/{id}:retry` — retry

## Required design choices

- Idempotency key on POST submission
- Cursor pagination on list (not offset)
- Status enum with documented state machine
- Multi-tenant: tenant header `X-Tenant-Id` required on all requests
- Per-tenant rate limiting documented in spec

## Deliverables

1. `openapi.yaml` — complete spec; `openapi-generator-cli validate` passes
2. `STATE_MACHINE.md` — diagram of training-job states + transitions
3. `DESIGN_DECISIONS.md` — rationale for each non-obvious choice (≤ 1500 words)

## Acceptance

- Spec validates
- Codegen produces a working Python SDK (`openapi-generator-cli generate -g python`)
- Submit + status flow works against a mock server (Prism or similar)
