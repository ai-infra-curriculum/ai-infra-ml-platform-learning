# Introduction to ML Platform Engineering

**Role**: ML Platform Engineer
**Level**: Foundations
**Duration**: 8 hours
**Prerequisites**:
- Strong Python programming (3+ years)
- Kubernetes fundamentals
- API design experience
- Understanding of ML workflows (training, inference)

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Define** what an ML platform is and articulate its value proposition to organizations
2. **Differentiate** between ML platform engineering, ML engineering, and infrastructure engineering roles
3. **Identify** the core components of a modern ML platform architecture
4. **Explain** multi-tenancy patterns and their importance in ML platforms
5. **Design** API-first interfaces for ML platform services
6. **Evaluate** platform abstractions and their impact on developer experience
7. **Apply** platform thinking principles to ML infrastructure problems

## Introduction (500 words)

### Why ML Platform Engineering Matters

The field of machine learning has evolved dramatically over the past decade. What began as experimental research projects has transformed into production systems serving millions of users. However, as organizations scale their ML initiatives from a handful of models to hundreds or thousands, they encounter significant operational challenges:

- **Fragmented Tooling**: Data scientists use different tools, frameworks, and workflows, making it difficult to standardize and maintain
- **Duplicate Infrastructure**: Teams rebuild the same infrastructure components (training pipelines, deployment systems, monitoring) repeatedly
- **Slow Time-to-Production**: Moving from notebook to production takes weeks or months due to manual handoffs and complex deployment processes
- **Resource Inefficiency**: Without centralized resource management, compute resources are underutilized or over-provisioned
- **Inconsistent Practices**: Lack of standardization leads to technical debt, security vulnerabilities, and compliance issues

**ML Platform Engineering** addresses these challenges by building self-service infrastructure that enables data scientists and ML engineers to be productive at scale. Rather than each team building their own MLOps tooling, a platform team creates shared services that abstract away infrastructure complexity while maintaining flexibility.

### What You'll Learn

This module establishes the foundational concepts of ML platform engineering. You'll learn what distinguishes platform engineering from traditional infrastructure work, understand the core components of modern ML platforms, and explore architectural patterns for building scalable, multi-tenant systems.

More importantly, you'll develop **platform thinking**—the mindset of treating internal infrastructure as a product with its own users, requirements, and success metrics. This perspective shift is crucial for building platforms that teams actually want to use.

### Real-World Context

Companies like **Uber**, **Netflix**, **Airbnb**, and **LinkedIn** have invested heavily in internal ML platforms:

- **Uber's Michelangelo**: Handles 100,000+ ML models across ride-sharing, food delivery, and freight
- **Netflix's Metaflow**: Used by hundreds of data scientists to build recommendation systems and content optimization models
- **Airbnb's Bighead**: Manages the entire ML lifecycle from experimentation to production deployment
- **LinkedIn's Pro-ML**: Supports thousands of models powering search, recommendations, and feed ranking

These platforms didn't emerge from a single project—they evolved through years of experience, iteration, and learning from production challenges. By studying these patterns, you'll accelerate your journey from months to weeks.

### What Makes Platform Engineering Unique

Platform engineering is distinct from both application development and infrastructure engineering:

| Aspect | Application Engineering | Infrastructure Engineering | Platform Engineering |
|--------|------------------------|---------------------------|---------------------|
| **Primary Users** | External customers | Internal engineers | Data scientists & ML engineers |
| **Success Metric** | User engagement, revenue | Uptime, performance | Developer productivity, adoption |
| **Abstraction Level** | Business logic | Compute, storage, networking | ML workflows, APIs, tools |
| **Iteration Speed** | Fast (days/weeks) | Slow (weeks/months) | Medium (weeks) |
| **Complexity** | Business rules | Distributed systems | Both + ML-specific challenges |

As an ML platform engineer, you're building products for internal customers. Your APIs become the interface through which hundreds of data scientists interact with infrastructure. Your abstractions determine whether teams can deploy models in minutes or spend weeks wrestling with Kubernetes manifests.

This responsibility demands a unique skill set: deep technical expertise in distributed systems and ML, combined with product thinking and empathy for developer experience.

---

## Theoretical Foundations (1,200 words)

### 1. Core Concepts

#### What is an ML Platform?

An **ML platform** is a centralized, self-service infrastructure that enables data scientists and ML engineers to:

1. **Train models** at scale without managing infrastructure
2. **Deploy models** to production with minimal operational overhead
3. **Monitor models** for performance, drift, and reliability
4. **Share artifacts** (features, models, datasets) across teams
5. **Collaborate** on ML projects with versioning and reproducibility

Think of it as **Kubernetes for machine learning**—it provides the orchestration, abstraction, and automation needed to run ML workloads efficiently at scale.

**Key Characteristics:**

- **Self-Service**: Users can provision resources, submit jobs, and deploy models through APIs or UIs without tickets to infrastructure teams
- **Multi-Tenant**: Supports multiple teams and projects with proper isolation, resource quotas, and access controls
- **Observable**: Comprehensive monitoring, logging, and tracing for both infrastructure and ML-specific metrics
- **Declarative**: Users describe what they want (train a model, deploy to staging) rather than how to do it
- **Extensible**: Supports multiple ML frameworks, deployment targets, and custom workflows

#### Platform vs Infrastructure Engineering

While related, these disciplines have different focuses:

**Infrastructure Engineering:**
- Manages physical/virtual resources (servers, networks, storage)
- Focuses on reliability, scalability, and cost optimization
- Abstractions: VMs, containers, load balancers, databases
- Users: Other engineers who understand infrastructure

**Platform Engineering:**
- Builds abstractions on top of infrastructure
- Focuses on developer experience and productivity
- Abstractions: ML workflows, feature stores, model serving
- Users: Data scientists and ML engineers with varying infrastructure knowledge

Example: An infrastructure engineer might manage a Kubernetes cluster, while a platform engineer builds a Kubernetes operator that allows data scientists to submit training jobs with a simple YAML manifest.

#### The Platform-as-a-Product Mindset

Successful platforms treat internal users as customers:

