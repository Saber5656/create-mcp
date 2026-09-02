# Review resolution addendum

- Repository: `Saber5656/create-mcp`
- Pull request: #1
- Original PR head before this resolution addendum: `0a5d11d363f39d1cf5b19552cd5e0737890e9a0a`
- The immutable current PR head is supplied by the parent task's fresh GitHub read immediately before review/reply/resolve; any later head change invalidates that evidence and requires a fresh review.
- Scope: each exact review thread below has a normative design contract and a focused verification gate.
- This is design-level handling only; it does not claim implementation, test, build, CI, or security validation is complete.
- Per task instruction, the PR review bot is not re-triggered after these responses/resolutions.

## 1. Thread `PRRT_kwDOTN39zM6PuhA5` — Align the `conformance` command semantics.

**Normative resolution**: The primary conformance flow explicitly runs `npm run build` before starting `npm run conformance`; builds are not implicit, and all examples/acceptance criteria use the same contract.

**Focused verification gate**: Run from a clean checkout with no `dist`; assert the documented build-then-conformance flow succeeds and conformance alone fails clearly without implying a build.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 2. Thread `PRRT_kwDOTN39zM6PuhA_` — Sanitize GitHub Step Summary content, not only terminal output.

**Normative resolution**: Every human-readable renderer, including `github-summary.ts`, applies bounded Markdown-safe escaping/link policy and terminal-control sanitization to server-controlled detail strings before writing the Step Summary.

**Focused verification gate**: Render details containing Markdown links, HTML, ANSI/C0/C1 controls, long text, and newlines; assert safe bounded summary output and no injected links/markup.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 3. Thread `PRRT_kwDOTN39zM6PuhBB` — Define the `--strict` CLI option or remove this behavior.

**Normative resolution**: The smoke-check contract either defines and parses `--strict` with its failure semantics or removes the flag and SHOULD-failure behavior from all docs/tests; no undocumented option remains.

**Focused verification gate**: Compare help/parser/docs and run strict/non-strict smoke cases; assert exact exit behavior and no dead option.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 4. Thread `PRRT_kwDOTN39zM6PuhBD` — Clarify whether expected failures still produce a green badge.

**Normative resolution**: Green means the official suite passes under the configured expected-failure baseline; the report and README distinguish raw failures, expected failures, and unexpected failures.

**Focused verification gate**: Run all-pass, expected-failure, and unexpected-failure scenarios and assert badge color, exit code, and summary text agree.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 5. Thread `PRRT_kwDOTN39zM6PuhBE` — Make expected-status precedence explicit.

**Normative resolution**: The parser first evaluates the enumerated expected/baseline marker keys, then applies the generic status mapping. Accepted status fields, marker fields, and precedence are listed in one canonical algorithm.

**Focused verification gate**: Test failed/pass/warning records with each marker spelling and conflicting values; assert expected failures never become generic failures due to first-match order.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 6. Thread `PRRT_kwDOTN39zM6PuhBH` — Align the SMOKE-INIT-01 protocol-version rule.

**Normative resolution**: SMOKE-INIT-01 names one canonical MCP protocol version, comparison rule, and supported-version error; generated templates, conformance wrapper, and acceptance test use that same value.

**Focused verification gate**: Exercise exact, compatible, unsupported, missing, and malformed protocol-version responses; assert one deterministic pass/fail mapping.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 7. Thread `PRRT_kwDOTN39zM6PuhBK` — Call the terminal renderer with the report and resolved color option.

**Normative resolution**: The flow obtains the report, resolves `--no-color`/TTY policy, and writes `process.stdout.write(renderTerminal(report, { color }))`; it never passes raw stdout where a report object is required.

**Focused verification gate**: Run TTY/non-TTY and color/no-color combinations and assert renderer input, output, and exit status are correct.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 8. Thread `PRRT_kwDOTN39zM6PuhBN` — Use an allocated port instead of hard-coding `3799`.

**Normative resolution**: HTTP smoke tests allocate a free ephemeral port and pass that value through generated config/server startup; a fixed fallback is not used without collision handling.

**Focused verification gate**: Run concurrent jobs and reserve the former fixed port; assert each test uses an isolated free port and reports the resolved value.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 9. Thread `PRRT_kwDOTN39zM6PuhBP` — Make the template harness resolve runtime dependencies explicitly.

**Normative resolution**: The harness validates templates in an isolated environment with an explicit install/symlink of `@modelcontextprotocol/sdk`, `zod`, `express`, and required type packages; accidental parent-workspace resolution is prohibited.

