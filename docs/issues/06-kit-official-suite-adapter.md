# Title

Implement adapter around the official conformance suite

## Summary

Implement `adapter/official.ts` (resolve the exactly-pinned
`@modelcontextprotocol/conformance` bin, build its CLI args from kit config, execute
it) and `adapter/results.ts` (locate and parse `checks.json` into the Report model),
with contract tests that pin upstream's observable behavior so dependency bumps fail
loudly instead of silently breaking.

## Context

ADR-001: the official suite is the engine; this adapter is the only module allowed to
know its CLI/output details. DESIGN.md §6.4 fixes the mechanics; §9 BND-5 forbids
`npx`-at-runtime and floating versions.

## Scope

Two modules, fixtures (captured real `checks.json` files + `--help` snapshot), unit +
contract tests. Adding the dependency `@modelcontextprotocol/conformance` at **exact**
`0.1.16` to the kit's package.json. Does not start the server (05 does) and does not
render (08 does).

## Detailed Requirements

1. Bin resolution: `createRequire(import.meta.url).resolve("@modelcontextprotocol/conformance/package.json")`,
   read `bin` field (`conformance` key), join to absolute path. Execute as
   `spawn(process.execPath, [binPath, ...args], { shell: false, cwd: runDir })`.
2. Arg mapping `buildArgs(http: HttpTarget, specVersion: "2025-11-25")` (pure,
   unit-tested): `["server", "--url", http.url]` + `--suite <suite>` + repeated
   `--scenario <s>` when `scenarios` set + `--spec-version <specVersion>` +
   `--expected-failures <path>` when set. The path is already absolute (the config
   loader resolved it against the config file's directory — issue 04); this module
   only asserts existence (missing file → typed config error before spawn).
3. Execution: capture stdout/stderr to `<runDir>/engine-stdout.log` /
   `engine-stderr.log`; timeout 10 minutes → kill + `EngineTimeoutError`.
   Record upstream exit code verbatim.
4. `results.ts` — `collectChecks(runDir)`: find `results/server-*/checks.json`
   (newest mtime wins if several); parse defensively: accept unknown extra fields,
   require per-check at minimum an id-ish field (first present of `id`, `name`,
   `scenario`) and a status-ish field. **Status mapping table** (case-insensitive;
   confirm/extend against the captured 0.1.16 fixtures and note deviations in the
   PR):

   | upstream status value | Report `CheckStatus` |
   |---|---|
   | `pass`, `passed`, `success` | `pass` |
   | `fail`, `failed`, `failure` | `fail` |
   | `skip`, `skipped`, `pending` | `skip` |
   | failing + marked expected (e.g. `expected: true` / baseline hit) | `expected_fail` |
   | passing + marked expected-to-fail | `unexpected_pass` |
   | anything else | `fail`, with the raw value quoted in `detail` |

   Missing/unparseable file → `EngineOutputError` including the last 20 lines of
   engine-stderr.
5. Top entry `runOfficial(http: HttpTarget, specVersion: "2025-11-25", runDir:
   string): Promise<OfficialSection>` composing 1–4; `status: "pass"` iff upstream
   exit code 0; `engine.version` read from the resolved engine's package.json (the
   same file used for bin resolution).
6. **Contract tests** (the ADR-001 tripwire; test names prefixed `contract:`):
   - resolve the bin from the installed dep and run `--help` (or `--version`);
     snapshot must contain the `server` subcommand and `--url`, `--suite`,
     `--spec-version`, `--expected-failures` flags;
   - parse two committed fixture `checks.json` captured from a real 0.1.16 run
     (implementer captures during this issue and commits under
     `test/fixtures/engine/`), one all-pass and one with failures;
   - assert the dependency is declared exact (no `^`/`~`) by reading the kit's own
     package.json in a test.

## Acceptance Criteria

- [ ] All arg-mapping combinations unit-tested (suite default, scenarios repeated,
      expected-failures missing-file error).
- [ ] Contract tests pass against the installed 0.1.16 and fail (verified once by
      simulation — e.g. renaming a fixture field) with a message naming the adapter
      as the fix site.
- [ ] No occurrence of `npx` or version ranges for the engine (grepped in a test).
- [ ] `runOfficial` never throws for engine test failures (that's a result), only for
      infrastructure errors (typed).

## Validation

Kit unit tests green in CI; attach one captured real `checks.json` (trimmed) and the
`--help` snapshot to the PR.

## Dependencies

04, 05.

## Non-goals

Server lifecycle (05), terminal/summary rendering (08), retry logic (none in v1),
supporting the 0.2.0-alpha line (Known unknown U2 — separate PR when it stabilizes).

## Design References

DESIGN.md §2.1, §6.4, §7.4, §9 BND-5; ADR-001; research survey §2.