1. **User Research**: Interview data scientists to understand pain points
2. **Requirements Gathering**: What features would 10x their productivity?
3. **API Design**: Create intuitive, well-documented interfaces
4. **Adoption Metrics**: Track which teams use the platform and why
5. **Support & Feedback**: Provide documentation, examples, and responsive help
6. **Iteration**: Continuously improve based on user feedback

This is counterintuitive for many infrastructure engineers who are used to mandate-driven adoption ("you must use our platform"). Platform thinking recognizes that **internal users have choices**—they can build their own tooling, use external services, or work around your platform. Your job is to make the platform so good that they choose to use it.

### 2. Key Principles of ML Platform Design

#### Principle 1: Progressive Disclosure

Design interfaces that are simple for common cases but allow complexity when needed:

**Bad Example:**
```python
# User must understand all parameters upfront
platform.train_model(
    model_type="xgboost",
    n_estimators=100,
    max_depth=6,
    learning_rate=0.1,
    subsample=0.8,
    colsample_bytree=0.8,
    gpu_enabled=True,
    num_gpus=4,
    instance_type="p3.8xlarge",
    min_replicas=1,
    max_replicas=10,
    autoscaling_metric="gpu_utilization",
    autoscaling_target=70,
    docker_image="company/xgboost:latest",
    dockerfile_path=None,
    env_vars={"NCCL_DEBUG": "INFO"},
    volume_mounts=[{"name": "data", "path": "/data"}],
    # ... 30 more parameters
)
```

**Good Example:**
```python
# Simple for 80% of use cases
job = platform.train_model(
    model="xgboost",
    data="s3://my-bucket/training-data.csv"
)

# Advanced users can customize
advanced_job = platform.train_model(
    model="xgboost",
    data="s3://my-bucket/training-data.csv",
    compute=platform.GPUInstance(type="p3.8xlarge", count=4),
    hyperparameters={"learning_rate": 0.05, "max_depth": 8},
    docker=platform.CustomImage("company/xgboost:latest"),
    autoscaling=platform.AutoScaling(min=1, max=10, metric="gpu_util", target=70)
)
```

#### Principle 2: Sensible Defaults

Most users should get good results without configuration:

- **Default instance types** based on workload (CPU for data processing, GPU for training)
- **Automatic parallelization** for distributed training when data size warrants it
- **Standard monitoring** configured out-of-the-box
- **Reasonable resource limits** that prevent accidental overspending

Example: If a user submits a training job with 1TB of data, the platform should automatically select a distributed training configuration rather than trying to cram everything into a single instance.

#### Principle 3: Convention Over Configuration

Establish conventions that reduce boilerplate:

```python
# Platform looks for train.py in project root by convention
# No need to specify: --entrypoint scripts/training/train.py

# Platform uses project structure conventions:
# - data/ for datasets
# - models/ for trained artifacts
# - src/ for source code
# - tests/ for test suite

job = platform.train_model()  # Just works if you follow conventions
```

#### Principle 4: Batteries Included, But Swappable

Provide integrated solutions but allow customization:

- **Default feature store** (e.g., Feast), but users can bring their own
- **Standard model registry** (e.g., MLflow), but support custom registries
- **Built-in monitoring** (Prometheus + Grafana), but allow integration with DataDog, New Relic, etc.

### 3. Core Components of an ML Platform

A production ML platform typically includes these layers:

#### Layer 1: Compute & Resource Management
- **Kubernetes cluster** for orchestration
- **Auto-scaling** for cost efficiency
- **GPU management** for training workloads
- **Resource quotas** per team/project

#### Layer 2: Data Infrastructure
- **Data catalog** for discovery
- **Feature store** for feature sharing and serving
- **Data validation** for quality assurance
- **Versioning** for reproducibility

#### Layer 3: Experimentation & Training
- **Notebook environments** (JupyterHub, VS Code)
- **Distributed training** (Horovod, PyTorch DDP)
- **Hyperparameter optimization** (Optuna, Ray Tune)
- **Experiment tracking** (MLflow, Weights & Biases)

#### Layer 4: Model Management
- **Model registry** for versioning
- **Model validation** for quality gates
- **A/B testing** framework
- **Model governance** (approvals, lineage)

#### Layer 5: Serving & Deployment
- **Online serving** (REST/gRPC APIs)
- **Batch inference** for large-scale predictions
- **Edge deployment** for mobile/IoT
- **Canary deployments** for safe rollouts

#### Layer 6: Monitoring & Observability
- **Infrastructure metrics** (CPU, memory, GPU utilization)
- **ML metrics** (model performance, latency, throughput)
- **Data drift detection**
- **Model drift detection**
- **Alerting** for anomalies

#### Layer 7: Developer Experience
- **CLI tools** for command-line users
- **Python SDK** for programmatic access
- **Web UI** for visual workflows
- **Documentation** (API docs, tutorials, examples)
- **Templates** for common use cases

### 4. Multi-Tenancy in ML Platforms

Multi-tenancy allows multiple teams to share platform infrastructure while maintaining:

1. **Isolation**: One team's work doesn't affect another
2. **Resource Fairness**: Fair allocation of compute, storage, and GPU
3. **Security**: Each tenant's data and models are protected
4. **Cost Tracking**: Accurate attribution of infrastructure costs

**Isolation Strategies:**

| Level | Kubernetes Primitive | Isolation | Overhead | Cost Efficiency |
|-------|---------------------|-----------|----------|----------------|
| **Cluster** | Separate K8s clusters | Maximum | High | Low |
| **Namespace** | K8s namespaces + RBAC | High | Medium | Medium |
| **Workload** | NetworkPolicies + PSP | Medium | Low | High |

Most ML platforms use **namespace-level isolation**:

```yaml
# Team A's resources
apiVersion: v1
kind: Namespace
metadata:
  name: team-a-ml
  labels:
    tenant: team-a
    cost-center: engineering

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a-ml
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    requests.nvidia.com/gpu: "10"
    persistentvolumeclaims: "50"
```

**Best Practices:**

