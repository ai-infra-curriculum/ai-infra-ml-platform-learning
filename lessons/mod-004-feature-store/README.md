# Module 04: Feature Store Architecture

**Duration**: ~10 hours guided + 12 hours exercises
**Prerequisites**: Modules 01-03; SQL + Python

## Overview

Feature stores solve the single most common silent failure mode in production
ML: training-serving skew. This module covers the architecture (offline +
online + registry), point-in-time correctness, and the operational discipline
required to make a feature store useful instead of a maintenance burden.

## Objectives

1. Articulate why training-serving skew exists + how feature stores fix it.
2. Implement an offline store (Parquet on S3) + online store (Redis) + registry (Feast).
3. Build point-in-time-correct training join.
4. Build materialization pipeline (offline → online) with backfill support.
5. Operate a feature store: monitoring freshness, drift, cost.

## Structure
```
mod-004-feature-store/
├── README.md
├── lecture-notes/  (3 files)
├── exercises/      (5)
├── quizzes/
└── resources.md
```

## Companion
[engineer-solutions/mod-106 ex-07](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops/exercise-07-feature-store-implementation) — full Feast setup.
