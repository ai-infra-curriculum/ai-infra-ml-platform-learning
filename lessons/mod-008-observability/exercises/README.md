# Module 08 Exercises: Observability & Monitoring

Build a comprehensive observability stack for ML platforms with metrics, logs, traces, and drift detection.

**Total Time**: ~8.5 hours
**Difficulty**: Intermediate to Advanced

---

## Exercise Overview

| Exercise | Title | Duration | Difficulty | Key Skills |
|----------|-------|----------|------------|------------|
| 01 | Set Up Prometheus + Grafana Stack | 90 min | Intermediate | Prometheus, Grafana, Docker Compose |
| 02 | Instrument ML Service with Metrics | 75 min | Intermediate | prometheus_client, custom metrics |
| 03 | Implement Structured Logging | 60 min | Basic | structlog, JSON logging |
| 04 | Add Distributed Tracing with OpenTelemetry | 90 min | Advanced | OpenTelemetry, Tempo, traces |
| 05 | Build Data Drift Detection | 120 min | Advanced | Evidently AI, statistical tests |
| 06 | Create Alerting Rules | 45 min | Intermediate | AlertManager, PagerDuty |

---

## Prerequisites

Before starting these exercises, ensure you have:

- [x] Completed Module 07 (Developer Experience & SDKs)
- [x] Python 3.9+
- [x] Docker and Docker Compose
- [x] Basic understanding of time-series data
- [x] Familiarity with HTTP APIs

**Installation**:
```bash
# Create virtual environment
python -m venv observability-env
source observability-env/bin/activate

# Install dependencies
pip install prometheus-client==0.19.0 \
    structlog==24.1.0 \
    opentelemetry-api==1.22.0 \
    opentelemetry-sdk==1.22.0 \
    opentelemetry-exporter-otlp==1.22.0 \
    evidently==0.4.0 \
    fastapi==0.104.0 \
    uvicorn==0.24.0 \
    pandas==2.1.0 \
    scipy==1.11.0
```

---

## Exercise 01: Set Up Prometheus + Grafana Stack

**Duration**: 90 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Deploy Prometheus and Grafana with Docker Compose
- Configure Prometheus to scrape metrics
- Create basic Grafana dashboards
- Set up data sources

### Implementation

**Step 1: Create Docker Compose Stack**

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Metrics storage - Prometheus
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    ports:
      - "9090:9090"
    restart: unless-stopped

  # Log storage - Loki
  loki:
    image: grafana/loki:2.9.3
    container_name: loki
    volumes:
      - ./loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  # Trace storage - Tempo
  tempo:
    image: grafana/tempo:2.3.1
    container_name: tempo
    volumes:
      - ./tempo-config.yml:/etc/tempo.yaml
      - tempo-data:/var/tempo
    ports:
      - "3200:3200"   # Tempo HTTP
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    command: ["-config.file=/etc/tempo.yaml"]
    restart: unless-stopped

  # Visualization - Grafana
  grafana:
    image: grafana/grafana:10.2.2
    container_name: grafana
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - loki
      - tempo
    restart: unless-stopped

  # ML Service (from previous exercises)
  ml-service:
    build: ./ml-service
    container_name: ml-service
    ports:
      - "8000:8000"
    environment:
      - PROMETHEUS_PORT=8000
    restart: unless-stopped

volumes:
  prometheus-data:
  loki-data:
  tempo-data:
  grafana-data:
```

**Step 2: Configure Prometheus**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'ml-service'
    static_configs:
      - targets: ['ml-service:8000']
    metrics_path: '/metrics'
```

**Step 3: Configure Grafana Data Sources**

```yaml
# grafana-datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false

  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    editable: false
```

**Step 4: Start the Stack**

```bash
docker-compose up -d

# Check services
docker-compose ps

# Access Grafana
open http://localhost:3000

# Access Prometheus
open http://localhost:9090
```

### Success Criteria

- [ ] All services running (prometheus, loki, tempo, grafana)
- [ ] Prometheus scraping ml-service successfully
- [ ] Grafana accessible at localhost:3000
- [ ] All data sources configured in Grafana

---

## Exercise 02: Instrument ML Service with Metrics

**Duration**: 75 minutes
**Difficulty**: Intermediate

### Learning Objectives

- Add Prometheus metrics to ML service
- Track prediction latency, volume, and errors
- Monitor model performance metrics
- Expose metrics endpoint

### Implementation

