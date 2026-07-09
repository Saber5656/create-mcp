# Title

Author Streamable HTTP entrypoint template with secure defaults

## Summary

Author the HTTP entrypoint pair for the `http` and `both` variants —
`templates/ts/http/src/http-app.ts` (exported factory: express 5 app +
`StreamableHTTPServerTransport`, the test seam) and `templates/ts/http/src/http.ts`
(thin executable entry) — with every secure default from DESIGN.md §5.4: loopback
bind, Origin/DNS-rebinding protection (403), SDK session management, 1 MB body cap,
error hygiene, graceful shutdown.

## Context

This file is the template's biggest attack surface (BND-6) and the target the
official conformance suite tests (via issue 15/16 wiring). Spec 2025-11-25 makes
invalid-Origin → HTTP 403 a MUST (research §1). Known unknown U4: exact SDK option
names for rebinding protection must be verified against installed SDK 1.29.x.

## Scope

Two template source files (`http/src/http-app.ts`, `http/src/http.ts`) + manifest
entries (`when.transport: ["http","both"]`) + behavioral tests in the template
harness. Conformance config ships in 15.

## Detailed Requirements

1. `src/http-app.ts` — the factory and test seam (DESIGN §5.4):
   - exports `createHttpApp(options: { host: string; port: number }): { app:
     express.Express; transport: StreamableHTTPServerTransport; close(): Promise<void> }`;
   - express 5 app; `express.json({ limit: "1mb" })`;
   - one `StreamableHTTPServerTransport` configured with session management
     (`sessionIdGenerator: () => randomUUID()`) and DNS-rebinding/Origin protection
     enabled with allowed hosts `127.0.0.1:<port>` and `localhost:<port>`.
     **U4 step**: implementer verifies exact option names against the installed SDK
     (`node_modules/@modelcontextprotocol/sdk` types), records them in the PR, and
     updates DESIGN.md §5.4 if names differ from `enableDnsRebindingProtection`/
     `allowedHosts`;
   - `app.all("/mcp", handler)` delegating POST/GET/DELETE to the transport per SDK
     idiom; unknown routes → 404 JSON.
   `src/http.ts` — thin entry: read env `PORT` (default `3000`) / `HOST` (default
   `127.0.0.1`), validate (port integer 1–65535) with fail-fast stderr message,
   call `createHttpApp`, `app.listen(port, host)`, install signal handlers.
   Behavioral tests import the factory and listen on an ephemeral port; only the
   shutdown test spawns the built entry.
2. Behavior required by tests:
   - request with `Origin: https://evil.example` → **403**;
   - request without Origin from loopback → served;
   - initialize → `mcp-session-id` response header present; follow-up request with
     that session id succeeds; bogus session id → 4xx (SDK semantics — assert
     non-2xx);
   - body >1 MB → 413;
   - unexpected handler error → 500 JSON `{"error":"internal error"}`, stack only on
     stderr (test mounts an extra throwing route on the factory-returned `app` —
     the template stays clean; the express error middleware under test is the
     template's);
   - `HOST` default binds loopback: the shutdown test spawns the built entry with
     no `HOST` set and asserts the logged listen address is `127.0.0.1` (and the
     factory test asserts `server.address().address === "127.0.0.1"` when
     listening with the default host value).
3. Shutdown: SIGINT/SIGTERM → stop accepting (`httpServer.close`), close transport +
   server, exit 0 ≤5s; force path exit 1 on second signal. README warning comment
   (one line) at the `HOST` read: "0.0.0.0 exposes this server to your network — add
   auth first (see README security notes)".
4. Startup log to **stderr** (`listening on http://127.0.0.1:3000/mcp`) so stdout
   stays clean in all variants (consistency with §5.3 policy).
5. Manifest entries + compile-harness inclusion.

## Acceptance Criteria

- [ ] Behavioral tests for every bullet in requirement 2 pass in CI (supertest or
      raw fetch against an ephemeral port instance of the template code).
- [ ] Shutdown test: SIGTERM → exit 0 ≤5s.
- [ ] `tsc --noEmit` clean; manifest coverage green.
- [ ] U4 verification note (actual SDK option names + SDK version) recorded in the
      PR and reflected in DESIGN.md if they differ.

## Validation

CI green; paste the 403-Origin test output and the U4 note in the PR.

## Dependencies

12.

## Non-goals

Authentication/OAuth (v1 non-goal; README documents), TLS termination, multi-session
scaling concerns, conformance wiring (15), rate limiting (the SDK's express
integration defaults suffice for a localhost dev server).

## Design References

DESIGN.md §5.4 (full table), §9 BND-6, §12 (auth non-goal); ISSUE_PLAN §8 U4;
research §1 (Origin 403 MUST).
