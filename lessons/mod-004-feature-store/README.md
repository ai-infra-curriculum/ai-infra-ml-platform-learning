# Module 04: Feature Store Architecture

**Duration**: 12 hours
**Level**: Core Concepts
**Prerequisites**: Module 03 completed, SQL knowledge, feature engineering basics

## Overview

Master feature store architecture and implementation using Feast, learning to build systems that serve features consistently to both training and inference with point-in-time correctness.

## Learning Objectives

By the end of this module, you will be able to:

1. ✅ Understand online vs offline stores and their use cases
2. ✅ Implement point-in-time correct feature joins to prevent data leakage
3. ✅ Design feature transformation pipelines with Feast
4. ✅ Materialize features from offline to online stores
5. ✅ Serve features with <10ms latency for real-time inference
6. ✅ Implement streaming feature computation with Flink
7. ✅ Monitor feature quality and detect drift

## Module Structure

### Lecture Notes
- **[01-feature-store-architecture.md](./lecture-notes/01-feature-store-architecture.md)** (5,000+ words)
  - Feature store fundamentals
  - Online vs offline stores architecture
  - Point-in-time correctness and data leakage prevention
  - Feast implementation and configuration
  - Feature materialization strategies
  - Streaming features with Apache Flink
  - Feature monitoring and drift detection
  - Production case studies (Uber, DoorDash, Netflix)

### Exercises
- **[6 Hands-On Exercises](./exercises/)** (~9.75 hours total)
  - Exercise 01: Set Up Feast Feature Store (60 min)
  - Exercise 02: Generate Point-in-Time Training Data (90 min)
  - Exercise 03: Materialize Features to Online Store (75 min)
  - Exercise 04: Build Real-Time Feature Serving API (120 min) - Advanced
  - Exercise 05: Implement Streaming Features (150 min) - Advanced
  - Exercise 06: Monitor Feature Quality and Drift (90 min)

## Key Topics Covered

### 1. Feature Store Fundamentals

**The Problem**:
- Training-serving skew (70% of production ML failures)
- Feature duplication across teams
- Data leakage from improper temporal joins
- Operational complexity at scale

**The Solution**:
- Centralized feature management
- Consistent definitions for training and serving
- Point-in-time correct historical data
- Low-latency online serving

### 2. Online vs Offline Stores

**Offline Store** (Historical features for training):
- Storage: Data warehouse (BigQuery, Snowflake, S3)
- Latency: Seconds to minutes
- Purpose: Training data generation, batch scoring
- Query: Point-in-time joins over terabytes

**Online Store** (Real-time features for inference):
- Storage: Key-value store (Redis, DynamoDB, Cassandra)
- Latency: <10ms
- Purpose: Real-time predictions
- Query: Point lookups by entity ID

**Architecture**:
```
Offline Store (Snowflake)     Online Store (Redis)
      ↓ (Batch ETL)                 ↓ (Materialize)
  Training Data              Real-Time Serving
```

### 3. Point-in-Time Correctness

**Critical Concept**: For each training example at time T, use only feature values available at or before time T.

**Without Point-in-Time**:
```sql
-- ❌ Data leakage: uses future information
SELECT t.*, f.*
FROM transactions t
JOIN features f ON t.user_id = f.user_id
WHERE t.timestamp < '2024-10-15'
```

**With Point-in-Time**:
```sql
-- ✅ Correct: features as of transaction time
SELECT t.*, f.*
FROM transactions t
JOIN LATERAL (
  SELECT * FROM features f
  WHERE f.user_id = t.user_id
    AND f.event_timestamp <= t.timestamp
  ORDER BY f.event_timestamp DESC LIMIT 1
) f
WHERE t.timestamp < '2024-10-15'
```

**Impact**:
- Without: AUC 0.95 training → 0.72 production (23% drop)
- With: AUC 0.88 training → 0.86 production (2% drop)

### 4. Feast Implementation

**Core Components**:
1. **Entities**: Primary keys (user_id, merchant_id)
2. **Feature Views**: Groups of related features
3. **Feature Services**: Versioned feature sets for models
4. **Data Sources**: Batch files, warehouses, streams
5. **Registry**: Metadata store for all definitions

