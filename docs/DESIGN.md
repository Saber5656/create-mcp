# create-mcp — v1 Design

Status: Approved for issue planning (2026-07-10).
Canonical sources: this file, `docs/decisions/ADR-*.md`, `docs/research/*.md`.
Issue plan: `docs/ISSUE_PLAN.md`. Issue drafts: `docs/issues/*.md`.

## 1. Product definition

**create-mcp** is a wizard that generates Model Context Protocol (MCP) server projects
that are *verifiably conformant*: every generated project ships wired to the official
MCP conformance suite, runs it in CI, and carries a ready-to-activate conformance badge
in its README (activation = replacing the badge's owner/repo placeholder after the
first push to GitHub — a documented one-line step, §6.3).

One sentence pitch: *"`npm create @saber5656/mcp` gives you an MCP server that proves
it speaks MCP."*

### 1.1 Confirmed product decisions

| # | Decision | Detail | Reference |
|---|---|---|---|
| D1 | Generated language | TypeScript only in v1 | user decision 2026-07-10 |
| D2 | Conformance engine | Wrap the official `@modelcontextprotocol/conformance` suite; own stdio smoke checks as complement | ADR-001 |
| D3 | Badge | GitHub Actions workflow badge produced by a generated conformance workflow; no hosted badge service | ADR-006 |
| D4 | Distribution | Scoped npm packages under `@saber5656/*` | ADR-003 |
| D5 | Transports in v1 | stdio and Streamable HTTP (wizard-selectable: `stdio`, `http`, `both`) | user decision 2026-07-10 |
| D6 | License | MIT (repo and published packages) | user decision 2026-07-10 |
| D7 | Spec target | MCP revision **2025-11-25**, SDK **1.x**; 2026-07-28/SDK v2 deferred to v1.1 | ADR-002 |

### 1.2 Users and primary flows

1. **Generate**: developer runs `npm create @saber5656/mcp` (or `pnpm create @saber5656/mcp`,
   `npx @saber5656/create-mcp`), answers ≤5 prompts, gets a working project.
2. **Develop**: developer edits `src/server.ts`, adds tools/resources/prompts; unit
   tests run with `npm test`.
3. **Verify**: `npm run conformance` builds and starts the server, runs the official
   conformance suite (HTTP) and stdio smoke checks, prints a report.
4. **Prove**: pushing to GitHub runs the generated conformance workflow; the README
   badge reflects the result.

Secondary user: an implementation/CI agent that must run everything non-interactively
(all wizard answers available as flags; `--yes` for defaults).

## 2. System architecture

Monorepo `Saber5656/create-mcp` (pnpm workspaces) publishing two packages:

```
create-mcp (repo)
├── packages/
│   ├── create-mcp/            → npm: @saber5656/create-mcp   (bin: create-mcp)
│   │   └── templates/         → bundled, offline template files (shipped in the npm tarball)
│   └── conformance-kit/       → npm: @saber5656/mcp-conformance-kit (bin: mcp-conformance-kit)
├── docs/                      → canonical design & issue docs (this file)
└── .github/workflows/         → repo CI (lint/test/e2e/release)
```

Dependency direction at run time:

```
generated project ──devDependency──▶ @saber5656/mcp-conformance-kit ──exact dep──▶ @modelcontextprotocol/conformance
        │                                                            (official suite, engine)
        └──dependency──▶ @modelcontextprotocol/sdk ^1.29.0
@saber5656/create-mcp ──(generates)──▶ generated project        (no runtime coupling)
```

`@saber5656/create-mcp` has no runtime dependency on the kit; it only writes the kit's
name and version into generated `package.json` files. To keep that version current it
deliberately carries the kit as a **devDependency used purely as the version source**
for the `__MCP_TMPL_KIT_DEP_VERSION__` token (§4.3) — nothing from the kit is imported.
The kit does not depend on create-mcp.

### 2.1 External dependency policy

Baseline versions are pinned in `docs/research/2026-07-mcp-ecosystem-survey.md` §6.
Rules:

