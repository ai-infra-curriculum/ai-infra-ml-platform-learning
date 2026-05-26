# Lecture 02: SLOs + Burn-Rate Alerts for ML Services

## SLOs for ML services

| SLI | Typical target |
|---|---|
| Availability (non-5xx) | 99.5% over 30 days |
| Latency (p95 < N ms) | 95% over 30 days |
| Model accuracy (rolling 7d) | within 5pp of baseline |
| Drift below threshold | PSI < 0.25 on key features |

## Multi-window burn-rate alerts (Google SRE pattern)

Page when 5m + 1h windows both burn at > 14.4×:
```yaml
alert: BurnFast
expr: |
  ((1 - sli:availability_rate1h)  / 0.005 > 14.4)
  and
  ((1 - sli:availability_rate5m)  / 0.005 > 14.4)
```

Ticket when 30m + 6h burn at > 6×.

## Error budget policies

- > 50% budget remaining: free to deploy
- 20-50%: deploy with caution
- < 20%: feature freeze; reliability work only

Make the policy explicit so teams self-regulate.

## Companion

[engineer-solutions/mod-108 ex-08 (slo-and-error-budgets)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-108-monitoring-observability/exercise-08-slo-and-error-budgets) — full Sloth-format SLO spec.
