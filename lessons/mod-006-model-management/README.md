# Module 06: Model Management & Registry

**Duration**: ~8 hours guided + 8-10 hours exercises
**Prerequisites**: Modules 01-05; Python; comfort reading a Kubernetes
manifest; at least one model you have personally re-trained and wished you
could roll back

## Overview

A model registry answers the single question every platform team ends up
being asked at 3am: *"what model is in production right now?"* Without a
registry, the answer is "whatever binary the last pod pulled into its
filesystem," which is both a serving incident and a compliance incident
rolled into one. With a registry, the answer is a named version with a
stage, an owner, a training run, an approval trail, and a rollback path.
Every other capability in this module — promotion gates, canary
deployments, governance artifacts, quarterly audits — hangs off that one
contract: the registry is the source of truth for what is deployed where,
and serving infrastructure reads from it rather than guessing.

Module 05 ended with an orchestrator running training, evaluation, and
promotion DAGs on a schedule. Module 06 is where those DAGs terminate: the
training DAG registers a new version, the evaluation DAG gates its
promotion, and the serving fleet watches the registry for the version it
should be loading. By the end of this module you will have stood up an
MLflow-backed registry on Postgres + MinIO, implemented a quality-gated
`promote.py` and an atomic `rollback.py`, wired an Argo Rollouts canary
against a success-rate analysis template, enforced a governance gate that
refuses promotion without a signed model card and bias review, and run a
quarterly-audit script against a mixed-compliance registry.

The tone of this module is deliberately operational. We assume you can
read a Kubernetes manifest and have at least watched a canary rollout in
progress, even if someone else wrote the AnalysisTemplate. We will spend
our time on the parts that ML platform teams consistently get wrong in
year one: a "registry" that is really a shared S3 bucket with a naming
convention, a promote button that moves a stage pointer but never records
*who* clicked it or *why*, a rollback that demotes the bad version but
never promotes the good one (leaving nothing in Production), a canary that
auto-reverts in the serving layer while the registry's Production pointer
still names the broken version, and a model-card process that produces a
template no one has read since the model was trained.

## Learning objectives

By the end of this module, you will be able to:

1. **Articulate** what a model registry is the source of truth for
   (environment → version mapping, approval trail, lifecycle state) and
   what it is explicitly *not* the source of truth for (training data
   lineage, feature freshness, per-request cost), and name the system in
   this curriculum that owns each of those other facts.
2. **Implement** an MLflow-backed registry on Postgres + MinIO, register a
   model, walk a version through None → Staging → Production → Archived,
   and use aliases (`champion`, `challenger`, `canary`) to rotate
   production pointers with zero downtime.
3. **Write** a quality-gated promotion workflow: training registers a
   version, the evaluation DAG transitions it to Staging only if it
   clears an accuracy delta threshold, smoke tests run in Staging, and a
   human approval is required for Production — with every transition
   audit-logged against actor, reason, and gate metric values.
4. **Operate** rollback as a transactional, both-sides action — demote the
   current Production version to Archived *and* promote the last good
   version in a single API call — so there is never a window where
   Production points at nothing or at two things, and write a tabletop
   walkthrough of a real "v6 is bad in prod" incident.
5. **Choose** the right deployment strategy for a change (rolling,
   blue-green, canary, shadow), explain the serving-layer semantics of
   each in one sentence, and name the class of change that pushes you
   toward each one.
6. **Wire** a canary rollout with Argo Rollouts (or equivalent) against an
   AnalysisTemplate that gates on real-time success rate, demonstrate
   auto-revert on a deliberately-broken model, and explain why the
   registry's Production pointer is untouched by a failed canary.
7. **Enforce** governance at the promote API: refuse the Production
   transition unless a current model card exists, a bias review record
   exists for protected-attribute models, and the accuracy delta versus
   the prior Production version is documented — and demonstrate the
   clear error message each missing artifact produces.
8. **Recognize** the failure modes that cost ML platform teams their
   deploy credibility — `joblib.load("/data/model.pkl")` in the serving
   path, a rollback that skipped promoting the good version, a canary
   whose AnalysisTemplate measures a metric the serving fleet can't
   actually emit, and a model card that was signed six months ago and no
   longer describes the model — and name the guardrail that prevents
   each.

## What's in this Module

