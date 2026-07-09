# Title

Add CLI end-to-end tests for all transport variants

## Summary

Add an e2e suite that runs the **built** create-mcp CLI non-interactively for all
three transport variants (npm and pnpm where it matters), then proves each generated
project stands on its own: installs, typechecks, builds, and passes its bundled unit
tests. Wire as a CI job.

## Context

DESIGN §10 "CLI e2e" row. Issues 12–22 each validated their slice; this issue
validates the composition the way a user experiences it — from `create-mcp --yes` to
green `npm test` inside the generated project. (Official-suite conformance of the
generated project is issue 24's dogfood job, not here.)

## Scope

`packages/create-mcp/test/e2e/*.test.ts` (vitest, tagged e2e config), a CI job
`cli-e2e`, temp-dir helpers. No production code changes except bugs found.

## Detailed Requirements

1. Matrix: `(stdio, npm)`, `(http, npm)`, `(both, npm)`, `(both, pnpm)` — four
   runs. pnpm coverage is deliberately limited to the `both` variant: template PM
   variance affects only the workflow file and the install command, both of which
   `(both, pnpm)` exercises.
   Each run:
   - `node packages/create-mcp/dist/index.js <tmp>/app --name e2e-app --transport
     <t> --pm <pm> --no-git --no-install --yes` (install performed explicitly in
     step 3 for better failure isolation);
   - assert exit 0 and the §3.6 next-steps block present in stdout.
2. Structure assertions per variant: generated file list equals the plan-derived
   expected list (import `buildPlan` directly to compute expectations — keeps the
   test manifest-synchronized); zero `__MCP_TMPL_` residue; `package.json.name ===
   "e2e-app"`; workflow file present with the PM-correct install command.
3. Self-sufficiency per variant: inside the generated dir run
   `<pm> install` (network allowed in CI; local registry not required — the kit
   devDependency `@saber5656/mcp-conformance-kit` is **not yet published**, so
   for e2e the test rewrites that one devDependency to `file:` + `npm pack` output
   of the workspace kit before installing; helper `linkLocalKit(dir)` implements
   this and logs the substitution) → `<pm> run build` (tsc) → `<pm> test` (vitest
   from the template) — all exit 0.
4. Negative e2e (asserted on the built binary): run into a non-empty dir → exit 3,
   dir untouched; non-TTY without `--yes` and without `--transport` → exit 2
   listing the missing flags (per issue 18's resolution matrix: defaults do not
   auto-apply non-interactively without `--yes`); non-TTY with `--yes` but no
   targetDir → exit 2 naming the positional.
5. CI job `cli-e2e`: needs `quality`, Node 24, `timeout-minutes: 20`; caches PM
   stores; runs the vitest e2e config.
6. Runtime budget: whole suite ≤ 10 min in CI (parallelize variant dirs with
   vitest workers if needed).

## Acceptance Criteria

- [ ] Four-variant matrix green in CI, including the pnpm run.
- [ ] `linkLocalKit` proves the generated project consumed the *workspace* kit
      build (assert installed kit version equals workspace version).
- [ ] Negative cases exit with the documented codes and messages.
- [ ] Suite runtime recorded in PR; ≤ budget.

## Validation

CI run link; one locally-run variant transcript pasted in the PR.

## Dependencies

17 (all template files), 20, 21, 22.

## Non-goals

Conformance/official-suite execution (24), interactive TTY automation (20 covered
manually), publishing (25), yarn/bun.

## Design References

DESIGN.md §3.2 (exit codes), §3.6, §10 (CLI e2e row); ISSUE_PLAN §6.
