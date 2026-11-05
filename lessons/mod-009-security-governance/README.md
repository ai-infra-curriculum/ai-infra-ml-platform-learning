# Module 09: Security & Governance

**Duration**: ~8 hours guided + 6-8 hours exercises
**Prerequisites**: Modules 01-08; Python; comfort reading a Kubernetes
manifest, a Kyverno or OPA policy, and a JWT claim set; at least one
compliance conversation (external audit, legal review, or security
incident) whose outcome was "we cannot prove what happened"

## Overview

Security and governance is the module where the platform you have built
so far meets the auditor, the legal team, and the attacker who treats
your model-serving fleet as production infrastructure rather than a
research playground. Modules 01-08 produced a tenanted cluster, an API,
a feature store, an orchestrator, a model registry, a developer-
experience surface, and the dashboards and alerts that keep it all
running — all of which assume a trusted caller, a trusted image, a
trusted artifact, and a trusted operator. This module is where each of
those assumptions becomes an explicit control: a signed-in human whose
identity came from your IdP, a service whose credential is a workload-
identity token rather than a long-lived key, an image whose provenance
was attested in CI and verified at admission, a model artifact whose
hash is on the audit log, and a promotion to production whose gate
produced a decision record a regulator can read six months from now.
A model that answers 200 OK to the wrong tenant, an image whose
`:latest` tag quietly shifted under the registry, a Vault secret that
was last rotated the quarter the platform launched, a "bias review"
whose artifact is a Slack thread, and an audit log that lives in a
rotating logfile with no retention policy — none of these show up on
the dashboard the Module 08 learner built, and all of them show up in
the legal discovery letter the first time a model decision is
challenged. This module teaches the primitives (AuthN, AuthZ, tenancy,
secrets, supply chain) and the governance layer on top of them (model
cards, bias reviews, decision logs, audit trails, compliance
attestations) that a specifically-ML platform owes its users, its
auditors, and its oncall engineer.

Modules 01-08 assumed security and governance as a background
contract. Module 09 is where that contract gets written down. By the
end of this module you will have wired OIDC authentication plus
IdP-group-mapped RBAC into your platform API, stood up HashiCorp
Vault behind External Secrets Operator with demonstrated auto-
rotation of a database credential, produced signed provenance and
SBOMs in CI and enforced signature verification at Kubernetes
admission via Kyverno (SLSA Build Level 2), implemented a governance-
gate contract that refuses promotion to production unless a model
card, a bias review, a decision log entry, and an audit event are all
present, and authored the tooling that produces a quarterly compliance
report whose audit-log integrity is verifiable and whose "outstanding
governance debt" is measurable. The deliverables intentionally cover
one identity surface, one secrets surface, one supply-chain surface,
one governance-gate surface, and one audit/compliance surface — the
smallest complete security-and-governance footprint that a platform
team can defend to a regulator without hiring a compliance team first.

The tone of this module is deliberately operational. We assume you
can read a Kubernetes manifest, have seen an OIDC flow end to end
(authorization endpoint, token endpoint, JWKS, ID token), and have at
some point been handed an audit requirement that resolved to "we
will build it when we need it." We will spend our time on the parts
that ML platform teams consistently get wrong in year one: an API
whose tenancy check is a header the client sets rather than a claim
the server verifies, static cloud keys baked into a pod environment
and rotated by rebuild, a `cosign verify` line in CI that nothing
downstream enforces, a Kyverno policy applied in `audit` mode and
never promoted to `enforce`, a Vault token whose TTL is longer than
the engineer who created it has been on the team, a "model card"
whose fields have no schema and whose owner has left the company, an
audit log that is really the API's structured logger routed to the
same Elasticsearch cluster the metrics are in, and a compliance
posture that terminates in "we comply with our policies" because
nobody has written down what the policies are.

## Learning objectives

By the end of this module, you will be able to:

