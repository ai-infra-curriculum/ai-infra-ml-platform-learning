# Lecture 01: AuthN, AuthZ, and Tenancy

## Three concerns, often confused

- **Authentication (AuthN)**: who is this caller?
- **Authorization (AuthZ)**: what are they allowed to do?
- **Tenancy**: what data do they see?

## AuthN options

| Mechanism | When |
|---|---|
| API keys | service-to-service; static tokens |
| OAuth 2.0 + OIDC | human users; SSO-aware |
| Workload identity (IRSA, GCP WI) | pod → cloud service auth, no static creds |
| mTLS | pod-to-pod inside the mesh |

Use OIDC for humans (their identity comes from your IdP); workload identity
for services. Static API keys only as a last-resort.

## AuthZ patterns

- **RBAC**: roles → permissions (admin, viewer, ml-engineer)
- **ABAC**: attributes (team membership, time of day, project)
- **PBAC (policy-based)**: declarative policies in OPA / Cedar

For most ML platforms: RBAC at the resource level (namespace, project) +
ABAC for fine-grained decisions ("only the model owner can promote to prod").

## Tenancy enforcement

- Every API request carries a tenant header
- AuthN verifies the caller; AuthZ checks the tenant they're acting on
- Audit log captures (actor, action, target tenant)
- Cross-tenant operations are explicit + audited

## Companion

[engineer-solutions/mod-109 ex-07 (secret-management)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-109-infrastructure-as-code/exercise-07-secret-management).
