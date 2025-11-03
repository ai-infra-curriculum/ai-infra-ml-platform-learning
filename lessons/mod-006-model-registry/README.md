# Module 06: Model Registry & Versioning

Master model registry architecture, versioning strategies, and lifecycle management for production ML systems.

**Duration**: 6-7 hours (lecture + exercises)
**Difficulty**: Intermediate to Advanced
**Prerequisites**: Modules 01-05 completed

---

## Learning Objectives

By the end of this module, you will be able to:

- [ ] Design and implement model registry architecture using MLflow
- [ ] Apply semantic versioning strategies for ML models
- [ ] Implement model lifecycle management (Staging → Production → Archived)
- [ ] Track complete model lineage (data, code, hyperparameters, environment)
- [ ] Build approval workflows for production model deployment
- [ ] Implement A/B testing with multiple model versions
- [ ] Serve models dynamically with automatic version reloading
- [ ] Query and manage model metadata programmatically

---

## Why Model Registry Matters

**The Challenge**: Without a model registry, teams face:
- **Lost context**: "Which training run produced this production model?"
- **Version chaos**: Multiple teams overwriting models without coordination
- **Deployment risks**: No validation gates before production deployment
- **Audit failures**: Cannot reproduce models or trace their lineage
- **Manual processes**: Deployment requires manual file copying and configuration

**The Solution**: A centralized model registry provides:
- **Single source of truth** for all trained models
- **Immutable versioning** with automatic version incrementing
- **Lifecycle stages** (None → Staging → Production → Archived)
- **Complete lineage tracking** from raw data to deployed model
- **Programmatic APIs** for automation and CI/CD integration
- **Approval workflows** with validation gates

**Real-World Impact**:
- **Uber**: Manages 30,000+ models across 200+ ML use cases using Michelangelo Model Registry
- **Netflix**: Tracks model lineage through their entire ML platform for reproducibility
- **Airbnb**: Uses MLflow Model Registry to manage 1,000+ production models with automated rollback

---

## Module Structure

### 📚 Lecture Notes (90 minutes)

#### [01: Model Registry & Versioning](./lecture-notes/01-model-registry-versioning.md)

**Topics Covered**:

1. **Model Registry Fundamentals**
   - Architecture: Backend store (PostgreSQL) + Artifact store (S3/MinIO)
   - Registered model vs. model version
   - Immutability principle

2. **Versioning Strategies**
   - Semantic versioning for ML (MAJOR.MINOR.PATCH-METADATA)
   - Auto-increment versioning in MLflow
   - Version naming conventions

3. **Model Lifecycle Management**
   - Stage-based progression: None → Staging → Production → Archived
   - Stage transition workflows
   - Single-production-version constraint

4. **Model Lineage Tracking**
   - Training data versioning (DVC, dataset hashes)
   - Code versioning (git commits)
   - Hyperparameter tracking
   - Environment tracking (OS, packages, GPU)

5. **Metadata Management**
   - Model signatures (input/output schemas)
   - Tags and descriptions
   - Custom metadata (business metrics, approval status)

6. **MLflow Model Registry**
   - Setup with Docker Compose
   - Registration patterns
   - Programmatic API usage

7. **Approval Workflows**
   - Validation gates (AUC thresholds, required metadata)
   - Multi-stakeholder approval tracking
   - Automated promotion/archival

8. **A/B Testing Patterns**
   - Multi-version serving
   - Hash-based traffic routing
   - Metrics collection per variant

**Key Code Example** (Model Registration):
```python
with mlflow.start_run(run_name="fraud-detection-v2") as run:
    # Track lineage
    mlflow.log_param("data_version", "v2.1")
    mlflow.log_param("git_commit", git.Repo().head.commit.hexsha)
    mlflow.log_params({"n_estimators": 100, "max_depth": 7})

    # Train and log
    model = RandomForestClassifier(n_estimators=100, max_depth=7)
    model.fit(X_train, y_train)

    mlflow.log_metrics({
        "train_auc": roc_auc_score(y_train, model.predict_proba(X_train)[:, 1]),
        "test_auc": roc_auc_score(y_test, model.predict_proba(X_test)[:, 1])
    })

    # Register model with signature
    signature = infer_signature(X_train, model.predict(X_train))
    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="model",
        signature=signature,
        registered_model_name="fraud-detection"
    )
```

