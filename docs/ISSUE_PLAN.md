# create-mcp — v1 Issue Plan

Status: Approved planning baseline (2026-07-10). Derived from `docs/DESIGN.md`.
GitHub Issues are generated from `docs/issues/NN-*.md`; if they diverge, these files
win and the GitHub Issues are stale.

## 1. v1 completion statement

v1 is complete when **all 27 issues below are closed with their acceptance criteria
validated**, at which point the following must be true end to end:

1. The create-mcp CLI generates a TypeScript MCP server project in any of the three
   transport variants (`stdio`, `http`, `both`), interactively or fully
   non-interactively (`--yes`/flags) — proven from the packed npm artifact by the
   dogfood job. The `npm create @saber5656/mcp` / `pnpm create @saber5656/mcp` /
   `npx @saber5656/create-mcp` invocations become live once the user completes the
   documented first publish (issue 25's RELEASING.md checklist — publishing
   credentials are a manual user step by policy).
2. Every generated project builds, its unit tests pass, and `npm run conformance`
   runs the exactly-pinned official `@modelcontextprotocol/conformance` suite
   (HTTP variants) plus the kit's stdio smoke checks (stdio variants) against spec
   **2025-11-25**, producing `conformance-results/report.json` and a terminal report.
3. Every generated project contains a SHA-pinned GitHub Actions conformance workflow
   whose status badge is embedded in its README (owner/repo placeholder; activation
   is a documented one-line step after first push — ADR-006), with a per-run summary
   table in `$GITHUB_STEP_SUMMARY`.
4. Both packages (`@saber5656/create-mcp`, `@saber5656/mcp-conformance-kit`) are
   publishable via the Changesets version-PR release workflow with npm provenance
   (manual npm account/scope prerequisites documented), and this repo's CI enforces
   lint, typecheck, unit tests on Node 22 and 24, plus integration, CLI e2e, and the
   dogfood e2e (generate → install → official suite green) on Node 24.
5. The security model in DESIGN.md §9 is implemented: every BND-1..8 mitigation is
   covered by a shipped issue and validated by a test or documented manual check.

Remaining product behavior is explicitly listed as v1 non-goals / deferred (§7) —
nothing in the v1 promise lives outside this issue list.

## 2. Issue list (recommended execution order)

| # | File | Title (GitHub issue title) | Package/Area |
|---|---|---|---|
| 01 | `issues/01-monorepo-scaffold.md` | Scaffold pnpm monorepo with package skeletons and shared toolchain | repo |
| 02 | `issues/02-repo-ci-quality-workflow.md` | Add repository CI workflow for lint, typecheck, and unit tests | repo |
| 03 | `issues/03-community-docs.md` | Add root README, CONTRIBUTING, SECURITY, and Code of Conduct | repo |
| 04 | `issues/04-kit-config-and-report-model.md` | Implement kit config schema and report data model | kit |
| 05 | `issues/05-kit-process-lifecycle.md` | Implement server process lifecycle (spawn, readiness, stop) | kit |
| 06 | `issues/06-kit-official-suite-adapter.md` | Implement adapter around the official conformance suite | kit |
| 07 | `issues/07-kit-stdio-smoke-checks.md` | Implement stdio smoke-check catalog and runner | kit |
| 08 | `issues/08-kit-report-rendering.md` | Implement report renderers (terminal, JSON, GitHub summary) | kit |
| 09 | `issues/09-kit-cli.md` | Implement mcp-conformance-kit CLI commands and exit codes | kit |
| 10 | `issues/10-kit-integration-test.md` | Add kit integration test against a fixture MCP server | kit |
| 11 | `issues/11-template-manifest-schema.md` | Implement template manifest schema and copy-plan resolver | cli |
| 12 | `issues/12-template-base-project.md` | Author base template (server, capabilities, unit tests, configs) | templates |
| 13 | `issues/13-template-stdio-entry.md` | Author stdio entrypoint template files | templates |
| 14 | `issues/14-template-http-entry.md` | Author Streamable HTTP entrypoint template with secure defaults | templates |
| 15 | `issues/15-template-conformance-wiring.md` | Wire conformance config and scripts into templates | templates |
| 16 | `issues/16-template-ci-workflow.md` | Author generated conformance CI workflow template | templates |
| 17 | `issues/17-template-readme.md` | Author generated README template with badge and security notes | templates |
| 18 | `issues/18-cli-arg-parsing.md` | Implement create-mcp argument parsing and option resolution | cli |
| 19 | `issues/19-cli-input-validation.md` | Implement project-name and target-directory validation | cli |
| 20 | `issues/20-cli-wizard-prompts.md` | Implement interactive wizard prompts | cli |
| 21 | `issues/21-cli-copier-engine.md` | Implement token substitution and copy-plan executor | cli |
| 22 | `issues/22-cli-post-generation.md` | Implement post-generation actions (git init, install, next steps) | cli |
| 23 | `issues/23-cli-e2e-tests.md` | Add CLI end-to-end tests for all transport variants | cli |
| 24 | `issues/24-dogfood-e2e.md` | Add dogfood e2e: generated project passes official conformance in CI | integration |
| 25 | `issues/25-release-pipeline.md` | Add Changesets release pipeline with npm provenance | release |
| 26 | `issues/26-repo-security-hardening.md` | Add Dependabot, CodeQL, and workflow permission hardening | security |
| 27 | `issues/27-docs-final-pass.md` | Final documentation pass with verified real outputs | docs |

