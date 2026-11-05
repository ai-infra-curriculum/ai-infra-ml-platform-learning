# Module 05: Workflow Orchestration

**Duration**: ~8 hours guided + 10-12 hours exercises
**Prerequisites**: Modules 01-04; Python; comfort reading a DAG; at least one
piece of work that was once a cron job and should not have been

## Overview

Orchestration is the connective tissue between every component built so far:
the data platform from Module 01, the API surface from Module 02, the
tenanted, quota'd cluster from Module 03, and the feature-store
materialization pipeline from Module 04 all terminate at the same place — a
scheduler that decides when the DAG runs, which task runs where, and what
happens when one of them fails at 3am on a Sunday. Pick the orchestrator
carefully: it is a *platform-level* choice that constrains every downstream
pipeline for the next 5 years, and switching orchestrators midstream is one
of the most expensive migrations an ML platform can take on.

Module 04 ended with a materialization DAG that moved features from offline
to online storage. Module 05 is where that DAG, the training DAG that
consumes those features, the evaluation DAG that gates promotion, and the
retraining DAG that reacts to drift all live under a single scheduler with a
coherent retry, SLA, and backfill story. By the end of this module you will
have evaluated five orchestrators against production criteria, implemented
the four canonical ML orchestration patterns (DAG-per-model, parametric DAG,
event-driven, continuous training), built a backfill driver that will not
silently DDoS downstream services, wired SLA + alerting on a multi-stage
training DAG, and produced a capacity plan for a multi-tenant Airflow
install.

The tone of this module is deliberately operational. We assume you can read
a Python DAG definition and have deployed *something* to a scheduler, even
if it was a cron job. We will spend our time on the parts that ML platform
teams consistently get wrong in year one: an orchestrator picked from a
trending blog post, a parametric DAG that generates 5,000 copies of itself
with no organizing principle, a backfill that overlaps itself and corrupts
yesterday's data, an "alert on every failure" policy that trains the
on-call team to ignore the pager, and a pool capacity plan that looks
reasonable on paper but has the scheduler thrashing against a GPU quota
one-third its size.

## Learning objectives

By the end of this module, you will be able to:

1. **Choose** an orchestrator (Airflow, Prefect, Dagster, Argo, Kubeflow,
   Flyte, or Metaflow) for a hypothetical team by scoring it against
   Python-nativeness, type safety, Kubernetes integration, production
   maturity, lock-in risk, and hiring-pool depth — and defend the choice in
   a peer review.
2. **Implement** the four canonical ML orchestration patterns —
   DAG-per-model, parametric DAG, event-driven, and continuous training —
   and name the workload shape that pushes you toward each.
3. **Write** Airflow DAGs that handle retries with exponential backoff,
   branching, dynamic task mapping, and per-DAG SLAs; and explain the
   difference between a task failure (normal, retryable) and an SLA miss
   (duration-based, alertable).
4. **Operate** a backfill safely: with an audit log of every operation, a
   concurrency cap so you don't replay 365 days in parallel, a dry-run mode
   that prints the plan, idempotent tasks so re-runs produce the same
   output, and the ability to re-run a single failed day without starting
   over.
5. **Plan** pool capacity for a multi-tenant Airflow install: distinguish
   pool slots (per-pool task cap) from `parallelism` (cluster-wide task
   cap), size each against the underlying GPU / CPU / API quota it draws
   on, and avoid the thundering-herd scheduling pattern that flattens every
   downstream service at 02:00.
6. **Recognize** the failure modes that cost platform teams their
   orchestrator credibility — DAGs that mutate global state and lose
   idempotency, model weights passed through XCom, a scheduler whose pool
   sits larger than the GPU quota so tasks stay Pending forever,
   per-failure pager alerts that drown the signal — and name the guardrail
   that prevents each.

## What's in this Module

```
lessons/mod-005-workflow-orchestration/
├── README.md                                       (you are here)
├── lecture-notes/
│   ├── 01-orchestrator-landscape.md
│   ├── 02-patterns.md
│   └── 03-operating-orchestrators.md
├── exercises/
│   ├── exercise-01-airflow-or-prefect-decision.md
│   ├── exercise-02-implement-four-patterns.md
│   ├── exercise-03-safe-backfill.md
│   ├── exercise-04-sla-and-alerting.md
│   └── exercise-05-pool-capacity-tuning.md
├── quizzes/
│   └── module-005-quiz.md
└── resources.md
```

### Lecture Notes (read in order)

| Chapter | Title | Approx. reading time | Maps to CURRICULUM.md topic |
| --- | --- | --- | --- |
| 01 | Orchestrator Landscape | 60 min | "Workflow Orchestration Fundamentals" (2h) |
| 02 | ML Orchestration Patterns | 75 min | "DAG Definition and Management" (2.5h) + "Scheduling and Triggers" (2h) |
| 03 | Operating Orchestrators | 75 min | "Task Execution" (2.5h) + "Error Handling and Monitoring" (3h) |

