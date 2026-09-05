# Title

Implement project-name and target-directory validation

## Summary

Implement `src/validate.ts`: npm-package-name validation, target-directory safety
rules (resolution, emptiness, forbidden locations), and the shared sanitizer used
for name defaults — the concrete enforcement of security boundary BND-1 with an
adversarial unit-test suite.

## Context

DESIGN.md §3.4 fixes the rules. These functions guard every filesystem write the
product ever makes on a user's machine; they must be small, pure where possible, and
tested against hostile inputs, because wizard/flag input is the product's first
trust boundary (§9 BND-1).

## Scope

`validate.ts` + unit tests. Consumed by options resolution (18), prompts (20 — for
inline validation messages), and the copier (21 — re-checks). No fs writes here;
read-only checks only.

## Detailed Requirements

1. `validatePackageName(name): { ok: true } | { ok: false, problems: string[] }`
   using `validate-npm-package-name` ^8 — require `validForNewPackages === true`;
   surface upstream `errors`/`warnings` strings as `problems`.
2. `sanitizeToPackageName(input): string | null` — the issue-18 rule (lowercase,
   spaces→`-`, strip chars outside `[a-z0-9-_.]`, collapse repeats of `-`, trim
   leading/trailing `-`/`.`); returns null when result is empty or still invalid
   per rule 1.
3. `checkTargetDir(rawPath, cwd): TargetDirResult` where result is one of
   `{ ok: true, absPath, mustCreate: boolean }` |
   `{ ok: false, reason: "not-empty" | "forbidden" | "invalid", detail }`:
   - resolve against cwd; reject `""`;
   - **forbidden**: resolved path is filesystem root; equals or contains (as prefix
     with separator) the CLI's own installation directory
     (`path.dirname(require.resolve of own package.json)`); or basename starts with
     `-` (argv-injection hygiene for later spawned tools);
   - existing path that is a file → `invalid`;
   - existing dir whose readdir (minus nothing — `.git` **counts** as content, per
     DESIGN §3.4) is non-empty → `not-empty`;
   - nonexistent → `ok` with `mustCreate: true` (creation itself happens in the
     copier issue, not here).
4. All checks must behave identically for relative input (`./x`, `x`, `../x` —
   allowed if it resolves somewhere legal) and absolute input.
5. Symlink rule: if the target exists and is a symlink, `fs.realpath` it first and
   apply the forbidden/emptiness rules to the real path.
6. Every `reason` maps to a fixed user-facing message table (exported) used by both
   CLI error paths (exit 3 for `not-empty`/`forbidden`/`invalid` per §3.2) and
   inline prompt validation.

## Acceptance Criteria

- [ ] Adversarial unit tests all pass: `../../etc`, absolute `/`, `~` literal (not
      expanded — treated as a normal char, documented), `-rf` basename, path into
      the CLI's own install dir, existing file, dir containing only `.git`, dir
      containing only `.DS_Store` (→ non-empty; no allowlist in v1), symlinked dir
      to a non-empty target, 300-char name, uppercase name (invalid per npm rules),
      scoped name `@scope/name` (valid for the package field — and legal as a
      package name; targetDir remains whatever the user gave).
- [ ] Property: for every `ok: false`, `detail` is non-empty and the message table
      has a row.
- [ ] No function in the module writes to the filesystem (fs spy test).

## Validation

`pnpm --filter @saber5656/create-mcp test` in CI (Node 22+24; symlink test skipped on
Windows if ever run there — mark with platform guard).

## Dependencies

18 (types; can proceed in parallel once `context.ts` lands — coordinate on the
result type names).

## Non-goals

Overwrite/`--force` semantics (not in v1), directory creation (21), prompt UX (20),
Windows reserved device names (document as untested best-effort in a code comment —
Windows is a v1 non-goal).

## Design References

DESIGN.md §3.2 (exit 3), §3.4, §9 BND-1.
