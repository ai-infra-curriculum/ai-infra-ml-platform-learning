# Lecture 02: Documentation Strategy

## The Diátaxis framework

Four types of documentation, each with a different shape:

| Type | Reader's question | Form |
|---|---|---|
| Tutorial | "I want to learn" | Step-by-step, hand-holdy |
| How-to | "I have a task to do" | Recipe |
| Reference | "I need precise details" | Exhaustive, scannable |
| Explanation | "Help me understand" | Conceptual, narrative |

Most platform teams have only reference. The other three are usually thin.

## Tiers

- **Quickstart**: 5-minute path to a working first deploy
- **Tutorials**: 30-minute lessons on common workflows
- **How-tos**: short recipes for specific tasks ("how to add a custom metric")
- **Reference**: API docs (generated from spec)
- **Explanation**: architecture pages, design rationale

## Documentation as infrastructure

- Versioned with the code
- CI fails if docs don't build
- Code examples that are tested
- Automatic stale-link detection
- Embedded analytics: which pages get visits, which trigger searches

## Anti-patterns

- "We have great docs" — but the team can't find them
- Wiki-rot: pages no one updates
- Mixed sources of truth (wiki + Notion + Confluence + README)
- Docs that show what the code does but never why
- Doc PRs that merge without review

## Workflow

- Treat doc PRs the same as code PRs
- Require docs for every customer-facing change
- Quarterly doc audit (find broken links, stale content)
- Survey users: rate the docs out of 10
