# Exercise 05: Cost Attribution Pipeline

## Objective

Build a daily rollup that attributes cluster cost to teams.

## Requirements

1. Cron job collects:
   - kube-state-metrics: per-namespace pod-hours by resource
   - Per-node cost (hardcoded from instance pricing or pulled from cloud cost API)
2. Computes per-team daily cost
3. Writes to Postgres table with schema: `(date, team, cpu_hours, gpu_hours, memory_gb_hours, cost_usd)`
4. Grafana panel: per-team cost over last 30 days
5. Per-team monthly budget defined in YAML; alert when projected to exceed

## Companion

[engineer-solutions/mod-106 ex-12 (cost-attribution)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops/exercise-12-model-cost-attribution) has the per-model variant.
