# Title

Author base template (server, capabilities, unit tests, configs)

## Summary

Author the always-copied `templates/ts/base/` file set: `package.json.tmpl` with the
per-variant dependency matrix, tsconfig/biome/vitest configs, `src/server.ts` with
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

1. `package.json.tmpl` exactly per DESIGN §5.1 including tokens
   (`__MCP_TMPL_PACKAGE_NAME__`, `__MCP_TMPL_KIT_DEP_VERSION__`), `"private": true`,
   engines node >=22 — with these variant rules expressed via THREE tmpl files if
   needed (`package.stdio.json.tmpl` etc. with disjoint `when`) OR one file iff the
   diff is token-expressible. Decide with ADR-004's rule: no in-file conditionals —
   given scripts/deps differ per variant, ship **three variant files** with disjoint
   `when.transport`, each a complete valid JSON document. Common fields must be kept
   in sync by a unit test that parses all three and asserts the invariant fields
   (name/version/engines/type/private, test+lint scripts) are identical.
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
   `vitest.config.ts` plain; `.gitignore.tmpl` (node_modules, dist,
   conformance-results, `.env`); `.env.example` with `PORT=3000`, `HOST=127.0.0.1`
   and a comment "never commit real secrets; .env is gitignored".
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
