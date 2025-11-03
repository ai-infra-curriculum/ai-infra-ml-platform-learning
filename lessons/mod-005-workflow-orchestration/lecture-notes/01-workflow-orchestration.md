# Lecture 01: Workflow Orchestration for ML Platforms

**Duration**: 90 minutes
**Level**: Core Concepts
**Prerequisites**: Module 04 completed, Python programming, basic DAG concepts

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

### The Challenge of ML Workflows

Modern machine learning systems require orchestrating complex, multi-step workflows that span data ingestion, feature engineering, model training, evaluation, deployment, and monitoring. These workflows must be:

- **Reproducible**: Same inputs → same outputs
- **Scalable**: Handle growing data and computation
- **Reliable**: Automatic retries and failure handling
- **Observable**: Track execution and debug failures
- **Maintainable**: Easy to understand and modify

**The Problem Without Orchestration**:
```python
# ❌ Manual workflow execution - error-prone
python fetch_data.py
python preprocess.py
python extract_features.py
python train_model.py
python evaluate.py
python deploy.py  # What if this fails?
```

**Issues**:
- No dependency tracking (what if preprocessing fails?)
- No automatic retries (need manual intervention)
- No parallel execution (slow sequential processing)
- No visibility (where did it fail?)
- No reproducibility (hard to replay failed steps)

**The Solution: Workflow Orchestration**:
```python
# ✅ DAG-based orchestration with Airflow
with DAG('ml_training_pipeline') as dag:
    fetch = PythonOperator(task_id='fetch_data', ...)
    preprocess = PythonOperator(task_id='preprocess', ...)
    features = PythonOperator(task_id='extract_features', ...)
    train = KubernetesPodOperator(task_id='train_model', ...)
    evaluate = PythonOperator(task_id='evaluate', ...)
    deploy = BranchPythonOperator(task_id='deploy', ...)

    fetch >> preprocess >> features >> train >> evaluate >> deploy
```

**Benefits**:
- Dependency management: Tasks execute in correct order
- Automatic retries: Transient failures handled automatically
- Parallel execution: Independent tasks run concurrently
- Full observability: Web UI shows execution state
- Reproducibility: Replay any historical run

### What is Workflow Orchestration?

**Definition**: Workflow orchestration is the automated coordination of multiple tasks in a defined sequence, managing dependencies, execution, monitoring, and failure handling.

**Key Components**:
1. **DAG (Directed Acyclic Graph)**: Workflow structure
2. **Tasks**: Individual units of work
3. **Operators**: Task implementations (Python, Kubernetes, Spark, etc.)
4. **Scheduler**: Triggers workflows based on time or events
5. **Executor**: Runs tasks (local, distributed, Kubernetes)
6. **Metadata Store**: Tracks execution history and state

**Workflow Orchestration vs Task Queuing**:

| Aspect | Orchestration (Airflow) | Task Queue (Celery) |
|--------|------------------------|---------------------|
| **Use Case** | Complex workflows with dependencies | Independent async tasks |
| **Scheduling** | Cron-based, event-driven | On-demand, immediate |
| **Dependencies** | Explicit DAG structure | No built-in dependencies |
| **Retry Logic** | Per-task retry policies | Manual implementation |
| **Observability** | Rich UI, DAG visualization | Basic monitoring |
| **Best For** | Batch ML pipelines | Real-time API backends |

### Why Workflow Orchestration Matters for ML

**Production ML Complexity**:
- **10-15 steps** in typical training pipeline
- **50+ tasks** in end-to-end MLOps workflow
- **Multiple systems**: Spark, Kubernetes, databases, APIs
- **Different schedules**: Hourly features, daily training, weekly retraining
- **Resource constraints**: GPU availability, memory limits

**Industry Statistics**:
- **87%** of ML projects fail to reach production (VentureBeat)
- **70%** of production ML issues stem from operational problems, not models
- **60%** reduction in time-to-production with proper orchestration (Netflix)
- **5x improvement** in data scientist productivity (Airbnb)

---

## Theoretical Foundations

### 1. DAG-Based Workflow Design

#### What is a DAG?

A **Directed Acyclic Graph** is a graph with:
- **Directed edges**: Tasks flow in one direction
- **No cycles**: Cannot loop back to previous tasks
- **Nodes**: Individual tasks
- **Edges**: Dependencies between tasks

**Valid DAG**:
```
fetch_data → preprocess → [feature_eng_A, feature_eng_B] → join_features → train → evaluate → deploy
```

**Invalid DAG (has cycle)**:
```
train → evaluate → tune_hyperparams → train  # ❌ Cycle!
```

#### DAG Design Principles

**1. Atomicity**: Each task should do one thing well
```python
# ❌ Bad: Monolithic task
def process_and_train():
    data = fetch_data()
    cleaned = preprocess(data)
    features = extract_features(cleaned)
    model = train_model(features)
    return model

# ✅ Good: Atomic tasks
fetch_data_task = PythonOperator(task_id='fetch', python_callable=fetch_data)
preprocess_task = PythonOperator(task_id='preprocess', python_callable=preprocess)
extract_features_task = PythonOperator(task_id='features', python_callable=extract_features)
train_task = PythonOperator(task_id='train', python_callable=train_model)
```

**Why?**
- Better failure isolation
- Easier to retry individual steps
- Parallel execution of independent tasks
- Clearer observability

**2. Idempotency**: Same input → same output, no side effects on retries
```python
# ❌ Not idempotent: Appends to table
def save_predictions(predictions):
    db.execute(f"INSERT INTO predictions VALUES {predictions}")

# ✅ Idempotent: Replaces partition
def save_predictions_idempotent(predictions, date):
    db.execute(f"DELETE FROM predictions WHERE date = '{date}'")
    db.execute(f"INSERT INTO predictions VALUES {predictions}")
```

**3. Modularity**: Reusable components across DAGs
```python
# Reusable task factory
def create_training_task(model_name: str, config: dict):
    return KubernetesPodOperator(
        task_id=f'train_{model_name}',
        image='ml-training:latest',
        env_vars={'MODEL': model_name, 'CONFIG': json.dumps(config)},
        ...
    )

# Use in multiple DAGs
xgboost_task = create_training_task('xgboost', xgboost_config)
lightgbm_task = create_training_task('lightgbm', lgb_config)
```

