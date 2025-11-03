# Module 08: Observability & Monitoring

Build comprehensive observability into your ML platform to detect issues before they impact users. Learn metrics, logs, traces, and ML-specific monitoring patterns.

**Duration**: 8-9 hours (lecture + exercises)
**Difficulty**: Intermediate to Advanced
**Prerequisites**: Modules 01-07 completed

---

## Learning Objectives

By the end of this module, you will be able to:

- [ ] Implement the three pillars of observability (metrics, logs, traces)
- [ ] Deploy and configure Prometheus for metrics collection
- [ ] Build Grafana dashboards for ML platform monitoring
- [ ] Implement structured logging with contextual data
- [ ] Set up distributed tracing with OpenTelemetry
- [ ] Detect data drift and model degradation
- [ ] Create actionable alerts with AlertManager
- [ ] Monitor infrastructure (GPU utilization, memory, CPU)

---

## Why Observability Matters

**The Problem**: ML systems fail silently in production.
- Models degrade gradually as data distributions shift
- Infrastructure issues (GPU OOM, network latency) cause intermittent failures
- Without observability, you're blind to what's happening in production
- Debugging production issues without logs/traces takes hours or days

**The Solution**: Comprehensive observability catches issues before users notice.
- **Uber**: Detected 15% accuracy drop in ETA model within 2 hours using drift monitoring
- **Netflix**: Reduced mean time to resolution (MTTR) from 4 hours to 15 minutes with distributed tracing
- **Airbnb**: Prevented $2M in revenue loss by detecting feature pipeline failures early
- **DoorDash**: Identified GPU memory leak causing 30% of prediction requests to fail

---

## Module Structure

### 📚 Lecture Notes (90 minutes)

#### [01: Observability & Monitoring](./lecture-notes/01-observability-monitoring.md)

**Topics Covered**:

1. **Three Pillars of Observability**
   - Metrics: Time-series data (counters, gauges, histograms)
   - Logs: Structured event records with context
   - Traces: Request flow through distributed systems
   - Correlation between metrics, logs, and traces

2. **Prometheus Metrics**
   - Counter: Monotonically increasing values
   - Gauge: Values that go up and down
   - Histogram: Distribution with buckets
   - Summary: Quantiles (p50, p95, p99)
   - PromQL query language

3. **ML-Specific Monitoring**
   - Prediction latency and throughput
   - Model confidence scores
   - Feature distributions
   - Data drift detection (covariate shift, concept drift)
   - Prediction distribution monitoring
   - A/B test metrics

4. **Structured Logging**
   - JSON-formatted logs with context
   - Correlation IDs for request tracing
   - Log levels (DEBUG, INFO, WARNING, ERROR, CRITICAL)
   - structlog for Python
   - Centralized log aggregation with Loki

5. **Distributed Tracing**
   - Spans and traces
   - OpenTelemetry standard
   - Automatic instrumentation for FastAPI/Flask
   - Manual instrumentation for custom code
   - Trace visualization with Grafana Tempo

6. **Data Drift Detection**
   - Statistical tests (KS, Chi-Squared, Wasserstein)
   - Evidently AI for drift reports
   - Feature distribution monitoring
   - Automated retraining triggers

7. **Infrastructure Monitoring**
   - GPU utilization and memory
   - CPU, memory, disk I/O
   - Network latency and bandwidth
   - Kubernetes metrics (pod restarts, resource limits)

8. **Alerting Strategies**
   - SLIs, SLOs, SLAs
   - Alert rules with PromQL
   - Runbooks and incident response
   - PagerDuty/Slack integration
   - Avoiding alert fatigue

**Key Code Example** (Prometheus Metrics):
```python
from prometheus_client import Counter, Histogram, Gauge

predictions_total = Counter(
    'ml_predictions_total',
    'Total number of predictions',
    ['model_name', 'model_version', 'status']
)

prediction_latency = Histogram(
    'ml_prediction_duration_seconds',
    'Time spent making predictions',
    ['model_name'],
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

gpu_utilization = Gauge(
    'ml_gpu_utilization_percent',
    'Current GPU utilization percentage',
    ['gpu_id']
)

# Usage
predictions_total.labels(
    model_name='fraud-detection',
    model_version='v2.1',
    status='success'
).inc()

with prediction_latency.labels(model_name='fraud-detection').time():
    prediction = model.predict(features)
```

**Key Code Example** (Structured Logging):
```python
import structlog

logger = structlog.get_logger()

logger.info(
    "prediction_completed",
    request_id="req-abc123",
    user_id="user-123",
    model_name="fraud-detection",
    model_version="v2.1",
    prediction_score=0.87,
    latency_ms=45,
    features_count=50
)
```

