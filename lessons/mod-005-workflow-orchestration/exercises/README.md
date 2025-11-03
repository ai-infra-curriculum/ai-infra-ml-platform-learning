# Module 05 Exercises: Workflow Orchestration

Complete hands-on exercises to master Apache Airflow for ML pipeline orchestration.

**Total Time**: ~9 hours
**Difficulty**: Intermediate to Advanced

---

## Exercise Overview

| Exercise | Title | Duration | Difficulty | Key Skills |
|----------|-------|----------|------------|------------|
| 01 | Build Your First ML Training DAG | 60 min | Basic | DAG design, operators, dependencies |
| 02 | Implement Dynamic Task Mapping | 90 min | Intermediate | TaskFlow API, parallel execution |
| 03 | Event-Driven Training Pipeline | 120 min | Advanced | Sensors, Kafka integration, triggers |
| 04 | Kubernetes Pod Operator for GPU Training | 90 min | Intermediate | K8s executor, resource management |
| 05 | Feature Store + Airflow Integration | 120 min | Advanced | Feast, point-in-time features |
| 06 | Monitoring and Alerting | 75 min | Intermediate | Metrics, SLA, callbacks |

---

## Prerequisites

Before starting these exercises, ensure you have:

- [x] Completed Module 04 (Feature Store Architecture)
- [x] Apache Airflow 2.7+ installed
- [x] Python 3.9+
- [x] Docker and Docker Compose
- [x] Kubernetes cluster access (for Exercise 04)
- [x] Basic understanding of DAGs and workflow concepts

**Installation**:
```bash
# Create virtual environment
python -m venv airflow-env
source airflow-env/bin/activate

# Install Airflow with providers
pip install "apache-airflow==2.7.0" \
    "apache-airflow-providers-cncf-kubernetes==7.4.0" \
    "apache-airflow-providers-apache-kafka==1.1.0"

# Initialize Airflow
export AIRFLOW_HOME=~/airflow
airflow db init
airflow users create \
    --username admin \
    --firstname Admin \
    --lastname User \
    --role Admin \
    --email admin@example.com \
    --password admin
```

---

## Exercise 01: Build Your First ML Training DAG

**Duration**: 60 minutes
**Difficulty**: Basic

### Learning Objectives

- Create a basic ML training DAG with proper structure
- Implement task dependencies
- Use PythonOperator and BranchPythonOperator
- Pass data between tasks using XCom
- Implement retry logic and error handling

### Scenario

You're building a simple sentiment analysis model training pipeline that:
1. Fetches tweet data from CSV
2. Preprocesses text (cleaning, tokenization)
3. Splits data into train/validation sets
4. Trains a logistic regression model
5. Evaluates model performance
6. Conditionally deploys if accuracy > 85%

Complete detailed implementation available in the exercise files.

### Success Criteria

- [ ] DAG appears in Airflow UI without import errors
- [ ] All tasks execute successfully in correct order
- [ ] Data flows correctly through preprocessing → training → evaluation
- [ ] Conditional branching works (deploy vs alert)
- [ ] Task retries work on simulated failures
- [ ] Model achieves >85% accuracy on validation set

---

## Exercise 02: Implement Dynamic Task Mapping

**Duration**: 90 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Use Airflow 2.0+ TaskFlow API
- Implement dynamic task mapping for parallel execution
- Train multiple models simultaneously
- Aggregate results and select best model

### Scenario

You want to try multiple classification algorithms in parallel (Logistic Regression, Random Forest, XGBoost, SVM) and automatically select the best performer.

### Key Implementation Pattern

```python
@task
def get_model_list():
    return ['xgboost', 'lightgbm', 'catboost', 'random_forest']

@task
def train_single_model(model_name: str, data_paths: dict):
    # Train one model
    return {'model': model_name, 'auc': ...}

# Dynamic task mapping
models = get_model_list()
results = train_single_model.expand(model_name=models)  # 4 parallel tasks!
best = select_best_model(results)
```

### Success Criteria

- [ ] DAG dynamically creates 4 training tasks (one per model)
- [ ] All models train in parallel
- [ ] Results correctly aggregated in `select_best_model`
- [ ] Best model selected based on F1 score
- [ ] Comparison table displays all model results

---

## Exercise 03: Event-Driven Training Pipeline

**Duration**: 120 minutes
**Difficulty**: Advanced

### Learning Objectives

- Implement file sensors for event-driven triggering
- Use Kafka sensors for real-time triggers
- Integrate with external monitoring systems
- Implement conditional workflow logic

### Scenario

Your training pipeline should trigger automatically when:
1. New labeled data arrives in S3/local storage
2. Model performance drops below threshold (Kafka alert)
3. Manual trigger via API

