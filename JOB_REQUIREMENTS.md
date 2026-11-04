# Job Requirements — AI/ML Platform Engineer

> **Role**: AI/ML Platform Engineer (Level 30)
> **Research window**: 2026-08-05 → 2026-11-04 (last 90 days)
> **Postings analyzed**: 62 distinct postings across 35 employers (3 new postings added this cycle; 2026-11 was another mid-cycle refresh two months after the comprehensive 2026-09 scan)
> **Last refreshed**: 2026-11-04

This document captures the recurring requirements observed in current job postings for the **AI/ML Platform Engineer** role, maps them to learning coverage in this repository, and links out to lower-level repositories when a requirement is genuinely owned elsewhere in the role hierarchy.

The **Ownership Rule** governs cross-role placement: if a requirement applies to multiple roles, the primary coverage lives in the *lowest-level* repository where it is genuinely required. Higher-level roles link to that owner unless they need additional depth, architectural context, or leadership framing.

For raw, machine-readable findings (postings, requirements, evidence, coverage paths), see [`.aicg/job-requirements.json`](./.aicg/job-requirements.json).

---

## Summary of the 2026 Shift

**2026-11 mid-cycle refresh note**: A targeted spot-check of the 2026-10-05 → 2026-11-04 window (two months after the comprehensive 2026-09 scan) surfaces no material shift. The three standing GenAI-platform gaps (LLM serving runtime integration, LLM Gateway & routing, eval harnesses + agent observability + absorbed MCP/named-agent-framework scope) remain the correct outstanding story and the standing 2026-06 proposal set (`mod-010`, `mod-011`, `mod-012`, `project-06`, `mod-005` KubeRay exercise) remains the correct fix. Three new in-window postings verified this cycle: **DoorDash SWE ML Platform Gen AI** (p060, 2026-10-08) — a rare triple-hit JD citing vLLM/SGLang/TensorRT-LLM, LLM Gateway + Agent Gateway + evals infrastructure + guardrails + cost attribution, and fine-tuning (SFT/DPO/LoRA) together as headline responsibilities; **DoorDash SWE Data and AI Platform** (p061, 2026-10-14) — MCP + skills layer + agent frameworks + tool-calling architectures; **Netflix Staff/Principal SWE L6/L7, AI Platform Model Serving** (p062, 2026-10-27) — unified serving layer spanning in-house inference, third-party model hosting, and external LLM providers with cross-provider routing and A/B experimentation. Two faint new sub-signals appeared — an **Agent Gateway** surface distinct from LLM Gateway, and **per-tenant GenAI cost attribution** — but both are single-employer (DoorDash) and best absorbed into the standing `mod-011` scope rather than opening new requirements. The **fine-tuning platform / LoRA-adapter-serving** sub-trend firms slightly (DoorDash p060 now explicitly names SFT/DPO/LoRA alongside serving) but still single-digit employer coverage at strict L30 ML-Platform titles — stays on Watch List, absorbed by `mod-010` scope when scaffolded. Voice/streaming inference and RAG-as-platform remain on the Watch List (unchanged). **Empty delta this cycle.** See [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json).

**2026-10 mid-cycle refresh note**: A targeted spot-check of the 2026-09-05 → 2026-10-04 window (one month after the comprehensive 2026-09 scan) surfaces no material shift. The three standing GenAI-platform gaps (LLM serving runtime integration, LLM Gateway & routing, eval harnesses + agent observability + absorbed MCP/named-agent-framework scope) remain the correct outstanding story and the standing 2026-06 proposal set (`mod-010`, `mod-011`, `mod-012`, `project-06`, `mod-005` KubeRay exercise) remains the correct fix. One new in-window posting verified this cycle — **Faire Sr Staff ML Platform Engineer** (p059) — reinforces classical ML Platform coverage (Databricks lakehouse, Spark, MLflow, Kubernetes, IaC) with LLM workflow integration mentioned but not as a dedicated LLM-serving/gateway surface. External posting-listings (EXL "Hybrid AI Platform Engineer - LLM Gateway & Observability", ALTCloud "AI Platform Engineer – MCP / Model Gateway / GenAI", Synopsys "AI Platform Engineer (MCP, Python/GO, LLM)") corroborate the standing gateway + MCP picture at non-headline-brand employers but add no new signal class. A **fine-tuning platform / LoRA-adapter-serving** sub-trend (Predibase/LoRAX, Fireworks, Together AI) is firming in external discourse but at strict generalist L30 ML-Platform titles still folds into existing serving/registry surfaces (Stripe p046 names LLM fine-tuning as one of many platform surfaces) — not clearing the ≥30% bar for a standalone module; absorbed by `mod-010` scope when scaffolded. **Empty delta this cycle.** See [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json).

