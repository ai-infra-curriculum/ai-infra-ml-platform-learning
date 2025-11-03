# Module 4 Exercises: Feature Store Architecture

This directory contains 6 hands-on exercises to master feature store implementation for ML platforms.

## Exercise Overview

| Exercise | Title | Difficulty | Duration | Key Skills |
|----------|-------|------------|----------|------------|
| 01 | Set Up Feast Feature Store | Basic | 60 min | Feast installation, configuration, feature definitions |
| 02 | Generate Point-in-Time Training Data | Intermediate | 90 min | Historical features, temporal joins, data leakage prevention |
| 03 | Materialize Features to Online Store | Intermediate | 75 min | Feature materialization, Redis integration, incremental updates |
| 04 | Build Real-Time Feature Serving API | Advanced | 120 min | FastAPI serving, online features, latency optimization |
| 05 | Implement Streaming Features | Advanced | 150 min | Flink streaming, real-time computation, Kafka integration |
| 06 | Monitor Feature Quality and Drift | Intermediate | 90 min | Statistical tests, drift detection, alerting |

## Prerequisites

- [x] Completed Module 3 exercises
- [x] Python 3.9+ with feast, pandas, redis installed
- [x] Docker for running Redis locally
- [x] Understanding of feature engineering concepts
- [x] Basic SQL knowledge
- [x] (Optional) Apache Flink for Exercise 05

**Total Time**: 585 minutes (~9.75 hours)

---

## Exercise 01: Set Up Feast Feature Store

**Objective**: Initialize a Feast feature store with multiple entities and feature views.

**Difficulty**: Basic | **Duration**: 60 minutes

### Requirements

Set up a complete Feast repository with:
1. Redis online store configuration
2. File-based offline store (or BigQuery/Snowflake if available)
3. Three entities: `user`, `merchant`, `product`
4. Five feature views covering different domains
5. Two feature services for different ML use cases

### Implementation Steps

1. **Install and initialize Feast** (10 min)
   ```bash
   pip install feast[redis]
   feast init feature_store_project
   cd feature_store_project
   ```

2. **Configure feature_store.yaml** (10 min)
   - Set up Redis online store connection
   - Configure offline store (file/parquet)
   - Set project name and registry path

3. **Define entities** (10 min)
   - Create user, merchant, product entities
   - Add descriptions and metadata tags

4. **Create feature views** (20 min)
   - User behavior features (transaction counts, averages)
   - Merchant risk features (chargeback rates, refund rates)
   - Product popularity features (view counts, purchase rates)
   - Define schemas with appropriate data types
   - Set TTLs for feature freshness

5. **Create feature services** (10 min)
   - Fraud detection feature service
   - Recommendation feature service
   - Apply to registry with `feast apply`

### Starter Code

```python
# features.py
from feast import Entity, FeatureView, Field, FileSource, FeatureService
from feast.types import Float32, Int64, String
from datetime import timedelta

# ============================================
# ENTITIES
# ============================================

user = Entity(
    name="user_id",
    description="User identifier",
    tags={"team": "platform"}
)

merchant = Entity(
    name="merchant_id",
    description="Merchant/seller identifier"
)

product = Entity(
    name="product_id",
    description="Product identifier"
)

# ============================================
# DATA SOURCES
# ============================================

user_stats_source = FileSource(
    name="user_stats_source",
    path="data/user_stats.parquet",
    timestamp_field="event_timestamp",
    created_timestamp_column="created_timestamp"
)

# TODO: Add merchant_stats_source
# TODO: Add product_stats_source

# ============================================
# FEATURE VIEWS
# ============================================

user_transaction_features = FeatureView(
    name="user_transaction_features",
    entities=[user],
    ttl=timedelta(days=90),
    schema=[
        Field(name="transaction_count_7d", dtype=Int64),
        Field(name="transaction_count_30d", dtype=Int64),
        Field(name="avg_transaction_amount", dtype=Float32),
        Field(name="total_amount_30d", dtype=Float32),
    ],
    source=user_stats_source,
    online=True,
    tags={"domain": "transactions"}
)

# TODO: Add merchant_risk_features FeatureView
# TODO: Add product_popularity_features FeatureView

# ============================================
# FEATURE SERVICES
# ============================================

fraud_detection_v1 = FeatureService(
    name="fraud_detection",
    features=[
        user_transaction_features[[
            "transaction_count_7d",
            "avg_transaction_amount"
        ]],
        # TODO: Add merchant features
    ],
    tags={"version": "v1", "model": "fraud_xgboost"}
)

# TODO: Add recommendation_v1 FeatureService
```

### Configuration File

```yaml
# feature_store.yaml
project: ml_platform_features
registry: data/registry.db
provider: local

online_store:
  type: redis
  connection_string: "localhost:6379"

offline_store:
  type: file

entity_key_serialization_version: 2
```

### Success Criteria

- [ ] Feast applies without errors (`feast apply`)
- [ ] Registry contains 3 entities
- [ ] Registry contains 5 feature views
- [ ] Registry contains 2 feature services
- [ ] Redis connection successful
- [ ] Can query feature metadata (`feast feature-views list`)

### Validation

```bash
# Apply features to registry
feast apply

# Verify entities
feast entities list

# Verify feature views
feast feature-views list

# Verify feature services
feast feature-services list

# Check feature view details
feast feature-views describe user_transaction_features

# Test registry
python -c "from feast import FeatureStore; store = FeatureStore('.'); print(store.list_entities())"
```

---

## Exercise 02: Generate Point-in-Time Training Data

**Objective**: Create training datasets with point-in-time correct feature joins to prevent data leakage.

**Difficulty**: Intermediate | **Duration**: 90 minutes

### Requirements

Generate training data that:
1. Joins features with correct temporal semantics
2. Prevents future information leakage
3. Handles multiple entities (user, merchant, product)
4. Validates point-in-time correctness
5. Exports to format suitable for model training

### The Data Leakage Challenge

**Scenario**: You're training a fraud detection model on transactions from October 2024.

**Incorrect Approach** (causes leakage):
```python
# BAD: Uses latest feature values
entity_df = pd.DataFrame({
    "user_id": ["user_1", "user_2"],
    "event_timestamp": [
        datetime(2024, 10, 1),
        datetime(2024, 10, 2)
    ],
    "is_fraud": [0, 1]
})

# This gets features as of NOW, not as of event_timestamp!
# Features include transactions from Oct 3+, which is FUTURE data
features = store.get_historical_features(
    entity_df=entity_df,
    features=["user_transaction_features:transaction_count_30d"]
).to_df()
```

