# Module 04: Feature Store Architecture

**Duration**: ~10 hours guided + 12 hours exercises
**Prerequisites**: Modules 01-03; working SQL + Python; comfort with Parquet,
Redis, and at least one streaming engine (Kafka + Flink or Spark Structured
Streaming)

## Overview

Feature stores solve the single most common silent failure mode in production
ML: training-serving skew. A feature computed one way at training time and a
subtly different way at serving time will make a model that scores well on
held-out data and performs worse than the baseline in production — and the
divergence is hard to catch because both pipelines run to completion, both
look healthy, and neither throws. A feature store removes the opportunity for
divergence by making the feature definition itself the single source of
truth, shared by offline training and online serving.

Module 01 introduced data platforms as a shared service; Module 03 made that
service multi-tenant. Module 04 is where the data layer earns its keep. By
the end of this module you will have stood up an open-source feature store
(Feast) with a Parquet offline store and a Redis online store, written a
point-in-time-correct training join, shipped a materialization pipeline with
backfill support, added a streaming feature, and wired the operational
dashboards + alerts that keep the whole thing honest.

The tone of this module is deliberately operational. We assume you can read a
SQL query and a PyFlink job without panic. We will spend our time on the
parts that ML platform teams consistently get wrong in year one: a "join"
that quietly leaks future data into the training set, a materialization job
whose freshness looks fine on average but misses the 3am window that matters,
a drift metric that fires on noise and misses real shift, and an online store
whose per-request latency is 2ms at the median and 180ms at the p99 when
the model actually needs the value.

## Learning objectives

By the end of this module, you will be able to:

1. **Articulate** the two problems a feature store exists to solve
   (training-serving skew and cross-team feature reuse), name a case where
   each dominates, and defend when *not* to adopt one.
2. **Implement** a three-layer feature store — offline (Parquet on S3 or
   equivalent), online (Redis or equivalent key-value store), and registry
   (Feast on Postgres) — and explain the role of each layer in one sentence.
3. **Write** a point-in-time-correct training join that retrieves each
   feature as of the label timestamp (not "now"), and demonstrate measurable
   accuracy inflation when the PIT contract is removed.
4. **Operate** a materialization pipeline (offline → online) with incremental
   daily runs, parameterized backfill over arbitrary date ranges, and
   per-FeatureView duration + row-count metrics.
5. **Build** a streaming feature with correct windowing, watermarking, and
   checkpointing semantics, and decide between Lambda and Kappa topologies
   for a given feature.
6. **Monitor** a feature store in production: per-FeatureView freshness,
   materialization SLA, online store capacity, per-feature drift (PSI), and
   per-feature read QPS + p99 latency — and write runbooks for the top five
   alerts.
7. **Recognize** the failure modes that cost ML platform teams six-figure
   outages — naive joins that leak future values, TTLs shorter than
   materialization lag, drift alerts set on PSI of raw counts, Redis
   evictions masquerading as missing features — and write guardrails to
   prevent each.

## What's in this Module

```
lessons/mod-004-feature-store/
├── README.md                                       (you are here)
├── lecture-notes/
│   ├── 01-why-feature-stores.md
│   ├── 02-feast-architecture.md
│   └── 03-streaming-features.md
├── exercises/
│   ├── exercise-01-feast-quickstart.md
│   ├── exercise-02-point-in-time-join.md
│   ├── exercise-03-materialization-pipeline.md
│   ├── exercise-04-streaming-feature.md
│   └── exercise-05-feature-store-operations.md
├── quizzes/
│   └── module-004-quiz.md
└── resources.md
```

### Lecture Notes (read in order)

| Chapter | Title | Approx. reading time | Maps to CURRICULUM.md topic |
| --- | --- | --- | --- |
| 01 | Why Feature Stores Exist | 60 min | "Feature Store Fundamentals" (2.5h) |
| 02 | Feast Architecture in Practice | 90 min | "Online Feature Store" (2.5h) + "Offline Feature Store" (2h) |
| 03 | Streaming Features | 90 min | "Real-Time Features" (as covered in CURRICULUM.md Module 04 topics) |

Chapter 01 covers the two problems feature stores exist to solve — training-
serving skew and cross-team feature reuse — and the three-layer architecture
(offline + online + registry) that solves them. It is deliberately short:
the mechanics are in Chapter 02, but the motivation needs to come first or
the mechanics look like gratuitous infrastructure.