**Key Code Example** (Distributed Tracing):
```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

@app.post("/predict")
async def predict(request: PredictionRequest):
    with tracer.start_as_current_span("predict_request") as span:
        span.set_attribute("model.name", "fraud-detection")

        with tracer.start_as_current_span("fetch_features"):
            features = await feature_store.get_features(request.user_id)

        with tracer.start_as_current_span("model_inference"):
            prediction = model.predict(features)

        return {"prediction": prediction}
```

---

### 🛠️ Hands-On Exercises (8 hours)

Complete 6 comprehensive exercises building a production observability stack:

#### Exercise 01: Deploy Prometheus & Grafana Stack (90 min, Intermediate)
- Set up LGTM stack (Loki, Grafana, Tempo, Mimir/Prometheus)
- Configure Prometheus scraping
- Create Grafana datasources
- Build basic dashboard with panels

**Success Criteria**: Running stack with Grafana displaying Prometheus metrics

---

#### Exercise 02: Instrument ML Service with Metrics (75 min, Intermediate)
- Add prometheus_client to FastAPI service
- Implement Counter, Histogram, Gauge metrics
- Expose /metrics endpoint
- Create Grafana dashboard for prediction metrics

**Success Criteria**: Dashboard showing prediction rate, latency, errors

---

#### Exercise 03: Implement Structured Logging (60 min, Basic)
- Configure structlog for JSON output
- Add correlation IDs to all logs
- Ship logs to Loki with Promtail
- Query logs in Grafana with LogQL

**Success Criteria**: Logs viewable in Grafana with searchable fields

---

#### Exercise 04: Add Distributed Tracing (90 min, Advanced)
- Integrate OpenTelemetry with FastAPI
- Auto-instrument HTTP requests
- Add custom spans for business logic
- Visualize traces in Grafana Tempo

**Success Criteria**: End-to-end request traces visible in Tempo

---

#### Exercise 05: Detect Data Drift (120 min, Advanced)
- Set up Evidently AI for drift detection
- Monitor feature distributions
- Run statistical tests (KS, Wasserstein)
- Generate drift reports and alerts

**Success Criteria**: Automated drift detection with alert on threshold

---

#### Exercise 06: Create Alerting Rules (45 min, Intermediate)
- Write AlertManager rules with PromQL
- Configure severity levels and labels
- Set up notification channels (Slack, email)
- Test alert firing and resolution

**Success Criteria**: Alerts fire when thresholds breached

---

## Tools & Technologies

**Required**:
- Python 3.9+
- Docker & Docker Compose
- Prometheus 2.48.0
- Grafana 10.2.2
- Loki 2.9.3 (log aggregation)
- Tempo 2.3.1 (distributed tracing)
- AlertManager 0.26.0
- prometheus_client 0.19.0
- OpenTelemetry 1.21.0
- Evidently AI 0.4.0
- structlog 24.1.0

**Python Packages**:
```bash
pip install prometheus-client==0.19.0 \
    opentelemetry-api==1.21.0 \
    opentelemetry-sdk==1.21.0 \
    opentelemetry-instrumentation-fastapi==0.42b0 \
    opentelemetry-exporter-otlp==1.21.0 \
    evidently==0.4.0 \
    structlog==24.1.0 \
    fastapi==0.104.0 \
    uvicorn==0.24.0
```

**Docker Stack**:
```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:10.2.2
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true

  loki:
    image: grafana/loki:2.9.3
    ports:
      - "3100:3100"

  tempo:
    image: grafana/tempo:2.3.1
    ports:
      - "3200:3200"
      - "4317:4317"  # OTLP gRPC
```

---

## Prerequisites

Before starting this module, ensure you have:

- [x] Completed Module 07 (Developer Experience & SDKs)
- [x] Strong Python programming skills
- [x] Understanding of REST APIs
- [x] Basic knowledge of Docker and Docker Compose
- [x] Familiarity with time-series data concepts
- [x] Understanding of statistical concepts (mean, percentiles, distributions)

**Recommended Background**:
- Experience with Prometheus and Grafana (helpful but not required)
- Understanding of distributed systems
- Familiarity with JSON and YAML formats
- Basic knowledge of SQL-like query languages (PromQL, LogQL)

---

## Time Breakdown

