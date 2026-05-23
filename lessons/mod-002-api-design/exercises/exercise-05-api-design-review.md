# Exercise 05: API Design Review (Pair Exercise)

## Objective

Practice giving + receiving substantive API design review.

## Setup

Pair with a peer (or two reviewers if solo). Each of you brings:
- Your exercise-01 OpenAPI spec
- Your exercise-02 v2 design
- A 1-page rationale doc

## Review protocol

90 minutes per side:

### Before the meeting (30 min)
- Read the spec end to end
- Run `openapi-generator-cli validate`
- Generate a Python client and try calling 3 endpoints
- Write a list of every concern: design, naming, error model, missing capabilities

### During the meeting (45 min per side)
- Author walks through key design decisions
- Reviewer asks open-ended questions ("what happens if...?")
- Both note the items that the author hadn't considered
- Capture an agreed-on list of changes

### After (15 min)
- Author writes the diff
- Both sign off

## Deliverable

`REVIEW.md` for each spec covering:
- Top 5 design strengths
- Top 5 issues found
- Resolution for each issue (accept / reject / defer)
- Lessons each party takes back to future API designs

## Acceptance

- Both parties submitted reviews of the other's spec
- Each review names ≥ 5 substantive issues (not just style)
- Each issue has a recorded resolution