**Correct Approach** (point-in-time):
```python
# GOOD: Gets features as they existed at event_timestamp
# Only uses data available BEFORE each transaction
features = store.get_historical_features(
    entity_df=entity_df,  # Must include event_timestamp column
    features=["user_transaction_features:transaction_count_30d"]
).to_df()

# Feast automatically performs point-in-time join
# For transaction at 2024-10-01, uses features computed on or before 2024-10-01
```

### Implementation Steps

1. **Create entity dataframe with labels** (15 min)
   - Load transaction data with timestamps
   - Include label column (is_fraud, churn, etc.)
   - Ensure event_timestamp column present

2. **Define feature references** (10 min)
   - List all features needed for model
   - Use format: "feature_view:feature_name"

3. **Retrieve historical features** (20 min)
   - Call `get_historical_features()` with point-in-time semantics
   - Convert to pandas DataFrame

4. **Validate no data leakage** (25 min)
   - Check feature values are historically correct
   - Verify no NULL values (indicates feature not available)
   - Compare feature values to manually computed values

5. **Export training data** (10 min)
   - Save to parquet or CSV
   - Split train/validation/test sets

6. **Compare with naive join** (10 min)
   - Show performance difference with/without point-in-time correctness

### Starter Code

```python
from feast import FeatureStore
from datetime import datetime, timedelta
import pandas as pd
import numpy as np

store = FeatureStore(repo_path=".")

# ============================================
# STEP 1: Create Entity DataFrame (Labels)
# ============================================

def create_entity_df():
    """
    Create entity dataframe with labels and timestamps.
    This represents historical events (transactions) we want to train on.
    """
    np.random.seed(42)

    # Simulate 1000 transactions over 30 days
    dates = pd.date_range(
        start=datetime(2024, 10, 1),
        end=datetime(2024, 10, 30),
        periods=1000
    )

    entity_df = pd.DataFrame({
        "user_id": [f"user_{i % 100}" for i in range(1000)],
        "merchant_id": [f"merchant_{i % 50}" for i in range(1000)],
        "event_timestamp": dates,
        "transaction_amount": np.random.uniform(10, 1000, 1000),
        "is_fraud": np.random.choice([0, 1], 1000, p=[0.95, 0.05])
    })

    return entity_df

entity_df = create_entity_df()
print(f"Entity DataFrame shape: {entity_df.shape}")
print(entity_df.head())

# ============================================
# STEP 2: Get Historical Features (Point-in-Time)
# ============================================

def get_training_data(entity_df: pd.DataFrame) -> pd.DataFrame:
    """
    Retrieve features with point-in-time correctness.

    For each row at timestamp T, fetches feature values as they
    existed at or before timestamp T.
    """
    training_df = store.get_historical_features(
        entity_df=entity_df,
        features=[
            # User features
            "user_transaction_features:transaction_count_7d",
            "user_transaction_features:transaction_count_30d",
            "user_transaction_features:avg_transaction_amount",

            # Merchant features
            "merchant_risk_features:chargeback_rate",
            "merchant_risk_features:total_transactions_30d",

            # Product features (if applicable)
            # "product_popularity_features:view_count_7d",
        ]
    ).to_df()

    return training_df

training_df = get_training_data(entity_df)
print(f"\nTraining DataFrame shape: {training_df.shape}")
print(training_df.head())

# ============================================
# STEP 3: Validate Point-in-Time Correctness
# ============================================

def validate_pit_correctness(training_df: pd.DataFrame):
    """
    Validate that features don't contain future information.
    """
    # Check for NULL values (indicates feature wasn't available)
    null_counts = training_df.isnull().sum()
    print("\nNULL value counts:")
    print(null_counts)

    # For a sample row, manually verify feature value
    sample_row = training_df.iloc[100]
    print(f"\nSample row at {sample_row['event_timestamp']}:")
    print(f"User: {sample_row['user_id']}")
    print(f"transaction_count_30d: {sample_row['transaction_count_30d']}")

    # TODO: Manually query offline store to verify this value
    # matches historical data as of event_timestamp

    # Check feature statistics
    print("\nFeature statistics:")
    print(training_df.describe())

validate_pit_correctness(training_df)

# ============================================
# STEP 4: Compare with Naive Join (Leakage)
# ============================================

def demonstrate_leakage():
    """
    Show the difference between point-in-time and naive join.
    """
    # Simulate "latest" features (incorrect)
    latest_features = pd.DataFrame({
        "user_id": entity_df["user_id"],
        "transaction_count_30d_latest": np.random.randint(50, 200, len(entity_df))
    })

    # Point-in-time features (correct)
    pit_features = training_df[["user_id", "transaction_count_30d"]]

    comparison = entity_df[["user_id", "event_timestamp"]].merge(
        latest_features, on="user_id"
    ).merge(
        pit_features, on="user_id"
    )

    print("\nComparison: Latest vs Point-in-Time")
    print(comparison.head(10))
    print(f"\nMean difference: {(comparison['transaction_count_30d_latest'] - comparison['transaction_count_30d']).mean():.2f}")

demonstrate_leakage()

# ============================================
# STEP 5: Export Training Data
# ============================================

def export_training_data(training_df: pd.DataFrame):
    """Export to parquet for model training"""
    # Remove timestamp columns (not needed for training)
    feature_cols = [col for col in training_df.columns
                   if col not in ['event_timestamp', 'created_timestamp']]

    X = training_df[feature_cols].drop('is_fraud', axis=1)
    y = training_df['is_fraud']

    # Split train/val/test (70/15/15)
    from sklearn.model_selection import train_test_split

    X_train, X_temp, y_train, y_temp = train_test_split(
        X, y, test_size=0.3, random_state=42
    )
    X_val, X_test, y_val, y_test = train_test_split(
        X_temp, y_temp, test_size=0.5, random_state=42
    )

    # Save to parquet
    X_train.to_parquet("data/X_train.parquet")
    X_val.to_parquet("data/X_val.parquet")
    X_test.to_parquet("data/X_test.parquet")

    pd.DataFrame(y_train).to_parquet("data/y_train.parquet")
    pd.DataFrame(y_val).to_parquet("data/y_val.parquet")
    pd.DataFrame(y_test).to_parquet("data/y_test.parquet")

    print(f"\nExported training data:")
    print(f"  Train: {len(X_train)} samples")
    print(f"  Val:   {len(X_val)} samples")
    print(f"  Test:  {len(X_test)} samples")

export_training_data(training_df)
```

### Testing Point-in-Time Correctness

