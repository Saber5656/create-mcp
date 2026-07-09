# Title

Implement token substitution and copy-plan executor

## Summary

Implement `engine/tokens.ts` (closed-set literal substitution with residue
detection) and `engine/copier.ts` (CopyPlan executor: safe writes, 0644 modes,
tracked-cleanup on failure), then replace the generation stub from issue 18 so
`create-mcp` actually generates projects end to end.

## Context

Second half of BND-2 (DESIGN §9): the manifest/plan (11) decided *what*; this issue
makes the writes and must re-verify *where* (defense in depth) and clean up after
itself. Token rules are DESIGN §4.3; copier guarantees are §4.4.

## Scope

Two engine modules, the `index.ts` wiring (options → loadManifest → buildPlan →
copy → post hooks seam for issue 22), dotfile-rename handling, and unit tests with
temp dirs. Post-generation actions remain stubbed until 22.

## Detailed Requirements

1. `tokens.ts`:
   - `TOKEN_VALUES(options, kitDepVersion): Record<TokenName, string>` for exactly
     the DESIGN §4.3 table (PACKAGE_NAME, KIT_DEP_VERSION, TRANSPORT, BADGE_PATH
     placeholder literal, NODE_MIN);
   - `substitute(content, values): { output, unknownTokens: string[] }` — replace
     all occurrences; then scan output for the pattern `__MCP_TMPL_[A-Z0-9_]+__`;
     any residue → listed in `unknownTokens`;
   - `kitDepVersion` source (DESIGN §4.3): create-mcp declares
     `@saber5656/mcp-conformance-kit` as a devDependency purely as a version
     reference; add `scripts/gen-kit-version.mjs` and wire the package build as
     `"build": "node scripts/gen-kit-version.mjs && tsup"` so the generated,
     gitignored `src/kit-version.ts` constant always exists before compilation
     (plus a test asserting the constant equals the devDependency spec).
2. `copier.ts` — `execute(plan: CopyPlan, tokenValues): Promise<CopyResult>` (the
   plan carries `targetRoot` and `mustCreate` per DESIGN §7.2):
   - create `plan.targetRoot` if `plan.mustCreate` (value originates from issue
     19's `checkTargetDir`);
   - per file: re-assert `targetAbs` starts with `plan.targetRoot + sep` (throw
     `CopierInvariantError` otherwise — this is a bug trap, not user error);
     mkdir -p parent; read source; if `substitute` → run tokens (any
     `unknownTokens` → fail the run); write with mode 0644, `flag: "wx"`
     (fail if exists — plan collisions are bugs);
   - dotfile sources never exist (DESIGN §4.1 rule, enforced by issue 11's schema:
     `dot-` prefixed sources map to dotfile targets) — the copier needs no rename
     logic; this issue adds the repo test asserting no file under `templates/`
     starts with a dot;
   - track every path created (files + dirs it made); on any error: remove tracked
     files, then tracked dirs deepest-first, never touching pre-existing paths;
     rethrow as `GenerationError` (CLI maps → exit 4).
4. Wiring in `index.ts`: resolve options (18/19/20) → load manifest → buildPlan →
   execute → (issue-22 seam) → success summary (files written count + target path).
   Exit codes: manifest/plan/copy internal failures → 4 (with cleanup note in the
   message); everything upstream keeps its 2/3/130 codes.
5. Post-substitution content assertions live here as unit tests using the real
   bundled template: generated `package.json` parses, `name` equals the option,
   kit devDependency spec matches the mechanism in 1.

## Acceptance Criteria

- [ ] Temp-dir e2e (unit level): full generate for `(both, npm)` produces exactly
      the plan's file list; every file mode 0644; zero token residue
      (`grep -r "__MCP_TMPL_" <target>` empty); `.gitignore` present with correct
      name.
- [ ] Failure injection: making the copier fail mid-plan (mock fs write error on
      file N) leaves the target exactly as before when `mustCreate` was false, or
      fully removed when the copier created it; pre-existing sibling files
      untouched.
- [ ] `flag: "wx"` collision test (pre-created file at a plan target inside an
      empty-dir edge → run fails cleanly with cleanup).
- [ ] Escape-attempt test: hand-built malicious plan (target outside root) throws
      `CopierInvariantError` before any write.
- [ ] No file under `templates/` begins with a dot (repo test), and every `dot-`
      prefixed source maps to the correct dotfile target (checked against the
      manifest).

## Validation

`pnpm --filter @saber5656/create-mcp test`; run the built CLI once locally
(`node dist/index.js /tmp/demo --yes`) and paste the summary output in the PR.

## Dependencies

11, 12 (real bundled template content used by the acceptance tests), 18, 19.

## Non-goals

Prompt UX (20), git/install/next-steps (22), `--force` overwrite, template content
correctness (12–17 own that), Windows ACL/mode nuances.

## Design References

DESIGN.md §3.1, §4.3, §4.4, §7.2, §9 BND-2.
