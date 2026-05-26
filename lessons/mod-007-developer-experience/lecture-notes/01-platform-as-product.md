# Lecture 01: Platform as Product

## The reframe

Engineers build platforms. Products are built by product teams. Platforms-
as-product apply product discipline to engineering platforms: customer
discovery, roadmap, NPS, deprecation discipline, marketing.

## Customer discovery

The platform team's customer is other engineers (data scientists, ML
engineers, product engineers). Customer discovery looks like:
- Quarterly interviews with 5+ users about top frustrations
- Weekly office hours; track top recurring questions
- Slack-channel telemetry: which questions repeat?

You don't need a PM. You need to read your own Slack channel.

## The success metrics

Bad metrics: number of models deployed, number of pipelines.

Good metrics:
- Time from "trained" → "in prod" using platform (lead time)
- Number of platform-template adopters
- NPS (or proxy: "would you recommend the platform to a peer?")
- Self-serve %: % of new models that didn't require a platform ticket

## Avoid the gatekeeper trap

A platform team is *not* a gatekeeper. If every project must pass through
your team's queue, you become the bottleneck. The path away:
- Make self-serve easy (templates that don't require deep platform knowledge)
- Office hours instead of tickets for "I'm stuck"
- Reserve 20% of capacity for unblocking; don't promise rapid turnaround

## Bad signs

- "We're working on next quarter's roadmap" without consulting users
- Documentation older than the code
- Users describing the platform as "the team that says no"
- Velocity of platform improvements declining
