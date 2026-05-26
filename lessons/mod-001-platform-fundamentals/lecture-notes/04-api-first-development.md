# Lecture 04: API-First Development

## Table of Contents

1. [Introduction](#introduction)
2. [What "API-First" Means](#what-api-first-means)
3. [API Styles for ML Platforms](#api-styles-for-ml-platforms)
4. [Designing a Platform API](#designing-a-platform-api)
5. [Versioning Strategies](#versioning-strategies)
6. [Backward Compatibility](#backward-compatibility)
7. [Deprecation and Sunset](#deprecation-and-sunset)
8. [Documentation Requirements](#documentation-requirements)
9. [Contract Testing](#contract-testing)
10. [Idempotency, Errors, and Pagination](#idempotency-errors-and-pagination)
11. [Summary](#summary)

---

## Introduction

Of all the design choices a platform team makes, the API design choices are the ones whose mistakes are hardest to undo. Once tenants have written code against your API, every breaking change has a cost — measured in their time, their tooling, and their patience. A platform that ships a clean v1, never breaks compatibility, and rarely deprecates anything will accumulate trust. A platform that ships a v1, breaks it three months later, and offers no migration path will accumulate cynicism.

This chapter is about how to write APIs you can live with for years. It is deliberately written at the level of *principles and patterns*, not specific frameworks (FastAPI, gRPC, etc.) — those come in later modules.

By the end you should be able to:

- Articulate what "API-first" means as a development practice, and distinguish it from "API-also."
- Compare REST, gRPC, GraphQL, and SDK-only approaches and pick a primary surface for an ML platform.
- Apply at least three versioning strategies and explain when each is appropriate.
- Identify the dimensions along which a change can be backward-compatible vs breaking.
- Outline a deprecation policy that respects user trust.
- Read an OpenAPI / protobuf schema and reason about whether a proposed change is breaking.

---

## What "API-First" Means

### Two definitions

"API-first" gets used in two related senses:

1. **API-first development**: the API contract is designed *before* the implementation. You write the OpenAPI spec or the protobuf, get sign-off, *then* implement.
2. **API-first product**: every capability of the platform is exposed through APIs *before* it's exposed through UIs. The UI is a consumer of the API like any other.

Both meanings are valuable, and they reinforce each other. A platform team practicing (1) without (2) ends up with elegant APIs that the UI bypasses (defeating the purpose). A team practicing (2) without (1) ends up reverse-engineering its own API every release.

For ML platforms, *both* meanings are critical. The "API-first product" idea matters especially because most ML users interact with the platform programmatically (via SDK or CLI), not through a web UI. If your APIs don't expose every feature, your most important users — the data scientists in Python — can't use that feature.

### What it gets you

- **Parallel work.** The API contract is the boundary between teams. Frontend and backend can develop simultaneously against the same contract.
- **Automation.** Once the API is a contract, you can auto-generate client SDKs, documentation, mock servers, and contract tests.
- **Reasoning at a higher level.** Design discussions happen at the API level (what the platform *promises* to users) rather than the implementation level (how the database is structured).
- **Multiple consumer surfaces.** A well-designed API can be consumed by a CLI, a Python SDK, a web UI, an Airflow operator, a Slack bot, etc. — without the platform team building each separately.

### What it costs

- **Upfront design effort.** "Just code it" feels faster in the short run. API-first slows you down at the start.
- **Spec discipline.** Specs (OpenAPI, protobuf) take work to write and update. If the team isn't disciplined, the spec falls out of date.
- **Negotiation overhead.** When the contract is explicit, every change is a negotiation. This is healthy but it has a cost.

For early-stage platforms with one or two consumers and rapid iteration, full API-first discipline may be too heavy. For mature platforms with many consumers, API-first is essential. Most platforms transition somewhere in their Phase 1 → Phase 2 (see chapter 02).

### The "API as the durable thing" principle

A useful framing: when a tenant integrates with your platform, *they are integrating with your API, not your implementation*. The implementation is allowed to change. The API is the promise.

If you internalize this, you'll find yourself making different design choices: hiding more, exposing less, refusing to leak internals into the API, treating the API surface as load-bearing.

---

## API Styles for ML Platforms

There are several ways to expose APIs, each with tradeoffs.

### REST (HTTP + JSON)

The default. HTTP verbs (GET, POST, PUT, PATCH, DELETE) on resource-shaped URLs (`/training-jobs/123`).

Strengths:
- Universal tooling (every language, every browser, every monitoring system).
- Easy to debug with curl/Postman.
- Easy to document with OpenAPI.
- Easy to cache (GET is cacheable).

Weaknesses:
- No native streaming (you can use SSE or chunked encoding, but it's awkward).
- Schema is loose by default. You have to opt into strict typing (via OpenAPI + validators).
- Pagination, filtering, and partial updates have to be designed; there's no canonical answer.

For ML platforms, REST is the right default for the *control plane* — submitting jobs, registering models, querying metadata. It's perfectly serviceable.

### gRPC (HTTP/2 + protobuf)

Strongly typed RPC over HTTP/2.

Strengths:
- Strict schemas (protobuf) with auto-generated client code.
- Native streaming (server-side, client-side, bidirectional).
- Fast on the wire (binary).
- Good when your platform has many internal services calling each other.

Weaknesses:
- Harder to debug (binary protocol).
- Browsers can't speak it directly; you need grpc-web or a gateway.
- Steeper learning curve for SDK consumers.

For ML platforms, gRPC is often the right choice for *high-throughput data planes* — feature serving for online inference, distributed training coordination — and a reasonable choice for internal service-to-service RPC.

### GraphQL

A query language for APIs. Clients specify exactly what fields they want.

Strengths:
- Avoids over-fetching (clients only get what they ask for).
- Single endpoint; flexible queries.
- Strong introspection — clients can discover the schema.

Weaknesses:
- Server complexity (query planner, depth limits, complexity limits).
- N+1 query problems are easy to create.
- Caching is harder (every query is unique).
- Authorization is more complex (you have to authorize each field, not just each endpoint).

GraphQL is occasionally used for ML platform UIs (dashboards that need flexible queries against many resources). It's rarely the right choice for the primary platform API.

### Python SDK as the surface (the Metaflow pattern)

Some platforms expose their primary user surface as a *Python SDK*, with the underlying API being an implementation detail.

Strengths:
- Optimized for the dominant user persona (data scientists in Python).
- Can hide the API entirely, letting the platform team evolve it without breaking users.
- Excellent IDE support (autocomplete, type hints, docstrings).

Weaknesses:
- Non-Python consumers (Airflow, the web UI, Java services) have to go around the SDK.
- Versioning the SDK *and* the underlying API requires care.
- The SDK becomes a critical-path dependency for users.

Most modern ML platforms ship *both* an SDK and a documented REST/gRPC API. The SDK is the front door for typical users; the API exists for the cases the SDK doesn't cover.

### Recommendation: REST control plane + SDK front door

For a new ML platform, the most common shape is:

- **REST API** for the control plane (resource CRUD, status queries).
- **gRPC** for any high-throughput data plane (online feature serving, online inference).
- **Python SDK** as the primary user-facing surface, wrapping the REST API.
- **CLI** as a thin wrapper over the SDK.
- **Web UI** that reads from the same APIs.

This shape is shown below:

```
            ┌────────────────────────────────────────┐
            │            User-facing surface         │
            │ Python SDK   CLI   Web UI   Airflow op │
            └────────┬────────┬───────┬─────────┬────┘
                     │        │       │         │
                     └────────┴───┬───┴─────────┘
                                  │
                       ┌──────────▼───────────┐
                       │  REST control plane  │
                       │  (OpenAPI-specced)   │
                       └──────────┬───────────┘
                                  │
                  ┌───────────────┼─────────────────┐
                  │               │                 │
            ┌─────▼─────┐ ┌───────▼──────┐ ┌────────▼────────┐
            │ Registry  │ │ Training svc │ │ Inference fleet │
            └───────────┘ └──────────────┘ └─────────────────┘
                                  │
                          ┌───────▼─────────┐
                          │ gRPC data plane │
                          │ (proto-specced) │
                          └─────────────────┘
```

This is not the only valid shape — Metaflow's design is materially different — but it is the most common.

---

## Designing a Platform API

Once you've picked a style, the next question is *what the API looks like*. Here are the rules of thumb that recur in platform-API design.

### Model the domain, not the implementation

A common mistake: the API mirrors the database tables. Don't do this. The database is an implementation detail; the API is the contract. The right API shape is determined by what *users want to express*, not what your storage layout happens to be.

Bad (mirrors storage):

```
POST /create-model-row
GET  /select-training-jobs-where?status=running
```

Good (models user intent):

```
POST /v1/models
GET  /v1/training-jobs?status=running
```

### Use nouns, not verbs (REST)

Resources have nouns. Actions on them use HTTP verbs.

```
POST /v1/training-jobs           # create
GET  /v1/training-jobs           # list
GET  /v1/training-jobs/{id}      # read
PATCH /v1/training-jobs/{id}     # update (partial)
DELETE /v1/training-jobs/{id}    # delete
```

For actions that don't map to CRUD (e.g., "cancel a running job"), use a sub-resource:

```
POST /v1/training-jobs/{id}/cancel
```

Don't:

```
POST /v1/cancelTrainingJob?id=123
```

This isn't dogma; sometimes a non-CRUD verb is the only way to express the action sensibly. But default to nouns.

### Hierarchical URLs reflect ownership

A model belongs to a project. A training run produced a model. Reflect that in URLs:

```
GET /v1/projects/{project}/models
GET /v1/projects/{project}/models/{model}/versions
GET /v1/projects/{project}/training-runs/{run}/artifacts
```

This makes the URL itself convey scope and ownership. It also makes per-project authorization easier to express.

But don't go too deep. Three levels of nesting is usually plenty. Beyond that, top-level resources with foreign-key fields read better.

### Resource identifiers

Identifiers should be:

- **Opaque to the user.** Don't expose internal database IDs that users could collide with. Use URL-safe IDs (UUIDs, base32 hashes, etc.).
- **Stable.** Once issued, never change.
- **Globally unique within the platform.** Don't reuse IDs across resource types unless you're sure you won't accidentally cross them.
- **Human-readable when possible.** `model-abc-2025-05` is more useful than `7f1c8b3e-9d0a-4c52-...`. Some platforms use a hybrid — a short opaque suffix + a human-readable prefix.

### Status fields and lifecycle

Resources with lifecycles (training jobs, deployments, runs) should have an explicit `status` field with a documented enum of states. Don't use booleans for state (`is_running`, `is_failed`, `is_complete`) — they don't compose into a state machine.

Good:

```json
{
  "id": "run-abc123",
  "status": "succeeded",
  "phase": "completed",
  "created_at": "2026-05-20T10:00:00Z",
  "started_at": "2026-05-20T10:00:05Z",
  "completed_at": "2026-05-20T11:23:48Z"
}
```

The enum:

```
pending → running → succeeded
                 → failed
                 → cancelled
```

Document the state machine. Diagrams help.

### Pagination

Long lists must be paginated. Two common patterns:

**Offset/limit** (the easy one):

```
GET /v1/training-jobs?offset=0&limit=50
GET /v1/training-jobs?offset=50&limit=50
```

Weakness: as items are inserted or deleted while paginating, you may skip or duplicate.

**Cursor-based** (the right one):

```
GET /v1/training-jobs?limit=50
→ returns items + next_page_token

GET /v1/training-jobs?page_token=xyz&limit=50
→ next page
```

The token is opaque (server-decoded). The pattern is stable under concurrent inserts/deletes.

For ML platform list APIs, default to cursor-based pagination from v1.

### Filtering and sorting

Common patterns:

```
GET /v1/training-jobs?status=running
GET /v1/training-jobs?owner=team-alpha
GET /v1/training-jobs?created_after=2026-05-01
GET /v1/training-jobs?sort=-created_at
```

Don't overdo it. Don't ship arbitrary SQL-like filtering ("WHERE foo > bar AND baz IN (...)") as your filtering language — it's a security and performance nightmare. Ship simple named filters that cover known use cases.

### Side effects: prefer async

Operations that take more than a few hundred milliseconds should be asynchronous. The POST creates a *job* that the client polls (or subscribes to via SSE / webhook).

```
POST /v1/training-runs
→ 202 Accepted
   Location: /v1/training-runs/run-abc123
   {"id":"run-abc123","status":"pending",...}

GET /v1/training-runs/run-abc123
→ 200 OK
   {"id":"run-abc123","status":"running",...}
```

A synchronous POST that blocks until training completes is wrong for two reasons: HTTP clients will time out, and the platform can't easily handle the load. Make async the default.

---

## Versioning Strategies

Every API needs a versioning strategy. The choice you make in v1 you live with forever.

### URL versioning (the most common)

```
/v1/training-jobs
/v2/training-jobs
```

The major version is in the URL path. Clients explicitly choose which version they want.

Pros:
- Visible. You always know what version you're calling.
- Easy to route (different versions to different services / handlers).
- Easy to deprecate.

Cons:
- URL changes propagate everywhere.
- Encourages "v1 → v2 → v3" thinking with hard breaks rather than smooth evolution.

This is the recommended default for ML platform REST APIs.

### Header versioning

```
GET /training-jobs
Accept: application/vnd.platform.v1+json
```

Pros:
- URLs stay stable across versions.
- Clean separation of resource and version.

Cons:
- Less visible. Easy to mis-set the header.
- Tooling (curl, browsers) is awkward.
- Caching keys must include the header.

This is more common in OAS-shaped enterprise APIs.

### Query parameter versioning

```
GET /training-jobs?api_version=2
```

The minor option. Easy to default. Less common in serious platforms because it can be silently dropped by intermediaries.

### Date-based versioning (the Stripe pattern)

```
Stripe-Version: 2025-04-30
```

The client passes the date of the API "snapshot" they want. The server has every snapshot encoded as a sequence of transformations.

Pros:
- Allows continuous evolution rather than discrete v1/v2 breaks.
- Existing clients keep working as long as you maintain old snapshots.
- Excellent for SaaS APIs.

Cons:
- Implementation complexity. You need a version-transformation framework.
- Not free; Stripe famously has a large team that maintains compatibility.

For most internal ML platforms, URL versioning is the right answer. Date-based versioning is brilliant but expensive; consider it only if you have multiple major external consumers.

### Semantic versioning for SDKs

SDKs are software libraries. They should follow [semver](https://semver.org/):

- MAJOR: breaking changes.
- MINOR: new functionality, backward-compatible.
- PATCH: bug fixes, backward-compatible.

A platform's SDK at version `2.7.3` can be used by a client that pins to `2.x.x` (any 2.x version) safely. Breaking changes require a new major version.

The SDK version may differ from the API version. SDK 2.x might wrap API v1 today, API v2 tomorrow. The mapping should be documented.

### How often to break

A reasonable cadence for internal ML platforms:

- **Patch / minor changes**: continuously, no announcement needed.
- **Backward-compatible additions** (new endpoints, new optional fields): release notes, weekly cadence.
- **Major breaking changes**: at most every 6-12 months, with at least 6 months of overlap (v1 and v2 both supported).

External-facing platforms (SaaS) typically slower — once a year at most, often less. Trust accumulates from stability.

---

## Backward Compatibility

A change is **backward-compatible** if existing clients continue to work without modification. A change is **breaking** otherwise. Many platform engineers underestimate the breadth of what "breaking" can mean.

### Changes that are *almost always* backward-compatible

- Adding a new endpoint.
- Adding a new optional request field.
- Adding a new field to a response.
- Adding a new enum value (but see warning below).
- Adding a new error code (but see warning below).
- Adding new optional query parameters.
- Relaxing a constraint (e.g., field is now optional that was required).

### Changes that are *almost always* breaking

- Removing an endpoint.
- Removing a field from a response.
- Renaming a field.
- Changing a field's type (string → int).
- Changing the semantics of a field (without renaming).
- Making an optional field required.
- Tightening a constraint.
- Changing the structure of a response object.
- Removing an enum value (clients may have explicit handling for it).
- Removing an error code (clients may switch on it).

### Changes that are *subtly* breaking

These bite people repeatedly:

- **Adding an enum value.** A client that exhaustively matches on the enum (case `RUNNING:`, case `SUCCEEDED:`, case `FAILED:`, default: error) will fail when the platform returns a new value. *Mitigation*: design APIs to make exhaustive matching opt-in (e.g., always include a "default" or "other" guidance in docs).
- **Adding a field to a response that clients then start depending on.** Not breaking when added, but the next time you remove or change it, you break clients that came to depend on it.
- **Changing default behavior.** "When `region` is omitted, we used to default to `us-east-1`; now we default to the user's home region." Clients depending on the old default break.
- **Loosening validation.** A field that used to reject empty strings now accepts them. Clients that assumed non-empty break.
- **Adding required headers.** "All requests must now include `X-Request-ID`." Clients that didn't include it break.
- **Changing rate limits or quotas.** Lowering a limit silently breaks clients that hit it.

A useful rule: **even apparently-additive changes can break clients if those clients are written defensively.** Treat *any* change to behavior as a potential break, and announce it.

### Tolerant readers, strict writers

A pattern from Postel's law: clients should be *tolerant* readers (ignore unknown fields), and servers should be *strict* writers (only send documented fields).

If clients are tolerant readers, they don't break when servers add new fields. If servers are strict writers, the docs are honest.

Encourage this discipline in SDK design. If your SDK uses strict deserialization (failing on unknown fields), you've made every additive change to the API into a breaking change for SDK users. Use lenient deserialization in clients.

### Schema evolution rules (protobuf)

protobuf has explicit rules:

- Don't change field numbers.
- Don't change field types (some safe conversions are allowed: int32 ↔ int64, but not int ↔ string).
- Don't reuse a field number that you removed without marking it reserved.
- New fields default to "unset," which is backward-compatible.

If you follow these rules, protobuf gives you backward compatibility almost for free. If you violate them, you've created a silent corruption bug.

---

## Deprecation and Sunset

Deprecation is the process of telling users "this API is going away, please migrate." Sunset is the act of actually removing it.

### A deprecation policy

A good deprecation policy has several elements.

1. **Public communication.** A new deprecation is announced in release notes, on a dedicated "deprecations" page, and ideally in the API response itself (via a `Deprecation` header or warning field).
2. **A timeline.** "This endpoint is deprecated and will be removed on 2027-01-01." Give users a long runway — at least 6 months for internal platforms, 12+ for external.
3. **A migration path.** "Use `POST /v2/training-jobs` instead. See the migration guide at ..." A deprecation without a migration path is a betrayal.
4. **Migration tooling where possible.** If you can write a script that converts old client code to new, do.
5. **Reminders before sunset.** A month out, a week out, a day out. Make it impossible for users to be surprised.
6. **A grace period after sunset.** When the endpoint is finally removed, return a clear error explaining what happened ("HTTP 410 Gone: this endpoint was deprecated on 2026-07-01 and removed on 2027-01-01. See [docs]."). Don't return 404 — that looks like a bug.

### The `Deprecation` and `Sunset` headers

[RFC 9745](https://datatracker.ietf.org/doc/rfc9745/) (Deprecation) and [RFC 8594](https://datatracker.ietf.org/doc/rfc8594/) (Sunset) define HTTP headers for this:

```
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Link: <https://docs.platform.example.com/migrate-v1-to-v2>; rel="successor-version"
```

Clients (or their SDKs) can detect these headers and surface warnings.

### The hardest deprecations

Some deprecations are politically hard:

- A team adopted v1 of your API two years ago and has not touched their integration since. They will be furious when you tell them to migrate.
- A team's code that uses v1 is owned by a person who left the company. There is no one to migrate.
- A team migrated to v2, but their dashboards still use v1. They forgot.

The mitigations:

- *Long deprecation periods* (12+ months for important APIs).
- *Migration partner programs* — the platform team allocates engineers to help large customers migrate.
- *Usage tracking* — you know who is still calling v1, by tenant, and can reach out personally.
- *Carrot-and-stick* — sometimes the only way to get the last 10% to migrate is to start charging more for v1 access, or to introduce intentional artificial latency.

### Never silently change behavior

The cardinal sin: changing behavior without deprecation. A client calls `POST /v1/jobs` and the result is now different — different shape, different defaults, different semantics. This destroys trust faster than any other failure.

If the change is truly necessary and you can't deprecate, at minimum *announce it* and give clients a way to opt out (a header, a flag) for at least one cycle.

---

## Documentation Requirements

We discussed documentation as a DX surface in chapter 02. Specific to APIs, the documentation requirements are:

### Reference documentation

For every endpoint:

- HTTP method, path, summary.
- Request schema (with field-by-field descriptions, types, defaults, required/optional).
- Response schema (same).
- Error responses (with error codes and example bodies).
- Rate limits.
- Authentication and authorization requirements.
- Idempotency behavior.
- Versioning notes.

Auto-generate this from the OpenAPI spec. Don't write it by hand — it will drift.

### Conceptual documentation

For every major concept:

- What it is.
- Why it exists.
- Its lifecycle.
- Its relationship to other concepts.

Examples for an ML platform: "Training Run", "Model Version", "Deployment Environment", "Feature View", "Experiment Tracking".

### Recipes / tutorials

For common end-to-end workflows:

- "Train a model and deploy it to prod"
- "Compare two models against held-out data"
- "Roll back a deployment"
- "Replay last week's predictions on the new model"

Recipes show users how to compose the API surface to do real work.

### Worked examples

For every endpoint, at least one realistic example:

```bash
curl -X POST https://api.platform.example.com/v1/training-jobs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "churn-experiment-7",
    "code": {
      "repo": "github.com/example/churn-model",
      "ref": "feature/v2"
    },
    "dataset": "warehouse://customers/snapshot/2026-05-15",
    "resources": {
      "gpus": 2
    }
  }'

# Response:
# {
#   "id": "run-abc123",
#   "status": "pending",
#   "created_at": "2026-05-22T22:00:00Z",
#   ...
# }
```

Also examples in the SDK:

```python
from platform_sdk import Platform

client = Platform()
run = client.training_jobs.create(
    name="churn-experiment-7",
    code_repo="github.com/example/churn-model",
    code_ref="feature/v2",
    dataset="warehouse://customers/snapshot/2026-05-15",
    gpus=2,
)
print(run.id)  # run-abc123
```

Same operation, two surfaces. Both documented.

---

## Contract Testing

A contract is a promise. Contract testing makes sure both sides keep their promise.

### Provider tests

The platform team tests that the platform serves the OpenAPI spec correctly:

- Every endpoint in the spec exists.
- Every endpoint returns responses matching the spec's schema.
- Every documented error condition can be triggered.

Tools: [Schemathesis](https://schemathesis.readthedocs.io/) (Python), [Dredd](https://dredd.org/) (multi-language), [Pact](https://pact.io/) for consumer-driven contracts.

### Consumer tests

The platform's *clients* test that their use of the API matches the spec:

- The SDK uses only documented endpoints.
- The SDK handles all documented error responses.
- The SDK does not depend on undocumented response fields.

### Why this matters

Without contract tests:

- The platform team can accidentally break the spec without realizing.
- Clients can accidentally start depending on undocumented behavior.
- Major version upgrades require manual verification.

With contract tests:

- The spec is enforced as code.
- Drift between spec and implementation is caught in CI.
- Migration tooling is more reliable.

For ML platforms, contract testing is high-leverage. Do it from v1.

---

## Idempotency, Errors, and Pagination

Three crosscutting concerns worth their own discussion.

### Idempotency

An operation is **idempotent** if calling it twice has the same effect as calling it once. GET is naturally idempotent (reading something twice doesn't change it). POST is naturally *not* idempotent (creating two things creates two things).

But POST sometimes *needs* to be idempotent — e.g., when the client times out and retries, you don't want two training jobs created.

The pattern: **idempotency keys**.

```
POST /v1/training-jobs
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
{...request body...}
```

The platform keeps a short-term cache of "we processed key X with result Y." If the same key comes in again, the platform returns the same result without re-creating.

Stripe pioneered this pattern for their API; it's a well-understood convention. Implement it for any non-idempotent mutating endpoint.

### Errors

Error responses should:

- **Use the right HTTP status code.** 4xx for client errors, 5xx for server errors. Don't return 200 with `{"error": "..."}` — that breaks every HTTP-aware tool.
- **Have a stable error code in the body.** Status codes are too coarse. Add an application-level error code:

```json
{
  "error": {
    "code": "quota_exceeded",
    "message": "Training job request exceeds the team's GPU quota.",
    "details": {
      "requested_gpus": 8,
      "remaining_quota": 2
    },
    "docs": "https://docs.platform.example.com/errors/quota-exceeded"
  }
}
```

- **Be helpful, not generic.** "Bad request" is useless. "Field `gpus` must be between 1 and 16 (got 100)" is useful.
- **Be safe.** Don't leak internal details (stack traces, database errors, internal IPs) in error bodies served to users. Save those for logs.
- **Have stable codes.** Once an error code is in the wild, don't change its meaning. Add new codes for new conditions.

### Pagination, again

We covered this earlier. The additional points:

- Always include a `next_page_token` (or equivalent), even on the last page (where it's null/empty). That way clients can write a simple "while has_next" loop.
- Include a `total_count` *if* you can compute it cheaply. Don't promise an exact count if the underlying store doesn't make it free.
- Default page size sensibly. 50-100 is typical. Cap the maximum (e.g., 500) so a single request can't pull a million records.

These details are unsexy. They are also the difference between an API that's pleasant to use and one that's a constant low-grade frustration.

---

## Summary

- **API-first** means designing the contract before the implementation *and* exposing every feature through APIs before UIs. Both meanings matter.
- For ML platforms, the most common shape is: **REST for control plane, gRPC for high-throughput data plane, Python SDK as the user-facing surface, CLI / web UI / integrations as additional consumers**.
- Good API design **models the domain, not the implementation**. Use nouns for resources, verbs for actions. Hierarchical URLs reflect ownership.
- **Status fields** with documented enums are better than booleans for stateful resources. Use **cursor-based pagination**. Default mutating operations to **async** with status polling.
- **URL versioning (`/v1/`, `/v2/`)** is the default for ML platform REST APIs. Reserve major version bumps for genuinely breaking changes. SDKs follow **semver** independently.
- **Backward compatibility** is broader than people expect. Even additive changes can break defensive clients. Practice "**tolerant readers, strict writers**" and "**add-only**" evolution where possible.
- **Deprecation policy** matters: announcement, timeline, migration path, reminders, headers, grace period after sunset. Never silently change behavior.
- **Documentation** has four layers: reference (auto-generated), conceptual, recipes, examples. All of them matter.
- **Contract testing** prevents drift between spec and implementation. Adopt it from v1.
- **Idempotency keys** make retries safe. **Stable error codes** make programmatic error handling possible. **Sensible defaults** for pagination prevent footguns.

In the next chapter, we zoom out from API design to overall **platform architecture patterns** — how the pieces of a platform are organized as a system.

---

## Reflection Questions

1. Imagine you are designing the `POST /v1/training-jobs` endpoint for an ML platform. List five fields the request body must have, five that should be optional, and one that should *not* be exposed (it should default internally). Defend each choice.
2. Your platform's v1 API has been live for 18 months. A senior data scientist has been calling `GET /v1/runs/{id}` and depending on an undocumented field called `_internal_state`. You want to remove it. Walk through the deprecation steps.
3. A new junior on the platform team proposes adding a `magic_action: bool` field to the training job request. Setting it to `true` enables a 20% faster training path that's currently in beta. How do you redesign this proposal to be a *good* API change?
4. Should an ML platform's "list training jobs" endpoint return all jobs, or only the caller's tenant's jobs, by default? What if the caller is a platform admin?

---

## Further Reading

- **["API Design Patterns"](https://www.manning.com/books/api-design-patterns) by JJ Geewax.** A long book of recurring API design patterns. Densely useful.
- **[Google API Design Guide](https://cloud.google.com/apis/design).** Google's internal API design conventions, made public. Heavy but comprehensive.
- **["Stripe's API: An Update"](https://stripe.com/blog/api-versioning) (Stripe Engineering Blog).** The famous date-based versioning writeup.
- **[OpenAPI Specification](https://swagger.io/specification/).** The spec itself.
- **[gRPC documentation](https://grpc.io/docs/).** For when you need streaming or strict types.
- **[RFC 9745 (Deprecation)](https://datatracker.ietf.org/doc/rfc9745/) and [RFC 8594 (Sunset)](https://datatracker.ietf.org/doc/rfc8594/).** Short, worth reading.

In the next chapter, we'll look at how the pieces of an ML platform fit together architecturally: microservices vs monolith, event-driven patterns, plugin systems, extension points.