**4. Separation of Concerns**: Logic in code, not in orchestrator
```python
# ❌ Bad: Business logic in DAG
with DAG('training') as dag:
    @task
    def train():
        data = pd.read_csv('/data/train.csv')
        X = data.drop('target', axis=1)
        y = data['target']
        model = XGBClassifier()
        model.fit(X, y)
        joblib.dump(model, '/models/model.pkl')

# ✅ Good: DAG orchestrates, logic in modules
from ml_pipeline.training import train_model

with DAG('training') as dag:
    train_task = PythonOperator(
        task_id='train',
        python_callable=train_model,
        op_kwargs={'data_path': '/data/train.csv', 'model_path': '/models/model.pkl'}
    )
```

#### Task Dependencies and Parallelism

**Sequential Dependencies**:
```python
task_a >> task_b >> task_c  # A → B → C
```

**Parallel Execution**:
```python
task_a >> [task_b, task_c, task_d] >> task_e
# task_b, task_c, task_d run in parallel after task_a
```

**Fan-out/Fan-in Pattern** (common in ML):
```python
fetch_data >> [
    train_model_a,
    train_model_b,
    train_model_c
] >> evaluate_all_models >> select_best >> deploy
```

**Dynamic Task Generation** (Airflow 2.0+):
```python
from airflow.decorators import task

@task
def get_model_configs():
    return ['xgboost', 'lightgbm', 'catboost']

@task
def train_model(model_name: str):
    # Training logic
    pass

with DAG('dynamic_training') as dag:
    models = get_model_configs()
    train_model.expand(model_name=models)  # Creates 3 tasks dynamically
```

### 2. Scheduling Strategies

#### Time-Based Scheduling (Cron)

**Common ML Schedules**:
```python
# Daily training at 2 AM
schedule_interval='0 2 * * *'

# Hourly feature materialization
schedule_interval='0 * * * *'

# Weekly model retraining (Sunday 1 AM)
schedule_interval='0 1 * * 0'

# Every 15 minutes (real-time features)
schedule_interval='*/15 * * * *'
```

**Scheduling Considerations**:
- **Data availability**: Schedule after upstream data arrives
- **Resource contention**: Avoid peak usage times
- **SLA windows**: Complete before downstream consumers need data
- **Catchup behavior**: Backfill historical runs or skip?

**Example**:
```python
with DAG(
    'daily_training',
    start_date=datetime(2025, 1, 1),
    schedule_interval='0 2 * * *',  # 2 AM daily
    catchup=False,  # Don't backfill historical runs
    max_active_runs=1,  # Only one run at a time
    sla=timedelta(hours=4),  # Must complete within 4 hours
) as dag:
    ...
```

#### Event-Driven Scheduling

**Trigger on External Events** (Airflow 2.2+):
```python
from airflow.sensors.filesystem import FileSensor
from airflow.datasets import Dataset

# Trigger when new data file arrives
wait_for_data = FileSensor(
    task_id='wait_for_data',
    filepath='/data/incoming/*.csv',
    poke_interval=60,  # Check every 60 seconds
    timeout=3600  # Timeout after 1 hour
)

# Trigger when upstream DAG produces dataset
dataset = Dataset('s3://bucket/features/user_features')

with DAG('training', schedule=[dataset]) as dag:
    # Runs when 'user_features' dataset is updated
    train = PythonOperator(...)
```

**Kafka-Based Event Triggering**:
```python
from airflow.providers.apache.kafka.sensors.kafka import AwaitMessageSensor

# Trigger when Kafka message arrives
kafka_sensor = AwaitMessageSensor(
    task_id='wait_for_model_request',
    topics=['ml.training.requests'],
    kafka_config_id='kafka_default',
    apply_function='process_message'
)
```

**Use Cases**:
- Train model when new labeled data arrives
- Retrain when model performance degrades (from monitoring system)
- Deploy when model passes evaluation checks
- Generate features when source data updates

#### Hybrid Scheduling: Time + Event

**Pattern**: Regular schedule with event-based overrides
```python
# Base schedule: Daily at 2 AM
schedule_interval='0 2 * * *'

# But also trigger immediately if:
# - Critical data quality issue detected
# - Model performance drops below threshold
# - New labeled data batch available

with DAG('adaptive_training', schedule_interval='0 2 * * *') as dag:
    # Regular daily training
    check_trigger_conditions = BranchPythonOperator(
        task_id='check_triggers',
        python_callable=lambda: 'urgent_train' if check_urgent() else 'normal_train'
    )

    normal_train = KubernetesPodOperator(...)
    urgent_train = KubernetesPodOperator(resources={'nvidia.com/gpu': 4}, ...)  # More GPUs
```

### 3. Executor Models

#### Local Executor

**How it Works**: Tasks run as subprocesses on the same machine as the scheduler.

**Pros**:
- Simple setup for development
- No additional infrastructure
- Easy debugging

**Cons**:
- Limited parallelism (single machine)
- Not scalable
- No resource isolation

**Use Cases**:
- Development and testing
- Small pipelines (<10 tasks)

#### Celery Executor

**How it Works**: Distributed task execution using Celery + Redis/RabbitMQ.

**Architecture**:
```
Scheduler → Message Broker (Redis) → [Worker 1, Worker 2, ..., Worker N]
```

**Pros**:
- Horizontal scaling (add more workers)
- Better resource isolation
- Queue management

**Cons**:
- Complex setup (broker, workers)
- Resource contention on shared workers
- Harder to manage dependencies

**Configuration**:
```python
# airflow.cfg
[core]
executor = CeleryExecutor

[celery]
broker_url = redis://localhost:6379/0
result_backend = db+postgresql://airflow:airflow@localhost/airflow
worker_concurrency = 16
```

#### Kubernetes Executor (Recommended for ML)

**How it Works**: Each task runs in its own Kubernetes Pod.

**Architecture**:
```
Scheduler → Kubernetes API → [Pod 1, Pod 2, ..., Pod N]
                                (each task gets own pod)
```

**Pros**:
- **Resource isolation**: Each task gets dedicated resources
- **Dynamic scaling**: Pods created/destroyed on demand
- **Custom resources**: Specify CPU, memory, GPU per task
- **Docker images**: Different dependencies per task
- **Cost-effective**: No idle workers

**Cons**:
- Kubernetes cluster required
- Pod startup overhead (~10-30s)

**Example Task**:
```python
train_model = KubernetesPodOperator(
    task_id='train_xgboost',
    name='train-xgboost-pod',
    namespace='ml-training',
    image='ml-training:v1.2.3',
    cmds=['python', 'train.py'],
    arguments=['--model=xgboost', '--data=/data/train.parquet'],

    # Resource requirements
    container_resources=k8s.V1ResourceRequirements(
        requests={'memory': '16Gi', 'cpu': '8'},
        limits={'memory': '32Gi', 'cpu': '16', 'nvidia.com/gpu': '1'}
    ),

    # Environment variables
    env_vars={'MLFLOW_TRACKING_URI': 'http://mlflow:5000'},

    # Volumes
    volumes=[...],
    volume_mounts=[...],

    # Retry policy
    retries=3,
    retry_delay=timedelta(minutes=5)
)
```

