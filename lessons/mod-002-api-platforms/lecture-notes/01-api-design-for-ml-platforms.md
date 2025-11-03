# API Design for ML Platforms

**Role**: ML Platform Engineer
**Level**: Core Concepts
**Duration**: 10 hours
**Prerequisites**: Module 01 completed, REST API experience, understanding of HTTP protocols

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Compare** REST and gRPC trade-offs for ML platform use cases
2. **Design** versioned APIs that support backward compatibility
3. **Implement** both synchronous and streaming API patterns
4. **Generate** client SDKs from OpenAPI and Protocol Buffer specifications
5. **Create** comprehensive API documentation for platform users
6. **Apply** rate limiting and authentication patterns
7. **Optimize** API performance for ML workloads

## Introduction (500 words)

### Why API Design Matters for ML Platforms

Your platform's API is its primary interface. While you might build sophisticated infrastructure with Kubernetes operators, distributed training, and auto-scaling—if your API is difficult to use, teams will build their own solutions or avoid the platform entirely.

**Great API design** for ML platforms requires balancing:

- **Simplicity**: Data scientists shouldn't need to understand Kubernetes to train models
- **Power**: Advanced users need access to low-level controls
- **Performance**: Training jobs might submit millions of data points; inference might serve millions of requests
- **Versioning**: APIs must evolve without breaking existing users
- **Documentation**: Self-service requires excellent docs

### Real-World Impact

Consider these scenarios:

**Scenario 1: Poor API Design**
```python
# User must construct complex JSON manually
response = requests.post("http://platform/api/v1/jobs", json={
    "job_type": "training",
    "spec": {
        "containers": [{
            "image": "pytorch/pytorch:latest",
            "command": ["python", "train.py"],
            "resources": {
                "limits": {"nvidia.com/gpu": "2", "memory": "32Gi"},
                "requests": {"nvidia.com/gpu": "2", "memory": "32Gi"}
            }
        }],
        "volumes": [{"name": "data", "persistentVolumeClaim": {"claimName": "ml-data"}}],
        "volumeMounts": [{"name": "data", "mountPath": "/data"}]
    }
})
# Users spend hours debugging Kubernetes YAML issues
```

**Scenario 2: Well-Designed API**
```python
# Simple, intuitive interface
from ml_platform import Client

client = Client()
job = client.train(
    model_type="pytorch",
    data="s3://bucket/data.csv",
    gpu_count=2,
    entrypoint="train.py"
)
print(f"Job submitted: {job.id}")
job.wait()  # Wait for completion
print(f"Model saved: {job.output_path}")
```

The second approach abstracts infrastructure complexity while maintaining power. This is the essence of platform API design.

### What You'll Build

In this module, you'll design and implement production-grade APIs for an ML platform, including:

- **RESTful APIs** for job submission, model management, and resource queries
- **gRPC services** for high-performance model serving
- **WebSocket endpoints** for real-time job logs and metrics
- **Python SDK** that wraps APIs for better developer experience
- **OpenAPI specification** for automatic documentation

By the end, you'll understand the trade-offs between different API approaches and how to choose the right pattern for each use case.

---

## Theoretical Foundations (1,500 words)

### 1. REST vs gRPC: When to Use Each

Both REST and gRPC have their place in ML platforms. Understanding the trade-offs is crucial for making the right choice.

#### RESTful APIs

**Strengths:**
- **Universal compatibility**: Works with any HTTP client
- **Easy debugging**: Can test with curl, Postman, browser
- **Human-readable**: JSON payloads are easy to inspect
- **Caching**: HTTP caching (ETags, conditional requests) built-in
- **Well-understood**: Most developers have REST experience

**Weaknesses:**
- **Verbose**: JSON is text-based, larger payloads than binary
- **No streaming**: Request/response only (though SSE/WebSockets help)
- **Performance overhead**: JSON parsing, HTTP/1.1 overhead
- **Loose contracts**: No enforced schema (unless using OpenAPI validation)

**Best Use Cases for ML Platforms:**
- **Job submission** - Not latency-critical, benefits from ease of use
- **Model management** - CRUD operations on model metadata
- **Resource queries** - Checking quotas, listing jobs, getting logs
- **Web dashboards** - UI components consume REST APIs easily

**Example REST Endpoint Design:**

```python
# RESTful API for ML platform jobs
# POST /v1/jobs/training - Submit training job
# GET  /v1/jobs/training/{job_id} - Get job status
# GET  /v1/jobs/training - List jobs with filters
# DELETE /v1/jobs/training/{job_id} - Cancel job
# GET  /v1/jobs/training/{job_id}/logs - Stream logs (SSE)
# GET  /v1/jobs/training/{job_id}/metrics - Get metrics

from fastapi import FastAPI, HTTPException, Query
from typing import List, Optional
from enum import Enum

app = FastAPI(title="ML Platform API", version="1.0.0")

class JobStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCEEDED = "succeeded"
    FAILED = "failed"

@app.get("/v1/jobs/training", response_model=List[TrainingJob])
async def list_jobs(
    status: Optional[JobStatus] = Query(None, description="Filter by status"),
    limit: int = Query(50, ge=1, le=1000),
    offset: int = Query(0, ge=0)
):
    """List training jobs with pagination and filtering"""
    # Implementation...
    pass
```

