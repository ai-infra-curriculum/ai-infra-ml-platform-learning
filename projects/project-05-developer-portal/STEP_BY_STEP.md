# Project 05 — Step-by-Step Build Guide

## Phase 0 — Setup (1-2 hours)

```
project-05-developer-portal/
├── web/                # Next.js portal
├── sdk/                # Python SDK
├── cli/                # CLI (Typer)
├── docs/               # Docusaurus / MkDocs site
├── status-probe/       # the health probe job
├── deploy/             # K8s manifests, ingress, etc.
├── tests/
└── Makefile
```

## Phase 1 — Unified SDK (7-8 hours)

The SDK is the contract. Build it first.

1. `Platform` client with sub-clients (`training`, `features`,
   `workflows`, `models`).
2. OIDC token loading: env var → interactive device flow
   fallback.
3. Async + sync variants of every method.
4. Type annotations strict; mypy passes.
5. Test against project-01 through project-04 services
   (mocked for unit tests; real for integration).

## Phase 2 — CLI (5-6 hours)

1. Typer-based; subcommands per SDK area.
2. JSON output flag (`--json`) for scripting.
3. Bash + zsh completion installable via
   `smartrecs completion install`.
4. Error messages: clear, actionable, link to docs where
   applicable.

## Phase 3 — Documentation site (7-8 hours)

1. Set up Docusaurus or MkDocs.
2. Auto-deploy on merge to main (GitHub Actions or similar).
3. Auto-generated API reference from OpenAPI specs of
   projects 01-04.
4. Quickstart page with runnable examples (the same code
   tested in CI).
5. Search wired (Algolia DocSearch, lunr, or built-in).

## Phase 4 — Portal web UI (18-20 hours)

The biggest single phase. Build incrementally.

1. **Auth + onboarding** (5-6 hours):
   - SSO sign-in (NextAuth or similar).
   - Tenant association screen.
   - SDK install verification.
   - Smoke-test runner.
2. **Dashboard** (5-6 hours):
   - Lists of your runs, models, features.
   - Resource-usage display.
   - Alerts surface.
3. **Catalog** (3-4 hours):
   - Available datasets, features, base models.
   - Filter + search.
4. **Status page** (2-3 hours):
   - Read from the status probe.
   - History view.
5. **Feedback widget** (2-3 hours):
   - In-page button.
   - Slack webhook + GitHub issue API integration.

## Phase 5 — Status probe (3-4 hours)

Lightweight Kubernetes Deployment that probes each service
every 30s, writes results to a small cache. Portal reads from
the cache.

## Phase 6 — Testing + accessibility (6-8 hours)

1. SDK unit + integration tests.
2. CLI tests.
3. Frontend smoke tests with Playwright or Cypress.
4. Lighthouse audit; achieve >= 80.
5. Accessibility audit; achieve WCAG 2.1 AA on key flows.
6. Doc-example tests: the code blocks in docs pass in CI.

## Phase 7 — Documentation (3-4 hours)

The meta-task: write the docs the portal serves. Per
`requirements.md` §F4.

## Time-budget recap

| Phase | Hours |
|---|---|
| 0 — Setup | 1-2 |
| 1 — SDK | 7-8 |
| 2 — CLI | 5-6 |
| 3 — Docs site | 7-8 |
| 4 — Portal UI | 18-20 |
| 5 — Status probe | 3-4 |
| 6 — Testing + a11y | 6-8 |
| 7 — Documentation | 3-4 |
| **Total** | **~60** |

## Common pitfalls

- **Hardcoded service URLs**: use an API gateway pattern; the
  portal/SDK/CLI all hit a single base URL.
- **Breaking the docs**: every API change should update both
  code and docs in the same PR. If docs are out of date, the
  portal lies.
- **Skipping accessibility**: easy to defer, expensive to
  retrofit. Build accessibility in from the start.
- **No tests for docs**: docs rot. Tested doc examples don't.
- **Onboarding that takes 30 minutes**: kills adoption. If
  the smoke test isn't passing in 10 minutes for a new user,
  the onboarding has a bug.

## When you're done

A new data scientist signs in, gets onboarded, runs their
first job, and sees the result in the dashboard — all in
under 15 minutes. The status page shows green. The docs
answer their first three "how do I..." questions without them
needing to ping the platform team.

That's a platform. The previous four projects built the
capabilities; this one made them usable.
