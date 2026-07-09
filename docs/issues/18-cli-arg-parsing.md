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
per DESIGN §7.1), unit tests. `index.ts` wiring becomes real here: exported
`run(argv: string[], deps?: { generate?: (options: ResolvedOptions) => Promise<void> })`
— the `deps.generate` injection point is the seam issue 21 fills; its default here
is a stub that prints the resolved options summary and returns (exit 0).

## Detailed Requirements

1. Commander (^15) definition exactly per DESIGN §3.2: positional `[targetDir]`,
   `--transport <stdio|http|both>` (choices-validated), `--pm <npm|pnpm>`
   (choices-validated), `--git`/`--no-git`, `--install`/`--no-install`, `--yes`,
   `--name <string>`, `--version`, `--help`. Unknown flags → commander error →
   exit 2 with usage hint.
2. `resolveOptions(raw, env, isTTY, promptFn)` — two-layer semantics (DESIGN §3.2
   default rule: defaults apply only via `--yes` or as prompt pre-selections; no
   silent defaulting in non-interactive mode):
   - **explicit layer**: values provided via argv (positional targetDir) and flags;
   - **unresolved set** = every field not explicitly provided, out of
     `{ targetDir, packageName, transport, packageManager, git, install }`;
   - default values (used by `--yes` and as prompt pre-selections only): transport
     `both`; git `true`; install `true`; packageManager from
     `npm_config_user_agent` (`pnpm/` prefix → pnpm, else npm); packageName =
     `sanitizeToPackageName(basename(targetDir))` (delegate to issue 19's function
     — single source of the sanitization rule; `null` result means the field stays
     unresolved and must be prompted or fails);
   - resolution matrix:
     | context | behavior |
     |---|---|
     | TTY, no `--yes`, unresolved ≠ ∅ | `promptFn(unresolved, partial)` asks **all** unresolved fields (S1–S6 reachable), defaults pre-selected |
     | TTY, `--yes` | fill unresolved from defaults; prompt only for `targetDir` if missing (it has no default) |
     | non-TTY, `--yes` | fill unresolved from defaults; missing `targetDir` → error naming the positional |
     | non-TTY, no `--yes`, unresolved ≠ ∅ | error listing the exact missing flags (`missing required options in non-interactive mode: --transport …`) |
   - Returns `{ ok: true, options } | { ok: false, exitCode: 2, message }`.
3. `ResolvedOptions` per DESIGN §7.1 (absolute targetDir via `path.resolve`).
4. Exit-code ownership: `index.ts` maps resolution failure → 2; prompt cancellation
   (promptFn returning a `cancelled` sentinel) → 130; generation stub success → 0.
5. `--version` prints the package version (single line) and exits 0.

## Acceptance Criteria

- [ ] Unit tests: every row of the resolution matrix above; both PM detections;
      `--pm` override; sanitization delegation (asserted to call issue 19's
      `sanitizeToPackageName`; unicode-only basename → field stays unresolved);
      `--yes` with and without targetDir in TTY and non-TTY; unknown flag → exit 2.
- [ ] `promptFn` is called with exactly the unresolved field set — including
      defaultable fields when they were not explicitly provided (asserted).
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
