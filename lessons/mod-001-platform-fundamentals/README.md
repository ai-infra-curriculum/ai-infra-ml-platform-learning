# Module 01: Platform Fundamentals

> "A platform is a product whose users are other engineers." — paraphrasing the platform engineering community

## Overview

This module is the on-ramp for the entire ML Platform Engineering curriculum. Before we build feature stores, model registries, training schedulers, or serving meshes, we need a shared mental model for what an "ML platform" actually *is*, why it exists as a distinct discipline from data infrastructure or generic SRE, and what makes platform work succeed or fail.

We will not write a feature store from scratch in this module. We will not deploy Kubeflow. Module 01 is deliberately conceptual and architectural — we are calibrating the way you think about platforms before you touch a single Helm chart in Module 02.

If you come out of this module able to walk into a 30-minute conversation with a Director of ML Infrastructure at a mid-stage company, sketch their platform on a whiteboard, name three plausible failure modes, and ask three sharp questions about multi-tenancy, you have hit the learning objectives.

## Duration

- **Total**: ~8 hours of guided study + ~6-10 hours of exercises
- **Pace**: 1 week if done full-time, 2 weeks at evenings/weekends pace
- **Prerequisites**: working familiarity with Linux, Kubernetes basics (pods, services, namespaces, RBAC at the level that you can read a manifest without panic), at least one production deployment under your belt in any stack, and *some* exposure to ML workflows (training a model, even a toy one, end-to-end). If you have none of the above, complete `ai-infra-junior-engineer-learning` first.

## Learning Objectives

By the end of this module, you will be able to:

1. **Distinguish a platform from a pipeline, an infrastructure layer, and an application.** Given a diagram of a stack, identify which boxes are platform surfaces and which are not, and justify that classification.
2. **Articulate the platform value proposition** to three audiences: an ML engineer, an SRE, and a CFO. Each audience cares about different things; you must speak each language.
3. **Recognize at least four platform architecture patterns** (monolith, microservices, event-driven, plugin) and identify which fits a given organizational context.
4. **Design a multi-tenancy strategy** for a Kubernetes-backed ML platform, including namespace layout, RBAC, resource quotas, network policy, and cost allocation. You won't implement it in this module — Module 03 does that — but you will be able to defend the design choices on paper.
5. **Apply API-first thinking** when adding a new capability to a platform: versioning, deprecation, contract testing, documentation, and the "stable URL" principle.
6. **Read at least two published ML platform case studies critically** (Uber Michelangelo, Netflix Metaflow) and extract design lessons rather than just admire the architecture diagrams.
7. **Identify the right team structure** (platform-as-product, embedded SRE, hybrid) for a given organization's size, ML maturity, and regulatory environment.

## What's in this Module

```
lessons/mod-001-platform-fundamentals/
├── README.md                                       (you are here)
├── lecture-notes/
│   ├── 01-introduction-to-ml-platform-engineering.md
│   ├── 02-platform-thinking.md
│   ├── 03-multi-tenancy-patterns.md
│   ├── 04-api-first-development.md
│   └── 05-platform-architecture-patterns.md
├── exercises/
│   ├── exercise-01-design-api-for-resource-provisioning.md
│   ├── exercise-02-namespace-isolation-in-kubernetes.md
│   ├── exercise-03-resource-quota-management.md
│   ├── exercise-04-build-a-simple-plugin-system.md
│   └── exercise-05-case-study-analysis-michelangelo-metaflow.md
├── quizzes/
│   ├── module-001-quiz.md
│   └── module-001-quiz-answers.md
└── resources.md
```

### Lecture Notes (read in order)

| Chapter | Title | Approx. reading time | Maps to CURRICULUM.md topic |
| --- | --- | --- | --- |
| 01 | Introduction to ML Platform Engineering | 90 min | "Introduction to ML Platform Engineering" (1.5h) |
| 02 | Platform Thinking | 120 min | "Platform Thinking" (2h) |
| 03 | Multi-Tenancy Patterns | 120 min | "Multi-Tenancy Patterns" (2h) |
| 04 | API-First Development | 90 min | "API-First Development" (1.5h) |
| 05 | Platform Architecture Patterns | 60 min | "Platform Architecture Patterns" (1h) |

### Exercises (do after the matching chapter)

| Exercise | After chapter | Estimated time | Format |
| --- | --- | --- | --- |
| 01 — Design API for a resource provisioning system | 04 | 90 min | Design doc + OpenAPI sketch |
| 02 — Implement namespace isolation in Kubernetes | 03 | 90 min | YAML + reflection |
| 03 — Create resource quota management for teams | 03 | 90 min | YAML + cost model |
| 04 — Build a simple plugin system | 05 | 90 min | Python design + walk-through |
| 05 — Case study analysis (Michelangelo + Metaflow) | 01 | 120 min | Written analysis |

