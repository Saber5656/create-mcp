# Title

Implement stdio smoke-check catalog and runner

## Summary

Implement `smoke/checks.ts` (the 11-check catalog from DESIGN.md §6.5) and
`smoke/run.ts` (SDK `Client` + `StdioClientTransport` executor) so the kit can verify
stdio MCP servers: handshake, stdout hygiene, tools/resources/prompts round-trips,
SEP-1303 error semantics, and clean shutdown. Output feeds the shared Report model.

## Context

The official suite cannot test stdio servers (research §2), so these smoke checks are
the kit's own contribution — clearly labeled smoke checks, never "official
conformance" (ADR-001). Catalog and MUST/SHOULD semantics are fixed in DESIGN.md §6.5.

## Scope

Two modules + a test fixture stdio server (SDK-based, well-behaved) + misbehaving
fixture variants. No CLI (09), no terminal rendering (08).

## Detailed Requirements

1. Check interface: `{ id: string, level: "MUST" | "SHOULD", title: string,
   run(ctx: SmokeContext): Promise<CheckOutcome> }` where `SmokeContext` exposes the
   connected SDK `Client`, negotiated init result, and a `stdoutViolations` accessor
   (see req 4). `CheckOutcome = { status: "pass" | "fail" | "skip", detail?: string }`.
2. Implement exactly the 11 checks SMOKE-INIT-01 … SMOKE-SHUTDOWN-01 as specified in
   DESIGN.md §6.5, including the `MUST*` conditional rule: capability-gated checks
   report `skip` with detail "capability not declared" when absent.
   SMOKE-INIT-01 verdict rule: the negotiated `protocolVersion` must equal
   `"2025-11-25"` exactly — anything else is a MUST **fail** with both the
   negotiated and expected values in `detail` (no soft "reported" state).
3. Sample-argument generation for SMOKE-TOOL-02/03: from the tool's JSON
   `inputSchema`, support object schemas whose properties are `string` (use
   `"sample"` or first `enum` value), `number`/`integer` (`1`), `boolean` (`true`);
   required-only population. Any other shape → `skip` with reason (never guess).
   For SMOKE-TOOL-03, invalid args = required string property sent as `12345`
   (number); if no such property exists → `skip`.
   SMOKE-PROMPT-02 argument generation follows the same leaf-schema rule applied to
   the prompt's declared arguments (required, string-valued: use `"sample"`); a
   prompt whose required arguments fall outside that rule → `skip` with reason.
4. stdout hygiene (SMOKE-STDOUT-01): run the child via
   `StdioClientTransport({ command, args, cwd })`. Since the SDK owns the pipe,
   detect pollution as follows: transport-level parse errors surfaced by the SDK
   (`onerror`) count as violations. Record them via the client error handler into
   `stdoutViolations`. (This catches interleaved non-JSON output — the realistic
   failure — without re-implementing the transport.)
5. Runner `runSmoke(stdio: StdioTarget, kitMeta): Promise<SmokeSection>`:
   - connect with 10s init timeout; init failure → SMOKE-INIT-01 fail and remaining
     checks `skip` (detail "not connected");
   - execute checks sequentially in catalog order (they share one session);
   - SMOKE-SHUTDOWN-01: `await client.close()` then watch the child exit ≤5s with
     code 0. Exit observation contract: obtain the child PID from the SDK
     transport's exposed process accessor (`StdioClientTransport` exposes the
     spawned child/pid in SDK 1.x — verify the exact property against the installed
     SDK and record it in the PR), then poll `process.kill(pid, 0)` until ESRCH or
     deadline. If the installed SDK exposes no such accessor, fail this issue's
     implementation back to design review rather than reaching into private fields
     — do not ship a private-field dependency;
   - `status: "fail"` iff any MUST check failed; SHOULD failures keep
     `status: "pass"`. Do not implement a `--strict` flag in v1 (reserved for
     v1.1 per DESIGN §6.5).
6. All server-provided strings placed into `detail` must pass through the report
   model's `truncateDetail` helper (owned by issue 04): max 500 chars, marker on
   truncation. (Sanitization of control characters is 08's renderer job; detail
   strings stay raw-but-truncated in the model.)
7. Fixtures under `test/fixtures/smoke/`: `good-server.ts` (echo tool with required
   string arg, one resource, one prompt, proper stderr logging), `bad-stdout.ts`
   (console.log noise before/while serving), `no-exit.ts` (ignores close),
   `protocol-error-on-bad-args.ts` (violates SEP-1303).

## Acceptance Criteria

- [ ] Against `good-server.ts`: all 11 checks `pass`.
- [ ] Against `bad-stdout.ts`: SMOKE-STDOUT-01 `fail`, section `status: "fail"`.
- [ ] Against `no-exit.ts`: SMOKE-SHUTDOWN-01 `fail` (SHOULD) but section still
      `"pass"`; child is force-killed by the test teardown (no leaks).
- [ ] Against `protocol-error-on-bad-args.ts`: SMOKE-TOOL-03 `fail` with detail
      naming SEP-1303.
- [ ] A tools-only fixture yields `skip` for RES/PROMPT checks with the exact detail
      string "capability not declared".
- [ ] Catalog IDs/levels in code match DESIGN.md §6.5 table byte-for-byte (unit test
      compares against a literal list).

## Validation

Kit tests green in CI Node 22+24; runtimes bounded (whole smoke suite over fixtures
< 60s).

## Dependencies

04.

## Non-goals

HTTP checks (official suite's job), `--strict` flag behavior (09/v1.1), fuzzing or
schema-based property testing (v2), rendering (08).

## Design References

DESIGN.md §6.5, §7.4; ADR-001 (labeling); spec 2025-11-25 changelog (SEP-1303,
stderr logging) via research survey §1.