### Key Patterns

**File Sensor**:
```python
from airflow.sensors.filesystem import FileSensor

wait_for_new_data = FileSensor(
    task_id='wait_for_new_data',
    filepath='/data/incoming/*.csv',
    poke_interval=60,
    timeout=3600
)
```

**API Trigger**:
```bash
curl -X POST "http://localhost:8080/api/v1/dags/event_driven_training/dagRuns" \
  -H "Content-Type: application/json" \
  -d '{"conf": {"data_path": "s3://bucket/new-data.csv"}}'
```

### Success Criteria

- [ ] DAG triggers when new file detected
- [ ] DAG respects timeout and fails gracefully
- [ ] Multiple trigger sources work independently
- [ ] Triggered run uses correct data path from trigger

---

## Exercise 04: Kubernetes Pod Operator for GPU Training

**Duration**: 90 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Use KubernetesPodOperator for isolated task execution
- Request GPU resources in Kubernetes
- Mount volumes for data access
- Handle pod logs and failures

### Key Implementation

```python
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from kubernetes.client import models as k8s

train_on_gpu = KubernetesPodOperator(
    task_id='train_bert_model',
    name='bert-training-pod',
    namespace='ml-training',
    image='huggingface/transformers-pytorch-gpu:latest',
    cmds=['python', 'train_bert.py'],
    arguments=['--epochs', '10', '--batch-size', '32'],

    # GPU resources
    container_resources=k8s.V1ResourceRequirements(
        requests={'memory': '16Gi', 'cpu': '4'},
        limits={'memory': '32Gi', 'cpu': '8', 'nvidia.com/gpu': '1'}
    ),

    # Retry config
    retries=3,
    retry_delay=timedelta(minutes=10)
)
```

### Success Criteria

- [ ] Pod successfully schedules on GPU node
- [ ] Training completes with GPU utilization
- [ ] Logs visible in Airflow UI
- [ ] Model artifacts saved to persistent volume

---

## Exercise 05: Feature Store + Airflow Integration

**Duration**: 120 minutes
**Difficulty**: Advanced

### Learning Objectives

- Integrate Feast feature store with Airflow
- Implement point-in-time correct feature fetching
- Use Airflow Datasets for data-aware scheduling
- Build feature materialization DAGs

### Key Integration Pattern

```python
from feast import FeatureStore
from airflow.datasets import Dataset

user_features_dataset = Dataset('s3://features/user_features')

@task(outlets=[user_features_dataset])
def materialize_features():
    store = FeatureStore(repo_path='/opt/feast')
    store.materialize_incremental(end_date=datetime.now())

@dag(schedule=[user_features_dataset])  # Trigger when features update
def train_model():
    @task
    def fetch_features():
        store = FeatureStore(repo_path='/opt/feast')
        training_df = store.get_historical_features(...)
        return training_df
```

### Success Criteria

- [ ] Features fetched with point-in-time correctness
- [ ] Integration with materialization DAG
- [ ] Training DAG triggered after feature updates (using Datasets)
- [ ] No data leakage in generated training data

---

## Exercise 06: Monitoring and Alerting

**Duration**: 75 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Implement SLA monitoring for tasks
- Export custom metrics to Prometheus
- Configure failure callbacks
- Create Grafana dashboards

### Key Patterns

**SLA Monitoring**:
```python
def sla_miss_callback(dag, task_list, blocking_task_list, slas, blocking_tis):
    send_slack_alert(f"SLA miss: {slas[0].task_id}")

with DAG(
    'monitored_training',
    default_args={'sla': timedelta(hours=2)},
    sla_miss_callback=sla_miss_callback
) as dag:
    ...
```

**Custom Metrics**:
```python
from airflow.stats import Stats

@task
def train_with_metrics():
    start = time.time()
    model = train_model()
    duration = time.time() - start

    Stats.timing('ml.training.duration', duration)
    Stats.gauge('ml.model.params', model.num_params)
    Stats.incr('ml.training.success')
```

**Failure Callbacks**:
```python
def failure_callback(context):
    task_instance = context['task_instance']
    send_alert(
        message=f"Task {task_instance.task_id} failed",
        logs=task_instance.log_url
    )

@task(on_failure_callback=failure_callback)
def critical_task():
    ...
```

### Success Criteria

- [ ] SLA alerts triggered on timeout
- [ ] Custom metrics exported to Prometheus
- [ ] Failure callbacks send notifications
- [ ] Grafana dashboard displays training metrics

---

## Additional Resources

- [Airflow Documentation](https://airflow.apache.org/docs/)
- [TaskFlow API Tutorial](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html)
- [Kubernetes Operator Guide](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/operators.html)
- [Dynamic Task Mapping](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html)

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025