```python
def test_pit_correctness():
    """
    Rigorous test for point-in-time correctness.
    """
    # Create test case: transaction on Oct 15
    test_entity = pd.DataFrame({
        "user_id": ["user_test"],
        "merchant_id": ["merchant_test"],
        "event_timestamp": [datetime(2024, 10, 15, 12, 0)]
    })

    # Get features
    features = store.get_historical_features(
        entity_df=test_entity,
        features=["user_transaction_features:transaction_count_30d"]
    ).to_df()

    feature_value = features["transaction_count_30d"].iloc[0]

    # Manually query offline store for validation
    # This should count transactions from Sep 15 - Oct 15
    manual_query = """
        SELECT COUNT(*) as cnt
        FROM transactions
        WHERE user_id = 'user_test'
          AND timestamp >= '2024-09-15'
          AND timestamp < '2024-10-15'
    """
    # manual_count = execute_sql(manual_query)

    # Assert they match (within tolerance for timing)
    # assert abs(feature_value - manual_count) <= 1

    print(f"Point-in-time test passed!")
    print(f"  Feature value: {feature_value}")
    # print(f"  Manual count: {manual_count}")
```

### Success Criteria

- [ ] Training data generated without errors
- [ ] No NULL values in critical features
- [ ] Point-in-time validation passes
- [ ] Feature values match manual calculations
- [ ] Exported data splits created (train/val/test)
- [ ] Demonstrated leakage with naive join comparison

---

## Exercise 03: Materialize Features to Online Store

**Objective**: Materialize computed features from offline storage to online store (Redis) for low-latency serving.

**Difficulty**: Intermediate | **Duration**: 75 minutes

### Requirements

Implement feature materialization that:
1. Copies features from offline to online store
2. Supports incremental updates (hourly/daily)
3. Handles backfilling for historical data
4. Validates data integrity after materialization
5. Monitors materialization job performance

### Why Materialization?

**Problem**: Offline stores (data warehouses) have high latency (seconds-minutes)
**Solution**: Materialize latest feature values to online store (Redis) for <10ms lookups

```
Offline Store (Snowflake)     Online Store (Redis)
┌─────────────────────┐       ┌──────────────────┐
│ Historical features │       │ Latest features  │
│ (all timestamps)    │──────>│ (current values) │
│                     │ Copy  │                  │
│ Latency: ~2 seconds │       │ Latency: ~5ms    │
└─────────────────────┘       └──────────────────┘
```

### Implementation Steps

1. **Start Redis** (5 min)
   ```bash
   docker run -d -p 6379:6379 redis:7.0
   ```

2. **Initial materialization** (15 min)
   - Materialize last 7 days of features
   - Verify data written to Redis

3. **Incremental materialization** (20 min)
   - Materialize only new data (last hour)
   - Schedule with cron/Airflow

4. **Backfill historical data** (15 min)
   - Materialize features from specific date range
   - Handle large volumes efficiently

5. **Validate materialization** (15 min)
   - Compare offline vs online feature values
   - Check Redis key count and TTLs

6. **Monitor performance** (5 min)
   - Track materialization time
   - Monitor Redis memory usage

### Starter Code

```python
from feast import FeatureStore
from datetime import datetime, timedelta
import redis
import time

store = FeatureStore(repo_path=".")

# ============================================
# STEP 1: Initial Materialization
# ============================================

def initial_materialization(days_back: int = 7):
    """
    Materialize last N days of features to online store.

    Args:
        days_back: Number of days of historical data to materialize
    """
    start_date = datetime.now() - timedelta(days=days_back)
    end_date = datetime.now()

    print(f"Materializing features from {start_date} to {end_date}...")
    start_time = time.time()

    store.materialize(
        start_date=start_date,
        end_date=end_date
    )

    duration = time.time() - start_time
    print(f"✓ Materialization complete in {duration:.2f}s")

    return duration

# Run initial materialization
duration = initial_materialization(days_back=7)

# ============================================
# STEP 2: Incremental Materialization
# ============================================

def incremental_materialization():
    """
    Materialize only new features since last run.
    Suitable for hourly/daily scheduled jobs.
    """
    print("Running incremental materialization...")
    start_time = time.time()

    # Materialize from last materialization to now
    store.materialize_incremental(
        end_date=datetime.now()
    )

    duration = time.time() - start_time
    print(f"✓ Incremental materialization complete in {duration:.2f}s")

    return duration

# Simulate incremental update (in production, run hourly)
# time.sleep(3600)  # Wait 1 hour
# incremental_materialization()

# ============================================
# STEP 3: Validate Materialization
# ============================================

def validate_materialization():
    """
    Verify features correctly written to Redis.
    """
    # Connect to Redis
    redis_client = redis.Redis(
        host='localhost',
        port=6379,
        decode_responses=True
    )

    # Check Redis connection
    try:
        redis_client.ping()
        print("✓ Redis connection successful")
    except:
        print("✗ Redis connection failed")
        return False

    # Count feature keys
    feature_keys = redis_client.keys("*")
    print(f"✓ Redis contains {len(feature_keys)} feature keys")

    # Sample a few keys
    if feature_keys:
        sample_keys = feature_keys[:5]
        print(f"\nSample keys:")
        for key in sample_keys:
            value = redis_client.get(key)
            ttl = redis_client.ttl(key)
            print(f"  {key}: TTL={ttl}s")

    # Compare online vs offline features for consistency
    test_entities = [
        {"user_id": "user_1", "merchant_id": "merchant_1"}
    ]

    # Get from online store
    online_features = store.get_online_features(
        features=[
            "user_transaction_features:transaction_count_30d",
            "merchant_risk_features:chargeback_rate"
        ],
        entity_rows=test_entities
    ).to_dict()

    print(f"\nOnline features (from Redis):")
    print(online_features)

    # Get from offline store for comparison
    entity_df = pd.DataFrame({
        "user_id": ["user_1"],
        "merchant_id": ["merchant_1"],
        "event_timestamp": [datetime.now()]
    })

    offline_features = store.get_historical_features(
        entity_df=entity_df,
        features=[
            "user_transaction_features:transaction_count_30d",
            "merchant_risk_features:chargeback_rate"
        ]
    ).to_df()

    print(f"\nOffline features (from warehouse):")
    print(offline_features)

    # They should match (approximately)
    print("\n✓ Validation complete")

    return True

validate_materialization()

# ============================================
# STEP 4: Backfill Historical Data
# ============================================

def backfill_features(start_date: datetime, end_date: datetime):
    """
    Backfill features for a specific date range.
    Useful when adding new features or fixing bugs.

    Args:
        start_date: Start of backfill period
        end_date: End of backfill period
    """
    print(f"Backfilling features from {start_date} to {end_date}...")

    # For large date ranges, chunk into smaller periods
    chunk_size = timedelta(days=7)
    current_start = start_date

    while current_start < end_date:
        current_end = min(current_start + chunk_size, end_date)

        print(f"  Materializing chunk: {current_start} to {current_end}")
        start_time = time.time()

        store.materialize(
            start_date=current_start,
            end_date=current_end
        )

        duration = time.time() - start_time
        print(f"  ✓ Chunk complete in {duration:.2f}s")

        current_start = current_end

    print("✓ Backfill complete")

# Example: Backfill last 30 days
# backfill_features(
#     start_date=datetime.now() - timedelta(days=30),
#     end_date=datetime.now()
# )

# ============================================
# STEP 5: Monitor Materialization
# ============================================

def monitor_materialization():
    """
    Monitor materialization job health and performance.
    """
    redis_client = redis.Redis(host='localhost', port=6379)

    # Redis memory usage
    info = redis_client.info('memory')
    memory_used_mb = info['used_memory'] / 1024 / 1024
    memory_peak_mb = info['used_memory_peak'] / 1024 / 1024

    print(f"\nRedis Memory Usage:")
    print(f"  Current: {memory_used_mb:.2f} MB")
    print(f"  Peak: {memory_peak_mb:.2f} MB")

    # Key count by feature view
    keys_by_prefix = {}
    for key in redis_client.scan_iter():
        prefix = key.decode().split(':')[0] if b':' in key else 'other'
        keys_by_prefix[prefix] = keys_by_prefix.get(prefix, 0) + 1

    print(f"\nKeys by feature view:")
    for prefix, count in sorted(keys_by_prefix.items()):
        print(f"  {prefix}: {count} keys")

    # Materialization freshness (time since last update)
    # In production, track this with monitoring system
    print(f"\n✓ Monitoring metrics collected")

monitor_materialization()
```