1. **Namespace per team/project** for clean isolation
2. **Resource quotas** to prevent single-tenant resource hogging
3. **Network policies** to restrict cross-namespace communication
4. **RBAC** for fine-grained access control
5. **Pod security policies** to enforce security baselines

---

## Practical Implementation (1,800 words)

### Building Your First ML Platform API

Let's build a simplified ML platform API that demonstrates core concepts. We'll create a training job submission service with multi-tenancy support.

#### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway                          │
│              (Authentication, Rate Limiting)            │
└──────────────────┬──────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────────┐
│               Platform API Service                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │  Jobs    │  │ Models   │  │Resources │             │
│  │Controller│  │Controller│  │Controller│             │
│  └─────┬────┘  └────┬─────┘  └────┬─────┘             │
└────────┼────────────┼─────────────┼────────────────────┘
         │            │             │
┌────────▼────────────▼─────────────▼────────────────────┐
│              Kubernetes API Server                      │
│    (Creates Pods, Jobs, Services, ConfigMaps)          │
└─────────────────────────────────────────────────────────┘
```

#### Step 1: Define Platform Resources

First, define the core data models for your platform:

```python
# platform/models.py
from typing import Optional, Dict, List
from pydantic import BaseModel, Field
from enum import Enum

class JobStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    CANCELLED = "cancelled"

class ComputeType(str, Enum):
    CPU = "cpu"
    GPU = "gpu"
    TPU = "tpu"

class InstanceConfig(BaseModel):
    """Compute instance configuration"""
    type: ComputeType = ComputeType.CPU
    cpu_cores: int = Field(default=4, ge=1, le=96)
    memory_gb: int = Field(default=16, ge=1, le=768)
    gpu_count: Optional[int] = Field(default=None, ge=1, le=8)
    gpu_type: Optional[str] = None  # "nvidia-tesla-v100", "nvidia-a100"

    class Config:
        json_schema_extra = {
            "example": {
                "type": "gpu",
                "cpu_cores": 8,
                "memory_gb": 64,
                "gpu_count": 2,
                "gpu_type": "nvidia-a100"
            }
        }

class TrainingJobSpec(BaseModel):
    """Specification for a training job"""
    name: str = Field(..., description="Job name (must be unique within tenant)")
    tenant_id: str = Field(..., description="Tenant identifier (team/project)")

    # Code and data
    docker_image: str = Field(..., description="Docker image containing training code")
    entrypoint: str = Field(default="python train.py", description="Command to execute")
    code_repo: Optional[str] = None  # Git repository URL
    code_commit: Optional[str] = None  # Git commit SHA
    data_path: str = Field(..., description="S3/GCS path to training data")

    # Compute resources
    compute: InstanceConfig = Field(default_factory=InstanceConfig)

    # Hyperparameters
    hyperparameters: Dict[str, any] = Field(default_factory=dict)

    # Environment
    env_vars: Dict[str, str] = Field(default_factory=dict)

    # Outputs
    model_output_path: str = Field(..., description="S3/GCS path for model artifacts")

    class Config:
        json_schema_extra = {
            "example": {
                "name": "churn-prediction-v1",
                "tenant_id": "team-ml-analytics",
                "docker_image": "company/ml-training:pytorch-1.13",
                "data_path": "s3://ml-data/churn/train.csv",
                "compute": {
                    "type": "gpu",
                    "cpu_cores": 8,
                    "memory_gb": 32,
                    "gpu_count": 1,
                    "gpu_type": "nvidia-tesla-v100"
                },
                "hyperparameters": {
                    "learning_rate": 0.001,
                    "batch_size": 64,
                    "epochs": 100
                },
                "model_output_path": "s3://ml-models/churn/v1/"
            }
        }

class TrainingJob(BaseModel):
    """Complete training job with metadata"""
    id: str
    spec: TrainingJobSpec
    status: JobStatus
    created_at: str
    started_at: Optional[str] = None
    completed_at: Optional[str] = None
    error_message: Optional[str] = None
    kubernetes_job_name: str
    metrics: Dict[str, float] = Field(default_factory=dict)
```

#### Step 2: Implement Multi-Tenant Resource Controller

```python
# platform/resource_controller.py
from typing import Dict, Optional
from kubernetes import client, config
import hashlib

