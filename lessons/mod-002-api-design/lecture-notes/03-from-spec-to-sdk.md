# Lecture 03: From OpenAPI Spec to Generated SDK

## Spec-first vs code-first

Two ways to get an OpenAPI spec:

- **Spec-first**: write the OpenAPI YAML by hand; generate server stubs + client SDKs from it
- **Code-first**: write the server code (FastAPI / Spring / etc.); generate the spec from annotations

Pick spec-first when:
- The API is a contract negotiated between teams
- You want to generate clients in multiple languages
- You want the spec reviewed before code is written

Pick code-first when:
- A single team owns both server + clients
- The server stack has excellent annotation support (FastAPI especially)
- You can tolerate occasional spec drift

ML platforms typically benefit from spec-first because the API is the cross-
team contract. The conversation about "should this be a header or a query
param" happens before someone has invested two weeks coding it the wrong way.

## Minimal spec structure

```yaml
openapi: 3.1.0
info:
  title: ML Platform API
  version: "1.0.0"
servers:
  - url: https://api.ml-platform.example.com/v1
paths:
  /training-jobs:
    post:
      summary: Submit a training job
      operationId: submitTrainingJob
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/TrainingJobRequest' }
      responses:
        '202':
          description: Accepted
          content:
            application/json:
              schema: { $ref: '#/components/schemas/TrainingJob' }
components:
  schemas:
    TrainingJobRequest:
      type: object
      required: [model_uri, dataset_uri]
      properties:
        model_uri: { type: string, format: uri }
        dataset_uri: { type: string, format: uri }
        gpu_count: { type: integer, minimum: 1, default: 1 }
    TrainingJob:
      type: object
      properties:
        id: { type: string }
        status: { type: string, enum: [pending, running, succeeded, failed] }
```

## Codegen

Generate clients from the spec:

```bash
openapi-generator-cli generate -i openapi.yaml -g python -o sdk/python
openapi-generator-cli generate -i openapi.yaml -g typescript-axios -o sdk/typescript
openapi-generator-cli generate -i openapi.yaml -g go -o sdk/go
```

CI publishes these clients to language package registries on every release.

## SDK quality

A generated SDK is acceptable. A *good* SDK adds:

- Retry + backoff for idempotent ops
- Authentication helper (OAuth flow / API key handling)
- Pagination iterator (transparent next-page fetching)
- Idempotency key generation
- Logging hooks
- Type hints / autocomplete that's pleasant to use

Most teams generate the low-level client + hand-write a thin "high-level" SDK
that wraps it.

## Contract testing

Spec-first lets you contract-test:
- Server-side: every endpoint's response is validated against the schema in tests
- Client-side: the SDK's response parsing is validated against the schema

Tools like Schemathesis (Python) generate property-based tests from your
spec — they find edge cases your handwritten tests missed.

## Pitfalls

- **Spec rot**: spec falls out of sync with server; consumers integrate against fiction
- **Over-modeled errors**: 50 different error response schemas; consumers can't handle them all
- **Polymorphic responses**: `oneOf` everything; clients struggle to use the SDK
- **No examples**: spec has no `examples:` blocks; consumers have no reference inputs

Fix spec rot with CI: every PR runs `schemathesis` against the spec + the
server. Fix the others with code review and standards.
