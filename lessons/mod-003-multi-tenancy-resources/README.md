# Module 03: Multi-Tenancy & Resource Management

**Duration**: ~10 hours guided + 10-15 hours exercises
**Prerequisites**: Modules 01-02; Kubernetes namespace + RBAC + ResourceQuota fluency

## Overview

Single-tenant platforms scale only as far as their first noisy team. Multi-
tenant platforms scale to many tenants if — and only if — isolation, quotas,
fair scheduling, and cost attribution are designed in from the start.

Module 01 introduced multi-tenancy as a design *principle* (chapter 03 of that
module sketched the patterns on paper). Module 03 is where you build it. By
the end of this module you will have stood up tenant namespaces with
enforceable resource caps, pinned cross-tenant network boundaries, run a
distributed training job through a fair-share scheduler, and produced a daily
per-team cost rollup that finance could actually use.

The tone of this module is deliberately operational. We assume you can read a
Kubernetes manifest without panic and have deployed *something* to a cluster.
We will spend our time on the parts that platform teams consistently get
wrong in year one: quotas that look right but admit the pod anyway, network
policies that deny DNS by accident, and cost dashboards whose "unallocated"
bucket silently eats 30% of the bill.

## Learning objectives

By the end of this module, you will be able to:

1. **Choose** between namespace-per-team, namespace-per-project, virtual-
   cluster, and separate-cluster isolation, and defend the choice on cost,
   blast-radius, and operational complexity grounds.
2. **Implement** Kubernetes `ResourceQuota` + `LimitRange` for a tenant such
   that a misbehaving pod is rejected at admission rather than throttled at
   runtime, and explain the difference.
3. **Design** a default-deny `NetworkPolicy` posture that still lets tenants
   reach DNS, ingress, and shared monitoring without opening up cross-tenant
   pod-to-pod traffic.
4. **Operate** a fair-share scheduler (Volcano or Apache YuniKorn) for a
   mixed training + serving GPU cluster, including gang scheduling, queue
   borrowing, preemption, and backfill.
5. **Build** a cost-attribution pipeline that joins `kube-state-metrics`
   usage with cloud cost APIs, enforces required `team` / `cost_center`
   labels via Kyverno or OPA Gatekeeper, and produces per-team daily
   rollups that are accurate to within a few percent.
6. **Recognize** the failure modes platform teams repeatedly ship with —
   missing `PriorityClass`, unlabelled pods, over-permissive RBAC,
   silently-expanding "unallocated" cost buckets — and write policy to
   prevent them.

## What's in this Module

```
lessons/mod-003-multi-tenancy-resources/
├── README.md                                       (you are here)
├── lecture-notes/
│   ├── 01-isolation-models.md
│   ├── 02-fair-share-scheduling.md
│   └── 03-cost-attribution.md
├── exercises/
│   ├── exercise-01-namespace-per-team-setup.md
│   ├── exercise-02-resource-quota-enforcement.md
│   ├── exercise-03-network-policy-isolation.md
│   ├── exercise-04-fair-share-with-volcano.md
│   └── exercise-05-cost-attribution-pipeline.md
├── quizzes/
│   └── module-003-quiz.md
└── resources.md
```

### Lecture Notes (read in order)

| Chapter | Title | Approx. reading time | Maps to CURRICULUM.md topic |
| --- | --- | --- | --- |
| 01 | Tenant Isolation Models on Kubernetes | 90 min | "Multi-Tenancy in Kubernetes" (3h) + "Authentication & Authorization" overlap |
| 02 | Fair-Share Scheduling for ML | 90 min | "Resource Management" (2.5h) + "Priority Scheduling" (1.5h) |
| 03 | Cost Attribution | 60 min | "Cost Allocation" (2h) |

Chapter 01 covers the four-way isolation taxonomy (shared namespace,
namespace-per-team, namespace-per-project, vcluster, separate clusters), the
namespace-per-team baseline, and the handful of controls — `ResourceQuota`,
`LimitRange`, `NetworkPolicy`, `RoleBinding` — that make it stick.

Chapter 02 covers what the default scheduler can't do well: gang scheduling
for distributed training, hierarchical fair-share queues, GPU-aware preemption,
and backfill. It introduces Volcano and YuniKorn as the two production options
and gives you the "when to bring this in" test.

Chapter 03 covers the three building blocks of cost attribution — enforced
labels, per-namespace usage metrics, daily rollup + dashboard — and the
difference between showback and chargeback. It names the four failure modes
that eat attribution accuracy in practice.

### Exercises (do after the matching chapter)

| Exercise | After chapter | Estimated time | Format |
| --- | --- | --- | --- |
| 01 — Namespace-per-team setup | 01 | 120 min | `apply.sh` + `verify.sh` + YAML |
| 02 — ResourceQuota enforcement test | 01 | 90 min | Experiment log (`TESTS.md`) |
| 03 — NetworkPolicy cross-tenant isolation | 01 | 90 min | Manifests + test pods |
| 04 — Fair-share with Volcano | 02 | 150 min | `BENCHMARK.md` + Helm values |
| 05 — Cost-attribution pipeline | 03 | 180 min | Cron job + Postgres schema + Grafana panel |

You can do exercises in any order *after* the matching lecture chapter, but
the sequence above is the one with the smoothest difficulty ramp. Exercises
01-03 build on each other; 04 and 05 are independent.

