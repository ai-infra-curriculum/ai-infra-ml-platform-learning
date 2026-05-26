# Lecture 01: Tenant Isolation Models on Kubernetes

## Four isolation models

| Model | Isolation | Cost | Operational complexity |
|---|---|---|---|
| Shared namespace | weakest | lowest | low |
| Namespace per team | strong | low | low-medium |
| Namespace per project + RBAC | strong | low | medium |
| Virtual cluster (vcluster) | strongest | medium | high |
| Hard multi-tenancy (separate clusters) | strongest | highest | high |

Default to **namespace-per-team**. Move to virtual clusters or separate
clusters only when justified by compliance, blast-radius, or extreme noisy
neighbors.

## Namespace-per-team layout

```yaml
metadata:
  name: team-ml-platform
  labels: { team: ml-platform, tier: ml }
```

Then for every team:
- `ResourceQuota` capping CPU/memory/GPU
- `LimitRange` setting per-container defaults
- `NetworkPolicy` deny-all + explicit allow for cross-namespace traffic
- `RoleBinding` granting team membership the `admin` role within their namespace

## Cost attribution

Tag every resource with `team` + `cost_center` labels. Enforce with Kyverno:

```yaml
validate:
  message: "Pods must declare team + cost_center labels"
  pattern:
    metadata:
      labels:
        team: "?*"
        cost_center: "?*"
```

Aggregate Kube state metrics by team labels in Prometheus → daily rollup → invoice.

## Common failures

- **Team A's batch job evicts team B's serving pod**: missing PriorityClass + PodDisruptionBudget
- **Team A consumes all GPUs**: missing GPU ResourceQuota
- **Cross-team Slack about etcd events**: missing RBAC; team A can see team B's secrets
- **Cost spike, no attribution**: missing labels; Finance can't tell who to bill