You can do exercises in any order *after* the matching lecture chapter, but the sequence above is the one with the smoothest difficulty ramp.

### Quiz

`quizzes/module-001-quiz.md` contains 18 questions: a mix of multiple-choice (concept checks), short-answer (definitions and trade-off articulation), and applied-scenario questions (you're shown a fictional company and asked to make a platform decision and justify it). The answer key is in `module-001-quiz-answers.md` — try not to peek until you've taken a full pass.

A passing score is **75% on the MCQ portion and a defensible answer on at least 3 of 4 scenario questions**. There is no autograder; you are expected to self-assess with the answer key, which includes rationales explaining why each correct answer is correct.

## How to Work Through This Module

There is no single right path, but here is one that has worked for past learners.

**Day 1 (2 hours)**: Read chapter 01. Skim the two case studies linked in `resources.md`. Start exercise 05 (the case-study analysis) — you don't need to finish, just take notes.

**Day 2 (2 hours)**: Read chapter 02. Reflect on a platform (or pseudo-platform) you've worked with — what was its abstraction, who used it, did it succeed? Write 3 paragraphs.

**Day 3 (2 hours)**: Read chapter 03. Do exercise 02 (namespace isolation). This is the most hands-on chapter.

**Day 4 (2 hours)**: Read chapter 04. Do exercise 01 (API design). This is the most design-heavy exercise.

**Day 5 (2 hours)**: Read chapter 05. Do exercise 04 (plugin system).

**Day 6 (2 hours)**: Finish exercise 03 (resource quotas) and exercise 05 (case study). Take the quiz.

**Day 7 (optional)**: Re-read any chapter that didn't stick. Read one of the "going deeper" links in `resources.md`.

If you're moving slowly, that's fine. Platform work rewards depth of understanding over breadth, and we will spiral back to most of these concepts in later modules with progressively more weight on each.

## How to Self-Assess

After finishing the module, you should be able to do the following without referring back to the notes:

- [ ] Define an ML platform in one sentence, without using the word "platform" in the definition.
- [ ] List three things that make ML platforms harder than generic application platforms.
- [ ] Sketch the architecture of Michelangelo *or* Metaflow on a whiteboard from memory (you may forget specifics; the topology should be right).
- [ ] Explain "self-service vs full-service" with an example of each from outside the ML world.
- [ ] Describe at least two failure modes for Kubernetes-namespace-per-tenant isolation, and at least one mitigation for each.
- [ ] Write a versioned API endpoint URL and explain when you would bump the major version vs add a query parameter vs add a header.
- [ ] Score 75%+ on the quiz MCQ portion without notes.

If you can do all of the above, you are ready for Module 02. If 2-3 of these feel shaky, that's normal — note which ones, flag them in your notes, and revisit during Module 02.

## A Note on Tools

This module names many tools by name — Feast, MLflow, Kubeflow, Airflow, FastAPI, Argo, KServe, BentoML, Tecton, Hopsworks, Databricks, Vertex AI, SageMaker. **You are not expected to know how to operate any of these yet.** They appear in this module as proper nouns so you can begin to recognize them and place them on a mental map. Later modules will get hands-on with specific tools.

When the lecture says "Feast is a feature store" we are describing the category, not endorsing a specific deployment. The principles in this module are tool-agnostic; the tools are illustrative.

## A Note on Case Studies

Module 01 leans heavily on two published case studies (Uber Michelangelo, Netflix Metaflow). We deliberately reference *published* writeups (linked in `resources.md`) rather than secondhand summaries. When the lecture says "Michelangelo did X", you can find that claim in the source. If you find a contradiction between the lecture and the source, the source wins; please file an issue.

We do not invent hypothetical company anecdotes. Where we need an illustrative scenario, we label it "Hypothetical example:" or "Fictional scenario:" — see Guardrail #20 in `_meta/GUARDRAILS.md` if you are curious about the rationale.

## Navigation

- Up: [`../../README.md`](../README.md) — repo overview
- Curriculum spec: [`../../CURRICULUM.md`](../../CURRICULUM.md) — full 10-module plan; Module 01 spec starts at the "Module 01: Platform Fundamentals" heading
- Next: Module 02 (forthcoming) — API Design for ML Platforms

Welcome to platform engineering. It's a peculiar discipline. You'll like it.
