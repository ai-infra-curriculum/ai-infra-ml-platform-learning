# Lecture 03: Cost Attribution

## The default state

Without attribution, every cost discussion is "the cloud bill went up; we
don't know why". The platform team becomes a slow lookup service for finance.

## Three building blocks

### 1. Labels everywhere

Enforce via Kyverno or OPA Gatekeeper:
- `team` (required)
- `cost_center` (required)
- `model_name` (required on inference + training resources)
- `environment` (required)

### 2. Per-namespace usage metrics

`kube-state-metrics` exposes:
- `kube_pod_container_resource_requests{namespace, container, resource}`
- `kube_pod_container_resource_limits{...}`

Combine with cloud cost APIs (AWS Cost Explorer, GCP BigQuery export) to
compute $/team/day.

### 3. Daily rollup + dashboard

Cron job runs at 06:00, fetches the prior day's resource usage + cloud cost,
produces a per-team breakdown, writes to a Postgres table. Grafana panel
shows trends.

Sample rollup SQL:
```sql
SELECT date, team, sum(cpu_hours) AS cpu_hours, sum(gpu_hours) AS gpu_hours,
       sum(memory_gb_hours) AS mem_gb_hours, sum(cost_usd) AS cost_usd
FROM daily_usage
WHERE date >= current_date - INTERVAL '30 days'
GROUP BY date, team;
```

## Showback vs chargeback

- **Showback**: teams see their cost; no real money changes hands.
- **Chargeback**: teams' budgets actually get debited.

Start with showback for the first 1-2 quarters. Move to chargeback once the
attribution is trusted (low dispute rate, accurate to within ~5%).

## Common failures

- **Missing labels**: 30% of resources are unlabeled; cost is "unallocated"
- **Disputed numbers**: usage measured one way, billed another way
- **Lag**: rollup runs daily but bill arrives monthly; lookback is painful
- **Overhead not allocated**: control plane, monitoring, ingress aren't billed to anyone

Address these with: enforced labels (Kyverno), one source of truth, daily
visibility, and an explicit "platform overhead" line in everyone's bill.
