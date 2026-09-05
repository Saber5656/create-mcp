# Title

Add repository CI workflow for lint, typecheck, and unit tests

## Summary

Add `.github/workflows/ci.yml` running lint, typecheck, build, and unit tests on a
Node 22 + 24 matrix for every PR and push to `main`, with SHA-pinned actions and
least-privilege permissions. This is the quality gate every later issue lands behind.

## Context

DESIGN.md §10 (Continuous row) and the security posture for workflows (§9 BND-8):
actions pinned by commit SHA, top-level `permissions: contents: read`, no
`pull_request_target`.

## Scope

One workflow file plus any tiny root-script adjustments needed to make the four steps
single-command. Integration/e2e/dogfood jobs are NOT here (issues 10, 23, 24 extend
this workflow or add jobs).

## Detailed Requirements

1. Triggers: `pull_request` (all branches), `push` to `main`, `workflow_dispatch`.
2. Top level: `permissions: { contents: read }`;
   `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }`.
3. Job `quality`, `runs-on: ubuntu-latest`, `strategy.matrix.node: [22, 24]`:
   checkout → pnpm setup → setup-node with matrix version + pnpm cache →
   `pnpm install --frozen-lockfile` → `pnpm lint` → `pnpm typecheck` → `pnpm build` →
   `pnpm test`.
4. Every `uses:` is a **full 40-char commit SHA** with trailing `# vX.Y.Z` comment
   (actions/checkout, pnpm/action-setup, actions/setup-node). Record the resolved
   SHAs in the PR description.
5. `timeout-minutes: 15` on the job.
6. No secrets referenced anywhere.

## Acceptance Criteria

- [ ] Workflow runs green on a PR touching any package on both Node versions.
- [ ] `permissions` block present at workflow top level; no job overrides broaden it.
- [ ] All actions SHA-pinned (grep `uses:.*@[a-f0-9]{40}` matches every uses line).
- [ ] Cancelling behavior verified: pushing twice quickly cancels the older run.
- [ ] A deliberately broken test on a scratch branch turns the workflow red
      (verified once, then reverted).

## Validation

Open a scratch PR with (a) clean state → green, (b) an intentional lint error → red;
attach run links. Then remove the scratch commit.

## Dependencies

01.

## Non-goals

Integration/e2e/dogfood jobs (10, 23, 24); CodeQL/Dependabot (26); release publishing
(25); Windows/macOS runners (v1 non-goal).

## Design References

DESIGN.md §8 (repo-side CI note), §9 BND-8, §10.