1. **Distinguish** the three concerns the platform's security layer
   routinely conflates — authentication (who is this caller?),
   authorization (what are they allowed to do?), and tenancy (what
   data do they see?) — and defend the separation to a reviewer who
   believes "check the header and move on" is a sufficient control.
2. **Choose** the right AuthN mechanism for the right surface: OIDC
   for human users (identity comes from your IdP), workload identity
   (IRSA on EKS, GKE Workload Identity, Azure AD pod identity) for
   pod → cloud-service auth, mTLS for pod-to-pod inside the mesh,
   and static API keys only as a last resort — and explain why
   "static IAM access keys baked into env vars" is a credential
   leak waiting to happen.
3. **Design** an AuthZ model that composes RBAC at the resource
   level (namespace, project, tenant) with ABAC for fine-grained
   decisions ("only the model owner can promote to prod," "only
   during business hours"), implemented against a policy engine
   like [OPA](https://www.openpolicyagent.org/) or
   [Cedar](https://www.cedarpolicy.com/) rather than hard-coded into
   handler bodies — and name the failure mode that policy-in-code
   produces (drift across services, no central audit).
4. **Implement** tenant enforcement as a server-side invariant —
   every API request carries a tenant claim, AuthN verifies the
   caller's identity, AuthZ checks that the caller's tenant matches
   the tenant on the resource, cross-tenant operations are explicit
   and audited — and explain why a trusted client header is the
   bug every multi-tenant postmortem eventually names.
5. **Secure** the supply chain end to end: produce signed provenance
   in CI via a hosted build service and
   [cosign keyless signing](https://docs.sigstore.dev/cosign/signing/overview/)
   (targeting [SLSA Build Level 2](https://slsa.dev/spec/v1.0/levels)),
   attest an SBOM generated by [syft](https://github.com/anchore/syft),
   enforce signature and attestation verification at Kubernetes
   admission via [Kyverno](https://kyverno.io/) or the
   [Sigstore Policy Controller](https://docs.sigstore.dev/policy-controller/overview/),
   and treat model artifacts (weights, adapters) as supply-chain
   objects — use safe formats like
   [safetensors](https://github.com/huggingface/safetensors) and sign
   them like images.
6. **Operate** secrets with no long-lived material on disk: deploy
   [HashiCorp Vault](https://www.vaultproject.io/) as the central
   secrets backend, project secrets into pods via
   [External Secrets Operator](https://external-secrets.io/latest/),
   configure dynamic credentials where the backend supports it (e.g.
   Vault's database secrets engine for short-lived DB users), and
   demonstrate rotation end to end — a password rotated in Vault
   produces a refreshed secret in the pod inside the ESO refresh
   interval, with no pod restart. Name the two anti-patterns that
   unwind this story (secrets committed to a chart's values file,
   token TTLs measured in months).
7. **Codify** the governance-gate contract that regulated ML
   platforms end up with: no Production promotion without a model
   card (per the spirit of the
   [Model Cards for Model Reporting](https://arxiv.org/abs/1810.03993)
   pattern), a bias / fairness review (with tooling such as
   [Fairlearn](https://fairlearn.org/) producing the artifact), a
   decision-log entry for the architectural choice behind the model,
   and an audit event capturing (actor, action, target, timestamp,
   artifacts referenced). Treat the promotion API as the enforcement
   point and the audit log as the tamper-evident ledger — write it
   append-only with hash chaining so a later rewrite is detectable.
8. **Map** the platform to a compliance posture that an auditor can
   read: the voluntary [NIST AI Risk Management
   Framework](https://www.nist.gov/itl/ai-risk-management-framework)
   as the US-side reference, the
   [EU AI Act](https://artificialintelligenceact.eu/) as the tiered
   regulatory regime that classifies systems by risk, the
   [OWASP Machine Learning Security Top 10](https://owasp.org/www-project-machine-learning-security-top-10/)
   as the attacker-surface checklist, and
   [ISO/IEC 23894:2023](https://www.iso.org/standard/77304.html) as
   the international AI risk-management standard. Explain why
   "we comply with our policies" is not a compliance posture until
   the policies are written, the gates enforce them, and the audit
   log proves it.
9. **Recognize** the failure modes that cost ML platform teams their
   security credibility — a trusted-client tenant header, static
   cloud keys in env vars, a Kyverno policy stuck in `audit` mode,
   a Vault token whose TTL has been "temporarily" extended for two
   years, a governance gate whose "bias review" field is a free-text
   URL that 404s, and an audit log whose retention is "whatever the
   log aggregator kept" — and name the guardrail that prevents each.

## What's in this Module

```
lessons/mod-009-security-governance/
├── README.md                                       (you are here)
├── lecture-notes/
│   ├── 01-authn-authz-tenancy.md
│   ├── 02-supply-chain.md
│   └── 03-governance-framework.md
├── exercises/
│   ├── exercise-01-implement-oidc.md
│   ├── exercise-02-vault-secret-rotation.md
│   ├── exercise-03-supply-chain-attestation.md
│   ├── exercise-04-governance-gates.md
│   └── exercise-05-quarterly-compliance-audit.md
├── quizzes/
│   └── module-009-quiz.md
└── resources.md
```

### Lecture Notes (read in order)

| Chapter | Title | Approx. reading time | Maps to CURRICULUM.md topic |
| --- | --- | --- | --- |
| 01 | AuthN, AuthZ, and Tenancy | 60 min | "Authentication" (2.5h) + "Authorization" (2.5h) |
| 02 | Supply Chain Security | 60 min | "Secrets Management" (1.5h) + attacker-surface framing from OWASP ML Top 10 |
| 03 | Governance Framework | 60 min | "Audit Logging" (1.5h) + "Data Privacy and Compliance" (2h) |

Chapter 01 covers the three concerns that most ML platform APIs
conflate — authentication (who is the caller?), authorization (what
are they allowed to do?), and tenancy (what data do they see?) — and
the mechanisms that answer each one at the right surface: OIDC for
human users, workload identity for pod → cloud-service auth, mTLS
for pod-to-pod inside the mesh, and static API keys only as a last
resort. It covers the AuthZ styles (RBAC at the resource level,
ABAC for fine-grained decisions, policy-based engines like OPA or
Cedar that externalize policy from handler code), the tenancy
invariant that the server verifies the tenant claim rather than
trusting a client header, and the audit obligations that every
cross-tenant operation carries. It closes with the three recurring
failure modes (AuthN confused with AuthZ, tenancy enforced in the
client, policy hard-coded into handler bodies rather than a central
engine) and the guardrail each one requires.

Chapter 02 is the supply-chain chapter. It walks the full chain —
code → CI → image → registry → Kubernetes admission → pod → pod
talking to Vault — and names the controls at each link.
[SLSA](https://slsa.dev/) is the framework we lean on (Build Level
2 is the realistic target for a platform team in year one: hosted
build service, signed provenance, source integrity).
[Sigstore](https://www.sigstore.dev/) provides the signing primitive
(`cosign` with OIDC-asserted keyless identity, no static keys to
rotate or leak),
[syft](https://github.com/anchore/syft) produces the SBOM, and
[Kyverno](https://kyverno.io/) (or the
[Sigstore Policy Controller](https://docs.sigstore.dev/policy-controller/overview/))
enforces verification at admission so an unsigned image is rejected
before it ever runs. The chapter extends the frame to model
artifacts — a compromised set of weights is as dangerous as a
compromised image, and the fix is the same: use safe serialization
(`safetensors` over `pickle`-based formats), sign the artifact when
it is published to the registry, and verify the hash at load time.
It closes with the three recurring failure modes (a `cosign verify`
line in CI that nothing downstream enforces, a Kyverno policy left
in `audit` mode and never promoted, model files loaded from an
unverified source by a job that is running with cluster
credentials) and the guardrail each one requires.
<!-- needs-research: confirm current SLSA v1.0 build-track level count and whether the "4 levels" phrasing from the existing lecture should be updated to the v1.0 three-track model; pin to slsa.dev/spec/v1.0/levels before publishing -->

Chapter 03 is the governance chapter — the process layer that sits
on top of the security primitives and converts them into an
auditable record. It covers the governance-gate set that regulated
ML platforms end up with (code review, security scan, bias review,
model card, decision log, compliance attestation), the owners that
each gate assigns (tech lead, platform, compliance, model owner),
and the enforcement point that converts a gate from a doc into a
control (the promotion API refuses the transition unless every
artifact is present and signed off). It covers the audit log
pattern — append-only with hash chaining so a later rewrite is
detectable, retention long enough to survive a compliance cycle,
schema stable enough to query — and the compliance-framework map
most platforms end up needing: the voluntary
[NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework)
on the US side, the
[EU AI Act](https://artificialintelligenceact.eu/) as the tiered
regulatory regime (low-risk tier is reachable for most internal
platforms with the gate set above plus model cards plus audit),
[ISO/IEC 23894:2023](https://www.iso.org/standard/77304.html) as
the international AI risk-management standard, and (where banking
is in scope)
[SR 11-7](https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm)
as the US Federal Reserve supervisory letter on model risk
management. It closes with the recurring failure modes (a
"governance" doc with no enforcement point, an audit log that is
really the API's structured logger with no retention, a "bias
review" whose artifact is a Slack thread, a compliance attestation
that was signed once at launch and never renewed) and the
guardrail each one requires.

### Exercises (do after the matching chapter)

| Exercise | After chapter | Estimated time | Format |
| --- | --- | --- | --- |
| 01 — Implement OIDC + RBAC | 01 | 150 min | OIDC AuthN wired into the platform API against a real IdP (Keycloak / Auth0 / Okta), IdP groups → RBAC roles → resource permissions, with 401 / 403 / 200 demonstrated for anonymous / unauthorized / authorized callers |
| 02 — Vault + ESO secret rotation | 01 | 120 min | Vault + External Secrets Operator stood up; a database password rotated in Vault refreshes in the pod inside the ESO refresh interval, with no pod restart — screenshotted end to end |
| 03 — SLSA L2 supply chain | 02 | 150 min | CI pipeline that builds + pushes an image, generates an SBOM via syft, signs with cosign keyless, attests the SBOM, and is verified at admission via a Kyverno policy — demonstrate that an unsigned image is rejected |
| 04 — Governance gates | 03 | 120 min | Production promotion wired to refuse without model card + bias review + decision log entry + audit log event; gate refusal returns a clear error naming the missing artifact |
| 05 — Quarterly compliance audit | 03 | 90 min | Tooling that produces a quarterly report: per-team model count, % with current model card, % with current bias review, audit-log integrity verification, outstanding governance debt — and the report itself |

Exercises 01 and 02 build the identity and secrets foundation; 03
is the supply-chain chain-of-custody exercise that most directly
resembles a real admission-webhook rollout (and most often fails
the first time because the Kyverno policy is scoped too narrowly
or too broadly); 04 and 05 are the governance pair that converts
the primitives from exercises 01-03 into an auditable record. The
two that most learners finish and then quietly realize are the
hardest in the module are exercise 03 (because the full SLSA L2
chain has more moving parts than any single doc covers end-to-end
and the first `kubectl apply` of the Kyverno policy is almost
always too strict) and exercise 05 (because writing a compliance
report whose audit-log integrity verification is actually
*verifiable* — not just a green checkmark — surfaces how much of
the audit story was previously held in tribal knowledge).

Several exercises reference companion templates in
`ai-infra-engineer-solutions` and the broader curriculum.
Specifically, `mod-109 ex-07` covers the Vault + ESO
implementation that backs Exercise 02 and `mod-109 ex-08` covers
the Kyverno policy set that backs the admission half of Exercise
03; `mod-103 ex-10` covers the SBOM-and-supply-chain implementation
that backs the CI half of Exercise 03; `mod-106 ex-10` covers the
model-governance template that backs Exercise 04; and
`project-4-governance` is the working composite that the quarterly-
audit tooling in Exercise 05 extends. Those templates are starting
points, not answer keys — you are expected to adapt them (changing
the IdP, policy scope, SBOM format, gate schema, and audit retention
to match your scenario) rather than copy them.
<!-- needs-research: confirm final naming / exercise numbering of mod-109, mod-103, mod-106 companions in ai-infra-engineer-solutions and the location of project-4-governance before publishing -->

### Quiz

`quizzes/module-009-quiz.md` contains 10 questions covering the
AuthN-vs-AuthZ distinction, the right mechanism for pod → cloud-
service auth, OIDC's appropriate surface, SLSA Level 2's
distinguishing requirement, cosign keyless signing, Kubernetes
admission-time image verification, the standard governance-gate
set, EU AI Act low-risk-tier compliance, audit-log properties, and
the security relevance of model artifacts in the supply chain. The
answers are embedded as checked boxes next to the correct option;
try to answer each question before you look at the next one.

A passing score is **70% (7 of 10)**. There is no separate answer
key file — the quiz is self-grading via the checkbox annotations,
and rationale for every answer is at the bottom of the quiz file.

## How to Work Through This Module

There is no single right path, but here is one that has worked for
past learners. The whole module is ~8 hours of guided study plus
~6-8 hours of exercises; most learners spread this over about a
week.

**Day 1 (1 hour)**: Read chapter 01. On paper, list the three most
privileged API endpoints on your current (or most recent) platform
and, for each one, write down which service verifies AuthN, which
service verifies AuthZ, and where the tenant claim is checked. If
any of the three resolves to "the client sends a header and we
trust it," that is itself a finding — note it for exercise 01.

**Day 2 (2.5 hours)**: Do exercise 01 (OIDC + RBAC). Pick a real
IdP you can stand up locally — Keycloak is the common open-source
choice, Auth0 and Okta have free tiers — rather than stubbing the
token issuer. The deliverable is the three demonstrations
(anonymous → 401, authenticated-but-unauthorized → 403,
authenticated-and-authorized → 200) plus the IdP-group-to-role
mapping file. Most learners discover during this exercise that
their 403 and their 401 are the same code path with a different
message and refactor the authorization check into a separate
middleware; if yours does not, make it so.

**Day 3 (2 hours)**: Do exercise 02 (Vault + ESO rotation). Stand
Vault up from the Helm chart (not a local binary — the rotation
story depends on the Kubernetes auth method), configure External
Secrets Operator to project a DB credential into a pod, then rotate
the DB password in Vault and watch ESO refresh the secret. The
deliverable is the end-to-end screenshot plus a one-paragraph note
on your refresh interval and why you picked it; most learners pick
the ESO default without reading what it is and then discover
during write-up that "15 minutes" is a choice, not a given.

**Day 4 (1 hour)**: Read chapter 02. If you have never produced a
`cosign` signature or read an SBOM, skim
[Sigstore's cosign quickstart](https://docs.sigstore.dev/cosign/signing/overview/)
and the [syft README](https://github.com/anchore/syft) before
starting exercise 03 so the pipeline you build is not the first
time you have seen either tool.

**Day 5 (2.5 hours)**: Do exercise 03 (SLSA L2 supply chain). Build
this against GitHub Actions (or the hosted CI your platform uses)
rather than a local shell script — the "hosted build service"
requirement in SLSA L2 is load-bearing, and signing from your
laptop is a different exercise. The deliverable is the CI
pipeline, the admission-webhook refusal of an unsigned image, and
a one-paragraph note on where the Kyverno policy's scope landed
(too broad flags every pod in `kube-system`; too narrow misses the
pods you care about). If you cannot produce the refusal
screenshot, the policy is not enforcing.

**Day 6 (1 hour)**: Read chapter 03. Pull up your current team's
(or any public ML platform's) promotion pipeline and ask: what
artifacts are required to go from staging to production, who
signs off, and where is that signoff recorded? If the answer is
"a Slack thread" or "the model owner's good judgement," that is
the pattern the chapter is naming.

**Day 7 (2 hours)**: Do exercise 04 (governance gates). The gate
is not a form; it is a server-side check on the promotion
endpoint. Write it so the refusal error names the specific
missing artifact (`model_card: missing`, `bias_review: stale`)
rather than a generic 403, because the whole point of the gate is
that the caller can self-correct. The deliverable is the refusal
error output plus the audit log entry that was written when the
successful promotion finally went through; both are required —
the gate without the audit is a nag, the audit without the gate
is paperwork.

**Day 8 (1.5 hours)**: Do exercise 05 (quarterly compliance
audit). The audit-log integrity verification is the discipline:
a hash-chained log whose "verify" step re-walks the chain and
produces a single pass/fail is the shape you want, not a
dashboard that counts entries. The "outstanding governance debt"
column is the one most teams underestimate — it is where you
discover that twelve models in the registry have no current model
card and three teams have never completed a bias review, and the
report is the thing that forces the conversation.

**Day 9 (1 hour)**: Take the quiz. Re-read any chapter whose
questions you had to guess at, especially chapter 02 if the SLSA
or cosign questions felt arbitrary — those are the parts most
learners skim and then cannot defend when a security reviewer
asks what "SLSA L2" actually requires.

If you are moving slowly, that is fine. Security and governance is
the module where ML platform teams most often ship a platform
whose access model works until the first real audit — depth here
pays itself back every time a legal review, an external audit, or
an incident post-mortem asks for evidence the platform can
produce rather than evidence the platform has to reconstruct.

## How to Self-Assess

After finishing the module, you should be able to do the following
without referring back to the notes:

- [ ] Distinguish AuthN from AuthZ in one sentence each and name
      the common bug conflating them produces (authorization
      decisions made against an unauthenticated principal, or
      vice versa).
- [ ] Name the four AuthN mechanisms from chapter 01 (OIDC,
      workload identity, mTLS, static API keys) and the surface
      each one is appropriate for.
- [ ] Describe the AuthZ composition pattern (RBAC at the resource
      level + ABAC for fine-grained + a policy engine like OPA or
      Cedar for central enforcement) and name the failure mode
      hard-coded policy produces.
- [ ] Write the tenancy invariant from memory (the server verifies
      a tenant claim; the client never sets a trusted tenant
      header) and give a concrete example of what the audit log
      records on a cross-tenant operation.
- [ ] Describe the SLSA L2 bar (hosted build service, signed
      provenance, source integrity) and name the signing primitive
      (cosign keyless, OIDC-asserted identity, no static keys) and
      the admission-time enforcement point (Kyverno or the
      Sigstore Policy Controller).
- [ ] Explain why model artifacts belong in the supply chain and
      name the safe serialization format (`safetensors`) that
      avoids the unsafe-deserialization risk of older `pickle`-
      based formats.
- [ ] Describe the end-to-end rotation story (Vault as the
      backend, External Secrets Operator as the projection layer,
      refresh interval as the freshness bound) and name the two
      anti-patterns that unwind it (committed secrets, long-TTL
      tokens).
- [ ] Name the six governance gates from chapter 03 (code review,
      security scan, bias review, model card, decision log,
      compliance attestation), the owner each one assigns, and the
      enforcement point that converts a gate from a doc into a
      control (the promotion API).
- [ ] Describe the audit-log properties that make the chain
      tamper-evident (append-only, hash-chained, retention past
      the compliance window, stable schema) and explain why a
      rotating application logfile does not meet the bar.
- [ ] Name the four compliance frames from chapter 03
      (NIST AI RMF, EU AI Act, ISO/IEC 23894, SR 11-7 where
      banking is in scope) and the attacker-surface checklist
      (OWASP ML Top 10) that complements them.
- [ ] Score 70%+ on the quiz without notes.

If you can do all of the above, you have finished the taught arc
of the ML Platform curriculum. If 2-3 of these feel shaky, that is
normal — security and governance is the module whose lessons most
reliably sharpen the second time you encounter them, usually when
a real audit or a real incident puts the controls through their
first production test. Revisit the gaps then.

## A Note on Tools

This module names several security, policy, and governance tools by
product:
[OpenID Connect (OIDC)](https://openid.net/connect/) and
[OAuth 2.0](https://oauth.net/2/) (the identity protocols),
[Keycloak](https://www.keycloak.org/),
[Auth0](https://auth0.com/), and
[Okta](https://www.okta.com/) (IdP options for exercise 01),
[Open Policy Agent (OPA)](https://www.openpolicyagent.org/) and
[Cedar](https://www.cedarpolicy.com/) (policy engines),
[HashiCorp Vault](https://www.vaultproject.io/) and
[External Secrets Operator](https://external-secrets.io/latest/)
(secrets backend + projection),
[Sigstore](https://www.sigstore.dev/) with
[cosign](https://docs.sigstore.dev/cosign/signing/overview/) (the
signing primitive),
[syft](https://github.com/anchore/syft) (SBOM generator),
[Kyverno](https://kyverno.io/) and the
[Sigstore Policy Controller](https://docs.sigstore.dev/policy-controller/overview/)
(admission-time verification),
[safetensors](https://github.com/huggingface/safetensors) (safe
model serialization),
[Fairlearn](https://fairlearn.org/) (fairness assessment tooling),
and [SLSA](https://slsa.dev/) (the supply-chain framework).
**You are not expected to be fluent in all of these.** The
exercises pin OIDC (via any IdP you can stand up) as the AuthN
protocol, Vault + ESO as the secrets stack, cosign + Kyverno as
the supply-chain pair, and a self-authored promotion check as the
governance-gate implementation — because the open-source footprint,
Kubernetes-native integration, and audit-trail story of this stack
are the closest match to what a self-hosted ML platform actually
runs. You can finish the module without evaluating every tool
first.

The resources list in [`resources.md`](./resources.md) is where to
go when you need to evaluate alternatives for a real deployment —
the principles here are portable; the specific tooling is not. In
particular, chapter 02 names the conditions under which this
stack is the wrong default: an org that has already standardized
on a commercial policy engine (Styra DAS wrapping OPA; AWS Cedar-
backed Verified Permissions in an AWS-locked shop), a hyperscaler-
locked team whose supply-chain contract is already AWS Signer or
Google Binary Authorization and whose procurement path does not
permit another verifier, or an air-gapped environment where cosign
keyless (which depends on reaching the public Fulcio and Rekor
endpoints) is not viable and `cosign` with an HSM-backed key is
the fallback. In each case the AuthN / AuthZ / tenancy separation,
the SLSA levels, the governance-gate set, and the audit-log shape
port unchanged; only the implementation plumbing differs.

## A Note on Platform / Engineer Split

This module lives in the ML Platform curriculum, which is the
*design and operations* track. Several exercises reference
companion implementations in `ai-infra-engineer-solutions` —
specifically `mod-109 ex-07` for the Vault + ESO secret-
management implementation that backs Exercise 02, `mod-109 ex-08`
for the Kyverno policy set that backs the admission half of
Exercise 03, `mod-103 ex-10` for the SBOM-and-supply-chain CI
template that backs the build half of Exercise 03, and
`mod-106 ex-10` for the model-governance template that backs
Exercise 04. The working composite for Exercise 05's quarterly
audit lives under `project-4-governance`. Those are the hands-on
engineer-track templates; this module leans on them rather than
reproducing them. If you want a deeper implementation pass on any
exercise, start from the companion and then come back to the
AuthN / AuthZ / tenancy separation (01), the SLSA L2 defense (03),
and the governance-gate contract (04), which are the deliverables
the engineer track does not cover.
<!-- needs-research: confirm final naming / exercise numbering of mod-109, mod-103, mod-106 companions in ai-infra-engineer-solutions and the location of project-4-governance before publishing -->

## A Note on Security and Governance Sources

The AuthN / AuthZ separation and the OIDC and OAuth 2.0 flows
used in this module come from the
[OpenID Foundation's OIDC Core 1.0 specification](https://openid.net/specs/openid-connect-core-1_0.html)
and the
[IETF OAuth 2.0 framework (RFC 6749)](https://www.rfc-editor.org/rfc/rfc6749),
which are the canonical references for the token-issuance and
authorization flows the exercise relies on. The supply-chain
framework is [SLSA](https://slsa.dev/) (Supply-chain Levels for
Software Artifacts), originated as an internal Google framework
("Binary Authorization for Borg") and now stewarded by the
[Open Source Security Foundation (OpenSSF)](https://openssf.org/).
The signing primitive is [Sigstore](https://www.sigstore.dev/),
whose keyless-signing model binds a signature to an OIDC-asserted
identity rather than a static key file; `cosign`, Fulcio, and
Rekor are the signer, CA, and transparency log that together make
the model work. The model-reporting pattern that the governance
gate codifies traces back to the
[Model Cards for Model Reporting paper (Mitchell et al., 2019)](https://arxiv.org/abs/1810.03993),
and the attacker-surface vocabulary (prompt injection, model
extraction, adversarial inputs, membership inference) follows the
[OWASP Machine Learning Security Top 10](https://owasp.org/www-project-machine-learning-security-top-10/).
The compliance frame draws on the
[NIST AI Risk Management Framework (AI RMF 1.0)](https://www.nist.gov/itl/ai-risk-management-framework)
on the US side, the
[EU AI Act](https://artificialintelligenceact.eu/) on the EU side,
[ISO/IEC 23894:2023](https://www.iso.org/standard/77304.html) as
the international AI risk-management standard, and (where
US banking is in scope)
[SR 11-7](https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm)
as the Federal Reserve supervisory letter on model risk
management. This module teaches the engineering pattern —
identity verified at the edge, policy enforced centrally,
secrets dynamic and short-lived, supply chain attested end-to-end,
governance gates implemented as server-side invariants, and an
audit log tamper-evident enough to survive discovery — that lets
a platform team hold up against a real audit rather than
reconstructing the story from logs after the fact.
<!-- needs-research: confirm current SLSA v1.0 stewardship (OpenSSF) and build-track level count before publishing; confirm NIST AI RMF latest-version pointer (1.0 vs. 1.1 drafts) -->

## Navigation

- Up: [`../../README.md`](../../README.md) — repo overview
- Curriculum spec: [`../../CURRICULUM.md`](../../CURRICULUM.md) — see
  "Module 09: Security & Governance"
- Previous: [`../mod-008-observability/`](../mod-008-observability/) — Observability & Monitoring
- Next: project-based learning — see [`../../projects/`](../../projects/) for the capstone sequence that composes Modules 01-09

Security and governance is the module where most ML platforms
either make an audit a defensible artifact — identities verified,
policies centrally enforced, secrets rotated, images attested,
promotions gated, logs tamper-evident — or discover, under the
first legal review, that the story they can tell about who did
what to which model is a Slack thread and a stale Confluence
page. Separate AuthN from AuthZ, enforce tenancy server-side,
run OIDC against a real IdP, run Vault + ESO for secrets, sign
and verify images and model artifacts, wire the governance gates
into the promotion API, and write the audit log append-only with
hash chaining — and the platform has a compliance posture it can
defend in front of an auditor rather than apologize for in front
of an incident commander.
