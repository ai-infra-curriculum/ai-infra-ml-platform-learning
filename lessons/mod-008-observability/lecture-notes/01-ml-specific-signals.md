# Lecture 01: ML-Specific Signals

Generic HTTP observability (RPS, latency, error rate) doesn't answer the
questions ML teams actually have:
- Is this model performing as well as last week?
- Is feature X distributionally stable?
- Which model costs most?

## The ML-specific signals

| Signal | What it answers | Source |
|---|---|---|
| Drift (PSI per feature) | Did inputs shift? | computed from feature store + reference |
| Per-class prediction distribution | Is model behavior shifting? | inference logs |
| Slice metrics | How does the model do per-segment? | inference logs + ground truth |
| TTFT (LLM) | First-token responsiveness | API instrumentation |
| Per-model cost | $/model/day | resource labels + cost rollup |
| Inference log freshness | Is the inference stream healthy? | log pipeline metrics |

## Dashboards

A model owner's dashboard should answer (top to bottom):
1. Is the model serving (RPS, error rate)?
2. Is it serving well (latency p95/p99)?
3. Is it serving correctly (slice accuracy, drift, prediction distribution)?
4. What's it costing?

Each of these maps to specific alerts.

## Companion

[engineer-solutions/mod-108 (all 10 exercises)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-108-monitoring-observability).
