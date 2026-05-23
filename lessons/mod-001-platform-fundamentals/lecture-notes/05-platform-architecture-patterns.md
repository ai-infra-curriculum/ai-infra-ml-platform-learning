# Lecture 05: Platform Architecture Patterns

## Table of Contents

1. [Introduction](#introduction)
2. [Monolith, Microservices, and the Spectrum](#monolith-microservices-and-the-spectrum)
3. [Event-Driven Architecture for Platforms](#event-driven-architecture-for-platforms)
4. [Plugin Systems and Extension Points](#plugin-systems-and-extension-points)
5. [Control Plane vs Data Plane](#control-plane-vs-data-plane)
6. [Stateful Services and Storage Choices](#stateful-services-and-storage-choices)
7. [The Reference ML Platform Topology](#the-reference-ml-platform-topology)
8. [Team Structure and Conway's Law](#team-structure-and-conways-law)
9. [Choosing Architecture for Your Stage](#choosing-architecture-for-your-stage)
10. [Summary](#summary)

---

## Introduction

We have spent the previous four chapters at the conceptual layer: what a platform *is*, how to think about it, how to handle multi-tenancy, how to design APIs. This final chapter zooms out and asks: *how is the platform organized as a system?*

An ML platform is, structurally, a small distributed system. It has services (registry, training, serving, observability), data stores (metadata DB, artifact store, feature store), event flows (job-completed, model-registered, deployment-promoted), and integration points with external systems (data warehouses, cloud IAM, CI/CD).

The architectural choices about *how those pieces fit together* — what's a service, what's a library, what's an event, what's a synchronous call — shape how the platform evolves, scales, and survives team changes. This chapter is a tour of the common patterns and the choices behind them.

By the end you should be able to:

- Distinguish monolith, microservices, modular monolith, and "majestic monolith" architectures, and identify which fits a given stage of platform maturity.
- Describe event-driven patterns common to ML platforms (work queues, change streams, audit pipelines) and identify when each is appropriate.
- Articulate the value of plugin systems and identify what *not* to expose as a plugin.
- Distinguish a platform's *control plane* from its *data plane* and explain why the distinction matters.
- Sketch a reference ML platform topology and name the common variations.

We are deliberately keeping implementation specifics light. The choices here are *patterns*, and they recur across whatever specific tools (Kubeflow, Argo, MLflow, KServe, etc.) you happen to be using. Later modules will get hands-on with the specific tools.

---

## Monolith, Microservices, and the Spectrum

The first architectural question is *how many services your platform is*. The answers people give to this question are often more dogmatic than they should be.

### The pure monolith

One service. One database. One deployment artifact. All the platform's features live in the same codebase and run in the same process.

When this works well:

- Small team (1-5 engineers).
- Single language stack.
- No need to scale parts of the platform independently.
- Rapid iteration is more important than operational sophistication.

When this breaks:

- Multiple teams need to ship independently (deployments serialize).
- Different parts of the platform have wildly different scaling characteristics.
- One bug in one feature can take down everything.
- The codebase becomes unmanageably large.

Monoliths are *underrated* as a starting point. Many platforms that "should be microservices" never reach the scale where the cost of a monolith exceeds the cost of microservices.

### The pure microservices stack

Each capability is its own service: registry, training-orchestrator, deployment-controller, observability-aggregator, ... ten or twenty independently-deployed services, often in different languages, with their own databases.

When this works well:

- Many teams, each owning one or two services.
- Components with wildly different scaling needs.
- Strong organizational discipline (the social problems of microservices are real).

When this breaks:

- The team is too small to staff all the services.
- Distributed-systems failure modes (cascading failures, partial outages, debugging across services) start to dominate.
- "Where does this feature go?" becomes a daily question without clear answers.

Microservices are *overrated* as a starting point. They have a real cost — operational, organizational, cognitive — that you pay every day. Don't pay it before you need to.

### The modular monolith (most common)

One deployment, but internally factored into well-bounded modules. Each module has a clear API. Modules don't share state. Some modules call each other through internal libraries; others communicate through events.

This is, in my experience, the most common shape for *successful* ML platforms in their first few years. It gives you:

- Single-deploy simplicity.
- Module-boundary discipline that prepares you to split later.
- Shared infrastructure (DB pool, observability, auth) without re-implementing it per service.
- Clear extraction path: when a module needs to scale independently, you split it out.

### The "majestic monolith" + a few satellites

A name from the Rails community. A single primary service containing 80% of the platform's logic, plus a few satellite services for parts that genuinely need to scale or be isolated (e.g., the inference fleet, the training-job runner).

This is often where mature ML platforms end up. The control plane is a "majestic monolith"; the data plane (which has wildly different scaling characteristics) is separately scaled.

### Picking your point on the spectrum

A useful heuristic:

- **0-3 engineers, < 1 year**: pure monolith.
- **3-10 engineers, 1-3 years**: modular monolith.
- **10+ engineers, 3+ years**: majestic monolith + satellites, evolving toward selective microservices.
- **Large org, mature platform**: microservices with shared platform-of-platforms underneath.

This is not a law; it's a starting point. Iterate based on the actual pain you're feeling. **Architecture is a response to current pain, not a hedge against hypothetical future pain.**

### A common failure: premature microservices

A team excited about microservices builds 12 services on day 1. They are 5 engineers. They spend 80% of their time fighting distributed-systems problems and 20% on actual platform features. After 18 months they re-merge half the services into a monolith.

This story is common. Don't be the next telling of it. The cost of merging services is high; the cost of splitting a well-factored monolith is moderate. Start in the cheap direction.

---

## Event-Driven Architecture for Platforms

Many platform operations are naturally event-driven: a training run completes, a model is registered, a deployment is promoted, a job fails. These events have many consumers (UI, metrics, downstream automation, audit log).

Event-driven architectures pay off for platforms because:

- **Loose coupling.** New consumers can subscribe without modifying producers.
- **Asynchronous workflows.** Long-running operations don't block.
- **Audit and replay.** Event logs give you a free audit trail.
- **Parallelism.** Multiple consumers process events in parallel.

But they also have real costs:

- **Eventual consistency.** Consumers see different views of the world at different times.
- **Debugging is harder.** You can't follow a single call stack through an event flow.
- **Ordering and exactly-once delivery are hard.** Most messaging systems give at-least-once, leaving idempotency to the consumer.

### Common patterns

#### Work queue

A producer puts a job on a queue; a pool of workers picks jobs off the queue, processes them, and reports completion.

```
[API] → [Queue] → [Worker pool] → [Result store]
                       │
                       └→ [Status events]
```

For ML platforms, this is the dominant pattern for training jobs: the API submits, a worker pool with GPU nodes processes, status flows back. Tools: Celery (Python), Kueue (Kubernetes-native), SQS (AWS), Cloud Tasks (GCP).

#### Change stream

A central source of truth emits a stream of changes. Consumers tail the stream and update their derived state.

```
[Authoritative service] → [Stream / log]
                              │
                              ├→ [Search index]
                              ├→ [Analytics warehouse]
                              ├→ [Audit log]
                              └→ [Cache invalidator]
```

This is the right pattern when many systems need to know about changes to one source-of-truth. Tools: Kafka, Debezium (CDC from databases), Pulsar.

For ML platforms: model-registry change streams feeding deployment automation, observability ingestion, etc.

#### Pub/sub with topics

A many-to-many version. Producers publish to topics; subscribers consume from topics. No assumed source-of-truth; the bus is the integration layer.

```
[Producer A] ─┐         ┌─→ [Consumer X]
              ├→ [Topic]─┤
[Producer B] ─┘         └─→ [Consumer Y]
```

Useful when many components produce and consume the same kinds of events. Common for cross-tenant or cross-component coordination.

#### Choreography vs orchestration

A subtle but important choice for event-driven workflows:

- **Choreography**: each service reacts to events. No central conductor. The workflow emerges from the interactions.
- **Orchestration**: a central service (workflow engine) drives the steps. Each step is a call to a service.

Choreography is more loosely coupled but harder to reason about end-to-end. Orchestration is easier to debug but introduces a central dependency.

For ML platforms, *training pipelines* are usually orchestrated (Airflow, Argo Workflows, Kubeflow Pipelines) because debugging a stuck pipeline is much easier when there's a central conductor. *Cross-platform events* (registry → deployment → observability) are often choreographed because there's no natural conductor and the components are loosely related.

### A worked example: model promotion

A typical event flow when a model is promoted from staging to production:

```
1. user calls POST /v1/models/{id}/promote-to-prod
   ↓
2. Registry service updates state; emits "model-promoted" event
   ↓
3. Deployment controller (subscribed) reads event;
   creates new inference deployment with new model
   ↓
4. Health check passes; deployment controller emits "deployment-live"
   ↓
5. Routing controller (subscribed) reads "deployment-live";
   shifts traffic via weighted routing
   ↓
6. Observability service (subscribed) reads both events;
   updates dashboards and alerts
   ↓
7. Audit service (subscribed) reads both;
   writes a permanent audit log entry
```

Each step is a separate service, doing its own work, reacting to events. None of the services know about each other directly — they know only about the event schemas. New consumers can be added without modifying existing ones.

This is the strength of event-driven design. It's also the difficulty: when something goes wrong at step 5, you have to trace back through events, not through call stacks. Tooling matters.

---

## Plugin Systems and Extension Points

A *plugin system* lets users extend the platform without modifying the platform's code. Done well, plugin systems are how platforms scale: the platform team builds the core; users (or other teams) build the long tail.

Done poorly, plugin systems are a maintenance nightmare and a security risk.

### When to add a plugin system

Add plugins when:

- Multiple users have asked for the same *category* of extension (e.g., "I want to plug in a new training framework").
- The category is well-bounded — you can write down what a plugin must do as an interface.
- The platform team is not the right party to build all the extensions (no domain expertise, no bandwidth).

Don't add plugins when:

- You have one or two extension requests; just build them in.
- The category is fuzzy — every "plugin" needs to be a special case in the core.
- The blast radius of a bad plugin is high (the plugin can crash the platform, leak data, escalate privileges).

### Plugin interface design

Three things to get right:

1. **What the plugin can see.** Inputs to the plugin's hooks should be clearly specified, minimal, and stable. Don't pass "the entire platform context" as a parameter — the plugin will depend on internal details that will change.
2. **What the plugin can do.** Plugins should have *limited* side effects. A plugin that's allowed to call back into arbitrary platform APIs becomes a back door. Constrain the plugin's outputs.
3. **How the plugin is loaded.** Same process? Subprocess? Container? Webhook? The choice has security and operational implications.

### Common plugin patterns

#### In-process plugin (library)

Plugins are Python packages installed in the platform's runtime. The platform imports them and calls their functions.

Pros:
- Fast (no IPC overhead).
- Simple.
- Easy local development.

Cons:
- A plugin can crash the platform.
- A malicious plugin can do anything.
- Plugin dependencies pollute the platform's dependency tree.

Appropriate for: platform-team-vetted plugins; first-party extension points.

#### Subprocess plugin

The platform spawns a separate process per plugin invocation. Communication via stdin/stdout, JSON-RPC, or similar.

Pros:
- Isolation (a crashed plugin doesn't crash the platform).
- Language-agnostic.

Cons:
- IPC overhead.
- Process management complexity.

Appropriate for: medium-trust plugins.

#### Webhook plugin

The platform calls an HTTP endpoint owned by the plugin author. Communication is over the network.

Pros:
- Total isolation.
- Language- and runtime-agnostic.
- Can be operated by separate teams.

Cons:
- Latency (network call).
- Failure modes (the webhook is down).
- Auth has to be designed.

Appropriate for: cross-team extensions; least-trusted plugins.

#### Sidecar plugin

The plugin runs as a container alongside the platform's service in the same pod. Communication via localhost.

Pros:
- Isolation at the container level.
- Lifecycle managed by Kubernetes.
- Can be operated separately.

Cons:
- Heavy.
- Pod-level coupling.

Appropriate for: long-running plugin functionality that needs to be co-located with platform state for performance.

### A specific example: pluggable training backends

A common ML platform extension point: the *training backend*. The platform supports running training on Kubernetes, on AWS Batch, on SageMaker, on Vertex AI — and possibly on user-supplied backends.

The interface is roughly:

```python
class TrainingBackend(Protocol):
    def submit(self, spec: TrainingJobSpec) -> JobHandle:
        """Submit a training job. Return a handle for status queries."""
        ...

    def status(self, handle: JobHandle) -> JobStatus:
        """Query current status."""
        ...

    def cancel(self, handle: JobHandle) -> None:
        """Cancel a running job."""
        ...

    def logs(self, handle: JobHandle, follow: bool = False) -> Iterator[str]:
        """Stream logs."""
        ...
```

Anyone implementing this Protocol can be a training backend. The platform routes training jobs to the appropriate backend based on configuration.

Note what is *not* in the interface:

- The plugin doesn't get to decide pricing.
- The plugin doesn't see the user's identity directly (the platform passes a scoped credential).
- The plugin doesn't see the registry or model artifacts directly (it deals with input/output paths the platform supplies).

The plugin sees the *minimum* needed to do its job, and no more. This is the discipline of good extension-point design.

### When the plugin system itself becomes a problem

Plugin systems can be *too* successful. Symptoms:

- Plugins start depending on each other implicitly.
- Plugin authors complain that the interface is too restrictive.
- The platform team spends most of its time on plugin-system maintenance.
- A misbehaving plugin causes a platform incident.

Mitigations:

- **Plugin manifest.** Plugins declare their dependencies, version, owner, and resource expectations. The platform can reject incompatible plugins.
- **Plugin sandboxing.** For untrusted plugins, run them in restricted containers or VMs.
- **Plugin observability.** Track plugin invocations, latencies, errors. Bad plugins surface in dashboards.
- **Plugin deprecation policy.** Old plugins must be updated or removed. Don't accumulate forever.

A useful slogan: **plugins are software you didn't write but you're operationally responsible for**. Plan accordingly.

---

## Control Plane vs Data Plane

A distinction borrowed from networking and increasingly common in platform terminology. Internalizing it sharpens your thinking.

- **Control plane**: the part of the platform that handles *configuration*, *coordination*, and *state changes*. Examples: the API that submits training jobs, the registry that holds model metadata, the deployment controller that promotes new models. Control planes are typically lower-throughput and higher-stakes — the data they hold is the source of truth.
- **Data plane**: the part of the platform that handles *runtime data flow*. Examples: the inference service that serves predictions, the feature server that serves features at request time, the training workers that read datasets. Data planes are typically higher-throughput and lower-individual-stakes — a single failed request is recoverable; a single failed control-plane action might lose data.

### Why the distinction matters

1. **Different scaling characteristics.** The control plane handles 100 requests per second; the data plane handles 100,000.
2. **Different availability requirements.** A 30-second control-plane outage is annoying; a 30-second data-plane outage costs revenue.
3. **Different change frequency.** The control plane changes slowly (new features); the data plane processes traffic continuously.
4. **Different observability needs.** Control plane observability is about *correctness of state*; data plane observability is about *latency and throughput*.

### Architectural implications

- The control plane often holds the **single source of truth** (a relational DB; the model registry).
- The data plane often **caches** or **denormalizes** control-plane state for performance. Updates propagate from control plane to data plane via events or pulls.
- The data plane should be **degraded-mode-tolerant** — if the control plane is down, the data plane keeps serving with its last known good state. This is achieved by avoiding synchronous calls from data plane to control plane on the hot path.

### A worked example

Inference serving:

- Control plane: registry holds "model X version Y is deployed to environment Z."
- Data plane: inference pods load model X version Y at startup and serve predictions.
- Update flow: registry update → event → deployment controller → new inference pods → traffic shift.

Crucially, the inference pods don't call the registry on every request. They load the model once and serve. If the registry is down, existing inference pods keep working; only *new* deployments are blocked.

Compare to a naive design: every inference request calls the registry to look up "which version of model X is current?" This couples the data plane to the control plane and creates a brittle dependency. Don't.

---

## Stateful Services and Storage Choices

Most platform services are stateful at some level. The choices about *what stores what* are some of the most consequential.

### A taxonomy of state

| State kind | Examples | Typical store |
| --- | --- | --- |
| **Authoritative metadata** | Model registry, training-run metadata, tenant config | Relational DB (Postgres / MySQL) |
| **Large artifacts** | Trained models, datasets, container images | Object store (S3 / GCS / Azure Blob) + content-addressable hash |
| **Hot serving state** | Online features, model weights in serving pods, autoscaling state | Redis, in-memory, local SSD |
| **Audit / event log** | Every action taken by every user, audit-compliant | Append-only log (Kafka, S3 with object lock, dedicated DB) |
| **Metrics and traces** | Latency histograms, request rates | TSDB (Prometheus, Cortex, Mimir) |
| **Logs** | Application logs, training output | Log store (Loki, Elasticsearch, CloudWatch) |
| **Search indexes** | Find a model by name, browse runs | Search (Meilisearch, Elasticsearch, OpenSearch) |

A few principles:

1. **Each kind of state has a primary store.** Don't try to use one store for everything. Postgres is bad at TSDB. S3 is bad at random small reads. Choose the right store per kind.
2. **Authoritative store is single-master.** For each piece of state, one store owns it. Other systems derive from it.
3. **Backups.** Each authoritative store needs backups, tested recovery procedures, and a documented RPO/RTO.
4. **Schema migrations.** Authoritative stores need disciplined schema migration tooling (Alembic, Flyway, etc.). This is unglamorous; do it anyway.

### Avoiding storage lock-in

A common temptation: use the cloud's managed offerings (DynamoDB, BigQuery, etc.) for everything because it's easy. The tradeoff: your platform becomes deeply coupled to that cloud.

For internal platforms, this is usually fine — multi-cloud is rarely the right priority. For SaaS platforms, it may matter.

A reasonable middle path: use **open-source-compatible** managed services. RDS Postgres is portable to any Postgres. S3 has work-alikes (MinIO, GCS via interop, Azure Blob via gateways). Kafka is portable. Elasticsearch is portable. Use the managed version for operational ease but design as if you could migrate.

### Cache layers

ML platforms benefit from aggressive caching at several layers:

- **Model artifact cache** at the inference pod level (load once, serve many).
- **Feature cache** for repeated lookups of the same entity.
- **Metadata cache** for frequently-read registry data.
- **Dataset cache** for repeated training on the same data.

Each cache has its own invalidation discipline. Cache invalidation is famously one of the two hard things in computer science; it's especially hard in platforms because invalidation must respect tenant boundaries and consistency requirements.

A useful pattern: **content-addressable storage**. The cache key is the hash of the content. If the content changes, the hash changes, so the cache key changes. No explicit invalidation needed. This works for immutable artifacts (models, datasets, container images); it doesn't work for mutable state.

---

## The Reference ML Platform Topology

Drawing the pieces together, here is the "reference" ML platform topology. Most production ML platforms look something like this, with variations.

```
                           ┌──────────────────────────────┐
                           │      Identity Provider       │
                           │       (OIDC / SSO)           │
                           └──────────────┬───────────────┘
                                          │
        ┌─────────────────────────────────┼─────────────────────────────────┐
        │                                 │                                 │
   ┌────▼─────┐                  ┌────────▼────────┐                  ┌─────▼─────┐
   │   CLI    │                  │   Python SDK    │                  │  Web UI   │
   └────┬─────┘                  └────────┬────────┘                  └─────┬─────┘
        │                                 │                                 │
        └─────────────────┬───────────────┴─────────────────┬───────────────┘
                          │                                 │
                          ▼                                 ▼
              ┌────────────────────────────────────────────────────┐
              │            Control Plane (REST API)                │
              │  - tenancy / RBAC / quotas                         │
              │  - training-jobs API                               │
              │  - registry API                                    │
              │  - deployments API                                 │
              │  - features API (offline)                          │
              └─────────────┬───────────────────┬──────────────────┘
                            │                   │
                ┌───────────▼─────┐    ┌────────▼────────────┐
                │  Metadata DB    │    │  Object Store       │
                │  (Postgres)     │    │  (S3 / GCS)         │
                └─────────────────┘    │  - models           │
                                       │  - artifacts        │
                                       │  - datasets         │
                                       └─────────────────────┘
                            │
              ┌─────────────┴──────────────┬───────────────────────┐
              │                            │                       │
       ┌──────▼─────────┐         ┌────────▼─────────┐    ┌────────▼─────────┐
       │   Training      │        │   Inference     │    │  Observability   │
       │   Orchestrator  │        │   Service       │    │  Pipeline        │
       │   (Argo/Kueue)  │        │   (KServe/...)  │    │  (Prom/Loki/...) │
       └──────┬──────────┘        └────────┬────────┘    └──────────────────┘
              │                            │
       ┌──────▼─────────┐         ┌────────▼─────────┐
       │  Training      │         │  Inference       │
       │  workers       │         │  pods            │
       │  (GPU nodes)   │         │  (CPU/GPU)       │
       └────────────────┘         └──────────────────┘

                Data Plane (gRPC):
                  - online feature serving
                  - high-throughput inference
                  - training data streaming
```

### Components, briefly

- **Identity Provider**: source of truth for who users are. Federates to platform via OIDC.
- **CLI / SDK / UI**: three surfaces for the same APIs. SDK is the dominant one for ML users.
- **Control Plane (REST API)**: the brain. Handles tenancy, configuration, lifecycle.
- **Metadata DB**: authoritative store for control-plane state.
- **Object Store**: where the big stuff lives (models, datasets, artifacts).
- **Training Orchestrator**: takes training-job submissions, schedules them, watches their completion.
- **Inference Service**: serves models behind APIs.
- **Observability Pipeline**: collects logs, metrics, traces, makes them queryable per tenant.

### Variations

- **Without a separate orchestrator**, training submits directly to Kubernetes Jobs. Works for small platforms; lacks fair-share and quota features.
- **With a feature store**, an additional service (Feast, Tecton) sits between data sources and the platform, providing point-in-time-correct features.
- **With a notebook server**, an additional service (JupyterHub) provides browser-based interactive environments inside tenant namespaces.
- **With workflow orchestration**, a workflow engine (Argo Workflows, Kubeflow Pipelines, Prefect) handles multi-step pipelines beyond single training jobs.
- **With model monitoring**, an additional service computes drift, accuracy, and other quality metrics on production traffic.

The reference topology is a starting point. Real platforms add or remove components based on their use cases.

---

## Team Structure and Conway's Law

> Conway's Law: "Organizations design systems that are copies of their communication structures."

Stated more bluntly: your platform's architecture will reflect your team's structure. If you have one team, you tend to build one service. If you have five teams, you tend to build five services with rough boundaries matching the teams.

This is the *inverse* of how we usually think about architecture (architecture first, then teams to match). In practice, the team structure usually wins. So thinking about team structure is part of thinking about architecture.

### Common team structures for ML platforms

1. **The single platform team.** One team, owns everything. Works at the modular-monolith stage. Common in companies with 1-3 ML platform engineers.
2. **The vertical team.** A team owns one "vertical" of the platform end-to-end: e.g., one team owns training, another owns serving, another owns the registry. Works at the satellite-services stage.
3. **The horizontal team.** A team owns a *layer* across the platform: e.g., one team for control plane, one for data plane, one for observability. Less common but reasonable.
4. **Platform + enablement teams.** Platform team builds the platform. Enablement teams (sometimes called "embedded SREs," "AI engineers," etc.) work *with* product teams to help them onboard. The platform team doesn't do user-facing support; the enablement teams do.
5. **Centralized platform with federated contributors.** A small core platform team plus contributions from product teams' engineers. The core team gates merges; contributors do most of the work. Hard to do well; powerful when it works.

The right structure depends on your scale and your maturity. Most successful ML platforms start with #1 (single team) and evolve toward #2 (vertical) or #4 (platform + enablement) over a few years.

### A small note on titles

The "ML Platform Engineer" title is relatively new and not consistently defined across companies. You may see overlapping titles:

- **ML Platform Engineer** — typical title at companies with a dedicated platform team.
- **MLOps Engineer** — often overlaps; sometimes emphasizes more of the deployment/ops side.
- **ML Infrastructure Engineer** — sometimes emphasizes the lower-level stack (Kubernetes, GPU drivers, networking).
- **AI Platform Engineer** — increasingly common, sometimes emphasizes LLM/foundation-model workloads.
- **Data Platform Engineer** — overlaps; typically more focused on data pipelines and warehouse than on model training/serving.

These titles are converging but not fully merged. If you are searching for jobs or recruiting, expect to see all of these for similar work.

### Avoiding the "platform team is a service team" trap

A common failure mode: the platform team becomes a *service team* for the rest of the engineering org. They handle tickets, configure tenants, debug user issues, run on-call for everyone.

This is not a platform team. It is an SRE team for ML. There may or may not be space for that role, but it is structurally different from platform engineering.

The mitigation: ruthlessly automate the toil. Every ticket is a candidate for automation. The platform team's success is measured by *not* having to handle tickets, not by handling more.

---

## Choosing Architecture for Your Stage

A summary heuristic for choosing platform architecture by stage:

### Stage 1: First year, 1-3 engineers, 1-3 tenants

- **Monolith or modular monolith.** One service. One database. Easy deploys.
- **REST API + Python SDK.** No need for gRPC yet.
- **Kubernetes namespaces for tenancy.** No need for virtual clusters.
- **Synchronous workflows where possible.** Defer event-driven design until you have a reason.
- **No plugin system.** Build features in.
- **Open-source components where possible.** Don't reinvent.

### Stage 2: Second year, 4-10 engineers, 5-20 tenants

- **Modular monolith with one or two extracted services** (training orchestrator, inference service).
- **REST control plane + gRPC for data plane.**
- **Event-driven workflows for cross-service coordination.**
- **Resource quotas and basic fair-sharing.**
- **First plugin or extension point** (e.g., pluggable training backends) if there's demand.
- **Cost showback** to tenants.

### Stage 3: Third year+, 10+ engineers, 20+ tenants

- **Majestic monolith + several satellites.** Continue extracting based on actual pain.
- **Workflow orchestration** (Argo / Kubeflow Pipelines).
- **Multi-cluster or virtual-cluster tenancy** for scale.
- **Sophisticated multi-tenancy** (priority classes, pre-emption, dominant-resource fair-share).
- **Chargeback** to tenants.
- **Mature deprecation policy** for APIs.
- **Self-service onboarding** for new tenants (automated namespace + quotas + dashboards + auth).

### What stays constant across stages

- **API-first design.** From day one, your API is your contract.
- **Multi-tenancy from day one** (even if just two tenants).
- **Observability from day one.**
- **Documentation as a first-class deliverable.**
- **User research as a continuous activity.**

The architecture evolves; the disciplines do not.

---

## Summary

- **Don't start with microservices.** Most platforms are best served by a monolith or modular monolith for the first 1-3 years. Microservices have a real ongoing cost.
- **Event-driven patterns** (work queues, change streams, pub/sub) are appropriate for asynchronous workflows, cross-service coordination, and audit pipelines. Use them when the asynchrony is real; don't introduce events for events' sake.
- **Plugin systems** are how platforms scale beyond the platform team's bandwidth. Design extension interfaces narrowly. Plan for plugin lifecycle, security, and observability.
- **Distinguish control plane from data plane.** They have different scaling, availability, and observability needs. Avoid synchronous dependencies from data plane to control plane on the hot path.
- **Store the right state in the right store.** Relational DB for authoritative metadata, object store for artifacts, TSDB for metrics, log store for logs, search for browse. Don't try to use one tool for everything.
- **The reference ML platform topology** — identity, CLI/SDK/UI, control plane API, metadata DB + object store, training/inference/observability subsystems — is a starting point. Variations are common.
- **Team structure shapes architecture** (Conway's Law). Choose team structure deliberately, not by accident. Avoid the "platform team becomes a service team" trap.
- **Architecture evolves with stage.** Year-1 monolith → year-2 modular monolith with satellites → year-3+ majestic monolith plus selective microservices. Don't skip ahead.

This concludes the lecture portion of Module 01. The next thing to do is the exercises (under `exercises/`) and the quiz (under `quizzes/`).

---

## Reflection Questions

1. Sketch the architecture of an ML platform you have used (or studied). Where on the monolith-to-microservices spectrum did it sit? Was that the right point for the stage of the company?
2. Identify one event-driven workflow in your sketched platform. What are the events? Who produces them? Who consumes them? Where does choreography or orchestration apply?
3. Pick one "extension point" in your sketched platform that could become a plugin interface. Define the interface (inputs, outputs, side effects) in 5-10 lines.
4. Sketch the control-plane / data-plane boundary in your platform. Are there any synchronous calls from data plane to control plane that should be removed?
5. If your platform team grew from 3 engineers to 30 in one year, what would have to change about the architecture? What would *stay the same*?

---

## Further Reading

- **["Modular Monolith"](https://www.kamilgrzybek.com/blog/posts/modular-monolith-primer) (Kamil Grzybek).** A concise primer on the pattern.
- **["The Majestic Monolith"](https://m.signalvnoise.com/the-majestic-monolith/) (DHH, 2016).** The contrarian-popular case for monoliths.
- **["Microservices"](https://martinfowler.com/articles/microservices.html) (Lewis & Fowler, 2014).** The original Fowler/Lewis piece. Read alongside the "Microservice Premium" follow-up.
- **["Building Event-Driven Microservices"](https://www.oreilly.com/library/view/building-event-driven-microservices/9781492057888/) by Adam Bellemare.** Comprehensive treatment of event-driven patterns.
- **["Designing Data-Intensive Applications"](https://dataintensive.net/) by Martin Kleppmann.** Not platform-specific, but the chapters on consistency, replication, and stream processing are foundational for anyone designing platforms.
- **["Team Topologies"](https://teamtopologies.com/) by Skelton & Pais.** Already cited in chapter 01; the "platform team" pattern they describe is core. Worth re-reading at the end of this module.
- **[CNCF Cloud Native Landscape](https://landscape.cncf.io/).** Browse it to see what tools fit where in the reference topology. Don't try to memorize.

You have now read the five chapters of Module 01. Next: hands-on exercises in `exercises/`, the quiz in `quizzes/`, and a curated reading list in `resources.md`. When you have done those, move on to Module 02 — API Design for ML Platforms (forthcoming).

Welcome again to platform engineering. The hardest and most interesting parts of this discipline live in the next several modules; you are now equipped to engage with them critically.
