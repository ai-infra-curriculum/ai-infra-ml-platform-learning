# Module 06 Exercises: Model Registry & Versioning

Complete hands-on exercises to master MLflow Model Registry for production ML systems.

**Total Time**: ~8.5 hours
**Difficulty**: Intermediate to Advanced

---

## Exercise Overview

| Exercise | Title | Duration | Difficulty | Key Skills |
|----------|-------|----------|------------|------------|
| 01 | Set Up MLflow Model Registry | 60 min | Basic | Installation, configuration, UI |
| 02 | Register and Version Models | 75 min | Intermediate | Registration, lineage, metadata |
| 03 | Model Lifecycle Management | 90 min | Intermediate | Stages, transitions, approval |
| 04 | Model Serving with Registry | 120 min | Advanced | Loading, deployment, versioning |
| 05 | Build Approval Workflow | 90 min | Advanced | Validation, approvals, automation |
| 06 | A/B Testing with Model Registry | 90 min | Advanced | Multi-version serving, metrics |

---

## Prerequisites

Before starting these exercises, ensure you have:

- [x] Completed Module 05 (Workflow Orchestration)
- [x] Python 3.9+
- [x] Docker and Docker Compose
- [x] MLflow 2.9+
- [x] PostgreSQL (via Docker)
- [x] Understanding of ML model lifecycle

**Installation**:
```bash
# Install MLflow
pip install mlflow==2.9.0 psycopg2-binary boto3

# Start MLflow with Docker Compose (from lecture notes)
docker-compose up -d

# Verify installation
mlflow --version
curl http://localhost:5000/health
```

---

## Exercise 01: Set Up MLflow Model Registry

**Duration**: 60 minutes
**Difficulty**: Basic

### Learning Objectives

- Set up MLflow tracking server with PostgreSQL backend
- Configure S3-compatible artifact storage (MinIO)
- Navigate the MLflow UI
- Understand registry architecture

### Tasks

#### Task 1.1: Deploy MLflow Stack

Create `docker-compose.yml` (from lecture notes) and start services:

```bash
docker-compose up -d
docker ps  # Verify all services running
```

#### Task 1.2: Configure Python Client

```python
import mlflow

# Set tracking URI
mlflow.set_tracking_uri("http://localhost:5000")

# Test connection
experiments = mlflow.search_experiments()
print(f"Connected! Found {len(experiments)} experiments")
```

#### Task 1.3: Explore the UI

Navigate to http://localhost:5000 and explore:
- Experiments tab
- Models tab (empty for now)
- Understand the layout

### Success Criteria

- [ ] All Docker containers running (mlflow, postgres, minio)
- [ ] Python client successfully connects
- [ ] MLflow UI accessible at localhost:5000
- [ ] Can create a test experiment via UI

---

## Exercise 02: Register and Version Models

**Duration**: 75 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Train and log models with MLflow
- Register models to the registry
- Track model lineage
- Add metadata and tags
- Query model versions

### Tasks

#### Task 2.1: Train and Register First Model

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score
from mlflow.models.signature import infer_signature

mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("fraud-detection-training")

# Generate sample data
X, y = make_classification(n_samples=10000, n_features=20, n_classes=2, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

with mlflow.start_run(run_name="rf-baseline") as run:
    # Parameters
    params = {"n_estimators": 100, "max_depth": 5, "random_state": 42}
    mlflow.log_params(params)

    # Train
    model = RandomForestClassifier(**params)
    model.fit(X_train, y_train)

    # Metrics
    train_auc = roc_auc_score(y_train, model.predict_proba(X_train)[:, 1])
    test_auc = roc_auc_score(y_test, model.predict_proba(X_test)[:, 1])
    mlflow.log_metrics({"train_auc": train_auc, "test_auc": test_auc})

    # Signature
    signature = infer_signature(X_train, model.predict_proba(X_train))

    # Register model
    model_info = mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="model",
        signature=signature,
        input_example=X_train[:5],
        registered_model_name="fraud-detection"
    )

    print(f"Model registered: {model_info.model_uri}")
    print(f"Run ID: {run.info.run_id}")
