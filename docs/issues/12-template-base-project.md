# Title

Author base template (server, capabilities, unit tests, configs)

## Summary

Author the always-copied `templates/ts/base/` file set: the three package variants
(`package.stdio.json.tmpl` / `package.http.json.tmpl` / `package.both.json.tmpl`),
tsconfig/biome/vitest configs, `src/server.ts` with
the three example capabilities (echo tool, project-info resource, greet prompt), the
capability modules, `.env.example`, `.gitignore.tmpl`, and InMemoryTransport unit
tests — everything DESIGN.md §5.1–5.2 promises except entrypoints, conformance
wiring, CI, and README (issues 13–17).

## Context

This file set defines what "a generated project" is (generated-project contract,
DESIGN §5). Templates are plain valid TypeScript files (ADR-004) — they must compile
as-is under the repo's typecheck harness so template rot is impossible.

## Scope

`templates/ts/base/**` + manifest entries + a repo-level "template compiles" test
harness (tsconfig that typechecks template sources with the SDK installed as a dev
dep of the create-mcp package for test purposes). Placeholders from issue 01 removed.

## Detailed Requirements

1. Three package variant files per DESIGN §4.1 — `base/package.stdio.json.tmpl`,
   `base/package.http.json.tmpl`, `base/package.both.json.tmpl` — each a complete
   valid JSON document targeting `package.json` with disjoint `when.transport`
   (ADR-004: no in-file conditionals). Content per DESIGN §5.1 including tokens
   (`__MCP_TMPL_PACKAGE_NAME__`, `__MCP_TMPL_KIT_DEP_VERSION__`), `"private": true`,
   engines node >=22 — **except the `conformance` script, which issue 15 adds**
   (this issue ships the variants without it; DESIGN §5.1 shows the final
   post-15 state). Common fields must be kept in sync by a unit test that parses
   all three and asserts the invariant fields (name/version/engines/type/private,
   test+lint scripts) are identical.
2. `src/server.ts`: `export function createServer(): McpServer` per DESIGN §5.2 —
   registers `echo` tool (zod v4 input `{ message: z.string().min(1).max(1000) }`,
   returns text content; `message === "trigger-error"` → `isError: true` tool result
   demonstrating SEP-1303), `project-info` static resource (`info://project`,
   text/plain), `greet` prompt (required arg `name`). Each capability implemented in
   its own module (`src/tools/echo.ts`, `src/resources/project-info.ts`,
   `src/prompts/greet.ts`) exporting a `register(server)` function; `server.ts` only
   composes. JSDoc on `createServer` states the no-transport/no-I/O rule.
3. `test/server.test.ts`: SDK `Client` ↔ `InMemoryTransport.createLinkedPair()`;
   asserts: initialize succeeds; tools/list contains `echo` with object inputSchema;
   echo happy path; `trigger-error` returns `isError: true` and NOT a protocol error;
   resource read round-trip; prompt get round-trip. These tests double as executable
   documentation for users.
4. Configs: `tsconfig.json` (strict, `module`/`moduleResolution` NodeNext, outDir
   dist, rootDir src, node types); `biome.json` minimal (recommended rules, 2-space);
   `vitest.config.ts` plain; `dot-gitignore.tmpl` → target `.gitignore`
   (node_modules, dist, conformance-results, `.env`); `dot-env.example` → target
   `.env.example` with `PORT=3000`, `HOST=127.0.0.1` and a comment "never commit
   real secrets; .env is gitignored". Dotfile sources use the `dot-` prefix per
   DESIGN §4.1 (npm strips `.gitignore` from tarballs; no template source may begin
   with a dot).
5. Manifest entries added for every file (coverage test from issue 11 enforces).
6. Template-compile harness: a vitest test copies base+stdio+http template sources
   into a temp dir (raw, tokens untouched but tokens only exist in .tmpl files —
   assert that: **no `.ts` template file contains `__MCP_TMPL_`**), installs nothing
   (uses the create-mcp package's devDependencies which must include
   `@modelcontextprotocol/sdk ^1.29.0`, `zod ^4`, `express ^5.2.1`,
   `@types/express`, `@types/node`), and runs `tsc --noEmit` over them with a
   generated tsconfig. Until issues 13/14 land their files, the harness covers what
   exists (keep it directory-driven).

## Acceptance Criteria

- [ ] Manifest coverage test passes; three package variant tmpls parse as JSON and
      pass the invariant-fields test.
- [ ] Template-compile harness: `tsc --noEmit` clean over all template TS sources.
- [ ] `test/server.test.ts` (run inside the harness with vitest against the template
      sources) passes — echo/SEP-1303/resource/prompt assertions as specified.
- [ ] No token strings inside any `.ts` file; tokens appear only in `.tmpl` files.
- [ ] `grep -r "TODO\|FIXME" templates/` is empty.

## Validation

CI green; paste the harness `tsc` invocation and the vitest summary in the PR.

## Dependencies

11.

## Non-goals

Entrypoints (13, 14), conformance config/scripts (15), workflow (16), README (17),
capability toggles (v2 non-goal — all three capabilities always ship).

## Design References

DESIGN.md §4.1, §4.3, §5.1, §5.2; ADR-004; research §6 (pins).
