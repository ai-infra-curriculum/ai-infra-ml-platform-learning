# Lecture 03: Governance Integration

The registry is also where governance hooks attach.

## Required artifacts at promotion

A model cannot enter Production without:
- A signed model card (template auto-filled from training metadata)
- A bias review record (for protected-attribute models)
- A decision log entry (architectural choices documented)
- Audit log entry of the promotion event

These are enforced programmatically — the promote API checks for them.

## Compliance reviews

Per quarter, audit-time check:
- Every Production model has a card less than 1 quarter old
- Every model with protected attributes has a current bias review
- All promotions in the period are in the audit log

## Audit log shape

Each promotion event records:
- Timestamp
- Actor (who clicked promote)
- From version, to version
- Reason ("scheduled retrain", "rollback", "incident response")
- Quality gate values (accuracy delta vs prior Production)

Store as append-only with hash chaining — see project-4-governance pattern.

## Companion

[engineer-solutions/mod-106 ex-10 (model-governance)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-106-mlops/exercise-10-model-governance) — templates + scripts.