Chapter 01 covers the orchestrator landscape — Airflow, Prefect, Dagster,
Argo Workflows, Kubeflow Pipelines, Flyte, Metaflow — on the dimensions
that actually matter for a platform choice: Python-nativeness, type
safety, Kubernetes integration, production maturity, lock-in risk, and
hiring-pool depth. It closes with a default recommendation (Airflow,
unless) and names the anti-pattern of choosing by "the most recent blog
post I read."

Chapter 02 is the pattern chapter. It covers the four shapes you will
actually ship: DAG-per-model (few models, diverse shapes), parametric DAG
(many models, one shape, driven from a config file), event-driven (S3
`PutObject` → SQS → Lambda → DAG trigger, for irregular arrivals), and
continuous training (cron + branching, retrain iff drift > threshold).
Most platforms run several simultaneously; the mistake is forcing every
model into one pattern.

Chapter 03 is where the module lives operationally: Airflow pool capacity
vs. `parallelism`, the thundering-herd scheduling pattern, the four most
common failure modes and their recovery paths, the five non-negotiable
pieces of backfill discipline (audit log, concurrency cap, dry-run,
idempotency, single-day replay), and the alerting policy that distinguishes
a normal retry from an incident.

### Exercises (do after the matching chapter)

| Exercise | After chapter | Estimated time | Format |
| --- | --- | --- | --- |
| 01 — Orchestrator selection memo | 01 | 120 min | `DECISION.md` (~1500 words) + peer review |
| 02 — Implement the four patterns | 02 | 180 min | Minimal working DAGs for all four patterns + `PATTERN_DECISION.md` |
| 03 — Safe backfill tooling | 02 | 150 min | Backfill driver (sequential / parallel-by-date / parallel-by-shard) + dry-run + retry-failed + audit log |
| 04 — SLA + alerting | 03 | 120 min | Multi-stage training DAG with SLAs + Slack alerts + injected-fault demo |
| 05 — Pool capacity tuning | 03 | 150 min | `CAPACITY_PLAN.md` for a 20-team multi-tenant install |

Exercises 01 and 02 build the foundation; 03, 04, and 05 are the
operational core of the module and the exercises most learners
underestimate. Exercise 03 (backfill) and Exercise 05 (capacity) are the
two that will most directly resemble a real on-call ticket in your first
year on an ML platform team — leave real time for them.

Several exercises reference companion templates in
`ai-infra-engineer-solutions`. Specifically, `mod-106 ex-11`
(ml-orchestration-patterns) has working code for all four patterns
exercised in 02, `mod-105 ex-09` (backfill-strategies) is the engineer-
track deep dive on the backfill discipline exercised in 03, and
`mod-105 ex-02` is a recommender training DAG with branching + dynamic
mapping + SLAs that you can study as a worked example for 04. Those
templates are starting points, not answer keys — you are expected to
adapt them (changing orchestrator choice, DAG shape, SLA thresholds, and
alert routing to match your scenario) rather than copy them.

### Quiz

`quizzes/module-005-quiz.md` contains 15 questions covering orchestrator
selection, the four patterns, XCom and operator semantics, pool vs.
parallelism, backfill discipline, SLA mechanics, and alerting policy. The
answers are embedded as checked boxes next to the correct option; try to
answer each question before you look at the next one.

A passing score is **75% (12 of 15)**. There is no separate answer key
file — the quiz is self-grading via the checkbox annotations, and
rationale for every answer is at the bottom of the quiz file.

## How to Work Through This Module

There is no single right path, but here is one that has worked for past
learners. The whole module is ~8 hours of guided study plus ~10-12 hours
of exercises; most learners spread this over one to two weeks.

**Day 1 (1 hour)**: Read chapter 01. On paper, sketch which orchestrator
your current (or most recent) team runs on and score it against the five
criteria in the decision matrix. If the answer is "we run three of them,"
that is itself a finding — note it for exercise 01.

**Day 2 (2 hours)**: Do exercise 01 (orchestrator selection memo). Treat
the 1500-word budget seriously — the point of the exercise is to practice
defending a platform-level choice in writing, not to produce a glossy
comparison table. If you cannot cite a specific workload shape that drives
your pick, the memo is not done.

**Day 3 (1.5 hours)**: Read chapter 02. Write down, for each of the four
patterns, one model or pipeline you have seen that fits it. Flag any
pattern where you have no example — you will build one in the next step.

**Day 4 (3 hours)**: Do exercise 02 (implement the four patterns). The
goal is minimal, working, side-by-side implementations — not production
hardening. You should finish this exercise able to point at four DAG
files and name, in one sentence each, when you would reach for each one.

**Day 5 (2.5 hours)**: Do exercise 03 (safe backfill tooling). Build all
three replay strategies (sequential, parallel-by-date, parallel-by-shard)
even if one of them feels redundant — the comparison is the lesson. The
audit log is not optional: without it, a two-engineer team cannot tell
whether yesterday's backfill overlapped last week's.

**Day 6 (1.5 hours)**: Read chapter 03. If you have never looked at the
Airflow pool admin UI before, open it (or screenshots in the official
docs) and locate the four fields that matter: pool name, slots, used
slots, queued slots. Capacity planning is concrete; it needs concrete
referents.

