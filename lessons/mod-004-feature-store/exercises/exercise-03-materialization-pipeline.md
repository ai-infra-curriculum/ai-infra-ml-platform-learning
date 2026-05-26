# Exercise 03: Materialization Pipeline with Backfill

Build an Airflow DAG that materializes Feast features daily and supports a
back-fill operation for arbitrary date ranges.

Requirements:
- Daily incremental: `fs.materialize_incremental(end_date=now)`
- Backfill: parameterized DAG run with custom start/end dates
- Safety: concurrent backfills are not allowed (use Airflow `max_active_runs=1`)
- Metrics: emit `materialization_duration_seconds`, `rows_written` per FeatureView

Deliverable: `materialize_dag.py` + smoke test of a 30-day backfill.
