# Lecture 03: CLIs and Error Messages

## Why CLIs matter

Most platform users will interact with the platform through:
1. UI (least common; usually self-managed dashboard)
2. SDK (libraries calling the API)
3. CLI (terminal commands)

The CLI is the unified, scriptable surface. Get it right and your power users
become your champions.

## CLI design principles

- **Hierarchical commands**: `ml jobs submit`, `ml jobs list`, `ml models promote`
- **Output formats**: support `--output json|yaml|table` everywhere
- **Composable**: outputs pipeable to jq + grep
- **Sane defaults**: 80% of use cases need zero flags
- **Helpful errors**: when something fails, explain how to fix it

## Error message anatomy

A good error message has four parts:

```
Error: failed to submit training job
Cause: insufficient GPU quota (requested 4, available 2)
Help: request quota increase or reduce gpu_count
Docs: https://docs.example.com/quotas
```

The bad pattern: `Error: 403` with no context.

## CLI examples

```bash
# Good
$ ml jobs submit --config recs.yaml
Submitted job-abc123 to queue 'training-priority'
Status: https://ui.example.com/jobs/abc123

# Bad (no helpful state, no link)
$ ml jobs submit --config recs.yaml
OK
```

## Telemetry

The CLI should anonymously emit:
- Command run + duration
- Error type if failure
- Version of CLI

You'll see immediately which commands fail most often + which flags are unused.
