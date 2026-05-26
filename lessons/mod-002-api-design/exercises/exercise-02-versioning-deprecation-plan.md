# Exercise 02: Versioning + Deprecation Plan

## Objective

Take an existing v1 API (use the one from exercise 01) and ship a v2 with one
breaking change, plus a full deprecation plan.

## The change

v1 has `gpu_count: int`. v2 needs to support fractional GPUs and per-tier
selection: `gpu: { type: "a100", count: 1, mig_slice: "1g.5gb" }`.

## Deliverables

1. v2 spec with the new schema
2. `MIGRATION.md` documenting:
   - What changed
   - How to translate v1 request bodies to v2 (with examples)
   - Deprecation timeline (T+0, T+90d, T+180d, T+365d) with concrete dates
   - Rollback plan if v2 has unexpected issues
3. `Sunset` + `Deprecation` headers configured on v1 responses
4. Working bidirectional translation: a tool that converts v1 manifests → v2 and back

## Acceptance

- v2 spec validates
- v1 + v2 both deployable side-by-side (different URL prefixes)
- Migration tool round-trips: `v1_doc == v2_to_v1(v1_to_v2(v1_doc))` for purely-additive subset
