# Project 05: Developer Portal & SDK

> **Tier**: Capstone
> **Track**: AI/ML Platform Engineering
> **Estimated effort**: 60 hours
> **Complexity**: Intermediate-Advanced
> **Primary modules**: mod-007 (Developer Experience), mod-002 (API Design)
> **Secondary modules**: mod-001 (Platform Fundamentals), mod-008 (Observability)

## 1. Overview

Build the **developer portal**: the unified user-facing surface
where data scientists discover what the platform offers, learn
how to use it, and get answers when things break.

The deliverable is the difference between "we have a platform"
and "a data scientist can self-serve productively in their
first week." Module 07's lecture notes make the case for *why*
DX matters; this project makes you build *what* DX looks like.

## 2. Why this project matters

A platform that no one uses is not a platform; it's a parked
project. Adoption is the platform team's product metric. The
portal is the single biggest lever on adoption.

A good portal answers, for any data scientist on day one:
- What can the platform do for me?
- How do I get started?
- How do I check the status of my running job?
- When something fails, where do I look?
- How do I get help?

## 3. What you will build

### Portal web UI

A web app (React or similar) covering:
- **Onboarding flow** — sign in via SSO; tenant association;
  initial setup steps.
- **Dashboard** — your runs, your models, your features,
  resource usage, alerts.
- **Catalog** — what's available (datasets, features, base
  models, environments).
- **Documentation hub** — searchable docs.
- **Runbook + playbook surface** — operational guides when
  things go wrong.
- **Status page** — platform health.

### Unified SDK

A Python SDK consolidating the surfaces from projects 01-04:

```python
from smartrecs import Platform

p = Platform()    # auth via OIDC token from environment

# Project 01: training runs
run = p.training.create_run(...)

# Project 02: features
training_set = p.features.get_historical(...)

# Project 03: workflows
pipeline = p.workflows.register(spec_path="pipeline.yaml")
run = p.workflows.run(pipeline.id)

# Project 04: model registry
model_version = p.models.register(name="recs", version="v17", ...)
p.models.promote(model_version.id, to="Production")
```

### CLI

`smartrecs` as a single command with subcommands matching the
SDK surface. Cross-platform (mac / linux). Bash + zsh
completion.

### Documentation

- Getting started guide.
- Per-module quickstarts.
- API reference (auto-generated from OpenAPI).
- Architecture overview.
- Operations runbooks.
- FAQ.

Built with Docusaurus, MkDocs, or similar; searchable.

### Developer feedback channels

- Built-in feedback widget (sends to Slack channel + tickets).
- Issue templates pre-populated with context (which command
  ran, which version, what failed).

## 4. Out of scope

- Designing a separate authentication system (use the same
  OIDC integration as the rest of the platform).
- A marketplace for community-contributed components.
- Implementing the underlying platform services (this project
  is the unified surface; projects 01-04 are the services).

## 5. Time budget

| Phase | Hours |
|---|---|
| Portal UI (React / Next.js) | 20 |
| Unified SDK | 8 |
| CLI | 6 |
| Documentation site | 8 |
| Onboarding flow | 4 |
| Status page + health | 4 |
| Feedback channels | 3 |
| Testing + accessibility | 7 |
| **Total** | **~60** |

## 6. Skills demonstrated

- Frontend engineering for ML-internal tooling.
- API-driven UI design.
- Documentation-as-product mindset.
- Discoverability + onboarding design.

## 7. Cross-references

- [Module 07 lecture notes](../../lessons/mod-007-developer-experience/) — the conceptual treatment of platform DX.
- All four previous projects — the portal sits on top of them.
- [Backstage](https://backstage.io/) — open-source IDP framework; reference for what production portals look like.
