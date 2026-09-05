# Title

Author stdio entrypoint template files

## Summary

Author `templates/ts/stdio/src/stdio.ts` (StdioServerTransport entrypoint following
the stderr-only logging rule and clean-shutdown contract) plus its manifest entries,
so `stdio` and `both` variants get a runnable local server that passes the kit's
smoke checks.

## Context

DESIGN.md §5.3 fixes the stdio rules; the smoke catalog (§6.5) is the executable
spec this entrypoint must satisfy — notably SMOKE-STDOUT-01 (stdout is JSON-RPC
only) and SMOKE-SHUTDOWN-01 (exit ≤5s after transport close).

## Scope

One template source file + manifest entries (`when.transport: ["stdio","both"]`).
Smoke-check *wiring* (scripts/config) is issue 15.

## Detailed Requirements

1. `src/stdio.ts`:
   - imports `createServer` from `./server.js` (NodeNext ESM specifier);
   - `const server = createServer(); const transport = new StdioServerTransport();
     await server.connect(transport);`
   - startup notice via a tiny local `logErr` helper writing to `process.stderr`
     only; a comment cites the 2025-11-25 rule "stdio servers may log to stderr,
     never stdout";
   - `transport.onclose` → `await server.close()` → `process.exit(0)`;
   - SIGINT/SIGTERM handlers: close server + transport, exit 0; double-signal →
     immediate exit 1;
   - top-level try/catch: fatal errors to stderr, exit 1;
   - **zero** `console.log` (which writes to stdout) anywhere.
2. Manifest: add the file with `when.transport: ["stdio","both"]`.
3. Extend the template-compile harness set (12) — file typechecks with strict tsc.
4. Add a harness-level behavioral test: spawn `tsx` (or the harness's compiled
   output) of stdio.ts wired to the template `createServer`, connect the SDK
   `Client` via `StdioClientTransport`, run initialize + one echo call, close,
   assert exit code 0 within 5s and that nothing non-JSON-RPC appeared on stdout
   (reuse the detection approach standardized in issue 07).

## Acceptance Criteria

- [ ] Behavioral test passes: initialize + echo + clean exit(0) ≤5s.
- [ ] Static check in tests: template file contains no `console.log` token and no
      direct `process.stdout` writes.
- [ ] `tsc --noEmit` clean in the harness.
- [ ] Manifest coverage test still green.

## Validation

CI green; behavioral test output pasted in PR.

## Dependencies

12.

## Non-goals

HTTP entry (14), npm scripts and conformance config (15), watch/dev tooling beyond
what 12 already defined, Windows signal-handling nuances (best-effort).

## Design References

DESIGN.md §4.1, §5.3, §6.5 (SMOKE-STDOUT-01, SMOKE-SHUTDOWN-01).