**Focused verification gate**: Run from a temporary directory with no node_modules, then with intentionally incompatible parent modules; assert only the explicit dependency setup passes.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 10. Thread `PRRT_kwDOTN39zM6PuhBT` — Make stdout cleanliness a runtime acceptance check.

**Normative resolution**: Acceptance criteria retain the runtime JSON-RPC stdout assertion: generated stdio servers must emit protocol frames only on stdout, with logs on stderr, and static grep is supplementary.

**Focused verification gate**: Run generated servers that log to stdout, stderr, and both; parse stdout as JSON-RPC and assert contamination fails the test.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 11. Thread `PRRT_kwDOTN39zM6PuhBX` — Define the error-middleware ordering for the factory seam.

**Normative resolution**: `createHttpApp()` exposes a finalization seam: routes are registered before the error middleware is installed, or the factory returns an explicit `installErrorHandler()` step. Tests mount a throwing route before finalization and assert the canonical 500 body.

**Focused verification gate**: Mount routes before/after finalization, throw sync/async errors, and assert only the supported ordering produces `500 {"error":"internal error"}`.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 12. Thread `PRRT_kwDOTN39zM6PuhBZ` — Preserve exit 3 for target-directory validation failures.

**Normative resolution**: Target-directory validation is a typed failure outside generic option parsing and maps `not-empty`, `forbidden`, and `invalid` results to exit 3; generic option errors remain exit 2.

**Focused verification gate**: Run each directory condition plus malformed CLI options and assert exact exit codes/messages.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 13. Thread `PRRT_kwDOTN39zM6PuhBb` — Validate the nearest existing ancestor for nonexistent targets.

**Normative resolution**: For a nonexistent target, resolve and validate the nearest existing ancestor (including symlink canonicalization), then re-check containment immediately before creation/copy; a symlinked ancestor into the installation directory is rejected.

**Focused verification gate**: Test nonexistent paths under safe and forbidden symlink ancestors, race the ancestor before creation, and assert no outside write occurs.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 14. Thread `PRRT_kwDOTN39zM6PuhBh` — Make the packed CLI the normative generation path.

**Normative resolution**: The conformance harness invokes the installed packed/tarball CLI as the normative path; workspace-built `dist/index.js` is an explicitly optional local-debug shortcut only.

**Focused verification gate**: Pack/install in a clean temporary project, invoke the installed binary, and assert generated output matches the acceptance contract without workspace resolution.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 15. Thread `PRRT_kwDOTN39zM6PuhBj` — Restrict publishing to the protected `main` ref.

**Normative resolution**: The release/publish job has an explicit `github.ref == 'refs/heads/main'` guard and documentation states recovery dispatches must target `main`; branch protection is an additional gate, not the only one.

**Focused verification gate**: Trigger workflow_dispatch from main, another branch, tag, and fork-like context; assert only protected main can publish.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 16. Thread `PRRT_kwDOTN39zM6PuhBl` — Do not execute arbitrary Markdown shell blocks in CI.

**Normative resolution**: CI never executes PR-controlled Markdown blocks. Verification uses a fixed command allowlist/static parser in a no-secrets restricted job, or dedicated checked-in scripts with no repository credentials and confined writes.

**Focused verification gate**: Place shell injection, redirection, command substitution, and credential-looking blocks in Markdown; assert no arbitrary command executes and the fixed checks still run.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 17. Thread `PRRT_kwDOTN39zM6PumWI` — Pass an output directory to the conformance CLI.

**Normative resolution**: The conformance invocation always passes a unique `--output-dir`/`-o` to the runner, and `collectChecks` reads that exact directory; missing output remains a typed failure.

**Focused verification gate**: Run a normal HTTP suite and assert every expected `checks.json` is created under the passed directory and collected.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 18. Thread `PRRT_kwDOTN39zM6PumWK` — Collect every suite result instead of only the newest file.

**Normative resolution**: The collector enumerates and validates every scenario's `checks.json` under the run output directory, preserves scenario identity/order, and aggregates all results into report/summary.

**Focused verification gate**: Run multiple active scenarios with mixed pass, expected-fail, and unexpected-fail results; assert none is dropped and aggregate badge/exit matches all suites.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 19. Thread `PRRT_kwDOTN39zM6PumWM` — Create a transport per HTTP session.