**Why Kubernetes Executor for ML?**
- Train multiple models in parallel with different resource needs
- Use GPU nodes only for training tasks
- Isolate dependencies (TensorFlow vs PyTorch)
- Auto-scaling based on workload

### 4. Data Passing Between Tasks

#### Anti-Pattern: XCom for Large Data

**XCom (Cross-Communication)** is Airflow's mechanism for passing small data between tasks.

**XCom Limitations**:
- Stored in metadata database (PostgreSQL/MySQL)
- Max size: ~1MB (depends on DB config)
- Serialization overhead

**❌ Bad**:
```python
@task
def fetch_data():
    df = pd.read_csv('data.csv')  # 10GB dataframe
    return df  # ❌ XCom will fail or thrash DB

@task
def preprocess(df):
    return df.dropna()
```

#### Best Practice: External Storage

**✅ Good**:
```python
@task
def fetch_data(execution_date: str):
    df = pd.read_csv('data.csv')
    output_path = f's3://bucket/data/{execution_date}/raw.parquet'
    df.to_parquet(output_path)
    return output_path  # Only pass path via XCom

@task
def preprocess(data_path: str, execution_date: str):
    df = pd.read_parquet(data_path)
    cleaned = df.dropna()
    output_path = f's3://bucket/data/{execution_date}/cleaned.parquet'
    cleaned.to_parquet(output_path)
    return output_path
```

**Data Passing Strategies**:

| Data Size | Strategy | Storage |
|-----------|----------|---------|
| < 1KB | XCom | Metadata DB |
| 1KB - 10MB | XCom (JSON) | Metadata DB |
| 10MB - 1GB | S3/GCS with path in XCom | Object storage |
| 1GB - 100GB | Parquet in S3 + path in XCom | Object storage |
| 100GB+ | Data warehouse with table name | BigQuery/Snowflake |

**Feature Store Integration**:
```python
@task
def compute_features(execution_date: str):
    # Compute features
    features_df = ...

    # Write to offline store
    store.write_to_offline_store(
        feature_view='user_features',
        df=features_df
    )

    # Return only metadata
    return {
        'feature_view': 'user_features',
        'timestamp': execution_date,
        'row_count': len(features_df)
    }

@task
def train_model(feature_metadata: dict):
    # Read from feature store
    training_df = store.get_historical_features(
        entity_df=entities,
        features=['user_features:*']
    )
    # Train model...
```

---

## Practical Implementation

### 1. Apache Airflow Architecture

#### Core Components

**1. Web Server**: UI for monitoring and management
```bash
# Start web server
airflow webserver --port 8080
```

**2. Scheduler**: Triggers DAGs based on schedules/events
```bash
# Start scheduler
airflow scheduler
```

**3. Metadata Database**: Stores DAG definitions, task state, history
```python
# airflow.cfg
[database]
sql_alchemy_conn = postgresql+psycopg2://airflow:password@localhost/airflow
```

**4. Executor**: Runs tasks (LocalExecutor, CeleryExecutor, KubernetesExecutor)

**5. Workers** (if using CeleryExecutor):
```bash
# Start Celery worker
airflow celery worker
```

#### Airflow 2.0+ Features for ML

**1. TaskFlow API** (Pythonic task definition):
```python
from airflow.decorators import dag, task
from datetime import datetime

@dag(start_date=datetime(2025, 1, 1), schedule_interval='@daily')
def ml_training_pipeline():

    @task
    def fetch_data():
        # Fetch logic
        return {'path': 's3://bucket/data.parquet', 'rows': 100000}

    @task
    def train(data_info: dict):
        # Training logic
        return {'model_id': 'model_v123', 'auc': 0.92}

    @task
    def deploy(model_info: dict):
        if model_info['auc'] > 0.90:
            # Deploy logic
            return 'deployed'
        return 'skipped'

    data = fetch_data()
    model = train(data)
    deploy(model)

ml_training_pipeline()
```

**2. Dynamic Task Mapping** (parallel model training):
```python
@task
def get_models():
    return ['xgboost', 'lightgbm', 'catboost', 'random_forest']

@task
def train_model(model_name: str):
    # Training logic for each model
    return {'model': model_name, 'auc': train(model_name)}

@task
def select_best(results: list):
    best = max(results, key=lambda x: x['auc'])
    return best

@dag(...)
def ensemble_training():
    models = get_models()
    results = train_model.expand(model_name=models)  # 4 parallel tasks
    best = select_best(results)

ensemble_training()
```

**3. Datasets** (data-aware scheduling):
```python
from airflow.datasets import Dataset

user_features_dataset = Dataset('s3://features/user_features')

# DAG 1: Produces dataset
@dag(schedule_interval='@hourly')
def compute_features():
    @task(outlets=[user_features_dataset])
    def materialize_features():
        # Compute and save features
        pass

    materialize_features()

# DAG 2: Consumes dataset (runs when features update)
@dag(schedule=[user_features_dataset])
def train_model():
    @task
    def train():
        # Use features for training
        pass

    train()
```

### 2. Building an ML Training Pipeline

#### Complete Example: End-to-End Training DAG

This is a production-ready example showing a complete fraud detection training pipeline with data validation, feature store integration, GPU training, MLflow tracking, and conditional deployment.

