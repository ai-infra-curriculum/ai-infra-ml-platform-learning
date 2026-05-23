# Exercise 01: Implement OIDC + RBAC

Add OIDC AuthN to your platform API. Use any IdP (Auth0 / Okta / Keycloak).
Map IdP groups → RBAC roles → resource permissions.

Demonstrate:
- Anonymous request: 401
- Authenticated user without permission: 403
- Authenticated user with permission: 200