#### gRPC APIs

**Strengths:**
- **High performance**: Binary Protocol Buffers, HTTP/2, multiplexing
- **Strongly typed**: .proto files define contracts, compile-time checking
- **Streaming**: Bidirectional streaming built-in
- **Code generation**: Auto-generate clients in multiple languages
- **Efficient**: 2.5x faster throughput than JSON/REST (see research)

**Weaknesses:**
- **Browser limitations**: No native browser support (need grpc-web proxy)
- **Debugging complexity**: Binary format, need special tools
- **Learning curve**: More complex than REST
- **Less universal**: Requires gRPC client libraries

**Best Use Cases for ML Platforms:**
- **Model serving** - High-throughput, low-latency inference critical
- **Feature serving** - Real-time feature lookups need speed
- **Distributed training** - Workers communicate with master node
- **Streaming predictions** - Continuous inference on data streams
- **Internal microservices** - Service-to-service communication

**Example gRPC Service Design:**

```protobuf
// model_serving.proto
syntax = "proto3";

package ml_platform.serving;

// Model serving service
service ModelServing {
  // Single prediction
  rpc Predict(PredictRequest) returns (PredictResponse);

  // Batch predictions
  rpc PredictBatch(PredictBatchRequest) returns (PredictBatchResponse);

  // Streaming predictions
  rpc PredictStream(stream PredictRequest) returns (stream PredictResponse);
}

message PredictRequest {
  string model_id = 1;
  string model_version = 2;
  repeated float features = 3;
  map<string, string> metadata = 4;
}

message PredictResponse {
  repeated float predictions = 1;
  map<string, float> confidence_scores = 2;
  int64 latency_ms = 3;
}

message PredictBatchRequest {
  string model_id = 1;
  string model_version = 2;
  repeated PredictRequest requests = 3;
}

message PredictBatchResponse {
  repeated PredictResponse responses = 1;
}
```

**Implementing the gRPC Service:**

```python
# server.py
import grpc
from concurrent import futures
import model_serving_pb2
import model_serving_pb2_grpc
import time

class ModelServingServicer(model_serving_pb2_grpc.ModelServingServicer):
    """Implementation of ModelServing service"""

    def __init__(self):
        self.models = {}  # model_id -> model object

    def Predict(self, request, context):
        """Handle single prediction request"""
        start_time = time.time()

        # Load model
        model = self.models.get(request.model_id)
        if not model:
            context.abort(grpc.StatusCode.NOT_FOUND, f"Model {request.model_id} not found")

        # Run inference
        predictions = model.predict([request.features])

        # Build response
        response = model_serving_pb2.PredictResponse(
            predictions=predictions[0].tolist(),
            latency_ms=int((time.time() - start_time) * 1000)
        )

        return response

    def PredictBatch(self, request, context):
        """Handle batch prediction request"""
        responses = []
        for req in request.requests:
            resp = self.Predict(req, context)
            responses.append(resp)

        return model_serving_pb2.PredictBatchResponse(responses=responses)

    def PredictStream(self, request_iterator, context):
        """Handle streaming predictions"""
        for request in request_iterator:
            response = self.Predict(request, context)
            yield response

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    model_serving_pb2_grpc.add_ModelServingServicer_to_server(
        ModelServingServicer(), server
    )
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

#### Performance Comparison

Based on 2025 benchmarks:

| Metric | REST (JSON) | gRPC (Protobuf) | Improvement |
|--------|-------------|-----------------|-------------|
| Throughput | 3,500 req/s | 8,700 req/s | **2.5x** |
| Latency (p50) | 28ms | 11ms | **2.5x faster** |
| Latency (p99) | 95ms | 35ms | **2.7x faster** |
| Payload Size | 100% | 40% | **2.5x smaller** |
| CPU Usage | 100% | 65% | **35% reduction** |

**Recommendation**: Use REST for control plane (job submission, management), gRPC for data plane (model serving, feature serving).

### 2. API Versioning Strategies

APIs must evolve without breaking existing users. ML platforms face unique versioning challenges:

- **Long-running jobs**: Job submitted with v1 API might run for days; must support v1 responses
- **Client diversity**: Teams might use different SDK versions
- **Model compatibility**: Models trained with v1 features must still work when platform upgrades

#### Versioning Approaches

**1. URL-based Versioning** (Recommended for ML platforms)

```python
# v1 API
POST /v1/jobs/training
GET  /v1/jobs/training/{id}

# v2 API (adds new fields, deprecates some options)
POST /v2/jobs/training
GET  /v2/jobs/training/{id}

