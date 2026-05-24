# Module 07: Developer Experience — Quiz

10 questions. 70% pass.

### 1. The most defensible success metric for a platform team is:
- [ ] a) Number of models deployed this quarter
- [ ] b) Test coverage % of the platform code itself
- [ ] c) Number of platform engineers hired
- [x] d) Time from "trained" to "in production" (user lead time)

### 2. The Diátaxis documentation framework distinguishes:
- [x] a) Tutorial, how-to, reference, explanation
- [ ] b) Docs, wiki, runbook, FAQ
- [ ] c) Code, comments, README, ADR
- [ ] d) Slack, email, ticket, meeting

### 3. A good error message includes (select all that apply):
- [x] a) What went wrong (the symptom)
- [x] b) Probable cause
- [x] c) Suggested fix or next step
- [x] d) Link to relevant documentation

### 4. Customer discovery for a platform team should be:
- [ ] a) An anonymous quarterly Surveymonkey form
- [ ] b) Delegated entirely to a PM
- [ ] c) Skipped — platform users will file tickets when they need things
- [x] d) Quarterly structured user interviews + ongoing Slack/ticket analysis

### 5. The "gatekeeper trap" for a platform team is:
- [ ] a) The same thing as governance
- [ ] b) A regulatory compliance requirement
- [ ] c) A scaling problem solved by hiring more engineers
- [x] d) Every project routes through your queue; you become the bottleneck

### 6. CLI output should support:
- [x] a) `--output json|yaml|table` everywhere; scriptable + readable
- [ ] b) Only human-readable text formatting
- [ ] c) Only JSON
- [ ] d) HTML

### 7. "Docs as infrastructure" means:
- [x] a) Versioned alongside code, built in CI, audited on a cadence
- [ ] b) Hosted on a wiki separate from the repo
- [ ] c) Maintained exclusively by tech writers
- [ ] d) Only updated at major release time

### 8. CLI telemetry is most useful for:
- [ ] a) Required compliance with Kubernetes ecosystem
- [ ] b) Improving CLI startup time
- [ ] c) Blocking unauthorized usage
- [x] d) Learning which commands fail in the wild + which flags go unused

### 9. The first thing to ship in a new platform is:
- [x] a) A 5-minute quickstart that produces a working end-to-end demo
- [ ] b) The internal data model
- [ ] c) The cost attribution dashboard
- [ ] d) The full CLI surface

### 10. Documentation that's older than the code it describes is:
- [ ] a) Acceptable; perfect is the enemy of good
- [ ] b) Required by SOC 2 for change control
- [ ] c) Standard practice across the industry
- [x] d) Worse than no documentation — it's actively misleading

---

## Answer key + rationale

1. **d** — Lead time directly captures the platform's job (make users productive). The other metrics are vanity or internal.
2. **a** — Diátaxis is the canonical four-quadrant model: tutorial/how-to/reference/explanation.
3. **a+b+c+d** — All four are required for a good error message; missing any one degrades the UX.
4. **d** — Surveys are too thin; tickets miss silent users. Structured interviews + telemetry covers both.
5. **d** — The trap: success → more requests → ticket queue → bottleneck. Self-serve is the antidote.
6. **a** — Scriptable + human-readable is non-negotiable for a power-user surface.
7. **a** — Treat docs like code: PR-reviewed, CI-tested, dated.
8. **d** — Telemetry surfaces gaps the team would otherwise miss.
9. **a** — Quickstart is the first impression; without it, users bounce before they evaluate the platform.
10. **d** — Stale docs make users debug the wrong problem. Better to delete than to mislead.