```
lessons/mod-006-model-management/
├── README.md                                       (you are here)
├── lecture-notes/
│   ├── 01-registry-as-source-of-truth.md
│   ├── 02-deployment-strategies.md
│   └── 03-governance-integration.md
├── exercises/
│   ├── exercise-01-mlflow-registry-deep-dive.md
│   ├── exercise-02-rollback-procedure.md
│   ├── exercise-03-canary-via-argo.md
│   ├── exercise-04-governance-gate.md
│   └── exercise-05-quarterly-audit.md
├── quizzes/
│   └── module-006-quiz.md
└── resources.md
```

### Lecture Notes (read in order)

| Chapter | Title | Approx. reading time | Maps to CURRICULUM.md topic |
| --- | --- | --- | --- |
| 01 | Registry as Source of Truth | 60 min | "Model Registry Fundamentals" (2h) + "Model Lifecycle Management" (2.5h) |
| 02 | Deployment Strategies (Registry-Driven) | 60 min | "Model Deployment" (2h) |
| 03 | Governance Integration | 60 min | "Model Lineage Tracking" (2h) + "Model Monitoring" (1.5h) |

Chapter 01 covers the registry's object model — names, versions, stages,
aliases, tags — and the promotion workflow that walks a version from a
training-run artifact through `Staging` to `Production` to `Archived`.
It names the three most common violations of registry-as-source-of-truth
(`joblib.load` from disk, model baked into a container image tag, direct
S3 path lookups in the serving code) and the one fix that resolves all
three: serving pulls by `(name, stage)` at startup, with the registry URI
in config.

Chapter 02 is the deployment-strategy chapter. It covers the four shapes
you will actually ship — rolling, blue-green, canary, shadow — on the
dimensions that decide which one you reach for: schema compatibility,
rollback cost, metric latency, and whether you need user-visible
side-effect-free traffic (shadow). It closes with the Argo Rollouts
AnalysisTemplate pattern for auto-gating a canary on real-time
success-rate and names the invariant that most teams get wrong: a failed
canary reverts the *serving* fleet, not the *registry* — the registry's
Production pointer never moved, so the "rollback" is the serving layer
noticing and refusing to promote the new weights.

Chapter 03 is the governance chapter. It covers the four artifacts that
most regulated and most self-regulated platforms require at Production
promotion (signed model card, bias review record, decision log entry,
audit log entry of the promotion event), the programmatic enforcement
pattern at the promote API, and the quarterly audit cycle that keeps
those artifacts from going stale. It closes with the audit-log shape
(timestamp, actor, from-version, to-version, reason, gate-metric values)
and the append-only storage pattern (hash-chained log) that makes an
audit answerable months after the fact.

### Exercises (do after the matching chapter)

| Exercise | After chapter | Estimated time | Format |
| --- | --- | --- | --- |
| 01 — MLflow registry deep dive | 01 | 150 min | Postgres+MinIO MLflow + `promote.py` with quality gate |
| 02 — Rollback procedure | 01 | 90 min | `rollback.py` + tabletop walkthrough of "v6 is bad in prod" |
| 03 — Canary via Argo Rollouts | 02 | 150 min | Argo Rollouts manifest + AnalysisTemplate + deliberately-broken-model demo |
| 04 — Governance gate at promotion | 03 | 120 min | Promote API that refuses without card + bias review + accuracy delta |
| 05 — Quarterly audit tooling | 03 | 90 min | `quarterly_audit.py` + Markdown report against mixed-compliance registry |

Exercises 01 and 02 build the registry and rollback foundation; 03 is the
deployment-strategy exercise and the one that most directly resembles a
real serving-team ticket; 04 and 05 are the governance pair that makes
the registry something an auditor can reason about rather than a
production-only artifact. Exercise 02 (rollback) and Exercise 04
(governance gate) are the two that most learners underestimate — the
rollback looks like "just demote the current version" until you try it
and realize nothing is in Production, and the governance gate looks like
documentation busywork until you watch it block a bad Friday promote.

Several exercises reference companion templates in
`ai-infra-engineer-solutions`. Specifically, `mod-106 ex-02` and
`mod-106 ex-03` cover the MLflow registry walk-through and the
quality-gated promotion script that back Exercise 01, `mod-106 ex-08`
has the Argo Rollouts canary manifests used as the starting point for
Exercise 03, and `mod-106 ex-10` has the governance-artifact templates
(model card, bias review, decision log) used in Exercise 04. Those
templates are starting points, not answer keys — you are expected to
adapt them (changing model name, stage names, gate thresholds, and
governance artifact requirements to match your scenario) rather than
copy them.

### Quiz

`quizzes/module-006-quiz.md` contains 15 questions covering
registry-as-source-of-truth, the promotion workflow, the four deployment
strategies and their serving-layer semantics, canary auto-gating, the
rollback transaction, governance artifacts, and the quarterly audit.
The answers are embedded as checked boxes next to the correct option;
try to answer each question before you look at the next one.