**2026-09 cycle note**: A fresh 90-day scan (2026-06-06 → 2026-09-04) confirms the three standing GenAI-platform gaps identified in the 2026-06 cycle — LLM serving runtime integration (vLLM/SGLang/TensorRT-LLM/KServe), LLM Gateway & routing, and eval harnesses + agent observability — remain the correct outstanding proposals. 24 new postings were added this cycle across DoorDash, Anthropic (3), OpenAI, Together AI (3), Groq, Reddit (2), Stripe, Samsara (2), Netflix, Scale AI (2), Airbnb, Robinhood, Instacart, Cartesia, PAR, OnePay, and Kaseya. The one **material change from 2026-07** is that the two prior-cycle Watch List items — **MCP / A2A protocols** and **named agent-framework fluency (LangGraph, LangChain, CrewAI, AutoGen)** — have migrated from Agentic-AI titles into L30 ML/AI Platform titles: 5+ platform-titled postings each in-window cite them as headline requirements (DoorDash GenAI ML Infra, Anthropic Platform Connectivity, Samsara AI Platform, PAR, OnePay, Kaseya). Per the 2026-07 forward-guidance (*"if MCP migrates from Agentic-AI titles into ML-Platform titles by next cycle, expect it to land under the existing mod-012 proposal rather than as its own module"*), **both signals are absorbed by the standing proposed `mod-012-eval-agent-observability` module — as within-module exercises, not new modules.** Empty delta this cycle. Voice/streaming inference infra firmed at voice-specialized employers (Together AI, Cartesia, Deepgram) but remains under 10% at generalist L30 titles — stays on Watch List. RAG-as-platform saw its first firmer in-window platform-titled hits (Reddit Indexing & Retrieval Platform, Scale AI MLE Platform) but is still under the ≥30% bar at generalist titles — stays on Watch List. See [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json) for the empty delta and full rationale.

**2026-07 cycle note (retained for continuity)**: A market re-check for the 2026-04-05 → 2026-07-04 window found no material shift since the 2026-06-01 refresh. The three gaps identified in June remained the standing story; two new postings (NVIDIA Sr ML Platform Engineer; Multiverse Computing Sr MLOps) reinforced the same picture. MCP and named-agent-framework signals were added to the Watch List; both have now firmed up in the 2026-09 cycle (see above).