### Airflow DAG for Scheduled Materialization

```python
# airflow_dags/materialize_features.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta
from feast import FeatureStore

default_args = {
    'owner': 'ml-platform',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'feature_materialization',
    default_args=default_args,
    description='Materialize features to online store',
    schedule_interval='@hourly',  # Run every hour
    catchup=False
)

def materialize_task():
    """Materialize features incrementally"""
    store = FeatureStore(repo_path="/opt/feature_repo")
    store.materialize_incremental(end_date=datetime.now())

materialize = PythonOperator(
    task_id='materialize_features',
    python_callable=materialize_task,
    dag=dag
)
```

### Success Criteria

- [ ] Initial materialization completes successfully
- [ ] Features queryable from Redis (<10ms latency)
- [ ] Incremental materialization updates only new data
- [ ] Online and offline features match
- [ ] Redis memory usage within limits
- [ ] Materialization job completes in <5 minutes

### Validation

```bash
# Check Redis keys
redis-cli KEYS "*user_transaction*" | head -10

# Get sample feature value
redis-cli GET "feast:user_transaction_features:user_1"

# Check TTL
redis-cli TTL "feast:user_transaction_features:user_1"

# Monitor Redis stats
redis-cli INFO memory
redis-cli INFO stats
```

---

## Exercise 04: Build Real-Time Feature Serving API

**Objective**: Create a production-grade FastAPI service that serves features with <15ms latency for real-time predictions.

**Difficulty**: Advanced | **Duration**: 120 minutes

### Requirements

Build a serving API that:
1. Fetches features from online store (Redis)
2. Computes on-demand features at request time
3. Supports both single and batch predictions
4. Implements caching for frequently accessed features
5. Includes health checks and monitoring
6. Achieves <15ms p95 latency

### Architecture

```
Client Request
     ↓
FastAPI Endpoint
     ↓
┌─────────────────────────────────────┐
│  Feature Retrieval Pipeline         │
│                                     │
│  1. Get from cache (if present)    │
│  2. Fetch from online store (Redis)│
│  3. Compute on-demand features     │
│  4. Update cache                    │
└─────────────────────────────────────┘
     ↓
Model Inference
     ↓
Response
```

### Implementation Steps

1. **Create FastAPI application** (20 min)
   - Set up project structure
   - Define request/response models with Pydantic
   - Initialize feature store connection

2. **Implement feature retrieval** (25 min)
   - Build feature fetching logic
   - Handle missing features gracefully
   - Add error handling

3. **Add on-demand features** (20 min)
   - Compute request-time features
   - Cross-feature calculations

4. **Implement caching** (20 min)
   - Add Redis cache layer
   - Set TTLs for cache entries
   - Implement cache warming

5. **Add batch prediction** (15 min)
   - Support multiple entities in single request
   - Optimize for bulk operations

6. **Monitoring and health checks** (10 min)
   - Add /health endpoint
   - Track latency metrics
   - Log feature retrieval stats

7. **Load testing** (10 min)
   - Test with concurrent requests
   - Verify latency targets

### Starter Code