**Example Feature View**:
```python
user_features = FeatureView(
    name="user_statistics",
    entities=[user],
    ttl=timedelta(days=90),
    schema=[
        Field(name="transaction_count_30d", dtype=Int64),
        Field(name="avg_transaction_amount", dtype=Float32),
    ],
    source=user_stats_source,
    online=True  # Enable online serving
)
```

### 5. Feature Materialization

**Process**: Copy computed features from offline to online store for low-latency serving.

**Materialization Types**:
- **Initial**: Backfill historical data
- **Incremental**: Update with new data (hourly/daily)
- **Streaming**: Real-time updates from event streams

**Implementation**:
```python
# Materialize last 7 days
store.materialize(
    start_date=datetime.now() - timedelta(days=7),
    end_date=datetime.now()
)

# Incremental (in scheduled job)
store.materialize_incremental(end_date=datetime.now())
```

### 6. Real-Time Feature Serving

**Architecture**:
```
Client → FastAPI → Feature Store (Redis) → Model → Prediction
          ↓ (cache)    ↓ (<10ms)
       Cache Layer   On-Demand Features
```

**Performance Targets**:
- Feature retrieval: <10ms (p99)
- Cache hit rate: >70%
- Total latency: <15ms (p95)

### 7. Streaming Features

**Use Case**: Real-time aggregations (last 5 minutes, last hour)

**Stack**:
- Kafka: Event ingestion
- Flink: Stream processing
- Redis: Feature storage

**Example**:
```python
# 5-minute tumbling window
t_env.execute_sql("""
    SELECT
        user_id,
        COUNT(*) AS transaction_count_5m,
        AVG(amount) AS avg_amount_5m,
        TUMBLE_END(event_time, INTERVAL '5' MINUTE) AS window_end
    FROM transaction_events
    GROUP BY user_id, TUMBLE(event_time, INTERVAL '5' MINUTE)
""")
```

### 8. Feature Monitoring

**Monitoring Dimensions**:
1. **Drift Detection**: Distribution changes over time
   - Kolmogorov-Smirnov test
   - Population Stability Index (PSI)

2. **Data Quality**: Null rates, outliers, schema violations

3. **Serving Metrics**: Latency, cache hit rates, availability

**Alert Thresholds**:
- PSI > 0.25: Significant drift
- Null rate > 5%: Data quality issue
- p95 latency > 15ms: Performance degradation

## Real-World Applications

### Uber Michelangelo Feature Store
**Scale**: 10,000+ features, 1,000+ models, 10M predictions/second

**Architecture**:
- Offline: Hive (batch features)
- Online: Cassandra (<5ms serving)
- Streaming: Flink for real-time features

**Results**:
- 90% reduction in feature engineering time
- 70% feature reuse rate

### DoorDash Feature Store
**Use Case**: Delivery time prediction with hybrid features

**Features**:
- Real-time: Dasher location, restaurant queue size
- Batch: Historical prep time, traffic patterns

**Results**:
- Prediction accuracy: 85% → 92%
- Serving latency: <5ms (p99)

### Netflix Feature Store
**Scale**: 50+ TB feature data, 200M+ users, 1M requests/second

**Architecture**:
- Offline: S3 + Spark
- Online: EVCache (distributed memcached)
- Streaming: Flink for user activity

**Results**:
- <10ms serving latency
- Consistent features across 50+ models

## Code Examples

This module includes production-ready implementations:

- **Feast setup**: Complete feature store configuration
- **Point-in-time joins**: Training data generation without leakage
- **Materialization pipeline**: Airflow DAG for scheduled updates
- **Serving API**: FastAPI with <15ms latency, caching, monitoring
- **Streaming features**: Flink job for real-time aggregations
- **Drift detection**: KS test + PSI with alerting

## Success Criteria

After completing this module, you should be able to:

- [ ] Set up Feast with online and offline stores
- [ ] Generate training data with zero data leakage
- [ ] Materialize features to Redis in <5 minutes
- [ ] Serve features with <10ms latency (p99)
- [ ] Build streaming features with Flink
- [ ] Detect feature drift automatically

## Tools and Technologies

**Feature Store**:
- Feast (open-source)
- Tecton (managed platform)

**Storage**:
- Offline: BigQuery, Snowflake, S3 + Athena
- Online: Redis, DynamoDB, Cassandra

**Streaming**:
- Apache Flink
- Apache Kafka
- Spark Streaming

