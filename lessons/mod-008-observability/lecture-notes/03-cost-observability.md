# Lecture 03: Cost Observability

Every model should be cost-tagged. Every dashboard should include cost.

## Required labels
- `team` + `cost_center` + `model_name` + `environment`

Enforced by Kyverno (see mod-003).

## Cost rollup

Daily:
- pod-hours by `(team, model_name)` from kube-state-metrics
- × per-pod cost (from cloud cost API)
- → per-model daily cost in Postgres
- → Grafana panel + monthly invoice

## Alerts

- Per-model budget exceeded (e.g., 95% of monthly cap)
- Per-team budget exceeded
- Per-cluster cost spike: > 25% week-over-week

## Common failures

- Missing labels → "unallocated" bucket grows
- Cluster overhead unallocated → platform team carries it
- Spot interruption rerun costs not attributed

## Companion

- engineer-solutions/mod-106 ex-12 (per-model cost attribution)
- engineer-solutions/mod-104 ex-15 (cluster cost optimization)