```

#### Task 2.2: Register Multiple Versions

Train 3 more versions with different hyperparameters:
- Version 2: `max_depth=7`
- Version 3: `n_estimators=200`
- Version 4: `max_depth=10, n_estimators=200`

#### Task 2.3: Add Metadata

```python
from mlflow import MlflowClient

client = MlflowClient()

# Update description
client.update_model_version(
    name="fraud-detection",
    version=4,
    description="Best performing model with max_depth=10, n_estimators=200. Test AUC: 0.XX"
)

# Add tags
client.set_model_version_tag("fraud-detection", 4, "validated", "true")
client.set_model_version_tag("fraud-detection", 4, "performance_tier", "high")
```

#### Task 2.4: Query and Compare Versions

```python
# Get all versions
versions = client.search_model_versions("name='fraud-detection'")

print(f"{'Version':<10} {'Test AUC':<12} {'Stage':<15} {'Description':<30}")
print("-" * 70)

for v in versions:
    run = client.get_run(v.run_id)
    test_auc = run.data.metrics.get("test_auc", 0.0)
    desc = v.description[:30] if v.description else "N/A"
    print(f"{v.version:<10} {test_auc:<12.4f} {v.current_stage:<15} {desc:<30}")
```

### Success Criteria

- [ ] 4 model versions registered
- [ ] Each version has complete lineage (params, metrics, artifacts)
- [ ] Descriptions and tags added
- [ ] Can query and compare versions programmatically

---

## Exercise 03: Model Lifecycle Management

**Duration**: 90 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Transition models through stages (None → Staging → Production)
- Implement single-production-version constraint
- Archive old models
- Track stage transition history

### Tasks

#### Task 3.1: Implement Stage Transitions

```python
from mlflow import MlflowClient

client = MlflowClient()

def transition_model_stage(name: str, version: int, stage: str):
    """Transition model to new stage with logging"""
    client.transition_model_version_stage(
        name=name,
        version=version,
        stage=stage
    )
    print(f"✅ Model {name} v{version} transitioned to {stage}")

# Transition best model to Staging
transition_model_stage("fraud-detection", 4, "Staging")
```

#### Task 3.2: Implement Single-Production Rule

```python
def promote_to_production(name: str, new_version: int):
    """
    Promote model to Production, archiving current Production model.
    Enforces single-production-version constraint.
    """
    # Get current Production model
    current_prod = client.get_latest_versions(name, stages=["Production"])

    if current_prod:
        old_version = current_prod[0].version
        print(f"Archiving current Production model: v{old_version}")
        client.transition_model_version_stage(name, old_version, "Archived")

    # Promote new version
    print(f"Promoting v{new_version} to Production")
    client.transition_model_version_stage(name, new_version, "Production")

    print(f"✅ Model v{new_version} now in Production")

# Promote version 4 to Production
promote_to_production("fraud-detection", 4)
```

#### Task 3.3: Build Stage History Tracker

```python
def get_stage_history(name: str, version: int):
    """Get complete stage transition history for a model version"""
    from datetime import datetime

    model_version = client.get_model_version(name, version)

    # Get activity log (if available in your MLflow version)
    # For now, show current state
    print(f"Model: {name} v{version}")
    print(f"Current Stage: {model_version.current_stage}")
    print(f"Last Updated: {datetime.fromtimestamp(model_version.last_updated_timestamp / 1000)}")
    print(f"Status: {model_version.status}")

get_stage_history("fraud-detection", 4)
```

### Success Criteria

- [ ] Models successfully transition through stages
- [ ] Only one model in Production at a time
- [ ] Old Production models archived automatically
- [ ] Can query current and historical stages

---

## Exercise 04: Model Serving with Registry

**Duration**: 120 minutes
**Difficulty**: Advanced

### Learning Objectives

- Load models from registry by version and stage
- Build FastAPI serving endpoint
- Implement version-pinned deployment
- Support dynamic model reloading

### Tasks

#### Task 4.1: Simple Model Loading

```python
import mlflow.pyfunc

# Load by version
model_v3 = mlflow.pyfunc.load_model("models:/fraud-detection/3")

# Load by stage
model_prod = mlflow.pyfunc.load_model("models:/fraud-detection/Production")

