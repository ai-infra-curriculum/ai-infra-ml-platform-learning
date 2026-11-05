# Module 07: Developer Experience & Tooling

**Duration**: ~8 hours guided + 6-8 hours exercises
**Prerequisites**: Modules 01-06; Python; comfort reading an OpenAPI spec and
a Click or Typer CLI; at least one piece of platform documentation that
drove you back to Slack instead of answering your question

## Overview

Developer experience is the module where the platform you have built so far
meets the humans who have to use it. Modules 01-06 produced a tenanted
cluster, a REST + gRPC API surface, a feature store, an orchestrator, and a
model registry — all of which are useless if the first thing a new data
scientist does on Monday morning is file a ticket asking what any of it is
called, how to authenticate, and why the error message they got said
`Error: 403`. This module covers the parts of a platform that do not show
up in an architecture diagram — SDK ergonomics, documentation shape, CLI
affordances, error-message anatomy, onboarding, and the feedback loops
that keep the first four from rotting — and treats them as first-class
engineering problems rather than something to hand to a technical writer
at the end of the quarter.

Modules 01-06 were about what the platform *does*. Module 07 is about
whether anyone can tell. By the end of this module you will have run
customer-discovery interviews against a real or hypothetical platform,
built a Click-based CLI with structured output and the four-part error
message pattern, written a quickstart that an actual stranger can follow
without getting stuck, audited and rewritten a set of production error
messages, and designed a quarterly NPS survey that produces roadmap-
grade signal instead of vanity scores. The deliverables are deliberately
asymmetric: three of them are writing exercises, because the typical ML
platform team under-invests in writing and ships surfaces no one outside
the team can decode.

The tone of this module is deliberately operational. We assume you can
read a Python SDK and have run `--help` on a CLI with more than ten
subcommands, even if you then closed the terminal. We will spend our
time on the parts that ML platform teams consistently get wrong in year
one: a "quickstart" that assumes the reader has already stood the
platform up, an SDK whose errors propagate as `requests.HTTPError: 500`
with no additional context, a CLI that returns `OK` on success and a
stack trace on failure, a documentation site that is really three wikis
in a trench coat, a feedback mechanism that is a single `#platform-help`
Slack channel moderated by whoever sees the ping first, and a success
metric that counts "pipelines deployed" rather than "hours saved per
user per week."

## Learning objectives

By the end of this module, you will be able to:

1. **Distinguish** developer experience from end-user experience — same
   practices (user research, success metrics, deprecation discipline,
   onboarding), different user population (other engineers inside the
   org, not end customers) — and defend the shift in writing to a
   leader who believes a platform team "shouldn't need product
   management."