# Both versions supported simultaneously
```

**Pros:**
- Clear which version client is using
- Easy to route requests to different implementations
- Can maintain separate documentation per version

**Cons:**
- URL changes might confuse users
- Proliferation of versions (v1, v2, v3, ...)

**2. Header-based Versioning**

```python
# Same URL, version in header
POST /jobs/training
Header: API-Version: 2024-10-01

# Date-based versioning (Stripe model)
```

**Pros:**
- Clean URLs
- Gradual migration path

**Cons:**
- Less discoverable
- Testing requires header manipulation

**3. Content Negotiation**

```python
# Version in Accept header
POST /jobs/training
Accept: application/vnd.ml-platform.v2+json
```

**Best Practice for ML Platforms:**

```python
# Hybrid approach: URL major versions, headers for minor
@app.post("/v1/jobs/training")
async def create_job_v1(spec: TrainingJobSpecV1):
    """V1 API - stable"""
    pass

@app.post("/v2/jobs/training")
async def create_job_v2(
    spec: TrainingJobSpecV2,
    api_date: str = Header(None, alias="API-Date")
):
    """V2 API with dated features"""
    # api_date enables/disables specific features
    # e.g., "2024-10-01" vs "2025-01-15"
    pass
```

#### Deprecation Strategy

```python
from datetime import datetime, timedelta
from fastapi import Response

@app.post("/v1/jobs/training")
async def create_job_v1(spec: TrainingJobSpecV1, response: Response):
    """V1 API - DEPRECATED"""
    # Add deprecation headers
    sunset_date = datetime(2025, 12, 31)
    response.headers["Sunset"] = sunset_date.isoformat()
    response.headers["Deprecation"] = "true"
    response.headers["Link"] = '</docs/v2-migration>; rel="sunset"'

    # Log usage for migration tracking
    logger.warning(f"V1 API used by {request.client.host} - migrate to V2")

    return await create_job(spec)
```

### 3. OpenAPI Specification

OpenAPI (formerly Swagger) enables:
- **Auto-generated documentation**: Interactive API explorer
- **Client code generation**: Generate SDKs in Python, Go, TypeScript
- **Validation**: Request/response schema validation
- **Contract testing**: Ensure API matches specification

FastAPI generates OpenAPI automatically:

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(
    title="ML Platform API",
    description="Self-service ML infrastructure platform",
    version="2.0.0",
    docs_url="/docs",  # Swagger UI
    redoc_url="/redoc",  # ReDoc documentation
)

class TrainingJobSpec(BaseModel):
    """Training job specification"""
    name: str = Field(..., description="Unique job name", example="churn-model-v1")
    docker_image: str = Field(..., description="Docker image with training code")
    data_path: str = Field(..., description="S3/GCS path to training data")

    class Config:
        schema_extra = {
            "example": {
                "name": "churn-prediction-v1",
                "docker_image": "company/ml-train:latest",
                "data_path": "s3://ml-data/churn.csv"
            }
        }

@app.post(
    "/v1/jobs/training",
    response_model=TrainingJob,
    tags=["Training"],
    summary="Submit training job",
    description="Create and submit a new ML training job to the platform",
    responses={
        201: {"description": "Job created successfully"},
        400: {"description": "Invalid job specification"},
        429: {"description": "Rate limit exceeded or quota exceeded"}
    }
)
async def create_training_job(spec: TrainingJobSpec):
    """Submit a training job to the ML platform."""
    # Implementation...
    pass
```

Access documentation at `http://localhost:8000/docs` - interactive API explorer with "Try it out" functionality.

#### Generating Client SDKs

```bash
# Export OpenAPI spec
curl http://localhost:8000/openapi.json > openapi.json

# Generate Python client
openapi-generator-cli generate \
  -i openapi.json \
  -g python \
  -o python-client/ \
  --additional-properties=packageName=ml_platform_client

# Generate TypeScript client
openapi-generator-cli generate \
  -i openapi.json \
  -g typescript-axios \
  -o typescript-client/

# Now users can install and use generated clients
pip install ml_platform_client
```

Generated client usage:

```python
from ml_platform_client import ApiClient, Configuration, TrainingApi
from ml_platform_client.models import TrainingJobSpec

config = Configuration(host="https://ml-platform.company.com")
config.api_key['BearerAuth'] = "your-token"

client = ApiClient(configuration=config)
api = TrainingApi(client)

# Submit job using generated client
spec = TrainingJobSpec(
    name="my-model",
    docker_image="pytorch/pytorch:latest",
    data_path="s3://data/train.csv"
)

job = api.create_training_job(spec)
print(f"Job created: {job.id}, Status: {job.status}")
```

### 4. Authentication & Authorization

ML platforms need robust auth to:
- Identify tenants (teams/projects)
- Enforce resource quotas
- Audit who did what
- Integrate with corporate SSO

