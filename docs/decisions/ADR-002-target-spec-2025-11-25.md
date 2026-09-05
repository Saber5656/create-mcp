# ADR-002: v1 targets MCP revision 2025-11-25 on SDK 1.x

Date: 2026-07-10 — Status: Accepted (user-confirmed)

## Context

Latest stable MCP revision is 2025-11-25. A major revision (2026-07-28: stateless
core, extensions framework) is an RC publishing ~18 days after this decision. The
TypeScript SDK implementing it (v2) is beta and **not published to npm**; npm `latest`
is 1.29.0. The official conformance suite's `active` scenarios target 2025-11-25.
Evidence: `docs/research/2026-07-mcp-ecosystem-survey.md` §1, §3.

## Decision

v1 builds entirely on **spec 2025-11-25 + SDK ^1.29.0 + zod v4**. The kit config
carries an explicit `specVersion` literal (`"2025-11-25"`), and the adapter passes
`--spec-version` to the official suite, so the 2026-07-28 migration widens a literal
and revises templates rather than reshaping the architecture. Migration work is a
named deferred wave in ISSUE_PLAN.md ("v1.1: 2026-07-28 / SDK v2"), not part of v1.

zod v4 is chosen now (SDK 1.29 supports `^3.25 || ^4.0`; the suite and SDK v2
direction are zod-4/Standard-Schema) to minimize v1.1 churn.

## Consequences

- v1 can start immediately on stable dependencies and ship before/around the new spec.
- A fast-follow (v1.1) will be needed soon after SDK v2 GA; this is accepted and
  planned, and generated projects pin what they were verified against.

## Alternatives rejected

- **Target 2026-07-28 now**: builds a user-facing product on an unpublished beta SDK
  and an RC spec; release date becomes hostage to SDK v2 GA.
- **Wait for GA**: ≥1 month idle, and the ecosystem (suite scenarios, SDK stability)
  still needs settling time after GA; niche risk meanwhile.
