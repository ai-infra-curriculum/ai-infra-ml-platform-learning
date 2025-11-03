# Module 05: Workflow Orchestration

**Duration**: 12 hours
**Level**: Core Concepts
**Prerequisites**: Module 04 completed, Python programming, basic DAG concepts

## Overview

Master workflow orchestration for ML platforms using Apache Airflow, learning to build production-grade pipelines that coordinate data processing, model training, evaluation, and deployment with automatic retries, monitoring, and event-driven triggers.

## Learning Objectives

By the end of this module, you will be able to:

1. ✅ Design DAG-based workflows with proper task atomicity and idempotency
2. ✅ Implement time-based and event-driven scheduling strategies
3. ✅ Use KubernetesExecutor for GPU-based training tasks
4. ✅ Pass data efficiently between tasks using external storage
5. ✅ Build dynamic task mapping for parallel model training
6. ✅ Integrate feature stores with Airflow pipelines
7. ✅ Implement comprehensive monitoring, alerting, and SLA tracking

## Module Structure

### Lecture Notes
- **[01-workflow-orchestration.md](./lecture-notes/01-workflow-orchestration.md)** (6,500+ words)
  - DAG-based workflow design principles
  - Scheduling strategies (cron, event-driven, hybrid)
  - Executor models (Local, Celery, Kubernetes)
  - Data passing best practices
  - Airflow 2.0+ features (TaskFlow API, dynamic mapping, Datasets)
  - Production ML training pipeline implementation
  - Event-driven workflows with Kafka
  - Monitoring and observability patterns
  - Advanced topics (distributed training, CI/CD, cost optimization)
  - Real-world case studies (Airbnb, Netflix, Uber)

### Exercises
- **[6 Hands-On Exercises](./exercises/)** (~9 hours total)
  - Exercise 01: Build Your First ML Training DAG (60 min)
  - Exercise 02: Implement Dynamic Task Mapping (90 min)
  - Exercise 03: Event-Driven Training Pipeline (120 min) - Advanced
  - Exercise 04: Kubernetes Pod Operator for GPU Training (90 min)
  - Exercise 05: Feature Store + Airflow Integration (120 min) - Advanced
  - Exercise 06: Monitoring and Alerting (75 min)

## Key Topics Covered

### 1. DAG Design Principles

**Core Concepts**:
- **Atomicity**: Each task does one thing well
- **Idempotency**: Same input → same output, retry-safe
- **Modularity**: Reusable components across DAGs
- **Separation of Concerns**: Logic in code, orchestration in DAG

**Example DAG Structure**:
```
fetch_data → preprocess → extract_features → train_model → evaluate → deploy
                                                ↓
                                           (parallel)
                                  [model_a, model_b, model_c]
```

**Task Dependencies**:
- Sequential: `task_a >> task_b >> task_c`
- Parallel: `task_a >> [task_b, task_c, task_d] >> task_e`
- Fan-out/fan-in: Common pattern for training multiple models

### 2. Scheduling Strategies

**Time-Based (Cron)**:
```python
schedule_interval='0 2 * * *'  # Daily at 2 AM
schedule_interval='0 * * * *'  # Hourly
schedule_interval='0 1 * * 0'  # Weekly (Sunday)
```

**Event-Driven**:
```python
# File sensor
FileSensor(filepath='/data/incoming/*.csv')

# Kafka sensor
AwaitMessageSensor(topics=['ml.training.requests'])

# Dataset-aware
@dag(schedule=[user_features_dataset])
```

**Hybrid**: Regular schedule + event-based overrides for urgent retraining

### 3. Executor Models

**Comparison**:

| Executor | Best For | Pros | Cons |
|----------|----------|------|------|
| **Local** | Development, small pipelines | Simple setup | Limited parallelism |
| **Celery** | Medium-scale, shared workers | Horizontal scaling | Resource contention |
| **Kubernetes** | ML workloads, GPU tasks | Resource isolation, GPU support | Pod startup overhead |

**Kubernetes Executor** (Recommended for ML):
```python
KubernetesPodOperator(
    task_id='train_model',
    image='ml-training:v1.2.3',
    container_resources=k8s.V1ResourceRequirements(
        limits={'nvidia.com/gpu': '1', 'memory': '32Gi'}
    )
)
```

**Why Kubernetes for ML?**
- Each task gets dedicated resources (CPU, memory, GPU)
- Different dependencies per task (TensorFlow vs PyTorch)
- Dynamic scaling based on workload
- Cost-effective (no idle workers)