#### JWT-based Authentication

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import jwt, JWTError
from datetime import datetime, timedelta

security = HTTPBearer()

SECRET_KEY = "your-secret-key"  # Store in env var
ALGORITHM = "HS256"

def create_token(tenant_id: str, user_id: str, expiration_hours: int = 24) -> str:
    """Generate JWT token"""
    expire = datetime.utcnow() + timedelta(hours=expiration_hours)
    to_encode = {
        "sub": user_id,
        "tenant_id": tenant_id,
        "exp": expire,
        "iat": datetime.utcnow()
    }
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

def decode_token(credentials: HTTPAuthorizationCredentials = Depends(security)) -> dict:
    """Verify and decode JWT token"""
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("exp") < datetime.utcnow().timestamp():
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Token expired")
        return payload
    except JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")

@app.post("/v1/jobs/training")
async def create_training_job(
    spec: TrainingJobSpec,
    token_data: dict = Depends(decode_token)
):
    """Create training job - requires valid JWT"""
    tenant_id = token_data["tenant_id"]
    user_id = token_data["sub"]

    # Verify tenant matches job spec
    if spec.tenant_id != tenant_id:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Cannot create jobs for other tenants")

    # Create job...
    pass
```

#### API Key Authentication (for programmatic access)

```python
from fastapi import Security
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key")

async def verify_api_key(api_key: str = Security(api_key_header)) -> dict:
    """Verify API key and return tenant info"""
    # Look up API key in database
    tenant_info = await db.get_tenant_by_api_key(api_key)
    if not tenant_info:
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Invalid API key")
    return tenant_info

@app.post("/v1/jobs/training")
async def create_training_job(
    spec: TrainingJobSpec,
    tenant: dict = Depends(verify_api_key)
):
    """Create training job using API key auth"""
    pass
```

---

## Practical Implementation (2,000 words)

### Building a Production ML Platform API

Let's build a complete, production-ready API for an ML platform including REST endpoints, gRPC services, and a Python SDK.

#### Architecture

```
┌────────────────────────────────────────────────────────┐
│                      API Gateway                        │
│  (Authentication, Rate Limiting, Request Routing)      │
└────────┬─────────────────────────────────┬─────────────┘
         │                                 │
         ▼                                 ▼
┌──────────────────┐           ┌──────────────────────┐
│   REST API       │           │    gRPC Services     │
│   (Control Plane)│           │    (Data Plane)      │
│                  │           │                      │
│  - Job Mgmt      │           │  - Model Serving     │
│  - Model Mgmt    │           │  - Feature Serving   │
│  - Resources     │           │  - Streaming         │
└──────────────────┘           └──────────────────────┘
```

#### Step 1: Define API Models with Pydantic

```python
# models/training.py
from pydantic import BaseModel, Field, validator
from typing import Optional, Dict, List
from enum import Enum
from datetime import datetime

class ComputeType(str, Enum):
    CPU = "cpu"
    GPU = "gpu"
    TPU = "tpu"

class FrameworkType(str, Enum):
    PYTORCH = "pytorch"
    TENSORFLOW = "tensorflow"
    SKLEARN = "sklearn"
    XGBOOST = "xgboost"

class InstanceConfig(BaseModel):
    compute_type: ComputeType = ComputeType.CPU
    cpu_cores: int = Field(4, ge=1, le=96, description="CPU cores")
    memory_gb: int = Field(16, ge=1, le=768, description="Memory in GB")
    gpu_count: Optional[int] = Field(None, ge=0, le=8, description="Number of GPUs")
    gpu_type: Optional[str] = Field(None, description="GPU type (e.g., nvidia-a100)")

    @validator('gpu_count')
    def validate_gpu_config(cls, v, values):
        if values.get('compute_type') == ComputeType.GPU and (v is None or v == 0):
            raise ValueError("gpu_count must be > 0 when compute_type is GPU")
        return v

class TrainingJobSpec(BaseModel):
    name: str = Field(..., min_length=1, max_length=63, regex=r'^[a-z0-9-]+$')
    framework: FrameworkType
    docker_image: str = Field(..., description="Docker image containing training code")
    entrypoint: str = Field("python train.py", description="Command to execute")
    data_path: str = Field(..., description="S3/GCS path to training data")
    output_path: str = Field(..., description="S3/GCS path for model output")
    compute: InstanceConfig = Field(default_factory=InstanceConfig)
    hyperparameters: Dict[str, any] = Field(default_factory=dict)
    env_vars: Dict[str, str] = Field(default_factory=dict)
    tags: Dict[str, str] = Field(default_factory=dict, description="Custom tags for organizing jobs")

    class Config:
        schema_extra = {
            "example": {
                "name": "churn-model-v1",
                "framework": "pytorch",
                "docker_image": "company/ml-train:pytorch-1.13",
                "data_path": "s3://ml-data/churn/train.csv",
                "output_path": "s3://ml-models/churn/v1/",
                "compute": {
                    "compute_type": "gpu",
                    "cpu_cores": 8,
                    "memory_gb": 32,
                    "gpu_count": 2,
                    "gpu_type": "nvidia-a100"
                },
                "hyperparameters": {
                    "learning_rate": 0.001,
                    "batch_size": 64,
                    "epochs": 100
                },
                "tags": {
                    "team": "ml-analytics",
                    "project": "customer-retention"
                }
            }
        }

class JobStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    CANCELLED = "cancelled"

class TrainingJob(BaseModel):
    id: str
    spec: TrainingJobSpec
    status: JobStatus
    tenant_id: str
    created_by: str
    created_at: datetime
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    error_message: Optional[str] = None
    metrics: Dict[str, float] = Field(default_factory=dict)
    resource_usage: Optional[Dict[str, float]] = None

class JobListResponse(BaseModel):
    jobs: List[TrainingJob]
    total: int
    page: int
    page_size: int
    has_more: bool
```

#### Step 2: Implement REST API with FastAPI

```python
# api/training.py
from fastapi import APIRouter, Depends, HTTPException, Query, status
from typing import List, Optional
import uuid
from datetime import datetime

from models.training import TrainingJobSpec, TrainingJob, JobStatus, JobListResponse
from services.job_service import JobService
from auth.dependencies import get_current_tenant

router = APIRouter(prefix="/v1/jobs/training", tags=["Training"])

@router.post("", response_model=TrainingJob, status_code=status.HTTP_201_CREATED)
async def create_training_job(
    spec: TrainingJobSpec,
    tenant_id: str = Depends(get_current_tenant),
    job_service: JobService = Depends()
):
    """
    Submit a new training job to the ML platform.

    The job will be queued and executed on available compute resources based on the specified configuration.
    """
    # Generate job ID
    job_id = str(uuid.uuid4())

    # Create job
    job = await job_service.create_job(
        job_id=job_id,
        spec=spec,
        tenant_id=tenant_id
    )

    return job

@router.get("/{job_id}", response_model=TrainingJob)
async def get_training_job(
    job_id: str,
    tenant_id: str = Depends(get_current_tenant),
    job_service: JobService = Depends()
):
    """Get details of a specific training job"""
    job = await job_service.get_job(job_id, tenant_id)
    if not job:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Job not found")
    return job

@router.get("", response_model=JobListResponse)
async def list_training_jobs(
    tenant_id: str = Depends(get_current_tenant),
    status: Optional[JobStatus] = Query(None, description="Filter by status"),
    framework: Optional[str] = Query(None, description="Filter by ML framework"),
    tags: Optional[str] = Query(None, description="Filter by tags (format: key1=value1,key2=value2)"),
    page: int = Query(1, ge=1, description="Page number"),
    page_size: int = Query(50, ge=1, le=500, description="Results per page"),
    job_service: JobService = Depends()
):
    """
    List training jobs with filtering and pagination.

    Supports filtering by status, framework, and custom tags.
    """
    # Parse tags filter
    tag_filters = {}
    if tags:
        for tag_pair in tags.split(','):
            key, value = tag_pair.split('=')
            tag_filters[key] = value

    jobs, total = await job_service.list_jobs(
        tenant_id=tenant_id,
        status_filter=status,
        framework_filter=framework,
        tag_filters=tag_filters,
        page=page,
        page_size=page_size
    )

    return JobListResponse(
        jobs=jobs,
        total=total,
        page=page,
        page_size=page_size,
        has_more=(page * page_size) < total
    )

@router.delete("/{job_id}", status_code=status.HTTP_204_NO_CONTENT)
async def cancel_training_job(
    job_id: str,
    tenant_id: str = Depends(get_current_tenant),
    job_service: JobService = Depends()
):
    """Cancel a running or pending training job"""
    success = await job_service.cancel_job(job_id, tenant_id)
    if not success:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Job not found")

@router.get("/{job_id}/logs")
async def stream_job_logs(
    job_id: str,
    tenant_id: str = Depends(get_current_tenant),
    follow: bool = Query(False, description="Stream logs in real-time"),
    tail: int = Query(100, ge=0, le=10000, description="Number of lines to return"),
    job_service: JobService = Depends()
):
    """
    Get job logs. If follow=true, streams logs using Server-Sent Events (SSE).
    """
    if follow:
        # Return SSE stream
        return EventSourceResponse(
            job_service.stream_logs(job_id, tenant_id)
        )
    else:
        # Return static logs
        logs = await job_service.get_logs(job_id, tenant_id, tail=tail)
        return {"logs": logs}

@router.get("/{job_id}/metrics", response_model=Dict[str, float])
async def get_job_metrics(
    job_id: str,
    tenant_id: str = Depends(get_current_tenant),
    job_service: JobService = Depends()
):
    """Get training metrics (loss, accuracy, etc.) for a job"""
    metrics = await job_service.get_metrics(job_id, tenant_id)
    if metrics is None:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Job not found or no metrics available")
    return metrics
