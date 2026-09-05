# Title

Add dogfood e2e: generated project passes official conformance in CI

## Summary

Add the release-gate CI job: generate a `both`-variant project with the built
create-mcp, install the workspace-packed tarballs of both packages into it, build
it, and run its own `conformance` script — asserting the **official conformance
suite** and the smoke checks pass exactly as a real user's project would. This is
the product promise, executed.

## Context

DESIGN §10 dogfood row and ISSUE_PLAN §6: every layer below is unit/integration
tested; this job proves the composed promise ("generated projects verifiably speak
MCP") and is the trigger for Known unknown U3 (expected-failures baseline for the
template server, if any).

## Scope

`test/dogfood/` script(s) at repo root (or under create-mcp — implementer's choice,
documented), a `dogfood` CI job, and any template/kit bug fixes it flushes out
(each fix goes to its owning module with a test there — this issue only hosts the
harness).

## Detailed Requirements

1. Harness steps (script, runnable locally via `pnpm dogfood`):
   1. `pnpm build` (workspace);
   2. `npm pack` both packages into a temp artifacts dir;
   3. run built CLI: `node packages/create-mcp/dist/index.js <tmp>/dogfood-app
      --name dogfood-app --transport both --pm npm --no-git --no-install --yes`;
   4. rewrite the generated devDependency `@saber5656/mcp-conformance-kit` to the
      packed tarball path (`file:`), then `npm install` in the app;
   5. `npm run build` in the app;
   6. `npm run conformance` in the app (kit orchestrates: starts dist/http.js,
      official suite `active` @ 2025-11-25, stops server, runs stdio smoke);
   7. assert exit 0; parse `conformance-results/report.json`: official section
      present with engine 0.1.16, `summary.fail === 0`, smoke MUST checks all pass.
2. Install create-mcp itself from its tarball too (step 3 alternative: `npm exec
   --prefix` from a scratch install) — REQUIRED: the CLI must run from the packed
   artifact at least once in this job to catch `files`/packaging mistakes
   (missing templates in the tarball is the classic failure). Implement: install
   `create-mcp-<v>.tgz` into a scratch dir and invoke the installed bin for the
   generation step.
3. U3 protocol: if any official scenario fails, triage per issue 10 requirement 3
   (fixture→template analog): template bug → fix in owning issue/module; upstream
   gap → dated, evidenced entry in the template's expected-failures baseline (15)
   plus README note (17) — and update ISSUE_PLAN §8 U3 with the outcome either way.
4. CI job `dogfood`: needs `quality` + `kit-integration` + `cli-e2e`; Node 24;
   `timeout-minutes: 25`; uploads the app's `conformance-results/` artifact
   (`if: always()`, 14 days) and the packed tarballs (7 days).
5. Local ergonomics: `pnpm dogfood` leaves the temp app path printed for
   inspection; CI cleans up via runner disposal.

## Acceptance Criteria

- [ ] Dogfood job green in CI from the packed-tarball CLI (requirement 2 verified
      by asserting the invoked bin path is inside the scratch `node_modules`).
- [ ] `report.json` artifact shows official engine 0.1.16, spec 2025-11-25,
      0 unexpected failures, all smoke MUSTs pass.
- [ ] Expected-failures baseline is either empty (ideal) or every entry carries
      date + evidence link, mirrored in ISSUE_PLAN §8 U3.
- [ ] Job runtime ≤ 15 min steady-state (record baseline in PR).
- [ ] Running `pnpm dogfood` locally documented in CONTRIBUTING.md (one paragraph
      added).

## Validation

Two consecutive green CI runs (stability); artifact inspected and linked in the PR.

## Dependencies

10, 16, 23.

## Non-goals

Publishing to npm (25), scratch-GitHub-repo badge proof (16 owns or defers it),
performance benchmarking, stdio-only and http-only dogfood variants (covered
piecewise by 23 + 10; `both` exercises the full surface).

## Design References

DESIGN.md §10 (dogfood row), §5.5, §6.2–6.4; ISSUE_PLAN §1 (completion statement),
§6, §8 U3.