```python
# ml_service/metrics.py
from prometheus_client import (
    Counter, Histogram, Gauge, generate_latest, CONTENT_TYPE_LATEST
)
from fastapi import Response

# Prediction metrics
predictions_total = Counter(
    'ml_predictions_total',
    'Total number of predictions',
    ['model_name', 'model_version', 'status']
)

prediction_latency = Histogram(
    'ml_prediction_duration_seconds',
    'Time spent making predictions',
    ['model_name', 'model_version'],
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

prediction_scores = Histogram(
    'ml_prediction_scores',
    'Distribution of prediction scores',
    ['model_name', 'model_version'],
    buckets=[0.0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0]
)

confidence_scores = Histogram(
    'ml_confidence_scores',
    'Model confidence distribution',
    ['model_name', 'model_version'],
    buckets=[0.5, 0.6, 0.7, 0.8, 0.9, 0.95, 0.99, 1.0]
)

# Feature metrics
feature_value = Histogram(
    'ml_feature_value',
    'Feature value distribution',
    ['model_name', 'feature_name'],
    buckets=[0, 10, 50, 100, 500, 1000, 5000, 10000]
)

# Model info (gauge)
model_info = Gauge(
    'ml_model_info',
    'Model information',
    ['model_name', 'model_version', 'framework']
)

# Active requests
active_requests = Gauge(
    'ml_active_requests',
    'Currently processing requests'
)

def get_metrics() -> Response:
    """Metrics endpoint for Prometheus."""
    return Response(
        content=generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )
```

```python
# ml_service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import time
import numpy as np
from .metrics import (
    predictions_total, prediction_latency, prediction_scores,
    confidence_scores, feature_value, active_requests, get_metrics
)

app = FastAPI(title="ML Service")

class PredictionRequest(BaseModel):
    features: dict[str, float]
    model_name: str = "fraud-detection"
    model_version: str = "v1.0"

class PredictionResponse(BaseModel):
    prediction: float
    confidence: float
    model_name: str
    model_version: str

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    """Make a prediction."""
    active_requests.inc()
    start_time = time.time()

    try:
        # Simulate model prediction
        features_array = np.array(list(request.features.values()))
        prediction_score = float(np.random.random())
        confidence = max(prediction_score, 1 - prediction_score)

        # Record metrics
        latency = time.time() - start_time

        prediction_latency.labels(
            model_name=request.model_name,
            model_version=request.model_version
        ).observe(latency)

        prediction_scores.labels(
            model_name=request.model_name,
            model_version=request.model_version
        ).observe(prediction_score)

        confidence_scores.labels(
            model_name=request.model_name,
            model_version=request.model_version
        ).observe(confidence)

        # Record feature distributions
        for feature_name, value in request.features.items():
            feature_value.labels(
                model_name=request.model_name,
                feature_name=feature_name
            ).observe(value)

        predictions_total.labels(
            model_name=request.model_name,
            model_version=request.model_version,
            status='success'
        ).inc()

        return PredictionResponse(
            prediction=prediction_score,
            confidence=confidence,
            model_name=request.model_name,
            model_version=request.model_version
        )

    except Exception as e:
        predictions_total.labels(
            model_name=request.model_name,
            model_version=request.model_version,
            status='error'
        ).inc()
        raise HTTPException(status_code=500, detail=str(e))

    finally:
        active_requests.dec()

@app.get("/metrics")
async def metrics():
    """Prometheus metrics endpoint."""
    return get_metrics()

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

### Success Criteria

- [ ] Metrics endpoint exposes Prometheus metrics
- [ ] Predictions tracked with labels (model_name, version, status)
- [ ] Latency histograms with appropriate buckets
- [ ] Prediction score distribution tracked
- [ ] Feature distributions monitored

### Testing

```bash
# Make predictions
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "features": {"amount": 100, "age": 35},
    "model_name": "fraud-detection",
    "model_version": "v1.0"
  }'

# Check metrics
curl http://localhost:8000/metrics
```

---

## Exercise 03: Implement Structured Logging

**Duration**: 60 minutes
**Difficulty**: Basic

### Implementation

```python
# ml_service/logging_config.py
import structlog
import logging