```

#### Step 3: Implement gRPC Model Serving

```protobuf
// proto/serving.proto
syntax = "proto3";

package ml_platform.serving;

service ModelServing {
  // Get model information
  rpc GetModel(GetModelRequest) returns (ModelInfo);

  // Single prediction
  rpc Predict(PredictRequest) returns (PredictResponse);

  // Batch predictions
  rpc PredictBatch(PredictBatchRequest) returns (PredictBatchResponse);

  // Streaming predictions (bidirectional)
  rpc PredictStream(stream PredictRequest) returns (stream PredictResponse);

  // Model health check
  rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse);
}

message GetModelRequest {
  string model_id = 1;
  string version = 2;  // "latest" or specific version
}

message ModelInfo {
  string model_id = 1;
  string version = 2;
  string framework = 3;
  repeated string input_features = 4;
  repeated string output_labels = 5;
  map<string, string> metadata = 6;
}

message PredictRequest {
  string model_id = 1;
  string version = 2;
  repeated Feature features = 3;
  map<string, string> metadata = 4;
}

message Feature {
  string name = 1;
  oneof value {
    float float_value = 2;
    int64 int_value = 3;
    string string_value = 4;
    bool bool_value = 5;
    FloatArray float_array = 6;
  }
}

message FloatArray {
  repeated float values = 1;
}

message PredictResponse {
  repeated float predictions = 1;
  map<string, float> confidence_scores = 2;
  int64 latency_microseconds = 3;
  string model_version_used = 4;
}

message PredictBatchRequest {
  string model_id = 1;
  string version = 2;
  repeated PredictRequest requests = 3;
}

message PredictBatchResponse {
  repeated PredictResponse responses = 1;
  int64 total_latency_microseconds = 2;
}

message HealthCheckRequest {
  string model_id = 1;
}

message HealthCheckResponse {
  bool healthy = 1;
  string status = 2;
  map<string, string> details = 3;
}
```

**Implementation:**

```python
# grpc_server/serving.py
import grpc
from concurrent import futures
import time
import logging

import serving_pb2
import serving_pb2_grpc

logger = logging.getLogger(__name__)

class ModelServingServicer(serving_pb2_grpc.ModelServingServicer):
    """gRPC service for model serving"""

    def __init__(self, model_loader):
        self.model_loader = model_loader

    def GetModel(self, request, context):
        """Get model metadata"""
        try:
            model_info = self.model_loader.get_model_info(
                request.model_id,
                request.version or "latest"
            )

            return serving_pb2.ModelInfo(
                model_id=model_info["id"],
                version=model_info["version"],
                framework=model_info["framework"],
                input_features=model_info["input_features"],
                output_labels=model_info["output_labels"],
                metadata=model_info["metadata"]
            )
        except KeyError:
            context.abort(grpc.StatusCode.NOT_FOUND, f"Model {request.model_id} not found")

    def Predict(self, request, context):
        """Handle single prediction"""
        start_time = time.time()

        try:
            # Load model
            model = self.model_loader.load_model(request.model_id, request.version or "latest")

            # Parse features
            features = self._parse_features(request.features)

            # Run prediction
            predictions = model.predict(features)

            # Calculate latency
            latency_us = int((time.time() - start_time) * 1_000_000)

            # Build response
            response = serving_pb2.PredictResponse(
                predictions=predictions.tolist(),
                latency_microseconds=latency_us,
                model_version_used=model.version
            )

            # Log metrics
            logger.info(f"Prediction: model={request.model_id}, latency={latency_us}us")

            return response

        except Exception as e:
            logger.error(f"Prediction failed: {e}")
            context.abort(grpc.StatusCode.INTERNAL, str(e))

    def PredictBatch(self, request, context):
        """Handle batch predictions"""
        start_time = time.time()

        responses = []
        for req in request.requests:
            try:
                resp = self.Predict(req, context)
                responses.append(resp)
            except Exception as e:
                # Continue processing other requests
                logger.error(f"Batch item failed: {e}")
                responses.append(serving_pb2.PredictResponse())

        total_latency_us = int((time.time() - start_time) * 1_000_000)

        return serving_pb2.PredictBatchResponse(
            responses=responses,
            total_latency_microseconds=total_latency_us
        )

    def PredictStream(self, request_iterator, context):
        """Handle streaming predictions"""
        for request in request_iterator:
            try:
                response = self.Predict(request, context)
                yield response
            except Exception as e:
                logger.error(f"Stream prediction failed: {e}")
                context.abort(grpc.StatusCode.INTERNAL, str(e))

    def HealthCheck(self, request, context):
        """Health check for model"""
        try:
            model = self.model_loader.load_model(request.model_id, "latest")
            is_healthy = model.is_healthy()

            return serving_pb2.HealthCheckResponse(
                healthy=is_healthy,
                status="healthy" if is_healthy else "unhealthy",
                details={"model_loaded": "true", "version": model.version}
            )
        except Exception as e:
            return serving_pb2.HealthCheckResponse(
                healthy=False,
                status="unhealthy",
                details={"error": str(e)}
            )

    def _parse_features(self, proto_features):
        """Parse protobuf features to numpy array"""
        features = []
        for feature in proto_features:
            if feature.HasField("float_value"):
                features.append(feature.float_value)
            elif feature.HasField("int_value"):
                features.append(float(feature.int_value))
            elif feature.HasField("float_array"):
                features.extend(feature.float_array.values)
        return np.array([features])

