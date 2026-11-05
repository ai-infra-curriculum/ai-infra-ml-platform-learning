# Module 08: Observability & Monitoring

**Duration**: ~8 hours guided + 6-8 hours exercises
**Prerequisites**: Modules 01-07; Python; comfort reading a PromQL expression
and a Grafana dashboard JSON; at least one incident you have been paged for
whose postmortem mentioned "we didn't have the right metric"

## Overview

Observability is the module where the platform you have built so far meets
the 3am version of itself. Modules 01-07 produced a tenanted cluster, an
API, a feature store, an orchestrator, a model registry, and the developer-
experience surface that lets anyone actually use them — all of which are
easy to run while everything is working. This module is about what the
platform tells you when something is not, specifically for ML workloads
whose failure modes do not show up in the generic HTTP metrics every web
service already emits. A model that is drifting silently, a slice of users
whose accuracy has collapsed, an LLM whose time-to-first-token has doubled
for a specific prompt pattern, a per-model cost that has tripled because a
retry loop is paying for autoscaled GPUs — none of these produce a 5xx,
and none of them show up on the dashboard a web-services SRE would have
built. This module teaches the signals, the dashboard hierarchy, the
alerting discipline, and the runbooks that a specifically-ML platform
owes its users.

Modules 01-07 assumed observability as a background service. Module 08 is
where that assumption gets paid for. By the end of this module you will
have built a per-model Grafana dashboard that answers the four questions a
model owner actually asks in order (is it serving? / how well? / is it
*correct*? / what does it cost?), wired multi-window burn-rate alerts
against an availability and latency SLO using the [Google SRE Workbook
pattern](https://sre.google/workbook/alerting-on-slos/), stood up a daily
drift job that computes PSI per feature and alerts above a conventional
threshold, built a per-model cost rollup pipeline that attributes
cluster-level spend back to `(team, model_name)`, and authored a runbook
library covering the five alerts that most ML platforms get paged on. The
deliverables intentionally cover one dashboard, one alerting pattern, one
drift pipeline, one cost pipeline, and one runbook set — the smallest
complete observability surface that a platform team can defend when the
first incident lands.

The tone of this module is deliberately operational. We assume you can
read a PromQL expression, have seen a Grafana dashboard rendered against
a real Prometheus, and have at some point been handed a terse alert that
required thirty minutes of archaeology to understand. We will spend our
time on the parts that ML platform teams consistently get wrong in year
one: a dashboard that is really five copies of a web-services template
with "model" in the title and no drift panel, a burn-rate alert that
pages on a single 1-minute spike and trains the oncall to ignore the
channel, a drift alert configured on the day of launch and never
recalibrated as the feature distribution naturally shifted, a "per-model
cost" that is really the whole cluster's bill divided by the number of
deployments, and a runbook that terminates in "page the model owner"
with no further guidance for the engineer who is holding the pager at
3am.

## Learning objectives

By the end of this module, you will be able to:

1. **Distinguish** the three pillars — metrics, logs, traces — on
   cardinality, cost, and the question each one is cheapest to answer,
   and name the ML-specific signals (per-class prediction distribution,
   per-feature drift, slice metrics, TTFT for LLMs, per-model cost,
   inference-log freshness) that do not come for free from any of the
   three and have to be instrumented on purpose.
2. **Design** a model-owner dashboard whose panels, read top to bottom,
   answer *is it serving? → how well is it serving? → is it serving
   correctly? → what does it cost?* — and defend the panel order in a
   platform-team review by showing which alert it short-circuits at
   each row.
3. **Write** an SLI and an SLO for an ML serving service: availability
   (non-5xx fraction), latency (p95 under N ms), model accuracy (rolling
   7-day delta from baseline), and drift (PSI below a threshold on a
   named feature set) — and explain why aggregating accuracy across
   slices is the SLI that most masks real regressions.
4. **Implement** multi-window burn-rate alerts against those SLOs using
   [the Google SRE Workbook two-window pattern](https://sre.google/workbook/alerting-on-slos/) —
   a fast-burn page when a 5-minute + 1-hour window are both consuming
   budget above a 14.4× threshold, and a slow-burn ticket when a
   30-minute + 6-hour window are both burning above a lower threshold —
   and explain why a single-window alert produces either false pages or
   missed outages depending on which single window you pick.
5. **Codify** an error-budget policy that names what happens at
   >50% / 20-50% / <20% budget remaining (free to deploy / deploy with
   caution / feature freeze and reliability work only), and defend it
   against a product manager who wants to ship on Friday with 10%
   budget left.
6. **Operate** a daily drift pipeline that reads inference logs, computes
   Population Stability Index per feature against a reference window,
   exports the result as a Prometheus gauge, and alerts when PSI sustains
   above a conventional "significant shift" threshold — and name the two
   failure modes that make drift alerts untrustworthy (threshold set at
   launch and never revisited, reference window set to a single day
   whose traffic was unrepresentative).
7. **Attribute** cluster spend to the right model with required labels
   (`team`, `cost_center`, `model_name`, `environment`) enforced at the
   policy layer, a nightly rollup that joins pod-hours to per-pod cost,
   and a dashboard that surfaces the per-model daily and per-team monthly
   view — and explain the three reasons "unallocated" tends to grow
   silently (missing labels, cluster overhead unassigned, spot-interrupt
   rerun cost not attributed).
8. **Recognize** the failure modes that cost ML platform teams their
   observability credibility — a dashboard that pages on noise and gets
   muted, a drift alert nobody actions because nobody knows what to do
   when it fires, a runbook that stops at "contact the model owner,"
   a cost rollup whose "platform overhead" bucket is bigger than any
   model's bill — and name the guardrail that prevents each.

## What's in this Module

```
lessons/mod-008-observability/
├── README.md                                       (you are here)
├── lecture-notes/
│   ├── 01-ml-specific-signals.md
│   ├── 02-slo-burn-rate-alerts.md
│   └── 03-cost-observability.md
├── exercises/
│   ├── exercise-01-build-model-dashboard.md
│   ├── exercise-02-burn-rate-alerts.md
│   ├── exercise-03-drift-monitoring.md
│   ├── exercise-04-cost-rollup-pipeline.md
│   └── exercise-05-incident-runbook.md
├── quizzes/
│   └── module-008-quiz.md
└── resources.md
```

### Lecture Notes (read in order)

| Chapter | Title | Approx. reading time | Maps to CURRICULUM.md topic |
| --- | --- | --- | --- |
| 01 | ML-Specific Signals | 60 min | "Observability Fundamentals" (2h) + "Metrics Collection" (2.5h) |
| 02 | SLOs and Burn-Rate Alerts | 60 min | "Alerting and SLOs" (3h) |
| 03 | Cost Observability | 60 min | "Metrics Collection" (0.5h) + "Platform Observability" (1h) |

Chapter 01 covers the ML-specific signals that generic HTTP observability
does not emit. It names the questions model owners actually ask (is
accuracy the same as last week? / is feature X distributionally stable? /
which model costs the most?) and maps each to the signal that answers it:
per-class prediction distribution against a reference, PSI per feature
against a reference window, slice metrics against ground-truth labels,
TTFT for LLM inference, per-model cost from the rollup pipeline, and
inference-log freshness from the log pipeline itself. It closes with the
dashboard hierarchy (serving → performance → correctness → cost) and the
mapping from each row to a specific alert — the panel order is not
cosmetic; it is the order a model owner reads the dashboard in when they
are already paged, and a dashboard whose correctness row is three scrolls
below the fold is a dashboard nobody reads under incident conditions.

Chapter 02 is the SLO and alerting chapter. It covers the four SLIs that
most ML services end up with (availability, latency, model accuracy,
drift), the typical SLO shapes (99.5% availability over 30 days, p95
latency under N ms, 7-day rolling accuracy within 5 percentage points of
baseline, PSI below 0.25 on a named feature set), and the multi-window
burn-rate pattern from [the Google SRE Workbook chapter on alerting on
SLOs](https://sre.google/workbook/alerting-on-slos/). It names why the
two-window-both-firing structure is the alert shape teams end up with:
a single short window pages on instantaneous spikes and trains people to
ignore the channel, a single long window misses the fast-burn outages
that eat the entire monthly budget in an afternoon, and the AND of two
windows at different time scales is the smallest reliable predicate. It
closes with the error-budget policy — the written rule that converts an
SLO from a dashboard number into an organizational commitment.

Chapter 03 is the cost chapter. It covers the required-labels contract
(`team`, `cost_center`, `model_name`, `environment`, enforced by the
Kyverno policies from Module 03), the nightly rollup pattern that joins
pod-hours from kube-state-metrics to per-pod cost from the cloud
provider's billing API, and the dashboard and alert surfaces that
expose the result (per-model daily cost, per-team monthly cost, a
per-cluster spike alert for week-over-week cost changes above a
threshold). It closes with the three recurring failure modes
(unlabeled workloads landing in "unallocated," cluster overhead that
the platform team quietly carries, spot-interrupt rerun costs not
attributed to the training run that triggered the rerun) and the
guardrail each one requires.

### Exercises (do after the matching chapter)

| Exercise | After chapter | Estimated time | Format |
| --- | --- | --- | --- |
| 01 — Per-model dashboard | 01 | 150 min | Grafana JSON with RPS / latency / errors / prediction distribution / drift / cost panels in the four-row hierarchy |
| 02 — SLO + burn-rate alerts | 02 | 120 min | PromQL for availability + latency SLIs, multi-window burn-rate alerts (5m+1h fast, 30m+6h slow), routing to PagerDuty + Slack |
| 03 — Drift monitoring end-to-end | 01 | 150 min | Daily job reading inference logs → PSI per feature → Prometheus gauges → alert at PSI > 0.25 |
| 04 — Cost rollup pipeline | 03 | 120 min | Nightly cron: pod-hours × $/pod-hour → `(date, team, model_name, cost_usd)` in Postgres → Grafana panel |
| 05 — Incident runbook library | 02 | 90 min | Runbooks for HighErrorRate, HighDrift, ModelAccuracyRegression, BudgetExhausted, TrainingDAGSLAMiss (detection → triage tree → common causes → remediation) |

Exercises 01 and 02 build the dashboard and alerting foundation; 03 and
04 are the two pipeline exercises (drift and cost) that extend the
dashboard with signals generic HTTP observability does not produce;
05 is the writing exercise that most learners underestimate — the
runbook is where you discover whether your dashboard is actually
actionable, because the runbook author is the first person who has to
turn an alert into a next step without a slack of context. Exercise 03
(drift) and Exercise 05 (runbooks) are the two that most learners
finish and then quietly realize are the hardest in the module — drift
because every synthetic dataset either triggers constantly or never
triggers and calibrating the threshold is the exercise, and runbooks
because catalog work surfaces how much of the on-call engineer's
understanding was previously held only in the senior engineer's head.

Several exercises reference companion templates in
`ai-infra-engineer-solutions`. Specifically, `mod-108 ex-04` covers the
dashboards-as-code implementation that backs Exercise 01, `mod-108 ex-08`
has the Sloth-format SLO spec and the full multi-window burn-rate alert
set that backs Exercise 02, the `mod-108` monitoring module backs
Exercise 03's drift pipeline (with a working PSI implementation against
real inference logs), `mod-106 ex-12` covers the per-model cost
attribution that backs Exercise 04, and `mod-108 ex-09` has the runbook
library that backs Exercise 05. Those templates are starting points,
not answer keys — you are expected to adapt them (changing panel
layouts, SLO thresholds, drift feature sets, cost labels, and runbook
failure modes to match your scenario) rather than copy them.
<!-- needs-research: confirm final naming / exercise numbering of mod-108 companion in ai-infra-engineer-solutions before publishing -->

### Quiz

`quizzes/module-008-quiz.md` contains 10 questions covering the
ML-specific-signals vocabulary, PSI and its interpretation, where
drift should be computed, the dashboard hierarchy, burn-rate alert
semantics, the multi-window pattern, per-model cost attribution, the
PSI threshold convention, error-budget policies, and the signal that
generic web observability most consistently misses. The answers are
embedded as checked boxes next to the correct option; try to answer
each question before you look at the next one.

A passing score is **70% (7 of 10)**. There is no separate answer key
file — the quiz is self-grading via the checkbox annotations, and
rationale for every answer is at the bottom of the quiz file.

## How to Work Through This Module

There is no single right path, but here is one that has worked for past
learners. The whole module is ~8 hours of guided study plus ~6-8 hours
of exercises; most learners spread this over about a week.

**Day 1 (1 hour)**: Read chapter 01. On paper, write down the top five
panels on the model-serving dashboard your current (or most recent)
team actually watches. For each one, note whether it answers *is it
serving?*, *how well?*, *is it correct?*, or *what does it cost?* If
the correctness row is empty or lives on a separate dashboard, that is
itself a finding — note it for exercise 01.

**Day 2 (2.5 hours)**: Do exercise 01 (per-model dashboard). Build it
against a Prometheus with real (or synthetic-but-moving) traffic rather
than a screenshot in a doc — a dashboard that has never been rendered
against live metrics has not been finished. The deliverable is the
Grafana JSON plus a one-paragraph write-up explaining what the drift
panel does when the model is behaving normally versus when it is not;
most learners discover during this write-up that their drift panel's
y-axis is useless under normal traffic and recalibrate the panel.

**Day 3 (1 hour)**: Read chapter 02. If you have never written a
multi-window burn-rate alert expression, skim the [Google SRE Workbook
alerting chapter](https://sre.google/workbook/alerting-on-slos/) before
starting exercise 02 so the PromQL you write is not the first
burn-rate alert you have ever seen. Note how the workbook justifies
the 14.4× threshold — that number is not arbitrary and the derivation
matters for defending the alert in review.

**Day 4 (2 hours)**: Do exercise 02 (SLO + burn-rate alerts). Define
the SLIs first, write the SLOs second, and only then write the alerts —
the ordering matters because "what percentage of what over how long"
is harder than the PromQL it compiles down to. The deliverable is the
alert manifests plus a one-paragraph defense of the 14.4× threshold
and the choice of 5m+1h versus 30m+6h windows. If you cannot defend
either choice, re-read the workbook chapter before marking the
exercise done.

**Day 5 (2.5 hours)**: Do exercise 03 (drift monitoring end-to-end).
The hardest part is not the PSI math — it is deciding what the
reference window is and what the feature set is. Pick both before you
start, write both decisions down in the exercise write-up, and defend
them in a one-paragraph note; the exercise is as much about naming
the right reference as it is about computing PSI. Trigger the alert
on purpose by injecting a shifted feature into the inference stream
and screenshot the resulting Prometheus value.

**Day 6 (1 hour)**: Read chapter 03. Pull up the per-pod cost dashboard
for your current team's cluster (or any public FinOps dashboard such as
[OpenCost's reference dashboards](https://www.opencost.io/)) and note
which panels present "per-team" or "per-service" versus which present
"per-cluster" or "unallocated." If "unallocated" is the biggest bar,
that is the pattern this chapter names.

**Day 7 (2 hours)**: Do exercise 04 (cost rollup pipeline). Build it as
a real cron (or a scheduled Argo Workflow) rather than a script you
run by hand — the discipline is partly about the pipeline being
reproducible and partly about discovering that the labels are missing
on a non-trivial fraction of your pods when you first try to join. The
deliverable is the `(date, team, model_name, cost_usd)` rows in
Postgres, the Grafana panel, *and* a one-paragraph note on what fell
into "unallocated" on the first run and what you did about it.

**Day 8 (1.5 hours)**: Do exercise 05 (incident runbook library). Each
runbook has four sections (detection, triage decision tree, common
causes, remediation) and no runbook is allowed to terminate in "page
the model owner" without specifying what information the pager-holder
provides to the model owner on the handoff — the whole point of a
runbook is that the oncall engineer narrows the problem before
escalating, not that they re-page a different human cold.

**Day 9 (1 hour)**: Take the quiz. Re-read any chapter whose
questions you had to guess at, especially chapter 02 if the burn-rate
or error-budget-policy questions felt arbitrary — those are the parts
most learners skim and then cannot defend when an SRE asks why the
alert fired at the threshold it did.

If you are moving slowly, that is fine. Observability is the module
where ML platform teams most often ship an operationally excellent
platform whose incidents nevertheless take an hour to diagnose — depth
here pays itself back every incident the oncall engineer resolves from
the runbook instead of from memory.

## How to Self-Assess

After finishing the module, you should be able to do the following without
referring back to the notes:

- [ ] Name the three pillars (metrics, logs, traces) and the question
      each one is cheapest to answer, plus the ML-specific signals that
      none of the three emit for free.
- [ ] Describe the four-row model-owner dashboard hierarchy (serving →
      performance → correctness → cost) and name the alert each row
      short-circuits.
- [ ] Write a PSI formula from memory and name the conventional
      "significant shift" threshold (0.25).
- [ ] Explain why drift is a daily batch job and not an inline
      per-request check, and name the two failure modes that make
      drift alerts untrustworthy (stale threshold, unrepresentative
      reference window).
- [ ] Write the four typical ML SLIs (availability, latency, accuracy,
      drift) with example SLO shapes for each.
- [ ] Describe the multi-window burn-rate pattern (two windows at
      different scales, both burning above threshold, AND'd together)
      and explain why a single-window alert is worse on both false
      positives and false negatives.
- [ ] Describe the three tiers of an error-budget policy (>50% free,
      20-50% caution, <20% freeze) and defend the policy to a PM who
      wants to ship on Friday.
- [ ] Name the four required labels for per-model cost attribution
      (`team`, `cost_center`, `model_name`, `environment`) and the
      three failure modes that grow the "unallocated" bucket (missing
      labels, unassigned cluster overhead, spot-interrupt rerun cost).
- [ ] Write the four-section runbook shape (detection, triage tree,
      common causes, remediation) and name the one thing a runbook is
      not allowed to terminate in ("page the model owner" without
      handoff information).
- [ ] Score 70%+ on the quiz without notes.

If you can do all of the above, you are ready for Module 09. If 2-3 of
these feel shaky, that is normal — flag them and revisit when you
encounter the relevant concept again in later modules, especially
when Module 09 (security & governance) wires the audit-log pattern
into the same alerting infrastructure named here, and Module 10
(capacity & scaling) builds on the cost rollup pipeline to drive
autoscaling and capacity-planning decisions.

## A Note on Tools

This module names several observability products by name:
[Prometheus](https://prometheus.io/),
[Grafana](https://grafana.com/),
[OpenTelemetry](https://opentelemetry.io/),
[Jaeger](https://www.jaegertracing.io/) and
[Tempo](https://grafana.com/oss/tempo/) (tracing backends),
[Loki](https://grafana.com/oss/loki/) (log aggregation),
[Evidently AI](https://docs.evidentlyai.com/) (open-source ML drift and
quality monitoring),
[Arize AI](https://arize.com/) (commercial ML observability),
[Datadog](https://www.datadoghq.com/) and
[New Relic](https://newrelic.com/) (commercial full-stack observability),
[OpenCost](https://www.opencost.io/) (open-source Kubernetes cost
allocation), and [Sloth](https://sloth.dev/) (an SLO spec compiler
that generates Prometheus recording rules and multi-window alert
expressions from a YAML SLO definition).
**You are not expected to be fluent in all of these.** The exercises
pin Prometheus + Grafana as the metric and dashboard stack (because
their open-source footprint, PromQL vocabulary, and dashboards-as-code
support are the closest match to what a self-hosted ML platform
actually runs) and the Google SRE Workbook multi-window pattern as the
alerting discipline (because it is framework-neutral and ports to any
SLO tooling you may later evaluate), so you can finish the module
without evaluating every tool first.

The resources list in [`resources.md`](./resources.md) is where to go
when you need to evaluate alternatives for a real deployment — the
principles here are portable; the specific tooling is not. In
particular, chapter 01 names the conditions under which Prometheus is
the wrong default: a VictoriaMetrics- or Mimir-standardized org where
long-term storage and high-cardinality are first-class concerns rather
than bolt-ons, a strongly tracing-first org that leads with
OpenTelemetry collectors and treats Prometheus as a sidecar, or a
hyperscaler-locked team whose observability contract is already
Datadog or CloudWatch and whose procurement path does not permit
another backend. In each case the dashboard hierarchy, the SLI and SLO
shapes, the burn-rate alert pattern, and the runbook structure port
unchanged; only the query language and the exporter plumbing differ.

## A Note on Observability Sources

The multi-window burn-rate pattern used in this module traces back to
[the Google SRE Workbook chapter on alerting on
SLOs](https://sre.google/workbook/alerting-on-slos/), which is the
canonical reference for the 14.4× fast-burn threshold and the two-window
AND structure. The three-pillars framing (metrics, logs, traces) and the
"observability vs. monitoring" distinction trace to the
[OpenTelemetry documentation](https://opentelemetry.io/docs/concepts/) and
the surrounding industry writing that reframed instrumentation as a
first-class engineering concern rather than an operations afterthought.
The dashboard hierarchy — serving → performance → correctness → cost —
is the ML-specific adaptation of the generic [RED method (Rate, Errors,
Duration)](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)
and [USE method (Utilization, Saturation, Errors)](https://www.brendangregg.com/usemethod.html)
dashboard patterns, extended with the correctness and cost rows that
are specifically-ML concerns. The PSI threshold convention (0.1 as
"small shift," 0.25 as "significant shift") traces back to credit-
scoring literature where PSI originated and is restated in the
[Evidently documentation](https://docs.evidentlyai.com/).
This module teaches the engineering pattern — ML-specific signals
plus Google SRE discipline plus per-model cost attribution — that lets
a platform team run a model-serving fleet whose incidents are
diagnosable from the dashboard rather than from tribal knowledge.
<!-- needs-research: confirm PSI threshold convention (0.1 / 0.25) origin and primary citation before presenting it as canonical -->

## A Note on Platform / Engineer Split

This module lives in the ML Platform curriculum, which is the *design
and operations* track. Several exercises reference companion
implementations in `ai-infra-engineer-solutions` — specifically
`mod-108 ex-04` for the dashboards-as-code implementation that backs
Exercise 01, `mod-108 ex-08` for the Sloth SLO spec and multi-window
alert set that backs Exercise 02, the broader `mod-108` monitoring
module for the drift pipeline that backs Exercise 03, `mod-106 ex-12`
for the per-model cost attribution that backs Exercise 04, and
`mod-108 ex-09` for the runbook library that backs Exercise 05.
Those are the hands-on engineer-track templates; this module leans on
them rather than reproducing them. If you want a deeper implementation
pass on any exercise, start from the companion and then come back to
the dashboard-hierarchy defense (01), the burn-rate threshold defense
(02), and the runbook triage-tree discipline (05), which are the
deliverables the engineer track does not cover.
<!-- needs-research: confirm final naming / exercise numbering of mod-108 companion in ai-infra-engineer-solutions before publishing -->

## Navigation

- Up: [`../../README.md`](../../README.md) — repo overview
- Curriculum spec: [`../../CURRICULUM.md`](../../CURRICULUM.md) — see
  "Module 08: Observability & Monitoring"
- Previous: [`../mod-007-developer-experience/`](../mod-007-developer-experience/) — Developer Experience & Tooling
- Next: [`../mod-009-security-governance/`](../mod-009-security-governance/) — Security & Governance

Observability is the module where most ML platforms either make
incidents a solved problem with a dashboard, an alert, and a runbook —
or discover, under incident conditions, that the signals they needed
were never emitted, the alert that fired was the one that pages on
noise, and the runbook terminates in "page the model owner." Instrument
the ML-specific signals, build the four-row dashboard, write the
multi-window burn-rate alerts, run the drift and cost pipelines,
author the runbooks, and the rest of the platform has an operational
story it can defend at 3am.
