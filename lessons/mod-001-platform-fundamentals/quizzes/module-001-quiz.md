# Module 01 Quiz: Platform Fundamentals

> Instructions: 18 questions total. Mix of multiple-choice (8), short-answer (6), and applied-scenario (4). Take a full pass without looking up answers. Then self-grade against `module-001-quiz-answers.md`. A passing score is 75%+ on MCQ + defensible answers on at least 3 of 4 scenarios. There is no time limit, but most learners complete this in 60-90 minutes.

> When grading: short-answer and scenario responses are graded by *defensibility*, not by exact wording. The answer key shows one defensible answer; yours may be different and still correct.

---

## Section A: Multiple Choice (8 questions, 1 pt each)

### Q1. Definition

Which of the following is the *best* definition of an ML platform, per Lecture 01?

A. A managed cloud service that hosts trained ML models.

B. A reusable, opinionated, self-service substrate that lets internal teams accomplish a class of ML tasks without re-inventing the supporting machinery each time.

C. The set of tools (MLflow, Feast, Kubeflow) that an organization has chosen to standardize on.

D. A Kubernetes cluster running ML workloads.

---

### Q2. Platform vs infrastructure

Which statement best distinguishes a platform from infrastructure?

A. Platforms run on Kubernetes; infrastructure runs on bare metal.

B. Platforms are opinionated and self-service; infrastructure provides raw capacity that platforms (and others) consume.

C. Infrastructure is open-source; platforms are proprietary.

D. Platforms are user-facing; infrastructure is back-end.

---

### Q3. Multi-tenancy isolation dimensions

Lecture 03 enumerates seven dimensions of multi-tenancy. Which of the following is *not* one of them?

A. Compute isolation

B. Network isolation

C. UI customization

D. Cost allocation

---

### Q4. Backward compatibility

Which of the following changes to a REST API is *most likely* to break clients, even if they appear "additive"?

A. Adding a new optional query parameter with a sensible default.

B. Adding a new top-level field to a response body.

C. Adding a new enum value to an existing enum field in a response.

D. Adding a new endpoint at a new URL path.

---

### Q5. The "rule of three"

The "rule of three" heuristic, as discussed in Lecture 02, says you should:

A. Always ship a v3 of your API before declaring stability.

B. Have at least three platform engineers on the team before building any extension points.

C. Not extract a shared abstraction until you have three real concrete consumers for it.

D. Triple-check all production deployments before promoting them.

---

### Q6. ResourceQuota vs LimitRange

A team writes a pod spec with no resource requests or limits and applies it to a namespace that has both a ResourceQuota (capping namespace-total resources) and a LimitRange (with `default` and `defaultRequest`). What happens?

A. The pod is rejected because it doesn't specify resources.

B. The pod is created, with resources filled in from the LimitRange defaults; quota is checked against the filled-in values.

C. The pod is created with zero resource requests.

D. The pod is created with the ResourceQuota's `hard` values.

---

### Q7. API versioning style

Which API versioning approach is the recommended default for an *internal* ML platform's REST API?

A. URL path versioning (`/v1/training-jobs`, `/v2/training-jobs`).

B. Date-based versioning (`Stripe-Version: 2026-04-30`).

C. No versioning; just keep the latest version live and require all clients to update simultaneously.

D. Semver query parameter (`?api_version=1.2.3`).

---

### Q8. Control plane vs data plane

In an ML platform, which of the following is *most clearly* a data-plane concern (not a control-plane concern)?

A. The model registry's metadata database holding "model X version Y was deployed at time Z."

B. The inference service serving 10,000 predictions per second to production traffic.

C. The training-job submission API receiving a POST from a data scientist.

D. The tenant onboarding workflow that creates a new namespace.

---

## Section B: Short Answer (6 questions, 2 pts each)

For each, write 3-5 sentences. The answer key shows one defensible answer; yours may differ.

### Q9. Distinguish a platform from a pipeline.

A pipeline (e.g., an Airflow DAG that runs nightly to refresh customer features) is sometimes called a "platform" colloquially. Lecture 01 argues they are different. Briefly explain the distinction. Use at least one specific characteristic to anchor your answer.

---

### Q10. The "Kubernetes passthrough" anti-pattern.

In your own words, what is the "Kubernetes passthrough" anti-pattern? Give one specific symptom and one specific mitigation.

---

### Q11. Self-service vs full-service.

You are designing the workflow for "tenant requests a new dataset access grant." Should this be self-service, full-service, or gated self-service? Defend your choice with reference to the criteria in Lecture 02.

---

### Q12. Why namespaces are not a security boundary.