```python
from airflow import DAG
from airflow.decorators import task
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from airflow.operators.python import BranchPythonOperator
from airflow.utils.trigger_rule import TriggerRule
from datetime import datetime, timedelta
from kubernetes.client import models as k8s
import mlflow

default_args = {
    'owner': 'ml-team',
    'depends_on_past': False,
    'email': ['ml-alerts@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'execution_timeout': timedelta(hours=6)
}

with DAG(
    'fraud_detection_training',
    default_args=default_args,
    description='Train fraud detection model with feature store',
    schedule_interval='0 2 * * *',  # 2 AM daily
    start_date=datetime(2025, 1, 1),
    catchup=False,
    max_active_runs=1,
    tags=['ml', 'training', 'fraud-detection']
) as dag:

    # Task 1: Validate data availability
    @task
    def validate_data_availability(ds: str):
        """Check if training data is available for date"""
        from datetime import datetime
        import boto3

        s3 = boto3.client('s3')
        prefix = f'transactions/{ds}/'
        response = s3.list_objects_v2(Bucket='ml-data', Prefix=prefix)

        if 'Contents' not in response:
            raise ValueError(f"No data found for {ds}")

        total_size = sum(obj['Size'] for obj in response['Contents'])
        return {'date': ds, 'size_gb': total_size / 1e9, 'file_count': len(response['Contents'])}

    # Task 2: Extract features from feature store
    @task
    def extract_features(data_info: dict):
        """Get historical features with point-in-time correctness"""
        from feast import FeatureStore
        import pandas as pd

        store = FeatureStore(repo_path='/opt/feast')

        # Load entity dataframe (transaction IDs + timestamps)
        entity_df = pd.read_parquet(f"s3://ml-data/transactions/{data_info['date']}/entities.parquet")

        # Get features as of transaction time (point-in-time correct)
        training_df = store.get_historical_features(
            entity_df=entity_df,
            features=[
                'user_transaction_features:transaction_count_7d',
                'user_transaction_features:transaction_count_30d',
                'user_transaction_features:avg_transaction_amount',
                'merchant_risk_features:chargeback_rate',
                'merchant_risk_features:total_transactions_30d',
            ]
        ).to_df()

        # Save to S3
        output_path = f"s3://ml-data/training/{data_info['date']}/features.parquet"
        training_df.to_parquet(output_path)

        return {
            'features_path': output_path,
            'row_count': len(training_df),
            'feature_count': len(training_df.columns)
        }

    # Task 3: Data quality checks
    @task
    def data_quality_checks(feature_info: dict):
        """Validate feature quality before training"""
        import pandas as pd
        from great_expectations.core.batch import RuntimeBatchRequest
        from great_expectations.data_context import DataContext

        df = pd.read_parquet(feature_info['features_path'])

        # Great Expectations validation
        context = DataContext('/opt/great_expectations')

        checkpoint_result = context.run_checkpoint(
            checkpoint_name='training_data_checkpoint',
            batch_request=RuntimeBatchRequest(
                datasource_name='training_data',
                data_asset_name='features',
                runtime_parameters={'batch_data': df}
            )
        )

        if not checkpoint_result['success']:
            raise ValueError("Data quality checks failed")

        return {'status': 'passed', 'path': feature_info['features_path']}

    # Task 4: Train model on Kubernetes with GPU
    train_model = KubernetesPodOperator(
        task_id='train_xgboost_model',
        name='train-fraud-detection',
        namespace='ml-training',
        image='ml-training:v1.5.0',
        cmds=['python', '-m', 'ml_pipeline.training.train'],
        arguments=[
            '--data-path', "{{ task_instance.xcom_pull(task_ids='data_quality_checks')['path'] }}",
            '--model-type', 'xgboost',
            '--experiment-name', 'fraud-detection',
            '--run-name', 'daily-training-{{ ds }}'
        ],

        # GPU resources
        container_resources=k8s.V1ResourceRequirements(
            requests={'memory': '32Gi', 'cpu': '8'},
            limits={'memory': '64Gi', 'cpu': '16', 'nvidia.com/gpu': '1'}
        ),

        # Environment
        env_vars={
            'MLFLOW_TRACKING_URI': 'http://mlflow.ml-platform:5000',
            'FEAST_REPO_PATH': '/opt/feast',
            'AWS_DEFAULT_REGION': 'us-west-2'
        },

        # Volumes
        volumes=[
            k8s.V1Volume(
                name='aws-credentials',
                secret=k8s.V1SecretVolumeSource(secret_name='aws-credentials')
            )
        ],
        volume_mounts=[
            k8s.V1VolumeMount(name='aws-credentials', mount_path='/root/.aws', read_only=True)
        ],

        # Execution config
        is_delete_operator_pod=True,
        get_logs=True,
        log_events_on_failure=True,

        # Retry config
        retries=3,
        retry_delay=timedelta(minutes=10),
        execution_timeout=timedelta(hours=4)
    )

    # Task 5: Evaluate model
    @task
    def evaluate_model(ds: str):
        """Evaluate model on holdout test set"""
        import mlflow
        from sklearn.metrics import roc_auc_score, precision_recall_curve
        import pandas as pd

        mlflow.set_tracking_uri('http://mlflow.ml-platform:5000')

        # Get latest model from this run
        run = mlflow.search_runs(
            experiment_names=['fraud-detection'],
            filter_string=f"tags.run_date = '{ds}'",
            order_by=['start_time DESC'],
            max_results=1
        ).iloc[0]

        model = mlflow.xgboost.load_model(f"runs:/{run.run_id}/model")

        # Load test data
        test_df = pd.read_parquet(f"s3://ml-data/test/{ds}/features.parquet")
        X_test = test_df.drop('is_fraud', axis=1)
        y_test = test_df['is_fraud']

        # Predictions
        y_pred_proba = model.predict_proba(X_test)[:, 1]

        # Metrics
        auc = roc_auc_score(y_test, y_pred_proba)
        precision, recall, thresholds = precision_recall_curve(y_test, y_pred_proba)

        # Log metrics to MLflow
        with mlflow.start_run(run_id=run.run_id):
            mlflow.log_metric('test_auc', auc)
            mlflow.log_metric('test_precision_at_90_recall',
                            precision[recall >= 0.90][0] if any(recall >= 0.90) else 0)

        return {
            'run_id': run.run_id,
            'auc': auc,
            'meets_threshold': auc >= 0.88  # Production threshold
        }

    # Task 6: Deployment decision
    @task.branch
    def deployment_gate(eval_results: dict):
        """Decide whether to deploy based on model performance"""
        if eval_results['meets_threshold']:
            return 'deploy_to_staging'
        else:
            return 'alert_low_performance'

    # Task 7a: Deploy to staging
    @task
    def deploy_to_staging(eval_results: dict):
        """Deploy model to staging environment"""
        import mlflow
        from kubernetes import client, config

        config.load_incluster_config()
        k8s_apps = client.AppsV1Api()

        # Register model in MLflow
        mlflow.set_tracking_uri('http://mlflow.ml-platform:5000')
        model_uri = f"runs:/{eval_results['run_id']}/model"

        model_details = mlflow.register_model(
            model_uri=model_uri,
            name='fraud-detection'
        )

        # Transition to staging
        client = mlflow.tracking.MlflowClient()
        client.transition_model_version_stage(
            name='fraud-detection',
            version=model_details.version,
            stage='Staging'
        )

        # Update Kubernetes deployment
        deployment = k8s_apps.read_namespaced_deployment(
            name='fraud-detection-staging',
            namespace='ml-serving'
        )

        deployment.spec.template.spec.containers[0].env = [
            k8s.V1EnvVar(name='MODEL_VERSION', value=str(model_details.version))
        ]

        k8s_apps.patch_namespaced_deployment(
            name='fraud-detection-staging',
            namespace='ml-serving',
            body=deployment
        )

        return {'status': 'deployed', 'version': model_details.version, 'stage': 'Staging'}

    # Task 7b: Alert on low performance
    @task
    def alert_low_performance(eval_results: dict):
        """Send alert if model doesn't meet threshold"""
        from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator

        message = f"""
        🚨 Model Performance Alert

        Training Run: {eval_results['run_id']}
        Test AUC: {eval_results['auc']:.4f}
        Threshold: 0.88

        Model did not meet deployment threshold. Please investigate.
        """

        SlackWebhookOperator(
            task_id='slack_alert',
            http_conn_id='slack_webhook',
            message=message
        ).execute(context={})

        return {'status': 'alerted'}

    # Task 8: Cleanup (always runs)
    @task(trigger_rule=TriggerRule.ALL_DONE)
    def cleanup(ds: str):
        """Clean up temporary files"""
        import boto3

        s3 = boto3.client('s3')

        # Delete temporary feature files
        s3.delete_object(Bucket='ml-data', Key=f'training/{ds}/features.parquet')

        return {'status': 'cleaned'}

    # Define task dependencies
    data = validate_data_availability('{{ ds }}')
    features = extract_features(data)
    quality = data_quality_checks(features)
    train_model.set_upstream(quality)
    eval_results = evaluate_model('{{ ds }}')
    eval_results.set_upstream(train_model)

    decision = deployment_gate(eval_results)
    deploy = deploy_to_staging(eval_results)
    alert = alert_low_performance(eval_results)

    decision >> [deploy, alert]

    cleanup_task = cleanup('{{ ds }}')
    [deploy, alert] >> cleanup_task
```

