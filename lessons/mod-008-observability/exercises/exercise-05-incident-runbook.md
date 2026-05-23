# Exercise 05: Incident Runbook Library

For an ML platform, write runbooks for:
- HighErrorRate (5xx > 5% for 5m)
- HighDrift (PSI > 0.25 for 30m)
- ModelAccuracyRegression (rolling 7d < baseline -5pp)
- BudgetExhausted (model exceeds monthly budget)
- TrainingDAGSLAMiss

Each runbook: detection → triage decision tree → common causes → remediation.

Companion: engineer-solutions/mod-108 ex-09 + ex-08.
