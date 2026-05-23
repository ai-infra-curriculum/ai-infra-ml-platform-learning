# Lecture 02: Feast Architecture in Practice

## The Feast objects

```python
from feast import Entity, FeatureView, Field, FileSource
from feast.types import Float32, Int64
from datetime import timedelta


user = Entity(name="user", join_keys=["user_id"])

clicks_source = FileSource(
    name="clicks_source",
    path="data/clicks.parquet",
    timestamp_field="event_ts",
)

user_clicks_fv = FeatureView(
    name="user_clicks",
    entities=[user],
    ttl=timedelta(days=7),
    schema=[
        Field(name="clicks_7d", dtype=Int64),
        Field(name="clicks_30d", dtype=Int64),
    ],
    source=clicks_source,
)
```

Concepts:
- **Entity**: a thing features describe (user, item, session)
- **FeatureView**: a named bundle of features for an entity, backed by a source
- **Source**: where the data lives (FileSource, BigQuerySource, KafkaSource)
- **TTL**: how stale a feature can be before it's considered missing

## Materialization

```python
fs.materialize(start_date=last_run, end_date=now)
```

Pulls all FeatureView data from the offline source for the time window, writes
to the online store. Runs every 5-30 minutes typically. Backfill is the same
call with an older `start_date`.

## Online vs offline serving

```python
# Offline: training
training_df = fs.get_historical_features(
    entity_df=labels_df,       # (user_id, event_ts) pairs
    features=["user_clicks:clicks_7d", "user_purchase:ltv"],
).to_df()

# Online: prediction
online = fs.get_online_features(
    features=["user_clicks:clicks_7d"],
    entity_rows=[{"user_id": 42}],
).to_dict()
```

The same feature name returns the same values, computed via the same logic.

## Deployment topology

```
┌─── Feature Registry (Postgres) ─────┐
│                                      │
│  ┌─ Offline Store ─┐  ┌─ Online ──┐ │
│  │ S3 / Snowflake  │  │  Redis    │ │
│  └────────┬────────┘  └─────┬─────┘ │
│           │                  │       │
│           └──── materialize ─┘       │
└──────────────────────────────────────┘
        ▲                  ▲
        │                  │
   Training job       Serving job
```

## Operational concerns

- **Source freshness**: alert when source data hasn't updated in N hours
- **Materialization lag**: alert when online lag from offline grows
- **Online store capacity**: monitor Redis memory + eviction rate
- **Feature drift**: per-feature PSI computed daily; alert > 0.25
- **Cost**: online store cost grows linearly with entity count × feature count

## When to switch from Feast

Feast is a great default but has limits:
- Streaming features: works but awkward; consider Tecton or Hopsworks
- Sub-10ms p99 at 100K+ QPS: profile the online store; consider managed (Tecton, Hopsworks)
- Strong enterprise governance: managed offerings have more features
