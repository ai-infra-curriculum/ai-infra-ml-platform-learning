# Project 04: Model Management System

> **Tier**: Capstone
> **Track**: AI/ML Platform Engineering
> **Estimated effort**: 70 hours
> **Complexity**: Advanced
> **Primary modules**: mod-006 (Model Management), mod-002 (API Design), mod-009 (Security & Governance)
> **Secondary modules**: mod-008 (Observability), mod-004 (Feature Store)

## 1. Overview

Build a **model management system** covering the full model
lifecycle: registration, versioning, signing, promotion,
deployment, and decommission. The registry is the system of
record for "what model is in production where and why."

Not a thin wrapper over MLflow — a proper management layer
with governance, lineage, and the audit-chain integration the
platform needs.

## 2. Why this project matters

Most ML incidents start with the question "which model is
running in production right now, and is that the one we
intended?" If the answer takes more than 30 seconds, the
system is broken.

A well-designed model management layer makes that question
trivial. Module 06's lecture notes set up the framework; this
project makes you build the system.

## 3. What you will build

### Model registry

- Models registered with: name, version, training metadata,
  dataset lineage, accuracy/fairness metrics, signature.
- Versioned (semver) with immutable releases.
- Search + filter by tenant, status, training data version,
  metric thresholds.

### Promotion gates (cross-references project-03)

A model moves from `Registered` → `Staging` → `Production`
through gates:
- Cosign signature valid.
- Accuracy meets threshold.
- Fairness metrics within tolerance.
- Adversarial robustness measured.
- Human approval recorded.

The gate evaluation reuses project-03's gate framework if both
are deployed; otherwise this project ships its own.

### Deployment surface

- `Deployment` resource that ties a model version to a target
  (staging cluster, prod cluster, specific tenants).
- Rollout strategies: rolling, blue-green, canary, shadow.
- Rollback to a prior version with a single command.

### Lineage + audit

- Every model traces back to: training code SHA, training
  data version(s), feature store snapshot, training-run ID.
- Every promotion is an audit-chain event with the
  approver's signed approval.
- Every deployment is an audit-chain event.

### Multi-tenancy

- Per-tenant model namespaces.
- A tenant's serving identity can read its own models, not
  other tenants'.
- Cross-tenant model sharing is explicit + audited.

## 4. Out of scope

- The actual serving runtime (mod-006 covers it; this project
  manages the *metadata* of what's served).
- The training pipeline (project-03).
- Model performance optimization (Performance track).

## 5. Time budget

| Phase | Hours |
|---|---|
| Model registry data model + API | 10 |
| Signing + signature verification | 6 |
| Promotion gates | 10 |
| Deployment + rollout strategies | 12 |
| Rollback | 4 |
| Lineage | 8 |
| Multi-tenancy + audit | 8 |
| Observability | 5 |
| Testing + docs | 7 |
| **Total** | **~70** |

## 6. Cross-references

- [Module 06 lecture notes](../../lessons/mod-006-model-management/).
- [MLflow Model Registry docs](https://mlflow.org/docs/latest/model-registry.html) — a reference for the conceptual surface.
- [`engineer-solutions/mod-106 exercise-03`](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops) — model registry basics.
- Security track Module 10 — supply-chain controls for model artifacts.
