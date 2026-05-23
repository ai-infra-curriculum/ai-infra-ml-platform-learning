# Exercise 01: Design an API for an ML Resource Provisioning Service

## Objective

Design a REST API for a *resource provisioning service* that lets ML platform users request training resources (GPU pods) on-demand, query their status, and release them. You will produce an OpenAPI sketch, a state-machine diagram, and a design defense, then critique three alternative designs.

The point of this exercise is not to write working code; it's to internalize the *design moves* that distinguish a good platform API from a bad one. Most of the work happens in markdown and on a napkin.

## Learning Outcomes

By completing this exercise, you will:

- Apply the API design principles from Lecture 04 to a concrete scenario.
- Practice the *resource lifecycle* discipline (states, transitions, error states).
- Decide where to put **versioning**, **idempotency**, **pagination**, **filtering**, and **async behavior**.
- Compare your design against alternative shapes and articulate the tradeoffs.
- Write API-design rationale that a reviewer could engage with critically.

## Prerequisites

- Read Lecture 04 (API-First Development) in full.
- Skim Lecture 03 (Multi-Tenancy) — the API has to respect tenancy.
- Familiarity with REST conventions (HTTP verbs, status codes).
- Familiarity with YAML / JSON (for the OpenAPI sketch).
- Optional but useful: have skimmed an existing OpenAPI spec.

## Scenario

You are the lead platform engineer at "Aurelia AI," a fictional Series B company with ~30 ML engineers spread across 6 teams. The ML platform you are building uses Kubernetes underneath. Today, when a data scientist wants a GPU pod for an experiment, they DM you on Slack. You provision it by hand. You are tired.

You are going to expose a self-service API. The API lets a tenant:

1. **Request** a pod with specified resources (GPU type, GPU count, CPU, memory, expected lifetime).
2. **Query** the status of the request (pending, scheduling, running, expired, failed, cancelled).
3. **Connect** to the running pod (the platform returns connection info — kubeconfig context, SSH-able address, or a Jupyter URL, depending on what the requester asked for).
4. **List** their team's current and recent requests.
5. **Cancel** an active request.
6. **Extend** the lifetime of a running request (subject to quota).

Resources are *ephemeral* — the platform auto-releases pods when their lifetime expires. The default lifetime is 4 hours; the maximum is 24 hours.

Constraints:

