# Exercise 05: Feature Store Operations

Build the monitoring + alerting that lets you run the feature store with
confidence.

Required dashboards + alerts:
1. Per-FeatureView freshness gauge + alert when staleness > 5× expected
2. Materialization SLA: alert when daily materialization fails or runs > 4h
3. Online store capacity: Redis memory + eviction rate
4. Per-feature PSI (drift) computed daily; alert > 0.25
5. Per-feature read QPS + p99 latency

Deliverable:
- `prometheus_exporter.py` exposing the gauges
- `alerts.yml` rules
- `dashboard.json` Grafana
- `RUNBOOK.md` covering top 5 alerts