**Key Code Example** (Stage Transitions):
```python
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Promote to Production (archives current Production model)
def promote_to_production(name: str, new_version: int):
    current_prod = client.get_latest_versions(name, stages=["Production"])

    if current_prod:
        old_version = current_prod[0].version
        client.transition_model_version_stage(name, old_version, "Archived")
        print(f"Archived v{old_version}")

    client.transition_model_version_stage(name, new_version, "Production")
    print(f"Promoted v{new_version} to Production")
```

---

### 🛠️ Hands-On Exercises (8.5 hours)

Complete 6 comprehensive exercises building a production-ready model registry system:

#### Exercise 01: Set Up MLflow Model Registry (60 min, Basic)
- Deploy MLflow with PostgreSQL backend and MinIO artifact store
- Configure authentication and access control
- Test model registration and retrieval
- Explore UI and programmatic API

**Success Criteria**: MLflow UI accessible, models registered successfully, artifacts stored in MinIO

---

#### Exercise 02: Register and Version Models (75 min, Intermediate)
- Train multiple model versions with different hyperparameters
- Log complete lineage (data version, git commit, environment)
- Implement semantic versioning tags
- Query models by tags and metadata

**Key Learning**: Model immutability, versioning best practices, metadata queries

---

#### Exercise 03: Model Lifecycle Management (90 min, Intermediate)
- Implement stage transition workflows
- Build automated promotion based on metrics
- Handle production model archival
- Create rollback procedures

**Key Pattern**:
```python
# Automated promotion based on AUC threshold
def auto_promote_if_better(name: str, new_version: int):
    new_mv = client.get_model_version(name, new_version)
    new_auc = client.get_run(new_mv.run_id).data.metrics["test_auc"]

    prod_versions = client.get_latest_versions(name, stages=["Production"])
    if prod_versions:
        prod_auc = client.get_run(prod_versions[0].run_id).data.metrics["test_auc"]

        if new_auc > prod_auc:
            promote_to_production(name, new_version)
            return True
    return False
```

---

#### Exercise 04: Model Serving with Registry (120 min, Advanced)
- Build FastAPI service that loads models from registry
- Implement dynamic model reloading (check for updates every 60s)
- Add health checks and model version endpoints
- Load test with concurrent requests

**Key Pattern** (Dynamic Reloading):
```python
class DynamicModelServer:
    def __init__(self, model_name: str, stage: str = "Production", check_interval: int = 60):
        self.model_name = model_name
        self.stage = stage
        self.current_model = None
        self.current_version = None

        self.reload_model()
        Thread(target=self._reload_loop, daemon=True).start()

    def reload_model(self):
        latest = client.get_latest_versions(self.model_name, stages=[self.stage])
        latest_version = latest[0].version

        if latest_version != self.current_version:
            model_uri = f"models:/{self.model_name}/{self.stage}"
            self.current_model = mlflow.pyfunc.load_model(model_uri)
            self.current_version = latest_version
            print(f"Reloaded model to version {latest_version}")
```

---

#### Exercise 05: Build Approval Workflow (90 min, Advanced)
- Implement validation gates (AUC threshold, required metadata)
- Track approvals with tags (approved_by, approved_at)
- Build rejection workflow with comments
- Create approval dashboard

