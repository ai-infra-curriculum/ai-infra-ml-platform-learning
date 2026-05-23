# Lecture 02: Deployment Strategies (Registry-Driven)

Once the registry holds Production + Staging versions, deploy strategies
become declarative:

## Strategies

| Strategy | Production replicas | New version receives |
|---|---|---|
| Rolling | progressive replacement | 25% → 50% → 75% → 100% |
| Blue-Green | full duplicate | 0%, then 100% on cutover |
| Canary | small initial | 5% → 25% → 50% → 100% with auto-gate |
| Shadow | mirrored | 0% real, 100% mirror |

The registry holds the version; the serving infra implements the strategy.

## When to use each

- **Rolling**: low-risk version bumps
- **Blue-Green**: schema-breaking changes or full migration
- **Canary**: risky changes with auto-gate on success-rate
- **Shadow**: new model architectures; want to validate without risk

## Auto-gating

Argo Rollouts AnalysisTemplate gates canary by:
```yaml
successCondition: result[0] >= 0.99   # 5xx-free rate
failureLimit: 1
```

If the gate fails, rollout auto-reverts. The registry stays as-is; only the
serving fleet changes.

## Companion

[engineer-solutions/mod-106 ex-08 (deployment-strategies)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops/exercise-08-deployment-strategies) — all four with manifests + measured comparison.
