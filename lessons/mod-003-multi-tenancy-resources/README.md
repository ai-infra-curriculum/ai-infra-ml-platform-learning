# Module 03: Multi-Tenancy & Resource Management

**Duration**: ~10 hours guided + 10-15 hours exercises
**Prerequisites**: Modules 01-02; Kubernetes namespace + RBAC + ResourceQuota fluency

## Overview

Single-tenant platforms scale only as far as their first noisy team. Multi-
tenant platforms scale to many tenants if — and only if — isolation, quotas,
fair scheduling, and cost attribution are designed in from the start.

## Learning objectives

1. Choose between namespace-per-team, namespace-per-project, and virtual-cluster patterns.
2. Implement K8s resource quotas + limit ranges to prevent noisy-neighbor failures.
3. Design network policies that default-deny across tenants.
4. Implement cost attribution: who paid for what compute this month.
5. Implement fair-share scheduling (Volcano, Yunikorn) for GPU contention.

## Structure

```
mod-003-multi-tenancy-resources/
├── README.md
├── lecture-notes/   (3 files)
├── exercises/       (5)
├── quizzes/
└── resources.md
```
