# Exercise 03: Drift Monitoring End-to-End

Wire a daily drift job that:
- Reads inference logs from S3 / warehouse
- Computes PSI per feature vs reference window
- Exports as Prometheus gauges
- Alerts when PSI > 0.25 for any monitored feature

Companion: engineer-solutions/mod-108 (monitoring module) + mlops-solutions modules/03-model-monitoring.
