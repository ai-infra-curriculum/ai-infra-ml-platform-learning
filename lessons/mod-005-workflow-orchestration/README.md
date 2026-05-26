# Module 05: Workflow Orchestration

**Duration**: ~8 hours guided + 10-12 hours exercises
**Prerequisites**: Modules 01-04; Python + DAGs

## Overview

Orchestration is the connective tissue between every component built so far:
training, monitoring, retraining, deployment. Pick the orchestrator carefully —
it constrains every downstream pipeline for the next 5 years.

## Objectives

1. Compare orchestrators (Airflow, Prefect, Dagster, Argo, Flyte) on real criteria.
2. Author DAGs that handle retries, branching, dynamic mapping, SLAs.
3. Implement event-driven + scheduled + continuous-training patterns.
4. Operate orchestrators: pool capacity, idle vs busy, queue management.

## Structure
```
mod-005-workflow-orchestration/
├── README.md
├── lecture-notes/ (3)
├── exercises/      (5)
├── quizzes/
└── resources.md
```

## Companion
[engineer-solutions/mod-106 ex-11 (ml-orchestration-patterns)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops/exercise-11-ml-orchestration-patterns).
