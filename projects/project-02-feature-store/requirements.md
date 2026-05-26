# Project 02 — Requirements

## Functional requirements

### F1 — Feature definitions

- [ ] YAML schema for feature definitions, validated at
      registration.
- [ ] Versioning: each definition has a semver version; new
      versions require materialization before they're queryable.
- [ ] Support at least 3 source kinds: PostgreSQL, S3+Parquet,
      static CSV.

### F2 — Offline retrieval (point-in-time correct)

- [ ] `get_historical_features(entity_df, features, timestamp_column)`.
- [ ] No future leak: features at row's timestamp, not "now".
- [ ] Tested with a deliberate skew scenario: a feature
      computed naively (without point-in-time correctness) is
      detectably wrong; the correct implementation passes.

### F3 — Online retrieval

- [ ] `get_online_features(features, entity_id)` returns
      current cached values.
- [ ] p99 latency under 50ms for a single-entity lookup.
- [ ] Returns staleness metadata so callers can decide whether
      to trust a stale value.

### F4 — Materialization pipeline

- [ ] Scheduled per feature (cron-style).
- [ ] Append-only writes to offline table (no destructive
      overwrites).
- [ ] Updates online cache from latest offline values.
- [ ] Recorded in lineage + audit log.

### F5 — Feature registry

- [ ] `POST /v1/features` registers a new feature.
- [ ] `GET /v1/features/{namespace}/{name}` retrieves
      definition.
- [ ] `GET /v1/features?namespace=...` lists features in a
      namespace.
- [ ] Lineage retrievable via `GET /v1/features/{...}/lineage`.

### F6 — Multi-tenancy

- [ ] Per-tenant namespaces.
- [ ] Serving identity for tenant A cannot read features in
      tenant B's namespace.
- [ ] Cross-tenant access attempts produce a clear 403 +
      audit-log entry.

### F7 — Observability

- [ ] Per-feature freshness metrics (`feature_last_materialized_seconds_ago`).
- [ ] Per-feature retrieval latency histogram.
- [ ] Per-feature retrieval volume counter.
- [ ] Grafana dashboard JSON included.

### F8 — Quality + drift

- [ ] Per-feature drift detection: alert when the distribution
      of materialized values differs significantly from the
      prior week's distribution (KL > 0.3 or similar threshold).
- [ ] Drift events visible in the registry as a flag on the
      feature.

## Non-functional requirements

- [ ] `make up` brings up the full system locally (kind /
      k3d acceptable).
- [ ] Unit tests on offline + online retrieval logic.
- [ ] Integration test demonstrating point-in-time correctness
      with a deliberate-skew scenario.
- [ ] All container images non-root + Cosign-signed.
- [ ] No long-lived static credentials.
- [ ] OpenAPI spec auto-generated from FastAPI.

## Acceptance demo

1. Register two features for two tenants.
2. Materialize once. Verify offline + online both populated.
3. Submit a `get_historical_features` query that crosses a
   materialization boundary; verify point-in-time correctness
   (the test fixture includes a deliberate-skew case).
4. Submit a serving-side `get_online_features` query.
5. Attempt cross-tenant read → fails with 403.
6. Force a drift scenario; verify the alert + flag.
7. Trigger a backfill on a new feature; verify historical
   values populate point-in-time-correctly.

## Submission

`deliverables/` contains:
- `code/`
- `docs/`
- `tests/`
- `screenshots/` — Grafana dashboard, point-in-time test
  passing, drift alert firing, cross-tenant denial.
- `SUBMISSION.md` (1-page summary).
