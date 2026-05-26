# Project 03 — Architecture

## Component map

```
┌──────────────────────────────────────────────────────────────┐
│ USER LAYER                                                   │
│  • Pipeline YAML (versioned in Git)                          │
│  • CLI (smartrecs workflow create | run | rerun)             │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ ORCHESTRATION ENGINE                                          │
│  ┌───────────────┐  ┌────────────────┐  ┌────────────────┐  │
│  │ Scheduler     │→→│ DAG state mgr  │→→│ Executor       │  │
│  │ (cron+events) │  │ (state machine)│  │ (pod runner)   │  │
│  └───────────────┘  └────────────────┘  └────────────────┘  │
│         │                  │                    │            │
│         ▼                  ▼                    ▼            │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ Platform DB (PostgreSQL)                                ││
│  │  • pipelines (definitions, versions)                    ││
│  │  • runs (one per execution)                             ││
│  │  • step_states (per-step status)                        ││
│  │  • gates (gate evaluations + approvals)                 ││
│  └─────────────────────────────────────────────────────────┘│
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼ creates step pods
              ┌──────────────────┐
              │ Kubernetes API   │
              └──────────────────┘
```

## Key design decisions

### 1. State machine, not a script

Each pipeline run is a state machine with explicit states:
`Pending → Running → Succeeded | Failed | Cancelled`. Each
step within the run has its own state. Transitions are
recorded in the DB and emit audit events.

This makes resumption + re-run from a specific step trivial:
restore the DB state, the engine picks up where it left off.

### 2. Pipelines are immutable; runs are mutable

A `Pipeline` definition is versioned. Once a version is
applied, it cannot be modified — only superseded by a new
version. Each `Run` references a specific pipeline version.

A run from yesterday's pipeline definition behaves exactly as
it did yesterday, even after the definition is updated. This
is the only way to reproduce a result months later.

### 3. Gates separate "ready to run next step" from "next step starts"

Conditional gates evaluate at the boundary between steps. They
have access to outputs of completed steps. They can:
- Pass (continue to dependent steps).
- Fail (the run halts with a specific gate-failure reason).
- Wait for human approval (a separate workflow involving the
  audit-chain).

### 4. Retry semantics are explicit

Three retry kinds, all configurable per-step:
- `transient` — network errors, OOM kills: retry with
  exponential backoff.
- `intermittent` — partial success: retry once.
- `permanent` — schema errors, bad config: do not retry.

The executor classifies step failures using exit codes +
status conditions.

### 5. Observability per-step + per-pipeline

Metrics:
- `workflow_step_duration_seconds{pipeline, step, status}`
- `workflow_step_retry_total{pipeline, step}`
- `workflow_gate_evaluation_total{pipeline, step, result}`
- `workflow_run_duration_seconds{pipeline, status}`

Traces: each run is a parent span, each step is a child span.

Logs: per-step structured logs piped to the central log store
with `run_id` + `step_name` labels.

## Data model

```sql
CREATE TABLE pipelines (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    namespace TEXT NOT NULL,
    version TEXT NOT NULL,
    spec JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    UNIQUE (namespace, name, version)
);

CREATE TABLE runs (
    id UUID PRIMARY KEY,
    pipeline_id UUID REFERENCES pipelines(id),
    triggered_by TEXT NOT NULL,    -- 'schedule', 'event', 'manual'
    triggered_at TIMESTAMPTZ NOT NULL,
    status TEXT NOT NULL,           -- Pending|Running|Succeeded|Failed|Cancelled
    finished_at TIMESTAMPTZ
);

CREATE TABLE step_states (
    run_id UUID REFERENCES runs(id),
    step_name TEXT NOT NULL,
    status TEXT NOT NULL,           -- Pending|Running|Succeeded|Failed|Skipped|WaitingApproval
    attempts INTEGER NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ,
    finished_at TIMESTAMPTZ,
    pod_name TEXT,
    outputs JSONB,
    error TEXT,
    PRIMARY KEY (run_id, step_name)
);

CREATE TABLE gate_evaluations (
    run_id UUID REFERENCES runs(id),
    step_name TEXT NOT NULL,
    condition TEXT NOT NULL,
    result TEXT NOT NULL,           -- pass|fail|pending
    evaluated_at TIMESTAMPTZ NOT NULL,
    approver TEXT,                   -- non-null when result is human approval
    PRIMARY KEY (run_id, step_name, condition)
);
```

## Scheduler

- **Cron-based**: a watcher reads `pipelines.spec.schedule`
  and inserts a new `run` record at the scheduled time.
- **Event-triggered**: a small webhook accepts events (e.g.,
  "training data updated") and triggers the matching pipeline.

For the capstone, both are simple — cron is a Kubernetes
CronJob, events go through the FastAPI service.

## Executor

For each step in a run:
1. Wait until all dependencies are `Succeeded`.
2. Evaluate gates if any.
3. Create the step pod with the configured image + env + resources.
4. Watch pod completion.
5. Classify exit (success, transient failure, permanent failure).
6. Retry per policy or update state.
7. Record outputs in `step_states.outputs`.

## Re-run from step

CLI: `smartrecs workflow rerun <run-id> --from <step-name>`.

Implementation: clone the run record, set steps prior to
`<step-name>` to `Succeeded` with their original outputs, set
`<step-name>` onward to `Pending`, restart the executor.

## Cross-references

- Module 05 lecture notes for the conceptual framework.
- [Module 03 / multi-tenancy](../../lessons/mod-003-multi-tenancy-resources/) for per-tenant isolation patterns.
- Module 09 (Security track) for the audit-chain integration.
