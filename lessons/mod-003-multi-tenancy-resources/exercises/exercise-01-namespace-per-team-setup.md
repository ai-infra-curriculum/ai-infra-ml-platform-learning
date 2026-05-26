# Exercise 01: Namespace-per-Team Setup

## Objective

Configure a fresh cluster to host 3 teams with proper isolation.

## Requirements

For each of teams ml-platform, data-science, trust-safety:
- Namespace with `team` + `tier` labels
- ResourceQuota: cpu/memory/GPU caps per team
- LimitRange: per-container defaults + max
- Default-deny NetworkPolicy + explicit-allow for monitoring + ingress
- RoleBinding granting team members `admin` on their namespace

## Deliverable

- Idempotent `apply.sh` that creates everything from a `teams.yaml` config
- `verify.sh` that confirms a tenant from team A cannot read team B's secrets

## Companion

[engineer-solutions/mod-104 ex-14 (multi-tenancy)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-104-kubernetes/exercise-14-resource-quotas-multitenancy) has the full template.