## 3. Dependency table

`⟵` = "is blocked by". Only direct blockers listed.

| Issue | Blocked by |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01 |
| 04 | 01 |
| 05 | 04 |
| 06 | 04, 05 |
| 07 | 04 |
| 08 | 04 |
| 09 | 06, 07, 08 |
| 10 | 09 |
| 11 | 01 |
| 12 | 11 |
| 13 | 12 |
| 14 | 12 |
| 15 | 04 (config contract), 13, 14 |
| 16 | 11, 15 |
| 17 | 13, 14, 15, 16 |
| 18 | 01 |
| 19 | 18 |
| 20 | 18, 19 |
| 21 | 11, 12, 18, 19 |
| 22 | 15, 18, 21 |
| 23 | 17, 20, 21, 22 |
| 24 | 10, 16, 23 |
| 25 | 24 |
| 26 | 02 |
| 27 | 24, 25, 26 |

## 4. Implementation waves

| Wave | Issues | Parallelizable? | Outcome |
|---|---|---|---|
| W0 Foundation | 01, 02, 03 | 02∥03 after 01 | repo builds, CI green, community docs |
| W1 Kit | 04 → (05∥07∥08) → 06 → 09 → 10 | see deps | kit verifies any MCP server (http official + stdio smoke) |
| W2 Templates | 11 → 12 → (13∥14) → 15 → 16 → 17 | 13∥14 | complete template set, statically valid |
| W3 Wizard | 18 → 19 → (20∥21∥22) → 23 | 20∥21∥22 (21 also needs W2's 12; 22 also needs W2's 15) | working `create-mcp` CLI, e2e-tested |
| W4 Ship | 24 → 25, 26∥, 27 | 26 anytime after 02 | dogfood gate green, releasable, hardened |

W1 and W2/W3 can proceed in parallel by different agents after W0; issue 15 is the
only cross-stream contract point before W4 (it consumes the kit's config schema from
issue 04 — by file contract, not by code import).

## 5. Coverage table (DESIGN.md → issues)

| DESIGN.md section | Covered by |
|---|---|
| §2 architecture, §2.1 dependency policy | 01, 06, 26 |
| §3.1–3.2 CLI modules/contract | 18 |
| §3.3 wizard | 20 |
| §3.4 input validation (BND-1) | 19 |
| §3.5 post-gen (BND-3) | 22 |
| §3.6 next steps | 22 |
| §4.1 template layout | 12, 13, 14 |
| §4.2 manifest (BND-2) | 11 |
| §4.3 tokens | 21 (engine), 12/16/17 (usage) |
| §4.4 copier (BND-2) | 21 |
| §5.1–5.2 generated base | 12 |
| §5.3 stdio rules | 13 |
| §5.4 HTTP security (BND-6) | 14 |
| §5.5 conformance wiring | 15 |
| §6.1–6.2 kit modules/CLI | 09 |
| §6.3 badge | 16, 17 |
| §6.4 adapter (BND-5) | 06 |
| §6.5 smoke catalog | 07 |
| §7.1–7.2 CLI data contracts | 18, 11 |
| §7.3 kit config schema | 04 |
| §7.4 report model | 04, 08 |
| §8 generated workflow (BND-8) | 16 |
| §9 security model | 19 (BND-1), 11/21 (BND-2), 05/09/22 (BND-3), 08/09 (BND-4), 06 (BND-5), 14 (BND-6), 25 (BND-7), 02/16/26 (BND-8) |
| §10 testing strategy | 02, 10, 23, 24 + per-issue Validation sections |
| §11 release | 25 |
| §12 scope ledger | this file §7, §8 |