# Test prediction
import numpy as np
sample_data = np.random.randn(1, 20)
prediction = model_prod.predict(sample_data)
print(f"Prediction: {prediction}")
```

#### Task 4.2: Build FastAPI Serving Endpoint

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import mlflow.pyfunc
import numpy as np
from typing import List

app = FastAPI(title="Fraud Detection API")

# Load model at startup
model = mlflow.pyfunc.load_model("models:/fraud-detection/Production")

class PredictionRequest(BaseModel):
    features: List[float]

class PredictionResponse(BaseModel):
    fraud_probability: float
    model_version: str

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    """Predict fraud probability"""
    if len(request.features) != 20:
        raise HTTPException(status_code=400, detail="Expected 20 features")

    # Reshape for sklearn
    X = np.array(request.features).reshape(1, -1)

    # Predict
    prediction = model.predict(X)[0]

    return PredictionResponse(
        fraud_probability=float(prediction),
        model_version="Production"  # TODO: Get actual version
    )

@app.get("/health")
def health():
    return {"status": "healthy", "model_loaded": model is not None}

# Run with: uvicorn serving:app --reload
```

#### Task 4.3: Implement Dynamic Model Reloading

```python
import time
from threading import Thread
from mlflow import MlflowClient

class DynamicModelServer:
    def __init__(self, model_name: str, stage: str = "Production", check_interval: int = 60):
        self.model_name = model_name
        self.stage = stage
        self.check_interval = check_interval
        self.client = MlflowClient()

        self.current_model = None
        self.current_version = None

        # Initial load
        self.reload_model()

        # Start background thread to check for updates
        self.reload_thread = Thread(target=self._reload_loop, daemon=True)
        self.reload_thread.start()

    def reload_model(self):
        """Check for new model version and reload if needed"""
        latest = self.client.get_latest_versions(self.model_name, stages=[self.stage])

        if not latest:
            print(f"⚠️  No model found in {self.stage} stage")
            return

        latest_version = latest[0].version

        if latest_version != self.current_version:
            print(f"🔄 Reloading model: v{self.current_version} → v{latest_version}")
            model_uri = f"models:/{self.model_name}/{self.stage}"
            self.current_model = mlflow.pyfunc.load_model(model_uri)
            self.current_version = latest_version
            print(f"✅ Model v{latest_version} loaded")

    def _reload_loop(self):
        """Background thread to periodically check for updates"""
        while True:
            time.sleep(self.check_interval)
            try:
                self.reload_model()
            except Exception as e:
                print(f"❌ Reload failed: {e}")

    def predict(self, X):
        """Make prediction with current model"""
        if self.current_model is None:
            raise ValueError("No model loaded")
        return self.current_model.predict(X)

# Usage
server = DynamicModelServer("fraud-detection", stage="Production", check_interval=60)
prediction = server.predict(np.random.randn(1, 20))
print(f"Prediction: {prediction}")
```

### Success Criteria

- [ ] Can load models by version and stage
- [ ] FastAPI endpoint serves predictions
- [ ] Dynamic reloader detects new Production models
- [ ] Server automatically updates without downtime

---

## Exercise 05: Build Approval Workflow

**Duration**: 90 minutes
**Difficulty**: Advanced

### Learning Objectives

- Implement model approval workflow with tags
- Validate models before promotion
- Track approvers and timestamps
- Send notifications

### Tasks

#### Task 5.1: Implement Approval Workflow