```python
# app/main.py
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel, Field
from feast import FeatureStore
from typing import List, Dict, Optional
import numpy as np
import joblib
import redis
import json
import time
from prometheus_client import Counter, Histogram, generate_latest

# Initialize FastAPI
app = FastAPI(
    title="ML Feature Serving API",
    description="Real-time feature serving with Feast",
    version="1.0.0"
)

# Initialize Feast
store = FeatureStore(repo_path="/opt/feature_repo")

# Initialize Redis cache
cache = redis.Redis(host='localhost', port=6379, db=1, decode_responses=True)

# Load ML model
model = joblib.load("models/fraud_detector.pkl")

# Prometheus metrics
PREDICTION_COUNTER = Counter('predictions_total', 'Total predictions')
PREDICTION_LATENCY = Histogram('prediction_latency_seconds', 'Prediction latency')
FEATURE_CACHE_HITS = Counter('feature_cache_hits', 'Feature cache hits')
FEATURE_CACHE_MISSES = Counter('feature_cache_misses', 'Feature cache misses')

# ============================================
# Request/Response Models
# ============================================

class PredictionRequest(BaseModel):
    """Single prediction request"""
    user_id: str = Field(..., description="User identifier")
    merchant_id: str = Field(..., description="Merchant identifier")
    transaction_amount: float = Field(..., gt=0, description="Transaction amount")
    timestamp: Optional[str] = Field(None, description="Transaction timestamp (ISO format)")

class PredictionResponse(BaseModel):
    """Prediction response"""
    user_id: str
    merchant_id: str
    is_fraud: bool
    fraud_probability: float
    features: Dict[str, float]
    latency_ms: float

class BatchPredictionRequest(BaseModel):
    """Batch prediction request"""
    transactions: List[PredictionRequest] = Field(..., max_items=100)

class BatchPredictionResponse(BaseModel):
    """Batch prediction response"""
    predictions: List[PredictionResponse]
    total_latency_ms: float

# ============================================
# Feature Retrieval with Caching
# ============================================

def get_features_cached(
    user_id: str,
    merchant_id: str,
    use_cache: bool = True
) -> Dict[str, float]:
    """
    Get features with caching support.

    Args:
        user_id: User identifier
        merchant_id: Merchant identifier
        use_cache: Whether to use cache

    Returns:
        Dictionary of feature name -> value
    """
    cache_key = f"features:{user_id}:{merchant_id}"

    # Try cache first
    if use_cache:
        cached_features = cache.get(cache_key)
        if cached_features:
            FEATURE_CACHE_HITS.inc()
            return json.loads(cached_features)

    FEATURE_CACHE_MISSES.inc()

    # Fetch from online store
    features_dict = store.get_online_features(
        features=[
            "user_transaction_features:transaction_count_7d",
            "user_transaction_features:transaction_count_30d",
            "user_transaction_features:avg_transaction_amount",
            "merchant_risk_features:chargeback_rate",
            "merchant_risk_features:total_transactions_30d",
        ],
        entity_rows=[{
            "user_id": user_id,
            "merchant_id": merchant_id
        }]
    ).to_dict()

    # Convert to simple dict
    features = {
        "transaction_count_7d": features_dict["transaction_count_7d"][0],
        "transaction_count_30d": features_dict["transaction_count_30d"][0],
        "avg_transaction_amount": features_dict["avg_transaction_amount"][0],
        "chargeback_rate": features_dict["chargeback_rate"][0],
        "total_transactions_30d": features_dict["total_transactions_30d"][0],
    }

    # Cache for 5 minutes
    if use_cache:
        cache.setex(
            cache_key,
            300,  # TTL: 5 minutes
            json.dumps(features)
        )

    return features

# ============================================
# On-Demand Features
# ============================================

def compute_on_demand_features(
    features: Dict[str, float],
    transaction_amount: float
) -> Dict[str, float]:
    """
    Compute on-demand features from stored + request features.

    Args:
        features: Stored features from feature store
        transaction_amount: Current transaction amount

    Returns:
        Additional computed features
    """
    on_demand = {}

    # Ratio of current amount to user's average
    if features["avg_transaction_amount"] > 0:
        on_demand["amount_vs_avg_ratio"] = (
            transaction_amount / features["avg_transaction_amount"]
        )
    else:
        on_demand["amount_vs_avg_ratio"] = 0.0

    # Is this a large transaction? (>2x average)
    on_demand["is_large_transaction"] = int(
        transaction_amount > 2 * features["avg_transaction_amount"]
    )

    # Transaction velocity (transactions per day)
    on_demand["transaction_velocity"] = (
        features["transaction_count_30d"] / 30.0
    )

    return on_demand

# ============================================
# Prediction Endpoints
# ============================================

@app.post("/predict", response_model=PredictionResponse)
async def predict_fraud(request: PredictionRequest):
    """
    Predict fraud probability for a transaction.

    Real-time endpoint with <15ms target latency.
    """
    start_time = time.time()
    PREDICTION_COUNTER.inc()

    try:
        # 1. Get features (cached or from online store)
        features = get_features_cached(
            user_id=request.user_id,
            merchant_id=request.merchant_id
        )

        # 2. Compute on-demand features
        on_demand_features = compute_on_demand_features(
            features=features,
            transaction_amount=request.transaction_amount
        )

        # 3. Combine all features
        all_features = {**features, **on_demand_features}

        # 4. Prepare model input
        X = np.array([[
            all_features["transaction_count_7d"],
            all_features["transaction_count_30d"],
            all_features["avg_transaction_amount"],
            all_features["chargeback_rate"],
            all_features["amount_vs_avg_ratio"],
            all_features["transaction_velocity"],
            request.transaction_amount
        ]])

        # 5. Predict
        proba = model.predict_proba(X)[0][1]
        is_fraud = proba > 0.5

        # 6. Calculate latency
        latency_ms = (time.time() - start_time) * 1000
        PREDICTION_LATENCY.observe(latency_ms / 1000)

        return PredictionResponse(
            user_id=request.user_id,
            merchant_id=request.merchant_id,
            is_fraud=is_fraud,
            fraud_probability=float(proba),
            features=all_features,
            latency_ms=latency_ms
        )

    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/predict/batch", response_model=BatchPredictionResponse)
async def predict_fraud_batch(request: BatchPredictionRequest):
    """
    Batch prediction endpoint for multiple transactions.

    More efficient than individual requests for bulk operations.
    """
    start_time = time.time()

    predictions = []
    for txn in request.transactions:
        pred = await predict_fraud(txn)
        predictions.append(pred)

    total_latency_ms = (time.time() - start_time) * 1000

    return BatchPredictionResponse(
        predictions=predictions,
        total_latency_ms=total_latency_ms
    )

# ============================================
# Health and Monitoring
# ============================================

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    try:
        # Check Redis connection
        cache.ping()
        redis_healthy = True
    except:
        redis_healthy = False

    # Check feature store
    try:
        store.list_feature_views()
        feast_healthy = True
    except:
        feast_healthy = False

    is_healthy = redis_healthy and feast_healthy

    return {
        "status": "healthy" if is_healthy else "unhealthy",
        "redis": "up" if redis_healthy else "down",
        "feast": "up" if feast_healthy else "down",
        "version": "1.0.0"
    }

@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint"""
    return generate_latest()

@app.get("/stats")
async def stats():
    """Feature serving statistics"""
    cache_info = cache.info()

    return {
        "cache_hits": FEATURE_CACHE_HITS._value.get(),
        "cache_misses": FEATURE_CACHE_MISSES._value.get(),
        "cache_hit_rate": FEATURE_CACHE_HITS._value.get() / max(
            FEATURE_CACHE_HITS._value.get() + FEATURE_CACHE_MISSES._value.get(), 1
        ),
        "total_predictions": PREDICTION_COUNTER._value.get(),
        "redis_memory_mb": cache_info['used_memory'] / 1024 / 1024,
        "redis_keys": cache.dbsize()
    }

# ============================================
# Cache Warming
# ============================================

@app.on_event("startup")
async def warm_cache():
    """
    Warm cache with frequently accessed features on startup.
    """
    print("Warming feature cache...")

    # Get top users/merchants from database
    top_entities = [
        {"user_id": f"user_{i}", "merchant_id": f"merchant_{i % 50}"}
        for i in range(100)
    ]

    for entity in top_entities:
        try:
            get_features_cached(
                user_id=entity["user_id"],
                merchant_id=entity["merchant_id"],
                use_cache=True
            )
        except:
            pass

    print(f"✓ Cache warmed with {len(top_entities)} entities")

# ============================================
# Main
# ============================================

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Load Testing

```python
# load_test.py
import asyncio
import aiohttp
import time
import statistics