Lecture 03 says "namespaces are not a kernel-level security boundary." What does this mean in practice? What additional Kubernetes-layer controls would you add for *hard* multi-tenancy?

---

### Q13. Why idempotency keys exist.

Lecture 04 mentions `Idempotency-Key` headers. Explain in 3-5 sentences why this header exists, what problem it solves, and what behavior the platform should exhibit when the same key is submitted with *different* request bodies.

---

### Q14. The platform-as-product mindset.

What does it mean to treat your platform "as a product"? Name two specific practices that follow from this framing (e.g., something you would *do* as platform engineer that you would *not* do if you only thought of yourself as a backend engineer).

---

## Section C: Applied Scenarios (4 questions, 5 pts each)

For each scenario, you are given a situation and asked to make a decision. Write 1-2 paragraphs covering your decision, the alternatives you considered, and at least one weakness of your decision.

### Q15. Scenario: The Overworked CI Pipeline

You are the lead engineer of a 4-person platform team at "Helio AI," a Series A company with 8 ML engineers. Today, every model deployment requires a platform engineer to manually merge a PR in a "deployments" repo, watch the CI run for 25 minutes, and verify the canary metrics before promoting. The platform team handles 6-10 such deployments per week. This consumes ~20% of the team's capacity. The ML engineers complain about the long turnaround.

A senior engineer on your team proposes building "DeployBot," a Slack-bot-driven workflow where ML engineers type `/deploy model-X-v17 to staging`, the bot auto-merges the PR, runs CI, and auto-promotes if canary metrics are healthy. The bot would take ~6 weeks to build.

Should you build DeployBot? What alternatives would you consider? What weaknesses does your decision have?

(Defend at least one decision; mention at least two alternatives; surface at least one weakness.)

---

### Q16. Scenario: The GPU-Hungry Tenant

Your platform serves 10 internal teams. Team `vision-research` has a `standard` tier quota of 4 GPUs but is queueing 30+ training jobs per day, often waiting hours. They request an upgrade to the `premium` tier (16 GPUs).

You investigate:
- Their utilization data shows they consume 4 GPUs about 8 hours per day on average; the queueing is bursty.
- The org's GPU budget is fully committed; the platform team cannot add new hardware this quarter.
- Other tenants have higher utilization (~60%) and would suffer if you reallocated capacity to vision-research.
- Vision-research's VP has emailed your VP threatening to spin up shadow GPU instances in a separate AWS account.

How do you respond? Cover: what you say to vision-research, what you say to your VP, what (if anything) you change about the platform's policies, and what you avoid doing.

---

### Q17. Scenario: The Breaking-Change Pressure

A staff engineer on your platform team proposes renaming a field in your public API: `training_run.exit_status` → `training_run.status`. Their argument: every other API in your platform uses `status`; the inconsistency is confusing for new users. The change is a breaking change.

You have ~40 ML engineers using the v1 API. Roughly 12 of them have references to `exit_status` in production code. Your platform shipped v1 about 14 months ago.

How do you handle this? Cover: whether you accept the proposal, how you communicate the change (if you make it), what timeline you choose, and how you handle the 12 affected engineers.

---

### Q18. Scenario: The Whiteboard Architecture Review

You are interviewing for a Director-of-Platform-Engineering role at "Coronet Robotics," a series-D company with ~80 ML engineers across 12 teams. In the technical interview, you're asked: "If you joined Coronet tomorrow, and we told you the existing ML platform is six bespoke pipelines plus a wiki page listing 14 tools, what would you do in your first 90 days?"

Outline your answer. You don't need to be exhaustive; cover the 3-5 most important moves. Reference at least one concept from each of Lectures 01, 02, and 03.

---

## End of Quiz

Total points: 8 (MCQ) + 12 (short answer) + 20 (scenario) = **40 points**.

**Passing:**
- MCQ: at least 6/8 = 75%.
- Short answer: at least 8/12, *and* no single answer scored 0.
- Scenario: defensible answers on at least 3 of 4. A "defensible" answer means: makes a choice, considers alternatives, surfaces a weakness.

Self-grade against `module-001-quiz-answers.md`. Don't peek until you've finished all 18 questions.

---

## Honest-effort guideline

If you find yourself stuck on more than 3 MCQ or unable to write 3 sentences on a short-answer, return to the corresponding lecture before grading. The point of this quiz is calibration, not measurement.

If you complete the quiz and find that you got 100%, that's also a signal — perhaps the quiz is too easy for you (skip ahead to Module 02), or perhaps the quiz didn't probe what would actually trip you up (let us know).

When done, see the answer key in `module-001-quiz-answers.md`.
