# Project 05 — Requirements

## Functional requirements

### F1 — Web portal

- [ ] Built with Next.js + TypeScript + Tailwind (or
      equivalent modern stack).
- [ ] OIDC sign-in.
- [ ] Tenant-scoped dashboard (your runs, models, features,
      resource usage).
- [ ] Catalog page (browse available datasets, features,
      base models).
- [ ] Documentation hub (searchable).
- [ ] Status page (real-time service health).
- [ ] Feedback widget on every page.

### F2 — Unified SDK

- [ ] Python 3.11+ package: `pip install ai-infra-platform`.
- [ ] Authenticates via OIDC token (from env or interactive
      device flow).
- [ ] Surfaces from projects 01-04 accessible under one
      client: `Platform().training`, `.features`,
      `.workflows`, `.models`.
- [ ] Async-friendly: every method has an async variant.
- [ ] Type-annotated; passes mypy strict.
- [ ] Docstrings on every public method.

### F3 — CLI

- [ ] `smartrecs` command with subcommands matching SDK.
- [ ] Cross-platform (mac + linux).
- [ ] Bash + zsh completion installable via `smartrecs completion install`.
- [ ] Output formats: human (default), JSON (for piping).
- [ ] Errors return non-zero exit codes with clear messages.

### F4 — Documentation site

- [ ] Docusaurus or MkDocs.
- [ ] Auto-deployed on merge to main.
- [ ] Searchable.
- [ ] Sections:
      - Getting started (5-minute quickstart).
      - Concepts (one page per module of the curriculum).
      - API reference (auto-generated).
      - Runbooks + playbooks.
      - FAQ.
- [ ] Example code in docs is tested in CI (no broken
      examples shipped).

### F5 — Onboarding flow

- [ ] SSO sign-in → tenant association → SDK install
      verification → smoke test → success.
- [ ] Each step's failure gives a specific actionable error.
- [ ] First-time experience completable in under 10 minutes.

### F6 — Status page

- [ ] Real-time health from probes against projects 01-04.
- [ ] Active incidents surfaced.
- [ ] Public (read-only) URL for customers.
- [ ] Per-service history (last 30 days uptime).

### F7 — Feedback channels

- [ ] In-portal feedback widget posts to Slack + creates
      GitHub issue.
- [ ] Issue template pre-populates with: page URL, user
      tenant, browser/SDK version, recent activity.
- [ ] Public roadmap (read-only) showing accepted feature
      requests.

## Non-functional requirements

### NF1 — Accessibility

- [ ] WCAG 2.1 AA target.
- [ ] All interactive elements keyboard-accessible.
- [ ] Color contrast meets minimum.
- [ ] Screen-reader tested with at least one tool.

### NF2 — Performance

- [ ] Portal home page loads in under 2s on a 3G connection
      (use Lighthouse score >= 80).
- [ ] CLI commands return in under 500ms for status queries.

### NF3 — Internationalization

- [ ] Primary language English; UI strings centralized to
      enable future translation (don't ship translations,
      but don't hardcode strings either).

### NF4 — Testing

- [ ] Unit tests on the SDK (>= 80% coverage).
- [ ] Integration tests on CLI (end-to-end against a test
      cluster).
- [ ] Frontend smoke tests (Playwright or Cypress).
- [ ] Doc-example tests (the code in docs is run in CI).

### NF5 — Documentation

- [ ] README at project root.
- [ ] Architecture document.
- [ ] CONTRIBUTING.md for portal contributions (since this is
      meta — the portal documenting how to add to the portal).

## Acceptance demo

1. New user signs in via SSO from a fresh browser.
2. Onboarding flow completes (tenant association, SDK
   install, smoke test).
3. User submits a training run via the CLI; sees it in the
   portal dashboard within seconds.
4. User triggers a deliberate failure (e.g., misconfigured
   workflow); the portal shows the error with a link to the
   runbook.
5. User reports the issue via the feedback widget; a Slack
   message + GitHub issue are created.
6. Status page reflects current service health.
7. Docs site search returns relevant results.
8. Lighthouse score on portal home: >= 80.

## Submission

`deliverables/` contains:
- `web/` — portal frontend.
- `sdk/` — Python SDK source.
- `cli/` — CLI source.
- `docs/` — documentation site source.
- `tests/` — full test suite.
- `screenshots/` — onboarding flow, dashboard, status page,
  error flow, doc search.
- `SUBMISSION.md`.
