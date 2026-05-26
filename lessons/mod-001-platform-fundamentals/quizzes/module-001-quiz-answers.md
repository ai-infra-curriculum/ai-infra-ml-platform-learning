# Module 01 Quiz: Answer Key with Rationales

> Read this *after* attempting the quiz. Each question's answer is followed by a rationale and (where applicable) the lecture that grounds it.

---

## Section A: Multiple Choice

### Q1. Answer: **B**

The definition we use throughout Module 01 is "a reusable, opinionated, self-service substrate that lets internal teams accomplish a class of tasks without re-inventing the supporting machinery each time" (Lecture 01).

- A is too narrow — model hosting is *one* capability of some platforms but not the definition.
- C confuses "the tools we use" with "the platform"; a tooling pile is an anti-pattern, not a platform.
- D conflates infrastructure (Kubernetes) with platform.

### Q2. Answer: **B**

From Lecture 01's "Platform vs Infrastructure" section: infrastructure provides raw capacity; platforms are opinionated, self-service abstractions on top.

- A is mechanically false; many platforms run on bare metal, and infrastructure can run on Kubernetes.
- C is irrelevant — many platforms are open-source (Metaflow), many infrastructure products are proprietary.
- D conflates user-visibility with structural role. End-users typically don't touch the platform directly; they use applications built on it.

### Q3. Answer: **C**

The seven dimensions in Lecture 03 are: compute, storage, network, identity, authorization, observability, cost. UI customization is not on this list. Lecture 02 mentions DX, but UI customization is not an *isolation* concern.

### Q4. Answer: **C**

Lecture 04's "subtly breaking changes" section calls this out explicitly. Adding an enum value can break clients that exhaustively match on the enum (case `RUNNING`, case `SUCCEEDED`, case `FAILED`, default: error).

- A is safe with sensible default.
- B is safe under "tolerant readers" — clients should ignore unknown fields.
- D is unambiguously safe; new endpoint, no impact on existing ones.

### Q5. Answer: **C**

The rule of three (Lecture 02) says don't extract a shared abstraction until you have three real concrete consumers. The point is to avoid premature generalization.

- A is unrelated to the rule.
- B is invented.
- D is unrelated.

### Q6. Answer: **B**

This is the *intended interaction* of LimitRange and ResourceQuota. LimitRange fills in defaults for missing fields; ResourceQuota then evaluates against the filled-in values. (Lecture 03, "Resource Quotas and LimitRanges" section.)

If there were a ResourceQuota *but no LimitRange*, the pod would be rejected (A). The presence of LimitRange is what makes the rejection unnecessary.

### Q7. Answer: **A**

URL path versioning is the recommended default for internal ML platforms (Lecture 04). It is visible, easy to route, easy to debug.

- B is brilliant but expensive; reserve for external SaaS APIs with broad audiences.
- C is reckless; clients have to update in lockstep.
- D works in principle but is easily dropped by intermediaries and less visible.

### Q8. Answer: **B**

The inference service is the canonical data-plane component (Lecture 05). It handles runtime traffic at high throughput; the others (registry DB, submission API, onboarding workflow) are all control-plane.

A subtle point: even the inference service has a *control-plane* aspect (which model version is deployed?). But the *serving traffic itself* is data-plane.

---

## Section B: Short Answer

### Q9. Distinguish platform from pipeline.

**Defensible answer:**
A pipeline is a *workflow* — typically a DAG of steps that runs on a schedule or trigger to produce a specific output. A platform is the substrate that lets *multiple teams* build, deploy, and operate many pipelines (and other workloads) without each team rebuilding the supporting machinery.

A pipeline serves one specific business workflow (e.g., refresh churn features nightly). A platform serves a *class of workflows* (any team can submit any kind of training/serving workload through the platform).

Anchoring example: Airflow on its own is a pipeline tool; an internal Airflow + permission system + cost-allocation + deployment patterns is the beginning of a platform.

**Common mistakes:**
- Saying "pipelines are simple, platforms are complex" — that's a sense of scale, not a structural distinction.
- Saying "platforms run pipelines" — true but doesn't capture the *substrate vs workload* distinction.

---

### Q10. Kubernetes-passthrough.

**Defensible answer:**
The Kubernetes-passthrough anti-pattern is when the "ML platform" is essentially a thin layer over raw Kubernetes — users still have to author manifests, manage namespaces, configure RBAC, set resource requests, etc. The platform adds *almost no abstraction* over the underlying infrastructure.

**Symptom:** A user asks the platform team "how do I deploy a model?" and the answer involves writing a YAML manifest. A data scientist who came from Pandas-and-Jupyter cannot succeed without learning Kubernetes; the platform has not absorbed that complexity.

**Mitigation:** The platform must hide YAML behind opinionated APIs. A Python SDK call like `platform.deploy(model)` should produce the manifest internally; the user never sees it.

---

### Q11. Self-service vs full-service for dataset access grants.

**Defensible answer:**
Dataset access grants should be *gated self-service* — the user requests via the platform UI, the request is automatically validated (correct format, requester is a member of the requesting team, etc.), and a *human approver* (the dataset owner or a delegate) clicks to approve.

