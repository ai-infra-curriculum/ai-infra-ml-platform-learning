# Exercise 03: Implement FastAPI Server from Spec

## Objective

Stand up a FastAPI implementation of the v1 training-jobs API. Validate
against your spec.

## Requirements

- Pydantic models generated from the spec (or hand-written matching it)
- In-memory store backing the API (no DB needed)
- Idempotency: same `Idempotency-Key` returns the same response
- Pagination: cursor-based
- Rate limit middleware: 60 req/min per tenant
- `/metrics` endpoint with prometheus_client
- Tests: pytest + httpx exercising every endpoint
- `schemathesis run openapi.yaml --base-url http://localhost:8000` passes

## Acceptance

- All tests pass
- Schemathesis property tests pass
- Generated Python SDK can successfully submit + poll a training job
- p95 latency < 50ms at 100 RPS in local load test

## Companion

[engineer-solutions/mod-101 ex-08 (production-model-serving)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-101-foundations/exercise-08-production-model-serving) for FastAPI factory + lifespan + rate-limit patterns.
