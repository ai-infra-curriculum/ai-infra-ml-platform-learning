# Module 02: API Design for ML Platforms

**Duration**: ~10 hours of guided study + 8-12 hours of exercises
**Prerequisites**: Module 01; comfort with HTTP + OpenAPI; some FastAPI or Flask exposure

## Overview

Module 01 established the conceptual frame: a platform is a product whose users
are other engineers. Module 02 is about the surface that connects platform
users to platform capabilities — the API.

Bad APIs strand the rest of the platform. Even a well-architected serving stack
behind a confusing API will get reluctant adoption. Conversely, a thoughtfully-
designed API can paper over an awkward backend for years.

This module covers REST + async + SDK + versioning + deprecation, with worked
examples for ML platform surfaces (training jobs, registry, feature lookup).

## Learning objectives

By the end of this module, you will be able to:

1. **Choose** between REST, gRPC, async patterns, and AsyncAPI based on use case.
2. **Design** versioned APIs that survive 3+ year contract evolution.
3. **Implement** OpenAPI-spec-first contracts with code generation.
4. **Add** idempotency, pagination, filtering, sparse fieldsets, and rate limiting.
5. **Build** an SDK on top of your API for one popular language (Python).
6. **Operate** an API: SLOs, contract testing, deprecation playbook, support model.

## Module structure

```
mod-002-api-design/
├── README.md
├── lecture-notes/
│   ├── 01-rest-vs-grpc-vs-async.md
│   ├── 02-versioning-and-deprecation.md
│   └── 03-from-spec-to-sdk.md
├── exercises/  (5 exercises)
├── quizzes/    (quiz + answers)
└── resources.md
```

## Cross-references

Module 02 in `ai-infra-engineer-learning/mod-101` covers FastAPI mechanics at
the implementation level. This module covers the design-decision layer that
sits above mechanics.