### 4. Data Passing Best Practices

**Anti-Pattern: Large Data in XCom**
```python
# ❌ Bad: Pass 10GB DataFrame via XCom
return df  # Will fail or thrash metadata DB
```

**Best Practice: External Storage**
```python
# ✅ Good: Pass path via XCom
output_path = f's3://bucket/data/{date}/train.parquet'
df.to_parquet(output_path)
return output_path  # Only path in XCom (~100 bytes)
```

**Data Size Strategy**:
- < 1KB: XCom (simple values)
- 1KB - 10MB: XCom JSON
- 10MB - 1GB: S3/GCS with path in XCom
- 1GB - 100GB: Parquet in S3
- 100GB+: Data warehouse with table name

### 5. Airflow 2.0+ Features

**TaskFlow API** (Pythonic):
```python
@dag(start_date=datetime(2025, 1, 1), schedule_interval='@daily')
def ml_training_pipeline():

    @task
    def fetch_data():
        return {'path': 's3://bucket/data.parquet', 'rows': 100000}

    @task
    def train(data_info: dict):
        return {'model_id': 'model_v123', 'auc': 0.92}

    data = fetch_data()
    model = train(data)
```

**Dynamic Task Mapping** (Parallel Training):
```python
@task
def get_models():
    return ['xgboost', 'lightgbm', 'catboost']

@task
def train_model(model_name: str):
    return {'model': model_name, 'auc': train(model_name)}

models = get_models()
results = train_model.expand(model_name=models)  # 3 parallel tasks!
```

**Datasets** (Data-Aware Scheduling):
```python
# DAG 1: Produces features
@task(outlets=[user_features_dataset])
def materialize_features():
    pass

# DAG 2: Consumes features (runs when updated)
@dag(schedule=[user_features_dataset])
def train_model():
    pass
```

### 6. Production ML Pipeline Pattern

**Complete Training Pipeline**:
```
1. Validate Data Availability
   ↓
2. Extract Features (from Feature Store with point-in-time correctness)
   ↓
3. Data Quality Checks (Great Expectations)
   ↓
4. Train Model (Kubernetes Pod with GPU)
   ↓
5. Evaluate Model (held-out test set)
   ↓
6. Deployment Gate (branch on performance threshold)
   ↓
7a. Deploy to Staging (if metrics good)
7b. Send Alert (if metrics bad)
   ↓
8. Cleanup (always runs)
```

**Key Patterns**:
- Data validation before expensive operations
- Feature store integration for consistency
- Data quality gates
- GPU-based training in isolated pods
- MLflow for experiment tracking
- Conditional deployment based on metrics
- Proper cleanup regardless of success/failure

### 7. Event-Driven Workflows

**Architecture**: Kafka + Airflow

**Use Cases**:
- Train when new labeled data arrives
- Retrain when model performance degrades (monitoring alert)
- Deploy when model passes evaluation
- Generate features when source data updates

**Implementation**:
```python
# Kafka sensor
wait_for_trigger = AwaitMessageSensor(
    task_id='wait_for_training_trigger',
    topics=['ml.training.requests'],
    kafka_config_id='kafka_default'
)

# REST API trigger
curl -X POST "http://airflow:8080/api/v1/dags/training/dagRuns" \
  -d '{"conf": {"data_path": "s3://bucket/new-data.csv"}}'
```

### 8. Monitoring and Alerting

**Key Metrics**:
- **DAG-level**: Success rate, duration (p50/p95/p99), SLA misses
- **Task-level**: Execution time, retry count, resource usage
- **System-level**: Scheduler lag, queue depth, DB connections

**Prometheus Integration**:
```python
from airflow.stats import Stats

@task
def train_with_metrics():
    Stats.timing('ml.training.duration', duration)
    Stats.gauge('ml.model.parameters', model.num_params)
    Stats.incr('ml.training.success')
```

**SLA Monitoring**:
```python
def sla_miss_callback(dag, task_list, blocking_task_list, slas, blocking_tis):
    send_alert(f"SLA miss: {slas[0].task_id}")

with DAG(
    'training',
    default_args={'sla': timedelta(hours=4)},
    sla_miss_callback=sla_miss_callback
):
    ...
```

**Failure Alerts**:
```python
def failure_callback(context):
    send_alert(
        message=f"Task {context['task_instance'].task_id} failed",
        logs=context['task_instance'].log_url
    )

@task(on_failure_callback=failure_callback)
def critical_task():
    ...
```

