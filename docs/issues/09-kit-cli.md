# Title

Implement mcp-conformance-kit CLI commands and exit codes

## Summary

Implement `src/bin.ts` wiring config loading → lifecycle → official adapter → smoke
runner → renderers behind the commands `run`, `smoke`, and `print-config`, with the
exit-code contract from DESIGN.md §6.2 and the `--allow-remote` opt-in from §9 BND-4.
Also finalize the programmatic API surface in `src/index.ts`.

## Context

This is the kit's composition root: everything before it (04–08) is libraries. The
generated projects call exactly `mcp-conformance-kit run` (DESIGN §5.5), so this
contract is frozen by semver from first publish (DESIGN §11).

## Scope

`bin.ts`, an orchestration module `src/run-all.ts`, `index.ts` exports, unit tests
with mocked section runners, plus one happy-path test over real modules against the
smoke fixture server (http path is integration-tested in issue 10).

## Detailed Requirements

1. Commander (^15) program `mcp-conformance-kit`, version from package.json:
   - `run [--config <path>] [--only http|stdio] [--allow-remote] [--no-color]`
   - `smoke [--config <path>]` — exact alias for `run --only stdio`
   - `print-config [--config <path>] [--allow-remote]` — prints resolved config JSON
     (with defaults applied) to stdout, exit 0/2 only.
2. `run` sequence: loadConfig (`allowRemote` from flag) → select sections by
   presence + `--only` → if http: startServer → waitUntilReady → runOfficial →
   stop (always, `finally`) → if stdio: runSmoke → merge Report → writeReport →
   renderTerminal(stdout) → appendGithubSummary.
3. Exit codes exactly per DESIGN §6.2: 0 all-green (expected-fail semantics
   respected), 1 any section failed, 2 config error (including `--only http` when
   config has no http section — message says so), 3 `ServerStartError`/
   `ReadyTimeoutError`, 4 anything else (with stack to stderr and a bug-report hint).
   The numeric mapping lives in one exported `exitCodeFor(outcome)` function.
4. Color decision: `--no-color` OR `NO_COLOR` env OR `!isTTY` → color off; passed
   down to the terminal renderer.
5. Failure UX: exit-3 paths print the stderr tail already captured by lifecycle
   errors; exit-2 paths print the config error list verbatim, one per line, prefixed
   `config:`.
6. Programmatic API (`index.ts`): `loadConfig`, `runAll(config, opts)` returning
   `{ report, exitCode }` without calling `process.exit`, plus the types from 04.
   `bin.ts` is the only file allowed to call `process.exit`.
7. `smoke` with a config lacking `stdio` → exit 2 (same rule as `--only`).

## Acceptance Criteria

- [ ] Unit tests with injected fakes cover every exit code 0–4, `--only` behavior,
      alias equivalence of `smoke`, and the always-stop guarantee (official adapter
      throwing still stops the server — spy on `stop`).
- [ ] Real-modules test: `run` against a config pointing at the good stdio fixture
      (07) exits 0 and writes a valid report.json.
- [ ] `print-config` output parses as JSON and equals the zod-resolved config.
- [ ] `--allow-remote` reaches the config parser (non-loopback URL accepted with the
      flag, rejected without — asserted at CLI level).
- [ ] README of the kit package documents the three commands, flags, and the exit
      code table (copy from DESIGN §6.2).

## Validation

`pnpm --filter @saber5656/mcp-conformance-kit test`; run the built bin manually against
the fixture (`node dist/bin.js run --config <fixture config>`) and paste output in PR.

## Dependencies

06, 07, 08.

## Non-goals

`--strict` (reserved, v1.1), watch mode, JUnit/other output formats, config
generation (`init` command — v2), parallel section execution (sequential is fine).

## Design References

DESIGN.md §5.5, §6.1, §6.2, §7.3, §9 BND-4, §11 (semver surface).
