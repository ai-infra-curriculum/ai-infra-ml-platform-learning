# Lecture 01: Introduction to ML Platform Engineering

## Table of Contents

1. [Introduction](#introduction)
2. [What Is a Platform?](#what-is-a-platform)
3. [Platform vs Infrastructure](#platform-vs-infrastructure)
4. [What Makes an ML Platform Specifically "ML"](#what-makes-an-ml-platform-specifically-ml)
5. [The ML Platform Engineer's Job](#the-ml-platform-engineers-job)
6. [The Platform Value Proposition](#the-platform-value-proposition)
7. [Case Study: Uber Michelangelo](#case-study-uber-michelangelo)
8. [Case Study: Netflix Metaflow](#case-study-netflix-metaflow)
9. [Anti-Patterns and Failure Modes](#anti-patterns-and-failure-modes)
10. [Summary](#summary)
11. [Further Reading](#further-reading)

---

## Introduction

You can build a single machine learning model with a notebook and a CSV file. Most data scientists do.

You cannot build *a hundred* machine learning models — used by different teams, against different data sources, on different cadences, with different SLAs — with a hundred notebooks. The notebook approach degrades nonlinearly: at two notebooks you have a side project, at twenty you have a confused team, at two hundred you have a crisis. Each notebook is its own snowflake: its own dependency pin, its own data preprocessing, its own training-inference skew, its own quiet path to production.

The job of an ML platform is to absorb that nonlinear cost growth. Instead of N teams each solving the same problems (where do my features come from? how do I version a model? how do I deploy to production? how do I roll back?) badly, you build *one* good solution that all N teams can reuse. The platform engineer is the person who builds and runs that one good solution.

This lecture defines what platform engineering means in an ML context, distinguishes it from adjacent disciplines (data engineering, MLOps, application platform engineering), and grounds the discussion in two well-documented industry case studies: Uber's Michelangelo and Netflix's Metaflow.

By the end you should be able to:

- Define an ML platform without using the word "platform" in the definition.
- Distinguish platform work from infrastructure work and from MLOps "as a tooling pile."
- Name three forces that make ML platforms qualitatively different from general application platforms.
- Sketch the major components of Michelangelo or Metaflow from memory.
- Recognize and name at least three platform anti-patterns.

This is the highest-altitude lecture in the module. Subsequent chapters will zoom into specific decisions — abstractions, multi-tenancy, APIs, architecture — but everything depends on the framing here.

---

## What Is a Platform?

### A definition we will use throughout the curriculum

> **A platform is a reusable, opinionated, self-service substrate that lets internal teams accomplish a class of tasks without re-inventing the supporting machinery each time.**

Let's break that definition apart, because every adjective is doing work.

- **Reusable.** The same platform serves multiple consumers. If only one team uses it, it is bespoke tooling, not a platform.
- **Opinionated.** The platform encodes a point of view about how to do the work. It does not offer infinite flexibility. "Use this serialization format. Run training as this kind of job. Use these tags." Opinionatedness is what makes platforms valuable; without it, you have an API gateway in front of raw infrastructure.
- **Self-service.** Consumers can use the platform without filing tickets with the platform team. If every action requires a human in the platform team's queue, it is a service, not a platform. (Counterexample: many "platforms" in real organizations are 80% self-service and 20% ticketed, and that's fine, but the goal is to push the ratio toward 100% self-service.)
- **Substrate.** The platform is *underneath* the work — it isn't the work itself. The platform doesn't ship features; the products built on top of it do.
- **Class of tasks.** A platform serves a category of similar workflows. "Train and serve ML models" is a class. "Deploy any program of any kind for any purpose" is too broad to be a platform — that's an operating system. "Recompute customer churn predictions for the EU market" is too narrow — that's an application.

This definition is intentionally narrow. There are platforms-of-platforms (Google's internal Borg, AWS) and there are platforms-of-applications (Shopify, Stripe), but for our purposes we are interested in *internal engineering platforms* that serve other engineers within the same organization.

### Two illustrative examples (from outside ML)

**Heroku** (circa 2010) is a paradigmatic platform. Developers write a small program in any of several supported languages, type `git push heroku main`, and the platform builds, deploys, and runs the program with sensible defaults for routing, logging, scaling, and persistence. Heroku is opinionated (it prefers 12-factor apps, stateless processes, ephemeral filesystems). It is self-service (no humans in the loop). It is reusable (millions of apps). It is a substrate (Heroku does not write your business logic).

**Backstage** (Spotify's developer portal, now an open source CNCF project) is another paradigmatic platform — but at a different layer. Backstage is *the place engineers go* to interact with their company's other platforms. It catalogs services, surfaces dashboards, lets you scaffold new repositories from templates. Backstage's opinion is about *how engineers discover and navigate their own organization*. It is not a deployment runtime — it is a developer-experience layer that aggregates other platforms.

Both fit the definition. Both have different *concerns*. Both fail in characteristically different ways. The "platform" label covers a wide range of altitudes.

### An ML-specific example

**MLflow Tracking** (the experiment-tracking component of MLflow) is a small but real example of a platform. It is reusable: many data scientists at many companies use it. It is opinionated: it models experiments as runs, runs have params and metrics, runs belong to experiments. It is self-service: you `import mlflow; mlflow.log_metric(...)` and the tracking server records the result without anyone in the platform team being involved. It is a substrate: it doesn't run your training code, it remembers what happened.

MLflow Tracking on its own is *not* a complete ML platform. It is a *component* in one. We will see in later chapters how components like this compose into a platform.

### What a platform is *not*

- **A platform is not a single API.** A REST API in front of S3 is not a platform. A platform tends to involve workflows, multiple services, opinionated defaults, and a developer experience layer.
- **A platform is not a tooling pile.** If your "platform" is a wiki page listing 12 disconnected tools that engineers must learn to coordinate, you have a tooling pile. The platform's job is to *integrate* the pieces so the user doesn't have to.
- **A platform is not a Kubernetes cluster.** Kubernetes is *infrastructure that platforms are built on*. Giving a data scientist a kubeconfig and saying "good luck" is not delivering a platform.

This last distinction — platform versus infrastructure — is so central that it gets its own section.

---

## Platform vs Infrastructure

### The boundary that confuses everyone

In most organizations, the platform team and the infrastructure team are different teams, but the boundary between them is fuzzy. A useful mental model:

- **Infrastructure** is *physical-ish*: compute, storage, network, the basic primitives. Whether it's a literal datacenter or a cloud account, infrastructure provides *raw capacity*. Its consumers are other engineers who know how to combine the raw primitives into a working system.
- **Platforms** are *opinionated-and-self-service*: they take raw infrastructure and add structure, abstractions, defaults, and a UX layer that lets non-experts use the underlying capacity correctly.

A useful test: if a new hire who has never used your stack before could be productive in a day using your tool, it's a platform. If they need a week of pair programming with a veteran to understand the kubeconfigs, the IAM policies, the queue names, and the secrets-management patterns, then what you have is infrastructure, possibly with some helper scripts. The helper scripts are not yet a platform — they're the *seeds* of one.

### A worked layering

For an ML system, the layering from bottom to top tends to look like this:

```
┌──────────────────────────────────────────────────────────┐
│  Application (the model that predicts customer churn)    │  ← business logic
├──────────────────────────────────────────────────────────┤
│  ML Platform                                             │
│  - feature store API                                     │
│  - training job submission API                           │
│  - model registry                                        │
│  - inference deployment workflow                         │  ← opinionated, self-service
│  - observability dashboards                              │
├──────────────────────────────────────────────────────────┤
│  Application Platform / Internal Developer Platform      │
│  - CI/CD, secrets, service catalog                       │  ← general-purpose, reusable
│  - logging/metrics/tracing pipeline                      │
├──────────────────────────────────────────────────────────┤
│  Infrastructure                                          │
│  - Kubernetes cluster(s), VPC, IAM,                      │
│  - object storage (S3/GCS), data warehouse,              │  ← raw capacity
│  - GPU node pools, batch schedulers                      │
└──────────────────────────────────────────────────────────┘
```

Each layer has its own audience. Infrastructure serves platform engineers and SREs. The general application platform serves all internal engineers. The ML platform serves ML engineers and data scientists. The application serves end users (or another internal service).

A common mistake — extremely common in early-stage ML teams — is to skip the ML platform layer and ask data scientists to talk directly to infrastructure. This works for one or two scientists and breaks down quickly thereafter.

### The "paved road" mental model

Many platform teams talk about "paved roads." A paved road is the *recommended way to do something* — it is supported, documented, opinionated, and self-service. Off the paved road, engineers can still get to the destination, but they're on their own for support.

An ML platform is, in effect, a set of paved roads:

- A paved road for "I want to train a model on a tabular dataset using GPU resources."
- A paved road for "I want to serve a real-time-inference HTTP API for an existing trained model."
- A paved road for "I want to backfill predictions for the last 90 days against the production model."

Most data scientists' work falls on the paved road. Edge cases (custom CUDA kernels, distributed training on 64 nodes, federated learning, on-device inference) may require leaving the paved road. That's fine, *as long as it's a conscious choice and the paved-road default works.*

A platform that has *only* paved roads and refuses to allow deviation will fail; a platform that has *no* paved roads and forces every team to build their own is not really a platform. The art is in the balance.

### A practical question

> When you're considering where to put a piece of functionality — in the platform or in the application — ask: *will more than one team need this?* If yes, it belongs in the platform. If no, leave it in the application until a second team asks for it. Premature platformization is a real and expensive failure mode.

We will return to this question, called the "rule of three" by some authors, in chapter 02.

---

## What Makes an ML Platform Specifically "ML"

If platforms in general are a known discipline, what does the "ML" prefix add? Quite a lot, as it turns out. ML platforms differ from general application platforms in at least five ways.

### 1. Data is a first-class citizen

A typical web service consumes some inputs from an HTTP request, runs some logic, and returns a response. The inputs are small, fresh, and not part of the service's own state.

An ML service consumes *features* — derived data that often comes from a separate data pipeline, may be hours or days old, and must match between training time and inference time. If the feature definition drifts between training and inference (training-inference skew), the model's predictions silently degrade in production.

This is unique to ML and forces the platform to provide a *feature store* (or some equivalent) — a system that ensures the same feature definition is used in training and serving. We discuss feature stores in depth in Module 06. For now: an ML platform has more *data plumbing* in its core than a general application platform does.

### 2. The artifact is not the source

In a typical service, you deploy *the code*. The code is the source of truth; you can rebuild the running service from git.

In an ML service, you deploy *a model artifact* — a binary blob of weights, often hundreds of megabytes to many gigabytes. The model is produced by running the *training pipeline*, which depends on:

- The training code (versioned in git, easy)
- The training data (versioned somewhere, harder)
- The hyperparameters (versioned somewhere, harder still)
- The library versions (deeply important, often un-versioned)
- The randomness seed (sometimes important)

Re-running the training pipeline does not, in general, produce a bit-identical model. So the platform has to treat *model artifacts* as first-class objects with their own lifecycle: registry, versioning, lineage back to training run, lineage back to data version.

This is the job of the *model registry* (e.g., MLflow's model registry, SageMaker Model Registry, Vertex AI Model Registry). We discuss model registries in Module 07.

### 3. Training is expensive and bursty

A typical web service runs at roughly constant cost. You provision N replicas; you pay for N replicas.

ML training is dramatically bursty. A given team might run no training jobs for three days and then need 64 A100 GPUs for 8 hours. This pattern is hostile to the standard auto-scaling assumptions of an application platform (which expects steady traffic with predictable peaks).

The platform has to provide:

- A *job queue* that buffers training requests when capacity is constrained.
- A *scheduler* that allocates GPUs (which are scarce and expensive) fairly across tenants.
- A *priority system* so urgent retraining doesn't starve behind speculative experiments.
- A *cost-attribution model* so the platform can answer "which team spent the most GPU-hours last month?"

We discuss training infrastructure in Modules 04 and 05.

### 4. Inference has strict latency and accuracy contracts

Serving an ML model for real-time inference is one of the harder serving problems. The model is large (so cold-start is slow), the request volume can be high (so you need horizontal scaling), and tail latency matters (so you need autoscaling that responds quickly).

On top of the operational latency concerns, models *drift*. The world the model was trained on diverges from the world it lives in. The platform must support:

- Canary deployments (route 1% of traffic to a new model, compare predictions).
- Shadow deployments (route 100% of traffic to old model, *also* to new model, compare predictions, don't use new model's output for anything).
- Rollback (when the new model is worse, get back to the old one fast).
- Monitoring of *accuracy* (not just latency and error rate).

We discuss model serving in Module 08.

### 5. The user is not a stereotypical software engineer

An ML platform serves data scientists, ML engineers, and applied researchers. These people:

- Live in Python and Jupyter, sometimes more than they live in the command line.
- Are sometimes (not always) less fluent in operational concerns like RBAC, Kubernetes manifests, CI/CD pipelines, network policies.
- Have a strong, legitimate preference for *iteration speed* — being able to change a hyperparameter, rerun training, and look at the loss curve in 10 minutes is a workflow much more valuable to them than to a typical backend engineer.

An ML platform that treats data scientists as backend engineers and demands they write Kustomize overlays will not be adopted. An ML platform that meets them in their Jupyter notebooks, with Python SDKs that hide the YAML, has a chance.

This shapes the developer-experience requirements of the platform in fundamental ways. We discuss platform DX in chapter 02.

### Summary of the five differences

| Concern | General app platform | ML platform |
| --- | --- | --- |
| What is deployed | Code | Model artifact + code |
| Reproducibility | git SHA | data version + code SHA + hyperparams + seed |
| Resource pattern | Steady, predictable | Bursty, expensive |
| Production health | Latency, error rate | Latency, error rate, **accuracy** |
| Typical user | Backend engineer | Data scientist / ML engineer |

These differences are why "use the application platform, add some Python" does not produce an adequate ML platform. The cross-cutting concerns are genuinely different.

---

## The ML Platform Engineer's Job

If you accept the framing above, then the ML platform engineer's job has roughly five facets. Different organizations weight these differently, and individual engineers specialize, but a healthy team covers all five.

### 1. Build and operate platform services

This is the most software-engineering-shaped part of the job: writing code for the platform's services (training-job submitter, registry, feature store integration, inference deployer). The platform team's services often look like a typical SaaS company's services — REST/gRPC APIs in front of databases, with their own SLOs, on-call, runbooks, and so on.

### 2. Curate and integrate third-party tools

A modern ML platform is not built from scratch — it is *assembled* from open-source and SaaS components. The team selects (e.g.) MLflow for the registry, Feast for the feature store, Argo Workflows for orchestration, KServe for serving, and writes the *integration glue* that makes them feel like one platform. The boring, deeply important work of "make MLflow's authentication match the rest of the company's IAM" lives here.

### 3. Define and uphold opinionated defaults

Platforms encode opinions. Someone has to make the opinions: "We use Parquet, not CSV. Models are stored as `.onnx`. Feature names are `snake_case`. Training jobs default to one GPU and must explicitly request more. Production deployments require a canary phase." These opinions are not free — they are the result of design conversations, RFCs, and learning from past failures.

### 4. Support, evangelize, and educate users

Once the platform exists, it must be *adopted*. Adoption is not automatic. The platform team writes documentation, gives lunch-and-learns, holds office hours, helps debug the inevitable confusing first contact between new users and the platform. A platform with 80% of its potential users still on shadow IT is failing, no matter how clean its code is.

### 5. Listen to users and evolve

Platforms are products. Like any product, they have a roadmap, user research, customer-success metrics, retention concerns. The platform team has to listen — through surveys, interviews, support-channel patterns, usage metrics — to where the platform is failing its users, and prioritize fixes accordingly.

A platform team that does (1) and (2) but not (3), (4), and (5) ends up building beautiful, unused tools. A platform team that does (3), (4), and (5) but not (1) and (2) ends up writing wiki pages with no underlying system.

### The "platform-as-product" mindset

It is worth saying outright: the most important shift for a new platform engineer to internalize is **platform-as-product**. The platform is a product. The users are internal engineers. The competition is "do it yourself with shadow IT" (which is often surprisingly fierce competition). The metrics are adoption, retention, time-to-first-value, support-ticket volume, NPS-style satisfaction.

If you came from a backend engineering background where success was measured by SLO compliance and feature throughput, the product-management aspects of platform work may feel foreign. Embrace them; they are the difference between a successful platform and a beautifully-engineered failure.

---

## The Platform Value Proposition

You will, repeatedly in your career, be asked to defend the existence of the platform team. The CFO wants to know why headcount keeps growing for "internal tools that don't ship features." The VP of Engineering wants to know why every product team can't just use the cloud directly. The new VP of AI wants to know if buying SageMaker would let them eliminate your team.

You need three answers, tailored to three audiences.

### To an ML engineer / data scientist (the user)

> "You don't want to learn Kubernetes, IAM, Terraform, and observability tooling just to ship a model. Use the platform; we have done that work for you. You will go from notebook to production deployment in a day, not a quarter."

This pitch is about *iteration speed* and *focus*. It is the most direct value proposition: the platform is a productivity multiplier for the people who use it.

A common failure mode is to pitch the platform on cost ("we save the company money"). Users do not care about company cost; they care about *their* iteration speed. If the platform slows them down by 20% even while saving 30% on cloud spend, they will rebel against it. Lead with the user value.

### To an SRE / production engineer (the partner)

> "Without the platform, every team will deploy ML models their own way, and you will be on call for all of them. With the platform, there is one well-instrumented path to production. When things break, you know how to debug them."

This pitch is about *operational sanity*. The SRE community is often a platform team's strongest internal ally, because they viscerally feel the cost of N teams each rolling their own. Use this.

### To a CFO / VP of Engineering (the funder)

> "We are amortizing the cost of solving N problems once instead of N times. If you don't have a platform, every ML team will spend 30-50% of their effort rebuilding the same wheels. With a platform, that work happens once. The platform team is N+1 engineers — and we save you many multiples of that in duplicated effort across all ML teams."

This pitch is about *unit economics*. You may need to back it up with numbers — actual numbers, derived from your own organization, not generic claims. We will discuss platform metrics later, but for now: keep a running ledger of "what would each ML team have to build if the platform didn't exist?" and you'll have your defense ready.

### When the pitch fails

There are organizations where the platform pitch *does not work*. The signs:

- The company has 1-2 ML teams and no plan to add more. (Platform amortizes across teams; no teams, no amortization.)
- ML is a side project, not a core part of the product. (Investment doesn't make sense.)
- The company has chosen a managed external platform (SageMaker, Vertex AI) and is committed to that path. (You may still need glue work, but not a homegrown platform.)

In these situations, the platform pitch will keep failing. That's information. Sometimes the right answer is "this org doesn't need a platform team yet" — and that is itself a valid platform-engineering insight to surface.

---

## Case Study: Uber Michelangelo

> Source: ["Meet Michelangelo: Uber's Machine Learning Platform"](https://www.uber.com/blog/michelangelo-machine-learning-platform/), Uber Engineering Blog, 2017. We reference this publicly-available writeup throughout the curriculum.

Uber's Michelangelo, first publicly described in 2017, is the canonical industry case study of an ML platform. Several things about it remain instructive for newer platform engineers.

### The motivation

By 2015, Uber had multiple teams running ML in production — uberPool's ETA model, surge pricing, eats demand forecasting — and each team was building its own pipeline from scratch. Training data was extracted in bespoke SQL queries. Models were trained in bespoke notebooks. Inference was deployed in bespoke services. There was no consistent way to ship a new feature, no consistent monitoring, and no consistent rollback.

Uber's response was to build an internal platform that took a model from data to production through a unified workflow. They named it Michelangelo.

### The components (as described publicly)

- **Manage data**: tools for assembling training datasets from Uber's data lake and warehouse, with feature pipelines that produced features for both training and serving.
- **Train models**: a job submission system that ran training on shared infrastructure, supporting multiple frameworks (XGBoost, TensorFlow, PyTorch).
- **Evaluate models**: tools for measuring model quality on held-out data and comparing candidate models to incumbents.
- **Deploy models**: a deployment workflow that pushed a trained model into a serving cluster.
- **Make predictions**: a real-time and batch inference service that exposed deployed models behind APIs.
- **Monitor predictions**: dashboards and alerts on prediction-quality drift, latency, and freshness.

Sketch from memory, on a whiteboard:

```
            ┌───────────────────┐
            │   Data Lake       │
            └─────────┬─────────┘
                      │
            ┌─────────▼─────────┐
            │ Feature Pipelines │
            └─────────┬─────────┘
                      │
                ┌─────▼─────┐
                │ Training  │  ←  job submission / resource allocation
                └─────┬─────┘
                      │
                ┌─────▼──────┐
                │  Registry  │  ←  versioned models
                └─────┬──────┘
                      │
                ┌─────▼──────┐
                │ Deployment │
                └─────┬──────┘
                      │
            ┌─────────▼─────────┐
            │ Inference Service │
            └─────────┬─────────┘
                      │
            ┌─────────▼─────────┐
            │     Monitoring    │
            └───────────────────┘
```

This shape — *features → training → registry → deployment → inference → monitoring* — has become the canonical "spine" of nearly every ML platform built since. When you look at MLflow + Feast + KServe + Evidently, you are looking at the open-source incarnation of this spine. When you look at SageMaker, you are looking at AWS's vertical-integrated version of it.

### What we can learn from Michelangelo

1. **Unified workflow, not unified code.** Michelangelo did not try to make all ML use cases use the same code. It tried to make them use the same *workflow*: every team submitted a training job, every team registered the resulting model, every team deployed through the same pipeline. The model code itself could vary.
2. **Multi-framework support is a value proposition.** By supporting XGBoost, TensorFlow, and PyTorch (and others over time), Michelangelo did not force teams to pick one framework. It was the *workflow* that was opinionated, not the algorithm.
3. **Monitoring is a first-class component.** Michelangelo treated prediction-quality monitoring as part of the platform, not an afterthought. Most teams discover the need for monitoring only after a model silently degrades in production for weeks; baking it in from day one saves that pain.
4. **The case study is *underspecified* on purpose.** The 2017 blog post does not give you implementation details for most of these systems — and that's intentional. Uber was sharing the *shape* of their platform, not a blueprint. When you read public ML platform case studies, treat them as shape, not blueprint.

### What's *not* shown in the public writeup

A common mistake is to treat the Michelangelo blog post as a complete spec. It isn't. The post omits:

- The IAM/auth story (multi-tenancy across Uber's many teams was almost certainly nontrivial; see Module 03).
- The cost allocation story (training is expensive; someone has to pay).
- The migration story (how did Uber get teams onto Michelangelo from their bespoke pipelines? This is usually the hardest part).
- The team structure (how many platform engineers? How did they prioritize?).

When you read case studies, ask "what is the author *not* saying?" The omissions are often more interesting than the inclusions.

---

## Case Study: Netflix Metaflow

> Source: ["Open-Sourcing Metaflow"](https://netflixtechblog.com/open-sourcing-metaflow-a-human-centric-framework-for-data-science-fa72e04a5d9), Netflix Tech Blog, 2019; and the [Metaflow documentation](https://docs.metaflow.org/). Both are publicly available.

Metaflow is a useful counterpoint to Michelangelo. Where Michelangelo is a "platform" in the heaviest sense — a coordinated suite of services with a unified workflow — Metaflow leans much harder on a *Python SDK* as the primary user surface. The contrast illuminates a real choice every ML platform team faces.

### The Metaflow philosophy

Metaflow's design centerpiece, repeated in nearly all of Netflix's public writeups, is **"human-centric framework for data science."** They mean it: Metaflow's biggest opinion is that *the data scientist's Python code should be the artifact*, and the platform should make that Python code runnable in many places (locally, on a cluster, on AWS Batch) without rewriting.

This is captured in Metaflow's central abstraction: a *flow*, written as a Python class with annotated steps:

```python
from metaflow import FlowSpec, step

class MyTrainingFlow(FlowSpec):
    @step
    def start(self):
        self.data = load_data()
        self.next(self.train)

    @step
    def train(self):
        self.model = train_model(self.data)
        self.next(self.end)

    @step
    def end(self):
        print("done")

if __name__ == "__main__":
    MyTrainingFlow()
```

The data scientist writes this file. Metaflow handles:

- Versioning every artifact attached to `self`.
- Running each step in its own container.
- Resuming from a failed step without rerunning the previous successful steps.
- Promoting from local execution to AWS Batch with a `--with batch` flag.
- Capturing every input and output for later debugging.

### The architectural shape

Metaflow sits at a different point in the design space than Michelangelo:

- **Michelangelo**: backend services (registry, training, serving) are the primary surface; SDKs are thin wrappers over those services.
- **Metaflow**: the Python SDK is the primary surface; the backend services (metadata, datastore, compute backends) are *implementation details* of the SDK.

This is a real design choice with real consequences. Metaflow's approach is more familiar to data scientists (they write Python, like they always have) and more decoupled from any specific backend (it supports AWS Batch, Kubernetes, AWS Step Functions, and Airflow as alternative execution backends). Michelangelo's approach is more centrally controllable (the platform team has direct ownership of all the services).

### What we can learn from Metaflow

1. **The SDK can be the platform.** A platform doesn't have to look like a constellation of services with REST APIs. Sometimes the most successful platform is a Python SDK that does the right thing by default.
2. **Local-to-prod parity matters.** Metaflow is obsessive about making "I ran it locally" and "I ran it on the cluster" the same experience. This is a huge productivity win.
3. **Versioning is opt-out, not opt-in.** Metaflow versions everything attached to `self` automatically. Most ML platforms make versioning opt-in (you have to call `mlflow.log_artifact`); Metaflow flips this. The result is that, for Metaflow users, they *cannot forget* to log an artifact.
4. **Open-source platforms are real platforms.** Metaflow was open-sourced in 2019 and is now used at many companies beyond Netflix. The open-source-internal-platform pattern (build internally, open-source later) has become common.

### Compare and contrast

| Dimension | Michelangelo | Metaflow |
| --- | --- | --- |
| Primary user surface | Services + workflow | Python SDK |
| Opinionatedness | High (workflow) | High (Python flow model) |
| Best fit | Many heterogeneous ML teams | Notebook-native ML teams |
| Open source? | No (internal to Uber) | Yes (since 2019) |
| Versioning model | Explicit (register model) | Implicit (auto-version flow artifacts) |

Most modern ML platforms borrow from both: services for the heavy lifting (storage, serving, scheduling), and a Python SDK as the front door so users don't have to call services directly.

We will return to these two as reference points throughout the curriculum. They are not the only ML platforms in the world — at the time of writing there are also LinkedIn Pro-ML, Spotify's internal platform, Airbnb's Bighead (since wound down), Lyft's LyftLearn — but Michelangelo and Metaflow are the most thoroughly publicly documented.

---

## Anti-Patterns and Failure Modes

Platforms fail in characteristic ways. Recognize these patterns; they recur.

### Anti-pattern 1: The "tooling pile"

Symptom: you have ten tools (Airflow + MLflow + Feast + Kubeflow + Prefect + DVC + Weights & Biases + ...) listed on a wiki page. There is no integration. Each user picks which subset they use. Users repeatedly write integration glue between the tools.

Why it happens: each tool was adopted in isolation, often by a different team, often without a platform team to integrate them.

Mitigation: a platform team has to *own the seams*. Tools that aren't integrated aren't a platform.

### Anti-pattern 2: Premature platformization

Symptom: you've built a platform for two teams and 0.2 use cases each. The platform is more complex than the use cases require. Teams find it easier to ignore the platform and build their own thing.

Why it happens: enthusiastic platform engineers, no real consumer demand.

Mitigation: rule of three (don't generalize a pattern until at least three real consumers want it), and constant user research.

### Anti-pattern 3: The accidental ticketing system

Symptom: the "platform" is actually a ticketing queue. Users file tickets ("please deploy my model"); platform engineers do the work manually. This pattern can deliver enormous value early on, but it does not scale.

Why it happens: ticketing is the fastest way to deliver early value. It feels like progress. It is necessary in the bootstrapping phase. It becomes the work.

Mitigation: track *what fraction of requests are self-service*. If that number is going up, the platform is succeeding. If it's going down or flat, you are building a service organization, not a platform.

### Anti-pattern 4: The Kubernetes-passthrough

Symptom: your "ML platform" is a thin wrapper around `kubectl apply`. Data scientists are expected to write Kubernetes manifests, but with slightly better defaults.

Why it happens: platform engineers come from a Kubernetes background and forget that the user does not.

Mitigation: never expose YAML to a data scientist. The platform has to absorb the YAML layer entirely; a Python SDK or a web form is the right surface. Manifests are an implementation detail.

### Anti-pattern 5: One platform per VP

Symptom: each VP-level engineering organization has built its own "platform." There are three feature stores in the company. There are two model registries. There are four training schedulers. Each platform team is too understaffed to run their platform well.

Why it happens: misaligned incentives. Each VP wants to own their own destiny.

Mitigation: this is a leadership problem, not a technical one. The platforms have to be consolidated, which means political work. We discuss this in chapter 05.

### Anti-pattern 6: The reverse-shadow-IT trap

Symptom: the platform team builds a beautiful platform that nobody uses. Users continue to ship via bespoke notebooks and cron jobs.

Why it happens: the platform was built without sufficient user research, or it was built for the wrong user persona (e.g., for ML engineers when the actual users are data scientists), or it has a worse user experience than the shadow-IT alternative.

Mitigation: ruthless user research. Adoption metrics. Time-to-first-value measurement. We discuss DX and adoption in chapter 02.

### Anti-pattern 7: The "we'll harden it later" leak

Symptom: the platform was built for one tenant, then onboarded to many tenants, without revisiting the multi-tenancy assumptions. There are no resource quotas; one team's training job can starve another's. Credentials leak between namespaces. Audit logs are incomplete.

Why it happens: time pressure. Multi-tenancy is hard; postponing it feels rational at the time.

Mitigation: build multi-tenancy in from the start *if* you anticipate more than one tenant — which you usually do. We discuss multi-tenancy in chapter 03.

These seven patterns recur across almost every ML platform team's history. You will see most of them in your career; you may build some of them yourself. The first step is naming them.

---

## Summary

- A platform is a **reusable, opinionated, self-service substrate** that lets internal teams accomplish a class of tasks without re-inventing the supporting machinery. ML platforms are a specialization of this idea for ML-specific workflows.
- Platforms are not the same as infrastructure. Infrastructure is the raw capacity layer; platforms add structure, abstractions, defaults, and UX on top.
- ML platforms differ from general application platforms in **five ways**: data is a first-class citizen, the artifact (model) is separate from the source, training is expensive and bursty, inference has accuracy as well as latency contracts, and the user is not a typical backend engineer.
- The platform engineer's job has **five facets**: build/operate platform services, integrate third-party tools, define opinionated defaults, support and evangelize, listen and evolve. A healthy team covers all five.
- The **platform value proposition** must be delivered to three audiences — users, operators, funders — in three different vocabularies (iteration speed, operational sanity, unit economics).
- **Michelangelo (Uber)** is the canonical "services + unified workflow" ML platform; **Metaflow (Netflix)** is the canonical "Python SDK as platform" alternative. Most real platforms borrow from both.
- **Anti-patterns** to avoid: tooling pile, premature platformization, accidental ticketing system, Kubernetes passthrough, one-platform-per-VP, reverse-shadow-IT, "we'll harden it later" multi-tenancy. Name them when you see them.

The rest of this module zooms into specific aspects: how to think about abstractions (chapter 02), how to handle multi-tenancy (chapter 03), how to design APIs (chapter 04), and the architectural patterns common across ML platforms (chapter 05).

---

## Further Reading

For each item, we recommend a "what to look for when reading" lens.

- **["Meet Michelangelo"](https://www.uber.com/blog/michelangelo-machine-learning-platform/) (Uber Engineering Blog, 2017).** Read for shape, not implementation. Note what *isn't* described.
- **["Open-Sourcing Metaflow"](https://netflixtechblog.com/open-sourcing-metaflow-a-human-centric-framework-for-data-science-fa72e04a5d9) (Netflix Tech Blog, 2019).** Read alongside the [Metaflow documentation](https://docs.metaflow.org/). Focus on the design philosophy section.
- **["Platform Engineering"](https://martinfowler.com/articles/platform-engineering.html) (Martin Fowler / Camille Fournier).** A short, dense piece on platform-as-product thinking. Required reading.
- **["Team Topologies"](https://teamtopologies.com/) by Skelton & Pais.** Not ML-specific, but the "platform team" pattern they describe is the team shape most ML platforms adopt. We will discuss this in chapter 05.
- **["Hidden Technical Debt in Machine Learning Systems"](https://papers.nips.cc/paper/2015/file/86df7dcfd896fcaf2674f757a2463eba-Paper.pdf) (Sculley et al., NeurIPS 2015).** The original "ML systems are 5% ML and 95% glue code" paper. Read once a year for the rest of your career.

In the next chapter, we shift from "what is a platform" to "how does a platform engineer *think*" — abstraction design, self-service vs full-service, developer experience, and adoption strategy.

> **Source note.** The six Michelangelo stages enumerated above (Manage data, Train models, Evaluate models, Deploy models, Make predictions, Monitor predictions) match the workflow described in Uber's 2017 ["Meet Michelangelo"](https://www.uber.com/blog/michelangelo-machine-learning-platform/) post. Verified 2026-05.
