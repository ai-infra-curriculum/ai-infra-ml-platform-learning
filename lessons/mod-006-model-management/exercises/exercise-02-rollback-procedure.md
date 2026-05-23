# Exercise 02: Rollback Procedure

Write a `rollback.py` that:
- Finds the current Production version
- Finds the most recent previously-Production version (now Archived or Staging)
- Atomically transitions current → Archived, prev → Production
- Logs the event to the audit log
- Returns clear error if no rollback target exists

Then write a tabletop exercise: walk through "v6 is bad in prod, what do we do?"