A passing score is **75% (12 of 15)**. There is no separate answer key
file — the quiz is self-grading via the checkbox annotations, and
rationale for every answer is at the bottom of the quiz file.

## How to Work Through This Module

There is no single right path, but here is one that has worked for past
learners. The whole module is ~8 hours of guided study plus ~8-10 hours
of exercises; most learners spread this over one to two weeks.

**Day 1 (1 hour)**: Read chapter 01. On paper, write down what your
current (or most recent) team uses as the source of truth for "what
model is in production." If the answer is "the container image tag" or
"a wiki page" or "we ask the senior engineer who wrote it," that is
itself a finding — note it for the self-assessment at the end of the
module.

**Day 2 (2.5 hours)**: Do exercise 01 (MLflow registry deep dive). Stand
the registry up on Postgres + MinIO (not SQLite + local filesystem —
SQLite is fine for a tutorial but it hides the contention and backup
questions that a real registry has to answer). You should finish this
exercise able to point at a registered model in the MLflow UI, walk a
version through Staging → Production, and run `promote.py` against a
synthetic accuracy gate.

**Day 3 (1.5 hours)**: Do exercise 02 (rollback procedure). The tabletop
is the deliverable — anyone can write the script; what matters is being
able to walk a second engineer through the sequence at 3am. If your
tabletop does not name who clicks promote, what the rollback target is
if the last good version has already been Archived, and where the
incident note lands, it is not done.

**Day 4 (1 hour)**: Read chapter 02. If you have never watched an Argo
Rollouts canary progress through its weighted steps in the UI, skim the
quickstart in [`resources.md`](./resources.md) so the manifest you write
in the next exercise is not the first canary you have ever seen.

**Day 5 (2.5 hours)**: Do exercise 03 (canary via Argo Rollouts). The
deliberately-broken-model demonstration is the deliverable — the
manifest without the proof is theatre. Before you call the exercise
done, trigger the auto-revert on purpose, screenshot the AnalysisRun
output, and write down in one sentence why the registry's Production
pointer did not move during the failed canary.