**Key Implementation**:
```python
class ModelApprovalWorkflow:
    def validate_model(self, name: str, version: int) -> dict:
        model_version = client.get_model_version(name, version)
        run = client.get_run(model_version.run_id)

        test_auc = run.data.metrics.get("test_auc", 0.0)
        has_description = bool(model_version.description)

        return {
            "auc_threshold": {"passed": test_auc >= 0.85, "value": test_auc},
            "metadata_complete": {"passed": has_description},
            "overall_passed": test_auc >= 0.85 and has_description
        }

    def approve(self, name: str, version: int, approver: str):
        validation = self.validate_model(name, version)

        if not validation["overall_passed"]:
            raise ValueError(f"Model failed validation: {validation}")

        client.set_model_version_tag(name, version, "approved_by", approver)
        client.set_model_version_tag(name, version, "approved_at",
                                     datetime.now().isoformat())
        client.transition_model_version_stage(name, version, "Production")
```

---

#### Exercise 06: A/B Testing with Model Registry (90 min, Advanced)
- Implement multi-version serving
- Use hash-based routing for consistent user assignment
- Collect metrics per variant (control vs treatment)
- Build winner selection logic

**Key Pattern** (Traffic Split):
```python
class ABTestManager:
    def get_model_for_request(self, model_name: str, request_id: str):
        """Route request to control or treatment based on hash"""
        hash_value = int(hashlib.md5(request_id.encode()).hexdigest(), 16)

        # Get traffic split from treatment model tag
        treatment = self.get_treatment_model(model_name)
        traffic_split = float(client.get_model_version_tag(
            model_name, treatment.version, "ab_traffic"
        ))

        route_to_treatment = (hash_value % 100) < (traffic_split * 100)
        return treatment.version if route_to_treatment else self.get_control_model(model_name).version
```

---

## Tools & Technologies

**Required**:
- MLflow 2.9+ (model registry, tracking server)
- PostgreSQL 14+ (metadata backend)
- MinIO (S3-compatible artifact storage)
- Python 3.9+
- Docker & Docker Compose

**Python Packages**:
```bash
pip install mlflow==2.9.0 \
    psycopg2-binary==2.9.9 \
    boto3==1.28.0 \
    scikit-learn==1.3.0 \
    fastapi==0.104.0 \
    uvicorn==0.24.0
```

**Optional** (for advanced exercises):
- GitPython (git commit tracking)
- Prometheus client (metrics export)
- Grafana (dashboard visualization)

---

## Prerequisites

Before starting this module, ensure you have:

- [x] Completed Module 05 (Workflow Orchestration)
- [x] Understanding of ML training workflows
- [x] Familiarity with REST APIs
- [x] Basic Docker Compose knowledge
- [x] SQL fundamentals (for metadata queries)

**Recommended Background**:
- Experience with version control (git)
- Basic understanding of CI/CD pipelines
- Familiarity with microservices architecture

---

## Time Breakdown

| Component | Duration | Format |
|-----------|----------|--------|
| Lecture notes | 90 min | Reading + code review |
| Exercise 01 | 60 min | Hands-on setup |
| Exercise 02 | 75 min | Hands-on coding |
| Exercise 03 | 90 min | Hands-on coding |
| Exercise 04 | 120 min | Hands-on coding |
| Exercise 05 | 90 min | Hands-on coding |
| Exercise 06 | 90 min | Hands-on coding |
| **Total** | **~10 hours** | Mixed |

**Recommended Schedule**:
- **Day 1**: Lecture notes + Exercise 01-02 (3.5 hours)
- **Day 2**: Exercise 03-04 (3.5 hours)
- **Day 3**: Exercise 05-06 (3 hours)

---

## Success Criteria

You have successfully completed this module when you can:

1. **Architecture** ✅
   - Deploy MLflow registry with PostgreSQL + MinIO backend
   - Explain the difference between backend store and artifact store
   - Configure authentication and access control

2. **Versioning** ✅
   - Register models with complete lineage tracking
   - Apply semantic versioning conventions
   - Query models by tags and metadata

