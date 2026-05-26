# Project 03 — Requirements

## Functional requirements

### F1 — Pipeline definitions

- [ ] YAML schema validated at registration.
- [ ] Immutable versions: each `Pipeline` version cannot be
      modified; new versions supersede.
- [ ] Steps form a DAG; the parser rejects cycles.
- [ ] Schedule + event triggers both supported.

### F2 — Scheduler

- [ ] Cron schedules trigger new runs at the configured time.
- [ ] Event triggers (HTTP webhook) trigger matching
      pipelines.
- [ ] Late schedule recovery: if the scheduler was down at
      the cron time, runs catch up within 1h of scheduler
      restart (configurable).

### F3 — Executor

- [ ] Step pods launched with the configured image,
      resources, env.
- [ ] Status reflected in DB within 5s of pod state change.
- [ ] Per-step outputs captured (a JSON file written to a
      known path inside the pod, retrieved into
      `step_states.outputs`).
- [ ] Retries classified correctly (transient vs.
      permanent).

### F4 — Gates

- [ ] Programmatic gates: `condition: "expr"` evaluated
      against prior step outputs.
- [ ] Human approval gates: approval recorded via
      `POST /v1/runs/{id}/approve` with approver identity +
      audit-chain entry.
- [ ] Gate failures halt the run with a clear reason in the
      run status.

### F5 — Re-run from step

- [ ] CLI: `smartrecs workflow rerun <run-id> --from <step>`.
- [ ] Successful steps before `<step>` are not re-executed.
- [ ] Outputs from the previous run are available to
      downstream steps.

### F6 — Per-tenant isolation

- [ ] Pipelines exist in a tenant namespace.
- [ ] Step pods run in the tenant namespace.
- [ ] Tenant A cannot trigger pipelines in tenant B's
      namespace.

### F7 — Audit chain

- [ ] Every run transition emits an audit event.
- [ ] Every gate evaluation emits an audit event.
- [ ] Every human approval emits a signed event identifying
      the approver.

### F8 — Observability

- [ ] Prometheus metrics per §5 of `architecture.md`.
- [ ] Per-step traces wired via OpenTelemetry.
- [ ] Grafana dashboard JSON included.

### F9 — Notifications

- [ ] On-failure: PagerDuty webhook (or Slack for non-critical
      runs).
- [ ] On-success: Slack channel notification.
- [ ] Notification destinations configurable per pipeline.

## Non-functional requirements

- [ ] `make up` brings up the full system locally.
- [ ] Unit tests on DAG parser, state machine, retry
      classifier.
- [ ] Integration test: full pipeline runs end-to-end.
- [ ] Crash recovery test: kill the executor mid-run; on
      restart, in-flight runs continue correctly.
- [ ] All container images non-root + Cosign-signed.

## Acceptance demo

1. Register a 5-step pipeline (`ingest → features → train →
   evaluate → promote`).
2. Trigger via schedule + via event; both produce runs.
3. Verify the DAG respects dependencies (train doesn't start
   until features finishes).
4. Force a transient failure in `features` (e.g., simulate
   OOM); verify retry succeeds.
5. Force a permanent failure in `train` (e.g., bad config);
   verify the run fails fast without retries.
6. Force an evaluation gate failure (`accuracy < 0.85`); verify
   `promote` does not run.
7. Re-run from `evaluate` (with a passing gate); verify only
   `evaluate` + `promote` execute.
8. Kill the executor mid-run; on restart, verify state
   recovers.
9. Audit-chain `verify`: clean.
10. Cross-tenant trigger attempt: 403.

## Submission

`deliverables/` contains:
- `code/`
- `docs/`
- `tests/`
- `screenshots/` — Grafana dashboard, audit-chain proof, gate-
  failure example, re-run-from-step demonstration.
- `SUBMISSION.md`.
