# Lecture 03: Streaming Features

## The shape of a streaming feature

```
user_id=42, event_ts=2026-05-23T10:00:00Z, event_type=click, item_id=xyz
  → Flink/Spark Streaming
  → aggregate: 5-minute tumbling window per user_id
  → emit: user_id=42, clicks_5m=8, last_event_ts=...
  → write to: Redis (online) + Kafka (event stream for batch consumers)
```

## Design choices

### Tumbling vs sliding windows

- **Tumbling** (5m, non-overlapping): simpler; per-window aggregate is independent
- **Sliding** (5m, every 1m): more responsive; ~5× compute

Pick tumbling unless responsiveness matters.

### Watermarks + lateness

Events arrive late. The window for `T-5m → T` may receive events with
`event_ts < T-5m`. Decisions:
- Drop late events → simpler but biased
- Configurable lateness window (e.g., 10 min) → accepts late events, may revise outputs
- Reconcile via batch later → most accurate; most operational complexity

For ML features, accept up to ~10 min lateness for low-volume events; rely on
batch reconciliation for high-stakes corrections.

### Exactly-once vs at-least-once

- Exactly-once: Flink checkpoints + idempotent sinks; complex but accurate
- At-least-once: simpler; OK if double-counted features are tolerable

Most feature workflows tolerate at-least-once. Verify on a per-feature basis.

## Lambda vs Kappa

- **Lambda**: streaming (recent) + batch (historical) + reconciliation
- **Kappa**: streaming-only; backfill via Kafka log replay

Lambda is the practical default. Kappa is elegant but requires Kafka retention
covering your full feature history (often years) and very mature streaming
engineering.

## Operational notes

- Per-feature freshness gauge: `feature_freshness_seconds{feature="clicks_5m"}`
- Alert when freshness > 5× the window size (signals stalled stream)
- Disable streaming feature in serving when freshness exceeds tolerance; fall
  back to batch value

## Companion

[engineer-solutions/mod-105 ex-11 (lambda-architecture)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-105-data-pipelines/exercise-11-lambda-architecture) — working PyFlink + Spark + serving stack.
