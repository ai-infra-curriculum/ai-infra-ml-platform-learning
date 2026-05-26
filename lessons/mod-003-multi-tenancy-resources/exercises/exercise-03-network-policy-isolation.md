# Exercise 03: Network Policy Cross-Tenant Isolation

## Objective

Implement a default-deny network posture that lets each tenant talk to its own
services + shared infra (DNS, monitoring) but blocks cross-tenant traffic.

## Requirements

- Default-deny ingress + egress in every team namespace
- Allow egress to kube-system DNS (port 53 UDP/TCP)
- Allow egress to monitoring namespace
- Allow ingress only from ingress-nginx namespace
- No cross-team pod-to-pod traffic
- Demonstrate: team-A pod cannot connect to team-B service

## Deliverable

- NetworkPolicy manifests
- Test pod in team-A that tries (and fails) to reach team-B's service
- Test pod in team-A that successfully reaches monitoring