**Key Patterns in This Pipeline**:

1. **Data Validation**: Check data availability before expensive operations
2. **Feature Store Integration**: Point-in-time correct features from Feast
3. **Data Quality Gates**: Great Expectations validation before training
4. **Kubernetes Execution**: GPU-based training in isolated pod
5. **MLflow Integration**: Experiment tracking and model registry
6. **Conditional Deployment**: Branch based on model performance
7. **Proper Cleanup**: Always run cleanup regardless of success/failure

### 3. Event-Driven Workflows with Kafka

#### Architecture: Kafka + Airflow

**Use Case**: Train model immediately when new labeled data arrives

**Components**:
1. **Kafka Topic**: `ml.training.triggers`
2. **Airflow DAG**: Triggered by Kafka messages
3. **Training Pipeline**: Standard DAG from above

**Implementation**:

**Step 1: Kafka Sensor**
```python
from airflow.providers.apache.kafka.sensors.kafka import AwaitMessageSensor

@dag(schedule_interval=None)  # Event-driven, no schedule
def event_driven_training():

    # Wait for Kafka message
    wait_for_trigger = AwaitMessageSensor(
        task_id='wait_for_training_trigger',
        kafka_config_id='kafka_default',
        topics=['ml.training.requests'],
        apply_function='process_trigger_message',
        xcom_push_key='trigger_info',
        timeout=86400  # 24 hours
    )

    @task
    def start_training(trigger_info: dict):
        """Start training with info from Kafka message"""
        # trigger_info contains: {'data_path': '...', 'priority': 'high', ...}
        return trigger_info

    trigger = wait_for_trigger
    training_config = start_training(trigger)
    # ... rest of training pipeline
```

**Step 2: Kafka Message Producer**
```python
# Separate service that produces training triggers
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Trigger training when new data arrives
def trigger_training(data_path: str, priority: str = 'normal'):
    message = {
        'data_path': data_path,
        'priority': priority,
        'timestamp': datetime.utcnow().isoformat(),
        'triggered_by': 'data-ingestion-service'
    }

    producer.send('ml.training.triggers', value=message)
    producer.flush()
```

**Step 3: DAG Trigger API (Alternative)**
```python
# Trigger DAG via REST API
import requests

def trigger_airflow_dag(data_path: str):
    response = requests.post(
        'http://airflow:8080/api/v1/dags/fraud_detection_training/dagRuns',
        json={
            'conf': {'data_path': data_path},
            'note': 'Triggered by data arrival event'
        },
        auth=('admin', 'password')
    )
    return response.json()
```

### 4. Monitoring and Observability

#### Key Metrics to Track

**1. DAG-Level Metrics**:
- **Success Rate**: Percentage of successful runs
- **Duration**: Total execution time (p50, p95, p99)
- **SLA Misses**: Runs that exceed SLA
- **Failure Rate**: Percentage of failed runs

**2. Task-Level Metrics**:
- **Task Duration**: Execution time per task
- **Retry Count**: How often tasks are retried
- **Resource Usage**: CPU, memory, GPU utilization
- **Queue Time**: Time waiting in queue before execution

**3. System Metrics**:
- **Scheduler Lag**: Time between scheduled start and actual start
- **Executor Queue Depth**: Number of tasks waiting to run
- **Database Connections**: Metadata DB connection pool usage

#### Airflow Metrics Integration

**Prometheus Metrics Export**:
```python
# airflow.cfg
[metrics]
statsd_on = True
statsd_host = localhost
statsd_port = 8125
statsd_prefix = airflow
```

**Custom Metrics in Tasks**:
```python
from airflow.stats import Stats

@task
def train_model():
    start_time = time.time()

    # Training logic
    model = train(...)

    # Record custom metrics
    training_duration = time.time() - start_time
    Stats.timing('ml.training.duration', training_duration)
    Stats.gauge('ml.model.parameters', model.get_num_params())
    Stats.incr('ml.training.success')

    return model
```

**Grafana Dashboard Queries**:
```promql
# Training success rate
rate(airflow_task_success_total{dag_id="fraud_detection_training"}[5m])

# Average training duration
avg(airflow_task_duration_seconds{task_id="train_xgboost_model"})

# Failed tasks
sum(airflow_task_failure_total) by (dag_id, task_id)
```

#### Alerting

**SLA Alerts**:
```python
def sla_miss_callback(dag, task_list, blocking_task_list, slas, blocking_tis):
    """Called when task misses SLA"""
    send_alert(
        channel='#ml-alerts',
        message=f"SLA miss: {slas[0].dag_id} - {slas[0].task_id}",
        severity='warning'
    )

with DAG(
    'fraud_detection_training',
    default_args={'sla': timedelta(hours=4)},
    sla_miss_callback=sla_miss_callback
) as dag:
    ...
```

