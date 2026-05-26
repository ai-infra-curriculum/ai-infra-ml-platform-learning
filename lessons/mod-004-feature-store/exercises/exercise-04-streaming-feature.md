# Exercise 04: Streaming Feature

Build a streaming feature using PyFlink (or Spark Structured Streaming):
- Source: Kafka topic `events`
- Window: 5-minute tumbling, keyed by user_id
- Aggregate: count of clicks per user per 5-min window
- Sink: Redis (online) + Kafka topic `features.user_clicks_5m` (batch consumer)

Operational requirements:
- Watermark with 30-second out-of-orderness
- Checkpointing every 60 seconds
- Per-key state TTL: 7 days

Deliverable: `feature_job.py` + a load-test script that pumps events and
verifies the Redis values converge to expected counts.

Companion: engineer-solutions/mod-105 ex-11 (lambda architecture).