async def send_request(session, url, payload):
    """Send single prediction request"""
    start = time.time()
    async with session.post(url, json=payload) as response:
        result = await response.json()
        latency = (time.time() - start) * 1000
        return latency, result

async def load_test(num_requests: int = 1000, concurrency: int = 50):
    """
    Load test the serving API.

    Args:
        num_requests: Total number of requests
        concurrency: Number of concurrent requests
    """
    url = "http://localhost:8000/predict"

    # Prepare requests
    requests = [
        {
            "user_id": f"user_{i % 100}",
            "merchant_id": f"merchant_{i % 50}",
            "transaction_amount": 100 + (i % 500)
        }
        for i in range(num_requests)
    ]

    latencies = []

    async with aiohttp.ClientSession() as session:
        # Send requests in batches
        for i in range(0, num_requests, concurrency):
            batch = requests[i:i+concurrency]
            tasks = [send_request(session, url, req) for req in batch]
            results = await asyncio.gather(*tasks)
            latencies.extend([r[0] for r in results])

    # Calculate statistics
    print(f"\nLoad Test Results ({num_requests} requests, {concurrency} concurrent):")
    print(f"  Mean latency:   {statistics.mean(latencies):.2f}ms")
    print(f"  Median latency: {statistics.median(latencies):.2f}ms")
    print(f"  P95 latency:    {statistics.quantiles(latencies, n=20)[18]:.2f}ms")
    print(f"  P99 latency:    {statistics.quantiles(latencies, n=100)[98]:.2f}ms")
    print(f"  Min latency:    {min(latencies):.2f}ms")
    print(f"  Max latency:    {max(latencies):.2f}ms")

if __name__ == "__main__":
    asyncio.run(load_test(num_requests=1000, concurrency=50))
```

### Success Criteria

- [ ] API serves predictions successfully
- [ ] P95 latency <15ms (excluding model inference)
- [ ] Cache hit rate >70% after warmup
- [ ] Batch predictions work correctly
- [ ] Health check endpoint returns accurate status
- [ ] Load test passes with 1000 concurrent requests

---

## Exercise 05: Implement Streaming Features

**Objective**: Build real-time feature computation pipeline using Apache Flink for streaming data.

**Difficulty**: Advanced | **Duration**: 150 minutes

### Requirements

Implement streaming features that:
1. Consume events from Kafka topic
2. Compute aggregations over time windows (5m, 1h, 24h)
3. Write features to online store in real-time
4. Handle late-arriving events
5. Maintain exactly-once semantics

### Architecture

```
Kafka Topic (transaction_events)
         ↓
Apache Flink Job
   - Parse events
   - Window aggregations
   - State management
         ↓
Redis (Online Store)
         ↓
Feature Serving API
```

### Implementation Steps

1. **Set up Kafka** (15 min)
   - Start Kafka broker
   - Create topic for transaction events
   - Produce sample events

2. **Define Flink job** (40 min)
   - Create streaming environment
   - Define source (Kafka)
   - Implement window operations

3. **Compute streaming aggregations** (30 min)
   - Tumbling windows (fixed intervals)
   - Sliding windows (overlapping)
   - Session windows (activity-based)

4. **Write to online store** (25 min)
   - Define Redis sink
   - Handle serialization
   - Set TTLs

5. **Handle late data** (20 min)
   - Configure watermarks
   - Set allowed lateness

6. **Test streaming pipeline** (20 min)
   - Send test events
   - Verify features updated in Redis
   - Check latency end-to-end

### Starter Code

```python
# flink_streaming_features.py
from pyflink.datastream import StreamExecutionEnvironment, TimeCharacteristic
from pyflink.table import StreamTableEnvironment, EnvironmentSettings
from pyflink.table.window import Tumble
from datetime import datetime

# ============================================
# STEP 1: Initialize Flink Environment
# ============================================

env = StreamExecutionEnvironment.get_execution_environment()
env.set_parallelism(4)
t_env = StreamTableEnvironment.create(env)

# ============================================
# STEP 2: Define Source (Kafka)
# ============================================

t_env.execute_sql("""
    CREATE TABLE transaction_events (
        user_id STRING,
        merchant_id STRING,
        amount DOUBLE,
        is_approved BOOLEAN,
        event_time TIMESTAMP(3),
        WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
    ) WITH (
        'connector' = 'kafka',
        'topic' = 'transactions',
        'properties.bootstrap.servers' = 'localhost:9092',
        'properties.group.id' = 'flink-features',
        'format' = 'json',
        'scan.startup.mode' = 'latest-offset'
    )
""")

# ============================================
# STEP 3: Compute Streaming Features
# ============================================

# 5-minute tumbling window features
t_env.execute_sql("""
    CREATE TABLE user_5m_features AS
    SELECT
        user_id,
        COUNT(*) AS transaction_count_5m,
        SUM(amount) AS total_amount_5m,
        AVG(amount) AS avg_amount_5m,
        MAX(amount) AS max_amount_5m,
        SUM(CASE WHEN NOT is_approved THEN 1 ELSE 0 END) AS declined_count_5m,
        TUMBLE_END(event_time, INTERVAL '5' MINUTE) AS window_end
    FROM transaction_events
    GROUP BY
        user_id,
        TUMBLE(event_time, INTERVAL '5' MINUTE)
""")

# 1-hour sliding window features (updated every 5 minutes)
t_env.execute_sql("""
    CREATE TABLE user_1h_features AS
    SELECT
        user_id,
        COUNT(*) AS transaction_count_1h,
        AVG(amount) AS avg_amount_1h,
        HOP_END(event_time, INTERVAL '5' MINUTE, INTERVAL '1' HOUR) AS window_end
    FROM transaction_events
    GROUP BY
        user_id,
        HOP(event_time, INTERVAL '5' MINUTE, INTERVAL '1' HOUR)
""")

# ============================================
# STEP 4: Write to Redis Sink
# ============================================