Why not fully self-service: granting access to data has *high blast radius* if the dataset is sensitive; approval requires judgment about whether the requester's stated purpose is appropriate. Lecture 02 calls out "judgment-required, trust-laden" actions as poor fits for full automation.

Why not fully full-service (ticketed): the request structure is consistent enough to be expressed as a form; the routing to an approver is mechanical; only the *decision* needs a human. Building a gated self-service workflow saves ticket overhead while keeping the human judgment that matters.

---

### Q12. Namespaces are not a kernel-level security boundary.

**Defensible answer:**
Kubernetes namespaces *scope names* (pods, services, configmaps) — they prevent name collisions and provide a unit for RBAC and policy. They do *not* prevent a pod that has escaped its container (via a kernel exploit, an excessive `hostPath` mount, etc.) from affecting other pods on the same node.

In practice: two namespaces' pods can land on the same kernel-shared node and share CPU, memory, network, etc. at the OS level.

**Additional controls for hard multi-tenancy:**
- Separate node pools per tenant (taints and tolerations).
- Pod Security Admission set to `restricted` (or at least `baseline`).
- A runtime sandbox like gVisor or Kata (each pod gets its own micro-VM).
- Image admission control (only trusted images can run).
- Possibly multi-cluster, where each tenant gets its own cluster.

---

### Q13. Idempotency keys.

**Defensible answer:**
`Idempotency-Key` exists so a *client retry* doesn't cause a duplicate side effect on the server. If a client POSTs `/training-jobs` but the response is lost (network blip), they retry — without idempotency, they get a second job. With an idempotency key, the server recognizes the second POST has the same key and returns the cached result without re-creating.

When the *same key* arrives with a *different body*, the server should reject with `409` (idempotency-key conflict). This catches a bug: the client thought it was retrying but actually sent different data. Silent acceptance would corrupt state.

---

### Q14. Platform-as-product.

**Defensible answer:**
Treating the platform as a product means recognizing that you have *users* (other engineers), the platform has *competition* (shadow IT), it must be *adopted* (success is measured in adoption, not in features shipped), and it must *evolve* in response to user feedback.

**Two practices that follow:**
- Running periodic user research sessions (interviews, surveys, watching users use the platform live) and shipping changes based on what you learn.
- Measuring adoption, time-to-first-value, and self-service ratio, and treating those numbers as primary KPIs alongside reliability metrics.

A backend engineer might focus on SLO compliance, feature throughput, and code quality alone. A platform engineer must also focus on whether anyone is *using* what they built and whether those users are happy.

---

## Section C: Applied Scenarios

These are graded on *defensibility*, not on agreement with the answer key. A different answer that names alternatives and surfaces a weakness is fine.

### Q15. The Overworked CI Pipeline — sample defensible answer

**Decision:** Yes, build something like DeployBot, but spend 1-2 weeks on a *minimum viable version* first, not 6 weeks on the full design.

**MVP version:** A CLI command (or simpler bot command) that automates the PR-merge step *only*. ML engineers still review canary metrics by hand. This captures 60% of the platform-team's time-saving in 10% of the build cost.

**Alternatives considered:**
1. *Do nothing.* Saves engineering time but the friction persists; ML engineers route around the platform (build shadow deployments).
2. *Hire one more engineer to handle deployments.* Linear scaling; doesn't fix the underlying problem.
3. *Full DeployBot now (6 weeks).* Big project; defers benefit; risk of over-engineering for the canary-metric automation.

**Weakness of my decision:** The MVP doesn't address the canary verification time (still 25 min) — that's the bulk of the wait. Future iterations need to either parallelize verification or trust automated canary metrics. Also, the MVP has lower social signal — ML engineers may not perceive the platform as having improved if the wait time is unchanged.

---

### Q16. The GPU-Hungry Tenant — sample defensible answer

**Response to vision-research:**
- Acknowledge the pain; the 30-job queue is real and we want to address it.
- Decline the permanent tier upgrade (would deny other tenants).
- Offer (a) a *temporary 7-day quota increase* of +4 GPUs for their highest-priority work this week; (b) a *priority-class* upgrade where their production-critical jobs preempt low-priority work from other tenants; (c) help re-architecting their training to use fewer GPUs more efficiently (batch sizes, gradient accumulation).
- Be specific about timelines: "Next quarter, the GPU budget grows by X; we will revisit tier upgrades then."

**To your VP:**
- Brief on the situation, including the shadow-GPU threat.
- Explain why immediate tier upgrade would harm other tenants (math from the cost/quota model).
- Propose the mitigations above.
- Surface the broader pattern: GPU contention is going to recur; we need a longer-term capacity plan.

**Platform policy change:**
- Introduce *burst credits* — small per-tenant pool of GPUs available for short windows above their tier, on a use-it-or-lose-it basis. Lets tenants handle their own bursty workloads without permanent upgrades.

**What to avoid:**
- Quietly stealing capacity from other tenants. (Trust-destroying.)
- Pretending nothing is wrong. (The VP threat is real and reasonable for vision-research from their POV.)
- Permanently increasing their quota without process. (Sets precedent; corrodes the system.)