**Python Libraries**:
- `feast` - Feature store framework
- `redis` - Online store client
- `pyflink` - Stream processing
- `scipy` - Statistical tests for monitoring

## Prerequisites Checklist

Before starting this module, ensure you have:

- [x] Completed Module 03 (Multi-Tenancy)
- [x] Python 3.9+ installed
- [x] Docker for running Redis
- [x] Understanding of SQL and data warehousing
- [x] Basic knowledge of feature engineering
- [x] (Optional) Access to BigQuery/Snowflake for production offline store
- [x] (Optional) Kafka + Flink for streaming exercises

## Estimated Time

| Component | Duration |
|-----------|----------|
| Lecture reading | 1.5 hours |
| Exercises (4 minimum) | 6-10 hours |
| Practice and experimentation | 1-2 hours |
| **Total** | **8.5-13.5 hours** |

**Recommended pace**: Complete over 3-4 days, allowing time to absorb concepts and practice with real data.

## Assessment

### Knowledge Check (10 questions)
Located in lecture notes - tests understanding of:
- Online vs offline stores
- Point-in-time correctness
- Feature materialization
- Streaming features
- Drift detection

### Practical Challenge
Build complete feature store pipeline:
- Multiple entities and feature views
- Point-in-time training data generation
- Materialization to Redis
- FastAPI serving endpoint
- Feature drift monitoring

**Success metrics**:
- Training data has zero data leakage
- Online serving <10ms latency
- Drift detection catches distribution changes

## What's Next?

**[Module 05: Workflow Orchestration](../mod-005-workflow-orchestration/)**

After mastering feature stores, you'll learn to orchestrate complex ML workflows including training, deployment, and monitoring pipelines.

---

## Additional Resources

### Documentation
- [Feast Documentation](https://docs.feast.dev/)
- [Feast GitHub](https://github.com/feast-dev/feast)
- [Tecton Documentation](https://docs.tecton.ai/)
- [Apache Flink Documentation](https://flink.apache.org/docs/)

### Articles & Papers
- ["What is a Feature Store?" (Tecton)](https://www.tecton.ai/blog/what-is-a-feature-store/)
- ["Point-in-Time Correctness" (Tecton)](https://www.tecton.ai/blog/point-in-time-correctness/)
- ["Uber Michelangelo" (Uber Engineering)](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
- ["DoorDash Feature Store" (DoorDash Engineering)](https://doordash.engineering/2020/11/19/building-a-gigascale-ml-feature-store-with-redis/)
- ["Feature Store for ML" (Google Cloud)](https://cloud.google.com/architecture/architecture-for-mlops-using-tfx-kubeflow-pipelines-and-cloud-build)

### Tools
- [Feast](https://feast.dev/) - Open-source feature store
- [Tecton](https://www.tecton.ai/) - Managed feature platform
- [Hopsworks Feature Store](https://www.hopsworks.ai/feature-store)
- [AWS SageMaker Feature Store](https://aws.amazon.com/sagemaker/feature-store/)
- [Databricks Feature Store](https://docs.databricks.com/machine-learning/feature-store/index.html)

### Videos & Tutorials
- [Feast Quickstart Tutorial](https://docs.feast.dev/getting-started/quickstart)
- [Feature Store Summit Talks](https://www.featurestoresummit.com/)
- [Tecton Feature Store Demo](https://www.youtube.com/results?search_query=tecton+feature+store)

### Community
- [Feast Slack Community](https://slack.feast.dev/)
- [MLOps Community - Feature Stores](https://mlops.community/)

---

## Common Pitfalls

### 1. Not Using Point-in-Time Joins
**Problem**: Naive joins cause data leakage
**Solution**: Always use Feast's `get_historical_features()` for training data

### 2. Stale Online Features
**Problem**: Online store not updated regularly
**Solution**: Schedule incremental materialization hourly/daily

### 3. High Serving Latency
**Problem**: Direct warehouse queries at inference time
**Solution**: Materialize to Redis, add caching layer

### 4. Feature Drift Undetected
**Problem**: Model degrades over time silently
**Solution**: Implement automated drift monitoring

### 5. Training-Serving Skew
**Problem**: Different feature code for training vs serving
**Solution**: Use feature store for BOTH training and serving

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025 | **Version**: 1.0

**Feedback**: Found an issue or have suggestions? Please open an issue in the repository.