def serve():
    """Start gRPC server"""
    server = grpc.server(
        futures.ThreadPoolExecutor(max_workers=10),
        options=[
            ('grpc.max_send_message_length', 100 * 1024 * 1024),  # 100MB
            ('grpc.max_receive_message_length', 100 * 1024 * 1024),
        ]
    )

    # Initialize model loader
    model_loader = ModelLoader(model_path="/models")

    # Add servicer
    serving_pb2_grpc.add_ModelServingServicer_to_server(
        ModelServingServicer(model_loader),
        server
    )

    # Start server
    port = 50051
    server.add_insecure_port(f'[::]:{port}')
    server.start()
    logger.info(f"gRPC server started on port {port}")
    server.wait_for_termination()

if __name__ == '__main__':
    logging.basicConfig(level=logging.INFO)
    serve()
```

#### Step 4: Build Python SDK

```python
# sdk/ml_platform/__init__.py
"""ML Platform Python SDK"""

from .client import Client, AsyncClient
from .resources import TrainingJob, Model, Deployment
from .exceptions import MLPlatformError, AuthenticationError, ResourceNotFoundError

__version__ = "1.0.0"
__all__ = [
    "Client",
    "AsyncClient",
    "TrainingJob",
    "Model",
    "Deployment",
    "MLPlatformError"
]
```

```python
# sdk/ml_platform/client.py
import requests
from typing import Optional, Dict, List
from urllib.parse import urljoin
import time

from .resources import TrainingJob, Model
from .exceptions import MLPlatformError, AuthenticationError

class Client:
    """Synchronous ML Platform client"""

    def __init__(
        self,
        base_url: str = "https://ml-platform.company.com",
        api_key: Optional[str] = None,
        timeout: int = 30,
        max_retries: int = 3
    ):
        self.base_url = base_url.rstrip('/')
        self.api_key = api_key
        self.timeout = timeout
        self.max_retries = max_retries
        self.session = requests.Session()

        if api_key:
            self.session.headers.update({"X-API-Key": api_key})

    def train(
        self,
        name: str,
        framework: str,
        docker_image: str,
        data_path: str,
        output_path: str,
        gpu_count: int = 0,
        **kwargs
    ) -> TrainingJob:
        """
        Submit a training job (simplified interface).

        Args:
            name: Job name
            framework: ML framework (pytorch, tensorflow, sklearn, xgboost)
            docker_image: Docker image containing training code
            data_path: S3/GCS path to training data
            output_path: S3/GCS path for model output
            gpu_count: Number of GPUs (0 for CPU)
            **kwargs: Additional parameters (hyperparameters, env_vars, etc.)

        Returns:
            TrainingJob object

        Example:
            >>> client = Client(api_key="your-key")
            >>> job = client.train(
            ...     name="churn-model-v1",
            ...     framework="pytorch",
            ...     docker_image="company/ml-train:latest",
            ...     data_path="s3://data/train.csv",
            ...     output_path="s3://models/churn/v1/",
            ...     gpu_count=2
            ... )
            >>> print(f"Job submitted: {job.id}")
        """
        # Build job spec
        spec = {
            "name": name,
            "framework": framework,
            "docker_image": docker_image,
            "data_path": data_path,
            "output_path": output_path,
            "compute": {
                "compute_type": "gpu" if gpu_count > 0 else "cpu",
                "gpu_count": gpu_count
            }
        }

        # Add optional parameters
        if "hyperparameters" in kwargs:
            spec["hyperparameters"] = kwargs["hyperparameters"]
        if "env_vars" in kwargs:
            spec["env_vars"] = kwargs["env_vars"]
        if "tags" in kwargs:
            spec["tags"] = kwargs["tags"]

        # Submit job
        response = self._post("/v1/jobs/training", json=spec)
        return TrainingJob(self, response)

    def get_job(self, job_id: str) -> TrainingJob:
        """Get training job by ID"""
        response = self._get(f"/v1/jobs/training/{job_id}")
        return TrainingJob(self, response)

    def list_jobs(
        self,
        status: Optional[str] = None,
        framework: Optional[str] = None,
        limit: int = 50
    ) -> List[TrainingJob]:
        """List training jobs with optional filters"""
        params = {"page_size": limit}
        if status:
            params["status"] = status
        if framework:
            params["framework"] = framework

        response = self._get("/v1/jobs/training", params=params)
        return [TrainingJob(self, job_data) for job_data in response["jobs"]]

    def _get(self, path: str, **kwargs) -> dict:
        """Make GET request with retries"""
        return self._request("GET", path, **kwargs)

    def _post(self, path: str, **kwargs) -> dict:
        """Make POST request with retries"""
        return self._request("POST", path, **kwargs)

    def _request(self, method: str, path: str, **kwargs) -> dict:
        """Make HTTP request with retries and error handling"""
        url = urljoin(self.base_url, path)
        kwargs.setdefault("timeout", self.timeout)

        for attempt in range(self.max_retries):
            try:
                response = self.session.request(method, url, **kwargs)

                # Handle errors
                if response.status_code == 401:
                    raise AuthenticationError("Invalid API key")
                elif response.status_code >= 400:
                    error_detail = response.json().get("detail", "Unknown error")
                    raise MLPlatformError(f"API error: {error_detail}", status_code=response.status_code)

                return response.json()

            except requests.RequestException as e:
                if attempt == self.max_retries - 1:
                    raise MLPlatformError(f"Request failed after {self.max_retries} attempts: {e}")
                time.sleep(2 ** attempt)  # Exponential backoff

