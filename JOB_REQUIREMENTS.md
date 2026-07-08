# Job Requirements — AI/ML Platform Engineer

> **Role**: AI/ML Platform Engineer (Level 30)
> **Research window**: 2026-04-05 → 2026-07-04 (last 90 days)
> **Postings analyzed**: 34 distinct postings across 24 employers
> **Last refreshed**: 2026-07-04

This document captures the recurring requirements observed in current job postings for the **AI/ML Platform Engineer** role, maps them to learning coverage in this repository, and links out to lower-level repositories when a requirement is genuinely owned elsewhere in the role hierarchy.

The **Ownership Rule** governs cross-role placement: if a requirement applies to multiple roles, the primary coverage lives in the *lowest-level* repository where it is genuinely required. Higher-level roles link to that owner unless they need additional depth, architectural context, or leadership framing.

For raw, machine-readable findings (postings, requirements, evidence, coverage paths), see [`.aicg/job-requirements.json`](./.aicg/job-requirements.json).

---

## Summary of the 2026 Shift

**2026-07 cycle note**: A market re-check for the 2026-04-05 → 2026-07-04 window found **no material shift** since the 2026-06-01 refresh. The three gaps identified in June remain the standing story; two new postings (NVIDIA Sr ML Platform Engineer; Multiverse Computing Sr MLOps) reinforce the same picture. Two adjacent signals — MCP / A2A protocols and named agent-framework fluency (LangGraph, CrewAI, AutoGen, Pydantic AI) — are firming up in *Agentic AI Engineer* postings but do not yet clear the ≥30% frequency bar at L30 ML Platform titles. Added to the [Watch List](#watch-list-re-evaluate-next-cycle). See [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json) for this cycle's empty delta and rationale.

Compared to the prior cycle, the AI/ML Platform Engineer role has materially shifted toward **GenAI platform ownership**. Across 34 postings, three patterns are now genuinely table-stakes for this role and were largely absent a year ago:

1. **LLM serving as a first-class platform runtime** — vLLM, SGLang, TensorRT-LLM, and KServe are no longer "the inference team's tools"; ML Platform Engineers are expected to integrate them as platform-managed serving runtimes with continuous batching, KV-cache awareness, and capacity planning (Reddit, Anthropic, Fireworks, DoorDash, Red Hat, Stripe, Roblox, Together AI, OpenAI, Samsara).
2. **LLM Gateway / model routing as a platform feature** — token/rate limits, intelligent failover across providers, prompt/version management, and cost attribution show up as explicit responsibilities (Reddit Sr Staff GenAI Platform, Anthropic Cloud Inference, DoorDash Principal, Mixpanel, Stripe, Match Group).
3. **Evaluation harnesses and agent observability** — LLM-as-judge pipelines, eval dataset curation, agent tracing, and tool-use logging are now a platform-team surface (Glean, Mixpanel, Anthropic Reward Models Platform, Reddit, DoorDash).

The classic ML platform surfaces (feature stores, model registries, workflow orchestration, multi-tenancy, developer experience, SDKs) remain core — these are already well covered by Modules 01–09 and Projects 01–05.

---

## Requirement Coverage Index

| ID | Category | Summary | Owner | Status |
|----|----------|---------|-------|--------|
| `req-platform-fundamentals` | Platform Architecture | Multi-tenant paved-road platform design | This repo (L30) | ✅ Covered |
| `req-api-sdk-design` | API & SDK | REST/gRPC APIs, Python SDKs, CLI | This repo (L30) | ✅ Covered |
| `req-multi-tenancy` | Multi-Tenancy | K8s isolation, quotas, RBAC, cost allocation | This repo (L30) | ✅ Covered |
| `req-feature-store` | Feature Stores | Online/offline, point-in-time, drift | This repo (L30) | ✅ Covered |
| `req-workflow-orchestration` | Orchestration | Airflow/Kubeflow/Dagster DAGs | This repo (L30) | ✅ Covered |
| `req-model-registry` | Model Lifecycle | Registry, versioning, lineage, A/B | This repo (L30) | ✅ Covered |
| `req-developer-experience` | DX | Paved-road, SDK/CLI, treat MLEs as customers | This repo (L30) | ✅ Covered |
| `req-observability` | Observability | Metrics, logs, traces, SLO/SLI | AI Infra Engineer (L20) | ↳ Covered at lower level |
| `req-security-governance` | Security | SSO, RBAC, audit, compliance | AI Infra Security (L35) | ↳ Covered at peer level |
| `req-kubernetes-fundamentals` | Infra | K8s deployments, CRDs, operators | AI Infra Engineer (L20) | ↳ Covered at lower level |
| `req-distributed-training` | Infra | Distributed training mechanics | AI Infra Engineer (L20) | ↳ Covered at lower level |
| `req-inference-engine-optimization` | Performance | CUDA, KV cache, attention internals | AI Infra Performance (L35) | ↳ Covered at peer level |
| `req-multi-cloud-capacity` | Operations | Multi-region serving strategy | AI Infra Architect (L45) | ↳ Covered at higher level |
| `req-incident-response` | Operations | On-call, runbooks, error budgets | AI Infra Engineer (L20) | ↳ Covered at lower level |
| `req-llm-serving-platform` | GenAI Platform | vLLM/SGLang/TensorRT-LLM/KServe as platform runtime | This repo (L30) | ⚠️ **Gap** |
| `req-llm-gateway` | GenAI Platform | LLM gateway, routing, token limits, prompt mgmt | This repo (L30) | ⚠️ **Gap** |
| `req-eval-harness-agent-observability` | GenAI Platform | Eval harnesses, LLM-as-judge, agent observability | This repo (L30) | ⚠️ **Gap** |
| `req-ray-on-kubernetes` | Orchestration | KubeRay self-service for users | This repo (L30) | ⚠️ Gap (exercise-level) |
| `req-rag-vector-platform` | GenAI Platform | RAG infra, vector DB, retrieval service | This repo (L30) | 🔭 Monitoring (evidence thin) |
| `req-research-data-platform` | Data Platform | Research-run lineage and analytics | This repo (L30) | 🔄 Partially covered |

---

## Group 1: Platform Architecture & Foundations

**Owner: This repo (Level 30)** — primary coverage.

Multi-tenant platform design, paved-road abstractions, build-vs-buy decisions, API-first thinking, and platform-as-product mindset show up in nearly every posting at this level.

**Evidence**:
- **Match Group (Hinge)** — *"Develop, maintain, and enhance frameworks for AI/ML model development and deployment while establishing best practices."* ([posting](https://jobs.lever.co/matchgroup/f6d1cdb5-efcd-4c13-bd51-570568138794))
- **Roblox** — *"The AI Platform team supports hundreds of ML use cases and billions of inferences daily."* ([posting](https://careers.roblox.com/jobs/7403998))
- **DoorDash Principal** — *"Define the technical backbone that powers all machine learning and generative AI workloads across DoorDash."* ([posting](https://careersatdoordash.com/jobs/principal-engineer-ai-ml-platform/7669280/))
- **Samsara** — *"Design, build, and operate Samsara's end-to-end ML platform (training, experimentation, batch/online inference, edge)."* ([posting](https://job-boards.greenhouse.io/samsara/jobs/7721193))

**Coverage**: [Module 01 — Platform Fundamentals](./lessons/mod-001-platform-fundamentals/) and [Project 01 — Platform Core](./projects/project-01-platform-core/) cover this fully.

---

## Group 2: APIs, SDKs, and Developer Tooling

**Owner: This repo (Level 30)** — primary coverage.

Across the postings, ML Platform Engineers own REST + gRPC APIs, Python SDKs, CLI tools, and OpenAPI documentation. The differentiator from generic backend roles is the *internal customer* framing — ML engineers and data scientists are users, not just consumers.

**Evidence**:
- **Reddit ML Training Platform** — *"Treat internal MLEs as your customers. Conduct user research, reduce friction in the 'Idea-to-Prototype' loop."* ([posting](https://job-boards.greenhouse.io/reddit/jobs/7074776))
- **Affirm Feature Platform** — *"Grow Affirm's business by building a delightful, self-serve platform for developing and serving data for machine learning and decisioning."* ([posting](https://job-boards.greenhouse.io/affirm/jobs/7694294003))
- **Stripe** — *"Build scalable, reliable services for notebooks, ML model training, experimentation, serving, and LLM applications."* ([posting](https://stripe.com/jobs/listing/software-engineer-machine-learning-infrastructure/7528260))
- **Pipedrive** — *"Build and maintain ML Platform components and frameworks used by Data Scientists and ML Engineers."* ([posting](https://jobs.lever.co/pipedrive/3a9016fd-262a-49cd-bedb-c36a8ed652d4))

**Coverage**: [Module 02 — API Design](./lessons/mod-002-api-design/), [Module 07 — Developer Experience](./lessons/mod-007-developer-experience/), and [Project 05 — Developer Portal & SDK](./projects/project-05-developer-portal/).

---

## Group 3: Multi-Tenancy & Resource Management

**Owner: This repo (Level 30)** — primary coverage.

Multi-tenant ML platforms with namespace isolation, resource quotas (CPU/GPU/memory), RBAC, and cost allocation are explicit requirements at most postings at this level.

**Evidence**:
- **Reddit GenAI Platform** — multi-tenant GenAI platform leadership ([posting](https://job-boards.greenhouse.io/reddit/jobs/7772274))
- **DoorDash Principal** — *"Multi-tenant ML platform at scale."* ([posting](https://careersatdoordash.com/jobs/principal-engineer-ai-ml-platform/7669280/))
- **Match Group** — *"Lead build-vs-buy discussions; participate in on-call."* ([posting](https://jobs.lever.co/matchgroup/f6d1cdb5-efcd-4c13-bd51-570568138794))

**Coverage**: [Module 03 — Multi-Tenancy & Resource Management](./lessons/mod-003-multi-tenancy-resources/) and [Project 01 — Platform Core](./projects/project-01-platform-core/).

---

## Group 4: Feature Stores

**Owner: This repo (Level 30)** — primary coverage.

Feature platforms continue to be a core ML Platform deliverable: online stores (<10ms p99), offline stores with point-in-time correctness, drift detection, and feature lineage. Feast and Tecton are the most-cited tools.

**Evidence**:
- **Affirm** — feature platform as their entire team mission ([posting](https://job-boards.greenhouse.io/affirm/jobs/7694294003))
- **Match Group** — *"Build millisecond real-time predictions at scale."* ([posting](https://jobs.lever.co/matchgroup/f6d1cdb5-efcd-4c13-bd51-570568138794))
- **Pinterest** — *"Unified feature store and model understanding tools."* ([posting](https://www.pinterestcareers.com/jobs/6865409/sr-staff-software-engineer-ml-platform/))
- **Penn Interactive** — *"Virtual feature store"* listed as nice-to-have ([posting](https://job-boards.greenhouse.io/penninteractive/jobs/5445237004))

**Coverage**: [Module 04 — Feature Store Architecture](./lessons/mod-004-feature-store/) and [Project 02 — Enterprise Feature Store](./projects/project-02-feature-store/).

---

## Group 5: Workflow Orchestration

**Owner: This repo (Level 30)** — primary coverage.

DAG-based ML pipelines using Airflow, Kubeflow Pipelines, or Dagster appear in roughly half of the postings. The new wrinkle in 2026 is **Ray** (and KubeRay specifically) showing up as a self-service compute primitive for ML platform users.

**Evidence**:
- **Penn Interactive** — *"Orchestration: Dagster, Airflow, or Kubeflow."* ([posting](https://job-boards.greenhouse.io/penninteractive/jobs/5445237004))
- **Anthropic Research Data Platform** — large-scale ETL pipelines for training run data ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5191226008))
- **Samsara** — *"Distributed computing frameworks (Ray and/or Spark)."* ([posting](https://job-boards.greenhouse.io/samsara/jobs/7721193))
- **Reddit ML Training Platform** — *"Distributed training (Ray, Kubernetes)."* ([posting](https://job-boards.greenhouse.io/reddit/jobs/7074776))
- **Afresh Staff** — *"Architect high-throughput distributed inference (Spark, Dask, Ray)."* ([posting](https://job-boards.greenhouse.io/afresh/jobs/5706905004))

**Coverage**: [Module 05 — Workflow Orchestration](./lessons/mod-005-workflow-orchestration/) and [Project 03 — ML Workflow Orchestration](./projects/project-03-workflow-orchestration/). A **proposed KubeRay exercise** within Module 05 addresses the Ray gap — see [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json).

---

## Group 6: Model Lifecycle & Registry

**Owner: This repo (Level 30)** — primary coverage.

MLflow remains the dominant tool, with KServe, Seldon, and Vertex AI / SageMaker showing up as integration points. Registry, lineage, promotion gates, A/B testing, and deployment strategies are explicit requirements.

**Evidence**:
- **Penn Interactive** — *"ML packaging/serving: MLflow, Seldon, KServe, Vertex, SageMaker."* ([posting](https://job-boards.greenhouse.io/penninteractive/jobs/5445237004))
- **Samsara** — *"Model lifecycle: registry, deployment, monitoring."* ([posting](https://job-boards.greenhouse.io/samsara/jobs/7721193))
- **Fireworks AI** — *"ML infrastructure (PyTorch, MLflow, Vertex AI, SageMaker, Kubernetes)."* ([posting](https://job-boards.greenhouse.io/fireworksai/jobs/4056271009))

**Coverage**: [Module 06 — Model Management & Registry](./lessons/mod-006-model-management/) and [Project 04 — Model Registry](./projects/project-04-model-registry/).

---

## Group 7: GenAI / LLM Serving Platform — **NEW GAP**

**Owner: This repo (Level 30)** — gap; new module proposed.

This is the largest 2026 shift. LLM serving engines (vLLM, SGLang, TensorRT-LLM, KServe with LLM-D) are no longer "the inference team's tools" — ML Platform Engineers are expected to integrate them as platform-managed serving runtimes with continuous batching, KV-cache awareness, prefill/decode disaggregation, and quantization rollout patterns.

> **Ownership boundary**: kernel-level CUDA optimization, attention math, and inference engine internals belong to the [AI Infrastructure Performance Engineer](https://github.com/ai-infra-curriculum/ai-infra-performance-learning) (Level 35). The ML Platform Engineer integrates these engines *as platform consumers* — choosing serving runtimes, autoscaling them, exposing them via SDK abstractions, and managing capacity. The new module proposal sticks to that boundary.

**Evidence (11 postings)**:
- **Red Hat** — *"Tackle challenges in scalable inference systems and Kubernetes-native deployments via vLLM and LLM-D."* ([posting](https://redhat.wd5.myworkdayjobs.com/en-US/Jobs/job/Principal-Machine-Learning-Engineer--Distributed-vLLM-Inference_R-050962-1))
- **Multiverse Computing** — *"Lead the deployment of LLMs using vLLM, TensorRT-LLM, or SGLang. You will implement and tune cutting-edge techniques — including PagedAttention, continuous batching, and advanced quantization."* ([posting](https://relocate.me/spain/san-sebastian/multiverse-computing/senior-mlops-engineer-training-inference-optimization-10243)) — *added 2026-07-04*
- **Together AI** — *"Hands-on LLM serving engines: vLLM, SGLang, TensorRT-LLM."* ([posting](https://job-boards.greenhouse.io/togetherai/jobs/5088817007))
- **Fireworks AI Cloud Infra** — *"Spearhead the creation of one of the world's first virtual clouds, seamlessly serving AI workloads across the globe."* ([posting](https://job-boards.greenhouse.io/fireworksai/jobs/4005894009))
- **Anthropic Cloud Inference** — *"End-to-end product of Claude on each cloud platform, from API integration and intelligent request routing to inference execution."* ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5107466008))
- **DoorDash Principal** — *"GenAI/LLM platform design and rollout."* ([posting](https://careersatdoordash.com/jobs/principal-engineer-ai-ml-platform/7669280/))
- **OpenAI** — *"Lead shared internal inference stack work."* ([posting](https://jobs.ashbyhq.com/openai/60db4c1d-0285-4fb0-9453-4a3ef50102b0))
- **Roblox** — *"Distributed inference systems for billions of inferences daily."* ([posting](https://careers.roblox.com/jobs/7403998))
- **Samsara** — *"Computer vision and/or LLM production systems."* ([posting](https://job-boards.greenhouse.io/samsara/jobs/7721193))
- **Stripe** — *"Build scalable, reliable services for notebooks, ML model training, experimentation, serving, and LLM applications."* ([posting](https://stripe.com/jobs/listing/software-engineer-machine-learning-infrastructure/7528260))
- **Fireworks AI Infra** — *"Basic LLM knowledge: context length, prefill, KV cache."* ([posting](https://job-boards.greenhouse.io/fireworksai/jobs/4056271009))

**Coverage**: ⚠️ **Gap** — proposed `mod-010-llm-serving-platform` and a project addition in [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json).

**External resources** (until the module lands):
- [vLLM Documentation](https://docs.vllm.ai/)
- [SGLang Documentation](https://docs.sglang.ai/)
- [KServe LLM Inference Service](https://kserve.github.io/website/latest/modelserving/v1beta1/llm/)
- [NVIDIA TensorRT-LLM](https://nvidia.github.io/TensorRT-LLM/)
- [Red Hat LLM-D project](https://github.com/llm-d/llm-d)

---

## Group 8: LLM Gateway & Routing — **NEW GAP**

**Owner: This repo (Level 30)** — gap; new module proposed.

LLM Gateways are showing up as discrete platform deliverables: a routing layer in front of one or many model providers (internal serving fleets + external APIs) that handles token/rate limits, intelligent failover, prompt/version management, response caching, and cost attribution per team. This was almost never mentioned in the prior research cycle.

**Evidence (7 postings)**:
- **Reddit Sr Staff GenAI Platform** — *"LLM Gateway design and implementation; rate/token limit management, intelligent failover."* ([posting](https://job-boards.greenhouse.io/reddit/jobs/7772274))
- **Anthropic Cloud Inference** — *"API integration and intelligent request routing to inference execution."* ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5107466008))
- **DoorDash Principal** — *"Define the technical backbone that powers all machine learning and generative AI workloads across DoorDash."* ([posting](https://careersatdoordash.com/jobs/principal-engineer-ai-ml-platform/7669280/))
- **Mixpanel AI Platform** — *"Build the scalable, secure, and reliable infrastructure that accelerates AI agent development."* ([posting](https://job-boards.greenhouse.io/mixpanel/jobs/7941944))
- **Stripe** — LLM application platform support ([posting](https://stripe.com/jobs/listing/software-engineer-machine-learning-infrastructure/7528260))
- **Match Group** — *"Support for generative AI models on the platform"; "Mentor ML/Backend/PM partners on MLOps and GenAI tooling."* ([posting](https://jobs.lever.co/matchgroup/f6d1cdb5-efcd-4c13-bd51-570568138794))
- **Samsara** — LLM production systems on a multi-team platform ([posting](https://job-boards.greenhouse.io/samsara/jobs/7721193))

**Coverage**: ⚠️ **Gap** — proposed `mod-011-llm-gateway-routing` in [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json).

**External resources** (until the module lands):
- [LiteLLM Proxy](https://docs.litellm.ai/docs/simple_proxy)
- [Portkey AI Gateway](https://portkey.ai/docs)
- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)

---

## Group 9: Eval Harnesses & Agent Observability — **NEW GAP**

**Owner: This repo (Level 30)** — gap; new module proposed.

Building evaluation harnesses for LLM-based features, curating eval datasets, designing LLM-as-judge scoring pipelines, and shipping agent observability (distributed tracing across tool calls, regression detection on offline scores) are now platform-team owned surfaces.

> **Ownership boundary**: research-quality model evaluation belongs to applied scientists. The ML Platform Engineer builds the *platform* — the data pipeline that scores production traffic, the dashboards showing regression vs. baseline, the SDK consumers call to log a custom eval, and the gating logic that blocks promotion on red metrics.

**Evidence (5 postings)**:
- **Glean** — *"Design and curate evaluation datasets; build large-scale evaluation pipelines for quality measurement; develop LLM-powered scoring judges; create observability infrastructure for AI agents."* ([posting](https://job-boards.greenhouse.io/gleanwork/jobs/4669417005))
- **Mixpanel** — *"Agent optimization and evaluation systems"; "AI-native systems built at production scale."* ([posting](https://job-boards.greenhouse.io/mixpanel/jobs/7941944))
- **Anthropic Reward Models Platform** — *"Build the tools and infrastructure that enable researchers across the organization to develop, evaluate, and optimize reward signals."* ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5024831008))
- **Reddit GenAI Platform** — *"Observability and monitoring for AI systems."* ([posting](https://job-boards.greenhouse.io/reddit/jobs/7772274))
- **DoorDash Principal** — paved-road developer experience for GenAI workloads ([posting](https://careersatdoordash.com/jobs/principal-engineer-ai-ml-platform/7669280/))

**Coverage**: ⚠️ **Gap** — proposed `mod-012-eval-agent-observability` in [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json).

**External resources** (until the module lands):
- [OpenAI Evals](https://github.com/openai/evals)
- [Anthropic Evals Guide](https://docs.anthropic.com/en/docs/build-with-claude/evals)
- [LangSmith / Langfuse / Helicone](https://github.com/langfuse/langfuse)
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)

---

## Group 10: RAG Infrastructure & Vector Platforms — **Monitoring (not yet a gap)**

**Owner: This repo (Level 30)** — emerging area; evidence not yet strong enough for a new module.

Vector databases (Pinecone, Weaviate, pgvector, Qdrant), retrieval service abstractions, and chunking/embedding pipelines as platform features are mentioned but mostly described as "use these tools" rather than "build these abstractions for your platform." Three postings explicitly frame it as a platform-team deliverable.

**Evidence**:
- **Reddit GenAI Platform** — *"RAG system architecture."* ([posting](https://job-boards.greenhouse.io/reddit/jobs/7772274))
- **Mixpanel** — AI-native infrastructure including RAG-style retrieval ([posting](https://job-boards.greenhouse.io/mixpanel/jobs/7941944))
- **Roblox** — *"Generative AI information retrieval."* ([posting](https://careers.roblox.com/jobs/7403998))

**Coverage**: 🔭 **Monitoring** — evidence is at the floor (3 postings); per the contract, evidence-thin items go in the watch list rather than triggering a new module. Re-evaluate next cycle.

**External resources** (in the interim):
- [Pinecone Architecture](https://docs.pinecone.io/docs/architecture)
- [pgvector](https://github.com/pgvector/pgvector)
- [LangChain RAG patterns](https://python.langchain.com/docs/concepts/rag/)

---

## Group 11: Research Data Platform — **Partially covered, monitoring**

**Owner: This repo (Level 30)** — partially covered; evidence too thin to expand.

Pipelines from research training runs to storage, experiment metadata, and researcher-facing query/visualization tools are explicit at AI labs (Anthropic) but appear in only 2 postings — too few to justify a new module per the contract.

**Evidence**:
- **Anthropic Research Data Platform** — *"Build and operate data pipelines that extract data from research training runs and land it in storage systems."* ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5191226008))
- **Anthropic Reward Models Platform** — researcher-facing data tooling ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5024831008))

**Coverage**: 🔄 **Partial** — Module 06 covers lineage and metadata. The researcher-facing analytics surface is out of scope this cycle.

---

## Group 12: Observability — **Owned by AI Infrastructure Engineer (L20)**

**Owner: [`ai-infra-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning)** (Level 20).

General observability (Prometheus, Grafana, structured logging, OpenTelemetry tracing, SLO/SLI design) is a Level-20 skill that this role builds on. This repo's [Module 08](./lessons/mod-008-observability/) covers the **platform-specific extensions** — training queue depth, inference p99 latency, model drift signals, multi-tenant cost dashboards — that build on the lower-level fundamentals.

**Evidence at this role**: Match Group, Reddit GenAI Platform, Anthropic Cloud Inference, Fireworks AI Cluster Mgmt, Mixpanel — but every posting treats observability as a *prerequisite*, not the differentiator.

---

## Group 13: Security & Governance — **Owned by AI Infrastructure Security Engineer (L35) and AI Infrastructure Engineer (L20)**

**Owner: [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning)** (Level 35) for deep security ownership; [`ai-infra-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning) for fundamentals.

SSO/OIDC, RBAC, audit logging, secrets management, GDPR/HIPAA compliance hooks. This repo's [Module 09](./lessons/mod-009-security-governance/) covers platform integration patterns; full ownership of policy, threat modeling, and red-teaming sits at the peer-level Security role.

**Evidence at this role**: Match Group, Profluent Bio, Penn Interactive — security shows up as a competency, not a primary focus area.

---

## Group 14: Kubernetes Fundamentals & Distributed Training — **Owned by AI Infrastructure Engineer (L20)**

**Owner: [`ai-infra-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning)** (Level 20).

Core Kubernetes (deployments, services, ingress), container orchestration basics, distributed training mechanics (PyTorch DDP, NCCL, Horovod), GPU drivers, and CUDA basics belong at Level 20. This repo assumes those as prerequisites and builds **platform-level** abstractions on top: ML CRDs and operators (Project 01), Ray-on-Kubernetes self-service (proposed in Module 05), and LLM serving runtimes (proposed in new Module 10).

**Evidence at this role**: nearly every posting requires Kubernetes; almost no posting requires teaching it. The shape of the requirement is *operator development* and *platform abstraction design*, not cluster admin. NVIDIA's mid-2026 Sr ML Platform posting explicitly frames the platform surface as *"Design, build, and maintain our core ML platform infrastructure as code, primarily using Ansible and Terraform, ensuring reproducibility"* ([posting](https://www.builtinsf.com/job/senior-ml-platform-engineer/9636008)) — reinforcing that IaC and operator patterns are the L30 shape.

---

## Group 15: Inference Engine Optimization — **Owned by AI Infrastructure Performance Engineer (L35)**

**Owner: [`ai-infra-performance-learning`](https://github.com/ai-infra-curriculum/ai-infra-performance-learning)** (Level 35).

CUDA kernel writing, attention math, KV cache implementation details, quantization algorithm design, and prefill/decode scheduler design belong to the Performance Engineer peer role. The ML Platform Engineer integrates the *output* of that work (vLLM, SGLang, TensorRT-LLM engines) as platform-managed runtimes — see Group 7.

**Evidence at this role**: NVIDIA, Together AI, Red Hat — postings that touch kernel internals trend toward Performance Engineer titling even when filed as "ML" or "AI" roles.

---

## Group 16: Multi-Cloud & Capacity Planning — **Owned by AI Infrastructure Architect (L45)**

**Owner: [`ai-infra-architect-learning`](https://github.com/ai-infra-curriculum/ai-infra-architect-learning)** (Level 45).

Multi-cloud deployment strategy, multi-region serving topology, and large-scale capacity planning are architecture-level decisions made above this role. ML Platform Engineers *implement and operate within* those decisions.

**Evidence at this role**: Anthropic Cloud Inference, Fireworks Cloud Infra, DoorDash Principal — all are Staff+ or Principal+ scoped, where the architecture context begins to dominate.

---

## Group 17: Incident Response & On-Call — **Owned by AI Infrastructure Engineer (L20)**

**Owner: [`ai-infra-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning)** (Level 20).

On-call rotations, runbook authoring, error budgets, blameless postmortems. This role participates in on-call and authors runbooks for platform services, but the underlying SRE practice is owned at Level 20.

**Evidence at this role**: Match Group, Anthropic Cloud Inference, Fireworks Cluster Mgmt — on-call participation is table stakes, not the focus.

---

## Proposed Curriculum Additions

Three new modules and one new project are proposed for the next merge cycle, each with at least 3 cited postings. See [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json) for the full proposal.

| Proposed | Title | Citations | Maps to gap |
|----------|-------|-----------|-------------|
| `mod-010-llm-serving-platform` | LLM Serving as a Platform Runtime | 10 | `req-llm-serving-platform` |
| `mod-011-llm-gateway-routing` | LLM Gateway, Routing & Prompt Management | 7 | `req-llm-gateway` |
| `mod-012-eval-agent-observability` | Eval Harnesses & Agent Observability | 5 | `req-eval-harness-agent-observability` |
| `project-06-genai-platform` | GenAI Platform: Gateway + Eval Harness + Serving | 8 | All three gaps above |
| `mod-005 exercise: KubeRay` | KubeRay self-service for platform users | 3 | `req-ray-on-kubernetes` |

The deltas are **strictly additive** and will not auto-merge — they go through human review with per-run caps applied by the runner.

---

## Watch List (Re-evaluate Next Cycle)

These signals are present but evidence remains too thin for new modules under the current contract:

- **RAG infrastructure as platform deliverable** — 3 postings; expect this to firm up.
- **Research-facing data platform / experiment metadata** — concentrated at AI labs.
- **Edge / on-device ML platform support** — Samsara explicitly, but otherwise rare at the L30 ML Platform role.
- **Voice/streaming AI infra** — Together AI, Cartesia, Deepgram — niche but growing.
- **MCP / A2A protocols as platform surface** *(new 2026-07)* — Model Context Protocol shows up strongly in *Agentic AI Engineer* postings (GoFundMe explicit) and as a vendor feature (Greenhouse MCP launched 2026-05-07). Not yet a headline requirement at L30 ML Platform titles. If this migrates from Agentic-AI titles into ML-Platform titles by the next cycle, expect it to land under the existing eval/agent observability gap (`mod-012` proposal) rather than as its own module.
- **Named agent-framework fluency** *(new 2026-07)* — LangGraph, CrewAI, AutoGen, Pydantic AI, Google ADK are now called out by name in Agentic AI postings (GoFundMe explicit). Concentrated in application-engineer titles; adjacent to but not yet inside the L30 ML Platform surface.

---

*Last updated: 2026-07-04. Next refresh: see [`.aicg/research/runs/ml-platform/research-run.env`](./.aicg/research/runs/ml-platform/research-run.env).*
