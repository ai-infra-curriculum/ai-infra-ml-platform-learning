# Lecture 02: Versioning and Deprecation

## Why versions exist

An API exists in time. The shape you ship today is the shape you owe consumers
to support for years. The promise is: "if you write code against this version,
I will not break you without notice."

Versioning is the mechanism for breaking that promise gracefully.

## Three versioning schemes

### URL versioning (recommended default)

```
/v1/training-jobs
/v2/training-jobs
```

- Pros: visible, easy to route, easy to operate independently
- Cons: feels heavyweight; clients hard-code the version

### Header versioning

```
GET /training-jobs
Accept: application/vnd.ml.v2+json
```

- Pros: same URL across versions
- Cons: less discoverable; harder to operate

### Field-level evolution (no version)

Add new optional fields, deprecate old fields with a sunset header, never
remove. Works only if the changes are purely additive.

- Pros: no version proliferation
- Cons: the API gets crufty; impossible for breaking changes

**Pick URL versioning unless you have a strong reason otherwise.**

## Semver for APIs

Treat the version like semantic versioning:
- **Major** (v1 → v2): breaking change; clients must opt in
- **Minor**: additive within a major version (new optional field, new endpoint)
- **Patch**: bug fixes, no API surface change

In practice, only majors appear in the URL. Minors are documented in the
changelog.

## Deprecation playbook

A well-behaved API deprecates a version like this:

| T+ | Action |
|---|---|
| 0 | Announce v2 GA; v1 marked deprecated in docs |
| 0 | Add `Deprecation: true` + `Sunset: <date>` headers to v1 responses |
| 90d | Send active v1 users an email + Slack DM (telemetry tells you who) |
| 180d | Add a banner to console + a deprecation warning on every v1 response |
| 270d | Reach out to remaining users individually |
| 365d | Remove v1 with 30-day notice; provide a migration script |

Time horizons depend on your contract: internal API = months, external API
serving customers = years.

## Migration support

- **Codegen migration guide**: if you ship an SDK, the new version of the SDK works against both old + new server versions for a transition period
- **Migration tool**: a script that consumes v1 manifests and produces v2 equivalents
- **Office hours during transition**: weekly drop-in for stuck users

## Backwards-compatibility patterns

Within a major version, these are safe:
- Adding optional fields to requests
- Adding fields to responses (clients should tolerate unknown fields)
- Adding new endpoints
- Loosening validation (accept what you previously rejected)

These are breaking:
- Removing fields
- Changing field types
- Tightening validation
- Renaming endpoints
- Changing semantics of existing fields

When in doubt, the test is: would a client written six months ago still work?

## Recording the rules

Document:
- Your versioning scheme (one paragraph)
- Your deprecation timeline (the table above, adapted)
- What changes are considered breaking vs additive
- Where the changelog lives

Tie it to your release process so you can't ship an undocumented breaking
change by accident.