3. **Lifecycle Management** ✅
   - Implement stage transitions (Staging → Production → Archived)
   - Build automated promotion workflows
   - Handle production rollbacks

4. **Serving** ✅
   - Build FastAPI service that loads models from registry
   - Implement dynamic model reloading
   - Load models by stage or version number

5. **Approval Workflows** ✅
   - Implement validation gates (metrics thresholds, metadata checks)
   - Track approvals with tags
   - Build rejection workflows

6. **A/B Testing** ✅
   - Serve multiple model versions simultaneously
   - Implement consistent hash-based routing
   - Collect and compare metrics per variant

---

## Real-World Applications

### Uber: Michelangelo Model Registry
- **Scale**: 30,000+ models across 200+ ML use cases
- **Architecture**: Centralized registry with versioning, lineage tracking, and deployment integration
- **Key Features**: Automated model validation, A/B testing, traffic splitting
- **Impact**: Reduced model deployment time from weeks to hours

### Netflix: Model Lineage Tracking
- **Challenge**: Reproducing models trained months ago for debugging
- **Solution**: Complete lineage tracking (data version, code commit, hyperparameters, environment)
- **Architecture**: MLflow registry integrated with data versioning (DVC) and feature store
- **Impact**: 100% model reproducibility, faster debugging

### Airbnb: Multi-Stage Model Deployment
- **Architecture**: Staging → Canary → Production stages with automated promotion
- **Validation**: Performance metrics, business metrics, A/B test results
- **Scale**: 1,000+ production models with automated lifecycle management
- **Impact**: 50% reduction in production incidents from bad model deployments

---

## Common Pitfalls

### 1. Model Mutability
**Problem**: Overwriting model versions or changing artifacts after registration

**Solution**: Enforce immutability - always create new versions instead of modifying existing ones

### 2. Missing Lineage
**Problem**: Cannot reproduce models because data version or code commit not tracked

**Solution**: Log complete lineage in every training run:
```python
mlflow.log_param("data_version", dataset.version)
mlflow.log_param("git_commit", git.Repo().head.commit.hexsha)
mlflow.log_param("git_branch", git.Repo().active_branch.name)
```

### 3. Production Conflicts
**Problem**: Multiple versions promoted to Production simultaneously

**Solution**: Implement single-production-version constraint in approval workflow

### 4. Large Artifacts
**Problem**: 10GB model files causing slow downloads and serving delays

**Solution**:
- Use model compression (quantization, pruning)
- Cache models locally on serving instances
- Use CDN for artifact distribution

### 5. Metadata Inconsistency
**Problem**: Tags and descriptions not standardized across teams

**Solution**: Enforce metadata schema in approval workflow:
```python
REQUIRED_TAGS = ["data_version", "approved_by", "business_metric"]

def validate_metadata(model_version):
    for tag in REQUIRED_TAGS:
        if not client.get_model_version_tag(model_name, version, tag):
            raise ValueError(f"Missing required tag: {tag}")
```

---

## Assessment

### Knowledge Check (after lecture notes)

1. What are the three core components of MLflow Model Registry architecture?
2. Why is model immutability important?
3. What are the four lifecycle stages in MLflow?
4. What information should be tracked for complete model lineage?
5. How does hash-based routing ensure consistent user assignment in A/B tests?

### Practical Assessment (after exercises)

Build a production-ready model registry system that:
- [ ] Deploys MLflow with PostgreSQL + MinIO backend
- [ ] Registers models with complete lineage tracking
- [ ] Implements automated promotion workflow (promote if AUC > current Production + 0.02)
- [ ] Serves models via FastAPI with dynamic reloading
- [ ] Supports A/B testing with configurable traffic split
- [ ] Collects metrics per model version

**Acceptance Criteria**:
- All components deployed via Docker Compose
- 3+ model versions registered with different hyperparameters
- Approval workflow enforces AUC > 0.85 threshold
- FastAPI service reloads model within 60s of new Production version
- A/B test correctly routes 10% traffic to treatment, 90% to control
- Metrics dashboard shows performance per variant