- Each team has a **resource quota** (e.g., 8 GPUs total across active requests).
- Requests beyond quota are **rejected** at submit time with a clear error.
- The platform is **multi-tenant** — a tenant must not see another tenant's requests.
- The API must be **versioned** from day one.
- The API must support **idempotent retries** (a flaky network shouldn't create duplicate pods).
- The API is **asynchronous** — POST returns quickly with a handle; status comes from polling.
- The API exposes **status events** (so future clients can stream rather than poll).

## Deliverables

By the end of this exercise, you will have created:

1. A `design.md` describing the API at the conceptual level.
2. An `openapi-sketch.yaml` containing the OpenAPI 3.x specification (at least the path structure, request/response schemas, error responses).
3. A `state-machine.md` documenting the lifecycle states of a `ResourceRequest`.
4. A `tradeoffs.md` documenting at least five design decisions and the alternatives considered.

If you are doing this exercise self-paced and have ~90 minutes, prioritize design.md, openapi-sketch.yaml, and tradeoffs.md. The state machine can be terse.

---

## Part 1: Conceptual Design (20 minutes)

### Task 1.1: Identify the resources

Before writing any endpoints, decide what your *resources* are. A resource is a *noun* in the API. Candidates for this service include:

- `ResourceRequest` (a user's request for a pod)
- `Pod` (the actually-running pod)
- `Allocation` (the link between request and pod)
- `Quota` (the team's overall capacity)
- `ResourceTemplate` (a named pre-configured shape, e.g., "small-gpu" = 1 GPU + 4 CPU + 16Gi)

Decide which of these you want to expose to the API surface and which are internal.

**TODO**: Write down your choice in `design.md` under a heading "Resources". For each resource, write one sentence describing what it represents.

**Reflection prompt**: Are `ResourceRequest` and `Pod` the same thing or different things? If you treat them as the same, what becomes simpler? What becomes harder? Some platforms expose one; some expose two. Justify your choice.

### Task 1.2: Identify the actions

For each resource, list the actions a user can take on it. Map each action to an HTTP verb and a URL.

Worked example for `ResourceRequest`:

| Action | Verb | URL |
| --- | --- | --- |
| Create | POST | `/v1/resource-requests` |
| List (own team) | GET | `/v1/resource-requests` |
| Read | GET | `/v1/resource-requests/{id}` |
| Cancel | POST | `/v1/resource-requests/{id}/cancel` |
| Extend | POST | `/v1/resource-requests/{id}/extend` |
| Connect | GET | `/v1/resource-requests/{id}/connection` |

Note that "cancel" and "extend" are *actions* on a resource, not new resources. The sub-resource verb pattern (`/{id}/cancel`) is a common way to express this in REST.

**TODO**: Write your action table in `design.md` under a heading "Endpoints". Be exhaustive.

### Task 1.3: Decide the request and response shapes

For each endpoint, sketch the JSON request body (if any) and response body. You don't have to nail every field yet; capture the important ones.

For `POST /v1/resource-requests`:

```json
// Request
{
  "name": "alice-finetune-experiment-3",
  "template": "small-gpu",         // OR explicit resources below
  "resources": {
    "gpus": 1,
    "gpu_type": "a100",
    "cpu": "4",
    "memory": "32Gi"
  },
  "lifetime_hours": 4,             // optional, default 4, max 24
  "purpose": "fine-tuning experiment for churn model",
  "metadata": {                    // arbitrary tenant labels
    "experiment_id": "exp-072",
    "cost_center": "cc-12345"
  }
}

// Response (202 Accepted)
{
  "id": "rr-7Hjkz9",
  "status": "pending",
  "tenant": "team-alpha",
  "owner": "alice@aurelia.example",
  "requested_at": "2026-05-22T22:10:13Z",
  "expires_at": null,              // set when running
  ...
}
```

**TODO**: Sketch the request and response shapes for *every* endpoint in `design.md`. Aim for completeness over polish.

### Task 1.4: Decide identifier format

You need unique identifiers for `ResourceRequest`. Options:

- **UUIDv4**: `7f1c8b3e-9d0a-4c52-aa0b-2f81df5d1c44`. Globally unique. Long.
- **Short opaque ID**: `rr-7Hjkz9`. Prefixed, short, human-readable. Less unique (but big enough namespace if you use enough characters).
- **Sequence + tenant scope**: `team-alpha-001234`. Sequential. Reveals ordering.
- **Stripe-style**: `rr_LpkW9q3jK`. Prefixed with type, underscore separator.

**TODO**: Pick one and justify in `design.md`. Note that the choice has security implications (sequential IDs allow tenant size estimation via observation) and DX implications (UUIDs are unfriendly to humans).

---

## Part 2: OpenAPI Sketch (25 minutes)

### Task 2.1: Skeleton

Create `openapi-sketch.yaml` with the OpenAPI 3.1 skeleton:

```yaml
openapi: 3.1.0
info:
  title: Aurelia Resource Provisioning API
  version: 0.1.0
  description: |
    Self-service API for requesting and managing ephemeral GPU resources
    on the Aurelia ML platform.
servers:
  - url: https://api.aurelia.example.com
paths:
  # TODO: fill in
components:
  schemas:
    # TODO: fill in
  securitySchemes:
    bearer:
      type: http
      scheme: bearer
      bearerFormat: JWT
security:
  - bearer: []
```

### Task 2.2: Define the `ResourceRequest` schema

Under `components.schemas`, add a `ResourceRequest` schema with full property documentation:

```yaml
components:
  schemas:
    ResourceRequest:
      type: object
      required:
        - id
        - status
        - tenant
        - owner
        - requested_at
      properties:
        id:
          type: string
          description: Opaque identifier, prefixed `rr-`.
          example: "rr-7Hjkz9"
        status:
          type: string
          enum: [pending, scheduling, running, expired, failed, cancelled]
        tenant:
          type: string
          description: The team that owns this request.
          example: "team-alpha"
        owner:
          type: string
          description: Email of the user who created this request.
          format: email
        requested_at:
          type: string
          format: date-time
        started_at:
          type: string
          format: date-time
          nullable: true
        expires_at:
          type: string
          format: date-time
          nullable: true
        resources:
          $ref: "#/components/schemas/ResourceSpec"
        ...
```

**TODO**: Write the full `ResourceRequest` schema with at least 10 properties. Include `nullable: true` where appropriate. Document each property with a `description`.

### Task 2.3: Define the `ResourceSpec` schema

This is the nested structure inside a `ResourceRequest` that describes the requested resources:

```yaml
ResourceSpec:
  type: object
  required:
    - cpu
    - memory
  properties:
    gpus:
      type: integer
      minimum: 0
      maximum: 8
      default: 0
    gpu_type:
      type: string
      enum: [a100, h100, v100, t4]
      nullable: true
    cpu:
      type: string
      description: Kubernetes-style resource quantity, e.g. "4" or "4000m".
      example: "4"
    memory:
      type: string
      description: Kubernetes-style resource quantity, e.g. "32Gi".
      example: "32Gi"
```

**TODO**: Complete this schema. Decide whether `cpu` and `memory` are strings (Kubernetes style) or numbers (more JSON-friendly, but you have to pick a unit).

### Task 2.4: Define error response schema

A consistent error schema across all endpoints:

```yaml
ErrorResponse:
  type: object
  required:
    - error
  properties:
    error:
      type: object
      required:
        - code
        - message
      properties:
        code:
          type: string
          description: Stable, machine-readable error code.
          example: "quota_exceeded"
        message:
          type: string
          description: Human-readable error message.
        details:
          type: object
          description: Structured details specific to the error code.
        docs:
          type: string
          format: uri
          description: Link to documentation for this error.
```

**TODO**: List at least 8 error codes your API will produce, in `design.md`, with a one-line description of when each fires. Examples to start: `quota_exceeded`, `invalid_resource_spec`, `not_found`, `forbidden`, `template_not_found`, `lifetime_too_long`, `already_cancelled`, `idempotency_conflict`.

### Task 2.5: Define the create endpoint

The most important endpoint. Be thorough:

```yaml
paths:
  /v1/resource-requests:
    post:
      summary: Create a new resource request
      operationId: createResourceRequest
      parameters:
        - in: header
          name: Idempotency-Key
          schema:
            type: string
          required: false
          description: |
            Client-supplied idempotency key. Submitting the same key twice
            returns the same response without creating a duplicate request.
            Recommended for all clients that may retry on network errors.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - name
              properties:
                name:
                  type: string
                  pattern: '^[a-z][a-z0-9-]{2,62}$'
                template:
                  type: string
                  description: Optional named template instead of `resources`.
                resources:
                  $ref: "#/components/schemas/ResourceSpec"
                lifetime_hours:
                  type: integer
                  minimum: 1
                  maximum: 24
                  default: 4
                purpose:
                  type: string
                  maxLength: 256
                  description: Required free-text description for audit.
                metadata:
                  type: object
                  additionalProperties:
                    type: string
              oneOf:
                - required: [template]
                - required: [resources]
      responses:
        "202":
          description: Request accepted, processing.
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ResourceRequest"
        "400":
          description: Bad request.
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "402":
          description: Quota exceeded.
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "401":
          description: Authentication required.
        "403":
          description: Forbidden.
        "409":
          description: Idempotency-key conflict.
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
```

**TODO**: Complete this endpoint with all error cases.

### Task 2.6: Define the other endpoints

For each of the remaining endpoints (`GET /v1/resource-requests`, `GET /v1/resource-requests/{id}`, `POST /v1/resource-requests/{id}/cancel`, `POST /v1/resource-requests/{id}/extend`, `GET /v1/resource-requests/{id}/connection`), write the OpenAPI path entry. Don't skip the error responses — they're the boring but important part.

For the list endpoint, include pagination parameters:

```yaml
get:
  summary: List resource requests
  parameters:
    - in: query
      name: status
      schema:
        type: string
        enum: [pending, scheduling, running, expired, failed, cancelled]
    - in: query
      name: page_size
      schema:
        type: integer
        minimum: 1
        maximum: 200
        default: 50
    - in: query
      name: page_token
      schema:
        type: string
  responses:
    "200":
      description: A page of resource requests.
      content:
        application/json:
          schema:
            type: object
            properties:
              items:
                type: array
                items:
                  $ref: "#/components/schemas/ResourceRequest"
              next_page_token:
                type: string
                nullable: true
```

**TODO**: Complete all five remaining endpoints in `openapi-sketch.yaml`.

---

## Part 3: State Machine (15 minutes)

### Task 3.1: Diagram

Document the state machine of a `ResourceRequest` in `state-machine.md`. The states are: `pending`, `scheduling`, `running`, `expired`, `failed`, `cancelled`.

A simple ASCII diagram:

```
                    +-----------+
                    |  pending  |
                    +-----+-----+
                          |
                          | scheduler picks up
                          v
                    +-------------+
                    | scheduling  |
                    +------+------+
                           |
        +------------------+------------------+
        | pod scheduled                       | scheduling failed
        v                                     v
   +---------+                          +----------+
   | running |                          |  failed  |
   +----+----+                          +----------+
        |
        +-----------+-----------+
        |           |           |
        | lifetime  | user      | platform error
        | expired   | cancelled |
        v           v           v
   +---------+ +-----------+ +--------+
   | expired | | cancelled | | failed |
   +---------+ +-----------+ +--------+
```

**TODO**: Draw this in `state-machine.md`. For each transition, document the trigger and any side effects (e.g., "pod created in Kubernetes", "user notified by email").

### Task 3.2: Terminal states

Some states are *terminal* — once a request reaches them, it never leaves. Terminal states for this API: `expired`, `failed`, `cancelled`.

**TODO**: List the terminal states in `state-machine.md`. Document what API behaviors should be denied for a request in a terminal state. (E.g., extending a `cancelled` request should fail with `already_cancelled`; cancelling an `expired` request is a no-op.)

### Task 3.3: Transition rules

For each transition, decide:

- Can a user initiate it? (Cancel: yes. Expire: no, the platform does that.)
- Should it emit an event? (We have an event bus.)
- Should it notify the user? (Email, Slack, in-platform.)

**TODO**: Write a small table in `state-machine.md` covering each transition.

---

## Part 4: Tradeoffs and Defense (20 minutes)

### Task 4.1: Document at least five design decisions

For each major decision in your API, write a paragraph in `tradeoffs.md` covering:

1. **What you decided.**
2. **Alternatives you considered.**
3. **Why you picked the one you did.**
4. **What you'd revisit if your assumptions change.**

Required decisions to address:

#### Decision 1: Resource modeling

Did you expose `ResourceRequest` only, or also `Pod` and/or `Allocation`? Why? What does the choice make easy or hard?

Some alternatives:

- **Only `ResourceRequest`**: simpler. The pod is an implementation detail.
- **`ResourceRequest` + `Pod`**: more powerful — users can ask "is my pod healthy?" directly. But it leaks Kubernetes.
- **`ResourceRequest` + `Allocation`**: middle ground. `Allocation` is the platform's abstraction of "the live thing this request produced," without exposing Kubernetes directly.

#### Decision 2: Identifier format

What format? Why? What does it cost you operationally?

#### Decision 3: Sync vs async

You picked async (POST returns 202 with a handle). Why? Walk through what the alternative — synchronous — would have looked like and what would go wrong.

#### Decision 4: Versioning

Where does the version live (URL path, header, query param)? Why? What is your breaking-change policy for v1?

#### Decision 5: Idempotency

Why did you add `Idempotency-Key`? What happens if a user submits two different requests with the same key? What if they submit the same request without a key?

#### Decision 6 (optional): Template vs explicit resources

You allowed both `template` and `resources` (via `oneOf`). Why? What's the tradeoff?

### Task 4.2: Critique three alternative designs

Now imagine three colleagues each propose alternative designs. For each, write a paragraph in `tradeoffs.md` explaining what's *good* about their proposal and what you don't like.

**Alternative 1: Verb-shaped URLs**

```
POST /v1/createResourceRequest
POST /v1/cancelResourceRequest
GET  /v1/listResourceRequests
GET  /v1/getResourceRequest?id=...
```

What's good? (Some teams genuinely prefer this; familiar from gRPC.) What's bad? (Doesn't map to HTTP semantics; harder to cache; harder to use with REST tooling.) Would you accept this proposal? Why or why not?

**Alternative 2: GraphQL**

```
mutation CreateResourceRequest($input: ResourceRequestInput!) {
  createResourceRequest(input: $input) {
    id
    status
    ...
  }
}
```

What's good? (Clients only fetch fields they need; introspection.) What's bad? (Server complexity, caching issues, harder to rate-limit per query.) Would you accept this for the Aurelia platform? Why or why not?

**Alternative 3: Sync API with long polling**

```
POST /v1/resource-requests?wait_for=running
```

The endpoint blocks until the request reaches the `running` state (or times out at 5 minutes). What's good? (Simpler for users; one call instead of poll loop.) What's bad? (HTTP timeouts; harder to scale the API server; complicated retry semantics.) Would you offer this as an alternative or stick with your async design?

### Task 4.3: Self-critique

Finally, identify at least three weaknesses in *your own* design. Be ruthless.

Examples of self-critique:

- "I exposed `gpu_type` as an enum; if we add a new GPU type, that's a breaking change for clients that exhaustively match."
- "I made `purpose` a required field, but I have no way to verify it's truthful; it's audit theater unless we have a downstream review step."
- "My pagination uses `next_page_token` which is opaque, but I haven't documented its size limit or its expiry."

**TODO**: Write three self-critiques. Then write one or two sentences on how you'd address each.

---

## Part 5: Deeper Exploration (Optional, 30+ minutes)

If you have time after the core deliverables, dig into one of these:

### Option A: Add a streaming status endpoint

Design a Server-Sent Events endpoint (`GET /v1/resource-requests/{id}/events`) that streams status changes as they happen. Document:

- The event types (status-changed, lifetime-extended, error).
- The event format.
- How clients should handle reconnection.
- Whether this duplicates the polling endpoint or replaces it.

### Option B: Webhook-style notifications

Design a webhook subscription endpoint that lets a tenant register a URL to receive status events.

- How does the tenant authenticate the webhook receiver (HMAC signature)?
- What happens when the receiver is down?
- How does the platform handle redelivery?
- What rate limits apply?

### Option C: Resource template management

The platform has a `ResourceTemplate` resource (`small-gpu`, `medium-gpu`, `large-gpu`). Design the CRUD API for templates.

- Who can create/edit templates? (Probably only platform admins.)
- How do versioned template references work? (`small-gpu@v2`)
- What happens to existing requests when a template is deleted?

### Option D: Cost transparency

Add cost information to the API. Each running `ResourceRequest` has a *current cost* and an *estimated cost at expiration*.

- What endpoints expose cost?
- How is cost calculated?
- How does the API surface to clients that they're approaching a cost threshold?

### Option E: Audit-event API

Every action on the platform should be auditable. Design an audit-event log resource:

- `GET /v1/audit-events` (with filtering)
- Schema of an audit event (actor, action, resource, before/after, timestamp)
- Retention and access-control semantics

These extensions are useful in real platforms; if you do one, write up the additions in `design.md`.

---

## Common Pitfalls

Recognize these as you work:

1. **Modeling the database instead of the user's mental model.** If your API exposes `pod_uid`, `cluster_id`, `node_name` as top-level fields, you're leaking implementation. Hide them.
2. **Exhaustive enums with no escape hatch.** If you ship `gpu_type: enum[a100, h100, v100, t4]` and tomorrow add `b200`, clients that exhaustively match break. Either document forward compatibility or design for it (e.g., `gpu_type: string` with documented current values).
3. **Sync where async would be right.** "Just make the POST block until done" is tempting and wrong.
4. **No idempotency key.** A network blip should not create two pods.
5. **Boolean status field.** `is_running: true` doesn't extend to all the states you actually need.
6. **No pagination by default.** Lists must be paginated from day one.
7. **Generic error messages.** "Bad Request" is useless; `quota_exceeded` with details is useful.
8. **Putting tenant in the URL.** `/v1/teams/{tenant}/resource-requests` looks good but it means a request to `/v1/teams/team-beta/...` from a `team-alpha` user looks like an attempted privilege escalation, which is not what they meant — they were just confused about URLs. Prefer keeping tenant *implicit from the bearer token*.

If your design has any of these, fix them before finalizing.

---

## Reflection Questions

Answer in 2-3 sentences each, in `design.md` under a "Reflection" heading.

1. **The `purpose` field.** You added a required `purpose` field for audit. Did you check it client-side, server-side, or both? Will users provide useful purpose text or boilerplate? How does the audit case fail if they boilerplate?
2. **Quota enforcement timing.** When you check quota — at submit time (fail fast) or at schedule time (lazy) — what happens if a team is at 7 of 8 GPUs and submits two requests simultaneously? Which one wins? Is this a problem?
3. **The connect endpoint.** What does `GET /v1/resource-requests/{id}/connection` return? A URL? A kubeconfig? Both? What if the user requested a Jupyter and we have to wait for it to boot — does this endpoint poll, redirect, or return a "not ready" state?
4. **Extending lifetime.** Can a user extend a request beyond the 24-hour max by extending repeatedly? Should they be able to? How do you express that limit?
5. **Versioning the OpenAPI spec.** Your OpenAPI doc says `version: 0.1.0`. When does this go to `1.0.0`? When you ship to users? When you have a deprecation policy? When you have contract tests? Take a position.

---

## Solution Hints

There is no canonical solution; the goal is the discipline. But here are some hints if you're stuck.

- **For identifier format**, the Stripe-style `rr_AbCd1234` is hard to beat for new platform APIs. Prefixed, short, opaque, indicates type.
- **For sync vs async**, async wins for anything that touches a scheduler. The exception is "the operation is guaranteed to complete in <100ms" — and submitting a Kubernetes pod isn't that.
- **For versioning**, URL path versioning (`/v1/`) is the default for internal platforms. Save date-based for SaaS APIs with large external audiences.
- **For idempotency**, accept `Idempotency-Key` as an optional header. Cache the response for ~24 hours. Reject mismatches with `409`.
- **For pagination**, cursor-based (`next_page_token`) is more robust than offset-based.
- **For errors**, ship a stable `code` field; users will program against it. Do not change the meaning of an existing code.

If you want to compare your design against a published reference, the Kubernetes API conventions document is a good benchmark, as is the [Google API Design Guide](https://cloud.google.com/apis/design).

---

## Self-Assessment

Before moving on, check that you can answer:

- [ ] Can I sketch the full URL set on a napkin from memory?
- [ ] Can I name the six lifecycle states and the legal transitions?
- [ ] Can I explain to a colleague *why* the API is async, in two sentences?
- [ ] Can I name three error codes and what condition fires each?
- [ ] Can I defend my identifier format choice against the other three reasonable options?
- [ ] Have I written down at least three weaknesses of my own design?

If you answered yes to all, you're done.

If you can answer yes to four or five, you're mostly done; revisit the gaps.

If you can only answer yes to one or two, your design isn't quite ready. Spend another 30 minutes on the gaps and try again.

---

## Suggested Time Allocation

| Section | Time |
| --- | --- |
| Part 1: Conceptual Design | 20 min |
| Part 2: OpenAPI Sketch | 25 min |
| Part 3: State Machine | 15 min |
| Part 4: Tradeoffs and Defense | 20 min |
| Part 5: Deeper Exploration (optional) | 30+ min |
| **Total core** | **80 min** |

If you finish faster than this, push into Part 5. If you take longer, the most important sections are 1, 2, and 4. Part 3 can be terse if you're tight.

---

## Where to Go from Here

After this exercise, you have a *design* — not an implementation. You have:

- A defended API shape.
- A formal-ish spec.
- A state machine.
- A pile of rationale.

This is the artifact you would bring to an architectural design review. Module 02 will pick up the API design thread and dig into versioning and evolution in depth. Module 03 will look at how this kind of resource provisioning is actually implemented on Kubernetes. Module 09 will revisit the security and audit side.

For now, push your `design.md` + `openapi-sketch.yaml` + `state-machine.md` + `tradeoffs.md` to your fork of this curriculum repo, request review from a peer if you have one, and move on to Exercise 02.

Welcome to platform-API design. Most of your career as a platform engineer will be variations on this exercise.

---

## Appendix A: A worked sample `tradeoffs.md` excerpt

If you want to compare against a reference, here is one defensible excerpt covering "Decision 1: Resource modeling."

> **Decision 1: We expose `ResourceRequest` only. We do not expose `Pod` or `Allocation` to the API.**
>
> *Alternatives considered:*
> 1. Expose `ResourceRequest` + `Pod` — gives users direct visibility into "is my pod alive?" but couples the API to Kubernetes, making it harder to add non-Kubernetes backends later.
> 2. Expose `ResourceRequest` + `Allocation` — gives a platform-native abstraction over the running thing without committing to Kubernetes vocabulary.
>
> *Why option chosen:* We expect to add non-Kubernetes backends (AWS Batch, Vertex AI) within 12 months. Exposing `Pod` would lock us into Kubernetes vocabulary. The `Allocation` middle ground is also viable, but at Aurelia's scale (5 backends max anticipated), the extra abstraction is not yet earned. We will reconsider if a user requests "is my running thing healthy" frequently enough to warrant exposing `Allocation`.
>
> *If our assumptions change:* If we never add a non-Kubernetes backend, the `ResourceRequest`-only design is unnecessarily abstract. Conversely, if we add 3+ backends and discover users need a unified runtime view, we will introduce `Allocation` then.

You will recognize this shape — *decision, alternatives, rationale, contingency* — in nearly every architectural design doc you read or write in your career. The discipline of writing it makes you a better designer.

---

## Appendix B: A note on extensions you may discover

While doing this exercise, you may find yourself wanting to add things not in the original spec:

- A `/v1/resource-templates` CRUD API for managing templates.
- A `/v1/usage` endpoint for tenants to see their consumption.
- A `/v1/audit-events` endpoint for security/compliance.
- A `/v1/events` SSE stream of all platform events.

These are reasonable. They are also out of scope for *this* exercise. The point is to nail the core `/v1/resource-requests` API shape — once that's solid, the others fall into place. If you have time and curiosity, add them as Part 5 deeper-exploration tasks. If not, note them in `design.md` as "future work" and move on.

The discipline of *staying in scope* during a design exercise is itself worth practicing.