def configure_logging():
    """Configure structured logging."""
    logging.basicConfig(
        format="%(message)s",
        level=logging.INFO
    )

    structlog.configure(
        processors=[
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.stdlib.add_log_level,
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer()
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )

logger = structlog.get_logger()
```

```python
# Usage in main.py
from .logging_config import logger

@app.post("/predict")
async def predict(request: PredictionRequest):
    logger.info(
        "prediction_started",
        model_name=request.model_name,
        model_version=request.model_version,
        feature_count=len(request.features)
    )

    try:
        # ... prediction logic ...

        logger.info(
            "prediction_completed",
            model_name=request.model_name,
            model_version=request.model_version,
            prediction_score=prediction_score,
            confidence=confidence,
            latency_ms=latency * 1000
        )

        return response

    except Exception as e:
        logger.error(
            "prediction_failed",
            model_name=request.model_name,
            error=str(e),
            exc_info=True
        )
        raise
```

### Success Criteria

- [ ] All logs in JSON format
- [ ] Timestamps in ISO format
- [ ] Structured context (model_name, version, etc.)
- [ ] Error logs include stack traces

---

## Exercise 04: Add Distributed Tracing

**Duration**: 90 minutes
**Difficulty**: Advanced

### Implementation

```python
# ml_service/tracing.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

def setup_tracing(app):
    """Configure OpenTelemetry tracing."""
    # Set up tracer provider
    trace.set_tracer_provider(TracerProvider())
    tracer_provider = trace.get_tracer_provider()

    # Configure OTLP exporter (to Tempo)
    otlp_exporter = OTLPSpanExporter(
        endpoint="tempo:4317",
        insecure=True
    )

    # Add span processor
    span_processor = BatchSpanProcessor(otlp_exporter)
    tracer_provider.add_span_processor(span_processor)

    # Instrument FastAPI
    FastAPIInstrumentor.instrument_app(app)

    return trace.get_tracer(__name__)
```

```python
# Add to main.py
from .tracing import setup_tracing

tracer = setup_tracing(app)

@app.post("/predict")
async def predict(request: PredictionRequest):
    with tracer.start_as_current_span("predict") as span:
        span.set_attribute("model.name", request.model_name)
        span.set_attribute("model.version", request.model_version)

        # Feature fetching span
        with tracer.start_as_current_span("fetch_features"):
            features = request.features

        # Model inference span
        with tracer.start_as_current_span("model_inference") as inference_span:
            prediction_score = float(np.random.random())
            inference_span.set_attribute("prediction.score", prediction_score)

        return response
```

### Success Criteria

- [ ] Traces exported to Tempo
- [ ] Spans show request flow
- [ ] Attributes attached to spans
- [ ] Traces visible in Grafana

---

## Exercise 05: Build Data Drift Detection

**Duration**: 120 minutes
**Difficulty**: Advanced

### Implementation

```python
# drift_detection.py
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset
import pandas as pd
from typing import Optional

class DriftDetector:
    def __init__(self, reference_data: pd.DataFrame):
        """Initialize with reference data (training set)."""
        self.reference_data = reference_data
        self.drift_threshold = 0.1

    def detect_drift(
        self,
        current_data: pd.DataFrame
    ) -> dict:
        """Detect drift between reference and current data."""

        report = Report(metrics=[DataDriftPreset()])

        report.run(
            reference_data=self.reference_data,
            current_data=current_data
        )

        result = report.as_dict()

        drift_detected = result['metrics'][0]['result']['dataset_drift']
        drift_score = result['metrics'][0]['result']['drift_share']

        drifted_features = [
            feature
            for feature, metrics in result['metrics'][0]['result']['drift_by_columns'].items()
            if metrics['drift_detected']
        ]

        return {
            "drift_detected": drift_detected,
            "drift_score": drift_score,
            "drifted_features": drifted_features,
            "total_features": len(current_data.columns),
            "report": result
        }

    def monitor_continuous(
        self,
        features: dict,
        window_size: int = 1000
    ):
        """Add data point and check for drift periodically."""
        # Implementation for streaming drift detection
        pass
```

### Success Criteria

- [ ] Drift detection runs on new data
- [ ] Statistical tests identify drifted features
- [ ] Drift metrics exported to Prometheus
- [ ] Alerts triggered on drift

---

## Exercise 06: Create Alerting Rules

**Duration**: 45 minutes
**Difficulty**: Intermediate

### Implementation

```yaml
# alert-rules.yml
groups:
  - name: ml_platform_alerts
    interval: 30s
    rules:
      - alert: HighPredictionLatency
        expr: |
          histogram_quantile(0.99,
            rate(ml_prediction_duration_seconds_bucket[5m])
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High prediction latency"
          description: "P99 latency is {{ $value }}s"

      - alert: HighErrorRate
        expr: |
          rate(ml_predictions_total{status="error"}[5m]) /
          rate(ml_predictions_total[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High prediction error rate"
          description: "Error rate: {{ $value | humanizePercentage }}"
```

### Success Criteria

- [ ] Alert rules configured in Prometheus
- [ ] Alerts fire when thresholds exceeded
- [ ] Notifications sent to PagerDuty/Slack
- [ ] Runbooks linked in annotations

---

## Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Evidently AI Documentation](https://docs.evidentlyai.com/)

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025