**Failure Alerts**:
```python
def failure_callback(context):
    """Called when task fails"""
    send_alert(
        channel='#ml-alerts',
        message=f"Task failed: {context['task_instance'].task_id}",
        severity='critical',
        logs=context['task_instance'].log_url
    )

@task(on_failure_callback=failure_callback)
def train_model():
    ...
```

---

## Advanced Topics

### 1. Distributed Training Orchestration

#### Multi-GPU Training with Horovod

**Challenge**: Train large model across multiple GPUs/nodes

**Solution**: Kubernetes + Horovod + MPI

```python
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator

# Horovod distributed training
distributed_training = KubernetesPodOperator(
    task_id='distributed_training_horovod',
    name='horovod-training',
    namespace='ml-training',
    image='horovod/horovod:latest',
    cmds=['mpirun'],
    arguments=[
        '-np', '8',  # 8 processes (GPUs)
        '--hostfile', '/etc/mpi/hostfile',
        '--mca', 'plm_rsh_no_tree_spawn', '1',
        '--bind-to', 'none',
        '--map-by', 'slot',
        '-x', 'NCCL_DEBUG=INFO',
        '-x', 'LD_LIBRARY_PATH',
        '-x', 'PATH',
        'python', 'train_distributed.py',
        '--batch-size', '256',
        '--epochs', '50'
    ],

    # Request 8 GPUs across nodes
    container_resources=k8s.V1ResourceRequirements(
        requests={'nvidia.com/gpu': '8', 'memory': '128Gi', 'cpu': '32'},
        limits={'nvidia.com/gpu': '8', 'memory': '256Gi', 'cpu': '64'}
    ),

    # Node affinity for GPU nodes
    affinity=k8s.V1Affinity(
        node_affinity=k8s.V1NodeAffinity(
            required_during_scheduling_ignored_during_execution=k8s.V1NodeSelector(
                node_selector_terms=[
                    k8s.V1NodeSelectorTerm(
                        match_expressions=[
                            k8s.V1NodeSelectorRequirement(
                                key='node-type',
                                operator='In',
                                values=['gpu-a100']
                            )
                        ]
                    )
                ]
            )
        )
    ),

    execution_timeout=timedelta(hours=12)
)
```

#### Distributed Hyperparameter Tuning

**Pattern**: Launch multiple training jobs with different hyperparameters in parallel

```python
from airflow.decorators import task

@task
def generate_hyperparameter_configs():
    """Generate grid of hyperparameters"""
    import itertools

    learning_rates = [0.001, 0.01, 0.1]
    max_depths = [3, 5, 7, 10]
    n_estimators = [100, 200, 500]

    configs = []
    for lr, depth, n_est in itertools.product(learning_rates, max_depths, n_estimators):
        configs.append({
            'learning_rate': lr,
            'max_depth': depth,
            'n_estimators': n_est,
            'config_id': f'lr{lr}_d{depth}_n{n_est}'
        })

    return configs  # 36 configurations

@task
def train_with_config(config: dict):
    """Train model with specific hyperparameters"""
    import mlflow
    from xgboost import XGBClassifier

    with mlflow.start_run(run_name=config['config_id']):
        mlflow.log_params(config)

        model = XGBClassifier(**config)
        model.fit(X_train, y_train)

        auc = roc_auc_score(y_val, model.predict_proba(X_val)[:, 1])
        mlflow.log_metric('val_auc', auc)

        return {'config_id': config['config_id'], 'auc': auc}

@task
def select_best_model(results: list):
    """Select best configuration"""
    best = max(results, key=lambda x: x['auc'])

    # Retrain with best config on full dataset
    retrain_with_best(best['config_id'])

    return best

@dag(...)
def hyperparameter_tuning():
    configs = generate_hyperparameter_configs()
    results = train_with_config.expand(config=configs)  # 36 parallel tasks!
    best = select_best_model(results)

hyperparameter_tuning()
```

### 2. CI/CD for ML Pipelines

#### Testing DAGs

**Unit Tests**:
```python
# tests/test_training_dag.py
from airflow.models import DagBag
import pytest

def test_dag_loads_successfully():
    """Test DAG can be imported without errors"""
    dag_bag = DagBag(dag_folder='dags/', include_examples=False)
    assert len(dag_bag.import_errors) == 0

def test_dag_has_correct_tags():
    """Test DAG has expected tags"""
    dag_bag = DagBag(dag_folder='dags/')
    dag = dag_bag.get_dag('fraud_detection_training')
    assert 'ml' in dag.tags
    assert 'training' in dag.tags

def test_task_dependencies():
    """Test task dependency graph"""
    dag_bag = DagBag(dag_folder='dags/')
    dag = dag_bag.get_dag('fraud_detection_training')

    # Check specific dependencies
    validate_task = dag.get_task('validate_data_availability')
    extract_task = dag.get_task('extract_features')

    assert extract_task in validate_task.downstream_list
```

**Integration Tests**:
```python
def test_training_pipeline_end_to_end():
    """Test complete pipeline with sample data"""
    from airflow.models import DagRun
    from airflow.utils.state import State
    from datetime import datetime

    dag_bag = DagBag(dag_folder='dags/')
    dag = dag_bag.get_dag('fraud_detection_training')

    # Trigger DAG run
    execution_date = datetime(2025, 1, 1)
    dag_run = dag.create_dagrun(
        run_id=f'test_{execution_date}',
        execution_date=execution_date,
        state=State.RUNNING,
        conf={'data_path': 's3://test-bucket/sample-data'}
    )

    # Execute all tasks
    for task_instance in dag_run.get_task_instances():
        task_instance.run()

    # Assert final state
    assert dag_run.get_state() == State.SUCCESS
```

#### CI/CD Pipeline

**GitHub Actions Workflow**:
```yaml
# .github/workflows/airflow-ci.yml
name: Airflow DAG CI/CD

on:
  push:
    branches: [main, develop]
    paths:
      - 'dags/**'
      - 'plugins/**'
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Install dependencies
        run: |
          pip install apache-airflow==2.7.0
          pip install -r requirements.txt
      - name: Lint with flake8
        run: |
          flake8 dags/ --max-line-length=120
      - name: Type check with mypy
        run: |
          mypy dags/ --ignore-missing-imports

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Install Airflow
        run: |
          pip install apache-airflow==2.7.0
          pip install -r requirements.txt
      - name: Initialize Airflow DB
        run: |
          airflow db init
      - name: Run unit tests
        run: |
          pytest tests/ -v
      - name: Check DAG integrity
        run: |
          python -m airflow dags test fraud_detection_training 2025-01-01

  deploy-dev:
    needs: [lint, test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Dev Airflow
        run: |
          # Sync DAGs to dev environment
          aws s3 sync dags/ s3://airflow-dev-dags/ --delete
          # Trigger DAG sync in Airflow
          curl -X POST https://airflow-dev.company.com/api/v1/dags/reload

  deploy-prod:
    needs: [lint, test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Prod Airflow
        run: |
          # Sync DAGs to production
          aws s3 sync dags/ s3://airflow-prod-dags/ --delete
          # Trigger DAG sync
          curl -X POST https://airflow-prod.company.com/api/v1/dags/reload
```

