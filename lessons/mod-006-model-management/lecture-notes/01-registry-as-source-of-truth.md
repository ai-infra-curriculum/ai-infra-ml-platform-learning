# Lecture 01: Registry as Source of Truth

The model registry answers: "what's in production?"

Without a registry, the answer is whatever's in some pod's image filesystem.
With a registry, it's `models:/<name>/Production`. The model in
production is *named* and *versioned*. Rollback becomes a single API call.

## Registry components

- **Name**: unique identifier per model (e.g., `iris-rf`)
- **Versions**: monotonically increasing integers
- **Stages**: None → Staging → Production → Archived
- **Aliases**: `champion`, `challenger`, `canary` etc.
- **Tags**: free-form key/value (owner, accuracy, build_sha, framework)
- **Description**: free-form text + optional schema

## The promotion workflow

```
[Training produces] → register → version N
                                  │
                              [eval gate]
                                  │
                              Staging (auto)
                                  │
                              [smoke test in staging]
                                  │
                          [manual approval]
                                  │
                              Production
                                  │
                          [old version → Archived]
```

Each transition is API-driven, audit-logged.

## Source of truth contract

Code that loads models from anywhere except the registry is suspect. Common
violations:
- `joblib.load("/data/model.pkl")` — bypasses registry entirely
- Image-baked models — model version is implicit in image tag
- Direct S3 path lookups

The fix: serving infrastructure pulls from the registry at startup, with
the registry URI in config.

## Companion

[engineer-solutions/mod-106 ex-03 (model-registry-promotion)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops/exercise-03-model-registry-promotion) — working quality-gated promote.py + rollback.py.