**Day 7 (2 hours)**: Do exercise 04 (SLA + alerting). The injected-fault
demonstrations are the deliverable — the configuration without the proof
is theatre. Before you call the exercise done, trip each of the four
conditions (missed SLA, 3× task failure, long-pending task, lost
scheduler heartbeat) on purpose and screenshot the alert.

**Day 8 (2.5 hours)**: Do exercise 05 (pool capacity tuning). This is the
exercise closest to a real capacity-planning ticket. Resist the temptation
to size pools to the headline GPU count — size them to what the *rest* of
the dependencies (feature store QPS, metadata DB connections, cluster API
server rate limits) will tolerate at the same time. Thundering herds at
02:00 are what the plan is really trying to prevent.

**Day 9 (1 hour)**: Take the quiz. Re-read any chapter whose questions
you had to guess at, especially chapter 02 if the pattern-fit questions
felt arbitrary.

If you are moving slowly, that is fine. Orchestration is the module where
platform teams most often ship a scheduler that works at 5 DAGs and
collapses at 500 — depth here pays itself back every year the platform
keeps growing.

## How to Self-Assess

After finishing the module, you should be able to do the following without
referring back to the notes:

- [ ] Name the five orchestrator criteria from chapter 01 and score any
      two orchestrators against them from memory.
- [ ] Explain, in one sentence each, when to reach for DAG-per-model,
      parametric DAG, event-driven, and continuous-training patterns.
- [ ] Explain the Airflow pool vs. `parallelism` distinction and give a
      concrete example where mis-sizing one against the other causes
      tasks to sit Pending.
- [ ] List the five pieces of backfill discipline (audit log, concurrency
      cap, dry-run, idempotency, single-day replay) and describe a bug
      each one prevents.
- [ ] Explain what `sla_miss_callback` fires on, and why it is *not* a
      task-failure alert.
- [ ] Describe an alerting policy that avoids pager fatigue — specifically,
      why "alert on every task failure" is the wrong default and what the
      right threshold looks like.
- [ ] Name the anti-patterns from chapter 01 (DAGs that mutate global
      state, 5K+ DAGs with no organizing principle, model weights in
      XCom, training as a DAG task without `KubernetesPodOperator` /
      Kueue / Volcano) and the fix for each.
- [ ] Score 75%+ on the quiz without notes.

If you can do all of the above, you are ready for Module 06. If 2-3 of
these feel shaky, that is normal — flag them and revisit when you
encounter the relevant concept again in later modules, especially when
Module 06 (model management) begins calling your training DAGs from a
registry promotion workflow.

## A Note on Tools

This module names several orchestrators by product:
[Apache Airflow](https://airflow.apache.org/),
[Prefect](https://www.prefect.io/),
[Dagster](https://dagster.io/),
[Argo Workflows](https://argoproj.github.io/argo-workflows/),
[Kubeflow Pipelines](https://www.kubeflow.org/docs/components/pipelines/),
[Flyte](https://flyte.org/), and
[Metaflow](https://metaflow.org/). **You are not expected to be fluent in
all of these.** The exercises pin Airflow as the default (because its
hiring pool, community, and Kubernetes executor make it the safest first
bet for an ML platform), so you can finish the module without evaluating
every tool first.

The resources list in `resources.md` is where to go when you need to
evaluate alternatives for a real deployment — the principles here are
portable; the specific tooling is not. In particular, chapter 01 names
the conditions under which Airflow is the wrong default: a Lyft/Spotify-
scale reproducibility story (Flyte), a strongly type-safe,
asset-centric codebase with a small team (Dagster), or a
Vertex-AI-bound ML-first org (Kubeflow Pipelines).

## A Note on Platform / Engineer Split

This module lives in the ML Platform curriculum, which is the *design and
operations* track. Several exercises reference companion implementations
in `ai-infra-engineer-solutions` — specifically `mod-106 ex-11` for the
four-pattern reference implementation and `mod-105 ex-09` for the
backfill deep dive, with `mod-105 ex-02` as a worked SLA + branching
example. Those are the hands-on engineer-track templates; this module
leans on them rather than reproducing them. If you want a deeper
implementation pass on any exercise, start from the companion and then
come back to the decision memo (01) and the capacity plan (05), which are
the deliverables the engineer track does not cover.

## Navigation

- Up: [`../../README.md`](../../README.md) — repo overview
- Curriculum spec: [`../../CURRICULUM.md`](../../CURRICULUM.md) — see
  "Module 05: Workflow Orchestration"
- Previous: [`../mod-004-feature-store/`](../mod-004-feature-store/) — Feature Store Architecture
- Next: [`../mod-006-model-management/`](../mod-006-model-management/) — Model Management & Registry

Orchestration is the module where most ML platforms either grow a
coherent, debuggable pipeline story or accumulate 5,000 one-off DAGs
with no organizing principle. Pick the orchestrator once, pick the
patterns per-model, write down the backfill discipline, size the pools
against the quotas they actually draw on, and the rest of the platform
has a scheduler it can trust.
