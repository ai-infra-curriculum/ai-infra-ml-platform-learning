# Lecture 01: REST vs gRPC vs Async (and when to use each)

## Why this matters

Platform APIs are read more often than they're written. Every consuming team
spends weeks integrating against your API; every wrong choice you make becomes
ten teams' tech debt. The framing decision — REST vs gRPC vs async — sets the
ceiling on developer experience for the lifetime of the platform.

## REST: the default

Use REST when:
- Callers are humans or scripts as much as services
- You want broad tooling (curl, Postman, browser dev tools)
- The shape is resource-centric (training jobs, models, features)
- Latency is acceptable in tens of milliseconds, not microseconds

REST shines for self-service surfaces. The training-job submission endpoint is
REST; the model registry is REST; the feature-lookup API for low-volume
serving is REST.

### Conventions to keep

- Plural resource names: `/v1/training-jobs`, not `/v1/training-job`
- HTTP verbs map to lifecycle: GET (read), POST (create), PUT (replace), PATCH (update), DELETE (delete)
- Status codes mean things: 2xx = OK, 4xx = your fault, 5xx = our fault
- Hypermedia where appropriate (the response includes the URLs of related resources)

## gRPC: when latency or schema rigor matters

Use gRPC when:
- Service-to-service traffic dominates (no human callers)
- Per-call latency budget is < 10ms
- The schema is stable enough to commit to .proto definitions
- All callers can adopt a generated client

gRPC excels for inner-platform traffic (feature store serving, model registry
lookups during prediction). The protobuf schema is a contract that wins
arguments. Bidirectional streaming + flow control is free.

The cost: harder to debug from a browser; protobuf is not human-eyeballable;
ecosystem outside of Go/Java/Python is uneven.

## Async: for long-running operations

Use async when:
- The operation takes longer than the caller wants to wait (training jobs, batch jobs)
- The result might be consumed by multiple downstream parties
- The system needs to survive caller disconnects

Two shapes:
- **Webhook callback**: caller registers a URL; server POSTs result there
- **Pub/sub event**: server publishes to Kafka / SNS; subscribers self-discover

Pair with a synchronous "submit" call: REST POST returns 202 + a job-id; the
job runs async; the caller polls a status endpoint or listens for the webhook.

## The hybrid pattern (the most common)

Most ML platforms end up with three surfaces:

1. **External REST API** for humans + scripts (training-jobs, registry, etc.)
2. **Internal gRPC** for hot-path service-to-service calls (feature lookup, model serve)
3. **Async event stream** for status updates + workflow coordination (Kafka topic per resource type)

These coexist without conflict if you draw the boundary right.

## Anti-patterns

- **CRUD-via-RPC**: gRPC where REST would do, just because the team likes protobuf
- **Async-everywhere**: every operation is fire-and-forget, callers can't tell what's running
- **REST-only for hot path**: trying to do 100K/s of feature lookups over JSON
- **Multiple sources of truth**: an event stream + a REST API that disagree about state

## Reading

- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [gRPC vs REST: When to use each (Cloudflare)](https://www.cloudflare.com/learning/cloud/what-is-grpc/)
