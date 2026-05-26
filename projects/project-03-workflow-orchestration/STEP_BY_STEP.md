# Project 03 — Step-by-Step Build Guide

## Phase 0 — Setup (1-2 hours)

Local Kubernetes + PostgreSQL. Project skeleton:

```
project-03-workflow-orchestration/
├── api/             # FastAPI control plane
├── scheduler/       # cron + event scheduler
├── executor/        # DAG state machine + pod runner
├── sdk/             # Python client
├── cli/             # smartrecs CLI
├── deploy/
├── tests/
└── Makefile
```

## Phase 1 — Pipeline parser (5-6 hours)

1. JSON Schema for `Pipeline` YAML.
2. Parser that builds a DAG from steps.
3. Cycle detection (`raise ValueError("cycle: …")`).
4. Test fixtures: valid, invalid (cycle), invalid (unknown
   dependency).

## Phase 2 — DAG state machine (10-12 hours)

The heart of the engine. Don't shortcut this.

1. Define states for runs + steps (see `architecture.md`).
2. Implement transitions as pure functions
   (`transition(current_state, event) -> new_state` or error).
3. Persist state changes in a DB transaction with audit-chain
   write.
4. Test: every transition that should be allowed succeeds;
   every disallowed transition raises.

## Phase 3 — Executor (10-12 hours)

1. For each Run in `Pending` or `Running`, find steps whose
   dependencies are `Succeeded` and that are `Pending`.
2. Evaluate gates.
3. Create the step pod, watch it, classify the result.
4. Update step state via the state machine.
5. On run completion, fire notifications.

Idempotency: every executor iteration is safe to repeat.
Restart-safe: on restart, the executor reads DB state and
resumes.

## Phase 4 — Scheduler (6-7 hours)

1. Cron loop reading `pipelines.spec.schedule`.
2. Event endpoint: `POST /v1/events` matches pipelines with
   matching event filters.
3. Both produce a new `runs` row, the executor picks it up.

## Phase 5 — Gates (6-7 hours)

1. Programmatic gate: evaluate a condition string against
   step outputs.
   - **Use a safe expression library** (e.g., `simpleeval`,
     `asteval`, or a custom AST-restricted evaluator). Never
     use Python's built-in dynamic-code-execution function
     directly — it's an arbitrary-code-execution vector.
   - Conditions like `"accuracy >= 0.85 AND fairness <= 1.25"`.
2. Human approval gate:
   - Run pauses at the gate.
   - `POST /v1/runs/{id}/approve` records approval.
   - State machine transitions on approval received.

## Phase 6 — Retry classification (4-5 hours)

Classify step failure:
- Exit code 137 (OOM) → transient.
- Exit code 1 with specific message patterns → permanent.
- Pod evicted → transient.
- Default → transient with backoff.

Each step config can override the classification.

## Phase 7 — Re-run from step (3-4 hours)

CLI command + control-plane endpoint. Clone the run, mark
prior steps as Succeeded with their outputs, mark
`<step>` onward as Pending, executor resumes.

## Phase 8 — Per-tenant isolation (3-4 hours)

- Pipelines in tenant namespaces.
- Step pods in tenant namespaces.
- API enforces caller's tenant matches pipeline's tenant on
  trigger.

## Phase 9 — Audit chain + observability (5-6 hours)

- Audit-chain table + insert-only constraint.
- Prometheus metrics on every state transition.
- OpenTelemetry traces per run + per step.
- Grafana dashboard.

## Phase 10 — Notifications (2-3 hours)

PagerDuty + Slack webhook clients. Config per pipeline.

## Phase 11 — Testing + docs (5-7 hours)

Full acceptance demo + documentation.

## Time-budget recap

| Phase | Hours |
|---|---|
| 0 — Setup | 1-2 |
| 1 — Parser | 5-6 |
| 2 — State machine | 10-12 |
| 3 — Executor | 10-12 |
| 4 — Scheduler | 6-7 |
| 5 — Gates | 6-7 |
| 6 — Retry classification | 4-5 |
| 7 — Re-run from step | 3-4 |
| 8 — Tenant isolation | 3-4 |
| 9 — Audit + observability | 5-6 |
| 10 — Notifications | 2-3 |
| 11 — Testing + docs | 5-7 |
| **Total** | **~75** |

## Common pitfalls

- **Unsafe expression evaluation in gate conditions**: never
  pass user-supplied strings into Python's dynamic-code-
  execution function. Use a whitelisted expression parser.
  This is the most common arbitrary-code-execution mistake in
  orchestration tools.
- **Non-atomic state writes**: a step transitioning from
  `Running` to `Succeeded` and the audit-chain write must be
  atomic. Wrap in a DB transaction.
- **Forgetting backoff on transient retries**: a tight retry
  loop on a flaky downstream service makes things worse.
- **Skipping the cycle check**: pipelines with cycles will
  spin forever in the executor.

## When you're done

You have an orchestration engine that:
- Runs declarative pipelines with full DAG semantics.
- Handles transient + permanent failures distinctly.
- Supports human approval gates.
- Re-runs cleanly from any step.
- Audits every state transition.
- Surfaces metrics + traces.

This is the workflow layer Module 06 (model management) sits
on top of.
