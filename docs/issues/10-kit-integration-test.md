# Title

Add kit integration test against a fixture MCP server

## Summary

Add a real end-to-end integration test for the kit: a fixture Streamable HTTP MCP
server (SDK-based) is built and started by the kit itself from a config file, the
exactly-pinned official conformance suite runs against it, smoke checks run against
its stdio twin, and the resulting report/exit codes are asserted. Wire this as a
separate CI job.

## Context

Issues 04–09 are unit-tested with fakes; this issue proves the composed kit works
against the *real* official suite (DESIGN §10 "Kit integration" row) and produces the
first evidence for Known unknown U3 (whether an expected-failures baseline is needed).

## Scope

`packages/conformance-kit/test/fixtures/http-server/` (fixture server),
`test/integration/run.test.ts`, and a `kit-integration` CI job. No template work.

## Detailed Requirements

1. Fixture server: minimal SDK 1.29 `McpServer` with echo tool + one resource +
   one prompt, `StreamableHTTPServerTransport` bound to `127.0.0.1:3799` (fixed
   test port, chosen to avoid common dev-port clashes), stdio twin entry sharing
   the same server definition. Fixtures are **prebuilt with `tsc`** in a test
   setup step (`tsc -p test/fixtures/tsconfig.json`) — do not add `tsx` or any new
   runtime dependency.
2. Integration test:
   - creates a temp run directory and writes `conformance.config.json` into it
     (http + stdio sections, port 3799, `suite: "active"`,
     `specVersion: "2025-11-25"`; `reportDir` left at its default);
   - invokes the **built** CLI (`node dist/bin.js run --config <tmpdir>/conformance.config.json`)
     as a subprocess with `cwd = <tmpdir>`;
   - asserts exit code 0; `<tmpdir>/conformance-results/report.json` exists at that
     exact path (default `reportDir` resolves against the config file's directory —
     issue 04) and is `summarize`-consistent; official section engine version
     equals the pinned 0.1.16; smoke section all-MUST pass.
3. If the real suite fails scenarios for the fixture server (U3): triage each failing
   scenario; fix the fixture if it's a fixture bug; if it is an upstream/spec gap,
   add `expected-failures.yaml` with a dated comment per entry and link evidence in
   the PR. The test then asserts exit 0 **with** that baseline and the report showing
   `expected_fail` statuses.
4. CI: new job `kit-integration` in ci.yml (needs: quality, Node 24 only —
   ISSUE_PLAN §1 fixes the matrix policy: quality on 22+24, heavier jobs on 24;
   `timeout-minutes: 20`) running `pnpm --filter @saber5656/mcp-conformance-kit
   test:integration` (separate vitest config/tag so unit stays fast).
5. Record actual wall-clock of the job in the PR (baseline for later regressions).

## Acceptance Criteria

- [ ] Integration test green in CI; total job < 10 min.
- [ ] `report.json` from the run uploaded as a CI artifact (retention 7 days).
- [ ] Any expected-failures baseline entries carry a comment with date + upstream
      issue/evidence link (empty baseline is the ideal outcome).
- [ ] Re-running the job twice in a row is stable (no port clashes — server killed;
      no timestamped-path collisions).

## Validation

CI run link ×2 (stability), artifact inspected once manually.

## Dependencies

09.

## Non-goals

Testing generated templates (23/24 do that), performance tuning, running the `draft`
suite (v1.1 concern).

## Design References

DESIGN.md §10 (integration row), §6.2–6.5; ISSUE_PLAN §8 U3.