Compared to the prior cycle, the AI/ML Platform Engineer role has materially shifted toward **GenAI platform ownership**. Across 58 postings analyzed to date, three patterns are now genuinely table-stakes for this role and were largely absent a year ago:

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
| `req-llm-serving-platform` | GenAI Platform | vLLM/SGLang/TensorRT-LLM/KServe as platform runtime | This repo (L30) | ⚠️ **Gap** (mod-010 proposed) |
| `req-llm-gateway` | GenAI Platform | LLM gateway, routing, token limits, prompt mgmt | This repo (L30) | ⚠️ **Gap** (mod-011 proposed) |
| `req-eval-harness-agent-observability` | GenAI Platform | Eval harnesses, LLM-as-judge, agent observability, **MCP tracing, agent-framework support** | This repo (L30) | ⚠️ **Gap** (mod-012 proposed; scope expanded) |
| `req-mcp-a2a-protocols` | GenAI Platform | MCP proxy, tool interoperability, A2A protocols | This repo (L30) | ⚠️ **Gap** — absorbed by mod-012 |
| `req-named-agent-frameworks` | GenAI Platform | LangGraph/LangChain/CrewAI/AutoGen as supported runtimes | This repo (L30) | ⚠️ **Gap** — absorbed by mod-012 |
| `req-ray-on-kubernetes` | Orchestration | KubeRay self-service for users | This repo (L30) | ⚠️ Gap (exercise-level) |
| `req-rag-vector-platform` | GenAI Platform | RAG infra, vector DB, retrieval service | This repo (L30) | 🔭 Monitoring (evidence still thin at generalist titles) |
| `req-voice-streaming-platform` | GenAI Platform | Streaming voice/audio inference infra, WebSocket APIs | This repo (L30) | 🔭 Monitoring (concentrated at voice-native cos) |
| `req-edge-ondevice-platform` | GenAI Platform | Edge model packaging, edge↔cloud feedback | This repo (L30) | 🔭 Monitoring (single-employer concentration) |
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
- **Faire Sr Staff ML Platform Engineer** *(new 2026-10)* — *"Own the technical vision and evolution of the ML platform; set standards and influence org-wide architecture; lead complex cross-functional initiatives."* ([posting](https://job-boards.greenhouse.io/faire/jobs/8731172002))
- **Netflix Staff/Principal SWE (L6/L7), AI Platform Model Serving** *(new 2026-11)* — *"Re-building a new model serving platform from the ground up that unifies in-house inference infrastructure, third-party model hosting, and external LLM providers."* ([posting](https://explore.jobs.netflix.net/careers/job/790316811164))

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

**Evidence (19 postings including 2 new in 2026-11 window)**:
- **DoorDash SWE, ML Platform — Gen AI** *(new 2026-11)* — *"Experience with LLM inference engines and serving frameworks (e.g., vLLM, SGLang, TensorRT-LLM) in production."* ([posting](https://job-boards.greenhouse.io/doordashusa/jobs/8264542))
- **Netflix Staff/Principal SWE (L6/L7), AI Platform Model Serving** *(new 2026-11)* — *"Re-building a new model serving platform from the ground up that unifies in-house inference infrastructure, third-party model hosting, and external LLM providers under a single serving, routing, and experimentation layer."* ([posting](https://explore.jobs.netflix.net/careers/job/790316811164))
- **Red Hat** — *"Tackle challenges in scalable inference systems and Kubernetes-native deployments via vLLM and LLM-D."* ([posting](https://redhat.wd5.myworkdayjobs.com/en-US/Jobs/job/Principal-Machine-Learning-Engineer--Distributed-vLLM-Inference_R-050962-1))
- **Together AI Sr Backend Eng, Inference Platform** *(new 2026-09)* — *"Open source inference ecosystem: SGLang, vLLM, or NVIDIA Dynamo (highly valuable)."* ([posting](https://job-boards.greenhouse.io/togetherai/jobs/4835763007))
- **Together AI Staff SWE, Inference/Compute Infra** *(new 2026-09)* — manifest-driven inference API via Kubernetes controllers ([posting](https://job-boards.greenhouse.io/togetherai/jobs/5186628007))
- **Groq Sr Staff SWE, HPI System** *(new 2026-09)* — *"FlashAttention, PagedAttention, continuous batching, speculative decoding, KV cache, prefill/decode scheduling."* ([posting](https://job-boards.greenhouse.io/groq/jobs/6010437003))
- **Anthropic Perf Eng, Inference Systems** *(new 2026-09)* — inference-fleet performance engineering as a platform surface ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5224564008))
- **Cartesia SWE, Platform** *(new 2026-09)* — *"Design and build low latency, scalable, and reliable model inference and serving stack for our cutting edge SSM foundation models."* ([posting](https://jobs.ashbyhq.com/cartesia/7628169d-3703-4b82-a1d0-ed3124790643))
- **Stripe Staff SWE, ML Platform** *(new 2026-09)* — *"low-latency model inference…LLM fine-tuning…LLM and agent orchestration."* ([posting](https://stripe.com/careers/listing/staff-software-engineer-machine-learning-platform/7939868))
- **Multiverse Computing** — *"Lead the deployment of LLMs using vLLM, TensorRT-LLM, or SGLang. You will implement and tune cutting-edge techniques — including PagedAttention, continuous batching, and advanced quantization."* ([posting](https://relocate.me/spain/san-sebastian/multiverse-computing/senior-mlops-engineer-training-inference-optimization-10243))
- **Together AI (2026-06 Voice AI role)** — *"Hands-on LLM serving engines: vLLM, SGLang, TensorRT-LLM."* ([posting](https://job-boards.greenhouse.io/togetherai/jobs/5088817007))
- **Fireworks AI Cloud Infra** — *"Spearhead the creation of one of the world's first virtual clouds, seamlessly serving AI workloads across the globe."* ([posting](https://job-boards.greenhouse.io/fireworksai/jobs/4005894009))
- **Anthropic Cloud Inference** — *"End-to-end product of Claude on each cloud platform, from API integration and intelligent request routing to inference execution."* ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5107466008))
- **DoorDash Principal** — *"GenAI/LLM platform design and rollout."* ([posting](https://careersatdoordash.com/jobs/principal-engineer-ai-ml-platform/7669280/))
- **OpenAI Platform ML Eng Mgr, Inference** — *"Lead shared internal inference stack work."* ([posting](https://jobs.ashbyhq.com/openai/60db4c1d-0285-4fb0-9453-4a3ef50102b0))
- **Roblox** — *"Distributed inference systems for billions of inferences daily."* ([posting](https://careers.roblox.com/jobs/7403998))
- **Samsara** — *"Computer vision and/or LLM production systems."* ([posting](https://job-boards.greenhouse.io/samsara/jobs/7721193))
- **Stripe (2026-06 posting)** — *"Build scalable, reliable services for notebooks, ML model training, experimentation, serving, and LLM applications."* ([posting](https://stripe.com/jobs/listing/software-engineer-machine-learning-infrastructure/7528260))
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

**Evidence (14 postings including 2 new in 2026-11 window)**:
- **DoorDash SWE, ML Platform — Gen AI** *(new 2026-11)* — *"Work on our open-weights serving stack… alongside the LLM Gateway, Agent Gateway, evals infrastructure, guardrails, and cost attribution."* ([posting](https://job-boards.greenhouse.io/doordashusa/jobs/8264542))
- **Netflix Staff/Principal SWE (L6/L7), AI Platform Model Serving** *(new 2026-11)* — unified serving + routing + experimentation layer where teams can deploy, route, and A/B test any model ([posting](https://explore.jobs.netflix.net/careers/job/790316811164))
- **DoorDash SWE, ML Infra – Gen AI** *(new 2026-09)* — LLM Gateway + cost attribution + guardrails as headline scope ([posting](https://jobs.khoslaventures.com/companies/doordash/jobs/83237566-software-engineer-machine-learning-infrastructure-gen-ai))
- **OpenAI MLE, API Multicloud** *(new 2026-09)* — *"extending OpenAI's API platform into strategic cloud environments"* — intelligent request routing across clouds ([posting](https://jobs.ashbyhq.com/openai/5acf4854-1d42-40ca-bff8-4f6f04cdce68))
- **Netflix Staff MLSE, Aims Platform Systems** *(new 2026-09)* — *"observability, cost, and platform subsystems that support next-generation AI workflows."* ([posting](https://www.showbizjobs.com/jobs/netflix-staff-ml-software-engineer-l6-platform-systems-aims-engineering-in-los-gatos/jid-6k7kq0))
- **Scale AI Sr Infra SWE, Enterprise AI** *(new 2026-09)* — *"agentic observability platform…architecting multi-cloud solutions"* ([posting](https://job-boards.greenhouse.io/scaleai/jobs/4599700005))
- **Stripe Staff SWE, ML Platform** *(new 2026-09)* — *"LLM and agent orchestration"* on the platform ([posting](https://stripe.com/careers/listing/staff-software-engineer-machine-learning-platform/7939868))
- **Reddit Sr Staff GenAI Platform** — *"LLM Gateway design and implementation; rate/token limit management, intelligent failover."* ([posting](https://job-boards.greenhouse.io/reddit/jobs/7772274))
- **Anthropic Cloud Inference** — *"API integration and intelligent request routing to inference execution."* ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5107466008))
- **DoorDash Principal** — *"Define the technical backbone that powers all machine learning and generative AI workloads across DoorDash."* ([posting](https://careersatdoordash.com/jobs/principal-engineer-ai-ml-platform/7669280/))
- **Mixpanel AI Platform** — *"Build the scalable, secure, and reliable infrastructure that accelerates AI agent development."* ([posting](https://job-boards.greenhouse.io/mixpanel/jobs/7941944))
- **Stripe (2026-06)** — LLM application platform support ([posting](https://stripe.com/jobs/listing/software-engineer-machine-learning-infrastructure/7528260))
- **Match Group** — *"Support for generative AI models on the platform"; "Mentor ML/Backend/PM partners on MLOps and GenAI tooling."* ([posting](https://jobs.lever.co/matchgroup/f6d1cdb5-efcd-4c13-bd51-570568138794))
- **Samsara (2026-06)** — LLM production systems on a multi-team platform ([posting](https://job-boards.greenhouse.io/samsara/jobs/7721193))

**Coverage**: ⚠️ **Gap** — proposed `mod-011-llm-gateway-routing` in [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json).

**External resources** (until the module lands):
- [LiteLLM Proxy](https://docs.litellm.ai/docs/simple_proxy)
- [Portkey AI Gateway](https://portkey.ai/docs)
- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)

---

## Group 9: Eval Harnesses & Agent Observability + MCP + Named Agent Frameworks — **EXPANDED GAP**

**Owner: This repo (Level 30)** — gap; module already proposed. **Scope expanded 2026-09** to include MCP tool-call tracing and named-agent-framework support.

Building evaluation harnesses for LLM-based features, curating eval datasets, designing LLM-as-judge scoring pipelines, and shipping agent observability (distributed tracing across tool calls, regression detection on offline scores) are now platform-team owned surfaces. **2026-09**: MCP proxy operation, MCP-based tool interoperability, and named-agent-framework fluency (LangGraph dominates; LangChain, CrewAI, AutoGen appear in kitchen-sink lists) have migrated from Agentic-AI-Engineer titles into L30 ML/AI Platform titles. Per the 2026-07 forward-guidance, these are absorbed into the same `mod-012` proposal as within-module exercises rather than triggering separate modules.

> **Ownership boundary**: research-quality model evaluation belongs to applied scientists. The ML Platform Engineer builds the *platform* — the data pipeline that scores production traffic, the dashboards showing regression vs. baseline, the SDK consumers call to log a custom eval, the MCP proxy that mediates tool calls, and the gating logic that blocks promotion on red metrics. Named-framework choice (LangGraph vs. LangChain vs. CrewAI) sits with application teams; the platform provides framework-agnostic observability and cost surfaces.

**Evidence (17 postings including 2 new in 2026-11 window)**:

*Core eval + agent observability:*
- **DoorDash SWE, ML Platform — Gen AI** *(new 2026-11)* — *"LLM Gateway, Agent Gateway, evals infrastructure, guardrails, and cost attribution"* as headline scope ([posting](https://job-boards.greenhouse.io/doordashusa/jobs/8264542))
- **DoorDash SWE, Data and AI Platform** *(new 2026-11)* — *"Semantic and Knowledge layer along with MCP and skills layer plus the full stack infrastructure"*; *"Experience with MCP, agent frameworks, or tool-calling architectures in production."* ([posting](https://job-boards.greenhouse.io/doordashusa/jobs/8192187))
- **Netflix Staff MLSE, Aims Platform Systems** *(new 2026-09)* — *"observability, cost, and platform subsystems that support next-generation AI workflows."* ([posting](https://www.showbizjobs.com/jobs/netflix-staff-ml-software-engineer-l6-platform-systems-aims-engineering-in-los-gatos/jid-6k7kq0))
- **Anthropic ML Infra, Safeguards Research** *(new 2026-09)* — *"Own the training, evaluation, and scoring workflows researchers use, with a focus on cutting the time between an idea and a result."* ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5364804008))
- **Scale AI Sr Infra SWE, Enterprise AI** *(new 2026-09)* — *"Agentic observability platform (logging, metrics, tracing, and analytics)."* ([posting](https://job-boards.greenhouse.io/scaleai/jobs/4599700005))
- **Glean** — *"Design and curate evaluation datasets; build large-scale evaluation pipelines for quality measurement; develop LLM-powered scoring judges; create observability infrastructure for AI agents."* ([posting](https://job-boards.greenhouse.io/gleanwork/jobs/4669417005))
- **Mixpanel** — *"Agent optimization and evaluation systems."* ([posting](https://job-boards.greenhouse.io/mixpanel/jobs/7941944))
- **Anthropic Reward Models Platform** — reward-model infra for researchers ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5024831008))
- **Reddit GenAI Platform** — *"Observability and monitoring for AI systems."* ([posting](https://job-boards.greenhouse.io/reddit/jobs/7772274))
- **DoorDash Principal** — paved-road developer experience for GenAI workloads ([posting](https://careersatdoordash.com/jobs/principal-engineer-ai-ml-platform/7669280/))

*MCP / A2A protocols (newly firmed 2026-09):*
- **DoorDash SWE, ML Infra – Gen AI** *(new 2026-09)* — *"Experience building and deploying MCP servers in production"* ([posting](https://jobs.khoslaventures.com/companies/doordash/jobs/83237566-software-engineer-machine-learning-infrastructure-gen-ai))
- **Anthropic Staff+ SWE, Platform Connectivity** *(new 2026-09)* — *"design, build, and operate the core connectivity infrastructure behind external tool calls — the MCP proxy and the OAuth and token management systems…contribute to the MCP standard and its ecosystem."* ([posting](https://job-boards.greenhouse.io/anthropic/jobs/5394948008))
- **Samsara Sr SWE II, AI Platform** *(new 2026-09)* — MCP-based tool interoperability, LangChain, prompt orchestration, multi-step reasoning ([posting](https://www.samsara.com/company/careers/roles/8050373))
- **PAR Sr AI Platform Eng** *(new 2026-09)* — *"MCP-compatible tool interoperability"* + LangGraph ([posting](https://jobs.ashbyhq.com/PAR%20Technology/c951556c-5c74-460f-b9bb-6a50e2867b3e))
- **OnePay Platform Eng, AI & Automation** *(new 2026-09)* — *"expertise with agent frameworks like LangGraph and MCP server development."* ([posting](https://jobs.ashbyhq.com/oneapp/a85bb5aa-9683-4d67-84e5-5d3e4bacda59))

*Named agent frameworks (newly firmed 2026-09):*
- **Kaseya Staff AI Platform Eng II** *(new 2026-09)* — *"agent frameworks such as LangChain, LangGraph, AutoGen, or CrewAI, and experience developing agent planning and reasoning algorithms."* ([posting](https://boards.greenhouse.io/embed/job_app?token=5589151004))
- **Robinhood Sr SWE, AI Platform** *(new 2026-09)* — *"LLMs and agentic systems, including prompt engineering, orchestration, or RAG patterns."* ([posting](https://simplify.jobs/p/f2b88455-8635-4765-a23d-e52d79f584d4/Senior-Software-Engineer))

**Coverage**: ⚠️ **Gap** — proposed `mod-012-eval-agent-observability` in [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json). **Scope note**: when scaffolded, mod-012 should include within-module exercises on MCP proxy operation, MCP tool-call tracing, and framework-agnostic evaluation/observability for LangGraph/LangChain-based agents (per absorbed requirements `req-mcp-a2a-protocols` and `req-named-agent-frameworks`). No new module is proposed for these signals per continuity bias — they remain under the ≥30% frequency bar for a standalone module at strict L30 ML-Platform titles (roughly 20–27%), and the 2026-07 forward-guidance explicitly anticipated this attachment path.

**External resources** (until the module lands):
- [OpenAI Evals](https://github.com/openai/evals)
- [Anthropic Evals Guide](https://docs.anthropic.com/en/docs/build-with-claude/evals)
- [LangSmith / Langfuse / Helicone](https://github.com/langfuse/langfuse)
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [Model Context Protocol specification](https://modelcontextprotocol.io/)
- [Anthropic MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [LangGraph documentation](https://langchain-ai.github.io/langgraph/)

---

## Group 10: RAG Infrastructure & Vector Platforms — **Monitoring (firming, not yet a gap)**

**Owner: This repo (Level 30)** — emerging area; evidence firming but still under threshold at generalist L30 titles.

Vector databases (Pinecone, Weaviate, pgvector, Qdrant, Milvus, Vespa), retrieval service abstractions, and chunking/embedding pipelines as platform features. The 2026-09 window produced the first two clearly in-window platform-titled deliverable hits (Reddit Indexing & Retrieval Platform, Scale AI MLE Platform); several other postings (Stripe, Robinhood, Roblox) list RAG as one of many platform surfaces they build.

**Evidence (8 postings including 5 new in 2026-09 window)**:
- **Reddit Sr Staff SWE, Indexing & Retrieval Platform** *(new 2026-09)* — *"Lead the technical strategy…of Reddit's next-generation ML Indexing & Retrieval engine, integrating capabilities across lexical and vector indexing, low-latency retrieval, and emerging GenAI applications. Requires expertise in Milvus, Vespa, or Elasticsearch."* ([posting](https://job-boards.greenhouse.io/reddit/jobs/7844238))
- **Scale AI MLE, Platform** *(new 2026-09)* — *"Deep hands-on understanding of retrieval systems, RAG, embeddings, vector indexing, knowledge representation, and semantic search."* ([posting](https://job-boards.greenhouse.io/scaleai/jobs/4711544005))
- **Scale AI Sr Infra SWE, Enterprise AI** *(new 2026-09)* — Agents + vector databases as platform substrate ([posting](https://job-boards.greenhouse.io/scaleai/jobs/4599700005))
- **Stripe Staff SWE, ML Platform** *(new 2026-09)* — RAG named among "most challenging ML Platform problems" ([posting](https://stripe.com/careers/listing/staff-software-engineer-machine-learning-platform/7939868))
- **Robinhood Sr SWE, AI Platform** *(new 2026-09)* — RAG patterns as platform frameworks ([posting](https://simplify.jobs/p/f2b88455-8635-4765-a23d-e52d79f584d4/Senior-Software-Engineer))
- **Reddit GenAI Platform** — *"RAG system architecture."* ([posting](https://job-boards.greenhouse.io/reddit/jobs/7772274))
- **Mixpanel** — AI-native infrastructure including RAG-style retrieval ([posting](https://job-boards.greenhouse.io/mixpanel/jobs/7941944))
- **Roblox** — *"Generative AI information retrieval."* ([posting](https://careers.roblox.com/jobs/7403998))

**Coverage**: 🔭 **Monitoring** — signal is firming but frequency at strict generalist L30 ML-Platform titles remains under the ≥30% bar (roughly 12–17% of sampled postings). Most explicit "build the RAG platform" postings come from search/retrieval-native companies. Re-evaluate next cycle; likely candidate for promotion to a `mod-013` proposal in 2026-12 if the trend continues.

**External resources** (in the interim):
- [Pinecone Architecture](https://docs.pinecone.io/docs/architecture)
- [pgvector](https://github.com/pgvector/pgvector)
- [Milvus documentation](https://milvus.io/docs)
- [Vespa documentation](https://docs.vespa.ai/)
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

**Standing outstanding proposals (unchanged 2026-11)** — three new modules, one new project, and one exercise from the 2026-06 delta, each with at least 3 cited postings. This cycle emits **no new proposals**; the 2026-11 evidence reinforces all standing items (DoorDash p060 is a rare triple-hit JD on all three gaps; Netflix p062 reinforces serving + gateway; DoorDash p061 reinforces absorbed MCP scope in `mod-012`).

| Proposed | Title | Citations (2026-11 refreshed) | Maps to gap |
|----------|-------|-------------------------------|-------------|
| `mod-010-llm-serving-platform` | LLM Serving as a Platform Runtime | 19 | `req-llm-serving-platform` |
| `mod-011-llm-gateway-routing` | LLM Gateway, Routing & Prompt Management | 14 | `req-llm-gateway` |
| `mod-012-eval-agent-observability` | Eval Harnesses, Agent Observability, MCP & Agent-Framework Support | 17 | `req-eval-harness-agent-observability` + absorbed `req-mcp-a2a-protocols` + absorbed `req-named-agent-frameworks` |
| `project-06-genai-platform` | GenAI Platform: Gateway + Eval Harness + Serving | 17+ | All three gaps above |
| `mod-005 exercise: KubeRay` | KubeRay self-service for platform users | 5 | `req-ray-on-kubernetes` |

The deltas are **strictly additive** and will not auto-merge — they go through human review with per-run caps applied by the runner.

---

## Watch List (Re-evaluate Next Cycle)

These signals are present but evidence remains too thin (or too concentrated at niche employers) for new modules under the current contract:

- **Fine-tuning platform / LoRA-adapter-serving** *(firming 2026-11 at single employer)* — DoorDash SWE ML Platform Gen AI (p060) explicitly names SFT/DPO/LoRA alongside serving as headline platform scope; external vendor discourse (Predibase/LoRAX, Fireworks, Together AI, Anyscale) continues to firm. Still single-digit employer coverage at strict L30 ML-Platform titles — remains under the ≥30% bar for a standalone module. Expected to be absorbed by `mod-010` scope when scaffolded.
- **Agent Gateway / per-tenant GenAI cost attribution** *(new 2026-11 — single employer)* — DoorDash p060 names "Agent Gateway" as distinct from "LLM Gateway" and names per-agent/per-tenant cost attribution as a deliverable. Single-employer so far; absorbed into standing `mod-011` scope rather than broken out. Revisit next cycle if a second employer picks it up.
- **RAG infrastructure as platform deliverable** *(firming 2026-09)* — 8 postings across cycles; 5 new hits in the 2026-09 window (Reddit Indexing & Retrieval, Scale AI Platform, Scale AI Enterprise, Stripe, Robinhood). Frequency at strict generalist L30 titles still ~12–17% — under the ≥30% bar. Likely `mod-013` candidate in the 2026-12 cycle if the trend continues.
- **Voice/streaming AI infra** *(firming 2026-09 at voice-native cos)* — Together AI Sr Platform Eng Voice AI, Cartesia SWE Platform, Deepgram MLOps Infra all in-window. Signal is broadening beyond original flags but remains concentrated at voice-specialized companies (Together, Cartesia, Deepgram, ElevenLabs). Frequency at generalist L30 titles <10%. Much of the WebSocket / bidirectional-streaming skill is general backend engineering; not clearly a new *ML platform* module.
- **Edge / on-device ML platform support** — Samsara's Lead MLE ML Infrastructure (2026-09) reinforces the same single-employer concentration seen in the 2026-06 cycle. Still one employer.
- **Research-facing data platform / experiment metadata** — Anthropic (3 postings across cycles) remains the primary source; concentrated at AI labs.
- **MCP / A2A protocols as platform surface** *(firmed 2026-09 — MOVED OUT OF WATCH LIST)* — see [Group 9](#group-9-eval-harnesses--agent-observability--mcp--named-agent-frameworks--expanded-gap). Signal migrated into L30 platform titles at DoorDash, Anthropic, Samsara, PAR, OnePay. Absorbed by `mod-012` proposal per 2026-07 forward-guidance.
- **Named agent-framework fluency (LangGraph, LangChain, CrewAI, AutoGen)** *(firmed 2026-09 — MOVED OUT OF WATCH LIST)* — see [Group 9](#group-9-eval-harnesses--agent-observability--mcp--named-agent-frameworks--expanded-gap). LangGraph dominates; kitchen-sink lists (Kaseya) name all four. Absorbed by `mod-012` proposal.

---

*Last updated: 2026-11-04. Next refresh: see [`.aicg/research/runs/ml-platform/research-run.env`](./.aicg/research/runs/ml-platform/research-run.env).*
