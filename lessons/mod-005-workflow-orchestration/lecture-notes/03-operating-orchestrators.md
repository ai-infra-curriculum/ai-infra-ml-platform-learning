# Lecture 03: Operating Orchestrators

## Capacity

Two levers in Airflow:
- **Pool capacity**: per-pool concurrent task limit
- **Parallelism**: cluster-wide concurrent task limit

ML common pitfalls:
- Pool capacity = N, but GPU quota = M < N → tasks pending, scheduler thrashes
- Cron-triggered DAGs scheduled simultaneously → thundering herd at 02:00

## Failure modes

| Failure | Detection | Recovery |
|---|---|---|
| Task fails repeatedly | retry exhausted; on_failure_callback | check task logs; fix code; rerun |
| DAG hangs | sla_miss_callback | check Airflow scheduler logs; look for dead worker |
| Backfill DDoS'd downstream | rate of API calls | rate-limit task pool; backfill with concurrency=1 |
| Pool exhausted | tasks Pending too long | resize pool; investigate slow task; preempt low-priority |

## Backfill discipline

Backfill is the most dangerous routine op. Required guardrails:
- Audit log of every backfill (who, what, when, range)
- Concurrency cap (don't backfill 365 days in parallel)
- Dry-run mode that prints the plan without executing
- Idempotency: re-running backfill produces same output
- Ability to backfill a single failed day, not start from scratch

## SLAs + alerting

- `sla` attribute on each DAG; `sla_miss_callback` posts to Slack
- DAG fail callback also posts to Slack
- Don't alert on every failure (retries are normal); alert on:
  - 3 consecutive failures
  - SLA missed
  - Task pending > 1h
  - Scheduler heartbeat lost

## Companion

[engineer-solutions/mod-105 ex-09 (backfill-strategies)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-105-data-pipelines/exercise-09-backfill-strategies) covers backfill safety in depth.
