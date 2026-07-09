# ADR-006: Conformance badge = generated GitHub Actions workflow badge

Date: 2026-07-10 — Status: Accepted (user-confirmed)

## Context

The product promise includes a README "conformance badge". Candidate mechanisms:
(a) static shields.io image, (b) GitHub Actions workflow status badge backed by a
generated CI workflow that actually runs the conformance suite, (c) a hosted badge
service that records verified results.

## Decision

Generated projects get `.github/workflows/conformance.yml` (DESIGN.md §8) and embed
that workflow's **status badge** in their README. Badge semantics: green means the
official suite (`active` scenarios, spec 2025-11-25) plus the kit's smoke checks
passed on the default branch of *that* repository. The kit writes a per-run summary
table to `$GITHUB_STEP_SUMMARY` as the human-readable detail behind the badge.

## Consequences

- Evidence-backed badge with zero operated infrastructure and zero credential
  handling; works the moment the user pushes to GitHub.
- The badge is only as fresh as the repo's CI runs, and users who don't push to
  GitHub get no badge (README documents local `npm run conformance` as the
  equivalent check). Accepted for v1.
- Non-GitHub forges are out of scope for v1 (non-goal; README says so).

## Alternatives rejected

- **Static shields badge**: decorative, proves nothing, invites false claims.
- **Hosted badge service**: strongest proof but requires running a trusted service
  (auth, anti-spoofing, abuse handling, uptime) — disproportionate for v1; revisit as
  a v2 idea only if there's demand.