class ResourceController:
    """Manages Kubernetes resources with multi-tenancy support"""

    def __init__(self):
        config.load_kube_config()
        self.core_v1 = client.CoreV1Api()
        self.batch_v1 = client.BatchV1Api()
        self.rbac_v1 = client.RbacAuthorizationV1Api()

    def ensure_tenant_namespace(self, tenant_id: str) -> str:
        """Ensure namespace exists for tenant with proper quotas"""
        namespace_name = f"ml-{tenant_id}"

        # Create namespace if it doesn't exist
        try:
            self.core_v1.read_namespace(namespace_name)
        except client.rest.ApiException as e:
            if e.status == 404:
                namespace = client.V1Namespace(
                    metadata=client.V1ObjectMeta(
                        name=namespace_name,
                        labels={
                            "tenant": tenant_id,
                            "managed-by": "ml-platform"
                        }
                    )
                )
                self.core_v1.create_namespace(namespace)

                # Create resource quota
                self._create_resource_quota(namespace_name, tenant_id)

                # Create default service account with RBAC
                self._create_service_account(namespace_name, tenant_id)

        return namespace_name

    def _create_resource_quota(self, namespace: str, tenant_id: str):
        """Create resource quota for tenant"""
        # Default quotas - would be tenant-specific in production
        quota = client.V1ResourceQuota(
            metadata=client.V1ObjectMeta(name=f"{tenant_id}-quota"),
            spec=client.V1ResourceQuotaSpec(
                hard={
                    "requests.cpu": "100",
                    "requests.memory": "200Gi",
                    "requests.nvidia.com/gpu": "10",
                    "persistentvolumeclaims": "50",
                    "pods": "100"
                }
            )
        )
        self.core_v1.create_namespaced_resource_quota(namespace, quota)

    def _create_service_account(self, namespace: str, tenant_id: str):
        """Create service account with limited permissions"""
        # Service account
        sa = client.V1ServiceAccount(
            metadata=client.V1ObjectMeta(name="ml-job-runner")
        )
        self.core_v1.create_namespaced_service_account(namespace, sa)

        # Role - can read secrets, configmaps; can create/get/list pods
        role = client.V1Role(
            metadata=client.V1ObjectMeta(name="ml-job-role"),
            rules=[
                client.V1PolicyRule(
                    api_groups=[""],
                    resources=["secrets", "configmaps"],
                    verbs=["get", "list"]
                ),
                client.V1PolicyRule(
                    api_groups=[""],
                    resources=["pods", "pods/log"],
                    verbs=["get", "list", "create"]
                )
            ]
        )
        self.rbac_v1.create_namespaced_role(namespace, role)

        # RoleBinding
        role_binding = client.V1RoleBinding(
            metadata=client.V1ObjectMeta(name="ml-job-binding"),
            subjects=[client.V1Subject(
                kind="ServiceAccount",
                name="ml-job-runner",
                namespace=namespace
            )],
            role_ref=client.V1RoleRef(
                api_group="rbac.authorization.k8s.io",
                kind="Role",
                name="ml-job-role"
            )
        )
        self.rbac_v1.create_namespaced_role_binding(namespace, role_binding)

    def create_training_job(self, job_spec: TrainingJobSpec) -> str:
        """Create Kubernetes Job for training"""
        namespace = self.ensure_tenant_namespace(job_spec.tenant_id)

        # Generate unique job name
        job_id = self._generate_job_id(job_spec)
        k8s_job_name = f"{job_spec.name}-{job_id[:8]}"

        # Build container spec
        container = self._build_container(job_spec)

        # Build pod template
        pod_template = client.V1PodTemplateSpec(
            metadata=client.V1ObjectMeta(
                labels={
                    "app": "ml-training",
                    "job-name": k8s_job_name,
                    "tenant": job_spec.tenant_id
                }
            ),
            spec=client.V1PodSpec(
                restart_policy="Never",
                service_account_name="ml-job-runner",
                containers=[container]
            )
        )

        # Create Job
        job = client.V1Job(
            metadata=client.V1ObjectMeta(name=k8s_job_name),
            spec=client.V1JobSpec(
                template=pod_template,
                backoff_limit=3,  # Retry failed jobs 3 times
                ttl_seconds_after_finished=86400  # Clean up after 24h
            )
        )

        self.batch_v1.create_namespaced_job(namespace, job)

        return k8s_job_name

    def _build_container(self, job_spec: TrainingJobSpec) -> client.V1Container:
        """Build Kubernetes container spec from job spec"""
        # Environment variables
        env_vars = [
            client.V1EnvVar(name="DATA_PATH", value=job_spec.data_path),
            client.V1EnvVar(name="MODEL_OUTPUT_PATH", value=job_spec.model_output_path),
        ]

        # Add hyperparameters as env vars
        for key, value in job_spec.hyperparameters.items():
            env_vars.append(
                client.V1EnvVar(name=f"HYPERPARAM_{key.upper()}", value=str(value))
            )

        # Add custom env vars
        for key, value in job_spec.env_vars.items():
            env_vars.append(client.V1EnvVar(name=key, value=value))

        # Resource requests and limits
        resources = client.V1ResourceRequirements(
            requests={
                "cpu": str(job_spec.compute.cpu_cores),
                "memory": f"{job_spec.compute.memory_gb}Gi"
            },
            limits={
                "cpu": str(job_spec.compute.cpu_cores),
                "memory": f"{job_spec.compute.memory_gb}Gi"
            }
        )

        # Add GPU resources if requested
        if job_spec.compute.type == ComputeType.GPU and job_spec.compute.gpu_count:
            resources.requests["nvidia.com/gpu"] = str(job_spec.compute.gpu_count)
            resources.limits["nvidia.com/gpu"] = str(job_spec.compute.gpu_count)

        container = client.V1Container(
            name="training",
            image=job_spec.docker_image,
            command=["/bin/sh", "-c"],
            args=[job_spec.entrypoint],
            env=env_vars,
            resources=resources
        )

        return container

    def _generate_job_id(self, job_spec: TrainingJobSpec) -> str:
        """Generate deterministic job ID"""
        content = f"{job_spec.tenant_id}{job_spec.name}{job_spec.docker_image}"
        return hashlib.sha256(content.encode()).hexdigest()

    def get_job_status(self, tenant_id: str, job_name: str) -> Dict:
        """Get status of a running job"""
        namespace = f"ml-{tenant_id}"

        try:
            job = self.batch_v1.read_namespaced_job(job_name, namespace)

            status = JobStatus.PENDING
            if job.status.active and job.status.active > 0:
                status = JobStatus.RUNNING
            elif job.status.succeeded and job.status.succeeded > 0:
                status = JobStatus.SUCCEEDED
            elif job.status.failed and job.status.failed > 0:
                status = JobStatus.FAILED

            return {
                "status": status,
                "start_time": job.status.start_time,
                "completion_time": job.status.completion_time,
                "active": job.status.active or 0,
                "succeeded": job.status.succeeded or 0,
                "failed": job.status.failed or 0
            }
        except client.rest.ApiException as e:
            if e.status == 404:
                return {"status": JobStatus.PENDING, "error": "Job not found"}
            raise
```

#### Step 3: Build FastAPI Service

```python
# platform/api.py
from fastapi import FastAPI, HTTPException, Depends, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from typing import List, Optional
import uuid
from datetime import datetime

from .models import TrainingJobSpec, TrainingJob, JobStatus
from .resource_controller import ResourceController

app = FastAPI(
    title="ML Platform API",
    description="Self-service API for ML training and deployment",
    version="1.0.0"
)

security = HTTPBearer()
resource_controller = ResourceController()

# In-memory job storage (use database in production)
jobs_db: Dict[str, TrainingJob] = {}

