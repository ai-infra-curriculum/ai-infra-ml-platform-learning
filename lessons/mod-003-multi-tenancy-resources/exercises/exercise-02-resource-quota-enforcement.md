# Exercise 02: ResourceQuota Enforcement Test

## Objective

Prove that ResourceQuota + LimitRange actually prevent noisy-neighbor outages.

## Tasks

1. Set up team-A with quota cpu=4 / memory=8Gi.
2. Deploy a "stress" pod from team A that requests cpu=10. Verify it's rejected at admission.
3. Deploy a "creep" pod that requests cpu=0.1 but tries to use 100% CPU in code. Verify the cgroup throttles it.
4. Deploy 5 pods each requesting cpu=1. Verify the 5th is rejected (4 already used).

## Deliverable

`TESTS.md` documenting:
- What was rejected, with admission webhook output
- What was throttled, with `kubectl top pod` output
- What happens at quota boundary (helpful error message? confusing?)
- Recommendation for error messaging improvements
