# Title

Add root README, CONTRIBUTING, SECURITY, and Code of Conduct

## Summary

Write the four community-facing documents for the repo root. The README states the
product promise (wizard + official conformance + badge), current status, and planned
usage; SECURITY.md defines the vulnerability-report channel; CONTRIBUTING.md explains
the pnpm/docs-first workflow; CODE_OF_CONDUCT.md adopts Contributor Covenant 2.1.

## Context

The repo is public from day one; these files set expectations before the first
release. DESIGN.md §1 (product definition), §9 (vulnerability reporting), ADR-006
(badge semantics — README must not overclaim).

## Scope

Four markdown files at repo root. English. No website, no logo work.

## Detailed Requirements

1. `README.md` (replaces the 2-line stub):
   - One-paragraph pitch matching DESIGN.md §1 wording ("verifiably conformant").
   - **Status banner**: pre-release, APIs may change until 0.1.0 publish.
   - Planned usage block: `npm create @saber5656/mcp`, wizard question list, the three
     transport variants, `npm run conformance`.
   - "How the badge works" section: exactly ADR-006 semantics — green = official
     suite (`active`, spec 2025-11-25) + smoke checks on default branch; explicitly
     states what it does NOT prove (security, quality of tool logic).
   - Architecture sketch (two packages + official suite, may reuse DESIGN §2 diagram).
   - Links: docs/DESIGN.md, docs/ISSUE_PLAN.md, MCP spec 2025-11-25, official
     conformance repo, inspector.
2. `SECURITY.md`: private reporting via GitHub Security Advisories ("Report a
   vulnerability" on this repo); no email channel; first response target 7 days;
   supported-versions table (latest minor only); explicitly asks reporters not to
   open public issues for vulnerabilities.
3. `CONTRIBUTING.md`: Node >=22 + pnpm prerequisites; clone→install→test loop;
   docs-first rule (design changes update docs/DESIGN.md + relevant docs/issues/*.md
   in the same PR); conventional-ish commit style (imperative subject, no enforced
   tooling in v1); PR checklist (CI green, docs updated, changeset added once issue 25
   lands — mark as "after release pipeline exists").
4. `CODE_OF_CONDUCT.md`: Contributor Covenant v2.1 verbatim with enforcement contact
   set to the GitHub profile contact of @Saber5656 (no personal email in the file).

## Acceptance Criteria

- [ ] All four files exist, pass `pnpm lint` (Biome markdown passthrough — no broken
      formatting), and contain no personal email addresses or secrets.
- [ ] README badge section matches ADR-006 semantics word-for-word on the two claims
      (what green means; what it does not prove).
- [ ] Every relative link in the four files resolves (`git ls-files` check or link
      script); every external link is HTTPS.
- [ ] README contains no invented CLI flags — only flags defined in DESIGN.md §3.2.

## Validation

Manual read-through against DESIGN.md §1/§3.2/ADR-006; run a link checker (or manual
click-through) and paste results in the PR.

## Dependencies

01.

## Non-goals

Final usage docs with real captured output (issue 27 rewrites usage sections after the
CLI exists); GitHub repo settings (26); issue/PR templates (nice-to-have, only if
trivial — otherwise defer to v2).

## Design References

DESIGN.md §1, §9 (reporting), §12; ADR-003 (names in examples), ADR-006.