---

## Additional Resources

### Official Documentation
- [MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html)
- [MLflow Model Signatures](https://mlflow.org/docs/latest/models.html#model-signature-and-input-example)
- [MLflow Tracking API](https://mlflow.org/docs/latest/python_api/mlflow.tracking.html)

### Tutorials & Guides
- [MLflow Model Registry Tutorial](https://mlflow.org/docs/latest/registry.html)
- [Building Production ML Systems with MLflow](https://www.databricks.com/blog/2020/04/16/production-ml-with-mlflow.html)
- [Model Versioning Best Practices](https://neptune.ai/blog/ml-model-versioning)

### Real-World Case Studies
- [Uber's Michelangelo](https://www.uber.com/blog/michelangelo-model-representation/)
- [Airbnb's ML Infrastructure](https://medium.com/airbnb-engineering/ml-platform-at-airbnb-79fcb9e1b037)
- [Netflix's Model Management](https://netflixtechblog.com/notebook-innovation-591ee3221233)

### Video Tutorials
- [MLflow Model Registry Deep Dive](https://www.youtube.com/watch?v=TKHU4RHdmOc) (Databricks)
- [Production ML Systems](https://www.youtube.com/watch?v=9Eh9xE0RB8k) (Google)

---

## Troubleshooting

### MLflow UI Not Accessible

**Symptom**: Cannot access MLflow UI at http://localhost:5000

**Solutions**:
```bash
# Check if MLflow container is running
docker ps | grep mlflow

# Check MLflow logs
docker logs mlflow

# Verify port mapping
docker port mlflow

# Check PostgreSQL connection
docker exec mlflow mlflow db upgrade postgresql://mlflow:mlflow@postgres:5432/mlflow
```

---

### Model Registration Fails

**Symptom**: `mlflow.exceptions.MlflowException: Failed to upload artifact`

**Solutions**:
```bash
# Check MinIO is accessible
curl http://localhost:9000/minio/health/live

# Verify S3 credentials
docker exec mlflow env | grep AWS

# Test artifact upload manually
docker exec mlflow aws --endpoint-url http://minio:9000 s3 ls s3://mlflow
```

---

### Slow Model Loading

**Symptom**: `mlflow.pyfunc.load_model()` takes >30 seconds

**Solutions**:
```python
# Cache models locally
import os
os.environ["MLFLOW_ARTIFACT_CACHE"] = "/tmp/mlflow-cache"

# Pre-download artifacts
model_uri = f"models:/{model_name}/{version}"
mlflow.artifacts.download_artifacts(model_uri, dst_path="/tmp/models")
```

---

### Version Conflicts

**Symptom**: Multiple Production versions exist simultaneously

**Solutions**:
```python
# Audit and fix
prod_versions = client.get_latest_versions(model_name, stages=["Production"])

if len(prod_versions) > 1:
    # Keep the latest version, archive others
    latest = max(prod_versions, key=lambda v: int(v.version))

    for mv in prod_versions:
        if mv.version != latest.version:
            client.transition_model_version_stage(
                model_name, mv.version, "Archived"
            )
```

---

## Next Steps

After completing this module, you're ready for:

- **Module 07: Developer Experience** - Build SDKs and CLIs for ML platform
- **Module 08: Observability & Monitoring** - Monitor models in production
- **Module 09: Security & Governance** - Implement RBAC, audit logs, compliance

**Recommended Path**: Proceed to Module 07 to learn how to wrap your model registry in developer-friendly tools (CLIs, SDKs, Python clients) that make model management effortless for data scientists.

---

## Feedback & Support

**Questions?** Open an issue in the repository with the `module-06` tag.

**Found a bug in the code?** Submit a PR with the fix.

**Want more exercises?** Check the `/exercises/bonus/` directory for advanced challenges.

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025 | **Version**: 1.0.0
