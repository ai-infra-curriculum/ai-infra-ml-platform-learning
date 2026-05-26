# Exercise 04: Governance Gate at Promotion

Build a promote API that *refuses* to promote to Production unless:
- A model card exists for this version
- A bias review record exists (if model uses protected attributes)
- The previous Production version's accuracy delta is documented

Test by trying to promote without each prerequisite + getting clear errors.

Companion: engineer-solutions/mod-106 ex-10.