| Component | Duration | Format |
|-----------|----------|--------|
| Lecture notes | 90 min | Reading + code review |
| Exercise 01 (Stack Setup) | 90 min | Hands-on deployment |
| Exercise 02 (Metrics) | 75 min | Hands-on coding |
| Exercise 03 (Logging) | 60 min | Hands-on coding |
| Exercise 04 (Tracing) | 90 min | Hands-on coding |
| Exercise 05 (Drift Detection) | 120 min | Hands-on coding |
| Exercise 06 (Alerting) | 45 min | Configuration |
| **Total** | **~9.5 hours** | Mixed |

**Recommended Schedule**:
- **Day 1**: Lecture + Exercise 01-02 (4 hours)
- **Day 2**: Exercise 03-04 (2.5 hours)
- **Day 3**: Exercise 05-06 (3 hours)

---

## Success Criteria

You have successfully completed this module when you can:

1. **Metrics** ✅
   - Implement Counter, Histogram, Gauge metrics
   - Expose /metrics endpoint in FastAPI
   - Write PromQL queries for aggregation
   - Build Grafana dashboards with panels

2. **Logging** ✅
   - Configure structured JSON logging with structlog
   - Add correlation IDs to all log entries
   - Ship logs to Loki with Promtail
   - Query logs with LogQL in Grafana

3. **Tracing** ✅
   - Integrate OpenTelemetry with FastAPI
   - Auto-instrument HTTP requests
   - Add custom spans for business logic
   - Correlate traces with logs and metrics

4. **ML Monitoring** ✅
   - Track prediction latency, throughput, errors
   - Monitor model confidence distributions
   - Detect data drift with Evidently AI
   - Alert on model performance degradation

5. **Infrastructure** ✅
   - Monitor GPU utilization and memory
   - Track CPU, memory, disk I/O
   - Set up Kubernetes metrics collection
   - Create dashboards for resource usage

6. **Alerting** ✅
   - Write AlertManager rules with PromQL
   - Configure notification channels
   - Set appropriate thresholds (avoid alert fatigue)
   - Include runbooks in alert annotations

---

## Real-World Applications

### Uber: ML Platform Observability

**Challenge**:
- 10,000+ ML models in production
- Silent model degradation costing millions
- No visibility into prediction quality
- Infrastructure issues causing service disruptions

**Solution**:
- Comprehensive metrics for all models (latency, throughput, errors)
- Automated drift detection with statistical tests
- Real-time dashboards showing model health
- Distributed tracing for debugging latency issues

**Impact**:
- Detected 15% accuracy drop in ETA model within 2 hours (previously would take days)
- Reduced MTTR from 6 hours to 30 minutes
- Prevented estimated $5M in revenue loss from degraded models
- 90% of issues caught before user impact

**Key Techniques**:
- Prediction distribution monitoring (detect shifts in output)
- Feature importance tracking (identify which features changed)
- Canary deployments with automated rollback on metric degradation
- Custom PromQL queries for complex alerting conditions

---

### Netflix: Distributed Tracing at Scale

**Challenge**:
- Microservices architecture with 1000+ services
- Debugging latency issues took days
- No visibility into request flow
- Intermittent failures hard to reproduce

**Solution**:
- OpenTelemetry instrumentation across all services
- Grafana Tempo for trace storage and visualization
- Automatic correlation of traces with logs
- Sampling strategies to reduce overhead (1% trace sampling)

**Impact**:
- MTTR reduced from 4 hours to 15 minutes
- Identified network bottlenecks causing 2s latency spikes
- Reduced infrastructure costs by 20% after identifying inefficient call patterns
- Engineers spend 70% less time debugging

**Key Techniques**:
- Context propagation via HTTP headers (trace IDs, span IDs)
- Tail-based sampling (keep only interesting traces)
- Trace aggregation and analytics
- Integration with incident management (PagerDuty)

---

### Airbnb: Data Drift Detection

**Challenge**:
- COVID-19 caused massive distribution shifts
- Models trained on pre-COVID data failing
- No automated detection of model degradation
- Manual model quality checks unreliable

**Solution**:
- Evidently AI for automated drift detection
- Statistical tests on feature distributions (KS test)
- Real-time prediction distribution monitoring
- Automated retraining triggers on drift detection

**Impact**:
- Prevented $2M revenue loss by catching feature pipeline failure
- Detected COVID-related drift within 6 hours
- Reduced false positives in fraud detection by 40%
- Automated retraining reduced manual effort by 80%

**Key Techniques**:
- Kolmogorov-Smirnov test for continuous features
- Chi-squared test for categorical features
- Wasserstein distance for distribution comparison
- Reference data windows (compare current vs. last 7 days)

---