def get_tenant_from_token(credentials: HTTPAuthorizationCredentials = Depends(security)) -> str:
    """Extract tenant ID from JWT token"""
    # In production, validate JWT and extract tenant_id
    # For demo, we'll use a simple token format: "Bearer tenant-{id}"
    token = credentials.credentials
    if token.startswith("tenant-"):
        return token[7:]  # Remove "tenant-" prefix
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Invalid authentication token"
    )

@app.post("/v1/jobs/training", response_model=TrainingJob, status_code=status.HTTP_201_CREATED)
async def create_training_job(
    job_spec: TrainingJobSpec,
    tenant_id: str = Depends(get_tenant_from_token)
):
    """Submit a new training job"""
    # Validate tenant matches token
    if job_spec.tenant_id != tenant_id:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Cannot create jobs for other tenants"
        )

    # Create Kubernetes job
    try:
        k8s_job_name = resource_controller.create_training_job(job_spec)
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"Failed to create training job: {str(e)}"
        )

    # Create job record
    job_id = str(uuid.uuid4())
    job = TrainingJob(
        id=job_id,
        spec=job_spec,
        status=JobStatus.PENDING,
        created_at=datetime.utcnow().isoformat(),
        kubernetes_job_name=k8s_job_name
    )

    jobs_db[job_id] = job

    return job

@app.get("/v1/jobs/training/{job_id}", response_model=TrainingJob)
async def get_training_job(
    job_id: str,
    tenant_id: str = Depends(get_tenant_from_token)
):
    """Get training job details"""
    if job_id not in jobs_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Job {job_id} not found"
        )

    job = jobs_db[job_id]

    # Verify tenant ownership
    if job.spec.tenant_id != tenant_id:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Cannot access jobs from other tenants"
        )

    # Update job status from Kubernetes
    k8s_status = resource_controller.get_job_status(
        job.spec.tenant_id,
        job.kubernetes_job_name
    )
    job.status = k8s_status["status"]
    if k8s_status.get("start_time"):
        job.started_at = k8s_status["start_time"].isoformat()
    if k8s_status.get("completion_time"):
        job.completed_at = k8s_status["completion_time"].isoformat()

    return job

@app.get("/v1/jobs/training", response_model=List[TrainingJob])
async def list_training_jobs(
    tenant_id: str = Depends(get_tenant_from_token),
    status: Optional[JobStatus] = None,
    limit: int = 50
):
    """List training jobs for tenant"""
    tenant_jobs = [
        job for job in jobs_db.values()
        if job.spec.tenant_id == tenant_id
    ]

    # Filter by status if specified
    if status:
        tenant_jobs = [job for job in tenant_jobs if job.status == status]

    # Sort by creation time (newest first)
    tenant_jobs.sort(key=lambda j: j.created_at, reverse=True)

    return tenant_jobs[:limit]

@app.delete("/v1/jobs/training/{job_id}", status_code=status.HTTP_204_NO_CONTENT)
async def cancel_training_job(
    job_id: str,
    tenant_id: str = Depends(get_tenant_from_token)
):
    """Cancel a running training job"""
    if job_id not in jobs_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Job {job_id} not found"
        )

    job = jobs_db[job_id]

    # Verify tenant ownership
    if job.spec.tenant_id != tenant_id:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Cannot cancel jobs from other tenants"
        )

    # Delete Kubernetes job
    namespace = f"ml-{tenant_id}"
    try:
        resource_controller.batch_v1.delete_namespaced_job(
            job.kubernetes_job_name,
            namespace,
            propagation_policy="Background"
        )
        job.status = JobStatus.CANCELLED
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"Failed to cancel job: {str(e)}"
        )

@app.get("/v1/health")
async def health_check():
    """Health check endpoint"""
    return {"status": "healthy", "service": "ml-platform-api"}
```

#### Step 4: Testing the Platform

```python
# test_platform.py
import requests
import json

API_BASE = "http://localhost:8000/v1"
TENANT_TOKEN = "tenant-team-ml-analytics"

headers = {
    "Authorization": f"Bearer {TENANT_TOKEN}",
    "Content-Type": "application/json"
}

# Create training job
job_spec = {
    "name": "churn-prediction-v1",
    "tenant_id": "team-ml-analytics",
    "docker_image": "pytorch/pytorch:1.13.0-cuda11.6-cudnn8-runtime",
    "entrypoint": "python train.py --epochs 100",
    "data_path": "s3://ml-data/churn/train.csv",
    "compute": {
        "type": "gpu",
        "cpu_cores": 8,
        "memory_gb": 32,
        "gpu_count": 1,
        "gpu_type": "nvidia-tesla-v100"
    },
    "hyperparameters": {
        "learning_rate": 0.001,
        "batch_size": 64,
        "epochs": 100
    },
    "model_output_path": "s3://ml-models/churn/v1/"
}

response = requests.post(f"{API_BASE}/jobs/training", headers=headers, json=job_spec)
job = response.json()
print(f"Created job: {job['id']}")
print(f"Kubernetes job: {job['kubernetes_job_name']}")

# Get job status
response = requests.get(f"{API_BASE}/jobs/training/{job['id']}", headers=headers)
job_status = response.json()
print(f"Job status: {job_status['status']}")

# List all jobs
response = requests.get(f"{API_BASE}/jobs/training", headers=headers)
jobs = response.json()
print(f"Total jobs: {len(jobs)}")
```

---

## Advanced Topics (800 words)

### 1. Handling GPU Resources Efficiently

GPUs are expensive and often the bottleneck in ML platforms. Effective GPU management requires:

**GPU Sharing Strategies:**

```python
# Option 1: Time-slicing (multiple jobs share GPU sequentially)
# Good for: Small models, interactive workloads
# Downside: Reduced throughput per job

# Option 2: Multi-Instance GPU (MIG)
# Good for: Isolation without time-sharing overhead
# Supported on: NVIDIA A100, A30

