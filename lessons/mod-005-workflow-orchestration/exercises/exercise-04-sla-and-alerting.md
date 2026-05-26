# Exercise 04: SLA + Alerting

Configure SLAs + Slack alerts on a multi-stage training DAG. Cover the cases:
- DAG misses 4-hour SLA → page on-call
- Task fails 3× in a row → Slack ticket
- Task pending > 1h → Slack warning
- Scheduler heartbeat lost → page on-call

Demonstrate each by injecting a fault.