- `@modelcontextprotocol/conformance` is an **exact** dependency of the kit (no caret).
  Upgrades are deliberate PRs (Dependabot proposes; the adapter's contract tests gate).
- Everything else uses caret ranges; lockfile committed; CI runs from the lockfile.
- The kit invokes the official suite by resolving its installed bin
  (`require.resolve("@modelcontextprotocol/conformance/package.json")` → `bin` field),
  **never** via `npx` at run time.

## 3. Package: `@saber5656/create-mcp` (wizard CLI)

### 3.1 Module layout

```
packages/create-mcp/
├── package.json          # bin: {"create-mcp": "dist/index.js"}, files: ["dist", "templates"]
├── templates/            # see §4
└── src/
    ├── index.ts          # #!/usr/bin/env node; calls run(process.argv)
    ├── cli.ts            # commander definition; builds RawOptions
    ├── options.ts        # RawOptions → ResolvedOptions (validation + defaults + TTY logic)
    ├── validate.ts       # project name & target dir validation (§3.4)
    ├── prompts.ts        # @clack/prompts wizard (§3.3)
    ├── engine/
    │   ├── manifest.ts   # manifest schema (zod) + loader (§4.2)
    │   ├── plan.ts       # manifest + options → CopyPlan (pure)
    │   ├── tokens.ts     # token substitution (§4.3)
    │   └── copier.ts     # CopyPlan executor (fs writes, §4.4)
    └── post/
        ├── git.ts        # git init (§3.5)
        ├── install.ts    # dependency install (§3.5)
        └── next-steps.ts # final summary output (§3.6)
```

### 3.2 CLI contract

```
create-mcp [targetDir] [options]

Options:
  --transport <stdio|http|both>   transport variant (default: both)
  --pm <npm|pnpm>                 package manager for install + lockfile (default: auto-detect)
  --git / --no-git                init a git repository (default: --git)
  --install / --no-install        install dependencies after generation (default: --install)
  --yes                           accept defaults for all unanswered questions
  --name <string>                 package name when it differs from targetDir basename
  --version, --help
```

Behavior matrix:

| Condition | Behavior |
|---|---|
| TTY + missing answers | interactive wizard fills the gaps |
| non-TTY + all answers derivable (flags and/or `--yes`) | run non-interactively |
| non-TTY + missing answers, no `--yes` | exit code 2 with a message listing missing flags |
| target dir exists and is not empty | exit code 3, no writes (no --force in v1) |
| user cancels wizard (Ctrl-C / clack cancel) | exit code 130, no partial writes |

Default semantics: the defaults listed above are applied only by `--yes` or as the
pre-selected answer in an interactive prompt — a field never silently defaults in
non-interactive mode without `--yes` (interactivity stays deliberate in CI).

Exit codes: `0` success, `2` invalid input/flags, `3` unsafe target directory,
`4` generation failure (cleanup attempted), `5` post-generation step failed
(project generated; message explains how to finish manually).

Package-manager auto-detect: parse `npm_config_user_agent` (`pnpm/…` → pnpm, else npm).

### 3.3 Wizard flow (state machine)

States, in order; each state is skipped when its value is already resolved from
argv/flags/`--yes`:

```
S0 intro → S1 targetDir → S2 packageName (default: sanitized basename of targetDir)
→ S3 transport → S4 packageManager → S5 git → S6 install
→ S7 confirm-summary (prints resolved plan; Enter to proceed)
→ RUN (plan → copy → post steps) → DONE
Any state: cancel → CANCELLED (exit 130)
```

`--yes` resolves S2–S6 to defaults and skips S7.

### 3.4 Input validation (security boundary BND-1)

`validate.ts` must enforce, with unit-tested adversarial cases:

- Package name: `validate-npm-package-name` must return `validForNewPackages: true`.
- targetDir: after `path.resolve`, the final path must not be `/`, must not be inside
  the CLI's own installation directory, and its basename must not start with `-`.
  Creation is `fs.mkdir(target, { recursive: true })` followed by an emptiness check
  (dir with only `.git` counts as non-empty → exit 3).
- No option value is ever passed through a shell (see §3.5).

### 3.5 Post-generation actions (BND-3)

- git: if `--git` and `git` binary found (`spawnSync("git", ["--version"])` succeeds):
  `spawn("git", ["init"], { cwd: target, shell: false })`. No initial commit in v1.
  Failure downgrades to a warning (exit stays 0) — git is a convenience here.
- install: `spawn(pm, ["install"], { cwd: target, shell: false, stdio: "inherit" })`
  where `pm ∈ {"npm","pnpm"}` (enum-validated — never user-typed free text).
  Failure → exit 5 with remediation text (`cd <dir> && <pm> install`).

### 3.6 Next-steps output

Transport-aware plain text (no color when `!process.stdout.isTTY` or `NO_COLOR`):
`cd` line, dev commands, `npm run conformance`, how to enable the badge (§6.3), and a
link to the generated README's security notes.

## 4. Template system

### 4.1 Variants and layout on disk

One base template composed with transport-conditional files — no divergent full copies.

```
packages/create-mcp/templates/ts/
├── template.json                  # manifest (§4.2)
├── base/                          # default set (package/README files select by transport)
│   ├── package.stdio.json.tmpl    # → package.json   when transport=stdio
│   ├── package.http.json.tmpl     # → package.json   when transport=http
│   ├── package.both.json.tmpl     # → package.json   when transport=both
│   ├── README.stdio.md.tmpl       # → README.md      when transport=stdio
│   ├── README.http.md.tmpl        # → README.md      when transport=http
│   ├── README.both.md.tmpl        # → README.md      when transport=both
│   ├── tsconfig.json
│   ├── biome.json
│   ├── vitest.config.ts
│   ├── dot-gitignore.tmpl         # → .gitignore (dot- prefix: npm always strips .gitignore from tarballs)
│   ├── dot-env.example            # → .env.example (dot- prefix for uniformity; no dot-named sources allowed)
│   ├── src/server.ts              # createServer(): McpServer with echo tool, info resource, greet prompt
│   ├── src/tools/echo.ts
│   ├── src/resources/project-info.ts
│   ├── src/prompts/greet.ts
│   └── test/server.test.ts        # SDK Client ↔ InMemoryTransport unit tests
├── stdio/
│   └── src/stdio.ts               # StdioServerTransport entry
├── http/
│   ├── src/http.ts                # thin executable entry: env parsing, start, signal handling
│   └── src/http-app.ts            # exported factory (app + transport) — the test seam (§5.4)
├── conformance/
│   ├── config.stdio.json.tmpl     # → conformance.config.json  when transport=stdio (stdio section only)
│   ├── config.http.json.tmpl      # → conformance.config.json  when transport=http  (http section only)
│   ├── config.both.json.tmpl      # → conformance.config.json  when transport=both  (both sections)
│   └── expected-failures.yaml     # → conformance-expected-failures.yaml (all variants; comments only)
└── ci/
    └── github/workflows/
        ├── conformance.npm.yml.tmpl    # when packageManager=npm  → .github/workflows/conformance.yml
        └── conformance.pnpm.yml.tmpl   # when packageManager=pnpm → same target
```

Template sources never begin with a dot (npm packaging safety); the manifest `target`
carries the real dotfile name, so the copier needs no rename logic.

Generated file sets:

| File | stdio | http | both |
|---|---|---|---|
| base/* (package.json / README.md resolve per variant) | ✓ | ✓ | ✓ |
| src/stdio.ts | ✓ | – | ✓ |
| src/http.ts, src/http-app.ts | – | ✓ | ✓ |
| conformance.config.json | ✓ (stdio section) | ✓ (http section) | ✓ (both sections) |
| conformance-expected-failures.yaml | ✓ | ✓ | ✓ |
| .github/workflows/conformance.yml | ✓ | ✓ | ✓ (file selected per packageManager) |

### 4.2 Manifest schema (`template.json`)

Zod-validated at CLI start (a broken bundled manifest is a defect → exit 4):

```jsonc
{
  "version": 1,
  "files": [
    { "source": "base/package.json.tmpl", "target": "package.json", "substitute": true },
    { "source": "http/src/http.ts", "target": "src/http.ts", "when": { "transport": ["http", "both"] } },
    { "source": "ci/github/workflows/conformance.pnpm.yml.tmpl",
      "target": ".github/workflows/conformance.yml",
      "when": { "packageManager": ["pnpm"] } }
  ]
}
```

Rules enforced by `manifest.ts` + `plan.ts`:
- `source` and `target` must be relative, must not contain `..` or be absolute, and must
  resolve inside the template dir / target dir respectively (checked after resolution).
- `when` supports exactly two condition keys in v1 — `transport`
  (`("stdio"|"http"|"both")[]`) and `packageManager` (`("npm"|"pnpm")[]`). Multiple
  keys on one entry AND together. The schema stays closed (`strict()` so unknown keys
  fail loudly).
- Two entries may share a `target` only if their `when` conditions cannot co-occur
  for any single `(transport, packageManager)` combination; the manifest loader
  rejects co-occurring duplicates at parse time.
- Every file in `templates/ts/**` except `template.json` must be referenced by exactly
  one manifest entry; a repo unit test enforces this (no dead or unshipped files).

### 4.3 Token substitution

Only files with `"substitute": true` are processed. Substitution is literal
string replacement of the closed token set — no expression evaluation, no template
engine (ADR-004):

| Token (literal text in file) | Replacement |
|---|---|
| `__MCP_TMPL_PACKAGE_NAME__` | validated package name |
| `__MCP_TMPL_KIT_DEP_VERSION__` | the version spec of the `@saber5656/mcp-conformance-kit` devDependency declared in create-mcp's own package.json, captured at build time into a generated `src/kit-version.ts` constant (Changesets bumps that devDependency on kit releases, keeping generated projects current) |
| `__MCP_TMPL_TRANSPORT__` | `stdio` \| `http` \| `both` |
| `__MCP_TMPL_BADGE_PATH__` | `<OWNER>/<REPO>` placeholder text `your-github-user/your-repo` (user fills; README explains) |
| `__MCP_TMPL_NODE_MIN__` | `22` |

`tokens.ts` must fail the run (exit 4) if, after substitution, any `__MCP_TMPL_` string
remains in a substituted file, and a repo test asserts the manifest marks every file
containing tokens as `substitute: true`.

### 4.4 Copier guarantees (BND-2)

- Refuses to write outside the resolved target dir (defense-in-depth re-check per file).
- Preserves no executable bits (all files 0644; there are no scripts in templates).
- On any failure mid-copy: remove only files/dirs it created this run (tracked list),
  then exit 4. Never delete a pre-existing path.

## 5. Generated project (contract)

### 5.1 package.json (both-variant shown)

```jsonc
{
  "name": "__MCP_TMPL_PACKAGE_NAME__",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "dev:stdio": "node --watch dist/stdio.js",          // stdio, both
    "dev:http": "node --watch dist/http.js",            // http, both
    "start:http": "node dist/http.js",                  // http, both
    "test": "vitest run",
    "conformance": "mcp-conformance-kit run",           // reads conformance.config.json
    "lint": "biome check ."
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.29.0",
    "zod": "^4.0.0",
    "express": "^5.2.1"                                  // http, both only
  },
  "devDependencies": {
    "@saber5656/mcp-conformance-kit": "__MCP_TMPL_KIT_DEP_VERSION__",
    "typescript": "^5.9.0", "vitest": "^4.1.10", "@biomejs/biome": "^2.5.3",
    "@types/express": "^5.0.0", "@types/node": "^24.0.0"
  }
}
```

(The stdio-only variant omits express/@types/express, `dev:http`, `start:http`; its
`conformance` script runs smoke checks only. Exact per-variant matrices live in the
template issues.)

### 5.2 Server definition

`src/server.ts` exports `createServer(): McpServer` — pure construction, no transport,
no I/O — so unit tests, both entrypoints, and future entrypoints share one definition.
Example capabilities (all three, always):

- tool `echo` — input `{ message: z.string().min(1).max(1000) }`; returns the message;
  demonstrates SEP-1303 by returning a tool-execution error (`isError: true`) for a
  reserved input (`message === "trigger-error"`), not a protocol error.
- resource `project-info` (`info://project`) — static text resource.
- prompt `greet` — one required string argument `name`.

### 5.3 stdio entrypoint rules

`src/stdio.ts`: connect `StdioServerTransport`; **all logging to stderr, never stdout**
(2025-11-25 clarification); exit 0 on transport close; handle SIGINT/SIGTERM with
`server.close()`.

### 5.4 HTTP entrypoint rules (security boundary BND-6)

The HTTP variant ships two files: `src/http-app.ts` exports the factory
(`createHttpApp(options)` returning the configured express app plus transport —
the seam that behavioral tests and future entrypoints import), and `src/http.ts` is
the thin executable entry (env parsing → factory → listen → signal handling).
Combined behavior (express 5 + `StreamableHTTPServerTransport`):

| Requirement | Value |
|---|---|
| Bind address | `127.0.0.1` default; `HOST` env overrides, README warns about exposure |
| Port | `PORT` env, default `3000`; endpoint path `/mcp` |
| Origin / DNS-rebinding | SDK protections enabled (`enableDnsRebindingProtection: true`, `allowedHosts: ["127.0.0.1:<port>", "localhost:<port>"]`); invalid Origin → HTTP 403 (spec MUST) |
| Sessions | SDK session management on (`sessionIdGenerator: randomUUID`) |
| Auth | none in v1 (non-goal); README security section documents this and links MCP authorization spec |
| Error hygiene | JSON-RPC errors carry messages, never stack traces; unexpected errors log to stderr, return generic message |
| Shutdown | SIGINT/SIGTERM → stop accepting, close transports, exit within 5s |
| Body limits | JSON body size limited to 1 MB |

### 5.5 Conformance wiring

`conformance.config.json` (all variants — the stdio variant carries only the
`stdio` section, the http variant only the `http` section; schema in §7.3).
The both-variant content:

```jsonc
{
  "$schema": "node_modules/@saber5656/mcp-conformance-kit/schema/conformance-config.schema.json",
  "specVersion": "2025-11-25",
  "http": {
    "url": "http://127.0.0.1:3000/mcp",
    "start": ["node", "dist/http.js"],
    "readyTimeoutMs": 30000,
    "suite": "active"
  },
  "stdio": { "command": ["node", "dist/stdio.js"] }     // stdio present in stdio/both variants
}
```

`npm run conformance` = `mcp-conformance-kit run` (builds are NOT implicit; the script
chain is `npm run build && mcp-conformance-kit run` in the generated workflow, and the
README documents the same order for local runs).

## 6. Package: `@saber5656/mcp-conformance-kit`

### 6.1 Module layout

```
packages/conformance-kit/
├── package.json            # bin: {"mcp-conformance-kit": "dist/bin.js"}; exact dep on @modelcontextprotocol/conformance 0.1.16
├── schema/conformance-config.schema.json   # generated from zod at build time
└── src/
    ├── bin.ts               # commander: run | smoke | print-config
    ├── index.ts             # programmatic API: loadConfig, runAll, runOfficial, runSmoke
    ├── config.ts            # zod schema + loader (§6.2 / §7.3)
    ├── lifecycle/
    │   ├── spawn.ts         # argv spawn, process-group, log capture
    │   ├── ready.ts         # readiness probe
    │   └── stop.ts          # SIGTERM → 5s → SIGKILL (process group)
    ├── adapter/
    │   ├── official.ts      # resolve bin, build args, execute (§6.4)
    │   └── results.ts       # parse results/**/checks.json → Report model
    ├── smoke/
    │   ├── run.ts           # SDK Client over StdioClientTransport, executes catalog
    │   └── checks.ts        # check catalog (§6.5)
    └── report/
        ├── model.ts         # Report types (§7.4)
        ├── json.ts          # writes conformance-results/report.json
        ├── terminal.ts      # TTY renderer; sanitizes server-controlled strings (§9 BND-4)
        └── github-summary.ts# appends markdown to $GITHUB_STEP_SUMMARY when set
```

### 6.2 CLI contract

```
mcp-conformance-kit run   [--config conformance.config.json] [--only http|stdio]
mcp-conformance-kit smoke [--config ...]        # alias: run --only stdio
mcp-conformance-kit print-config                # resolved config as JSON (debugging)
```

`run` executes: http section present → official suite; stdio section present → smoke
checks; merges into one Report; renders terminal + JSON + (in CI) GitHub summary.

Exit codes: `0` all passed (expected-failures respected), `1` conformance/smoke
failure, `2` config invalid, `3` server failed to start or become ready, `4` internal
kit error. `conformance-results/report.json` is written whenever at least one check
executed (always for exit 0/1, and for exit 4 when partial results exist); exit 2/3
report on stderr only — there is nothing measured to persist.

### 6.3 Badge contract

The badge is the generated workflow's status badge:
`https://github.com/<owner>/<repo>/actions/workflows/conformance.yml/badge.svg`.
Semantics: green = official suite (`active`, spec 2025-11-25) + smoke checks passed on
default branch. The kit's GitHub summary table is the human-readable proof detail.

### 6.4 Official-suite adapter (BND-5)

- Resolve engine: `createRequire(import.meta.url).resolve("@modelcontextprotocol/conformance/package.json")`;
  read its `bin.conformance`; spawn `process.execPath [binPath, "server", "--url", url, ...]`
  with `shell: false`, cwd = a per-run temp results dir.
- Args mapping from config: `suite` → `--suite` (v0.1.16 default `active`),
  `scenarios[]` → repeated `--scenario`, `specVersion` → `--spec-version`,
  `expectedFailures` → `--expected-failures` (path validated to exist).
- Results discovery: newest `results/server-*/checks.json` under the run's cwd; parse
  defensively (unknown fields ignored; missing file → exit 4 with captured
  stdout/stderr excerpt).
- Upstream exit-code semantics (0/1 incl. baseline logic) pass through into the
  Report's `official.status`.
- **Contract tests** (fixture copies of real checks.json + `--help` snapshot) exist so
  a Dependabot bump of the exact pin fails loudly when upstream drift breaks the
  adapter (this is the isolation demanded by ADR-001).

### 6.5 stdio smoke-check catalog

Client = SDK `Client` + `StdioClientTransport` (spawns `config.stdio.command` argv).
Each check: `{ id, level: "MUST" | "SHOULD", title, run(ctx) }`, independent, ordered:

| ID | Level | Assertion |
|---|---|---|
| SMOKE-INIT-01 | MUST | initialize succeeds; negotiated `protocolVersion` = `2025-11-25` (or server's declared supported version — mismatch reported) |
| SMOKE-INIT-02 | MUST | `serverInfo.name` and `.version` non-empty |
| SMOKE-STDOUT-01 | MUST | stdout carried only JSON-RPC frames during the session (no log pollution) |
| SMOKE-TOOL-01 | MUST | `tools/list` returns ≥1 tool; every tool has non-empty name and object `inputSchema` |
| SMOKE-TOOL-02 | MUST | calling the first listed tool with schema-valid sample args returns a result without protocol error |
| SMOKE-TOOL-03 | SHOULD | calling with schema-invalid args yields tool-execution error (`isError: true`) or structured validation error — not a JSON-RPC protocol error (SEP-1303) |
| SMOKE-RES-01 | MUST* | `resources/list` succeeds (*when capability declared) |
| SMOKE-RES-02 | MUST* | `resources/read` on first listed resource returns contents |
| SMOKE-PROMPT-01 | MUST* | `prompts/list` succeeds |
| SMOKE-PROMPT-02 | MUST* | `prompts/get` on first prompt with required args returns messages |
| SMOKE-SHUTDOWN-01 | SHOULD | process exits ≤5s after transport close, exit code 0 |

MUST failure → overall fail; SHOULD failure → warning (non-zero only with `--strict`,
default off in v1). Sample-args generation supports string/number/boolean/enum leaf
schemas; anything else → check reports `skip` with reason (never guesses).

## 7. Data contracts

### 7.1 ResolvedOptions (create-mcp, internal)

```ts
type ResolvedOptions = {
  targetDir: string;             // absolute
  packageName: string;           // validated
  transport: "stdio" | "http" | "both";
  packageManager: "npm" | "pnpm";
  git: boolean; install: boolean;
  interactive: boolean;          // wizard used
};
```

### 7.2 CopyPlan (create-mcp, internal)

`{ targetRoot: string; mustCreate: boolean; files: { sourceAbs, targetAbs,
substitute }[] }` — fully computed before any write (`mustCreate` comes from the
target-directory validation result, §3.4).

### 7.3 Kit config schema (public)

Zod source of truth (`config.ts`), JSON Schema exported at build to
`schema/conformance-config.schema.json`:

```ts
const HttpTarget = z.object({
  url: z.string().url().refine(isLoopbackOrExplicitOptIn),   // §9 BND-4
  start: z.array(z.string()).min(1),          // argv; [0] is the executable
  cwd: z.string().optional(),
  readyTimeoutMs: z.number().int().positive().max(300_000).default(30_000),
  suite: z.enum(["active", "all", "draft", "pending"]).default("active"),
  scenarios: z.array(z.string()).optional(),
  expectedFailures: z.string().optional(),    // path to YAML baseline
}).strict();
const StdioTarget = z.object({
  command: z.array(z.string()).min(1),
  cwd: z.string().optional(),
}).strict();
export const KitConfig = z.object({
  $schema: z.string().optional(),
  specVersion: z.literal("2025-11-25"),       // widened in v1.1
  http: HttpTarget.optional(),
  stdio: StdioTarget.optional(),
  reportDir: z.string().default("conformance-results"),
}).strict().refine(c => c.http || c.stdio);
```

`env` passthrough: child processes inherit `process.env` unchanged in v1 (the config
file deliberately has **no** env-injection field — see §9 BND-3).

### 7.4 Report model (public, written to report.json)

```ts
type CheckStatus = "pass" | "fail" | "expected_fail" | "unexpected_pass" | "skip";
type Report = {
  kitVersion: string;
  specVersion: "2025-11-25";
  startedAt: string; finishedAt: string;      // ISO-8601 UTC
  official?: {
    engine: { name: "@modelcontextprotocol/conformance"; version: string };
    suite: string; url: string;
    status: "pass" | "fail";
    checks: { scenario: string; id: string; status: CheckStatus; detail?: string }[];
  };
  smoke?: {
    command: string[];
    status: "pass" | "fail";
    checks: { id: string; level: "MUST" | "SHOULD"; status: CheckStatus; detail?: string }[];
  };
  summary: { total: number; pass: number; fail: number; expectedFail: number; skip: number };
};
```

## 8. Generated CI workflow

`.github/workflows/conformance.yml` (two complete template files selected by
`when.packageManager` — §4.2; transport variance is fully encapsulated by the
generated `conformance` script, so the workflow itself is transport-independent and
token-free):

- Triggers: `push` (default branch), `pull_request`, `workflow_dispatch`.
- Permissions: `contents: read` only. Concurrency: cancel-in-progress per ref.
- Steps: checkout → setup-node (Node 24, cache for the chosen PM) → install
  (`npm ci` / `pnpm install --frozen-lockfile`) → build script → conformance script.
  The npm and pnpm variants are two complete workflow template files selected by the
  manifest's `when.packageManager` condition (§4.2) — no in-file conditionals, and
  the pnpm variant adds the `pnpm/action-setup` step. Both render to the same target
  path `.github/workflows/conformance.yml` (the badge URL depends on that filename).
- All third-party actions pinned to **full commit SHAs** with a trailing
  `# vX.Y.Z` comment.
- The kit auto-detects `GITHUB_STEP_SUMMARY` and publishes the report table; the
  workflow also uploads `conformance-results/` as an artifact (retention 14 days).

Repo-side CI (this repo, not generated) is specified in the corresponding issues:
lint+typecheck+unit (Node 22 & 24 matrix), kit integration test against a fixture
server, and the dogfood e2e (§10.3).

## 9. Security model

Assumption: this repo becomes public OSS. Security posture is part of v1 scope, not a
hardening pass. Trust boundaries and required mitigations:

| Boundary | Threat | Mitigation (owning issue in ISSUE_PLAN coverage table) |
|---|---|---|
| BND-1 wizard/CLI input | path traversal via project name/targetDir; flag injection | §3.4 validation; argv-only spawns (`shell: false` everywhere); enum-validated pm |
| BND-2 templates → user disk | writing outside target; leftover partial state; hidden executable files | §4.2 manifest path rules; §4.4 copier re-checks, 0644 modes, cleanup-own-writes |
| BND-3 kit spawning processes | config-file-driven command execution surprise; orphaned processes; env leakage into reports | commands run with same privileges as the npm scripts the user already runs (documented); argv arrays, no shell; process-group kill; reports never echo env; no env-injection config field in v1 |
| BND-4 kit ↔ server traffic & reports | terminal escape / ANSI injection via server-controlled strings; kit pointed at third-party servers | terminal renderer strips C0/C1 controls & ANSI sequences from `detail` strings; config URL must be loopback unless `--allow-remote` is passed (explicit opt-in flag, documented as "only against servers you are authorized to test") |
| BND-5 official-suite dependency | supply-chain drift/compromise via floating versions or npx | exact version pin; run installed bin only; lockfile; Dependabot + adapter contract tests gate upgrades |
| BND-6 generated server surface | DNS rebinding; cross-origin access; secret leakage; oversized payloads; stack-trace disclosure | §5.4 table (127.0.0.1 bind, Origin→403, SDK rebinding protection, 1 MB body cap, error hygiene); `.env` gitignored + `.env.example` pattern; zod input validation on the example tool |
| BND-7 release pipeline | npm token theft; tampered artifacts | npm **Trusted Publishing (OIDC) + provenance**; no long-lived npm tokens in repo secrets; scoped packages `publishConfig.access: "public"`; `files` allowlists; Changesets version-PR release workflow (§11) with least-privilege permissions |
| BND-8 CI workflows (repo + generated) | action supply chain; permission escalation | SHA-pinned actions; top-level `permissions: contents: read`; no `pull_request_target`; Dependabot for actions ecosystem |

Secret-handling rules (repo-wide): no secrets in code, templates, tests, or fixtures;
`.env*` ignored in both repo and generated projects; publishing credentials exist only
as GitHub OIDC federation (user configures npm Trusted Publishing manually — agents
never handle tokens).

Vulnerability reporting: `SECURITY.md` with private disclosure via GitHub Security
Advisories, target first-response SLO 7 days (solo maintainer honesty).

## 10. Testing & validation strategy

| Layer | Where | What proves |
|---|---|---|
| Unit | each package, vitest | validation edge cases, manifest/plan purity, token engine, adapter arg-building, results parsing (fixtures), report rendering & sanitization, smoke-check logic against in-process fake |
| Kit integration | repo CI | kit `run` end-to-end against a minimal fixture MCP server (SDK-based, in `packages/conformance-kit/test/fixtures/`) — real official suite executes |
| CLI e2e | repo CI | `create-mcp --yes` for all 3 transport variants into temp dirs → `tsc` build passes → generated unit tests pass → no `__MCP_TMPL_` residue |
| Dogfood e2e | repo CI (the release gate) | generate `both` project → `npm pack` both packages, install tarballs → `npm run build && npm run conformance` → **official suite green** |
| Manual gate | pre-release checklist | wizard UX on real TTY; README accuracy; fresh-machine `npm create` flow after first publish |

v1 completion = all issue acceptance criteria + the dogfood e2e green in CI.

## 11. Versioning & release

- Changesets; independent semver per package; both start at `0.1.0`.
- Publish workflow: the standard Changesets **version-PR flow** — on push to `main`
  the changesets action maintains a "Version Packages" PR; merging that PR triggers
  publish (GitHub Actions, npm provenance, OIDC Trusted Publishing; the action also
  pushes the release tags it creates). No long-lived npm tokens. Manual user
  prerequisites (npm account/scope, Trusted Publisher registration) are documented
  in the release issue as human steps.
- The kit's public surface for semver purposes: CLI contract (§6.2), config schema
  (§7.3), report.json shape (§7.4). The create-mcp public surface: CLI contract (§3.2)
  and the generated-project contract (§5).

## 12. Scope ledger

### v1 non-goals (explicit)

Python/other languages; auth/OAuth in generated servers; deploy targets (Docker,
Workers, serverless); hosted badge/registry service; yarn/bun package managers;
template capability toggles (tools/resources/prompts are always all included);
`upgrade` command for previously generated projects; Windows CI runners (Windows
support is best-effort via pure-Node APIs; tested on macOS/Linux).

### v2 / deferred

2026-07-28 spec + SDK v2 migration (templates, kit `specVersion` widening, suite
`--spec-version` bump); Python templates; deploy recipes; OAuth template; richer
shields.io endpoint badge with scenario counts; bun/yarn; capability toggles;
`create-mcp upgrade`; Windows CI lane; MCP registry (server.json) publication helper.

### Known unknowns (may spawn issues during implementation)

1. npm scope: `@saber5656` assumed and verified free; user must create the npm
   account/scope and enable Trusted Publishing manually before the release issue.
2. Official suite 0.x drift (0.2.0-alpha exists): adapter contract tests are the
   tripwire; flag/results changes may add work to the adapter issue.
3. Initial `active`-suite results for the generated template may require an
   expected-failures baseline entry (template or upstream gaps) — to be recorded with
   evidence in the dogfood issue.
4. SDK 1.29 `StreamableHTTPServerTransport` option names for rebinding/origin
   protection must be verified against the installed SDK version at implementation
   time (option surface changed across 1.x minors).
5. `npm create @saber5656/mcp` argument forwarding quirks across npm 10/11 versions —
   verify during release QA (documented fallback: `npx @saber5656/create-mcp`).