## Real-World Applications

### Airbnb ML Workflows

**Scale**: 1,000+ models, 10,000+ DAGs, 200,000+ daily tasks

**Architecture**:
- Airflow on Kubernetes (isolated pods per task)
- Dynamic task generation for new markets
- Zipline feature store integration
- Multi-tenancy (50+ data science teams)

**Results**:
- **5x faster** model iteration (days → hours)
- **80% reduction** in feature engineering duplication
- **99.9% SLA** for production pipelines

### Netflix Orchestration Platform

**Scale**: 2,000+ DAGs, 150,000+ daily tasks, 1 PB+ data processed daily

**Technology**: Conductor (similar to Airflow) + Titus containers

**Key Innovation**: Event-driven retraining
```
User behavior change → Kafka event → Trigger retraining →
Deploy if better → Monitor impact
```

**Results**:
- **40%** of models retrain automatically
- **<4 hours** from drift detection to deployment
- **70%** cluster utilization (vs 35% without orchestration)

### Uber Michelangelo + Airflow

**Integration**: Michelangelo ML platform uses Airflow for orchestration

**Workflows**:
- **Continuous training**: Hourly for real-time models, daily for batch
- **Feature backfill**: Compute historical features when new feature added
- **A/B testing**: Automated deployment and evaluation

**Results**:
- **90% time savings** on feature engineering
- **10M predictions/second** serving capacity
- **<5ms p99** feature retrieval latency

## Code Examples

This module includes production-ready implementations:

- **Complete training DAG**: End-to-end pipeline with validation, feature store, GPU training, MLflow, conditional deployment
- **Dynamic task mapping**: Parallel model training with automatic best model selection
- **Event-driven pipeline**: File sensors and Kafka triggers for real-time training
- **KubernetesPodOperator**: GPU-based training with resource management
- **Feature store integration**: Feast + Airflow for point-in-time correct features
- **Monitoring setup**: Prometheus metrics, SLA tracking, failure callbacks

## Success Criteria

After completing this module, you should be able to:

- [ ] Design DAGs with proper atomicity and idempotency
- [ ] Implement both time-based and event-driven scheduling
- [ ] Use KubernetesExecutor for GPU training tasks
- [ ] Pass data efficiently using external storage (not XCom)
- [ ] Build dynamic task mapping for parallel execution
- [ ] Integrate Feast feature store with Airflow
- [ ] Set up comprehensive monitoring and alerting
- [ ] Deploy production-ready ML training pipelines

## Tools and Technologies

**Core Orchestration**:
- Apache Airflow 2.7+ (TaskFlow API, dynamic mapping, Datasets)
- Airflow Providers (Kubernetes, Kafka, AWS, GCP)

**Execution**:
- KubernetesExecutor (recommended for ML)
- Kubernetes 1.25+
- Docker for task isolation

**Integration**:
- Feast (feature store)
- MLflow (experiment tracking, model registry)
- Great Expectations (data quality)
- Kafka (event streaming)

**Monitoring**:
- Prometheus (metrics)
- Grafana (dashboards)
- StatsD (Airflow metrics export)

**Python Libraries**:
- `apache-airflow` - Core orchestration framework
- `apache-airflow-providers-cncf-kubernetes` - Kubernetes integration
- `apache-airflow-providers-apache-kafka` - Kafka sensors
- `feast` - Feature store client
- `mlflow` - Experiment tracking

## Prerequisites Checklist

Before starting this module, ensure you have:

- [x] Completed Module 04 (Feature Store Architecture)
- [x] Apache Airflow 2.7+ installed
- [x] Python 3.9+
- [x] Docker and Docker Compose
- [x] Understanding of ML training workflows
- [x] Basic knowledge of DAGs and workflow concepts
- [x] (Optional) Kubernetes cluster for Exercise 04
- [x] (Optional) Kafka cluster for Exercise 03

## Estimated Time

| Component | Duration |
|-----------|----------|
| Lecture reading | 1.5-2 hours |
| Exercises (4 minimum) | 6-8 hours |
| Practice and experimentation | 1-2 hours |
| **Total** | **8.5-12 hours** |

**Recommended pace**: Complete over 3-4 days, allowing time to absorb concepts and practice building DAGs.

## Assessment