Chapter 02 is the longest chapter and covers the Feast object model
(`Entity`, `FeatureView`, `Source`, `FeatureService`, TTL), the online +
offline storage choices, materialization semantics, the deployment topology,
and the operational concerns you will need to monitor from day one. It
closes with the "when to switch from Feast" heuristic — this is a module
that names a tool, not a module that ties you to it.

Chapter 03 covers streaming features: tumbling vs sliding windows,
watermarks and lateness handling, exactly-once vs at-least-once semantics,
and the Lambda-vs-Kappa topology decision. It names the per-feature
freshness gauge as the one metric you cannot ship without.

### Exercises (do after the matching chapter)

| Exercise | After chapter | Estimated time | Format |
| --- | --- | --- | --- |
| 01 — Feast quickstart | 02 | 150 min | `feature_repo/` + `materialize.py` + `get_training.py` + `serve.py` |
| 02 — Point-in-time correctness | 02 | 120 min | `LEAKAGE_REPORT.md` with measured accuracy delta |
| 03 — Materialization pipeline with backfill | 02 | 180 min | Airflow DAG + 30-day backfill smoke test |
| 04 — Streaming feature | 03 | 180 min | PyFlink (or Spark Structured Streaming) job + load test |
| 05 — Feature store operations | 03 | 180 min | Prometheus exporter + Grafana dashboard + `RUNBOOK.md` |

Exercises are sequenced to build on each other. Exercise 01 stands up the
feature repo you will reuse in 02 and 03. Exercise 04 introduces a streaming
source that Exercise 05 then monitors. You can skip ahead if you already
have a Feast repo lying around, but the point-in-time exercise (02) and the
operations exercise (05) are the two most people underestimate — leave real
time for them.

Several exercises point to companion templates in
`ai-infra-engineer-solutions`. Specifically, `mod-106 ex-07` is the
end-to-end Feast setup used as the starting point for Exercise 01, and
`mod-105 ex-11` is the lambda-architecture stack (PyFlink + Spark +
serving) that Exercise 04 can reuse rather than reimplement from scratch.
Those templates are starting points, not answer keys — you are expected to
adapt them (changing data sources, feature definitions, and drift thresholds
to match the exercise) rather than copy them.

### Quiz

`quizzes/module-004-quiz.md` contains 15 questions covering feature store
concepts, PIT correctness, Feast mechanics, streaming semantics, and
operational tradeoffs. The answers are embedded as checked boxes next to the
correct option; try to answer each question before you look at the next one.

A passing score is **75% (12 of 15)**. There is no separate answer key
file — the quiz is self-grading via the checkbox annotations.

## How to Work Through This Module

There is no single right path, but here is one that has worked for past
learners. The whole module is ~10 hours of guided study plus ~12 hours of
exercises; most learners spread this over one to two weeks.

**Day 1 (1 hour)**: Read chapter 01. Write down, in one paragraph, the
training-serving skew story from your current (or most recent) project —
even a single instance where a computed feature did not match what serving
saw. If you cannot think of one, pick a public post-mortem from the
`resources.md` reading list; the DoorDash and Airbnb writeups both have
concrete examples.

**Day 2 (3 hours)**: Read chapter 02. Do exercise 01 (Feast quickstart). The
goal here is not a perfect design — it is to feel the three layers in your
hands. You should finish this exercise able to point at, in your own files,
the thing that is the registry, the thing that is the offline store, and
the thing that is the online store.

**Day 3 (2 hours)**: Do exercise 02 (point-in-time correctness). Expect the
naive join to produce an offline accuracy that looks too good; the delta is
the point of the exercise. If you measure less than ~5% inflation, your
synthetic dataset probably has features that don't actually evolve in time
and the exercise did not test what it meant to — go back and fix the data
generator before concluding.

**Day 4 (3 hours)**: Do exercise 03 (materialization pipeline). Treat the
backfill case seriously. The daily increment is the easy path; the backfill
is where real pipelines corrupt data by running two overlapping jobs or by
silently skipping a window.

**Day 5 (2 hours)**: Read chapter 03. If you have never deployed PyFlink or
Spark Structured Streaming before, skim the quickstart docs linked in
`resources.md` so you know what you are installing.

