# Lecture 01: Orchestrator Landscape

## The choices

| Tool | Strength | Watch out for |
|---|---|---|
| Airflow | mature, large community, K8s exec | tasks-as-script; XCom limits; schema is everywhere |
| Prefect | Python-native, easy local dev | smaller ecosystem; Cloud/OSS divergence |
| Dagster | asset-centric, type-safe, dev UI | learning curve; smaller community |
| Argo Workflows | K8s-native, gang-friendly | YAML; less ML-domain ergonomics |
| Kubeflow Pipelines | ML-specific, Vertex-AI-compatible | heavy install; opinionated |
| Flyte (Lyft/Spotify) | type-safe + reproducibility-first | Python-only; complex bootstrap |
| Metaflow (Netflix) | DS-friendly, one-decorator usage | weaker prod-ops story without paid tier |

## Decision matrix

| Criterion | Airflow | Prefect | Dagster | Kubeflow | Metaflow |
|---|---|---|---|---|---|
| Python-native | medium | high | high | medium | high |
| Type-safety | low | medium | high | low | medium |
| K8s integration | high | medium | high | very high | medium |
| Production maturity | very high | high | medium | high | medium |
| Lock-in risk | low | low | low | medium | medium |
| Hiring pool | very large | medium | small | medium | small |

## Recommendation

- **Default**: Airflow. Predictable, large community, K8s executor works.
- **If you're heavily ML-specific + Vertex-bound**: Kubeflow Pipelines.
- **If you have strong typing instincts + small team**: Dagster.
- **If you're Lyft/Spotify scale**: Flyte.

Avoid choosing by "this is what we read about most recently."

## Anti-patterns

- One-orchestrator-per-team (no standardization)
- DAGs that mutate global state (not idempotent)
- 5K+ DAGs with no organizational principle
- Running training jobs as DAG tasks (use K8sPodOperator or Kueue/Volcano)
- Storing data in XCom (use a registry / S3; XCom for paths only)