class AsyncClient:
    """Async ML Platform client using aiohttp"""
    # Implementation similar to Client but using aiohttp
    pass
```

```python
# sdk/ml_platform/resources.py
from typing import Optional, Dict
from datetime import datetime
import time

class TrainingJob:
    """Represents a training job"""

    def __init__(self, client, data: dict):
        self.client = client
        self._data = data

    @property
    def id(self) -> str:
        return self._data["id"]

    @property
    def status(self) -> str:
        return self._data["status"]

    @property
    def name(self) -> str:
        return self._data["spec"]["name"]

    @property
    def created_at(self) -> datetime:
        return datetime.fromisoformat(self._data["created_at"].replace('Z', '+00:00'))

    @property
    def output_path(self) -> Optional[str]:
        if self.status == "succeeded":
            return self._data["spec"]["output_path"]
        return None

    def refresh(self):
        """Refresh job status from API"""
        updated_data = self.client._get(f"/v1/jobs/training/{self.id}")
        self._data = updated_data

    def wait(self, polling_interval: int = 10, timeout: Optional[int] = None):
        """
        Wait for job to complete.

        Args:
            polling_interval: Seconds between status checks
            timeout: Maximum seconds to wait (None = indefinite)

        Raises:
            TimeoutError: If job doesn't complete within timeout
            MLPlatformError: If job fails
        """
        start_time = time.time()

        while self.status in ["pending", "running"]:
            if timeout and (time.time() - start_time) > timeout:
                raise TimeoutError(f"Job {self.id} did not complete within {timeout} seconds")

            time.sleep(polling_interval)
            self.refresh()

        if self.status == "failed":
            error_msg = self._data.get("error_message", "Unknown error")
            raise MLPlatformError(f"Job {self.id} failed: {error_msg}")

        print(f"Job {self.id} completed successfully!")

    def cancel(self):
        """Cancel the job"""
        self.client._request("DELETE", f"/v1/jobs/training/{self.id}")
        self.refresh()

    def get_logs(self, tail: int = 100) -> str:
        """Get job logs"""
        response = self.client._get(f"/v1/jobs/training/{self.id}/logs", params={"tail": tail})
        return response["logs"]

    def get_metrics(self) -> Dict[str, float]:
        """Get training metrics"""
        return self.client._get(f"/v1/jobs/training/{self.id}/metrics")

    def __repr__(self):
        return f"TrainingJob(id='{self.id}', name='{self.name}', status='{self.status}')"
```

**SDK Usage Example:**

```python
from ml_platform import Client

# Initialize client
client = Client(
    base_url="https://ml-platform.company.com",
    api_key="your-api-key"
)

# Simple training job submission
job = client.train(
    name="churn-prediction-v1",
    framework="pytorch",
    docker_image="company/ml-train:pytorch-1.13",
    data_path="s3://ml-data/churn/train.csv",
    output_path="s3://ml-models/churn/v1/",
    gpu_count=2,
    hyperparameters={
        "learning_rate": 0.001,
        "batch_size": 64,
        "epochs": 100
    },
    tags={
        "team": "ml-analytics",
        "project": "customer-retention"
    }
)

print(f"Job submitted: {job.id}")
print(f"Status: {job.status}")

# Wait for completion
try:
    job.wait(timeout=3600)  # 1 hour timeout
    print(f"Model saved to: {job.output_path}")
    print(f"Metrics: {job.get_metrics()}")
except TimeoutError:
    print("Job timed out")
    job.cancel()
except Exception as e:
    print(f"Job failed: {e}")
    logs = job.get_logs(tail=50)
    print(f"Last 50 log lines:\n{logs}")
```

---

*(Continued in next message due to length limit...)*