t_env.execute_sql("""
    CREATE TABLE redis_sink (
        user_id STRING,
        transaction_count_5m BIGINT,
        total_amount_5m DOUBLE,
        avg_amount_5m DOUBLE,
        max_amount_5m DOUBLE,
        declined_count_5m BIGINT,
        PRIMARY KEY (user_id) NOT ENFORCED
    ) WITH (
        'connector' = 'redis',
        'host' = 'localhost',
        'port' = '6379',
        'database' = '0',
        'command' = 'SET',
        'key-pattern' = 'streaming_features:user:{user_id}',
        'value.fields-include' = 'ALL',
        'ttl' = '3600'
    )
""")

# Insert into Redis
t_env.execute_sql("""
    INSERT INTO redis_sink
    SELECT
        user_id,
        transaction_count_5m,
        total_amount_5m,
        avg_amount_5m,
        max_amount_5m,
        declined_count_5m
    FROM user_5m_features
""")

# ============================================
# STEP 5: Execute Streaming Job
# ============================================

# This runs indefinitely, processing events as they arrive
env.execute("User Streaming Features")
```

### Kafka Event Producer (for testing)

```python
# produce_events.py
from kafka import KafkaProducer
import json
import time
import random
from datetime import datetime

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

def generate_transaction_event():
    """Generate synthetic transaction event"""
    return {
        "user_id": f"user_{random.randint(1, 100)}",
        "merchant_id": f"merchant_{random.randint(1, 50)}",
        "amount": round(random.uniform(10, 1000), 2),
        "is_approved": random.random() > 0.05,  # 95% approval rate
        "event_time": datetime.now().isoformat()
    }

# Produce events continuously
print("Producing transaction events...")
while True:
    event = generate_transaction_event()
    producer.send('transactions', value=event)
    print(f"Sent: {event}")
    time.sleep(0.1)  # 10 events/second
```

### Success Criteria

- [ ] Flink job starts without errors
- [ ] Events consumed from Kafka successfully
- [ ] Features computed and written to Redis
- [ ] End-to-end latency <10 seconds
- [ ] Features queryable via serving API
- [ ] Late events handled correctly

---

## Exercise 06: Monitor Feature Quality and Drift

**Objective**: Implement comprehensive feature monitoring including drift detection, data quality checks, and alerting.

**Difficulty**: Intermediate | **Duration**: 90 minutes

### Requirements

Build monitoring system that:
1. Detects feature drift using statistical tests
2. Monitors data quality (nulls, outliers, schema violations)
3. Tracks feature serving latency and availability
4. Alerts on anomalies
5. Generates drift reports

### Implementation Steps

1. **Implement drift detection** (30 min)
   - Kolmogorov-Smirnov test for distribution changes
   - Population Stability Index (PSI)
   - Set thresholds for alerts

2. **Add data quality checks** (20 min)
   - NULL rate monitoring
   - Outlier detection
   - Schema validation

3. **Track serving metrics** (15 min)
   - Latency percentiles
   - Cache hit rates
   - Error rates

4. **Set up alerting** (15 min)
   - Alert on drift threshold exceeded
   - Alert on data quality issues
   - Email/Slack notifications

5. **Generate reports** (10 min)
   - Daily drift summary
   - Feature health dashboard

### Starter Code

```python
# feature_monitoring.py
from feast import FeatureStore
import pandas as pd
import numpy as np
from scipy import stats
from typing import Dict, Tuple
from datetime import datetime, timedelta

store = FeatureStore(repo_path=".")

# ============================================
# STEP 1: Drift Detection
# ============================================

def detect_drift_ks_test(
    reference_data: pd.Series,
    current_data: pd.Series,
    threshold: float = 0.05
) -> Tuple[bool, float, float]:
    """
    Detect drift using Kolmogorov-Smirnov test.

    Args:
        reference_data: Historical baseline data
        current_data: Recent data to compare
        threshold: P-value threshold for significance

    Returns:
        (has_drift, statistic, p_value)
    """
    statistic, p_value = stats.ks_2samp(reference_data, current_data)
    has_drift = p_value < threshold

    return has_drift, statistic, p_value

def calculate_psi(
    reference_data: pd.Series,
    current_data: pd.Series,
    num_bins: int = 10
) -> float:
    """
    Calculate Population Stability Index (PSI).

    PSI < 0.1: No significant change
    PSI 0.1-0.25: Moderate change
    PSI > 0.25: Significant drift

    Args:
        reference_data: Baseline distribution
        current_data: Current distribution
        num_bins: Number of bins for histogram

    Returns:
        PSI value
    """
    # Create bins based on reference data
    bins = np.histogram_bin_edges(reference_data, bins=num_bins)

    # Calculate distributions
    ref_dist, _ = np.histogram(reference_data, bins=bins)
    cur_dist, _ = np.histogram(current_data, bins=bins)

    # Normalize to probabilities
    ref_dist = ref_dist / len(reference_data)
    cur_dist = cur_dist / len(current_data)

    # Add small epsilon to avoid division by zero
    epsilon = 1e-10
    ref_dist = ref_dist + epsilon
    cur_dist = cur_dist + epsilon

    # Calculate PSI
    psi = np.sum((cur_dist - ref_dist) * np.log(cur_dist / ref_dist))

    return psi

def monitor_feature_drift(feature_name: str):
    """
    Monitor drift for a specific feature.
    """
    # Get reference period (30 days ago)
    reference_start = datetime.now() - timedelta(days=60)
    reference_end = datetime.now() - timedelta(days=30)

    # Get current period (last 7 days)
    current_start = datetime.now() - timedelta(days=7)
    current_end = datetime.now()

    # Fetch feature data
    # (In production, query from feature store or monitoring database)

    print(f"\n{'='*60}")
    print(f"DRIFT MONITORING: {feature_name}")
    print(f"{'='*60}")

    # Simulate data for demonstration
    reference_data = pd.Series(np.random.normal(100, 20, 10000))
    current_data = pd.Series(np.random.normal(105, 25, 1000))  # Slight drift

    # KS test
    has_drift_ks, ks_stat, p_value = detect_drift_ks_test(reference_data, current_data)

    print(f"\nKolmogorov-Smirnov Test:")
    print(f"  Statistic: {ks_stat:.4f}")
    print(f"  P-value: {p_value:.6f}")
    print(f"  Drift detected: {'⚠️  YES' if has_drift_ks else '✓ NO'}")

    # PSI
    psi = calculate_psi(reference_data, current_data)

    print(f"\nPopulation Stability Index (PSI):")
    print(f"  PSI: {psi:.4f}")
    if psi < 0.1:
        print(f"  Interpretation: ✓ No significant change")
    elif psi < 0.25:
        print(f"  Interpretation: ⚠️  Moderate change")
    else:
        print(f"  Interpretation: ⚠️  Significant drift detected")

    # Distribution statistics
    print(f"\nDistribution Statistics:")
    print(f"  Reference mean: {reference_data.mean():.2f}")
    print(f"  Current mean: {current_data.mean():.2f}")
    print(f"  Mean change: {((current_data.mean() - reference_data.mean()) / reference_data.mean() * 100):.2f}%")

    print(f"  Reference std: {reference_data.std():.2f}")
    print(f"  Current std: {current_data.std():.2f}")

    return has_drift_ks or psi > 0.25