**Day 6 (3 hours)**: Do exercise 04 (streaming feature). Budget time for the
load test — it is where the exercise actually proves the stream converges,
and it is the part most learners skip.

**Day 7 (3 hours)**: Do exercise 05 (operations). The dashboard is the
visible deliverable; the runbook is the deliverable you will actually use.
Write the runbook for a tired engineer at 3am, not for a future reviewer.

**Day 8 (1 hour)**: Take the quiz. Re-read any chapter whose questions you
had to guess at.

If you are moving slowly, that is fine. Feature stores are the module where
ML platform teams most often ship something that looks correct and silently
leaks future data or silently stales out — depth matters more than speed
here.

## How to Self-Assess

After finishing the module, you should be able to do the following without
referring back to the notes:

- [ ] State the two problems feature stores solve, and name a workload where
      each is the dominant motivation.
- [ ] Draw the three-layer architecture (offline / online / registry) and
      label the arrow where a materialization job sits.
- [ ] Explain, in one paragraph, what a point-in-time-correct join is and
      how a naive join leaks future data.
- [ ] Describe what Feast's `TTL` controls and why it is not a cache
      eviction setting.
- [ ] Decide between tumbling and sliding windows for a given feature, and
      defend the choice on cost and responsiveness grounds.
- [ ] Explain the Lambda vs Kappa choice and name a signal that pushes you
      toward one over the other.
- [ ] List at least four of the per-feature or per-pipeline metrics from
      Exercise 05 and describe the alert threshold + the first runbook step
      for each.
- [ ] Score 75%+ on the quiz without notes.

If you can do all of the above, you are ready for Module 05. If 2-3 of these
feel shaky, that is normal — flag them and revisit when you encounter the
relevant concept again in later modules, especially when workflow
orchestration (Module 05) starts calling materialization DAGs.

## A Note on Tools

This module names several tools by category and by product:
[Feast](https://feast.dev/), [Tecton](https://www.tecton.ai/),
[Hopsworks](https://www.hopsworks.ai/),
[Featureform](https://featureform.com/),
[Feathr](https://feathr-ai.github.io/feathr/), Redis, DynamoDB, Bigtable,
Snowflake, BigQuery, Parquet, PyFlink, Spark Structured Streaming, Kafka,
Airflow, Prometheus, Grafana. **You are not expected to be fluent in all of
these.** The exercises pin specific choices (Feast for the registry, Redis
for the online store, Parquet for the offline store, PyFlink for streaming,
Airflow for orchestration) so you can finish the module without evaluating
every tool first.

The resources list in `resources.md` is where to go when you need to
evaluate alternatives for a real deployment — the principles here are
portable; the specific tooling is not. In particular, Chapter 02 names the
conditions under which Feast is the wrong default (sub-10ms p99 at 100K+
QPS, strong enterprise-governance requirements, heavy streaming-first
feature volume), and the production case studies in `resources.md` (Uber
Michelangelo, Airbnb Zipline, DoorDash) are the canonical reads for what a
managed or in-house build looks like at scale.

## A Note on Platform / Engineer Split

This module lives in the ML Platform curriculum, which is the *design and
operations* track. Several exercises reference companion implementations in
`ai-infra-engineer-solutions` — specifically `mod-106 ex-07` for the Feast
walkthrough and `mod-105 ex-11` for the lambda streaming + batch
composition. Those are the hands-on engineer-track templates; this module
leans on them rather than reproducing them. If you want a deeper
implementation pass on any exercise, start from the companion and then come
back to the operations and PIT-correctness exercises here, which are the
parts the engineer track does not cover in depth.

## Navigation

- Up: [`../../README.md`](../../README.md) — repo overview
- Curriculum spec: [`../../CURRICULUM.md`](../../CURRICULUM.md) — see
  "Module 04: Feature Store Architecture"
- Previous: [`../mod-003-multi-tenancy-resources/`](../mod-003-multi-tenancy-resources/) — Multi-Tenancy & Resource Management
- Next: [`../mod-005-workflow-orchestration/`](../mod-005-workflow-orchestration/) — Workflow Orchestration

Feature stores are the module where most ML platforms either take training-
serving skew seriously or learn about it the expensive way. Build the three
layers once, enforce point-in-time correctness everywhere, monitor freshness
on every feature, and the rest of the ML platform has data it can trust.
