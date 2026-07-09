# Title

Final documentation pass with verified real outputs

## Summary

Replace every "planned" statement in the root README with reality captured from the
finished product: real wizard transcript, real generated-project tree, real
conformance report excerpt, real badge markup — and reconcile DESIGN.md/ISSUE_PLAN.md
scope-ledger entries against what actually shipped, so v1 documentation contains no
aspirational claims.

## Context

Issue 03 wrote the README before the product existed (by design). ISSUE_PLAN §6
(manual gate) requires README accuracy at release; docs are the canonical source and
must end v1 truthful. This is deliberately the last issue.

## Scope

Root `README.md`, `docs/DESIGN.md` (status header + any drift notes), `docs/
ISSUE_PLAN.md` (§7/§8 outcomes), `CONTRIBUTING.md` touch-ups. No template or code
changes (drift found here files bugs against owning issues instead).

## Detailed Requirements

1. README usage section: replace planned blocks with captured output of the real
   CLI — wizard screenshot-as-text (S1–S7 transcript from a real TTY run),
   `--help` output verbatim, generated tree (`find`-style listing for `both`
   variant), next-steps block. Every captured block gets a comment marker
   (`<!-- captured: create-mcp vX.Y.Z, 2026-MM-DD -->`) so future drift is datable.
2. README conformance section: excerpt of a real terminal report (color-off) and a
   real `$GITHUB_STEP_SUMMARY` table screenshot from the dogfood run artifacts;
   badge how-to updated with the confirmed activation steps (16/24 evidence).
3. Verify every command in README/CONTRIBUTING by executing it from a clean clone
   (script the check where feasible: extract fenced `sh` blocks marked
   `<!-- verify -->` and run them in CI once — lightweight, not a doc-test
   framework).
4. DESIGN.md: set status header to "v1 shipped (as-built)"; append an "As-built
   deviations" subsection to §12 listing every place implementation diverged from
   this design (source: issue threads' U1–U6 outcomes and any mid-flight DESIGN
   edits), each with a one-line rationale. Empty list is a valid outcome.
5. ISSUE_PLAN.md: mark §8 known-unknowns with their resolutions; confirm §7
   deferred list still matches reality (move anything that accidentally shipped or
   grew).
6. Cross-check: no doc anywhere still says "planned"/"will" about shipped v1
   behavior (`grep -ri "planned\|will be" README.md docs/` triaged line by line —
   remaining hits must be about v1.1/v2).

## Acceptance Criteria

- [ ] All captured blocks carry the capture marker with version + date.
- [ ] The `<!-- verify -->` command check passes in CI (or a one-shot script run
      pasted in PR).
- [ ] DESIGN.md as-built section present; ISSUE_PLAN §8 all resolved or explicitly
      carried to v1.1.
- [ ] grep triage from requirement 6 attached to the PR with dispositions.
- [ ] A newcomer dry-read (self-review checklist in the PR): can they go from
      README top to a working conformance run without hitting a wrong statement?

## Validation

CI + the attached triage/transcripts; final read-through recorded in the PR.

## Dependencies

24, 25, 26.

## Non-goals

New features, blog posts/announcements, version bumps (release flow owns those),
restructuring docs (v1 keeps the current layout).

## Design References

DESIGN.md §12; ISSUE_PLAN §1 (completion), §6 (manual gate); ADR-006 (badge claims
stay honest).
