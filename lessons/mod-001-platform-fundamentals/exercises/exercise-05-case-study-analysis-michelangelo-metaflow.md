# Exercise 05: Case Study Analysis — Michelangelo vs Metaflow

## Objective

Read the two canonical ML platform case studies (Uber Michelangelo, Netflix Metaflow) closely and extract design lessons. By the end of this exercise you will have produced a side-by-side comparison, a critical reading of each, a "what they don't say" analysis, and a personal synthesis of the design lessons that would apply to your hypothetical platform.

This exercise is the most heavily reading-based of the five. The reading is the work. Budget time for it.

## Learning Outcomes

By completing this exercise, you will:

- Read two published ML platform writeups critically — not as blueprints but as design artifacts.
- Identify the implicit design choices behind each platform and the constraints those choices imply.
- Recognize what each writeup is and is not telling you.
- Synthesize design lessons that generalize across platforms.
- Write criticism that is specific and defensible, not vague.

## Prerequisites

- Read Lecture 01 (Introduction to ML Platform Engineering) — especially the case study sections.
- Read Lecture 05 (Platform Architecture Patterns) — gives you the vocabulary for the comparison.
- ~60 minutes of reading time available for the source materials (not counting your writing).
- Optional but useful: Lecture 02 (Platform Thinking) for the abstraction-and-DX lens.

## Scenario

You have been asked to write an internal RFC at Aurelia AI titled "What we should and should not learn from Michelangelo and Metaflow." The audience is the platform team plus the VP of Engineering. The point of the RFC is to inform decisions for the next year's platform roadmap.

You will spend the first half of the exercise *reading* the case studies, and the second half *writing the RFC*.

## Source Materials

The two primary sources:

