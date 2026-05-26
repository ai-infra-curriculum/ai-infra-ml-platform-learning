# Project 02 — Architecture

## Component map

```
┌──────────────────────────────────────────────────────────────┐
│ USER LAYER                                                   │
│  • Python SDK (training + serving)                           │
│  • Feature-definition YAML (Git-versioned)                   │
└──────────────────┬─────────────────────┬─────────────────────┘
                   │                     │
   batch retrieval │                     │ online retrieval
                   ▼                     ▼
┌──────────────────────────┐  ┌──────────────────────────┐
│ Offline store            │  │ Online store              │
│  PostgreSQL or S3+Parquet│  │  Redis / DynamoDB         │
│  Point-in-time queries   │  │  Low-latency key lookups  │
└──────────┬───────────────┘  └──────────▲───────────────┘
           │                              │
           │ source of truth              │ materialized from offline
           │                              │
           ▼                              │
┌──────────────────────────────────────────┴───────────────────┐
│ Feature platform                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │ Registry       │  │ Materializer   │  │ Lineage tracker│ │
│  │ (definitions,  │  │ (scheduled)    │  │                │ │
│  │  versions)     │  │                │  │                │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

## Feature-definition example

```yaml
apiVersion: features.smartrecs.io/v1
kind: Feature
metadata:
  name: user_recency_days
  namespace: recs-team
  labels:
    owner: recs-team
    tenant: recs
spec:
  description: "Days since the user's last interaction"
  type: integer
  entity: user_id
  source:
    kind: postgres
    connection: warehouse
    query: |
      SELECT user_id, EXTRACT(DAY FROM NOW() - MAX(event_time)) AS user_recency_days
      FROM user_events
      WHERE event_time > NOW() - INTERVAL '90 days'
      GROUP BY user_id
  materialization:
    schedule: "@hourly"
    backfill_from: "2025-01-01"
  ttl: 24h
  freshness_sla: 2h
```

## Key design decisions

### 1. One source of truth: feature definitions

The YAML definitions are the source of truth. The offline
table is computed from them. The online cache is computed from
the offline table. Skew is impossible because there's exactly
one definition path.

Trade-off: changing a definition requires re-materialization.
A definition cannot be silently mutated without leaving evidence.

### 2. Point-in-time correctness for training

Offline retrieval supports `get_historical_features(entity_df,
features, timestamp_column)` where each row's features are
those that were valid at that row's timestamp. This prevents
"future leak" in training datasets.

Implementation: an as-of-join in SQL with the materialized
feature table's `event_time` column.

### 3. Online store is materialized, not authoritative

The online store is a cache. The offline store is authoritative.
If the online store is corrupted, we re-materialize. The online
store never receives writes from inference.

### 4. Multi-tenancy via namespace + IAM

Features have a `namespace` field. The serving identity for
tenant A holds an IAM policy that permits reading only features
in `namespace: recs-team`. Cross-tenant reads fail at IAM.

### 5. Backfill safety

When a new feature is added, materialization can backfill
historical values for training. The backfill is point-in-time
correct: it computes what the feature *would have been* at
each historical timestamp, not what it is now.

## Data model

```sql
-- Feature definitions (registry)
CREATE TABLE features (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    namespace TEXT NOT NULL,
    spec JSONB NOT NULL,
    version TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    created_by TEXT NOT NULL,
    UNIQUE (namespace, name, version)
);

-- Materialized feature values (offline store)
-- One table per feature, partitioned by event_time.
-- Example for the feature above:
CREATE TABLE feature__recs_team__user_recency_days (
    entity_id TEXT NOT NULL,
    value INTEGER NOT NULL,
    event_time TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (entity_id, event_time)
);
CREATE INDEX ON feature__recs_team__user_recency_days (entity_id, event_time DESC);

-- Lineage (which datasets feed which features)
CREATE TABLE feature_lineage (
    feature_id UUID REFERENCES features(id),
    source_kind TEXT NOT NULL,
    source_identifier TEXT NOT NULL,
    PRIMARY KEY (feature_id, source_identifier)
);
```

## SDK API

```python
from smartrecs_features import FeatureStore

fs = FeatureStore(namespace="recs-team")

# Training: get historical features for an entity dataframe
entity_df = pd.DataFrame({"user_id": [...], "label_time": [...]})
training_set = fs.get_historical_features(
    entity_df=entity_df,
    features=["user_recency_days", "user_lifetime_value", "user_country"],
    timestamp_column="label_time",
)

# Serving: get current features
user_features = fs.get_online_features(
    features=["user_recency_days", "user_lifetime_value", "user_country"],
    entity_id="user-42",
)
```

## Materialization pipeline

A scheduled job per feature that:
1. Runs the feature's `spec.source.query` against the warehouse.
2. Writes results to the offline table (append, no overwrite —
   each materialization is a new snapshot).
3. Updates the online cache with the latest values.
4. Records a lineage event (which source rows produced which
   feature rows).

## Per-tenant quotas

- Max features per namespace.
- Max storage per namespace (rough byte estimate).
- Max online-cache entries per namespace.

Enforced at registry-write time.

## Cross-references

- Module 04 lecture notes for the conceptual treatment.
- Module 03 lecture notes for the multi-tenancy patterns.
- [Feast](https://docs.feast.dev/) for a production reference.
