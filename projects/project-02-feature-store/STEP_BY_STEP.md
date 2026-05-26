# Project 02 — Step-by-Step Build Guide

Build the feature store in phases. Get each phase running
before moving on; the materialization + retrieval boundary
must work cleanly.

## Phase 0 — Setup (1-2 hours)

1. Local Kubernetes (`kind` or `k3d`).
2. PostgreSQL + Redis running in-cluster.
3. Project skeleton:
   ```
   project-02-feature-store/
   ├── registry/         # FastAPI control plane
   ├── materializer/     # scheduled job
   ├── sdk/              # Python client
   ├── definitions/      # example feature YAMLs
   ├── deploy/
   ├── tests/
   └── Makefile
   ```

## Phase 1 — Feature-definition schema (4-5 hours)

1. Author the YAML schema (JSON Schema) for `Feature`.
2. Write a validator (`features validate <file>`).
3. Author 5-10 example features covering different source
   kinds.
4. Test: the validator rejects malformed definitions and
   accepts valid ones.

## Phase 2 — Offline store + retrieval (12-15 hours)

This is the largest single phase because point-in-time
correctness is subtle.

1. Define the offline table schema (per architecture.md).
2. Implement `register_feature(definition)` — creates the
   table.
3. Implement `materialize(feature, start_time, end_time)` —
   runs the source query, writes results.
4. Implement `get_historical_features(entity_df, features,
   timestamp_column)`:
   - For each (entity, timestamp) row, find the most recent
     materialized value at-or-before the timestamp.
   - SQL: `LATERAL JOIN` is a good fit. Or window functions.
5. **Critical test**: a deliberate-skew fixture.
   - Set up a feature whose materialized value at T1 is 5,
     and at T2 (later) is 10.
   - Query at (entity, T2 - 1) — must return 5, not 10.
   - This catches the bug where you naively join on
     "current value."

## Phase 3 — Online store + retrieval (8-10 hours)

1. Choose Redis (simpler) or PostgreSQL read-replica.
2. Implement `materialize_to_online(feature)` — copies the
   latest offline value into the online cache.
3. Implement `get_online_features(features, entity_id)` —
   batch read from cache.
4. Add staleness metadata to responses.
5. Test p99 latency: under 50ms for a single-entity lookup.

## Phase 4 — Materialization pipeline (6-8 hours)

1. A cron-scheduled job per feature.
2. On each run: materialize from source → offline → online.
3. Update lineage table.
4. Emit audit-log event.

For the capstone, a simple Kubernetes CronJob is sufficient.
Production-grade scheduling (Airflow, Prefect) is out of scope.

## Phase 5 — Feature registry + lineage (5-7 hours)

1. FastAPI service exposing the registry endpoints.
2. PostgreSQL-backed lineage table.
3. Lineage events on materialization + retrieval.
4. `GET /v1/features/{...}/lineage` returns the chain.

## Phase 6 — Multi-tenancy (5-7 hours)

1. Namespace field on feature definitions.
2. IAM policy / RBAC for serving identities.
3. Test: tenant A's serving identity attempts to read tenant
   B's features → 403.
4. Quota: max-features-per-namespace enforced at registration.

## Phase 7 — Observability + drift (5-6 hours)

1. Prometheus metrics.
2. Grafana dashboard.
3. Drift detection: scheduled job compares this week's
   feature distribution to last week's; flags if KL > 0.3.
4. Flagged features visible in the registry.

## Phase 8 — Testing + docs (8-10 hours)

1. Unit tests across components.
2. Integration test: full register-materialize-retrieve cycle.
3. The acceptance demo from `requirements.md`.
4. Documentation matching the standard project structure.

## Time-budget recap

| Phase | Hours |
|---|---|
| 0 — Setup | 1-2 |
| 1 — Definition schema | 4-5 |
| 2 — Offline store + PIT correctness | 12-15 |
| 3 — Online store | 8-10 |
| 4 — Materialization | 6-8 |
| 5 — Registry + lineage | 5-7 |
| 6 — Multi-tenancy | 5-7 |
| 7 — Observability + drift | 5-6 |
| 8 — Testing + docs | 8-10 |
| **Total** | **~70** |

## Common pitfalls

- **Naive feature joins**: joining "current value" instead of
  point-in-time value. The deliberate-skew test catches this.
- **Online-store as source of truth**: writes flowing to the
  online store outside of materialization create
  training/serving skew silently.
- **Forgetting backfill**: a new feature without backfill
  means historical training data has nulls for it. The
  feature definition's `backfill_from` field exists for this
  reason.
- **Time-zone bugs**: `event_time` columns should be
  `TIMESTAMPTZ`, not `TIMESTAMP`. Off-by-time-zone errors are
  the worst-class feature store bugs.

## When you're done

Your feature store handles:
- Point-in-time-correct training-data retrieval.
- Low-latency online retrieval.
- Multi-tenant access boundaries.
- Lineage + drift detection.
- Backfill of historical values for new features.

This is the substrate Modules 05, 06, and 07 build on. Submit
once the acceptance demo passes end-to-end.
