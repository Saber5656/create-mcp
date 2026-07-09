# Title

Implement post-generation actions (git init, install, next steps)

## Summary

Implement `post/git.ts`, `post/install.ts`, and `post/next-steps.ts`: optional
`git init`, optional dependency install with the chosen package manager, and the
transport-aware closing summary — with the §3.5 degradation rules (git failure =
warning; install failure = exit 5 with manual remediation).

## Context

DESIGN.md §3.5/§3.6 and BND-3: these are the only child processes create-mcp ever
spawns; both are argv-array, `shell: false`, enum-or-detected executables — never
free text.

## Scope

Three `post/` modules + wiring after the copier (21's seam) + unit tests with
spawn-spies and one real-git test. No changes to generation itself.

## Detailed Requirements

1. `git.ts` — `initGit(targetDir): Promise<GitResult>`:
   - availability probe `spawnSync("git", ["--version"])`; missing/failing →
     `{ skipped: "git-not-found" }` (warning line, still exit 0);
   - `spawn("git", ["init"], { cwd: targetDir, shell: false })`; non-zero →
     `{ failed: stderrTail }` (warning line, still exit 0 per DESIGN §3.5);
   - never runs when options.git is false; **no initial commit** (v1 rule).
2. `install.ts` — `installDeps(targetDir, pm): Promise<InstallResult>`:
   - `pm` is the ResolvedOptions enum value only;
   - `spawn(pm, ["install"], { cwd, shell: false, stdio: "inherit" })` so the user
     sees live PM output;
   - non-zero/spawn-error → `InstallResult.failed` with captured code; CLI maps to
     **exit 5** and prints: project was generated, then
     `cd <dir> && <pm> install` remediation (exact §3.5 wording);
   - skipped when options.install is false → next-steps must include the install
     step instead.
3. `next-steps.ts` — `renderNextSteps(options, results): string`:
   - always: `cd <relative-or-absolute-shortest-form>`;
   - if install skipped/failed: `<pm> install`;
   - build + test lines; then per transport: `npm run dev:stdio` / `dev:http`
     (or pnpm equivalents based on chosen PM — command strings must match the
     generated package.json scripts from issue 12/15 exactly, asserted by test
     against the template files);
   - conformance line: build-then-conformance pair (DESIGN §5.5 order);
   - badge activation pointer: "see README — replace the badge owner/repo
     placeholder after pushing to GitHub";
   - plain text, no color codes when `!isTTY` or `NO_COLOR`.
4. Ordering (wired after copier): git → install → next-steps; each step's outcome
   line printed as it completes (clack-style `log.step`/`log.warn` reused from 20's
   adapter where TTY, plain lines otherwise).

## Acceptance Criteria

- [ ] Spawn-spy tests: exact argv arrays for git and both PMs; `shell` never true;
      cwd always the target dir.
- [ ] Degradation matrix tests: git missing → exit 0 + warning; git init fails →
      exit 0 + warning; install fails → exit 5 + remediation text; install skipped →
      next-steps contains install line.
- [ ] Real-git test (CI has git): after a full generate with `--git`, `.git/HEAD`
      exists and `git -C <dir> status --porcelain` shows only untracked generated
      files (no commit was made).
- [ ] Next-steps snapshot per transport×PM (6 snapshots), each command verified to
      exist in the corresponding generated package.json variant.

## Validation

Unit + the real-git test in CI; one manual full run (`--yes`, both, npm, real
install) timing pasted in PR.

## Dependencies

18 (options/seam), 21 (runs after copier).

## Non-goals

Initial git commit, `--no-git` prompting subtleties (18/20 own flags), PM version
checks, offline install caching, opening editors/browsers.

## Design References

DESIGN.md §3.5, §3.6, §5.5 (command ordering), §9 BND-3.
