# Project 05 — Architecture

## Component map

```
┌──────────────────────────────────────────────────────────────┐
│ Data scientist's browser / terminal                          │
└────────────────────────────────┬─────────────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Web Portal       │  │ CLI              │  │ Python SDK       │
│ (React/Next.js)  │  │ (smartrecs)      │  │ (in their code)  │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                     │
         │  OIDC token          │  OIDC token         │  OIDC token
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               ▼
                  ┌──────────────────────────┐
                  │ API Gateway              │
                  │  (single ingress point)  │
                  └────┬─────────────────────┘
                       │
       ┌───────────────┼───────────────────────────┐
       ▼               ▼                           ▼
┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Project 01  │  │ Project 02   │  │ Project 03   │  │ Project 04   │
│ Platform    │  │ Features     │  │ Workflows    │  │ Models       │
│ Core        │  │              │  │              │  │              │
└─────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

## Key design decisions

### 1. Unified entry point

The portal, CLI, and SDK all hit a single API gateway that
fans out to projects 01-04. From the user's perspective there's
one platform; the implementation can refactor service
boundaries without breaking clients.

### 2. SDK is the contract

The Python SDK is the canonical interface. The CLI is a thin
wrapper over the SDK. The portal's frontend calls the same
APIs the SDK does, through the API gateway.

This means: any feature on the platform is testable from the
SDK. Any test that passes in the SDK proves the surface works.

### 3. Documentation is part of the deliverable

Docs are not an afterthought. They ship with the same release
cycle as code. Broken links + outdated examples are bugs.

The docs site is built from markdown in the project repo,
auto-deployed on every merge to main, and includes:
- API reference auto-generated from OpenAPI.
- Quickstart code blocks tested in CI (the example code in
  the docs is the same code in the tests/ directory).

### 4. Onboarding is a first-class flow

The "first 5 minutes" experience matters disproportionately.
The onboarding flow:
1. Sign in via SSO.
2. Associate with a tenant (or request access).
3. Get tenant-scoped credentials installed locally.
4. Run a smoke-test command that verifies all four service
   layers are reachable.

If any step fails, the user gets a specific actionable error
(not "something went wrong").

### 5. Status page reflects reality

The portal includes a status page that:
- Probes each service every 30s.
- Shows green/yellow/red per service.
- Surfaces active incidents from the platform's IR tooling.
- Doesn't lie. If the platform is degraded, the status page
  shows it.

## Stack choices

| Layer | Stack | Rationale |
|---|---|---|
| Web UI | Next.js + TypeScript + Tailwind | Modern; well-supported; easy to deploy as static + SSR mix |
| Docs | Docusaurus or MkDocs | Mature; built-in search; markdown-native |
| SDK | Python 3.11+ | Same Python as data scientists use |
| CLI | Typer or Click | Standard Python CLI frameworks |
| Status probe | Lightweight background job (cron + Prometheus) | Avoid over-engineering |

## Onboarding flow (sequence)

```
User                    Portal           IdP             Tenant Mgr      Platform
 │                        │                │                │                │
 │ Sign in                │                │                │                │
 ├───────────────────────▶│                │                │                │
 │                        │ Redirect       │                │                │
 │                        ├───────────────▶│                │                │
 │                        │                │ Auth + token   │                │
 │                        │◀───────────────┤                │                │
 │                        │ Has tenant?    │                │                │
 │                        ├───────────────────────────────▶│                │
 │                        │◀───────────────────────────────┤                │
 │                        │                │                │                │
 │                  (yes) │ Welcome dashboard               │                │
 │                  (no)  │ Request tenant access screen    │                │
 │                        │                                  │                │
 │ Install SDK            │                                  │                │
 │◀───────────────────────┤ pip install ai-infra-platform   │                │
 │                        │                                  │                │
 │ Run smoke test         │                                  │                │
 ├──────────────────────────────────────────────────────────▶│                │
 │                        │                                  │ check 01-04   │
 │                        │                                  ├───────────────▶│
 │                        │                                  │◀───────────────┤
 │◀───────────────────────────────────────────────────────────┤ all green ✓    │
```

## Status probe

```python
# Pseudocode
SERVICES = {
    "platform-core": "http://platform-core:8080/healthz",
    "features": "http://features:8080/healthz",
    "workflows": "http://workflows:8080/healthz",
    "models": "http://models:8080/healthz",
}

def probe():
    results = {}
    for name, url in SERVICES.items():
        try:
            r = requests.get(url, timeout=5)
            results[name] = "green" if r.ok else "yellow"
        except Exception:
            results[name] = "red"
    return results
```

Status page reads from this every 30s.

## Cross-references

- Module 07 lecture notes.
- [Backstage](https://backstage.io/) for inspiration.
- [Vercel + Next.js docs](https://nextjs.org/docs) for the
  portal stack.