2. **Run** a customer-discovery cycle against a platform's users:
   quarterly 30-minute structured interviews with 3-5 representative
   teams, ongoing Slack / ticket telemetry on recurring questions, and
   a synthesis step that separates feature gaps ("the platform does not
   do X") from knowledge gaps ("the platform does X, but no one knows
   where to look") — because the fix is different.
3. **Choose** platform success metrics that survive a leadership review:
   lead time from "trained" to "in production," self-serve percentage
   (new models shipped without a platform ticket), template-adopter
   count, and NPS or a proxy ("would you recommend this platform to a
   peer on another team?"). Explain why "number of pipelines deployed"
   and "lines of code in the platform" are vanity metrics.
4. **Design** documentation along the [Diátaxis
   framework](https://diataxis.fr/) — tutorials, how-tos, reference, and
   explanation — and explain why most platform teams ship only
   reference and wonder why users still can't onboard. Treat docs as
   infrastructure: versioned alongside code, built in CI, link-checked
   on a cadence, with tested code examples.
5. **Build** a hierarchical Click or Typer CLI with sane defaults,
   `--output json|yaml|table` on every read command, scriptable
   composability (pipeable to `jq`), shell completion for bash and zsh,
   and command-level telemetry — and name the three CLI anti-patterns
   that most frustrate power users (flat command surface, single hard-
   coded output format, uninformative exit codes).
6. **Write** error messages to the four-part anatomy (**Error** —
   what failed, **Cause** — why, **Help** — the next action,
   **Docs** — where to read more) at every layer the user touches: the
   API response body, the SDK exception, and the CLI stderr output —
   and demonstrate that the same underlying failure produces a
   coherent message at all three layers rather than a stack trace at
   one and a status code at another.
7. **Operate** a platform-as-product feedback loop: a 5-minute
   quickstart that always works against the current release, office
   hours instead of a ticket queue, a quarterly NPS survey whose open-
   ended question actually gets read, and a deprecation policy that
   gives users more notice than the platform team wishes it had to.
8. **Recognize** the failure modes that cost platform teams their
   adoption credibility — the gatekeeper trap (every project routes
   through the platform team's queue), documentation that is older
   than the code, SDK exceptions that leak HTTP status codes without
   context, CLIs that return `0` on silent partial failure, and the
   "we don't need a PM" posture that quietly means "we don't do
   customer research" — and name the guardrail that prevents each.

## What's in this Module

```
lessons/mod-007-developer-experience/
├── README.md                                       (you are here)
├── lecture-notes/
│   ├── 01-platform-as-product.md
│   ├── 02-documentation-strategy.md
│   └── 03-cli-and-error-messages.md
├── exercises/
│   ├── exercise-01-customer-interviews.md
│   ├── exercise-02-build-a-cli.md
│   ├── exercise-03-tutorial-write-up.md
│   ├── exercise-04-error-message-audit.md
│   └── exercise-05-nps-survey-design.md
├── quizzes/
│   └── module-007-quiz.md
└── resources.md
```

### Lecture Notes (read in order)

| Chapter | Title | Approx. reading time | Maps to CURRICULUM.md topic |
| --- | --- | --- | --- |
| 01 | Platform as Product | 60 min | "Developer Experience Principles" (2h) + "Platform Adoption" (1.5h) |
| 02 | Documentation Strategy | 60 min | "Documentation and Tutorials" (2h) |
| 03 | CLIs and Error Messages | 60 min | "SDK Design" (2.5h) + "CLI Tool Development" (2h) |

Chapter 01 covers the platform-as-product reframe — the platform team's
customer is other engineers, and the discipline that works there is the
same product discipline that works for an external customer, applied to
an internal user population. It covers customer discovery (quarterly
interviews, ongoing Slack / ticket telemetry, the "feature gap vs.
knowledge gap" distinction), the success metrics that survive a
leadership review (lead time, self-serve rate, template adopters, NPS)
versus the vanity metrics that do not (pipelines deployed, platform
headcount, lines of code), and the gatekeeper trap: the pattern where
every team routes through your queue, you become the bottleneck, and the
only way out is self-serve templates plus office hours rather than more
headcount.

Chapter 02 is the documentation chapter. It covers the [Diátaxis
framework](https://diataxis.fr/) — tutorials (learning), how-tos
(tasks), reference (precise lookup), and explanation (conceptual) — and
names the most common failure: shipping reference only and treating the
other three as "someone else's job." It covers the five tiers most
platforms end up needing (5-minute quickstart, 30-minute tutorials,
task-sized how-tos, generated API reference, architecture explanation),
the discipline that keeps them current (docs versioned with code, CI
builds docs, tested examples, scheduled link-check, embedded page
analytics), and the anti-patterns that most predict a docs disaster
(mixed sources of truth across wiki + Notion + Confluence + README,
pages older than the code they describe, doc PRs that bypass review).

Chapter 03 is the surface-area chapter. It covers the three client
surfaces a platform ships — UI, SDK, CLI — and why the CLI is the one
that disproportionately decides whether power users become champions.
It covers hierarchical command design (`ml jobs submit` /
`ml models promote`), the output-format contract
(`--output json|yaml|table` on every read command, so humans and
scripts share the same tool), sane defaults (80% of invocations should
need zero flags), and the four-part error-message anatomy (**Error** /
**Cause** / **Help** / **Docs**) that transforms `Error: 403` into
something a user can act on. It closes with the CLI telemetry pattern
(anonymous command + duration + error-type + CLI version) that surfaces
the commands your users actually hit and the flags they never touch.

### Exercises (do after the matching chapter)

| Exercise | After chapter | Estimated time | Format |
| --- | --- | --- | --- |
| 01 — Customer discovery interviews | 01 | 120 min | 1-page synthesis ranking top 5 platform investments by user pain |
| 02 — Build a CLI | 03 | 180 min | Click-based CLI against the Module 02 API + `--output json/yaml/table` + 4-part errors + bash/zsh completion |
| 03 — Tutorial write-up | 02 | 90 min | ≤800-word "train your first model" tutorial + user-test log + iteration |
| 04 — Error message audit | 03 | 120 min | PR rewriting ≥10 error messages into the 4-part shape + "fix in code vs. docs" column |
| 05 — NPS survey design | 01 | 90 min | `SURVEY_PLAN.md` (5 questions, sampling, how results drive roadmap) + peer review |

Exercises 01 and 05 build the customer-discovery and feedback-loop
foundation; 02 and 04 are the surface-area pair that most directly
resembles a real platform-team ticket (build the CLI; then audit the
errors it produces); 03 is the writing exercise that most learners
underestimate — the eight-hundred-word budget on the tutorial is the
whole point, because that is roughly what a new user will read before
giving up. Exercise 03 (tutorial write-up) and Exercise 04 (error
message audit) are the two that most learners finish and then quietly
realize are the hardest in the module — the tutorial because writing
something a stranger can follow without getting stuck requires a
discipline most engineers do not practice, and the error-message audit
because catalog work surfaces how often the team's own surface fails
its own standard.

Several exercises reference companion templates in
`ai-infra-engineer-solutions`. Specifically, `mod-107 ex-02` covers the
Click-based CLI implementation that backs Exercise 02, and `mod-107 ex-04`
has the error-message audit worked against a real SDK that backs
Exercise 04. Those templates are starting points, not answer keys —
you are expected to adapt them (changing the API surface, error
catalog, and output formats to match your scenario) rather than copy
them.
<!-- needs-research: confirm final naming / exercise numbering of mod-107 companion in ai-infra-engineer-solutions before publishing -->

### Quiz

`quizzes/module-007-quiz.md` contains 10 questions covering platform-as-
product success metrics, the Diátaxis four-quadrant model, error-message
anatomy, customer-discovery discipline, the gatekeeper trap, CLI output
contracts, docs-as-infrastructure, CLI telemetry, the first thing to
ship in a new platform, and the cost of stale documentation. The
answers are embedded as checked boxes next to the correct option; try
to answer each question before you look at the next one.

A passing score is **70% (7 of 10)**. There is no separate answer key
file — the quiz is self-grading via the checkbox annotations, and
rationale for every answer is at the bottom of the quiz file.

## How to Work Through This Module

There is no single right path, but here is one that has worked for past
learners. The whole module is ~8 hours of guided study plus ~6-8 hours
of exercises; most learners spread this over about a week.

**Day 1 (1 hour)**: Read chapter 01. On paper, write down the three
metrics your current (or most recent) platform team reports upward,
then annotate each one with whether it is a lead-time / self-serve /
adoption metric, or a vanity metric. If more than one is vanity, that
is itself a finding — note it for exercise 05.

**Day 2 (2 hours)**: Do exercise 01 (customer discovery interviews).
If you cannot interview real users, pick three archetypes (a data
scientist shipping their first model, an ML engineer on a mature
team, a platform-adjacent engineer debugging a production incident)
and run the interview against the archetype. The deliverable is the
ranked investment list, not the interview transcripts — the point of
the exercise is forcing synthesis across users rather than treating
any single user's complaint as the roadmap.

**Day 3 (1 hour)**: Read chapter 02. Pull up your current team's docs
site (or any platform's public docs you have recently tried to use)
and classify the top ten pages against Diátaxis: tutorial, how-to,
reference, or explanation. If more than seven of them are reference,
that is the pattern the module is naming.

**Day 4 (1.5 hours)**: Do exercise 03 (tutorial write-up). The 800-
word cap is the point of the exercise — it forces the author to cut
everything the reader does not need in the first hour. Have a real
person follow it; log every place they got stuck; iterate once. The
user-test log is the deliverable; a tutorial that has never been
followed by a stranger is not done.

**Day 5 (1 hour)**: Read chapter 03. If you have never used Click or
Typer, skim the quickstart in [`resources.md`](./resources.md) before
starting exercise 02; the goal of the exercise is CLI design, not
Click boilerplate.

**Day 6 (3 hours)**: Do exercise 02 (build a CLI). Build against the
API you shipped in Module 02 so the CLI is exercising a real surface
with real error modes. Hit the five commands (`submit`, `list`,
`status`, `cancel`, `retry`) with all three output formats, trip at
least three failure modes on purpose, and screenshot the error output
to confirm the four-part shape. Completion scripts are not optional —
power users will not adopt a CLI they have to type out in full.

**Day 7 (2 hours)**: Do exercise 04 (error message audit). The
`fix in code vs. docs` column is the deliverable that separates a
cleanup PR from a one-off rewrite — half of what shows up in a
real audit is a message that needs to point at documentation that
does not yet exist, and the audit is where you discover that.

**Day 8 (1.5 hours)**: Do exercise 05 (NPS survey design). The open-
ended question ("what would most help your work?") is the one that
produces roadmap-grade signal — the numeric score is the one that
produces headline quotes. Both matter; the survey design has to
support both. Defend your five questions in a peer review before
you call the exercise done.

**Day 9 (1 hour)**: Take the quiz. Re-read any chapter whose
questions you had to guess at, especially chapter 01 if the success-
metric or gatekeeper questions felt arbitrary — those are the parts
most learners skim and then cannot defend when a VP asks what
"platform success" means this quarter.

If you are moving slowly, that is fine. Developer experience is the
module where platform teams most often ship an architecturally
excellent platform that no one adopts, discover six months later that
three teams are quietly running their own parallel tooling, and then
blame "lack of buy-in" — depth here pays itself back every quarter
the platform has to compete for adoption against a team's own shell
scripts.

## How to Self-Assess

After finishing the module, you should be able to do the following without
referring back to the notes:

- [ ] Distinguish a feature gap from a knowledge gap and describe the
      different fix each one calls for.
- [ ] Name the four platform success metrics from chapter 01 (lead
      time, self-serve rate, template adopters, NPS or proxy) and the
      vanity metrics each one replaces.
- [ ] Describe the gatekeeper trap in one sentence and name the two
      mechanisms (self-serve templates, office hours instead of
      tickets) that get you out of it.
- [ ] Name the four Diátaxis quadrants and describe the reader
      question each one answers.
- [ ] Explain what "docs as infrastructure" requires in practice
      (versioning with code, CI build, tested examples, link-check
      cadence, page analytics).
- [ ] Write the four-part error message anatomy (Error / Cause /
      Help / Docs) from memory and give a concrete example rewriting
      `Error: 403` into it.
- [ ] Describe the CLI output-format contract
      (`--output json|yaml|table`) and explain why a CLI without it
      cannot serve both power users and scripts.
- [ ] Describe the CLI telemetry payload (command, duration, error
      type, version) and name two decisions that payload unlocks
      which usage estimates cannot.
- [ ] Explain why the first thing to ship in a new platform is a
      5-minute quickstart, not the internal data model or the full
      CLI surface.
- [ ] Score 70%+ on the quiz without notes.

If you can do all of the above, you are ready for Module 08. If 2-3 of
these feel shaky, that is normal — flag them and revisit when you
encounter the relevant concept again in later modules, especially
when Module 08 (observability) wires platform-level telemetry into the
same product-metric dashboards named here, and Module 09 (security &
governance) adds the deprecation and change-communication discipline
that the platform-as-product posture assumes.

## A Note on Tools

This module names several SDK, CLI, and documentation tools by product:
[Click](https://click.palletsprojects.com/),
[Typer](https://typer.tiangolo.com/),
[FastAPI](https://fastapi.tiangolo.com/) (SDK generation via its
OpenAPI output),
[MkDocs Material](https://squidfunk.github.io/mkdocs-material/),
[Docusaurus](https://docusaurus.io/),
[Sphinx](https://www.sphinx-doc.org/),
[Backstage](https://backstage.io/) (internal developer portal), and
[the Diátaxis framework](https://diataxis.fr/) (not a tool, a
structure). **You are not expected to be fluent in all of these.**
The exercises pin Click as the CLI framework (because its hierarchical
group / command model maps directly to the subcommand surface this
module teaches, and the Python ecosystem's muscle memory for it is
the deepest) and Diátaxis as the documentation structure (because it
is framework-neutral and survives swapping MkDocs for Docusaurus or
Sphinx without being rewritten), so you can finish the module without
evaluating every tool first.

The resources list in [`resources.md`](./resources.md) is where to go
when you need to evaluate alternatives for a real deployment — the
principles here are portable; the specific tooling is not. In
particular, chapter 03 names the conditions under which Click is the
wrong default: a team already standardized on Typer (whose type-hint-
driven signature is harder to read for CLI novices but produces
better SDK symmetry), or an org whose CLIs are already shipped as Go
binaries via Cobra (Kubernetes / Argo / Flux shops), in which case the
CLI design principles port but the implementation idioms do not.

## A Note on Platform / Engineer Split

This module lives in the ML Platform curriculum, which is the *design
and operations* track. Several exercises reference companion
implementations in `ai-infra-engineer-solutions` — specifically
`mod-107 ex-02` for the Click-based CLI implementation and
`mod-107 ex-04` for the error-message audit against a worked SDK.
Those are the hands-on engineer-track templates; this module leans on
them rather than reproducing them. If you want a deeper implementation
pass on any exercise, start from the companion and then come back to
the customer-discovery synthesis (01), the tutorial write-up (03), and
the NPS survey design (05), which are the deliverables the engineer
track does not cover.
<!-- needs-research: confirm final naming / exercise numbering of mod-107 companion in ai-infra-engineer-solutions before publishing -->

## A Note on Platform-as-Product Sources

The platform-as-product framing used in this module traces back to
[Team Topologies](https://teamtopologies.com/) (Skelton & Pais, 2019)
and the operating patterns in [Camille Fournier's *Platform
Engineering: A Guide for Technical, Product, and People
Leaders*](https://www.oreilly.com/library/view/platform-engineering/9781098153632/).
The documentation framework is [Diátaxis](https://diataxis.fr/) (Daniele
Procida), which has become the de facto structural vocabulary across the
Python documentation ecosystem. The CLI-design principles draw on
[the *Command Line Interface Guidelines*](https://clig.dev/), a modern
restatement of the Unix command-line conventions in a form that is
easier to hand to a new CLI author than the original Bell Labs papers.
This module teaches the engineering pattern — product discipline
applied to an internal user population, with measurable feedback loops
— that lets a platform team sustain adoption rather than relying on
mandate.
<!-- needs-research: confirm Camille Fournier's "Platform Engineering" publication year + ISBN before citing to learners -->

## Navigation

- Up: [`../../README.md`](../../README.md) — repo overview
- Curriculum spec: [`../../CURRICULUM.md`](../../CURRICULUM.md) — see
  "Module 07: Developer Experience & Tooling"
- Previous: [`../mod-006-model-management/`](../mod-006-model-management/) — Model Management & Registry
- Next: [`../mod-008-observability/`](../mod-008-observability/) — Observability & Monitoring

Developer experience is the module where most ML platforms either make
adoption a measurable, defensible outcome or discover, two quarters in,
that teams are quietly routing around the platform because the quickstart
was broken on Tuesday and the error messages did not survive contact with
a user. Run the interviews, ship the quickstart, write the errors in four
parts, wire the telemetry, and the rest of the platform has an adoption
story it can defend in a staffing review.
