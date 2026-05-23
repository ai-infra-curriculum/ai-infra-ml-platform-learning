# Exercise 03: Canary Deployment via Argo Rollouts

Wire up a model deploy that uses Argo Rollouts canary strategy:
- 5% → 25% → 50% → 100% with success-rate gate
- Failure mode: auto-revert
- Demonstrate by deploying a deliberately-broken model and watching it auto-revert

Companion: engineer-solutions/mod-106 ex-08.