Several exercises point to companion templates in
`ai-infra-engineer-solutions` (specifically `mod-104` for Kubernetes
multi-tenancy and `mod-106` for cost attribution). Those templates are
starting points, not answer keys — you are expected to adapt them, not copy
them.

### Quiz

`quizzes/module-003-quiz.md` contains 15 questions on isolation models,
quota mechanics, network policy, scheduling, and cost attribution. The
answers are embedded as checked boxes next to the correct option; try to
answer each question before you look at the next one.

A passing score is **75% (12 of 15)**. There is no separate answer key
file — the quiz is self-grading via the checkbox annotations.

## How to Work Through This Module

There is no single right path, but here is one that has worked for past
learners. The whole module is ~10 hours of guided study plus 10-15 hours of
exercises; most learners spread this over one to two weeks.

**Day 1 (2 hours)**: Read chapter 01. Skim the Kubernetes multi-tenancy
reference in `resources.md`. Sketch your target namespace layout for 3
fictional teams on paper.

**Day 2 (2-3 hours)**: Do exercise 01 (namespace-per-team setup). Treat the
`verify.sh` requirement seriously — the point of the exercise is the proof
that isolation holds, not the apply.

**Day 3 (2 hours)**: Do exercise 02 (ResourceQuota enforcement test). This
one is short but forces you to distinguish admission-time rejection from
runtime throttling, which is the most commonly-confused mechanic in the
module.

**Day 4 (2 hours)**: Do exercise 03 (NetworkPolicy isolation). Expect the
first pass to break DNS; that failure is the lesson.

**Day 5 (2 hours)**: Read chapter 02. If you have never deployed Volcano or
YuniKorn before, skim their respective docs linked in `resources.md` so you
know what you're installing.

**Day 6 (3 hours)**: Do exercise 04 (fair-share with Volcano). Run the
benchmark against the default scheduler as a baseline — the comparison is
the deliverable, not the installation.

**Day 7 (1-2 hours)**: Read chapter 03. Pick a cost model (per-pod-hour,
per-node-share, or a weighted hybrid) and write a one-paragraph defense of
your pick before moving on.

**Day 8 (3 hours)**: Do exercise 05 (cost-attribution pipeline). Budget time
for the labelling policy — enforcing labels is where real pipelines fail.

**Day 9 (1 hour)**: Take the quiz. Re-read any chapter whose questions you
had to guess at.

If you are moving slowly, that is fine. Multi-tenancy is the module where
platform teams most often ship something that looks correct and silently
isn't — depth matters more than speed here.

## How to Self-Assess

After finishing the module, you should be able to do the following without
referring back to the notes:

- [ ] Name the four tenant isolation models and give one scenario each where
      it is the right choice.
- [ ] Explain, in one paragraph, the difference between a `ResourceQuota`
      and a `LimitRange`, and why both are needed.
- [ ] Describe at least two ways a default-deny `NetworkPolicy` can silently
      break a tenant's workload, and how to prevent each.
- [ ] Explain gang scheduling, and give an ML workload that fails without it.
- [ ] Describe a hierarchical queue setup where `training` and `serving` can
      borrow from each other but each has a guaranteed floor.
- [ ] Explain showback vs chargeback and name the signal you would use to
      decide the platform is ready to transition.
- [ ] List the four cost-attribution failure modes from chapter 03 and the
      mitigation for each.
- [ ] Score 75%+ on the quiz without notes.

If you can do all of the above, you are ready for Module 04. If 2-3 of these
feel shaky, that is normal — flag them and revisit when you encounter the
relevant concept again in later modules.

## A Note on Tools

This module names several tools by category and by product: Volcano,
Apache YuniKorn, Kueue, vcluster, Hierarchical Namespace Controller, Kyverno,
OPA Gatekeeper, OpenCost, Kubecost, `kube-state-metrics`. **You are not
expected to be fluent in all of these.** The exercises pin specific choices
(Volcano for scheduling, Kyverno for policy, a hand-written rollup for cost)
so you can finish the module without evaluating every tool first.

The resources list in `resources.md` is where to go when you need to
evaluate alternatives for a real deployment — the principles here are
portable; the specific tooling is not.

## A Note on Platform / Engineer Split

This module lives in the ML Platform curriculum, which is the *design and
operations* track. Several exercises reference companion implementations in
`ai-infra-engineer-solutions` (for example, `mod-104 ex-14` for the
multi-tenancy template and `mod-106 ex-12` for a per-model cost variant).
Those are the hands-on engineer-track templates; this module leans on them
rather than reproducing them. If you want a deeper implementation pass on
any exercise, start from the companion.

## Navigation

- Up: [`../../README.md`](../../README.md) — repo overview
- Curriculum spec: [`../../CURRICULUM.md`](../../CURRICULUM.md) — see
  "Module 03: Multi-Tenancy & Resource Management"
- Previous: [`../mod-002-api-design/`](../mod-002-api-design/) — API Design for ML Platforms
- Next: [`../mod-004-feature-store/`](../mod-004-feature-store/) — Feature Store Architecture

Multi-tenancy is the module where most platforms either grow up or ossify.
Build the controls once, build them correctly, and the rest of the platform
has room to scale.