---

### Q17. The Breaking-Change Pressure — sample defensible answer

**Decision:** I accept the *goal* of consistent naming, but I reject the *specific* "rename in v1" proposal. Instead:

1. Ship v2 of the API with `status` as the new canonical field name. Keep `exit_status` *also* in v1 responses for backward compat.
2. Document the rename as a deprecation: v1's `exit_status` is deprecated; v2 uses `status`.
3. Set a deprecation timeline: v1 supported for at least 12 more months. (We are at 14 months; total v1 lifetime ~2.5 years.)
4. Reach out personally to the 12 affected engineers, share the migration guide, offer to help.
5. After 12 months, when usage is approximately zero, sunset v1.

**Why not rename in v1:** The proposal violates backward compatibility for a stylistic preference. It would break 12 engineers' production code for a cosmetic improvement. Lecture 04 is explicit: "never silently change behavior" and "deprecation policy matters."

**Why accept the underlying goal:** The staff engineer is right that consistency matters; v2 is the right place to fix it.

**Communication:**
- Release notes calling out the rename and the deprecation timeline.
- `Deprecation` and `Sunset` HTTP headers on v1 responses with the new field name.
- Direct outreach to the 12 affected engineers.
- Periodic reminders as the sunset date approaches.

**Weakness of my decision:** v2 is now permanently slightly inconsistent with v1 (both fields exist in v1, only one in v2). Some clients will write code that handles both fields awkwardly. Mitigation: clear docs and SDK helpers.

---

### Q18. The Whiteboard Architecture Review — sample defensible answer

**My first 90 days at Coronet:**

**Day 1-30: Listen.**
- Meet with each of the 12 ML teams; understand their workflows, pain points, what they wish existed.
- Read the wiki page; for each of the 14 tools, who uses it, why, and how often?
- Talk to SRE and security partners.
- *Lecture 01 reference:* avoid the "platform team building what excites them, not what users need" trap; do real user research first.

**Day 30-60: One paved road.**
- Identify the *single most painful* workflow that >50% of teams share. (Likely: "deploy a trained model into production.")
- Build the paved road for that one workflow end-to-end, working with 1-2 *design partner* teams (Lecture 02: Phase 1 of adoption).
- Don't try to fix the other 13 tools yet; that's later.

**Day 60-90: Foundations.**
- Tenancy model. Coronet has 12 teams; namespace-per-team with quotas, RBAC, NetworkPolicy. (Lecture 03 reference.)
- Cost showback. Coronet's 80 engineers are not seeing their cost; even basic visibility will surface a 30% spend reduction.
- A platform-team product roadmap, shared with the engineering org, treating the platform as a product (Lecture 02: platform-as-product).

**Things I won't try in 90 days:**
- Replacing all 14 tools at once. (Too aggressive; will fail.)
- Building a feature store from scratch. (Adopt Feast or similar.)
- Setting up multi-cluster tenancy. (Too early; namespaces are enough.)

**Key risk:**
- The 14 tools have entrenched users. Replacing them is political work, not just engineering work. I need exec sponsorship before any tool is sunsetted.

---

## Grading Guidance

### Section A (MCQ)
- 7-8 correct: strong. Move on.
- 5-6 correct: passing. Re-read the corresponding lectures for any you missed.
- ≤4 correct: regroup. Re-read Lectures 01-04 before moving on to Module 02.

### Section B (Short Answer)
- Defensible answer with at least one specific example/anchor → full credit (2 pt).
- Defensible answer but too generic ("platforms are products") → partial (1 pt).
- Missing the point of the question → 0 pt.

Sum your short-answer points (max 12). 8+ is passing.

### Section C (Scenario)
For each scenario, you get full credit (5 pt) if your response:

1. Makes a clear decision.
2. Names at least two alternatives you considered.
3. Surfaces at least one weakness of your chosen decision.

If you only do 1-2 of these, give yourself partial credit (2-3 pts). If you only restated the scenario without making a decision, 0 pts.

Defensible answers may differ substantially from the key. The key is one defense; yours may be another. What matters is the *quality of the reasoning*, not the specific decision.

### Passing the quiz

- MCQ: ≥ 6/8
- Short answer: ≥ 8/12
- Scenario: ≥ 3 of 4 scenarios with full credit (or equivalent total)

If you pass: move on to Module 02.

If you don't pass: identify the weakest section, re-read the corresponding lecture(s), and try the quiz again after a day. Don't simply re-attempt the same questions; do the lecture reading first.

---

## Common patterns in mistakes (from past learners)

- **Confusing platform with infrastructure.** Re-read Lecture 01.
- **Forgetting that adding an enum value is breaking.** Re-read Lecture 04 "subtly breaking changes."
- **Missing that LimitRange + ResourceQuota work together.** Re-read Lecture 03.
- **Treating multi-tenancy isolation as one dimension instead of seven.** Re-read Lecture 03 "Isolation Dimensions."
- **Scenario answers that don't surface a weakness.** Re-read Lecture 02 "Common Cognitive Traps" — every decision has costs; naming them is part of the discipline.

---

End of answer key. Welcome to platform engineering. Module 02 starts next.
