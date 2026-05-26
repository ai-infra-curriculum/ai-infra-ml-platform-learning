# Lecture 02: Platform Thinking

## Table of Contents

1. [Introduction](#introduction)
2. [Abstraction Design: The Core Skill](#abstraction-design-the-core-skill)
3. [Levels of Abstraction](#levels-of-abstraction)
4. [Self-Service vs Full-Service](#self-service-vs-full-service)
5. [Developer Experience (DX) Fundamentals](#developer-experience-dx-fundamentals)
6. [Platform Adoption Strategies](#platform-adoption-strategies)
7. [The Rule of Three (and Other Heuristics)](#the-rule-of-three-and-other-heuristics)
8. [Measuring Platform Success](#measuring-platform-success)
9. [Common Cognitive Traps](#common-cognitive-traps)
10. [Summary](#summary)

---

## Introduction

Chapter 01 was about *what a platform is*. This chapter is about *how a platform engineer thinks*. The work of choosing abstractions, drawing service boundaries, deciding what to expose and what to hide, balancing self-service against operator burden — this is the core craft of platform engineering and the part that separates "we wrote some scripts" from "we shipped a platform."

You can build a working platform without explicit training in these skills. You will build a *better* platform if you have a vocabulary for the choices you're making. The aim of this chapter is to give you that vocabulary.

By the end you should be able to:

- Name at least three "levels of abstraction" common to platform design and identify which one a given service is at.
- Articulate the difference between self-service and full-service modes, and identify which one fits a given workflow.
- Apply at least five developer-experience heuristics when reviewing a platform feature's UX.
- Sketch a platform adoption strategy from "shadow IT" to "85% adoption" with named phases.
- Recognize at least three common cognitive traps platform engineers fall into.

The lens we will keep returning to is: *the platform is a product, its users are engineers, and the product manager is you*.

---

## Abstraction Design: The Core Skill

### What is an abstraction?

An **abstraction** is a representation of a system that hides some details and exposes others, chosen to make a specific class of user productive. A REST API is an abstraction over a database query. A Kubernetes Deployment is an abstraction over a process supervisor and a placement scheduler. A Python `dict` is an abstraction over a hash table.

A *good* abstraction:

- **Hides what the user doesn't need to know.** Hiding is the whole point. If your abstraction requires the user to understand the implementation to use it correctly, it has failed.
- **Exposes what the user does need to control.** Don't hide so much that the user is helpless. A "submit ML training job" abstraction must let the user specify which dataset, which code, which hyperparameters, *and* let them retrieve the result.
- **Leaks predictably.** All abstractions leak. (Joel Spolsky's "Law of Leaky Abstractions.") A good abstraction leaks in *predictable, debuggable* ways. When the user trips over an implementation detail, they should be able to understand what happened and recover. The worst abstractions leak in mysterious ways that require the platform team to debug for the user.
- **Composes with other abstractions.** Real workflows combine multiple abstractions. The training-job abstraction must work with the dataset abstraction, the credentials abstraction, the artifact-store abstraction. If your abstractions only work in isolation, you don't have a platform — you have a museum of clever ideas.
- **Survives evolution.** Today's abstraction will need to grow. The right abstractions can be extended; the wrong ones break their consumers when extended.

These five criteria are easy to state and hard to satisfy.

### A worked example: the "training job" abstraction

Consider what is hidden in the platform that exposes a `submit_training_job(...)` SDK call. Behind that call:

- The platform must find capacity (which Kubernetes cluster? which node pool? what about GPU constraints?).
- The platform must materialize the user's code somewhere (a Docker image? a pip-installable source distribution? a Git ref?).
- The platform must mount the user's dataset (S3 path? mounted volume? streaming reader?).
- The platform must set up credentials (which IAM role? scoped to which buckets?).
- The platform must capture stdout/stderr, logs, metrics, artifacts.
- The platform must enforce quotas (this user has a budget; is this job within it?).
- The platform must handle preemption (if a higher-priority job comes in, can we preempt this one?).
- The platform must report status (running, succeeded, failed, killed).

A *good* `submit_training_job` abstraction hides all of this *except* what the user needs:

- which code to run (a function reference or a script path);
- which dataset to use (a dataset reference, not an S3 path with credentials);
- what resources to ask for (GPUs and memory, with sensible defaults);
- where to put the result (an artifact reference, with sensible defaults).

The user does *not* need to know which cluster, which IAM role, which node pool, which Helm release of which controller. If they ever need to know — to debug an unusual failure — the platform's logs and observability surface should reveal it. That's "predictable leakage."

A *bad* `submit_training_job` abstraction:

- Asks the user to specify the namespace, the service account, the image pull secret, the resource limits, and the tolerations. (This is "Kubernetes passthrough" — see chapter 01 anti-patterns.)
- Or, the opposite extreme: gives the user no way to specify GPU count, dataset, or output destination, and just runs "a job" on their behalf. The user has no control.

The middle is the art.

### Abstractions vs configuration

A common temptation, especially early in a platform's life, is to expose every internal knob as a configuration option. "Just let the user override it if they need to" feels democratic; in practice, it pushes complexity onto every user. Every config option is a maintenance liability and a thing the user has to learn.

A useful heuristic: **default first, expose later**. Ship the platform with strong opinions and minimal configuration. Add a configuration option only when at least two real users (not hypothetical ones) need to override it.

Compare:

```python
# Bad: 47 parameters, all required
submit_training_job(
    namespace="ml-platform",
    service_account="training-sa",
    image="registry.internal/training:v2.3.1",
    cpu_request="4",
    cpu_limit="8",
    memory_request="8Gi",
    memory_limit="16Gi",
    gpu_type="a100",
    gpu_count=1,
    tolerations=["gpu-node=true:NoSchedule"],
    node_selector={"gpu-pool": "training"},
    ... # 35 more parameters
    code_path="./train.py",
)

# Better: 3 parameters required, sensible defaults for everything else
submit_training_job(
    code_path="./train.py",
    dataset="warehouse://sales/q3_2025",
    gpus=1,  # default
)
```

The second API does the same job for 95% of users. The 5% with unusual needs can use kwargs or a more advanced API. This is good abstraction design.

### Naming matters

The name of an abstraction shapes how users think about it. `Job` is more general than `TrainingJob` which is more specific than `XGBoostTrainingJob`. Choosing the right level of generality in your naming determines what users will and won't try to put into the abstraction.

A useful test: when you tell a new user the name of your abstraction, do they correctly guess what it does? If yes, you've named it well. If no, rename it.

Bad: `Resource`, `Object`, `Thing`, `Entity` — these reveal that you don't actually know what the abstraction is for. If you can't name it specifically, you haven't designed it specifically.

Good: `TrainingRun`, `FeatureView`, `ModelEndpoint`, `DataSnapshot` — these are specific enough that the user knows what they're getting.

We will see this play out throughout the curriculum. Some of the most heated design discussions on real platforms are arguments over names. They are not bikeshedding; they are deciding what the platform *means*.

---

## Levels of Abstraction

Platforms are stacked: low-level primitives, mid-level capabilities, high-level workflows. A good platform engineer can move between these levels and identify which one a given feature is at.

### A useful three-level breakdown

| Level | Example (general) | Example (ML platform) | Typical user |
| --- | --- | --- | --- |
| **L1: primitives** | filesystem, process | container, pod, IAM role | platform engineers |
| **L2: capabilities** | "deploy a web service" | "run a training job", "register a model" | ML engineers |
| **L3: workflows** | "scaffold a new microservice" | "ship a new model end-to-end" | data scientists |

L1 is the underlying infrastructure. Users typically should not need to touch L1 directly.

L2 is the verbs the platform supports. A platform with rich L2 surface lets engineers compose those verbs into their own workflows.

L3 is the *opinionated, end-to-end* workflows. They use L2 verbs internally. L3 is what a data scientist actually wants — "I have a notebook; I want a deployed model" — but L3 is hard to do well because it requires opinions about the whole workflow.

A platform that lives only at L1 is "Kubernetes with documentation." A platform that lives only at L2 is "a constellation of APIs." A platform that lives only at L3 is rigid and breaks the first time a use case deviates. A *healthy* platform offers all three:

- **L1 for the platform team itself**, to build the rest on.
- **L2 for advanced users** who want to compose primitives.
- **L3 for typical users** who want the paved road.

### When you find yourself building at the wrong level

A common failure mode is to build L3 first (because that's what users ask for) and discover later that you have no L2 underneath. Adding new L3 workflows requires you to re-implement the same logic in each workflow because there's no shared L2 verb to call.

Another failure mode is to build L2 only and assume users will compose. Most users won't; they'll ask for L3 anyway. Your L2 will languish.

The right order, when bootstrapping a platform, is usually:

1. Identify the most-common L3 workflow that solves the most user pain.
2. Build it *as if* it were just a script — wire-throughed but functional.
3. Refactor it into L2 verbs once a *second* L3 workflow needs them.
4. Repeat.

This pattern is sometimes called "extract from the third use case." Don't extract until you have three concrete consumers.

### Compositional thinking

Once L2 verbs exist, advanced users can compose them. Metaflow is a good example: a `FlowSpec` is an L3 workflow, but `@step`, `@batch`, `@conda`, `@retry` are L2 verbs that the user composes inside the flow. The platform team supplies the verbs; the user composes them; the result is a workflow.

Plan for composition from the start. Even if you only have one user, design your L2 verbs as if you had ten — that way, when you have ten, you're not re-architecting.

---

## Self-Service vs Full-Service

### The two modes

Every platform feature exists somewhere on a spectrum from **fully self-service** (the user does everything themselves through the platform UI/API) to **fully full-service** (the user files a ticket, a platform engineer does the work). The spectrum looks like this:

```
Fully ticketed                                      Fully automated
[--------|---------|---------|---------|---------|---------|--------]
   user      user       user        platform-       gated      fully
  files     files       fills        approved      self-      self-
  Slack    JIRA       form;          self-         service     service
  msg     ticket      manual         service       w/ async
                       fulfill                     approval
```

Each point on this spectrum has a place. Even mature platforms have some fully-ticketed paths (e.g., requesting a new GPU node pool, which involves cloud-account changes a regular user shouldn't make).

### When self-service is right

- The action is **safe** — bounded blast radius, reversible.
- The action is **frequent** — many users, many times per day.
- The action's correctness is **automatable** — you can validate inputs without human judgment.

Example: submitting a training job. Frequent, safe (each tenant has resource quotas), automatable. Self-service is the right answer.

### When full-service is right

- The action is **rare**.
- The action has **broad blast radius** (e.g., affecting other tenants).
- The action requires **judgment** that's hard to encode.
- The action involves **trust** decisions (granting elevated access).

Example: granting a tenant access to a sensitive dataset. Rare, judgment-required, trust-laden. Full-service (i.e., a request goes to a data-owner approver) is the right answer.

### When *gated* self-service is right (the middle)

Many actions live in the middle. A common pattern:

- The user requests the action via the platform UI/API.
- The platform validates the request automatically.
- If the request is "obviously safe" (matches a pattern, within quotas), it goes through automatically.
- If the request is "needs review," it queues for an approver who can click-to-approve.

Example: provisioning a new namespace for a team. The action is safe enough to automate, but the platform team wants visibility. Gated self-service: the user fills out a form, a platform engineer click-approves, the namespace is created.

### The "what fraction of requests are self-service?" metric

A useful platform health metric: **percentage of user actions that completed without a human in the loop**. This metric should trend up over time. If it's trending down, your platform is regressing into a ticketing service.

Some actions will *never* go fully self-service (the rare, judgment-laden ones). That's fine. But the fraction that *can* be self-service should be self-service.

This metric is also a forecasting tool. If you know your weekly self-service rate, you can estimate the platform team's headcount needs: the team scales with the non-self-service fraction, not with the user population.

### A common failure mode

A team builds a "platform" that requires every action to be ticketed. They are then overwhelmed by tickets. They hire more platform engineers to keep up with tickets. The tickets keep coming. Eventually they realize they should automate the tickets. By that point they have a much larger team than they would have needed if they'd designed for self-service from the start.

The lesson: **automate the ticketing system *before* it grows**. Once a workflow exists as a ticket queue, it accumulates expectations that are hard to dislodge.

---

## Developer Experience (DX) Fundamentals

DX is the discipline of making your platform pleasant to use. The DX of a platform determines whether users adopt it (and stay) or route around it.

### The "time to first value" principle

The single most important DX metric for an internal platform is **time to first value**: how long from a user landing on your docs to a working end-to-end run of the platform.

If TTFV is 10 minutes, your platform will be adopted. If TTFV is two weeks, it won't.

This means:

- The "Getting Started" guide must work end-to-end on the first try. Test it weekly. Pay someone outside the platform team to follow it.
- There must be a "hello world" workflow that proves the platform works without requiring any real data, real credentials, or real configuration.
- Error messages must be actionable. "Connection refused" is not actionable; "Service X is unreachable; check that you have run `platform-cli login`" is.

### The "principle of least astonishment"

The platform should behave the way a user reasonably expects, given the surface they see. Surprises are costly.

Examples of astonishment:

- A user calls `submit_training_job(...)` and the function returns successfully. The user thinks the job is running. In fact, the function just enqueued the job, which won't start for an hour. (Fix: return a job handle that exposes status. Don't make the user discover this from the docs.)
- A user re-runs the same flow twice. The first run took 5 minutes; the second took 30. The user has no idea why. (Fix: surface cache hits/misses. Tell the user what was reused and what was recomputed.)
- A user updates their requirements file and pushes. The platform silently keeps using the old image. (Fix: detect changes; either rebuild or fail with a clear error.)

Every surprise is an opportunity to improve DX. Track surprises through support channels and fix them.

### Errors are part of the product

Most platforms put 95% of design effort into the happy path. This is upside-down. **The error path is where users live when they need the platform most.** Treat it as a first-class part of the product.

Bad error message: `KeyError: 'model_name'`

Better error message:
```
Model submission failed: 'model_name' is required.

  Got:        {"version": "1.0"}
  Expected:   {"model_name": str, "version": str}

  Example:    submit_model(model_name="churn-predictor", version="1.0")
  Docs:       https://internal-docs/platform/model-registry#submit
```

The better error tells the user *what* went wrong, *what they sent*, *what the platform expected*, an *example* of the right call, and a *link to docs*. It costs the platform team three lines of code and saves the user 20 minutes of digging.

A useful test: read a year's worth of your platform's user-support questions. For each question, ask: "could a better error message have answered this without the user needing to ask?" If the answer is yes (it usually is), prioritize fixing those error messages.

### Documentation as a feature

Documentation is not a deliverable separate from the platform; it is *part of* the platform.

Practical rules:

- Every user-facing API has docs co-located with the code (docstrings + a generated reference site).
- Every user-facing workflow has a tutorial that takes the user from zero to working.
- Every concept (model, run, feature, deployment) has a glossary entry.
- Every error code has a docs page that explains causes and fixes.
- Docs are versioned alongside the platform. Old docs for old versions remain accessible.

Documentation that is "out of date" is documentation that is being maintained badly. Build doc-freshness into the platform's CI: link-check, example-execution, etc.

### CLI / SDK / web UI — and the order to build them

Most ML platforms eventually have three user surfaces: a CLI, a Python SDK, and a web UI. The order to build them, in our experience:

1. **CLI first.** It's the fastest to build and the easiest to demo. It's also pleasant for power users.
2. **SDK second.** Once the CLI is solid, build a Python SDK on top of the same underlying APIs. The SDK is where data scientists live, and they will use it heavily once it exists.
3. **Web UI third.** A web UI is a big undertaking. Build it once the platform has stable APIs and a clear set of read-only views to surface (runs, models, dashboards).

A common mistake is to start with a web UI. Web UIs are visible (good for buy-in) but expensive to build and slow to evolve. Most data scientists prefer Python to web forms anyway. CLI and SDK first; UI when stable.

### The doc-is-the-product principle, restated

If your platform's documentation reads like a generated API reference, your platform feels like a bag of APIs. If it reads like a guided tour through opinionated workflows, your platform feels like a product. Aim for the latter.

---

## Platform Adoption Strategies

Platforms don't get adopted by being announced. They get adopted by a deliberate, multi-quarter campaign of helping users move from where they are to where the platform wants them to be. This is part of the platform engineer's job, even though it doesn't look like engineering.

### Phases of adoption

A useful mental model: most platform rollouts go through four phases.

**Phase 0: Pre-existence.** No platform; every team rolls its own. The platform team doesn't exist yet, or exists as a couple of engineers writing the first version.

**Phase 1: First adopters.** The platform exists in a usable form. One or two enthusiastic teams adopt it, often working closely with the platform team. The platform shape changes a lot in response to their feedback.

**Phase 2: Spreading.** The platform reaches 5-10 teams. It is "the way we do things" for several use cases but not yet the default. There is still significant shadow IT.

**Phase 3: Default.** The platform is the default. New teams use it without question. Legacy shadow IT is being deprecated or migrated. The platform team is shifting from "build" mode to "operate and improve" mode.

The transitions between phases are *not automatic*. Each one requires deliberate work.

### From Phase 0 to Phase 1: find a "design partner"

Don't try to launch the platform to everyone at once. Find one team that:

- Has real, painful pain (so they're motivated to try something new).
- Has bandwidth to work with you (so the feedback loop is fast).
- Has executive air cover (so they won't be punished if things break).

Work with this team in close partnership for 1-2 quarters. Their use case shapes the platform's first iteration. Their endorsement opens the door to Phase 2.

### From Phase 1 to Phase 2: institutionalize the success story

Once one team is happy, generalize. Specifically:

- Document the design-partner team's journey.
- Run lunch-and-learns where the design partner team presents to the wider organization.
- Solicit a small batch (3-5) of new adopter teams with diverse use cases.
- Be picky: only onboard teams that fit the platform's current capabilities. Don't onboard teams whose use cases stretch the platform too far at this stage; they will be frustrated, and word will spread that the platform "doesn't work."

### From Phase 2 to Phase 3: deprecate shadow IT

This is the hardest transition because it requires political work. You must:

- Build the case that shadow IT is *worse* than the platform. Data: how much engineering time was lost to the shadow IT path in the last quarter? How many production incidents traced to the shadow IT path?
- Get executive sponsorship for the migration. "All new use cases must use the platform" is an executive decision, not a platform-team decision.
- Provide migration tooling and migration partner support. If you ask each team to migrate alone, most won't.
- Set a *deprecation date* for the shadow-IT path. Without a date, migration never happens.
- Be willing to keep the shadow-IT path on life support for a long tail of edge cases that aren't worth migrating.

### The "carrots vs sticks" tension

Most platform adoption is driven by *carrots* in Phase 1 (the platform is genuinely better) and a *mix* in Phases 2-3 (carrots for new use cases, sticks for migrating legacy ones).

Sticks (mandates, deprecation dates, policy enforcement) are corrosive when used too early. If the platform isn't yet better than shadow IT, mandating it breeds resentment and shadow-shadow-IT. Use sticks only after Phase 2.

Carrots (better DX, less work for the user, faster iteration) are what get you to Phase 2.

### The metric: adoption rate

Track adoption as **percentage of relevant workloads running on the platform**. "Relevant" is doing work — define which workloads are in scope. (Toy experiments by individual data scientists are probably not in scope; production-ish workloads are.)

A typical maturity curve:

- Phase 1: 5-15% adoption.
- Phase 2: 25-60% adoption.
- Phase 3: 75-95% adoption.

Going past 95% requires absorbing all of the long tail, which often isn't worth it.

---

## The Rule of Three (and Other Heuristics)

Platform engineers carry a small toolkit of heuristics that help with day-to-day design decisions. These are not laws; they are guidelines. We list the most useful ones below.

### Rule of three

> Don't extract a shared abstraction until you have three real concrete consumers for it.

Variants: "rule of three duplication" (don't refactor duplicate code until it appears three times), "don't generalize until you have three examples."

The point is that a single example is too little to know what the right shape is. Two examples might both be coincidentally similar. Three examples is enough signal to extract a real pattern.

This rule is the antidote to premature platformization.

### Postel's law (the robustness principle)

> Be conservative in what you do; be liberal in what you accept from others.

For a platform: be strict about what *you* send (clean, well-formed, documented), but tolerant of what *users* send (accept reasonable variations, give clear errors on truly broken inputs). This lowers the friction for users while keeping your contracts crisp.

A caveat: Postel's law has gotten some pushback in security contexts ("tolerant parsing creates ambiguity that attackers exploit"). For platforms, use judgment: be tolerant where it helps the user without creating ambiguity, strict where ambiguity is dangerous.

### "Make the easy things easy and the hard things possible"

A platform should make the common case trivial. It should also leave an escape hatch for the unusual case, even if that escape hatch is verbose. A platform with no escape hatch for the hard case becomes the source of forks and workarounds.

Practical implication: every L3 workflow should compose underlying L2 verbs that users can also call directly. Users hitting the limits of the L3 workflow can drop down to L2.

### "The first version is wrong"

Whatever you ship in v1 will be wrong in some way you can't foresee. Plan for v2.

Implications:

- Version everything from day one. URLs, schemas, manifests, SDK names. (We dive deep into versioning in chapter 04.)
- Don't promise long-term backward compatibility before you have real users; you don't know yet what shape the compatibility needs to be.
- *Do* communicate clearly when something changes.

### "Owned interfaces beat shared databases"

A pattern from the microservices world that applies to platforms too: when two services need to interact, they should do so through an *owned interface* (e.g., a REST API or gRPC) rather than through a *shared database* (where they both read/write the same tables).

Shared databases are tempting because they're easy. They lock you into the schema. Owned interfaces are harder to set up but evolve more cleanly.

For an ML platform: never let users access your underlying databases directly. Always front them with APIs. The first time someone asks "can I just query the metadata DB directly?" the answer is no, and you give them an API to use instead.

### "Internal platforms are still platforms"

Sometimes platform teams convince themselves that because the users are inside the company, the bar for DX is lower. This is wrong. Internal users have less patience than external ones — they have other ways to get the work done (shadow IT), and they have political weight if the platform is bad.

Hold yourself to the same DX standards you'd expect from a paid SaaS.

---

## Measuring Platform Success

You cannot manage what you cannot measure. Here are the metrics platform teams actually use.

### Adoption metrics

- **Active users** (per week / per month): people who used the platform at least once in the period.
- **Active workloads**: jobs, deployments, runs counted in the period.
- **Adoption rate**: active workloads / (active + non-platform) workloads = % of the universe on the platform.
- **First-time-success rate**: fraction of new users whose first attempt with the platform succeeded.

### Operational metrics

- **Job success rate**: fraction of platform-submitted jobs that completed successfully (not failed due to platform issues).
- **Platform-caused failure rate**: fraction of failed jobs whose failure was caused by the platform itself (not the user's code).
- **Time to recover from platform outage**: how long does it take to detect, root-cause, and fix platform incidents?

### User-experience metrics

- **Time to first value**: time from "user signs up" to "user's first successful end-to-end run."
- **NPS-style satisfaction**: periodic surveys.
- **Support burden**: number of support questions per active user (lower is better).
- **Self-service ratio**: fraction of user actions completed without human intervention.

### Business / cost metrics

- **Cost per workload**: total platform cost (cloud + people) / number of active workloads.
- **Cost amortization vs DIY**: estimated cost of each tenant if they did it themselves vs cost on the platform.
- **Platform team headcount vs user population**: ratio of platform engineers to users served (lower = better leverage).

### Avoid vanity metrics

"Lines of platform code" is not a metric. "Number of features shipped" is not a metric. These are *activity* indicators, not *outcome* indicators. A platform team that ships 50 features and has 5% adoption is failing; one that ships 5 features and has 80% adoption is succeeding.

### A small dashboard

A useful starting dashboard for a Phase 1 → Phase 2 platform:

| Metric | Goal | This week | Last week | Trend |
| --- | --- | --- | --- | --- |
| Active users | grow | 23 | 19 | up |
| Adoption rate | grow | 18% | 16% | up |
| First-time success rate | ≥ 80% | 76% | 81% | down (investigate) |
| Job success rate | ≥ 95% | 96.2% | 95.1% | up |
| Platform-caused failure rate | ≤ 1% | 0.4% | 0.6% | down |
| Self-service ratio | grow | 87% | 84% | up |
| Time to first value (p50) | ≤ 1 day | 6h | 8h | down |
| Support tickets/active-user/week | ≤ 0.5 | 0.6 | 0.7 | down |

Update weekly. Surface the trend column. Investigate any metric that's regressing.

---

## Common Cognitive Traps

Beyond technical anti-patterns (chapter 01), there are cognitive traps that platform engineers fall into. Naming them helps.

### Trap 1: "Building the platform we want, not the one users need"

Symptom: the platform team builds elegant abstractions for problems that excite the platform team. Users find those abstractions baffling and route around them.

Mitigation: *do real user research*. Sit with users while they use the platform. Watch them get stuck. Ask what they were trying to do and where the platform failed them. Do this monthly, not annually.

### Trap 2: "Solving for hypothetical users"

Symptom: the platform supports 17 use cases. Only 2 actually exist. The other 15 are imagined.

Mitigation: the rule of three. Don't build for use cases that don't have a real, named consumer.

### Trap 3: "Confusing complexity with sophistication"

Symptom: the platform has a 47-step deployment workflow with 11 controllers. The team is proud of the architecture diagram.

Mitigation: complexity is debt. Sophistication is the *judicious application of complexity to a real need*. If you can't draw a line from each component to a user-visible benefit, the component is debt.

### Trap 4: "Confusing engineering taste with user taste"

Symptom: the platform team prefers gRPC; users prefer REST. The team ships gRPC. Users complain. The team explains why gRPC is better. Users continue to complain.

Mitigation: users get to pick what feels good to them. Engineering taste is real but it loses to user adoption.

### Trap 5: "Postponing the people work"

Symptom: the platform exists but adoption is stuck. The team keeps shipping features instead of doing the harder work of user research, evangelism, docs, training, mentorship.

Mitigation: adoption is the *outcome* of platform work; building features is just the *means*. Track adoption. When adoption is stuck, switch from building features to people work.

### Trap 6: "Forgetting that the platform also has a CFO"

Symptom: the platform consumes cloud resources and engineering headcount without anyone tracking the unit economics.

Mitigation: maintain a basic cost model. Know your cost per workload, your cost per active user, your platform team's burn rate. Be ready to defend the math.

### Trap 7: "Treating the platform as eternal"

Symptom: the platform team treats the platform as a permanent fixture. It accumulates features no one uses but no one removes.

Mitigation: deprecation discipline. Every quarter, identify the bottom 10% of features by usage and ask "should this stay?" Many should be removed. Removing features is a sign of platform health, not weakness.

---

## Summary

- **Abstraction design** is the core skill. Good abstractions hide what users don't need, expose what they do, leak predictably, compose with others, and survive evolution.
- Platforms live at **three levels**: primitives (L1), capabilities (L2), workflows (L3). A healthy platform offers all three; an unhealthy one specializes too early.
- The **self-service vs full-service spectrum** is a design choice for each feature, not a stance for the whole platform. Track self-service ratio as a health metric.
- **Developer experience** is the platform's product. Time to first value is the most important metric. Error messages and documentation are not afterthoughts; they are the product.
- **Adoption** moves through four phases: pre-existence, first adopters, spreading, default. Each transition requires deliberate work. Carrots first, sticks later.
- **Heuristics** like the rule of three, Postel's law, "easy things easy, hard things possible," and "owned interfaces beat shared databases" carry surprising weight in day-to-day design.
- **Measure what matters**: adoption, operational health, user experience, unit economics. Avoid vanity metrics. Investigate regressions weekly.
- **Cognitive traps** — building for ourselves not users, hypothetical-user solving, confusing complexity with sophistication, engineering taste vs user taste, postponing people work, ignoring the CFO, treating the platform as eternal — are the recurring human failure modes. Name them when you see them.

Next chapter: multi-tenancy, which is where most of the *operational* hardness of platforms lives.

---

## Exercises and Reflections

These reflection questions are not part of the formal exercises (those live in `exercises/`) but are worth pondering before moving on.

1. Pick a platform you have personally used (Heroku, Vercel, AWS, an internal one at a past job). At which of the three levels (L1, L2, L3) is the user-facing surface? What did you like about it? What was friction?
2. Estimate the **time to first value** of the last new internal tool you onboarded onto. Was it under 10 minutes, 1 hour, 1 day, or 1 week? What was the bottleneck?
3. Of the seven anti-patterns from chapter 01 and the seven cognitive traps from this chapter, which two scare you most as a platform-engineer-in-training? Why?
4. Imagine you have to make the platform you'd build at your current organization (real or hypothetical). What's the most important abstraction you'd start with? Defend the name.

There are no right answers; the questions are for self-calibration.

---

## Further Reading

- **["Platform as a Product"](https://martinfowler.com/articles/talk-summaries/platform-as-a-product.html) (Camille Fournier, summary by Martin Fowler).** The "platform-as-product" frame, articulated by one of its most prominent advocates.
- **["The Twelve-Factor App"](https://12factor.net/).** Heroku-era. The most concentrated set of opinions about how applications should behave to work well on a platform. Many of these opinions are still right.
- **["Building Evolutionary Architectures"](https://www.oreilly.com/library/view/building-evolutionary-architectures/9781492097532/) by Ford, Parsons, and Kua.** On designing systems that can change. Relevant to abstraction longevity.
- **["A Philosophy of Software Design"](https://web.stanford.edu/~ouster/cgi-bin/book.php) by John Ousterhout.** Short, dense, opinionated book on abstraction design. The chapter "Modules Should Be Deep" is the single best thing you can read on the topic.
- **["Trillion Dollar Coach"](https://www.trilliondollarcoach.com/) by Schmidt, Rosenberg, and Eagle.** Not a platform book, but the chapter on coaching engineering managers applies almost word-for-word to platform-team-leads. Worth a skim.

In the next chapter, we tackle multi-tenancy — the architectural and operational discipline of running many tenants on one platform without them stepping on each other.
