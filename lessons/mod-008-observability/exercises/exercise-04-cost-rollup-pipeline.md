# Exercise 04: Cost Rollup Pipeline

Build a daily cron that:
- Pulls per-pod resource usage by label
- Joins with cloud cost API for $/pod-hour
- Writes `(date, team, model_name, cost_usd)` to Postgres
- Powers a Grafana panel

Companion: engineer-solutions/mod-106 ex-12.
