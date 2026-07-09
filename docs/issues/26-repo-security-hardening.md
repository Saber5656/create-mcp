# Title

Add Dependabot, CodeQL, and workflow permission hardening

## Summary

Add automated dependency and code scanning to this repo (Dependabot for npm +
GitHub Actions ecosystems, CodeQL for JS/TS), verify every workflow obeys the
least-privilege rules, and document the GitHub repo-settings checklist (secret
scanning, push protection) as user-verifiable steps.

## Context

DESIGN §9 BND-5 (the exact-pinned conformance engine needs an upgrade proposer —
Dependabot — because it won't float) and BND-8 (workflow hardening); the repo is
public OSS from day one.

## Scope

`.github/dependabot.yml`, `.github/workflows/codeql.yml`, a workflow-audit test,
and `docs/SECURITY-SETTINGS.md` (checklist of GitHub UI settings). No product code.

## Detailed Requirements

1. `dependabot.yml`:
   - `package-ecosystem: npm`, weekly, grouped minor/patch updates; separate
     ungrouped rule so `@modelcontextprotocol/conformance` bumps arrive as
     individual PRs labeled `conformance-engine` (they trigger adapter contract
     tests — issue 06's tripwire);
   - `package-ecosystem: github-actions`, weekly (keeps SHA pins fresh across repo
     workflows).
2. `codeql.yml`: default CodeQL setup expressed as a workflow (language
   `javascript-typescript`), on `push` to main, `pull_request`, and weekly
   schedule; `permissions: { security-events: write, contents: read }`; SHA-pinned
   actions; `timeout-minutes: 30`.
3. Workflow-audit unit test (repo test, runs in `quality`): parse every
   `.github/workflows/*.yml` in this repo and assert: top-level or job-level
   `permissions` present; no `pull_request_target` trigger; every `uses:` SHA-pinned
   (40-hex + comment); `timeout-minutes` present on every job. (This locks BND-8
   as a regression test rather than a review-time hope. Template workflows under
   `templates/` are covered by issue 16's own tests — exclude that path here.)
4. `docs/SECURITY-SETTINGS.md`: checklist with exact GitHub UI paths for — secret
   scanning ON, push protection ON, private vulnerability reporting ON (pairs with
   SECURITY.md from 03), branch protection on `main` (already provisioned by the
   user's rulesets — verify and record), tag protection for release tags,
   Dependabot alerts ON. Each item phrased as verifiable ("Settings → … shows …")
   so the user can confirm and tick.
5. Do not change GitHub settings via API in this issue — settings are user-owned;
   the deliverable is config-as-code + the verified checklist (user ticks recorded
   in the PR description or a follow-up comment).

## Acceptance Criteria

- [ ] Dependabot config valid (GitHub UI shows both ecosystems active; screenshot
      or settings link in PR).
- [ ] CodeQL run completes green on the PR (first scan link attached).
- [ ] Workflow-audit test passes and demonstrably fails when a workflow drops
      `permissions` (mutation-tested once on a scratch branch, link in PR).
- [ ] SECURITY-SETTINGS.md checklist complete; items verifiable without ambiguity.
- [ ] The conformance-engine Dependabot rule produces individually-labeled PRs
      (verified on first real bump, or config-reviewed if none arrives — note
      which).

## Validation

CI + GitHub UI evidence links in the PR.

## Dependencies

02.

## Non-goals

OpenSSF Scorecard/Best-Practices badge (v2), SBOM generation (v2), fuzzing,
release pipeline (25), changing org-level settings.

## Design References

DESIGN.md §2.1, §9 BND-5/BND-8; ISSUE_PLAN §6 (security validation row).
