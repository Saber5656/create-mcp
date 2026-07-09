# Title

Author generated conformance CI workflow templates

## Summary

Author the two package-manager variants of the generated conformance workflow
(`templates/ts/ci/github/workflows/conformance.npm.yml.tmpl` and
`conformance.pnpm.yml.tmpl`), both rendering to `.github/workflows/conformance.yml`
in generated projects: install → build → `conformance` script, SHA-pinned actions,
least privilege, results artifact upload — the workflow whose status badge is the
product's core promise.

## Context

DESIGN.md §8 fixes the workflow skeleton and the two-variant mechanism: package-
manager variance is structural (manifest `when.packageManager`, DESIGN §4.2 /
ADR-004), never in-file conditionals. ADR-006 fixes badge semantics; §9 BND-8 fixes
the hardening rules. `npm ci` requires `package-lock.json` and
`pnpm install --frozen-lockfile` requires `pnpm-lock.yaml`, so each variant matches
the lockfile the chosen PM produced at generation time.

## Scope

Two workflow template files + manifest entries (`when.packageManager: ["npm"]` /
`["pnpm"]`, no transport condition — the `conformance` script from issue 15 already
encapsulates transport variance). Badge markup in README is issue 17.

## Detailed Requirements

1. Both variants (shared skeleton, per DESIGN §8):
   - `name: MCP Conformance`; triggers `push`, `pull_request`, `workflow_dispatch`;
   - top-level `permissions: { contents: read }`;
   - `concurrency: { group: conformance-${{ github.ref }}, cancel-in-progress: true }`;
   - single job `conformance`, `runs-on: ubuntu-latest`, `timeout-minutes: 15`;
   - steps end with: run build script → run conformance script →
     `actions/upload-artifact` of `conformance-results/` with `if: always()` and
     `retention-days: 14`.
2. npm variant steps: checkout → setup-node (node-version 24, `cache: npm`) →
   `npm ci` → `npm run build` → `npm run conformance`.
3. pnpm variant steps: checkout → `pnpm/action-setup` → setup-node (node-version
   24, `cache: pnpm`) → `pnpm install --frozen-lockfile` → `pnpm run build` →
   `pnpm run conformance`.
4. Every `uses:` pinned to a full 40-char commit SHA with a trailing `# vX.Y.Z`
   comment (actions/checkout, actions/setup-node, actions/upload-artifact,
   pnpm/action-setup). Record resolved SHAs in the PR.
5. A comment at the top of both files: the filename `conformance.yml` is load-bearing
   (README badge URL points at it — do not rename).
6. No tokens in these files (they carry no project-specific values); therefore no
   `substitute` flag in their manifest entries.
7. Manifest: two entries sharing target `.github/workflows/conformance.yml` with
   disjoint `when.packageManager` — this is the canonical exercise of issue 11's
   duplicate-target rule.

## Acceptance Criteria

- [ ] `actionlint` passes on both rendered variants (repo test invoking
      `npx actionlint` on plan output for a dummy npm project and a dummy pnpm
      project).
- [ ] Grep test: every `uses:` line in both files matches `@[0-9a-f]{40} # v`.
- [ ] Manifest coverage green; `buildPlan` for `(both, npm)` and `(both, pnpm)`
      each select exactly one workflow file targeting
      `.github/workflows/conformance.yml`.
- [ ] Diff between the two variants touches only the PM-specific steps (checked by
      a normalization test or eyeballed diff pasted in the PR).
- [ ] Manual once-off proof: push one generated project to a scratch GitHub repo,
      workflow runs, badge renders (screenshot in PR) — or an explicit deferral note
      pointing at issue 24's dogfood run as the proof site.

## Validation

actionlint + grep + plan tests in CI; the badge proof or deferral note in the PR.

## Dependencies

15 (conformance script exists), 11 (packageManager condition support).

## Non-goals

Repo-side CI (02), README badge markup (17), other forges/GitLab (v1 non-goal),
matrix over Node versions in generated projects (single Node 24 is the v1 contract).

## Design References

DESIGN.md §4.2, §8, §9 BND-8; ADR-004, ADR-006.