### Knowledge Check (10 questions)
Located in lecture notes - tests understanding of:
- DAG design principles
- Scheduling strategies
- Executor models
- Data passing best practices
- Dynamic task mapping
- Event-driven workflows
- Monitoring and alerting

### Practical Challenge
Build complete ML training pipeline:
- Multiple dependent tasks (data → features → train → evaluate → deploy)
- Dynamic task mapping for parallel model training
- Conditional deployment based on metrics
- Monitoring with SLA and failure callbacks
- Integration with feature store (Feast)

**Success metrics**:
- DAG executes successfully end-to-end
- Parallel tasks execute correctly
- Conditional logic works (deployment gate)
- Monitoring captures all key events

## What's Next?

**[Module 06: Model Registry & Versioning](../mod-006-model-registry/)**

After mastering workflow orchestration, you'll learn to build model registries for versioning, lineage tracking, and promoting models through deployment stages (staging → production).

---

## Additional Resources

### Documentation
- [Apache Airflow Documentation](https://airflow.apache.org/docs/)
- [Astronomer Airflow Guides](https://www.astronomer.io/docs/learn/)
- [TaskFlow API Tutorial](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html)
- [Dynamic Task Mapping Guide](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html)
- [KubernetesPodOperator Docs](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/operators.html)

### Articles & Case Studies
- [Airbnb: Bighead ML Platform](https://medium.com/airbnb-engineering/bighead-airbnbs-end-to-end-machine-learning-platform-f0b2e2c0caa0)
- [Netflix: Conductor Orchestration](https://netflixtechblog.com/conductor-a-microservices-orchestrator-2e8d4771bf40)
- [Uber: Michelangelo ML Platform](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
- [Astronomer: MLOps with Airflow](https://www.astronomer.io/docs/learn/airflow-mlops)
- [MLOps Community: Workflow Orchestration](https://mlops.community/)

### Videos & Talks
- [Airflow Summit 2025 Talks](https://airflowsummit.org/)
- [Event-Based DAGs with Airflow](https://www.youtube.com/results?search_query=airflow+event+driven)
- [ML Pipelines with Airflow (Astronomer)](https://www.youtube.com/results?search_query=airflow+ml+pipeline)

### Community
- [Apache Airflow Slack](https://apache-airflow-slack.herokuapp.com/)
- [Astronomer Registry](https://registry.astronomer.io/) - Reusable DAGs and operators
- [MLOps Community](https://mlops.community/)

---

## Common Pitfalls

### 1. Large Data in XCom
**Problem**: Passing DataFrames directly between tasks via XCom
**Solution**: Store in S3/GCS, pass only path via XCom

### 2. Non-Idempotent Tasks
**Problem**: Appending to tables on retry causes duplicates
**Solution**: Delete partition before insert, use MERGE/UPSERT

### 3. Tight Task Coupling
**Problem**: Tasks directly calling each other's functions
**Solution**: Use XCom for data passing, maintain task independence

### 4. Missing SLA Monitoring
**Problem**: Long-running tasks go unnoticed
**Solution**: Set SLA on critical tasks, configure callbacks

### 5. No Failure Handling
**Problem**: Silent failures, no alerts
**Solution**: Configure on_failure_callback, email alerts, Slack notifications

### 6. Ignoring Catchup
**Problem**: Accidentally backfilling months of historical runs
**Solution**: Set `catchup=False` unless backfill is needed

### 7. Shared Mutable State
**Problem**: Tasks sharing global variables causing race conditions
**Solution**: Keep tasks stateless, pass data explicitly

---

## Troubleshooting Guide

### Issue: DAG Not Appearing in UI
- Check for Python syntax errors in DAG file
- Verify DAG file is in `dags/` folder
- Check scheduler logs: `airflow scheduler --stderr`
- Validate DAG: `python dags/my_dag.py`

### Issue: Task Stuck in Queued State
- Check executor capacity (pool limits)
- Verify Kubernetes cluster has available resources
- Check scheduler is running: `ps aux | grep scheduler`

### Issue: XCom Size Limit Exceeded
- Reduce data size or use external storage
- Increase `max_allowed_packet` in metadata DB (not recommended)
- Pass file paths instead of data

### Issue: Kubernetes Pod Failed to Schedule
- Check resource requests vs node capacity
- Verify GPU nodes have correct taints/labels
- Check namespace resource quotas
- Review pod events: `kubectl describe pod -n airflow`

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025 | **Version**: 1.0

**Feedback**: Found an issue or have suggestions? Please open an issue in the repository.
