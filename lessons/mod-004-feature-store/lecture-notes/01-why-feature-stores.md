# Lecture 01: Why Feature Stores Exist

## The two problems

### Training-serving skew

A feature is computed two different ways:
- **At training time**: a SQL query over a Parquet snapshot
- **At serving time**: live Python aggregating recent events

These two paths drift. Different rounding. Different time windows. Different
default values for missing data. The training-time feature distribution does
not match the serving-time feature distribution, and the model silently
performs worse in production than offline metrics suggested.

A feature store solves this by being the *single* source of truth for feature
values, used by both training and serving.

### Feature reuse

Three data scientists each compute "user's 30-day purchase count" with three
slightly different definitions (rolling window vs calendar month vs UTC vs
local time). Same name, different values. The platform team learns about it
when bills don't add up.

A feature store solves this by hosting feature definitions in code,
discoverable by other teams, computed once.

## The three layers

| Layer | Role | Storage | Latency |
|---|---|---|---|
| Offline store | training data + historical analysis | Parquet on S3 / Snowflake / BigQuery | minutes |
| Online store | serving lookup | Redis / DynamoDB / Bigtable | < 10ms |
| Registry | feature definitions + metadata | Postgres + Git | n/a |

A materialization job copies the latest values from offline → online on a
schedule.

## Point-in-time correctness

For training, the join must respect time:
```
For each (user_id, label_timestamp) in the label table:
  retrieve features as they were AT label_timestamp
  (not "now"; not "yesterday"; the specific moment)
```

This prevents data leakage. A common bug: training data joins against current
features, so the model learns from features the user *would* have at serving
time but didn't *yet* at the time the label was generated. The offline model
looks great; the online model is awful.

Feast, Tecton, and Feathr all enforce PIT joins by default.

## What feature stores don't solve

- Feature engineering itself: the SQL / Python code is still yours to write
- Data quality at the source: bad upstream data still produces bad features
- Model selection: feature store is upstream of training
- Inference compute: serving still happens in a model server

## When you don't need one

- Single team, single model: just be careful
- Static features only (don't change at serving time): a Parquet file is fine
- Sub-100 model org with strong contracts: defer

When you do need one:
- 3+ teams sharing features
- Mix of batch + streaming features
- High cost-of-skew (regulated industries, high-traffic serving)