# Option 3: Dedicated GPU allocation
# Good for: Large training jobs
# Downside: Low utilization if jobs are small
```

**Dynamic GPU Allocation:**

```python
class SmartGPUScheduler:
    """Allocate GPUs based on workload characteristics"""

    def recommend_gpu_config(self, dataset_size_gb: float, model_params: int) -> Dict:
        """Recommend GPU configuration"""
        # Small model + small data: MIG or shared
        if dataset_size_gb < 10 and model_params < 1e8:
            return {"type": "shared", "fraction": 0.25}

        # Medium model: Full GPU
        elif dataset_size_gb < 100 and model_params < 1e9:
            return {"type": "dedicated", "count": 1}

        # Large model: Multi-GPU
        else:
            num_gpus = min(8, max(2, model_params // 1e9))
            return {"type": "dedicated", "count": num_gpus, "distributed": True}
```

### 2. Cost Optimization Patterns

```python
# Pattern 1: Spot instances for fault-tolerant training
@dataclass
class SpotInstanceConfig:
    use_spot: bool = True
    max_price: float = 0.5  # $ per hour
    checkpoint_frequency: int = 10  # Save every N minutes
    auto_resume: bool = True

# Pattern 2: Automatic scaling down during idle
@dataclass
class IdleScalingConfig:
    idle_threshold_minutes: int = 30
    scale_to_zero: bool = True
    warmup_pool_size: int = 2  # Keep N instances warm

# Pattern 3: Resource right-sizing
def analyze_job_metrics(job_id: str) -> Dict:
    """Analyze actual resource usage vs requested"""
    actual_cpu_usage = get_metric("cpu_utilization", job_id)
    actual_memory_usage = get_metric("memory_utilization", job_id)
    actual_gpu_usage = get_metric("gpu_utilization", job_id)

    recommendations = {}
    if actual_cpu_usage < 0.3:
        recommendations["cpu"] = "Over-provisioned by 70%"
    if actual_gpu_usage < 0.5:
        recommendations["gpu"] = "Consider GPU sharing or smaller GPU"

    return recommendations
```

### 3. Handling Job Failures and Retries

```python
class JobRetryPolicy:
    """Intelligent retry logic for training jobs"""

    def should_retry(self, job: TrainingJob, failure_reason: str) -> bool:
        """Determine if job should be retried"""
        # Don't retry user code errors
        if "ValueError" in failure_reason or "TypeError" in failure_reason:
            return False

        # Retry infrastructure failures
        if "OutOfMemory" in failure_reason:
            # Retry with more memory
            job.spec.compute.memory_gb *= 1.5
            return True

        if "ConnectionTimeout" in failure_reason:
            # Transient network issue
            return True

        if "SpotInstanceTerminated" in failure_reason:
            # Spot instance preempted
            return True

        return False

    def exponential_backoff(self, attempt: int) -> int:
        """Calculate wait time before retry"""
        return min(300, 2 ** attempt)  # Max 5 minutes
```

### 4. Production Deployment Considerations

**High Availability:**

```yaml
# Deploy API with multiple replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-platform-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ml-platform-api
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - ml-platform-api
            topologyKey: kubernetes.io/hostname
```

**API Versioning:**

```python
# Support multiple API versions simultaneously
@app.post("/v1/jobs/training")  # Legacy
async def create_job_v1(spec: TrainingJobSpecV1): ...

@app.post("/v2/jobs/training")  # New features
async def create_job_v2(spec: TrainingJobSpecV2): ...
```

**Observability:**

```python
from prometheus_client import Counter, Histogram, Gauge

# Metrics
jobs_created_total = Counter("ml_platform_jobs_created", "Total jobs created", ["tenant"])
job_duration_seconds = Histogram("ml_platform_job_duration_seconds", "Job duration")
active_jobs = Gauge("ml_platform_active_jobs", "Currently running jobs", ["tenant"])

@app.post("/v1/jobs/training")
async def create_training_job(spec: TrainingJobSpec, tenant_id: str = Depends(get_tenant_from_token)):
    jobs_created_total.labels(tenant=tenant_id).inc()
    active_jobs.labels(tenant=tenant_id).inc()
    # ... create job ...
```

---

## Hands-On Practice (600 words)

### Exercise 1: Deploy the Platform API (Basic)

**Objective**: Deploy the ML platform API to a local Kubernetes cluster and submit your first training job.

**Steps**:

1. **Setup local Kubernetes**:
   ```bash
   # Using minikube
   minikube start --cpus 4 --memory 8192
   kubectl create namespace ml-platform
   ```

2. **Build and deploy API**:
   ```bash
   docker build -t ml-platform-api:v1 .
   kubectl apply -f k8s/deployment.yaml
   kubectl port-forward svc/ml-platform-api 8000:8000
   ```

3. **Submit test job**:
   ```bash
   curl -X POST http://localhost:8000/v1/jobs/training \
     -H "Authorization: Bearer tenant-test" \
     -H "Content-Type: application/json" \
     -d @examples/simple-job.json
   ```

**Expected Outcome**: Job created successfully, visible in `kubectl get jobs -n ml-test`

**Time**: 45 minutes

### Exercise 2: Implement Resource Quotas (Intermediate)

**Objective**: Add resource quota enforcement to prevent tenants from exceeding limits.

**Steps**:

1. Modify `ResourceController` to check current resource usage
2. Add validation in `create_training_job` API endpoint
3. Return HTTP 429 (Too Many Requests) when quota exceeded
4. Test by creating multiple jobs until quota is hit

**Hints**:
- Use `kubectl get resourcequota -n <namespace> -o json`
- Parse `status.used` and compare against `spec.hard`
- Consider both current usage + new job request

**Expected Outcome**: API rejects jobs that would exceed quota with clear error message

**Time**: 60 minutes

### Exercise 3: Add Job Monitoring Dashboard (Intermediate)

**Objective**: Create a Grafana dashboard showing job metrics per tenant.

**Steps**:

1. Instrument API with Prometheus metrics (job count, duration, resource usage)
2. Deploy Prometheus to scrape API metrics
3. Create Grafana dashboard with panels:
   - Jobs created per tenant (counter)
   - Average job duration (histogram)
   - Active jobs by status (gauge)
   - Resource utilization per tenant

**Time**: 75 minutes

### Exercise 4: Implement Distributed Training (Advanced)

**Objective**: Extend the platform to support multi-node distributed training with PyTorch DDP.

**Steps**:

1. Create `DistributedTrainingJobSpec` with `num_workers` field
2. Modify `ResourceController` to create multiple pods (1 master + N workers)
3. Configure PyTorch distributed environment variables
4. Test with example distributed training script

**Hints**:
- Use Kubernetes `Job` with `parallelism` and `completions`
- Set `MASTER_ADDR`, `MASTER_PORT`, `WORLD_SIZE`, `RANK` env vars
- Consider using PyTorch Elastic (torchrun) for fault tolerance

**Expected Outcome**: Successfully train model across multiple pods with linear speedup

**Time**: 120 minutes

### Exercise 5: Build Python SDK (Advanced)

**Objective**: Create a Python SDK that wraps the REST API for better developer experience.

**Steps**:

1. Design SDK interface:
   ```python
   from ml_platform import Client

   client = Client(api_url="http://localhost:8000", token="tenant-test")

   # Simple interface
   job = client.train(
       name="my-model",
       image="pytorch/pytorch:latest",
       data="s3://data/train.csv",
       compute="gpu-1x-v100"
   )

   # Wait for completion
   job.wait()
   print(f"Model saved to: {job.output_path}")
   ```

2. Implement SDK classes: `Client`, `TrainingJob`, `DeployedModel`
3. Add retries, timeouts, and error handling
4. Write unit tests with mocked API responses

**Expected Outcome**: SDK provides Pythonic interface that's easier than raw API calls

**Time**: 180 minutes

---

## Real-World Applications (500 words)

### Case Study 1: Uber's Michelangelo

**Scale**: 100,000+ models, 1,000+ ML engineers

**Architecture**:
- **Feature Store**: Centralized feature definitions with online/offline stores
- **Training Platform**: Supports Spark MLlib, XGBoost, TensorFlow, PyTorch
- **Model Serving**: Containerized serving with auto-scaling
- **Monitoring**: Real-time feature and prediction drift detection

**Key Learnings**:
1. **Standardization reduces toil**: Common interfaces for training/serving reduced deployment time from weeks to hours
2. **Feature reuse drives impact**: Teams reused 70% of features, preventing duplicate work
3. **Platform adoption requires evangelism**: Dedicated team worked with ML engineers to onboard use cases

**Metrics**:
- Time to production: 95% reduction (weeks → hours)
- Feature reuse: 70% of features used by multiple teams
- Cost savings: 40% reduction in compute costs through consolidation

### Case Study 2: Netflix's Metaflow

**Scale**: 400+ data scientists, thousands of workflows

**Philosophy**: "Make the right thing the easy thing"

**Design Principles**:
1. **Python-first**: No YAML, no config files—everything in Python
2. **Reproducibility**: Automatic versioning of code, data, and dependencies
3. **Scalability**: Seamlessly scale from laptop to AWS Batch/Kubernetes
4. **Debuggability**: Rich debugging tools and checkpointing

**Example Workflow**:
```python
from metaflow import FlowSpec, step, Parameter, resources

class ChurnPredictionFlow(FlowSpec):
    learning_rate = Parameter('lr', default=0.001)

    @step
    def start(self):
        self.data_path = 's3://data/churn.csv'
        self.next(self.preprocess)

    @step
    def preprocess(self):
        import pandas as pd
        self.df = pd.read_csv(self.data_path)
        self.next(self.train)

    @resources(cpu=8, memory=32000, gpu=1)
    @step
    def train(self):
        # Training code runs on AWS Batch automatically
        from sklearn.ensemble import RandomForestClassifier
        model = RandomForestClassifier()
        model.fit(self.X_train, self.y_train)
        self.model = model
        self.next(self.end)

    @step
    def end(self):
        print(f"Model accuracy: {self.accuracy}")

if __name__ == '__main__':
    ChurnPredictionFlow()
```

**Impact**:
- Reduced experimentation friction by 80%
- Increased model deployment rate by 3x
- Open-sourced in 2019, adopted by hundreds of companies

### Case Study 3: Airbnb's Bighead

**Scale**: 1,000+ models, millions of predictions/second

**Unique Challenges**:
- **Online and offline**: Support both batch and real-time inference
- **Feature freshness**: Features computed from recent booking data must be served in <10ms
- **Explainability**: Regulatory requirements for understanding model decisions

**Solutions**:
1. **Unified feature store**: Precompute features offline, serve online from Redis
2. **Standardized serving**: All models wrapped in common interface
3. **Shadow deployments**: Run new models alongside old ones to validate performance
4. **Built-in explainability**: SHAP values computed and cached for common predictions

**Adoption Strategy**:
- Started with high-impact use cases (search ranking, pricing)
- Demonstrated 5x deployment speedup to gain buy-in
- Gradually migrated teams through incentives and support

---

## Summary and Key Takeaways (300 words)

### Core Concepts Mastered

1. **Platform Engineering Mindset**: Treating internal infrastructure as a product with customers, requirements, and success metrics

2. **Multi-Tenancy**: Isolating teams and projects using Kubernetes namespaces, RBAC, and resource quotas

3. **API-First Design**: Building intuitive, well-documented APIs that abstract infrastructure complexity

4. **Progressive Disclosure**: Simple interfaces for common cases, advanced options for power users

5. **Resource Management**: Efficient allocation of expensive resources (GPUs) across teams

### Skills Acquired

- ✅ Design and implement multi-tenant ML platform APIs
- ✅ Manage Kubernetes resources programmatically
- ✅ Create abstractions that balance simplicity and flexibility
- ✅ Implement authentication and authorization for platform services
- ✅ Monitor platform health and usage metrics

### Platform Engineering Principles

1. **Empathy First**: Understand your users' workflows and pain points
2. **Start Simple**: Build MVPs and iterate based on feedback
3. **Make it Fast**: Reduce time-to-value for users
4. **Make it Safe**: Provide guardrails and validate inputs
5. **Make it Observable**: Users should know what's happening and why

### Next Steps

In **Module 02: API Design for ML Platforms**, you'll dive deeper into:
- RESTful vs gRPC trade-offs for ML workloads
- API versioning strategies
- SDK design patterns
- OpenAPI specification and documentation
- Client code generation

Continue building on the platform you started here by adding:
- Model deployment endpoints
- Feature store integration
- Experiment tracking APIs
- Resource monitoring dashboards

### Recommended Practice

1. **Deploy your platform**: Get hands-on experience with the exercises
2. **Read source code**: Study Kubeflow, MLflow, and Metaflow codebases
3. **Interview users**: Talk to data scientists about their workflows
4. **Measure everything**: Instrument your platform from day one
5. **Iterate quickly**: Ship early, gather feedback, improve

---

## Assessment Questions

### Knowledge Check (10 questions)

1. **What is the primary difference between ML platform engineering and infrastructure engineering?**
   - A) Platform engineers work with ML models
   - B) Platform engineers focus on developer experience and abstractions
   - C) Platform engineers only work with Kubernetes
   - D) There is no difference

   **Answer**: B

2. **In a multi-tenant ML platform, what is the recommended Kubernetes isolation strategy?**
   - A) Separate clusters per tenant
   - B) Namespace-level isolation with RBAC and quotas
   - C) Workload-level isolation only
   - D) No isolation needed

   **Answer**: B

3. **What does "progressive disclosure" mean in API design?**
   - A) Releasing features progressively over time
   - B) Simple interfaces for common cases, advanced options when needed
   - C) Gradually documenting APIs
   - D) Slowly increasing API rate limits

   **Answer**: B

4. **Why is "platform-as-a-product" thinking important?**
   - A) Because internal users have choices and you must earn adoption
   - B) To justify platform team headcount
   - C) To create monetization opportunities
   - D) It's not important for internal platforms

   **Answer**: A

5. **What are the key components of a production ML platform? (Select all that apply)**
   - A) Compute & resource management
   - B) Feature store
   - C) Model serving
   - D) Custom UI framework
   - E) Monitoring & observability

   **Answers**: A, B, C, E

6. **How should GPU resources be allocated efficiently?**
   - A) Always use dedicated GPUs
   - B) Mix of time-slicing, MIG, and dedicated based on workload
   - C) Never share GPUs
   - D) Random allocation

   **Answer**: B

7. **What authentication pattern is recommended for platform APIs?**
   - A) No authentication needed for internal platforms
   - B) Basic auth with username/password
   - C) JWT tokens with tenant identification
   - D) API keys only

   **Answer**: C

8. **What is the purpose of resource quotas in multi-tenant platforms?**
   - A) To charge tenants for usage
   - B) To prevent single tenants from monopolizing resources
   - C) To reduce infrastructure costs
   - D) Quotas are not necessary

   **Answer**: B

9. **Which companies have open-sourced their ML platform technologies?**
   - A) Netflix (Metaflow)
   - B) Uber (Michelangelo)
   - C) Google (Kubeflow)
   - D) A and C only

   **Answer**: D

10. **What is the recommended approach for handling training job failures?**
    - A) Never retry failed jobs
    - B) Always retry all failures
    - C) Intelligent retries based on failure reason (infrastructure vs code errors)
    - D) Retry only once

    **Answer**: C

### Practical Challenge

**Task**: Extend the ML platform API to support model deployment

**Requirements**:
1. Add a `/v1/deployments` endpoint for deploying trained models
2. Spec should include: model path, instance type, scaling config (min/max replicas)
3. Create Kubernetes Deployment + Service for serving
4. Support rolling updates (deploy new version without downtime)
5. Add health check endpoint for deployed models

**Deliverables**:
- Updated `models.py` with `DeploymentSpec` and `Deployment` classes
- Extended `ResourceController` with deployment management
- New FastAPI endpoints in `api.py`
- Example usage script

**Time**: 3 hours

**Evaluation Criteria**:
- Correctly creates Kubernetes Deployment and Service
- Handles multi-tenancy (namespace isolation)
- Implements rolling updates
- Returns deployment status and endpoint URL
- Includes error handling and validation

---

## Resources

### Official Documentation
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [Prometheus Python Client](https://github.com/prometheus/client_python)

### ML Platform Projects
- [Kubeflow](https://www.kubeflow.org/) - ML toolkit for Kubernetes
- [MLflow](https://mlflow.org/) - ML lifecycle management
- [Metaflow](https://metaflow.org/) - Netflix's ML framework
- [Feast](https://feast.dev/) - Feature store
- [Seldon Core](https://www.seldon.io/) - ML deployment

### Books
- *Building Machine Learning Powered Applications* by Emmanuel Ameisen
- *Designing Data-Intensive Applications* by Martin Kleppmann
- *Building Multi-Tenant SaaS Architectures* by Tod Golding

### Articles & Papers
- [Building ML Platforms at Uber](https://www.uber.com/blog/michelangelo-machine-learning-platform/)
- [Open-Sourcing Metaflow](https://netflixtechblog.com/open-sourcing-metaflow-a-human-centric-framework-for-data-science-fa72e04a5d9)
- [Bighead: Airbnb's ML Platform](https://medium.com/airbnb-engineering/bighead-airbnbs-end-to-end-machine-learning-platform-f3a2dab1d3c9)
- [Hidden Technical Debt in ML Systems](https://papers.nips.cc/paper/2015/file/86df7dcfd896fcaf2674f757a2463eba-Paper.pdf)

### Video Tutorials
- [Building ML Platforms - MLOps Community](https://www.youtube.com/c/MLOps)
- [Kubernetes for ML Engineers](https://www.youtube.com/playlist?list=PLIivdWyY5sqLq-eM4W2bIgbrpAsP5aLtL)
- [Platform Engineering Best Practices](https://www.youtube.com/watch?v=ghcb0kIryBY)

### Community Resources
- [r/mlops Subreddit](https://www.reddit.com/r/mlops/)
- [MLOps Community Slack](https://mlops.community/)
- [CNCF Slack #kubeflow](https://cloud-native.slack.com/)

---

**Module Complete! 🎉**

You now understand the foundals of ML platform engineering. Continue to Module 02 to dive deeper into API design patterns and build production-grade platform interfaces.
