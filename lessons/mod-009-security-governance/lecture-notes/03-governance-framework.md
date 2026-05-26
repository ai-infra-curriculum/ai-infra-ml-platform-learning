# Lecture 03: Governance Framework

## What governance means for an ML platform

Beyond the security primitives: governance is the process layer that ensures
models reach production with the right artifacts + reviews.

## Required gates

| Gate | When | Owner |
|---|---|---|
| Code review | before merge | tech lead |
| Security scan | CI | platform |
| Bias review | before Production promotion | compliance |
| Model card | before Production promotion | model owner |
| Decision log | for architectural changes | model owner |
| Compliance attestation | quarterly | compliance |

## Audit trail

Every gate produces an audit entry. The full chain is reconstructable from
the audit log. See project-4-governance for a working hash-chained example.

## Compliance frameworks

- **NIST AI RMF**: voluntary, US
- **EU AI Act**: regulatory, EU; tiered by risk class
- **ISO/IEC 23894**: international, AI risk management
- **SR 11-7**: US banking; rigorous model risk management

Most platforms can comply with EU AI Act low-risk tier by implementing the
gates above + maintaining model cards + bias reviews.

## Companion

[engineer-solutions/mod-106 ex-10 (model-governance)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops/exercise-10-model-governance) + project-4-governance.