1. **["Meet Michelangelo: Uber's Machine Learning Platform"](https://www.uber.com/blog/michelangelo-machine-learning-platform/)** — Uber Engineering Blog, 2017. ~6,000 words.
2. **["Open-Sourcing Metaflow"](https://netflixtechblog.com/open-sourcing-metaflow-a-human-centric-framework-for-data-science-fa72e04a5d9)** — Netflix Tech Blog, 2019. ~3,500 words.

Plus, recommended secondary materials:

3. **[Metaflow documentation](https://docs.metaflow.org/)** — at least the "Introduction" and "Basics of Metaflow" sections. ~30 minutes of reading.
4. **[Michelangelo Palette: Uber's Feature Store](https://www.uber.com/blog/michelangelo-machine-learning-platform/)** (in the same post, or follow-on posts in Uber's Engineering Blog). Skim if time allows.

If the URLs above are inaccessible (paywalls, link rot), do your best with what is available, and note in your RFC which sources you couldn't access.

## Deliverables

By the end of this exercise, you will have created:

1. `notes-michelangelo.md` — your raw reading notes on Michelangelo.
2. `notes-metaflow.md` — your raw reading notes on Metaflow.
3. `comparison.md` — the side-by-side comparison table.
4. `whats-not-said.md` — what each writeup omits and why it matters.
5. `lessons-for-aurelia.md` — the RFC itself: 5-10 lessons for our platform.

If you are tight on time, prioritize `comparison.md` and `lessons-for-aurelia.md`. The notes can be terse.

---

## Part 1: Read Michelangelo Closely (45 minutes)

### Task 1.1: First pass — what's there

Read the Michelangelo post end-to-end. As you read, take *brief* notes in `notes-michelangelo.md`. Use this structure:

```markdown
# Michelangelo Reading Notes

## Components mentioned
- (list each named component with one-sentence description)

## Workflows described
- (list each end-to-end flow the post walks through)

## Tools / frameworks named
- (e.g., XGBoost, TensorFlow, Spark)

## Scale / metrics mentioned
- (number of models, GPUs, teams, etc., if cited)

## User personas implied
- (whose problem is this platform solving?)

## Design choices stated explicitly
- (verbatim or paraphrased)
```

**TODO**: Create `notes-michelangelo.md` and fill in each section. Be concise; bullets are fine. Aim for ~200-400 lines total.

### Task 1.2: Second pass — the design choices

Read again (skimming this time), but instead of cataloging, *look for design choices*. A design choice is a place where the post says "we did X" and the alternative "we could have done Y" was a real option.

Examples of design choices to look for:

- Did Michelangelo support multiple ML frameworks or pick one? (Multiple — but why?)
- Did Michelangelo expose a Python SDK or only REST APIs?
- Did Michelangelo unify training and serving, or are they separate subsystems?
- How did Michelangelo handle feature engineering — embedded in the platform or external?
- Did Michelangelo handle distributed training, or only single-node?
- How did Michelangelo treat model versioning?

For each design choice you can identify, add a bullet to `notes-michelangelo.md` under a "Design Choices" heading:

```markdown
## Design Choices

- **Multi-framework support.** Michelangelo deliberately supports
  multiple ML frameworks (XGBoost, TensorFlow, etc.) rather than
  standardizing on one. Implied rationale: ...
```

**TODO**: Identify at least 6 explicit or implicit design choices.

### Task 1.3: What questions does the post raise?

A good critical reader of a case study notices what the post *doesn't* answer. Note questions in `notes-michelangelo.md` under "Open Questions":

- Who runs the platform team? How big is it?
- How does Michelangelo enforce multi-tenancy?
- How are GPU resources allocated when multiple teams want them?
- How does Michelangelo handle model promotion (staging → prod)?
- How does the platform respond to a bad model in production?
- How were teams migrated from pre-Michelangelo workflows?

**TODO**: List 8-12 questions the post does not answer. These will feed into `whats-not-said.md`.

---

## Part 2: Read Metaflow Closely (35 minutes)

### Task 2.1: First pass

Create `notes-metaflow.md` using the same structure as Michelangelo notes. Read the open-sourcing post end-to-end, plus the "Introduction" page of the Metaflow docs.

Focus areas (because Metaflow is more SDK-shaped, the notes will look different):

- **The flow abstraction.** What is a `FlowSpec`? What are `@step`, `@batch`, `@conda`?
- **Local-to-cloud parity.** How does Metaflow make local-run and cloud-run feel the same?
- **Versioning.** What does Metaflow auto-version?
- **Resume.** How does the resume-from-failed-step feature work?
- **Backend abstraction.** What can Metaflow execute on? (AWS Batch, Step Functions, Kubernetes, Argo, etc.)

**TODO**: Fill in `notes-metaflow.md`. Aim for ~200-400 lines.

### Task 2.2: The philosophical center

Metaflow is unusually clear about its philosophy. The phrase "human-centric framework for data science" is repeated.

In `notes-metaflow.md`, write a section "Stated Philosophy" with verbatim quotes (where you can find them) and your paraphrase. Then write 2-3 sentences on what this philosophy implies for the design.

Example structure:

```markdown
## Stated Philosophy

> "Metaflow takes a human-centric approach to data science tooling..."
> — Netflix Tech Blog

This implies:
- The user's code (their Python class) is the primary artifact, not
  the platform's manifests.
- Versioning is automatic, not opt-in (so users can't forget).
- Local execution must feel identical to cloud execution, so users
  don't context-switch.
```

**TODO**: Write this section.

### Task 2.3: What Metaflow doesn't do

Metaflow's philosophy commits it to certain things and *excludes* others. What does Metaflow not try to do?

Some candidates:

- It does not include a feature store.
- It does not include a model serving fleet.
- It does not enforce multi-tenant isolation (it relies on the underlying compute backend).
- It does not provide a built-in UI for monitoring (until later additions).

**TODO**: List at least 5 things Metaflow deliberately does not do. For each, note whether you think this is a strength or a limitation, and why.

---

## Part 3: Comparison Table (15 minutes)

Create `comparison.md`. Build a side-by-side comparison table.

```markdown
# Michelangelo vs Metaflow

## Side-by-side comparison

| Dimension                  | Michelangelo (Uber)         | Metaflow (Netflix)            |
| -------------------------- | --------------------------- | ----------------------------- |
| Primary user surface       | Services + workflow UI      | Python SDK                    |
| Opinionatedness            | Unified workflow, multi-fw  | Flow abstraction, multi-backend |
| ML framework support       | XGBoost, TF, PyTorch, ...   | Any (no framework opinion)    |
| Model registry             | Built-in                    | Implicit (artifacts in flow)  |
| Feature store              | Palette (built-in)          | Not provided                  |
| Serving / inference        | Built-in                    | Not provided                  |
| Distributed training       | Supported (Horovod, etc.)   | Supported (via decorators)    |
| Local-to-prod parity       | (not emphasized)            | First-class                   |
| Versioning                 | Explicit (register model)   | Implicit (auto for flow artifacts) |
| Multi-tenancy              | (not emphasized publicly)   | Inherited from backend        |
| Open source                | No                          | Yes (since 2019)              |
| Year published             | 2017                        | 2019                          |
| Target user persona        | ML engineers + ops          | Data scientists               |
| Workflow unit              | Job (training, batch, inference) | Flow (DAG of steps)      |
```

**TODO**: Fill in or extend this table. Try to find at least 12 dimensions to compare.

### Task 3.2: Visualize the architecture difference

Below the table, sketch the two architectures side-by-side in ASCII:

```markdown
## Architecture shapes

### Michelangelo (heavy services)

User → Web UI / CLI
        ↓
       APIs
        ↓
[Train svc] → [Registry] → [Deploy svc] → [Inference svc]
                  ↓
            [Monitoring svc]

### Metaflow (Python SDK + thin backend)

User writes FlowSpec in Python
        ↓
   `python my_flow.py run`
        ↓
   Metaflow client (decorators decide where each step runs)
        ↓
[Local] | [AWS Batch] | [Kubernetes] | [Step Functions] | [Airflow]
        ↓
   Artifacts stored in S3
   Metadata stored in metadata service
```

**TODO**: Refine these sketches based on what you actually read.

---

## Part 4: What's Not Said (20 minutes)

Create `whats-not-said.md`. This is the most important deliverable; do not skip.

### Task 4.1: Michelangelo's omissions

Both posts are *marketing* artifacts (in the broadest sense — they exist to make their company look good and to recruit). Marketing posts have systematic omissions. Identify Michelangelo's:

```markdown
## Michelangelo omissions

### Multi-tenancy
The 2017 post does not describe how Uber's many ML teams shared Michelangelo
without stepping on each other. Quotas, RBAC, cost allocation, and noisy-neighbor
mitigation are all absent. Yet at Uber scale (~50+ ML teams by 2017), these
must have existed. What's the implication?

### Migration
The post does not describe how teams moved off their pre-Michelangelo bespoke
pipelines onto the platform. This is usually 50-70% of the effort of building
a platform. We can infer it was hard but cannot read the lessons.

### Team structure
The post does not say how many engineers built and ran Michelangelo. We can
infer it's nontrivial (dozens of people based on the breadth of features) but
the precise numbers and team topology are absent.

### Failures
The post does not describe what Michelangelo *failed* at, what they got wrong,
what teams refused to use, what they wished they had done differently. Every
mature platform has these stories; this post has none.

...
```

**TODO**: Identify at least 5 categories of omission and write a paragraph on each.

### Task 4.2: Metaflow's omissions

```markdown
## Metaflow omissions

### Serving
Metaflow does not include model serving. Netflix presumably has serving
infrastructure, but it's not part of Metaflow. The implicit message: Metaflow
solves training and data-science workflow; serving is somebody else's problem.

### Feature stores
Same: no built-in feature store. Netflix likely has one elsewhere.

### Multi-tenancy
Metaflow leans on the backend (Kubernetes, AWS) for isolation. This works at
Netflix where every Metaflow user is presumably an internal employee with
existing AWS credentials, but it's silent on tenant boundaries.

...
```

**TODO**: Identify at least 5 omissions in Metaflow. For each, hypothesize *why* the omission exists.

### Task 4.3: What the omissions tell you

Pull back and synthesize:

```markdown
## What the omissions tell us

Both posts emphasize the *technical artifacts* (workflows, abstractions,
APIs) and underplay the *socio-organizational* work (multi-tenancy, team
structure, migration, failure recovery). This is the systematic blind spot
of published platform writeups. As a reader, the lesson is:

- *Most of the cost of building a platform is invisible in the writeup.*
- *Successful platforms are 30% technical and 70% socio-organizational; the
  posts cover the 30%.*
- *Reading "we built X" should always prompt "what did the team have to do
  besides build X?"*

For Aurelia, this means we should plan for the 70% even though we have no
case-study guide for it.
```

**TODO**: Write this synthesis.

---

## Part 5: Lessons for Aurelia (40 minutes)

Now the RFC itself: `lessons-for-aurelia.md`. Aim for 5-10 specific, defensible lessons.

### Task 5.1: Structure

Use this skeleton:

```markdown
# RFC: Lessons from Michelangelo and Metaflow for Aurelia's ML Platform

**Author**: [you]
**Date**: [today]
**Status**: Draft for discussion

## Background

The Aurelia platform team is planning the next year's roadmap. To inform
those plans, we read the two canonical published ML platform writeups
(Uber Michelangelo, Netflix Metaflow) and extracted lessons.

The lessons below are *not* a recommendation to clone either platform.
Aurelia is a Series B company with ~30 ML engineers; both Uber and
Netflix were vastly larger when they wrote these posts. The lessons are
calibrated to our scale.

## Lessons

### Lesson 1: ...

(2-4 paragraphs)

### Lesson 2: ...

(...)

## What we should NOT copy

(2-4 anti-lessons — things from the writeups that are wrong for us)

## Roadmap implications

(What this implies for the next 6 months of work)

## Open questions for the team

(Things we couldn't decide from the reading alone)
```

### Task 5.2: Candidate lessons

Below are some lessons you *might* arrive at. Don't just copy these — use them as starters, and pick the ones that are actually defensible after your reading.

#### Lesson candidates (pick ~5-8 to elaborate)

**Candidate A: Build the unified workflow before the unified code.**

Michelangelo's insight: don't try to make all ML teams use the same training code; make them use the same *workflow*. Submit → train → register → deploy → monitor — that's the spine. The model code itself can vary by framework.

For Aurelia: in our first year, the platform should enforce *workflow* consistency (all teams register models the same way, deploy through the same pipeline, monitor with the same metrics) but allow *implementation* variety. We are too small to mandate a single ML framework.

**Candidate B: Local-to-cloud parity is worth investing in early.**

Metaflow's insight: data scientists want their local run and their cloud run to feel the same. A decorator changes execution location; the code doesn't.

For Aurelia: building this is hard but pays off. If a data scientist has to rewrite their code to deploy, they won't deploy.

**Candidate C: The Python SDK is the front door.**

Metaflow's insight: data scientists live in Python. The platform should meet them there. The REST APIs are an implementation detail.

For Aurelia: prioritize the SDK. The CLI and Web UI can lag.

**Candidate D: Multi-framework support is value, not chaos.**

Michelangelo's insight: not picking a single ML framework is a feature. Teams pick what fits their problem; the platform supports them.

For Aurelia: our small size makes this even more important — we cannot mandate a framework on 6 teams that all use what fits.

**Candidate E: Monitoring is part of the platform, not an afterthought.**

Both Michelangelo and (later) Metaflow treat model-quality monitoring as first-class.

For Aurelia: build monitoring into the model-deployment workflow from v1. Models that ship without monitoring should be the exception, not the default.

**Candidate F: Open-source-first changes the cost-benefit.**

Metaflow open-sourced after a few years of internal use. This:
- Increases the audience of users and bugfixers.
- Forces clean abstractions (you can't expose internal APIs).
- Becomes a recruiting and PR asset.

For Aurelia: we are too small to open-source from day one, but we should build in a way that *would be* open-sourceable later — clean abstractions, no Aurelia-specific assumptions baked into the public-ish surface.

**Candidate G: Treat your platform as a product, not a project.**

Both writeups, read between the lines, show platforms that evolved continuously based on user feedback. They are not "shipped once and done."

For Aurelia: budget for ongoing iteration. Plan to ship something every quarter, not "deliver the platform in 18 months."

**Candidate H: The writeups underplay the people work — do not.**

Multi-tenancy, migration, team structure, change management — all absent from the writeups. All critical. We should over-invest here relative to what the case studies suggest.

### Task 5.3: Write 5-10 lessons

**TODO**: Pick at least 5 lessons (some from the list above, some of your own). For each, write:

- A one-sentence summary.
- 2-4 paragraphs of defense and implication for Aurelia.
- A specific action item: "What we should do in the next 6 months because of this."

Aim for `lessons-for-aurelia.md` to be 300-600 lines total.

### Task 5.4: What NOT to copy

Both writeups have things you *should not* copy. Examples:

- Michelangelo's scale assumptions don't fit a Series B.
- Metaflow's reliance on AWS Batch may not fit a company on GCP.
- Both posts predate modern Kubernetes maturity; their compute scheduling stories are out of date.
- Neither addresses LLM/foundation-model workloads (the posts predate the GenAI surge); some assumptions may not transfer.

**TODO**: Write at least 3 "do not copy" items in `lessons-for-aurelia.md`.

### Task 5.5: Open questions

Conclude with 5-8 open questions the team should debate.

Examples:

- Should we expose a Python SDK as the primary front door (Metaflow path) or as a wrapper on REST APIs (Michelangelo-ish)?
- Should we build a feature store or buy/integrate Feast?
- Should we standardize on Kubernetes or remain backend-agnostic like Metaflow?
- How will we measure platform adoption?
- When do we deprecate shadow-IT alternatives?

**TODO**: Write these.

---

## Part 6: Critical Self-Check (10 minutes)

Before finalizing, review your work against these prompts. Write your answers in a "Self-Critique" section at the bottom of `lessons-for-aurelia.md`.

1. **Am I treating these posts as authoritative or as artifacts?** A common failure of case-study reading is to treat them as ground truth. They are not — they are stories told by their authors at a specific time. Did I read critically?
2. **Have I generalized beyond what the posts say?** Some of my "lessons" might be projections, not things I can actually defend from the source. Flag those.
3. **Is my advice specific or vague?** "Treat the platform as a product" is vague. "Add a quarterly user-survey ritual managed by the platform lead" is specific. Bias toward specific.
4. **Am I using the writeups to justify what I already believed?** Confirmation bias is the case-study-reader's enemy. Look for places where the source surprised you, not just where it confirmed you.
5. **Did I cite specific phrases or stop at general gestures?** Where possible, link or quote the specific passage I'm drawing on.

**TODO**: Answer each in 1-2 sentences in `lessons-for-aurelia.md`.

---

## Common Pitfalls

1. **Reading too quickly.** Each post is dense. Re-read.
2. **Conflating Michelangelo with "the Uber ML platform" in 2026.** The 2017 post is a snapshot; Uber's platform has evolved enormously since. Don't assume the post still describes Uber's reality.
3. **Treating the posts as engineering specs.** They are not; they are public communications. Lots of detail is omitted intentionally.
4. **Picking one platform as "the right one."** They are different. Both are interesting. Aurelia is a third thing, neither of them.
5. **Skipping `whats-not-said.md`.** This is the most valuable deliverable; the omissions are where the real lessons live.
6. **Writing lessons that are not actionable.** Every lesson should imply a *thing to do* in the next 6 months at Aurelia.

---

## Reflection Questions

In `lessons-for-aurelia.md` under a "Reflection" heading:

1. If you had to choose: would you rather work on Michelangelo (services-heavy, framework-internal) or Metaflow (SDK-heavy, open-source-friendly)? Why?
2. Which writeup felt more honest to you? Where did you feel each one was selling?
3. What is the *most surprising thing* you learned in this reading?
4. What single decision do you think *each* platform got most right?
5. What single decision do you think *each* platform probably regrets, in hindsight?

---

## Self-Assessment

- [ ] Can I sketch each platform's architecture from memory?
- [ ] Can I name 5 things each platform deliberately did, and 3 things each one didn't?
- [ ] Did I write at least 5 specific, actionable lessons for Aurelia?
- [ ] Did I identify at least 3 things Aurelia should *not* copy?
- [ ] Am I confident my self-critique honestly flags my projections vs my reading?

If yes to all, you're done.

---

## Suggested Time Allocation

| Section | Time |
| --- | --- |
| Part 1: Read Michelangelo | 45 min |
| Part 2: Read Metaflow | 35 min |
| Part 3: Comparison | 15 min |
| Part 4: What's Not Said | 20 min |
| Part 5: Lessons for Aurelia | 40 min |
| Part 6: Critical Self-Check | 10 min |
| **Total** | **165 min** |

This is the longest exercise in Module 01 because the reading is part of the work. If you're constrained, prioritize:

- Reading Michelangelo (45 min)
- Reading Metaflow's intro (25 min)
- Comparison table (15 min)
- Lessons RFC (40 min)

Total 125 min; skip `whats-not-said.md` only if you must.

---

## Where to Go from Here

You now have a defensible, source-grounded set of lessons for an ML platform of Aurelia's size. These should inform Module 02 (API Design — where you'll go deeper on Metaflow's SDK-as-platform pattern) and Module 03 (Kubernetes — where you'll see how Michelangelo's services-heavy approach maps onto modern Kubernetes).

If you go on to write platform RFCs in your career, this exercise is the *template*: source, comparison, critique, recommendation, open questions. Most influential engineering decisions are made in artifacts shaped like this one.

Push your deliverables to your fork and take a break. The quiz comes next.

---

## Appendix A: A worked partial RFC (for reference, not for copying)

If you finish your RFC and want to compare against a sample, here is a *partial* example. This is one defensible answer, not the only one.

### Sample Lesson 1: The unified workflow is the platform's spine

Michelangelo's most quotable insight is that the *workflow* — submit → train → register → deploy → predict → monitor — is the platform's spine, not the choice of ML framework. Teams write their model code in whatever framework fits the problem; the platform binds them to a consistent lifecycle.

For Aurelia, this means our v1 platform should *not* mandate a single ML framework (we should be polite about XGBoost and TensorFlow and PyTorch alike), but should mandate that every model goes through the same registration, the same canary deployment, the same monitoring. The opinion is the workflow, not the algorithm.

Concrete action items for the next 6 months:
- Codify the 6-stage workflow as the platform's product specification.
- Build the registry first (Q2). Without a registry, no other stage works.
- Build the canary deployment workflow second (Q3). Without canary, monitoring has no on-ramp.
- Defer training-runtime work to Q4. Most Aurelia teams have their own training scripts; we don't need to standardize those before standardizing the lifecycle.

Weakness: this approach assumes teams are willing to register their models. Some won't be. We need a carrot-only first year (no mandates) and assume only 60-70% adoption by year-end.

### Sample Lesson 2: Build the SDK before the UI

Metaflow's lesson — *the Python SDK is the front door* — is especially true for an ML-engineer audience.

For Aurelia: every API surface gets a Python wrapper in the same release. We do not build a web UI for at least 12 months. The CLI is a thin wrapper over the SDK.

Concrete action items:
- Ship the SDK alongside the registry API (Q2).
- Generate the SDK from the OpenAPI spec; do not maintain it by hand.
- The CLI is a 200-line wrapper that calls the SDK; the platform team writes it in a week.
- The web UI is a Q4-or-later project.

Weakness: SDK-first means non-Python teams (we have one Go team and one Rust team) have a slightly worse experience. We accept this; their volume is low.

### Sample anti-lesson: Don't replicate Michelangelo's apparent monolith

Re-reading the 2017 Michelangelo post, it reads as a *single, large, vertically-integrated platform*. At Aurelia's scale (Series B, 30 engineers, 4-person platform team), we cannot operate a system that large. We will build the modular monolith from Lecture 05, with a few satellites — not the full Michelangelo.

Concrete implication: our roadmap should *not* try to ship feature-stores, model-registries, canary-deployment, drift-monitoring, and explainability-tooling all in year one. Pick two; defer the rest.

### Sample open question: GCP or AWS or multi-cloud?

We have not yet decided whether the Aurelia platform commits to GCP, AWS, or aims to be cloud-neutral. Each choice has implications:

- *GCP commit*: simpler operations; less optionality. Vertex AI integration is straightforward.
- *AWS commit*: same, with SageMaker.
- *Cloud-neutral*: Metaflow's pattern. More work for the platform team; lets product teams self-host or migrate.

This is a foundational decision; we should debate before Q1 ends.

---

## Appendix B: Suggested re-reading after the rest of the curriculum

After completing Modules 02-05 (API, Kubernetes, training, serving), come back to the Michelangelo and Metaflow writeups. You will read them differently the second time.

Specifically, after Module 03 you will read multi-tenancy concerns *into* the posts (which barely mention them). After Module 04 you will recognize specific training-orchestration patterns. After Module 08 you will critique their serving architectures with a more informed eye.

The case studies become a living reference. The first read is calibration; the re-reads are where the depth lives.

---

## Appendix C: Other ML platforms worth reading about (briefly)

The Michelangelo / Metaflow comparison is the *canonical* one but it is not the only one worth knowing. Time permitting, glance at:

- **LinkedIn's Pro-ML and DARWIN.** Several public talks; emphasizes feature engineering pipelines and embeddings.
- **Airbnb's Bighead.** Now wound down; the public writeups (~2017-2019) are useful as a "what looked good but didn't survive" story.
- **Spotify's TFX-based ML platform.** Heavy use of TensorFlow Extended; useful for understanding the TFX-centric school.
- **Lyft's LyftLearn.** Useful for the Airflow-centric school.
- **Databricks' MLflow + Unity Catalog story.** The most-developed commercial ML platform; useful as a comparison point.
- **AWS SageMaker.** The most-deployed managed ML platform. Reading the SageMaker docs as a *product designer* (not a user) is informative.
- **Google's Vertex AI.** Similar to SageMaker, with GCP-flavor.

You do not need to know all of these. Knowing 2-3 in addition to Michelangelo/Metaflow is plenty for most platform-engineering contexts.

---

## Appendix D: Rubric for self-grading your RFC

If you want to self-grade your `lessons-for-aurelia.md`:

| Criterion | Strong | Adequate | Weak |
| --- | --- | --- | --- |
| Lessons grounded in sources | Cites specific passages | Paraphrases the source | Generic, source-less |
| Lessons actionable | Concrete next-quarter action | Vague intent | Aspirational only |
| Alternatives considered | Explicit alternatives w/ trade-offs | Mentioned briefly | None |
| Weaknesses surfaced | At least one per lesson | Some lessons have weaknesses | None |
| Audience awareness | Pitched to platform team + VP | Pitched to one or the other | Unclear audience |
| Calibrated to Aurelia | Reflects Series-B, 30-engineer scale | Reflects some scale | Reads like a Fortune 500 plan |

If you scored "Strong" on 4+ rows, you're done. If you scored "Adequate" or "Weak" on more than 2, revise.

> **Source note.** Metaflow was open-sourced by Netflix in late 2019 (see the ["Open-Sourcing Metaflow"](https://netflixtechblog.com/open-sourcing-metaflow-a-human-centric-framework-for-data-science-fa72e04a5d9) post). The Michelangelo framework support enumerated in lecture-notes/01 (XGBoost, TensorFlow, PyTorch) is paraphrased from the 2017 Uber Engineering post; verify the exact framework list there if you plan to reuse this in a written deliverable. Verified 2026-05.