**Day 6 (1 hour)**: Read chapter 03. Pull up any model card you (or
your team) have previously signed, or the Google Model Card for a
public model such as [Gemma](https://ai.google.dev/gemma/docs/model_card)
or [Face Detection](https://modelcards.withgoogle.com/face-detection).
Note which fields are specific and auditable versus which fields are
marketing copy — the distinction matters for the governance gate.

**Day 7 (2 hours)**: Do exercise 04 (governance gate at promotion). The
clear error messages are the deliverable, not the gate itself —
"promotion blocked: missing model card" is useless; "promotion blocked:
model card for version 7 either does not exist at
`cards/iris-rf/v7/card.md` or is older than 90 days (signed
2026-07-01)" is actionable. Write the errors for an engineer trying to
ship on a Friday, not for a future reviewer.

**Day 8 (1.5 hours)**: Do exercise 05 (quarterly audit tooling). Run it
against a registry you have deliberately populated with a mix of
compliant and out-of-compliance models — the whole point is to catch
the ones that have drifted out of compliance since the last audit, so
if every model passes, the test data is wrong.

**Day 9 (1 hour)**: Take the quiz. Re-read any chapter whose questions
you had to guess at, especially chapter 02 if the deployment-strategy
questions felt arbitrary — the serving-layer semantics are the part
most learners skim and then can't defend in a design review.

If you are moving slowly, that is fine. Model management is the module
where platform teams most often ship a "registry" that is really a
shared bucket and discover a year later that nobody can answer "what
model was in production on March 5" — depth here pays itself back every
quarter the platform keeps running.

## How to Self-Assess

After finishing the module, you should be able to do the following without
referring back to the notes:

- [ ] Name what the registry is and is not the source of truth for, and
      name the system that owns each fact it does not own.
- [ ] Walk a model version through None → Staging → Production →
      Archived from memory, naming the gate at each transition.
- [ ] Explain, in one sentence each, when to reach for rolling,
      blue-green, canary, and shadow deployments.
- [ ] Explain why a failed Argo Rollouts canary does *not* move the
      registry's Production pointer, and what the serving fleet does
      instead.
- [ ] Describe the atomic-both-sides rollback (demote current + promote
      prev in a single transactional action) and name the failure mode
      that non-transactional rollback produces.
- [ ] List the four governance artifacts required at Production
      promotion (model card, bias review, decision log, audit entry)
      and describe the error message each missing artifact should
      produce at the promote API.
- [ ] Explain what the quarterly audit checks and why "every Production
      model has a card" is not the same question as "every Production
      model has a *current* card."
- [ ] Name the three common registry-as-source-of-truth violations
      (`joblib.load` from disk, image-baked model, direct S3 path
      lookup) and the single fix that resolves all three.
- [ ] Score 75%+ on the quiz without notes.

If you can do all of the above, you are ready for Module 07. If 2-3 of
these feel shaky, that is normal — flag them and revisit when you
encounter the relevant concept again in later modules, especially when
Module 08 (observability) wires drift and performance alerts back into
the registry's retraining trigger and Module 09 (security & governance)
builds on the audit log and governance-gate patterns introduced here.

## A Note on Tools

This module names several registries and rollout engines by product:
[MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html),
[SageMaker Model Registry](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html),
[Vertex AI Model Registry](https://cloud.google.com/vertex-ai/docs/model-registry),
[Weights & Biases Model Registry](https://docs.wandb.ai/guides/registry/),
[Argo Rollouts](https://argo-rollouts.readthedocs.io/), and
[Flagger](https://flagger.app/). **You are not expected to be fluent in
all of these.** The exercises pin MLflow as the registry (because its
open-source footprint, Python SDK, and Postgres + object-store backend
are the closest match to what a self-hosted ML platform will stand up)
and Argo Rollouts as the canary engine (because its AnalysisTemplate
model is the most transferable to other rollout engines you may
encounter), so you can finish the module without evaluating every tool
first.

The resources list in [`resources.md`](./resources.md) is where to go
when you need to evaluate alternatives for a real deployment — the
principles here are portable; the specific tooling is not. In
particular, chapter 01 names the conditions under which MLflow is the
wrong default: a tightly SageMaker- or Vertex-bound org (where the
hyperscaler's registry already has IAM + lineage wired in), an
experiment-tracking-heavy team that already runs W&B as the single
pane of glass, or a strongly governance-first regulated deployment
where the hyperscaler's model-governance tooling (SageMaker Model
Cards, Vertex AI Model Registry governance) is a procurement
requirement rather than a preference.

## A Note on Model Cards and Governance Sources

The model-card pattern used in this module traces back to [Mitchell et
al., "Model Cards for Model Reporting" (2019)](https://arxiv.org/abs/1810.03993)
and the operationalization in [Google's Model Card
Toolkit](https://github.com/tensorflow/model-card-toolkit). For
higher-stakes regulatory contexts, the [NIST AI Risk Management
Framework (AI RMF 1.0)](https://www.nist.gov/itl/ai-risk-management-framework)
and, for EU-operating teams, the
[EU AI Act](https://artificialintelligenceact.eu/) set the bar for what
a "sufficient" governance artifact looks like. This module teaches the
engineering pattern — artifact enforced at the promote API, checked
quarterly — that lets a platform team satisfy any of those frameworks
without re-architecting the registry each time a new one lands.
<!-- needs-research: confirm current NIST AI RMF version + latest EU AI Act stage before citing specific requirements to learners -->

## A Note on Platform / Engineer Split

This module lives in the ML Platform curriculum, which is the *design
and operations* track. Several exercises reference companion
implementations in `ai-infra-engineer-solutions` — specifically
`mod-106 ex-02` and `mod-106 ex-03` for the MLflow walk-through and
quality-gated promotion, `mod-106 ex-08` for the Argo Rollouts canary
manifests, and `mod-106 ex-10` for the governance-artifact templates
(model card, bias review, decision log). Those are the hands-on
engineer-track templates; this module leans on them rather than
reproducing them. If you want a deeper implementation pass on any
exercise, start from the companion and then come back to the rollback
tabletop (02), the governance gate (04), and the quarterly audit (05),
which are the deliverables the engineer track does not cover.

## Navigation

- Up: [`../../README.md`](../../README.md) — repo overview
- Curriculum spec: [`../../CURRICULUM.md`](../../CURRICULUM.md) — see
  "Module 06: Model Management & Registry"
- Previous: [`../mod-005-workflow-orchestration/`](../mod-005-workflow-orchestration/) — Workflow Orchestration
- Next: [`../mod-007-developer-experience/`](../mod-007-developer-experience/) — Developer Experience & Tooling

Model management is the module where most ML platforms either make the
registry the single answer to "what is in production" or discover, under
incident conditions, that there is no single answer. Register every
version, gate every promotion, log every transition, keep the governance
artifacts current, and the rest of the platform has a model lifecycle
it can reason about.