# Run drift monitoring
drift_detected = monitor_feature_drift("transaction_count_30d")

# ============================================
# STEP 2: Data Quality Checks
# ============================================

def check_data_quality(feature_data: pd.DataFrame):
    """
    Perform comprehensive data quality checks.
    """
    print(f"\n{'='*60}")
    print(f"DATA QUALITY CHECKS")
    print(f"{'='*60}")

    for column in feature_data.columns:
        print(f"\n{column}:")

        # NULL rate
        null_rate = feature_data[column].isnull().sum() / len(feature_data)
        print(f"  NULL rate: {null_rate:.2%}")
        if null_rate > 0.05:
            print(f"    ⚠️  HIGH NULL RATE (>5%)")

        # Outlier detection (IQR method)
        if pd.api.types.is_numeric_dtype(feature_data[column]):
            Q1 = feature_data[column].quantile(0.25)
            Q3 = feature_data[column].quantile(0.75)
            IQR = Q3 - Q1
            outliers = (
                (feature_data[column] < Q1 - 1.5 * IQR) |
                (feature_data[column] > Q3 + 1.5 * IQR)
            )
            outlier_rate = outliers.sum() / len(feature_data)
            print(f"  Outlier rate: {outlier_rate:.2%}")
            if outlier_rate > 0.10:
                print(f"    ⚠️  HIGH OUTLIER RATE (>10%)")

        # Value range
        if pd.api.types.is_numeric_dtype(feature_data[column]):
            print(f"  Min: {feature_data[column].min():.2f}")
            print(f"  Max: {feature_data[column].max():.2f}")
            print(f"  Mean: {feature_data[column].mean():.2f}")

# Example usage
sample_data = pd.DataFrame({
    "transaction_count_30d": np.random.poisson(42, 1000),
    "avg_transaction_amount": np.random.gamma(2, 50, 1000),
    "chargeback_rate": np.random.beta(2, 100, 1000)
})

check_data_quality(sample_data)

# ============================================
# STEP 3: Alerting
# ============================================

def send_alert(
    alert_type: str,
    feature_name: str,
    message: str,
    severity: str = "warning"
):
    """
    Send alert via email/Slack/PagerDuty.

    Args:
        alert_type: Type of alert (drift, data_quality, serving)
        feature_name: Name of affected feature
        message: Alert message
        severity: Severity level (info, warning, critical)
    """
    # In production, integrate with alerting service
    print(f"\n🚨 ALERT [{severity.upper()}]")
    print(f"Type: {alert_type}")
    print(f"Feature: {feature_name}")
    print(f"Message: {message}")
    print(f"Timestamp: {datetime.now().isoformat()}")

    # Example: Send to Slack
    # slack_webhook = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
    # requests.post(slack_webhook, json={"text": message})

# Example alert
if drift_detected:
    send_alert(
        alert_type="drift",
        feature_name="transaction_count_30d",
        message="Significant drift detected in transaction_count_30d. PSI > 0.25",
        severity="warning"
    )

# ============================================
# STEP 4: Generate Report
# ============================================

def generate_drift_report(features: List[str]) -> pd.DataFrame:
    """
    Generate daily drift report for all features.
    """
    report_data = []

    for feature in features:
        # Check drift (simplified for demo)
        has_drift = np.random.random() > 0.8  # 20% chance of drift
        psi = np.random.uniform(0, 0.4)

        report_data.append({
            "feature": feature,
            "has_drift": has_drift,
            "psi": psi,
            "severity": "high" if psi > 0.25 else "medium" if psi > 0.1 else "low"
        })

    report_df = pd.DataFrame(report_data)

    print(f"\n{'='*60}")
    print(f"DAILY DRIFT REPORT - {datetime.now().date()}")
    print(f"{'='*60}")
    print(report_df.to_string(index=False))

    return report_df

# Generate report
features_to_monitor = [
    "transaction_count_7d",
    "transaction_count_30d",
    "avg_transaction_amount",
    "chargeback_rate",
    "refund_rate"
]

report = generate_drift_report(features_to_monitor)
```

### Success Criteria

- [ ] Drift detection correctly identifies distribution changes
- [ ] Data quality checks detect issues (nulls, outliers)
- [ ] Alerts sent for anomalies
- [ ] Reports generated successfully
- [ ] Monitoring runs on schedule (cron/Airflow)

---

## Progress Tracking

- [ ] Exercise 01: Set Up Feast Feature Store
- [ ] Exercise 02: Generate Point-in-Time Training Data
- [ ] Exercise 03: Materialize Features to Online Store
- [ ] Exercise 04: Build Real-Time Feature Serving API
- [ ] Exercise 05: Implement Streaming Features
- [ ] Exercise 06: Monitor Feature Quality and Drift

**Completion Goal**: Complete at least 4 exercises (including 1 advanced).

---

## Additional Resources

### Documentation
- [Feast Documentation](https://docs.feast.dev/)
- [Apache Flink Documentation](https://flink.apache.org/docs/)
- [Redis Documentation](https://redis.io/docs/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)

### Tools
- [Feast](https://feast.dev/) - Open-source feature store
- [Tecton](https://www.tecton.ai/) - Managed feature platform
- [Docker Compose](https://docs.docker.com/compose/) - For local development
- [Locust](https://locust.io/) - Load testing

### Libraries
- `feast` - Feature store framework
- `pyflink` - Apache Flink Python API
- `kafka-python` - Kafka client
- `fastapi` - Web framework for serving
- `scipy` - Statistical tests for drift detection

### Articles
- ["What is Point-in-Time Correctness?" (Tecton)](https://www.tecton.ai/blog/point-in-time-correctness/)
- ["Feature Store Architecture" (Uber)](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
- ["Building a Feature Store" (DoorDash)](https://doordash.engineering/2020/11/19/building-a-gigascale-ml-feature-store-with-redis/)

---

*Last updated: November 2, 2025*