Every DESIGN section maps to ≥1 issue; issues 05 and 10 additionally cover §6.1
lifecycle and integration proof.

## 6. Validation strategy (whole product)

1. **Per-issue**: every issue carries executable acceptance criteria (unit tests or
   scripted checks) — nothing is "done by prose". Issues that produce templates are
   validated by compiling/testing the *generated* output, not the template text.
2. **Continuous**: repo CI (issue 02) runs lint/typecheck/unit on Node 22+24 for every
   PR; integration (10), CLI e2e (23), and dogfood (24) jobs join as they land.
3. **Release gate**: the dogfood e2e (24) is the product guarantee — generate `both`
   project → install packed tarballs → build → **official conformance suite green**.
   The release pipeline (25) refuses to publish if CI is not green on the tag.
4. **Security validation**: each BND row in DESIGN §9 has a named owning issue whose
   acceptance criteria include its adversarial cases (traversal names, ANSI injection,
   non-loopback URL refusal, etc.). Issue 26 adds automated dependency/action/code
   scanning on top.
5. **Manual pre-release checklist** (in issue 25): real-TTY wizard pass, fresh-machine
   `npm create` after first publish, README accuracy (issue 27 locks it).

## 7. Deferred to v1.1 / v2 (not in v1)

| Item | Earliest | Note |
|---|---|---|
| MCP 2026-07-28 + SDK v2 migration (templates, kit `specVersion`, suite `--spec-version`) | v1.1, after SDK v2 GA | ADR-002; the planned fast-follow |
| Python template track | v2 | second language doubles template+conformance matrix |
| Deploy recipes (Dockerfile, Workers, serverless) | v2 | zueai/create-mcp territory; needs opinions |
| OAuth/authorization template for HTTP | v2 | spec area still moving (2026-07-28 hardening) |
| Rich shields.io endpoint badge (scenario counts) | v2 | workflow badge suffices for v1 |
| yarn / bun package managers; Windows CI lane | v2 | detection design already PM-agnostic |
| Capability toggles in wizard (choose tools/resources/prompts) | v2 | v1 always generates all three |
| `create-mcp upgrade` for existing projects | v2 | requires template versioning story |
| MCP registry (server.json) publication helper | v2 | registry ecosystem still settling |

## 8. Known unknowns (may create issues during implementation)

| # | Unknown | Trigger to act | Likely blast radius |
|---|---|---|---|
| U1 | npm scope `@saber5656` account/Trusted-Publishing setup is a manual user step | before issue 25 can complete | release pipeline config values |
| U2 | Official suite 0.x drift (0.2.0-alpha exists) — flags/results layout may change | adapter contract tests fail on dep bump | issue 06 rework; possibly 15/16 |
| U3 | Template server may need an initial expected-failures baseline against `active` scenarios | first real run in issue 24 | baseline file + README wording |
| U4 | Exact SDK 1.29 `StreamableHTTPServerTransport` DNS-rebinding/Origin option surface | implementing issue 14 | option names in http.ts + its tests |
| U5 | `npm create @scope/name` arg-forwarding quirks across npm 10/11 | release QA in issue 25 | docs wording; possibly 18 flag handling |
| U6 | Biome 2.x rule set friction with generated code style | issues 12–14 | biome.json tuning only |

Discovery protocol: implementer records the finding in the issue thread, updates the
affected `docs/issues/*.md` (docs are canonical), and opens a follow-up issue if scope
grows beyond the owning issue.
