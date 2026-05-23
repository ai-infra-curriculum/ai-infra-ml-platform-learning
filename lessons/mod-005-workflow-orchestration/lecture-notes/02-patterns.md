# Lecture 02: ML Orchestration Patterns

## Four production patterns

### Pattern 1 — DAG-per-model

One DAG file per model. Easy to read, easy to debug, scales linearly in code.

```python
# dags/iris_train.py
@dag(...)
def iris():
    ingest >> train >> evaluate >> deploy
```

Right when: few (< 20) models with substantially different shapes.

### Pattern 2 — Parametric DAG

One DAG generates N pipelines from a registry config.

```python
for cfg in MODELS_YAML:
    make_dag(cfg)
```

Right when: many (50+) models with similar shape, differing in config only.

### Pattern 3 — Event-driven

S3 `PutObject` → SQS → Lambda → trigger DAG with the object key.

Right when: data arrival is irregular; cron would be wasteful.

### Pattern 4 — Continuous training

Cron + branching: every hour, check drift; retrain iff drift > threshold.

```python
@dag(schedule="0 * * * *")
def ct():
    s = check_drift()
    if s["psi"] > 0.25 or s["new_rows"] > 1_000_000:
        retrain()
```

Right when: drift-sensitive models; cost of stale model > cost of retraining.

## Picking

Most platforms run several patterns simultaneously. The mistake is forcing
all models into one pattern. Document a `PATTERN_DECISION.md` per model:
which pattern, why.

## Companion

[engineer-solutions/mod-106 ex-11](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops/exercise-11-ml-orchestration-patterns) has working code for all four.