### 3. Cost Optimization

#### Resource-Based Task Routing

**Problem**: Not all tasks need expensive resources (GPUs, large memory)

**Solution**: Use task pools and queues to route tasks to appropriate executors

```python
# Small tasks: Run on general workers
@task(pool='small_tasks', queue='general')
def validate_data():
    # Light validation logic
    pass

# CPU-intensive: Run on CPU-optimized nodes
@task(pool='cpu_tasks', queue='cpu-optimized')
def feature_engineering():
    # Pandas/Spark processing
    pass

# GPU training: Run on GPU nodes
train_task = KubernetesPodOperator(
    task_id='train',
    queue='gpu',
    container_resources=k8s.V1ResourceRequirements(
        limits={'nvidia.com/gpu': '1'}
    )
)
```

**Airflow Pools** (limit concurrency):
```python
# airflow.cfg or via UI
[pools]
small_tasks = 50
cpu_tasks = 10
gpu_tasks = 2  # Limit expensive GPU tasks
```

#### Spot Instance Usage

**Pattern**: Use spot instances for fault-tolerant training tasks

```python
train_with_spot = KubernetesPodOperator(
    task_id='train_on_spot',
    # Tolerate spot instance node taints
    tolerations=[
        k8s.V1Toleration(
            key='node-lifecycle',
            operator='Equal',
            value='spot',
            effect='NoSchedule'
        )
    ],
    # Require spot instances
    node_selector={'node-lifecycle': 'spot'},

    # Aggressive retry policy (spot instances can be preempted)
    retries=5,
    retry_delay=timedelta(minutes=15),
    retry_exponential_backoff=True,
    max_retry_delay=timedelta(hours=2)
)
```

#### Dynamic Resource Allocation

**Pattern**: Adjust resources based on data size

```python
@task
def calculate_training_resources(data_info: dict):
    """Calculate optimal resources based on data size"""
    data_size_gb = data_info['size_gb']

    if data_size_gb < 10:
        return {'memory': '8Gi', 'cpu': '4', 'gpu': '0'}
    elif data_size_gb < 100:
        return {'memory': '32Gi', 'cpu': '16', 'gpu': '1'}
    else:
        return {'memory': '128Gi', 'cpu': '32', 'gpu': '4'}

@task
def train_with_dynamic_resources(resources: dict):
    # Use calculated resources
    ...
```

---

## Real-World Applications

### Case Study 1: Airbnb's ML Workflows

**Scale**:
- 1,000+ ML models in production
- 10,000+ Airflow DAGs
- 200,000+ daily task executions

**Architecture**:
- **Airflow on Kubernetes**: Each task runs in isolated pod
- **Dynamic Task Generation**: Auto-generate training tasks for new markets
- **Feature Store Integration**: Zipline feature store + Airflow for materialization
- **Multi-tenancy**: 50+ data science teams share platform

**Key Workflow**:
```
1. Feature Computation (Spark) → 2. Feature Validation → 3. Materialize to Online Store
                                                          ↓
4. Training Data Generation ← 5. Model Training (TensorFlow) → 6. Model Evaluation
                                                                  ↓
                                                      7. A/B Test Deployment
```

**Results**:
- **5x faster** model iteration (days → hours)
- **80% reduction** in feature engineering duplication
- **99.9% SLA** for production pipelines

### Case Study 2: Netflix's Orchestration Platform

**Scale**:
- 2,000+ DAGs
- 150,000+ daily task executions
- Processes 1 PB+ data daily

**Technology Stack**:
- **Conductor** (Netflix's workflow engine, similar to Airflow)
- **Titus** (container platform on AWS)
- **Meson** (ML platform using Conductor)

**Key Innovation**: Event-driven retraining
```
User behavior change → Kafka event → Trigger retraining → Deploy if better → Monitor impact
```

**Workflow Example**:
```python
# Recommendation model retraining (simplified)
1. Detect distribution shift (monitoring system)
2. Trigger retraining via Kafka event
3. Fetch latest user interactions (Spark)
4. Generate features (feature store)
5. Train multiple models in parallel (A100 GPUs)
6. Offline evaluation (held-out test set)
7. Online A/B test (1% traffic)
8. Gradual rollout if successful
```

**Results**:
- **Automated retraining**: 40% of models retrain automatically
- **Response time**: <4 hours from drift detection to deployment
- **Resource efficiency**: 70% cluster utilization vs 35% without orchestration

### Case Study 3: Uber's Michelangelo + Airflow

**Integration**:
- Michelangelo (ML platform) uses Airflow for orchestration
- 10,000+ features, 1,000+ models

**Workflows**:

**1. Continuous Training**:
```
Schedule: Hourly for real-time models, daily for batch
├─ Fetch new data (Hive)
├─ Compute features (Spark + Feast)
├─ Train model (Horovod distributed)
├─ Evaluate (A/B test framework)
└─ Deploy if metrics improve
```

**2. Feature Backfill**:
```
Trigger: New feature added
├─ Compute feature for historical data (6 months)
├─ Validate consistency
├─ Materialize to offline store
└─ Update online store
```

**Results**:
- **90% time savings** on feature engineering
- **10M predictions/second** serving capacity
- **<5ms p99** feature retrieval latency

---

## Assessment

### Knowledge Check Questions

**Question 1**: What is the primary purpose of using a DAG in workflow orchestration?

A) To enable parallel execution of all tasks
B) To define task dependencies and execution order
C) To store intermediate results between tasks
D) To schedule tasks at fixed intervals

<details>
<summary>Answer</summary>
**B) To define task dependencies and execution order**

Explanation: DAGs (Directed Acyclic Graphs) explicitly define which tasks depend on others, ensuring correct execution order and enabling parallel execution where dependencies allow.
</details>

---

**Question 2**: When should you use XCom to pass data between Airflow tasks?

A) Always, it's the recommended approach
B) For datasets larger than 1GB
C) Only for small metadata (<1MB) like file paths or IDs
D) Never, always use external storage

<details>
<summary>Answer</summary>
**C) Only for small metadata (<1MB) like file paths or IDs**

