# Research: MCP Ecosystem Survey (2026-07-10)

Status: Complete. This survey materially shaped DESIGN.md and the v1 issue plan.
All npm facts below were verified against the npm registry on 2026-07-10 via `npm view`.
All spec facts were verified against modelcontextprotocol.io and the MCP blog on 2026-07-10.

## 1. MCP specification state

| Item | Fact | Evidence |
|---|---|---|
| Latest stable protocol revision | **2025-11-25** | https://modelcontextprotocol.io/specification/latest (schema reference points at `schema/2025-11-25/schema.ts`) |
| Next revision | **2026-07-28**, currently a Release Candidate; final publication planned 2026-07-28 (18 days after this survey) | https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/ |
| Nature of 2026-07-28 changes | Stateless protocol core (initialize handshake removed; per-request `_meta` carries protocol version / identity / capabilities), Extensions framework (reverse-DNS IDs), Tasks moved out of core into an extension, authorization hardening, formal deprecation policy | Same blog post; https://modelcontextprotocol.io/specification/draft/changelog |
| Key 2025-11-25 changes vs 2025-06-18 | OIDC discovery for auth; icons metadata; incremental scope consent; ElicitResult/EnumSchema rework; URL-mode elicitation; sampling tool-calls; experimental tasks; **HTTP 403 required for invalid `Origin`** (Streamable HTTP); tool input-validation errors must be Tool Execution Errors, not Protocol Errors (SEP-1303); JSON Schema 2020-12 as default dialect; stdio servers may use stderr for all logging | https://modelcontextprotocol.io/specification/2025-11-25/changelog |

**Design consequence:** v1 targets **2025-11-25** (stable spec + stable SDK + active
conformance scenarios). The 2026-07-28 migration is a planned v1.1 wave, and the
conformance orchestration must carry an explicit `specVersion` so the migration is a
data change plus template revision, not a redesign. See ADR-002.

## 2. Official conformance suite (decisive finding)

`@modelcontextprotocol/conformance` exists, is active, and is becoming normative
(Standards-Track SEPs cannot reach Final without a conformance scenario; the official
SDK tiering system is scored against this suite).

| Item | Fact |
|---|---|
| npm package | `@modelcontextprotocol/conformance`, latest **0.1.16** (modified 2026-07-01), `alpha` dist-tag 0.2.0-alpha.9 |
| Server testing CLI | `conformance server --url <url> [--scenario <scenario>]` — connects as an MCP client over **Streamable HTTP only**; **no stdio mode for servers** |
| Suites | `active` (default), `all`, `draft`, `pending`; scenario list via `conformance list --server` |
| Spec versions | Scenarios filtered by `--spec-version`; supports 2025-11-25 (active) and 2026-07-28 (draft) |
| Output | `results/server-<scenario>-<timestamp>/checks.json`; exit code 0 = pass or expected-failure, 1 = regression or stale baseline |
| Expected failures | YAML baseline via `--expected-failures <path>`; unexpected passes also fail (forces baseline hygiene) |
| GitHub Action | Composite action `modelcontextprotocol/conformance@v0.1.11` with `mode: server`, `url: ...` |
| Internal deps | Built on `@modelcontextprotocol/sdk` ^1.27.1, zod ^4, commander 14 |

Sources: https://github.com/modelcontextprotocol/conformance, npm registry.

**Design consequences:**
- Do not build a competing conformance runner. Wrap the official suite (ADR-001).
- The wrapper must pin an exact suite version as a dependency and execute its
  installed bin. Never `npx <latest>` at run time (supply-chain and reproducibility).
- stdio-only servers cannot be covered by the official suite; the kit provides its own
  stdio smoke checks (clearly labeled as smoke checks, not "official conformance").
- Upstream is 0.x: an adapter layer must isolate CLI-argument and results-format drift.

## 3. TypeScript SDK state

| Item | Fact |
|---|---|
| npm `latest` | `@modelcontextprotocol/sdk` **1.29.0** (modified 2026-06-04); only dist-tag is `latest` — no published beta |
| SDK v2 | Announced in repo README as beta implementing 2026-07-28; **not published to npm** as of 2026-07-10 |
| 1.29.0 deps | `zod: ^3.25 || ^4.0`, `express: ^5.2.1`, `hono ^4.11.4`, ajv, jose, etc. |
| 1.29.0 engines | `node >=18` |
| Server API (1.x) | `McpServer` + `registerTool` / `registerResource` / `registerPrompt`; `StdioServerTransport`; `StreamableHTTPServerTransport` (session management, DNS-rebinding/Origin protections) |

**Design consequences:** templates depend on SDK `^1.29.0` and **zod v4**
(supported by SDK 1.29, used by the conformance suite, and the path SDK v2 is taking via
Standard Schema — minimizes v1.1 migration churn).

## 4. Node.js baseline

As of 2026-07: Node 24 = Active LTS (support to 2028-04); Node 22 = Maintenance LTS
(to 2028-04); Node 20 Maintenance ends 2027-04. Source: https://endoflife.date/nodejs

**Design consequence:** `engines.node: ">=22"` for both published packages and generated
projects. CI tests on 22 and 24.

## 5. Competing / adjacent tools

| Tool | State (2026-07-10) | Relation to this product |
|---|---|---|
| `create-mcp` (npm, zueai) | v0.4.17, **stale since 2025-02-25**; Cloudflare-Workers-specific scaffolder | Occupies the unscoped npm name. Not conformance-oriented. We publish scoped (ADR-003) |
| `@modelcontextprotocol/create-typescript-server` | **404 on npm** — no official scaffolder is published | The scaffolding niche is open |
| `@modelcontextprotocol/inspector` | 0.22.0, active (2026-07-03) | Complementary interactive debugger; generated README links to it; not a dependency |
| `mcp-validator` (Janix-ai) | Community validator, targets 2025-06-18 era | Superseded by the official suite for our purposes |
| `@modelcontextprotocol/conformance` | See §2 | The engine we orchestrate |

**Positioning:** no maintained tool combines wizard scaffolding + official conformance
wiring + CI badge. That combination is this product's identity.

## 6. Version pin sheet (verified 2026-07-10)

Implementation issues must use these as the baseline (caret ranges unless stated):

| Package | Version |
|---|---|
| `@modelcontextprotocol/sdk` | ^1.29.0 |
| `@modelcontextprotocol/conformance` | 0.1.16 (**exact pin** in kit) |
| `zod` | ^4.0.0 (templates, kit config) |
| `express` | ^5.2.1 (HTTP template; matches SDK's own dependency) |
| `@clack/prompts` | ^1.7.0 |
| `commander` | ^15.0.0 |
| `validate-npm-package-name` | ^8.0.0 |
| `tsup` | ^8.5.1 |
| `vitest` | ^4.1.10 |
| `@biomejs/biome` | ^2.5.3 |
| `@changesets/cli` | ^2.31.0 |

## 7. Open items carried into "known unknowns"

- npm account/scope for publication: assumed `@saber5656` (verified free on npm:
  `@saber5656/create-mcp` → 404). Creating the npm account/scope and tokens is a
  **manual user step** per operating rules.
- Official suite 0.x drift: CLI flags / results layout may change between 0.1.x and
  0.2.x (alpha exists). Adapter issue owns this risk.
- Exact set of `active` scenarios at implementation time may require an initial
  expected-failures baseline for template servers; implementer records findings.
