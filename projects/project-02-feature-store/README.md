# Project 02: Enterprise Feature Store

> **Tier**: Capstone
> **Track**: AI/ML Platform Engineering
> **Estimated effort**: 70 hours
> **Complexity**: Advanced
> **Primary modules**: mod-004 (Feature Store), mod-002 (API Design), mod-003 (Multi-Tenancy)
> **Secondary modules**: mod-008 (Observability), mod-009 (Security & Governance)

## 1. Overview

Build a **multi-tenant feature store** with both **batch** (for
training) and **online** (for serving) interfaces. The store
must guarantee training/serving parity, enforce per-tenant
isolation, and provide point-in-time correctness for backfills.

The deliverable is a feature store at the level a real ML
platform team would operate. Not a thin wrapper over Redis or
a single table — a proper feature platform with feature
definitions, materializations, registry, and lineage.

## 2. Why this project matters

The single biggest source of production-ML bugs is **training/
serving skew**: the features your model trained on differ from
the features it sees at inference. A feature store, designed
well, eliminates the class.

Module 04's lecture notes explain *why* feature stores exist;
this project makes you build one that holds up under multi-team
load and produces evidence of correctness.

## 3. What you will build

### Feature definitions

A YAML-based feature definition with:
- Source (Postgres table, S3 Parquet, Kafka topic, etc.).
- Transformation logic.
- Materialization schedule.
- TTL + freshness expectations.
- Owner + tenant.

### The two interfaces

- **Batch**: point-in-time correct feature retrieval for
  training datasets. Given a list of `(entity_id, timestamp)`
  pairs, return the features as they were at that timestamp.
- **Online**: low-latency lookup for serving. Given an
  `entity_id`, return the current features.

Both backed by the same feature definitions to prevent skew.

### Feature registry

A control plane that:
- Versions feature definitions (Git-style).
- Tracks lineage (which datasets feed which features).
- Records who registered each feature, when.
- Surfaces feature health (freshness, drift).

### Storage

- **Offline store**: PostgreSQL or S3 + Parquet (for the batch
  layer).
- **Online store**: Redis or DynamoDB or a Postgres replica
  (for the serving layer).
- **Materialization pipeline**: pushes offline → online on a
  schedule.

### Multi-tenancy

- Per-tenant feature namespaces.
- Per-tenant quotas (features per tenant, storage per
  tenant).
- Per-tenant access policies (the serving identity for tenant
  A cannot read tenant B's features).

## 4. Out of scope

- A web UI.
- Real-time streaming feature engineering (use Kafka source
  with batch materialization, not stream processing).
- Federated feature sharing across organizations.

## 5. Time budget

| Phase | Hours |
|---|---|
| Feature-definition design | 6 |
| Offline store + retrieval | 12 |
| Online store + retrieval | 10 |
| Materialization pipeline | 8 |
| Feature registry | 8 |
| Multi-tenancy + access control | 8 |
| Observability + lineage | 6 |
| Testing + documentation | 12 |
| **Total** | **~70** |

## 6. Deliverables

See [`requirements.md`](./requirements.md) and follow [`STEP_BY_STEP.md`](./STEP_BY_STEP.md).

## 7. Cross-references

- [Module 04 lecture notes](../../lessons/mod-004-feature-store/).
- [Feast documentation](https://docs.feast.dev/) — production-grade feature store you can reference.
- [`engineer-solutions/mod-106 exercise-07`](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops/exercise-07-feature-engineering) — feature engineering basics.
