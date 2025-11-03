# Feature Store Architecture for ML Platforms

**Module 04 - Lecture 01**
**Duration**: 90 minutes
**Last Updated**: November 2, 2025

---

## Table of Contents

1. [Introduction](#introduction)
2. [Theoretical Foundations](#theoretical-foundations)
3. [Practical Implementation](#practical-implementation)
4. [Advanced Topics](#advanced-topics)
5. [Real-World Applications](#real-world-applications)
6. [Assessment](#assessment)

---

## Introduction

### What is a Feature Store?

A **feature store** is a centralized platform for managing, storing, and serving machine learning features to both training and inference systems. It acts as a data management layer that sits between raw data sources and ML models.

**Core Purpose**:
- **Consistency**: Same features used in training and inference (no training-serving skew)
- **Reusability**: Share features across teams and models
- **Governance**: Track feature lineage, versions, and quality
- **Performance**: Low-latency feature serving for real-time inference

### The Feature Store Problem

Without a feature store, ML teams face:

1. **Training-Serving Skew** (70% of production ML failures)
   - Training uses batch-computed features from data warehouse
   - Serving recomputes features in real-time with different code
   - Result: Model performs well in training, fails in production

2. **Feature Duplication**
   - Multiple teams compute same features differently
   - "user_30d_transaction_count" exists in 5 different codebases
   - Inconsistent definitions lead to model errors

3. **Data Leakage**
   - Training data includes future information
   - Features computed without point-in-time correctness
   - Models learn from data that wouldn't be available at inference time

4. **Operational Complexity**
   - Each model maintains its own feature pipelines
   - No centralized monitoring or quality checks
   - Scaling challenges (N models × M features = N×M pipelines)

### Feature Store Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Feature Store                        │
│                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────┐ │
│  │   Feature    │    │   Feature    │    │ Feature  │ │
│  │  Registry    │───>│ Computation  │───>│ Storage  │ │
│  │  (Metadata)  │    │  (Transform) │    │(On/Off)  │ │
│  └──────────────┘    └──────────────┘    └──────────┘ │
│         │                    │                   │     │
│         ↓                    ↓                   ↓     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────┐ │
│  │   Feature    │    │   Feature    │    │ Feature  │ │
│  │  Discovery   │    │  Monitoring  │    │ Serving  │ │
│  │     (UI)     │    │   (Quality)  │    │  (API)   │ │
│  └──────────────┘    └──────────────┘    └──────────┘ │
└─────────────────────────────────────────────────────────┘
         ↑                                         ↓
    Data Sources                            ML Applications
  (Warehouse, Stream)                    (Training, Inference)
```

### Learning Objectives

By the end of this lecture, you will:

1. ✅ Understand online vs offline stores and their use cases
2. ✅ Implement point-in-time correct feature joins
3. ✅ Design feature transformation pipelines
4. ✅ Integrate Feast feature store with ML workflows
5. ✅ Prevent data leakage in training sets
6. ✅ Serve features with <10ms latency
7. ✅ Monitor feature quality and drift

---

## Theoretical Foundations

### 1. Online vs Offline Stores

Feature stores maintain two separate storage systems optimized for different workloads:

#### Offline Store (Batch Features)

**Purpose**: Historical feature values for training data generation

**Characteristics**:
- **Storage**: Data warehouse (BigQuery, Snowflake, S3 + Athena)
- **Latency**: Seconds to minutes (batch queries)
- **Data Volume**: Terabytes to petabytes
- **Access Pattern**: Range queries, point-in-time joins
- **Update Frequency**: Hourly to daily batch jobs

**Use Cases**:
- Training data generation
- Batch scoring (offline predictions)
- Backtesting models
- Feature validation and analysis

**Example Query**:
```sql
-- Get features as they existed at label timestamp
SELECT
  entity_id,
  label,
  user_30d_transactions,  -- Feature at event_timestamp
  user_avg_amount         -- Feature at event_timestamp
FROM offline_store
WHERE event_timestamp <= label_timestamp
  AND entity_id = 'user_123'
ORDER BY event_timestamp DESC
LIMIT 1
```

#### Online Store (Real-Time Features)

**Purpose**: Low-latency feature retrieval for real-time inference

**Characteristics**:
- **Storage**: Key-value store (Redis, DynamoDB, Cassandra)
- **Latency**: <10ms (p99 latency target)
- **Data Volume**: Gigabytes (latest values only)
- **Access Pattern**: Point lookups by entity key
- **Update Frequency**: Real-time streams or near-real-time (minutes)

**Use Cases**:
- Real-time predictions (web, mobile, API)
- Streaming feature computation
- A/B testing with fresh features
- Personalization engines

**Example Access**:
```python
# Get latest features for entity
features = online_store.get_online_features(
    feature_refs=[
        "user_features:30d_transactions",
        "user_features:avg_amount"
    ],
    entity_rows=[{"user_id": "user_123"}]
)
# Returns: {"30d_transactions": 42, "avg_amount": 127.5}
# Latency: ~5ms
```

#### Comparison Table

| Aspect | Offline Store | Online Store |
|--------|--------------|--------------|
| **Latency** | Seconds-minutes | <10ms |
| **Storage** | Warehouse (columnar) | KV Store (row-based) |
| **Data Size** | TBs-PBs (historical) | GBs (latest only) |
| **Query Type** | Analytical (joins, aggregations) | Lookup (get by key) |
| **Cost** | $$$$ (storage) | $ (compute) |
| **Updates** | Batch (hourly/daily) | Stream (real-time) |
| **Point-in-Time** | Yes (critical) | No (latest only) |

### 2. Point-in-Time Correctness

**The Data Leakage Problem**:

Imagine training a credit card fraud model on October 15, 2024:

**❌ Incorrect (Data Leakage)**:
```sql
-- BAD: Uses latest feature values, not values at transaction time
SELECT
  t.transaction_id,
  t.amount,
  t.timestamp AS transaction_time,
  t.is_fraud AS label,
  f.user_30d_transaction_count,  -- Computed on Oct 20 (includes future!)
  f.user_avg_amount                -- Includes transactions after Oct 15
FROM transactions t
JOIN user_features f ON t.user_id = f.user_id
WHERE t.timestamp < '2024-10-15'
```

**✅ Correct (Point-in-Time Join)**:
```sql
-- GOOD: Uses feature values as they existed at transaction time
SELECT
  t.transaction_id,
  t.amount,
  t.timestamp AS transaction_time,
  t.is_fraud AS label,
  f.user_30d_transaction_count,  -- Value at transaction_time
  f.user_avg_amount                -- Value at transaction_time
FROM transactions t
JOIN LATERAL (
  SELECT *
  FROM user_features f
  WHERE f.user_id = t.user_id
    AND f.event_timestamp <= t.timestamp  -- Point-in-time constraint
  ORDER BY f.event_timestamp DESC
  LIMIT 1
) f
WHERE t.timestamp < '2024-10-15'
```

**Point-in-Time Semantics**:

For each training example at time `T`, use feature values as they existed at time `T - ε` (some small delay for feature computation):

```
Timeline:
─────────────────────────────────────────────────────>
         T-2h    T-1h    T (event)   T+1h   T+2h
          │       │         │          │      │
          └───────┴─────────┘          │      │
          Feature window         ✓ Use this  │
          (historical)                  │      │
                                        ✗ Not this!
                                   (future leakage)
```

**Impact of Point-in-Time Correctness**:
- **Without**: AUC 0.95 in training → 0.72 in production (23% drop)
- **With**: AUC 0.88 in training → 0.86 in production (2% drop)

### 3. Feature Transformation Pipeline

Features flow through multiple stages from raw data to model input:

```
Raw Data → Ingestion → Transform → Storage → Serving → Model
   │          │           │          │         │        │
   ↓          ↓           ↓          ↓         ↓        ↓
Events    Validation   Feature    On/Off   Retrieval  Inference
         Schema Check  Compute    Store     (API)
```

#### Feature Types by Computation

**1. Batch Features** (Computed offline, high latency acceptable)
- Window aggregations: "user_30d_transactions", "product_7d_views"
- Complex joins: "category_avg_price", "merchant_fraud_rate"
- ML-derived: Embeddings, clustering features
- Update: Hourly/daily batch jobs

**2. Streaming Features** (Computed in real-time)
- Recent activity: "user_last_5_clicks", "session_duration"
- Real-time aggregations: "1h_transaction_count"
- Event-driven: "time_since_last_purchase"
- Update: Stream processing (Flink, Spark Streaming)

**3. On-Demand Features** (Computed at request time)
- Request-dependent: "hour_of_day", "day_of_week"
- Cross-feature operations: "price_vs_category_avg"
- Lightweight transforms: Normalization, encoding
- Update: Computed synchronously during serving

#### Example Feature Pipeline (Feast)

```python
from feast import Entity, FeatureView, Field, FileSource
from feast.types import Float32, Int64
from datetime import timedelta

# Define entity (primary key)
user = Entity(
    name="user_id",
    description="User identifier"
)

# Define data source
user_stats_source = FileSource(
    path="data/user_stats.parquet",
    timestamp_field="event_timestamp"
)

# Define feature view (feature group)
user_stats_fv = FeatureView(
    name="user_statistics",
    entities=[user],
    ttl=timedelta(days=90),  # Feature freshness requirement
    schema=[
        Field(name="transaction_count_30d", dtype=Int64),
        Field(name="avg_transaction_amount", dtype=Float32),
        Field(name="days_since_signup", dtype=Int64),
    ],
    source=user_stats_source,
    online=True,  # Enable online serving
    tags={"team": "risk", "domain": "user_behavior"}
)
```

### 4. Feature Versioning

As features evolve, versioning ensures reproducibility:

**Versioning Strategies**:

1. **Schema Evolution** (Backward compatible changes)
   ```python
   # v1: Original feature
   "user_30d_transactions"  # Count of all transactions

   # v2: Add new field (backward compatible)
   "user_30d_transactions"       # Original (unchanged)
   "user_30d_online_transactions"  # New filter (additive)
   ```

2. **Breaking Changes** (New feature version)
   ```python
   # v1: Bug in computation logic
   "user_avg_amount:v1"  # Incorrect: includes refunds

   # v2: Fixed computation
   "user_avg_amount:v2"  # Correct: excludes refunds

   # Both versions coexist for rollback safety
   ```

3. **Feature Lineage** (Track dependencies)
   ```
   user_transaction_count:v1
     ← derived from: transactions_table
     ← transform: COUNT(*) WHERE status = 'completed'
     ← created_at: 2024-01-15
     ← created_by: data-team@company.com
     ← models_using: [fraud_model:v3, churn_model:v2]
   ```

**Version Management in Feast**:
```python
from feast import FeatureService

# Define feature service (versioned feature set)
fraud_detection_v1 = FeatureService(
    name="fraud_detection",
    features=[
        user_stats_fv[["transaction_count_30d", "avg_transaction_amount"]],
        merchant_stats_fv[["chargeback_rate"]],
    ],
    tags={"version": "v1", "model": "xgboost_fraud"}
)

# Later: Update to v2 with new features
fraud_detection_v2 = FeatureService(
    name="fraud_detection",
    features=[
        user_stats_fv[["transaction_count_30d", "avg_transaction_amount"]],
        merchant_stats_fv[["chargeback_rate", "avg_ticket_size"]],  # Added
        device_stats_fv[["suspicious_device_count"]],  # New feature view
    ],
    tags={"version": "v2", "model": "xgboost_fraud_v2"}
)
```

---

## Practical Implementation

### 1. Setting Up Feast Feature Store

Feast is the leading open-source feature store, originally developed by Gojek and now maintained by Tecton.

#### Installation and Configuration

```bash
# Install Feast
pip install feast[redis]  # Includes Redis online store support

# Initialize Feast repository
feast init feature_repo
cd feature_repo
```

**Feature Repository Structure**:
```
feature_repo/
├── feature_store.yaml      # Configuration file
├── features.py             # Feature definitions
├── data/                   # Sample data sources
│   ├── user_stats.parquet
│   └── merchant_stats.parquet
└── feature_repo/
    └── __init__.py
```

**Configuration (feature_store.yaml)**:
```yaml
project: ml_platform
registry: data/registry.db
provider: local
online_store:
  type: redis
  connection_string: "localhost:6379"
offline_store:
  type: file  # Use 'bigquery', 'snowflake', 'redshift' in production
entity_key_serialization_version: 2
```

#### Defining Features

**features.py**:
```python
from feast import Entity, FeatureView, Field, FileSource, FeatureService
from feast.types import Float32, Int64, String
from datetime import timedelta

# ============================================
# STEP 1: Define Entities (Primary Keys)
# ============================================

user = Entity(
    name="user_id",
    description="User identifier",
    tags={"team": "platform"}
)

merchant = Entity(
    name="merchant_id",
    description="Merchant identifier"
)

# ============================================
# STEP 2: Define Data Sources
# ============================================

# Batch data source (for offline store)
user_stats_source = FileSource(
    name="user_stats_source",
    path="data/user_stats.parquet",
    timestamp_field="event_timestamp",
    created_timestamp_column="created_timestamp"  # For point-in-time joins
)

merchant_stats_source = FileSource(
    name="merchant_stats_source",
    path="data/merchant_stats.parquet",
    timestamp_field="event_timestamp"
)

# ============================================
# STEP 3: Define Feature Views
# ============================================

user_statistics = FeatureView(
    name="user_statistics",
    entities=[user],
    ttl=timedelta(days=90),
    schema=[
        Field(name="transaction_count_7d", dtype=Int64,
              description="Transaction count in last 7 days"),
        Field(name="transaction_count_30d", dtype=Int64,
              description="Transaction count in last 30 days"),
        Field(name="avg_transaction_amount", dtype=Float32,
              description="Average transaction amount (30d)"),
        Field(name="max_transaction_amount", dtype=Float32,
              description="Max transaction amount (30d)"),
        Field(name="account_age_days", dtype=Int64,
              description="Days since account creation"),
    ],
    source=user_stats_source,
    online=True,
    tags={"domain": "user_behavior", "pii": "false"}
)

merchant_statistics = FeatureView(
    name="merchant_statistics",
    entities=[merchant],
    ttl=timedelta(days=30),
    schema=[
        Field(name="total_transactions_30d", dtype=Int64),
        Field(name="avg_ticket_size", dtype=Float32),
        Field(name="chargeback_rate", dtype=Float32,
              description="Chargeback rate (30d)"),
        Field(name="refund_rate", dtype=Float32),
        Field(name="category", dtype=String),
    ],
    source=merchant_stats_source,
    online=True,
    tags={"domain": "merchant_risk"}
)

# ============================================
# STEP 4: Define Feature Services (Versioned Sets)
# ============================================

fraud_detection_features = FeatureService(
    name="fraud_detection",
    features=[
        user_statistics[[
            "transaction_count_7d",
            "transaction_count_30d",
            "avg_transaction_amount"
        ]],
        merchant_statistics[[
            "chargeback_rate",
            "refund_rate"
        ]]
    ],
    tags={"version": "v1", "model": "fraud_xgboost"}
)
```

#### Applying Features to Registry

```bash
# Apply feature definitions to registry
feast apply

# Output:
# Created entity user_id
# Created entity merchant_id
# Created feature view user_statistics
# Created feature view merchant_statistics
# Created feature service fraud_detection
#
# Deploying infrastructure for user_statistics
# Deploying infrastructure for merchant_statistics
```

### 2. Generating Training Data (Offline)

**Point-in-Time Correct Training Set**:

```python
from feast import FeatureStore
from datetime import datetime
import pandas as pd

# Initialize feature store
store = FeatureStore(repo_path="feature_repo/")

# Load entity dataframe (labels + timestamps)
entity_df = pd.DataFrame({
    "user_id": ["user_1", "user_2", "user_3"],
    "merchant_id": ["merch_1", "merch_2", "merch_1"],
    "event_timestamp": [
        datetime(2024, 10, 1, 12, 0),
        datetime(2024, 10, 2, 14, 30),
        datetime(2024, 10, 3, 9, 15),
    ],
    "is_fraud": [0, 1, 0]  # Labels
})

# Get historical features (point-in-time correct)
training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "user_statistics:transaction_count_30d",
        "user_statistics:avg_transaction_amount",
        "merchant_statistics:chargeback_rate",
        "merchant_statistics:refund_rate"
    ]
).to_df()

print(training_df.head())
```

**Output**:
```
   user_id merchant_id       event_timestamp  is_fraud  transaction_count_30d  avg_transaction_amount  chargeback_rate  refund_rate
0  user_1   merch_1    2024-10-01 12:00:00         0                     42                   127.5            0.015         0.032
1  user_2   merch_2    2024-10-02 14:30:00         1                      3                   892.3            0.087         0.054
2  user_3   merch_1    2024-10-03 09:15:00         0                     18                   203.1            0.015         0.032
```

**Key Points**:
- Features retrieved as they existed at `event_timestamp`
- No data leakage: only historical data used
- Feast handles complex temporal joins automatically

### 3. Materializing Features to Online Store

To serve features at low latency, materialize them to the online store (Redis):

```python
from datetime import datetime, timedelta

# Materialize last 7 days of features to online store
store.materialize(
    start_date=datetime.now() - timedelta(days=7),
    end_date=datetime.now()
)

# Output:
# Materializing 2 feature views from 2024-10-26 to 2024-11-02
# Materializing feature view user_statistics from 2024-10-26 to 2024-11-02
# ...100% complete
# Materializing feature view merchant_statistics from 2024-10-26 to 2024-11-02
# ...100% complete
```

**What Happens**:
1. Feast queries offline store for latest feature values
2. Transforms data to online store format (key-value)
3. Writes to Redis with TTL based on feature `ttl` setting
4. Features now available for real-time serving

**Materialization Pipeline (Production)**:
```python
# Schedule materialization job (runs every hour)
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def materialize_features():
    """Materialize last hour of features"""
    store = FeatureStore(repo_path="/opt/feature_repo")
    store.materialize_incremental(
        end_date=datetime.now()
    )

dag = DAG(
    'feature_materialization',
    schedule_interval='@hourly',
    start_date=datetime(2024, 1, 1),
)

materialize_task = PythonOperator(
    task_id='materialize',
    python_callable=materialize_features,
    dag=dag
)
```

### 4. Serving Features Online (Real-Time Inference)

```python
# Get online features for real-time prediction
entity_rows = [
    {"user_id": "user_123", "merchant_id": "merch_456"}
]

online_features = store.get_online_features(
    features=[
        "user_statistics:transaction_count_30d",
        "user_statistics:avg_transaction_amount",
        "merchant_statistics:chargeback_rate"
    ],
    entity_rows=entity_rows
).to_dict()

print(online_features)
# Output:
# {
#   'user_id': ['user_123'],
#   'merchant_id': ['merch_456'],
#   'transaction_count_30d': [42],
#   'avg_transaction_amount': [127.5],
#   'chargeback_rate': [0.015]
# }

# Latency: ~5-8ms (Redis lookup)
```

**Production Serving API (FastAPI)**:
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from feast import FeatureStore
import numpy as np
import joblib

app = FastAPI()
store = FeatureStore(repo_path="/opt/feature_repo")
model = joblib.load("fraud_model.pkl")

class PredictionRequest(BaseModel):
    user_id: str
    merchant_id: str
    transaction_amount: float

class PredictionResponse(BaseModel):
    is_fraud: bool
    fraud_probability: float
    features_used: dict

@app.post("/predict", response_model=PredictionResponse)
async def predict_fraud(request: PredictionRequest):
    """Real-time fraud prediction with feature store"""

    # 1. Fetch features from online store (5-10ms)
    features_dict = store.get_online_features(
        features=[
            "user_statistics:transaction_count_30d",
            "user_statistics:avg_transaction_amount",
            "merchant_statistics:chargeback_rate"
        ],
        entity_rows=[{
            "user_id": request.user_id,
            "merchant_id": request.merchant_id
        }]
    ).to_dict()

    # 2. Prepare model input
    X = np.array([[
        features_dict["transaction_count_30d"][0],
        features_dict["avg_transaction_amount"][0],
        features_dict["chargeback_rate"][0],
        request.transaction_amount
    ]])

    # 3. Predict
    proba = model.predict_proba(X)[0][1]
    is_fraud = proba > 0.5

    return PredictionResponse(
        is_fraud=is_fraud,
        fraud_probability=float(proba),
        features_used={
            "transaction_count_30d": features_dict["transaction_count_30d"][0],
            "avg_transaction_amount": features_dict["avg_transaction_amount"][0],
            "chargeback_rate": features_dict["chargeback_rate"][0]
        }
    )

# Performance:
# - Feature retrieval: ~7ms (p95)
# - Model inference: ~3ms
# - Total latency: ~10ms (p95)
```

### 5. Feature Monitoring

Monitor feature quality to detect issues early:

```python
from feast import FeatureStore
import pandas as pd
from scipy import stats

store = FeatureStore(repo_path="feature_repo/")

def monitor_feature_drift(
    feature_name: str,
    reference_df: pd.DataFrame,
    current_df: pd.DataFrame,
    threshold: float = 0.05
):
    """
    Detect feature drift using Kolmogorov-Smirnov test.

    Returns:
        bool: True if significant drift detected
    """
    statistic, p_value = stats.ks_2samp(
        reference_df[feature_name],
        current_df[feature_name]
    )

    has_drift = p_value < threshold

    if has_drift:
        print(f"⚠️  DRIFT DETECTED in {feature_name}")
        print(f"   KS statistic: {statistic:.4f}")
        print(f"   p-value: {p_value:.6f}")
        print(f"   Reference mean: {reference_df[feature_name].mean():.2f}")
        print(f"   Current mean: {current_df[feature_name].mean():.2f}")

    return has_drift

# Example: Check for drift in last 7 days vs previous 30 days
reference_period = store.get_historical_features(
    entity_df=entities_last_30d,
    features=["user_statistics:avg_transaction_amount"]
).to_df()

current_period = store.get_historical_features(
    entity_df=entities_last_7d,
    features=["user_statistics:avg_transaction_amount"]
).to_df()

monitor_feature_drift(
    "avg_transaction_amount",
    reference_period,
    current_period
)
```

---

## Advanced Topics

### 1. Stream Feature Computation

Real-time features computed from event streams:

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment
from feast import FeatureStore

# Flink job for streaming features
env = StreamExecutionEnvironment.get_execution_environment()
t_env = StreamTableEnvironment.create(env)

# Define source (Kafka topic with transaction events)
t_env.execute_sql("""
    CREATE TABLE transaction_events (
        user_id STRING,
        amount DOUBLE,
        timestamp TIMESTAMP(3),
        WATERMARK FOR timestamp AS timestamp - INTERVAL '5' SECOND
    ) WITH (
        'connector' = 'kafka',
        'topic' = 'transactions',
        'properties.bootstrap.servers' = 'localhost:9092'
    )
""")

# Compute streaming features (5-minute tumbling window)
t_env.execute_sql("""
    CREATE TABLE user_realtime_features AS
    SELECT
        user_id,
        COUNT(*) AS transaction_count_5m,
        AVG(amount) AS avg_amount_5m,
        MAX(amount) AS max_amount_5m,
        TUMBLE_END(timestamp, INTERVAL '5' MINUTE) AS event_timestamp
    FROM transaction_events
    GROUP BY user_id, TUMBLE(timestamp, INTERVAL '5' MINUTE)
""")

# Write to feature store (Redis sink)
t_env.execute_sql("""
    CREATE TABLE feature_store_sink (
        user_id STRING,
        transaction_count_5m BIGINT,
        avg_amount_5m DOUBLE,
        max_amount_5m DOUBLE,
        PRIMARY KEY (user_id) NOT ENFORCED
    ) WITH (
        'connector' = 'redis',
        'host' = 'localhost',
        'port' = '6379',
        'key-pattern' = 'feast:user_realtime_features:{user_id}'
    )
""")

t_env.execute_sql("""
    INSERT INTO feature_store_sink
    SELECT user_id, transaction_count_5m, avg_amount_5m, max_amount_5m
    FROM user_realtime_features
""")

# Execute streaming job
env.execute("User Realtime Features")
```

### 2. On-Demand Feature Views

Features computed at request time (low overhead transforms):

```python
from feast import Field, on_demand_feature_view, RequestSource
from feast.types import Float32, Int64

# Define request data schema
request_source = RequestSource(
    name="transaction_request",
    schema=[
        Field(name="transaction_amount", dtype=Float32),
        Field(name="hour_of_day", dtype=Int64),
    ]
)

@on_demand_feature_view(
    sources=[request_source, user_statistics],
    schema=[
        Field(name="amount_vs_avg_ratio", dtype=Float32),
        Field(name="is_night_transaction", dtype=Int64),
    ]
)
def transaction_derived_features(inputs: pd.DataFrame) -> pd.DataFrame:
    """
    Compute on-demand features from request + stored features.
    Executed at serving time (<1ms overhead).
    """
    df = pd.DataFrame()

    # Ratio of current amount to user's 30d average
    df["amount_vs_avg_ratio"] = (
        inputs["transaction_amount"] / inputs["avg_transaction_amount"]
    )

    # Is this a night transaction? (10 PM - 6 AM)
    df["is_night_transaction"] = (
        (inputs["hour_of_day"] >= 22) | (inputs["hour_of_day"] <= 6)
    ).astype(int)

    return df

# Usage in serving
features = store.get_online_features(
    features=[
        "user_statistics:avg_transaction_amount",  # From online store
        "transaction_derived_features:amount_vs_avg_ratio",  # Computed on-demand
        "transaction_derived_features:is_night_transaction"
    ],
    entity_rows=[{"user_id": "user_123"}],
    request_data={
        "transaction_amount": [500.0],
        "hour_of_day": [23]
    }
).to_dict()
```

### 3. Feature Store High Availability

Production-grade deployment with redundancy:

```yaml
# Kubernetes deployment for Feast serving
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feast-serving
spec:
  replicas: 5  # High availability
  selector:
    matchLabels:
      app: feast-serving
  template:
    metadata:
      labels:
        app: feast-serving
    spec:
      containers:
      - name: feast-serving
        image: feast/feature-server:latest
        ports:
        - containerPort: 6566
        env:
        - name: FEAST_REDIS_HOST
          value: "redis-cluster.default.svc.cluster.local"
        - name: FEAST_REDIS_PORT
          value: "6379"
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        livenessProbe:
          httpGet:
            path: /health
            port: 6566
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 6566
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Redis cluster for high availability
apiVersion: redis.redis.opstreelabs.in/v1beta1
kind: RedisCluster
metadata:
  name: feast-redis
spec:
  clusterSize: 6
  kubernetesConfig:
    image: redis:7.0
  redisExporter:
    enabled: true
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi
```

---

## Real-World Applications

### Case Study 1: Uber Michelangelo Feature Store

**Scale**:
- 10,000+ features
- 1,000+ ML models
- 100+ data sources
- 10M+ predictions/second

**Architecture**:
```
Data Sources → Feature Jobs (Spark) → Cassandra (Online) + Hive (Offline) → Models
                     ↓
              Feature Registry (Metadata)
```

**Key Features**:
1. **Automatic backfill**: Historical feature computation for new features
2. **Multi-environment**: Dev, staging, prod feature isolation
3. **Monitoring**: Feature drift detection, data quality checks
4. **Governance**: Feature approval workflow, access controls

**Results**:
- 90% reduction in feature engineering time
- 70% feature reuse rate across models
- 99.99% serving availability

### Case Study 2: DoorDash Feature Store

**Problem**: Predicting delivery times requires real-time + historical features

**Solution**:
```
Online Features (Redis):              Offline Features (Snowflake):
- dasher_current_location             - dasher_30d_avg_delivery_time
- restaurant_current_queue_size       - restaurant_historical_prep_time
- dasher_5m_delivery_count            - neighborhood_traffic_patterns
```

**Implementation**:
- Feast for feature management
- Flink for streaming features
- Snowflake for batch features
- Redis for online serving

**Results**:
- Delivery time prediction accuracy: 85% → 92%
- Feature serving latency: <5ms (p99)
- Feature reuse: 60% across 15 models

### Case Study 3: Netflix Feature Store

**Challenge**: Personalization across 200M+ users requires consistent features

**Architecture**:
- Offline: S3 + Spark for batch feature computation
- Online: EVCache (distributed memcached) for low-latency serving
- Registry: Internal metadata service

**Feature Types**:
1. **User features**: Viewing history, preferences, engagement patterns
2. **Content features**: Genre, cast, popularity trends
3. **Context features**: Device type, time of day, location

**Scale**:
- 50+ TB feature data
- 1M+ feature requests/second
- <10ms serving latency (p99)

---

## Assessment

### Knowledge Check (10 Questions)

1. **What is the primary difference between online and offline feature stores?**
   - A) Online stores use SQL, offline stores use NoSQL
   - B) Online stores optimize for latency, offline stores for throughput
   - C) Online stores are more expensive
   - D) There is no difference

   <details><summary>Answer</summary>B) Online stores optimize for low-latency point lookups (<10ms), while offline stores optimize for high-throughput analytical queries (seconds-minutes)</details>

2. **Why is point-in-time correctness critical for training data?**
   - A) It makes queries faster
   - B) It prevents data leakage by ensuring features use only historical data
   - C) It reduces storage costs
   - D) It's not actually important

   <details><summary>Answer</summary>B) Point-in-time correctness prevents data leakage by ensuring that for each training example at time T, only features available before or at time T are used. This prevents the model from learning from future information.</details>

3. **Which storage backend is typically used for online feature serving?**
   - A) PostgreSQL
   - B) Redis or DynamoDB (key-value stores)
   - C) S3 (object storage)
   - D) Local filesystem

   <details><summary>Answer</summary>B) Key-value stores like Redis or DynamoDB are used for online serving because they provide <10ms latency for point lookups by primary key.</details>

4. **What is the purpose of feature materialization?**
   - A) To delete old features
   - B) To copy features from offline to online store for low-latency serving
   - C) To compress features
   - D) To encrypt features

   <details><summary>Answer</summary>B) Materialization copies computed feature values from the offline store (warehouse) to the online store (Redis) to enable low-latency real-time serving.</details>

5. **When should you use on-demand feature views?**
   - A) For all features
   - B) For lightweight transforms computed at request time
   - C) For complex aggregations
   - D) Never, they're deprecated

   <details><summary>Answer</summary>B) On-demand features are for lightweight transforms (e.g., ratios, time-of-day) that can be computed at request time with minimal latency overhead (<1ms).</details>

6. **What problem does feature versioning solve?**
   - A) Storage optimization
   - B) Reproducibility and safe feature evolution
   - C) Query performance
   - D) User authentication

   <details><summary>Answer</summary>B) Feature versioning enables reproducibility (train/serve with same feature definitions) and safe evolution (roll out new features without breaking existing models).</details>

7. **What is training-serving skew?**
   - A) When models are trained slowly
   - B) When training and inference use different feature implementations
   - C) When servers are overloaded
   - D) When data is encrypted

   <details><summary>Answer</summary>B) Training-serving skew occurs when features are computed differently in training (e.g., SQL in warehouse) vs inference (e.g., Python in API), leading to model degradation in production.</details>

8. **What is the recommended latency target for online feature serving?**
   - A) <1ms
   - B) <10ms
   - C) <100ms
   - D) <1 second

   <details><summary>Answer</summary>B) <10ms is the standard target for p99 latency in online feature serving to support real-time inference without adding significant overhead.</details>

9. **Which tool is the leading open-source feature store?**
   - A) TensorFlow
   - B) Feast
   - C) Docker
   - D) Kubernetes

   <details><summary>Answer</summary>B) Feast is the most widely adopted open-source feature store, originally developed by Gojek and now maintained by Tecton.</details>

10. **What should you monitor in a feature store?**
    - A) Feature drift, data quality, serving latency
    - B) Only serving latency
    - C) Only storage costs
    - D) Nothing, it's automatic

    <details><summary>Answer</summary>A) Comprehensive monitoring includes feature drift (distribution changes), data quality (null rates, schema violations), and serving latency (p50/p95/p99)</details>

### Practical Challenge

**Build a Complete Feature Store Pipeline**:

1. Define a feature store with 3 entities (user, product, merchant)
2. Create 5+ feature views with different aggregation windows
3. Generate point-in-time correct training data
4. Materialize features to online store (Redis)
5. Build a FastAPI serving endpoint with <15ms latency
6. Implement feature drift monitoring

**Success Criteria**:
- Point-in-time joins produce no data leakage
- Online serving latency <15ms (p95)
- Materialization completes in <5 minutes
- Drift detection correctly identifies distribution changes

---

## Summary

In this lecture, you learned:

✅ **Feature Store Fundamentals**: Purpose, architecture, and key components
✅ **Online vs Offline Stores**: Latency, storage, and access pattern differences
✅ **Point-in-Time Correctness**: Preventing data leakage in training data
✅ **Feast Implementation**: Setup, feature definition, and serving
✅ **Feature Materialization**: Copying features from offline to online stores
✅ **Production Patterns**: Versioning, monitoring, high availability
✅ **Real-World Applications**: Uber, DoorDash, Netflix feature stores

**Key Takeaways**:

1. **Training-serving consistency is critical**: Use the same feature store for both to eliminate skew
2. **Point-in-time joins prevent leakage**: Always use temporal correctness in training data
3. **Dual storage architecture**: Offline (throughput) + Online (latency) = complete solution
4. **Feature reuse drives efficiency**: 60-90% feature reuse across models in mature platforms
5. **Monitor feature health**: Drift detection and data quality checks are essential

**Next Steps**: Module 05 - Workflow Orchestration

---

**Resources**:
- [Feast Documentation](https://docs.feast.dev/)
- [Tecton Feature Store](https://www.tecton.ai/)
- [Uber Michelangelo](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
- [Feature Store Summit Talks](https://www.featurestoresummit.com/)

*Last updated: November 2, 2025 | Version 1.0*