Explanation: XCom stores data in the metadata database, which is limited in size. For large datasets, pass only the storage path (e.g., S3 URI) via XCom, not the data itself.
</details>

---

**Question 3**: Which Airflow executor is best suited for ML training workloads requiring GPUs?

A) LocalExecutor
B) CeleryExecutor
C) KubernetesExecutor
D) SequentialExecutor

<details>
<summary>Answer</summary>
**C) KubernetesExecutor**

Explanation: KubernetesExecutor creates a pod per task, allowing you to specify GPU requirements, memory, and CPU per task. This provides better resource isolation and efficient GPU utilization compared to shared worker pools.
</details>

---

**Question 4**: What does it mean for a task to be idempotent?

A) It runs very quickly
B) Running it multiple times with the same input produces the same output
C) It doesn't depend on any other tasks
D) It always succeeds

<details>
<summary>Answer</summary>
**B) Running it multiple times with the same input produces the same output**

Explanation: Idempotency ensures that retrying a task doesn't cause unintended side effects (like duplicate data). This is critical for reliable pipelines with automatic retries.
</details>

---

**Question 5**: In event-driven workflows, what triggers a DAG to run?

A) Only time-based schedules (cron)
B) External events like Kafka messages, file arrivals, or API calls
C) Manual user intervention
D) Completion of the previous run

<details>
<summary>Answer</summary>
**B) External events like Kafka messages, file arrivals, or API calls**

Explanation: Event-driven workflows respond to external triggers rather than running on fixed schedules, enabling real-time ML pipelines that react to data availability or system events.
</details>

---

**Question 6**: What is the purpose of Airflow's BranchPythonOperator?

A) To run tasks in parallel
B) To conditionally execute different downstream tasks based on logic
C) To split data across multiple workers
D) To retry failed tasks

<details>
<summary>Answer</summary>
**B) To conditionally execute different downstream tasks based on logic**

Explanation: BranchPythonOperator evaluates a condition and selects which downstream task(s) to execute, enabling conditional workflows (e.g., deploy model only if metrics meet threshold).
</details>

---

**Question 7**: Why is it important to avoid cycles in DAGs?

A) Cycles make the DAG run slower
B) Cycles prevent the scheduler from determining task order
C) Cycles consume too much memory
D) Cycles are actually allowed in Airflow 2.0+

<details>
<summary>Answer</summary>
**B) Cycles prevent the scheduler from determining task order**

Explanation: A cycle (A → B → C → A) creates infinite loops where tasks depend on themselves transitively. DAGs must be acyclic to have a well-defined execution order.
</details>

---

**Question 8**: What is the benefit of using Airflow Datasets for scheduling?

A) Faster execution
B) Automatic data-aware triggering when upstream DAGs produce data
C) Reduced memory usage
D) Better error handling

<details>
<summary>Answer</summary>
**B) Automatic data-aware triggering when upstream DAGs produce data**

Explanation: Datasets enable data-driven scheduling where a DAG runs automatically when its input datasets are updated by upstream DAGs, replacing complex time-based coordination.
</details>

---

**Question 9**: In a distributed training workflow, what is the role of the orchestrator (Airflow)?

A) To perform the actual model training
B) To coordinate resource allocation, job submission, and monitoring
C) To store training data
D) To serve predictions

<details>
<summary>Answer</summary>
**B) To coordinate resource allocation, job submission, and monitoring**

Explanation: The orchestrator manages the workflow but delegates actual training to specialized executors (like Kubernetes pods with GPUs). It handles dependencies, retries, and monitoring.
</details>

---

**Question 10**: What is a key difference between workflow orchestration (Airflow) and task queues (Celery)?

A) Orchestration is faster
B) Orchestration handles complex dependencies; task queues are for independent jobs
C) Task queues are more scalable
D) They are the same thing

<details>
<summary>Answer</summary>
**B) Orchestration handles complex dependencies; task queues are for independent jobs**

Explanation: Workflow orchestrators excel at managing multi-step pipelines with dependencies (DAGs), while task queues like Celery are better for independent, on-demand tasks in applications.
</details>

---

## Summary

In this lecture, you learned:

1. **DAG-Based Workflow Design**: How to structure ML pipelines as Directed Acyclic Graphs with proper task atomicity, idempotency, and modularity

2. **Scheduling Strategies**: Time-based (cron), event-driven (Kafka, file sensors), and hybrid scheduling approaches for different ML use cases

3. **Executor Models**: Trade-offs between LocalExecutor, CeleryExecutor, and KubernetesExecutor, with Kubernetes recommended for ML workloads requiring GPUs

4. **Data Passing**: Best practices for passing data between tasks using external storage (S3, feature stores) rather than XCom for large datasets

5. **Airflow 2.0+ Features**: TaskFlow API, dynamic task mapping, and Datasets for modern Pythonic workflows

6. **Production ML Patterns**: Complete training pipeline with validation, feature stores, model registry, conditional deployment, and cleanup

7. **Event-Driven Workflows**: Integrating Kafka and external triggers for real-time ML pipelines

8. **Monitoring & Observability**: Metrics, alerting, and SLA tracking for production workflows

9. **Advanced Topics**: Distributed training orchestration, CI/CD for DAGs, and cost optimization strategies

10. **Real-World Scale**: How Airbnb, Netflix, and Uber use orchestration to manage thousands of models and millions of daily tasks

---

## Next Steps

**Practice**:
- Complete all 6 hands-on exercises in this module
- Build end-to-end training pipeline for your own ML project
- Implement event-driven workflow with Kafka

**Deepen Knowledge**:
- Explore Airflow provider packages (AWS, GCP, Kubernetes)
- Study Argo Workflows and Kubeflow Pipelines as alternatives
- Learn Prefect for Python-native orchestration

**Production Readiness**:
- Set up monitoring and alerting for your DAGs
- Implement CI/CD pipeline for DAG deployments
- Optimize costs with spot instances and resource pools

---

**Additional Resources**:
- [Apache Airflow Documentation](https://airflow.apache.org/docs/)
- [Astronomer Airflow Guides](https://www.astronomer.io/docs/learn/)
- [Airflow Summit Talks](https://airflowsummit.org/)
- [Netflix Tech Blog: Conductor](https://netflixtechblog.com/conductor-a-microservices-orchestrator-2e8d4771bf40)
- [Uber Engineering: Michelangelo](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
- [Airbnb Engineering: Bighead](https://medium.com/airbnb-engineering/bighead-airbnbs-end-to-end-machine-learning-platform-f0b2e2c0caa0)

---

**Total Words**: ~6,500
**Reading Time**: ~90 minutes
**Status**: ✅ Complete