**Normative resolution**: The HTTP template creates a fresh stateful `StreamableHTTPServerTransport` per initialized session and disposes it at session end; the process-level app does not reuse one initialized transport across scenarios.

**Focused verification gate**: Run multiple initialize/session scenarios in one process and assert each receives a unique session transport and valid lifecycle.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 20. Thread `PRRT_kwDOTN39zM6PumWP` — Provide pnpm version to the generated workflow.

**Normative resolution**: The generated workflow pins a supported pnpm version in `pnpm/action-setup`, or generated `package.json` contains an equivalent pinned `packageManager`; the two sources cannot be ambiguous.

**Focused verification gate**: Generate a project and run the workflow parser/action setup from a clean runner; assert pnpm is installed before install.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 21. Thread `PRRT_kwDOTN39zM6PumWR` — Preserve env vars for stdio smoke servers.

**Normative resolution**: Generated `StdioClientTransport` receives the documented explicit copy of `process.env` (with any allowlist/redaction policy), matching DESIGN §7.3; required API keys/feature flags are not silently dropped.

**Focused verification gate**: Run a stdio fixture that reads an environment marker under the generated kit and assert it matches the parent script while forbidden injection remains blocked.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 22. Thread `PRRT_kwDOTN39zM6PumWT` — Don’t await a close method that force-kills before checking shutdown.

**Normative resolution**: SMOKE-SHUTDOWN-01 observes child exit/acknowledgement while initiating graceful shutdown, before any force-killing `close()` completes, or uses a lower-level child handle that reports voluntary versus forced exit.

**Focused verification gate**: Use a fixture that exits gracefully and one that ignores shutdown; assert the test distinguishes voluntary exit from SIGTERM/SIGKILL and enforces the SHOULD behavior.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 23. Thread `PRRT_kwDOTN39zM6PumWX` — Map upstream WARNING/INFO statuses explicitly.

**Normative resolution**: The wrapper maps upstream `WARNING` and `INFO` to their documented non-failure categories before applying unknown-status fallback; only `FAILURE`/unexpected statuses affect failure counts.

**Focused verification gate**: Feed every upstream status plus unknown values and assert check classification, badge, summary, and exit code.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 24. Thread `PRRT_kwDOTN39zM6PumWa` — Configure allowedOrigins for Origin rejection.

**Normative resolution**: The generated HTTP server configures both `allowedHosts` and the explicit `allowedOrigins` policy required by the threat model, including same-origin/localhost values; hostile Origin with valid Host is rejected.

**Focused verification gate**: Send valid/invalid Host and Origin combinations, absent Origin, preflight, and localhost cases; assert exact 403/allowed behavior.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 25. Thread `PRRT_kwDOTN39zM6PumWc` — Reference the generated baseline when it becomes non-empty.

**Normative resolution**: Generated configs reference the canonical `expectedFailures` baseline stub/file and the runner applies it whenever non-empty; the empty-baseline path is explicit and does not hide upstream failures.

**Focused verification gate**: Generate with empty and populated baselines, run mixed expected/unexpected failures, and assert baseline matching affects only expected outcomes.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 26. Thread `PRRT_kwDOTN39zM6PumWe` — Generate a publishable kit dependency spec.

**Normative resolution**: Generated projects receive a publish-time dependency spec resolved to published package versions, never a monorepo-only `workspace:` protocol; `kit-version` and package rewriting are derived from the same registry/version source.

**Focused verification gate**: Pack the kit, generate outside the monorepo, install with npm/pnpm, and assert dependency resolution succeeds without workspace links.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 27. Thread `PRRT_kwDOTN39zM6PumWi` — Don’t pass multiple --scenario flags to v0.1.16.

**Normative resolution**: For multiple scenarios the wrapper invokes the pinned CLI once per scenario (or rejects multi-scenario configs); it never repeats a single-valued `--scenario` option and falsely reports full coverage.

**Focused verification gate**: Configure one, multiple, duplicate, and invalid scenarios; assert invocation count, scenario identity, and aggregate report.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 28. Thread `PRRT_kwDOTN39zM6PumWo` — Apply expected-failure mapping outside checks.json.

**Normative resolution**: The wrapper applies configured baseline matching to raw `checks.json` records and annotates the derived report with `expected_fail`; upstream process exit alone is not used to infer per-check expectation.

**Focused verification gate**: Use a raw FAILURE matching and not matching the baseline, with mixed checks; assert report annotations, badge, and exit code distinguish expected from unexpected.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.