### DoorDash: GPU Monitoring

**Challenge**:
- GPU out-of-memory errors causing 30% prediction failures
- No visibility into GPU utilization
- Manual investigation took hours
- Revenue loss from failed orders

**Solution**:
- NVIDIA DCGM exporter for Prometheus
- Grafana dashboards for GPU metrics
- Alerts on high memory usage (>90%)
- Automatic pod restarts on OOM

**Impact**:
- Reduced prediction failures from 30% to 0.5%
- Identified memory leak in feature preprocessing
- Optimized batch sizes saving $50K/month in GPU costs
- Prevented estimated $500K revenue loss

**Key Techniques**:
- DCGM exporter for GPU metrics (utilization, memory, temperature)
- Correlation of GPU metrics with prediction latency
- Alerts on GPU memory trends (predictive, not reactive)
- Automatic model offloading on high memory pressure

---

## Common Pitfalls

### 1. Too Many Metrics (Cardinality Explosion)

**Problem**: Creating metrics with high-cardinality labels (user IDs, timestamps)

**Bad Example**:
```python
# DON'T: user_id has millions of values
predictions_total = Counter(
    'ml_predictions_total',
    'Total predictions',
    ['model_name', 'user_id']  # ⚠️ High cardinality!
)
```

**Solution**: Use low-cardinality labels only
```python
# DO: Keep cardinality low
predictions_total = Counter(
    'ml_predictions_total',
    'Total predictions',
    ['model_name', 'model_version', 'status']  # ✅ Low cardinality
)
```

---

### 2. Logging Sensitive Data

**Problem**: Accidentally logging PII (names, emails, SSNs)

**Bad Example**:
```python
# DON'T: Logs contain PII
logger.info(
    "prediction_completed",
    user_email="john@example.com",  # ⚠️ PII!
    ssn="123-45-6789"  # ⚠️ PII!
)
```

**Solution**: Redact or hash sensitive fields
```python
# DO: Hash or exclude PII
logger.info(
    "prediction_completed",
    user_id_hash=hashlib.sha256(user_id.encode()).hexdigest(),
    request_id="req-abc123"
)
```

---

### 3. Alert Fatigue

**Problem**: Too many alerts firing, team ignores them

**Solution**: Tune thresholds and use multi-window alerts
```yaml
# Good: Alert only if sustained high latency
- alert: HighPredictionLatency
  expr: |
    histogram_quantile(0.99,
      rate(ml_prediction_duration_seconds_bucket[5m])
    ) > 1.0
  for: 5m  # Must be true for 5 minutes
  labels:
    severity: warning
```

---

### 4. Missing Trace Context

**Problem**: Traces not connected across service boundaries

**Solution**: Propagate trace context via HTTP headers
```python
from opentelemetry.propagate import inject

headers = {}
inject(headers)  # Adds traceparent header

response = requests.post(
    "http://feature-service/features",
    headers=headers
)
```

---

### 5. Ignoring Drift Until It's Too Late

**Problem**: Waiting for user complaints before checking model quality

**Solution**: Automated drift detection with scheduled jobs
```python
# Run drift detection every hour
@celery.task(cron="0 * * * *")
def check_data_drift():
    report = Report(metrics=[DataDriftPreset()])
    report.run(reference_data=ref_data, current_data=current_data)

    if report.as_dict()['metrics'][0]['result']['dataset_drift']:
        alert_ops_team()
```

---

## Assessment

### Knowledge Check (after lecture notes)

1. What are the three pillars of observability?
2. What's the difference between a Counter and a Gauge in Prometheus?
3. When would you use a Histogram vs. a Summary?
4. What is data drift and how do you detect it?
5. How do you correlate logs with traces using correlation IDs?
6. What is the Wasserstein distance and when is it useful?
7. What are SLIs, SLOs, and SLAs?
8. How do you avoid high cardinality in Prometheus labels?

### Practical Assessment (after exercises)

Build a complete observability stack that:
- [ ] Deploys LGTM stack (Loki, Grafana, Tempo, Mimir)
- [ ] Instruments ML service with Counter, Histogram, Gauge metrics
- [ ] Implements structured JSON logging with correlation IDs
- [ ] Adds distributed tracing with OpenTelemetry
- [ ] Detects data drift with Evidently AI
- [ ] Creates AlertManager rules for high latency and error rate
- [ ] Builds Grafana dashboards for all key metrics

