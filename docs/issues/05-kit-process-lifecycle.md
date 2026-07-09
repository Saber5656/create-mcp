# Title

Implement server process lifecycle (spawn, readiness, stop)

## Summary

Implement `lifecycle/spawn.ts`, `lifecycle/ready.ts`, `lifecycle/stop.ts`: start the
server-under-test from an argv array (never a shell), capture its output to log
files, wait until the HTTP endpoint is ready, and stop it reliably (SIGTERM → 5s →
SIGKILL on the whole process group), including on kit crash paths.

## Context

DESIGN.md §6.1 module layout and §9 BND-3: config-driven commands run argv-only with
the user's own privileges; orphaned server processes must not survive a kit run.
Consumed by the adapter (06) and CLI (09).

## Scope

Three modules + unit tests with a throwaway fixture script (`test/fixtures/`
long-running Node one-liners). HTTP readiness only (stdio smoke spawning is owned by
issue 07 via the SDK client).

## Detailed Requirements

1. `spawn.ts` — `startServer(target: HttpTarget, runDir: string): Promise<RunningServer>`:
   - `child_process.spawn(argv[0], argv.slice(1), { cwd, shell: false, detached: true,
     stdio: ["ignore", "pipe", "pipe"] })`; `detached: true` so the negative PID kills
     the group. `target.cwd` is already absolute when it reaches this module (the
     config loader resolves relative paths against the config file's directory —
     issue 04); absent `cwd` defaults to the config file's directory.
   - stdout/stderr piped to `<runDir>/server-stdout.log` / `server-stderr.log`
     (create runDir; append mode off — one file per run).
   - `RunningServer = { pid, logs: {stdout, stderr}, stop(): Promise<StopResult>,
     exited: Promise<{code, signal}> }`.
   - Spawn failure (ENOENT etc.) → typed `ServerStartError` carrying the argv[0] name.
2. `ready.ts` — `waitUntilReady(url, timeoutMs, running: RunningServer): Promise<void>`
   (takes the full `RunningServer` so it can race on `running.exited` and read
   `running.logs.stderr` for error tails):
   - Poll every 250 ms with `fetch(url, { method: "GET" })` and 1s per-attempt
     abort; **any HTTP response** (including 4xx/405 — Streamable HTTP servers may
     reject bare GET) counts as ready; only network-level failures keep polling.
   - Races against `running.exited` — early server death → `ServerStartError` that
     includes the last 20 lines of `running.logs.stderr`.
   - Timeout → `ReadyTimeoutError` (also with stderr tail).
3. `stop.ts` — `stop()`:
   - `process.kill(-pid, "SIGTERM")`; if not exited within 5s, `process.kill(-pid,
     "SIGKILL")`; resolve with `{ method: "sigterm" | "sigkill" | "already-exited" }`.
   - Idempotent; ESRCH treated as already-exited.
4. Crash safety: module registers a `process.on("exit")` best-effort SIGKILL for any
   still-running child it started (a module-level registry), so a kit bug can't leak
   servers.
5. No environment mutation: child inherits `process.env` as-is (DESIGN §7.3 note).

## Acceptance Criteria

- [ ] Unit tests (with real spawned Node fixture processes): successful start+ready
      against a fixture HTTP server; ready on 405 response; ENOENT command;
      early-exit command (stderr tail present in error); readiness timeout; SIGTERM
      honored; SIGTERM-ignoring fixture gets SIGKILLed ≤6s; double `stop()` safe.
- [ ] After every test, `ps` shows no leaked fixture processes (test helper asserts).
- [ ] No `shell: true` or string-command spawn anywhere (grep in test).
- [ ] Log files exist and contain the fixture's output after a run.

## Validation

`pnpm --filter @saber5656/mcp-conformance-kit test` green locally and in CI on
Node 22+24 (timing-sensitive tests must use generous margins; no flaky sleeps —
poll with deadlines).

## Dependencies

04 (HttpTarget type).

## Non-goals

Official-suite invocation (06), stdio client spawning (07), CLI wiring (09),
Windows process-group semantics (best-effort only; not CI-tested — v1 non-goal).

## Design References

DESIGN.md §6.1, §7.3 (env note), §9 BND-3.