```python
from enum import Enum
from datetime import datetime
from mlflow import MlflowClient

class ApprovalStatus(Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"

class ModelApprovalWorkflow:
    def __init__(self, min_auc_threshold: float = 0.85):
        self.client = MlflowClient()
        self.min_auc_threshold = min_auc_threshold

    def request_approval(self, name: str, version: int, requester: str, reason: str):
        """Request approval for Production deployment"""
        # Set approval metadata
        self.client.set_model_version_tag(name, version, "approval_status", ApprovalStatus.PENDING.value)
        self.client.set_model_version_tag(name, version, "requested_by", requester)
        self.client.set_model_version_tag(name, version, "requested_at", datetime.now().isoformat())
        self.client.set_model_version_tag(name, version, "approval_reason", reason)

        print(f"📋 Approval requested for {name} v{version} by {requester}")
        print(f"   Reason: {reason}")

    def validate_model(self, name: str, version: int) -> dict:
        """Run validation checks on model"""
        model_version = self.client.get_model_version(name, version)
        run = self.client.get_run(model_version.run_id)

        # Check 1: AUC threshold
        test_auc = run.data.metrics.get("test_auc", 0.0)
        auc_pass = test_auc >= self.min_auc_threshold

        # Check 2: Has description
        has_description = bool(model_version.description)

        # Check 3: Has required tags
        has_validated_tag = model_version.tags.get("validated") == "true"

        validation_results = {
            "auc_threshold": {"passed": auc_pass, "value": test_auc, "threshold": self.min_auc_threshold},
            "has_description": {"passed": has_description},
            "has_validated_tag": {"passed": has_validated_tag},
            "overall_passed": auc_pass and has_description and has_validated_tag
        }

        return validation_results

    def approve(self, name: str, version: int, approver: str, comment: str = ""):
        """Approve model for Production"""
        # Validate model
        validation = self.validate_model(name, version)

        if not validation["overall_passed"]:
            print("❌ Validation failed:")
            for check, result in validation.items():
                if check != "overall_passed" and not result["passed"]:
                    print(f"   - {check}: FAILED")
            raise ValueError("Model failed validation checks")

        # Update approval status
        self.client.set_model_version_tag(name, version, "approval_status", ApprovalStatus.APPROVED.value)
        self.client.set_model_version_tag(name, version, "approved_by", approver)
        self.client.set_model_version_tag(name, version, "approved_at", datetime.now().isoformat())
        if comment:
            self.client.set_model_version_tag(name, version, "approval_comment", comment)

        # Promote to Production
        self.client.transition_model_version_stage(name, version, "Production")

        print(f"✅ Model {name} v{version} approved by {approver} and promoted to Production")
        if comment:
            print(f"   Comment: {comment}")

    def reject(self, name: str, version: int, reviewer: str, reason: str):
        """Reject model for Production"""
        self.client.set_model_version_tag(name, version, "approval_status", ApprovalStatus.REJECTED.value)
        self.client.set_model_version_tag(name, version, "rejected_by", reviewer)
        self.client.set_model_version_tag(name, version, "rejected_at", datetime.now().isoformat())
        self.client.set_model_version_tag(name, version, "rejection_reason", reason)

        print(f"❌ Model {name} v{version} rejected by {reviewer}")
        print(f"   Reason: {reason}")

# Usage
workflow = ModelApprovalWorkflow(min_auc_threshold=0.85)

# Request approval
workflow.request_approval(
    "fraud-detection",
    4,
    requester="data-scientist@company.com",
    reason="Improved performance with new features"
)

# Approve
workflow.approve(
    "fraud-detection",
    4,
    approver="ml-lead@company.com",
    comment="Performance validated in shadow mode. Approved for production."
)
```

### Success Criteria

- [ ] Approval workflow tracks requests and decisions
- [ ] Validation checks enforce quality standards
- [ ] All metadata tracked (requester, approver, timestamps)
- [ ] Rejected models cannot be promoted

---

## Exercise 06: A/B Testing with Model Registry

**Duration**: 90 minutes
**Difficulty**: Advanced

### Learning Objectives

- Tag models for A/B testing
- Implement traffic splitting logic
- Track A/B test metrics
- Promote winning model

### Tasks

#### Task 6.1: A/B Test Setup