**Acceptance Criteria**:
- Metrics visible in Prometheus at /metrics endpoint
- Grafana dashboard shows prediction rate, latency (p50/p95/p99), error rate
- Logs searchable in Grafana with LogQL queries
- Traces viewable in Tempo with spans for each service call
- Drift detection runs automatically and alerts on threshold breach
- Alerts fire when p99 latency > 1s for 5 minutes
- Dashboard includes GPU utilization and memory metrics

---

## Troubleshooting

### Prometheus Not Scraping Metrics

**Symptom**: `/metrics` endpoint works but no data in Prometheus

**Solutions**:
```yaml
# Check prometheus.yml scrape config
scrape_configs:
  - job_name: 'ml-service'
    static_configs:
      - targets: ['ml-service:8000']  # Correct hostname?

# Verify in Prometheus UI
# Status > Targets should show "UP"

# Check network connectivity
docker exec prometheus wget -O- http://ml-service:8000/metrics
```

---

### Loki Logs Not Showing

**Symptom**: Logs not appearing in Grafana

**Solutions**:
```bash
# Check Promtail is running
docker ps | grep promtail

# Verify Promtail config
# /etc/promtail/config.yml
clients:
  - url: http://loki:3100/loki/api/v1/push

# Check Loki logs
docker logs loki

# Test direct push
curl -X POST http://loki:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -d '{"streams":[{"stream":{"job":"test"},"values":[["1234567890000000000","test message"]]}]}'
```

---

### Traces Not Appearing in Tempo

**Symptom**: No traces visible in Grafana Tempo

**Solutions**:
```python
# Verify exporter configuration
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

exporter = OTLPSpanExporter(
    endpoint="tempo:4317",  # Correct endpoint?
    insecure=True
)

# Check Tempo receiver config
# /etc/tempo.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

# Test with telemetrygen
docker run --rm --network host \
  otel/telemetrygen:latest traces \
  --otlp-endpoint localhost:4317 \
  --otlp-insecure
```

---

### High Cardinality Issues

**Symptom**: Prometheus using excessive memory, slow queries

**Solutions**:
```bash
# Check cardinality
curl http://localhost:9090/api/v1/status/tsdb

# Find high-cardinality metrics
curl http://localhost:9090/api/v1/label/__name__/values

# Reduce labels
# Before: labels=['model', 'user_id', 'timestamp']
# After: labels=['model', 'status']
```

---

## Next Steps

After completing this module, you're ready for:

- **Module 09: Security & Governance** - Implement RBAC, audit logs, compliance
- **Module 10: Advanced Topics** (if available) - Cost optimization, multi-region, disaster recovery

**Recommended Path**: Proceed to Module 09 to learn how to secure your ML platform with authentication, authorization, audit logging, and compliance controls.

---

## Additional Resources

### Official Documentation
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/)
- [Tempo Documentation](https://grafana.com/docs/tempo/)
- [OpenTelemetry Python](https://opentelemetry.io/docs/instrumentation/python/)
- [Evidently AI Documentation](https://docs.evidentlyai.com/)
- [structlog Documentation](https://www.structlog.org/)

### Tutorials & Guides
- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)
- [OpenTelemetry Instrumentation](https://opentelemetry.io/docs/instrumentation/)
- [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)

### Real-World Examples
- [Uber's Observability Platform](https://www.uber.com/blog/observability-at-scale/)
- [Netflix's Distributed Tracing](https://netflixtechblog.com/building-netflixs-distributed-tracing-infrastructure-bb856c319304)
- [Airbnb's ML Monitoring](https://medium.com/airbnb-engineering/monitoring-machine-learning-models-at-scale-8c7c8e6b3a3e)
- [DoorDash's ML Platform](https://doordash.engineering/2020/01/28/monitoring-machine-learning-models-in-production/)

### Video Tutorials
- [Prometheus Crash Course](https://www.youtube.com/results?search_query=prometheus+tutorial)
- [Grafana Fundamentals](https://www.youtube.com/results?search_query=grafana+tutorial)
- [OpenTelemetry Overview](https://www.youtube.com/results?search_query=opentelemetry+tutorial)

### Books
- **"Observability Engineering"** by Charity Majors, Liz Fong-Jones, George Miranda
- **"Prometheus: Up & Running"** by Brian Brazil
- **"Distributed Tracing in Practice"** by Austin Parker et al.

---

## Feedback & Support

**Questions?** Open an issue in the repository with the `module-08` tag.

**Found a bug in the code?** Submit a PR with the fix.

**Want more exercises?** Check the `/exercises/bonus/` directory for advanced challenges.

---

**Status**: ✅ Complete | **Last Updated**: November 2, 2025 | **Version**: 1.0.0
