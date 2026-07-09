# Title

Implement create-mcp argument parsing and option resolution

## Summary

Implement `src/cli.ts` (commander program: flags, help, version) and `src/options.ts`
(RawOptions → ResolvedOptions: defaults, TTY policy, `--yes` semantics, package-
manager auto-detect), establishing the CLI contract of DESIGN.md §3.2 and the
`ResolvedOptions` type every other CLI module consumes.

## Context

This issue owns the behavior matrix in §3.2 (interactive vs non-interactive vs
error) and exit codes 2/130 groundwork. Validation internals (name/dir safety) are
issue 19; prompting is issue 20 — here, the wizard is an injected async callback so
this issue is fully testable without a TTY.

## Scope

`cli.ts`, `options.ts`, `src/context.ts` (shared types: RawOptions, ResolvedOptions
per DESIGN §7.1), unit tests. `index.ts` wiring becomes real here (parse → resolve →
placeholder "would generate" output until issue 21 lands — keep behind a
`GENERATE_IMPL` seam: options resolution returns; generation invoked via an injected
function defaulting to a stub that prints the resolved plan summary and exits 0).

## Detailed Requirements

1. Commander (^15) definition exactly per DESIGN §3.2: positional `[targetDir]`,
   `--transport <stdio|http|both>` (choices-validated), `--pm <npm|pnpm>`
   (choices-validated), `--git`/`--no-git`, `--install`/`--no-install`, `--yes`,
   `--name <string>`, `--version`, `--help`. Unknown flags → commander error →
   exit 2 with usage hint.
2. `resolveOptions(raw, env, isTTY, promptFn)`:
   - defaults: transport `both`, git `true`, install `true`;
   - `packageManager` default from `npm_config_user_agent` (`pnpm/` prefix → pnpm,
     else npm); explicit `--pm` wins;
   - `--name` defaults to sanitized basename of targetDir (sanitization rule:
     lowercase, spaces→`-`, strip characters outside `[a-z0-9-_.]`; if result is
     empty → must prompt or fail);
   - TTY policy matrix (DESIGN §3.2): compute the set of unresolved fields; if
     non-empty and TTY → call `promptFn(unresolved, partial)`; if non-empty, no TTY,
     no `--yes` → return error listing exact missing flags (message format:
     `missing required options in non-interactive mode: --transport …`); `--yes`
     fills all remaining defaults (targetDir has no default — targetDir missing +
     `--yes` + no TTY is still an error naming the positional).
   - Returns `{ ok: true, options } | { ok: false, exitCode: 2, message }`.
3. `ResolvedOptions` per DESIGN §7.1 (absolute targetDir via `path.resolve`).
4. Exit-code ownership: `index.ts` maps resolution failure → 2; prompt cancellation
   (promptFn returning a `cancelled` sentinel) → 130; generation stub success → 0.
5. `--version` prints the package version (single line) and exits 0.

## Acceptance Criteria

- [ ] Unit tests: every §3.2 matrix row; both PM detections; `--pm` override;
      name sanitization cases (`My App` → `my-app`, unicode-only → error/prompt);
      `--yes` with and without targetDir; unknown flag → exit 2.
- [ ] `promptFn` is called with exactly the unresolved field set (asserted).
- [ ] No direct `process.exit` outside `index.ts` (grep test).
- [ ] `node dist/index.js --help` output lists every flag from DESIGN §3.2 and no
      others (snapshot test).

## Validation

`pnpm --filter @saber5656/create-mcp test`; paste `--help` output in the PR.

## Dependencies

01.

## Non-goals

Name/directory *safety* validation (19 — resolution calls it but here it may be a
pass-through stub), interactive prompt implementation (20), template work (21),
`--force`/`--dry-run` flags (not in v1).

## Design References

DESIGN.md §3.1, §3.2, §7.1.