```python
from mlflow import MlflowClient
from datetime import datetime

class ABTestManager:
    def __init__(self):
        self.client = MlflowClient()

    def start_test(self, model_name: str, control_version: int, treatment_version: int, traffic_split: float = 0.1):
        """Start A/B test with control and treatment models"""
        # Tag control
        self.client.set_model_version_tag(model_name, control_version, "ab_role", "control")
        self.client.set_model_version_tag(model_name, control_version, "ab_traffic", str(1 - traffic_split))

        # Tag treatment
        self.client.set_model_version_tag(model_name, treatment_version, "ab_role", "treatment")
        self.client.set_model_version_tag(model_name, treatment_version, "ab_traffic", str(traffic_split))
        self.client.set_model_version_tag(model_name, treatment_version, "ab_start_time", datetime.now().isoformat())

        print(f"🧪 A/B Test Started:")
        print(f"   Control: v{control_version} ({int((1-traffic_split)*100)}% traffic)")
        print(f"   Treatment: v{treatment_version} ({int(traffic_split*100)}% traffic)")

    def get_model_for_request(self, model_name: str, request_id: str):
        """Select model version based on traffic split"""
        import hashlib

        # Get test config
        versions = self.client.search_model_versions(f"name='{model_name}'")
        test_versions = [v for v in versions if v.tags.get("ab_role")]

        if not test_versions:
            # No A/B test, return Production
            return "Production"

        control = [v for v in test_versions if v.tags.get("ab_role") == "control"][0]
        treatment = [v for v in test_versions if v.tags.get("ab_role") == "treatment"][0]
        traffic_split = float(treatment.tags.get("ab_traffic", "0.1"))

        # Hash-based routing (consistent per request_id)
        hash_value = int(hashlib.md5(request_id.encode()).hexdigest(), 16)
        route_to_treatment = (hash_value % 100) < (traffic_split * 100)

        if route_to_treatment:
            return treatment.version
        else:
            return control.version

# Usage
ab_manager = ABTestManager()

# Start A/B test: v3 (control) vs v4 (treatment)
ab_manager.start_test("fraud-detection", control_version=3, treatment_version=4, traffic_split=0.1)

# Route requests
request_id = "user_12345_txn_67890"
version = ab_manager.get_model_for_request("fraud-detection", request_id)
print(f"Route request {request_id} to model v{version}")
```

#### Task 6.2: Simulate A/B Test with Metrics

```python
import numpy as np
from collections import defaultdict

def simulate_ab_test(control_model, treatment_model, n_requests: int = 1000, traffic_split: float = 0.1):
    """Simulate A/B test and collect metrics"""
    control_predictions = []
    treatment_predictions = []

    X_test = np.random.randn(n_requests, 20)
    y_true = np.random.binomial(1, 0.1, n_requests)  # 10% fraud rate

    for i in range(n_requests):
        # Route based on traffic split
        if np.random.random() < traffic_split:
            # Treatment
            pred = treatment_model.predict(X_test[i:i+1])[0]
            treatment_predictions.append({"pred": pred, "true": y_true[i]})
        else:
            # Control
            pred = control_model.predict(X_test[i:i+1])[0]
            control_predictions.append({"pred": pred, "true": y_true[i]})

    # Calculate metrics
    from sklearn.metrics import roc_auc_score

    control_auc = roc_auc_score(
        [p["true"] for p in control_predictions],
        [p["pred"] for p in control_predictions]
    )

    treatment_auc = roc_auc_score(
        [p["true"] for p in treatment_predictions],
        [p["pred"] for p in treatment_predictions]
    )

    results = {
        "control": {
            "auc": control_auc,
            "n_requests": len(control_predictions)
        },
        "treatment": {
            "auc": treatment_auc,
            "n_requests": len(treatment_predictions)
        },
        "winner": "treatment" if treatment_auc > control_auc else "control"
    }

    return results

# Run simulation
control = mlflow.pyfunc.load_model("models:/fraud-detection/3")
treatment = mlflow.pyfunc.load_model("models:/fraud-detection/4")

results = simulate_ab_test(control, treatment, n_requests=10000, traffic_split=0.1)
print(f"A/B Test Results:")
print(f"  Control (v3):   AUC={results['control']['auc']:.4f}, N={results['control']['n_requests']}")
print(f"  Treatment (v4): AUC={results['treatment']['auc']:.4f}, N={results['treatment']['n_requests']}")
print(f"  Winner: {results['winner']}")
```

### Success Criteria

- [ ] A/B test configuration tracked in model tags
- [ ] Traffic routing works based on hash
- [ ] Metrics collected for both variants
- [ ] Winner can be promoted to full Production

---

## Additional Resources

- [MLflow Model Registry Documentation](https://mlflow.org/docs/latest/model-registry.html)
- [MLflow Python API Reference](https://mlflow.org/docs/latest/python_api/index.html)
- [Model Registry Best Practices](https://www.databricks.com/blog/2020/10/13/managing-ml-model-lifecycle-databricks-model-registry.html)

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025
