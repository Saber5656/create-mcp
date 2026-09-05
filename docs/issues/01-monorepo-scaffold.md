# Title

Scaffold pnpm monorepo with package skeletons and shared toolchain

## Summary

Create the pnpm-workspaces monorepo layout with two buildable-but-empty packages
(`@saber5656/create-mcp`, `@saber5656/mcp-conformance-kit`), shared TypeScript/Biome/
vitest/tsup configuration, MIT license, and repo hygiene files. After this issue,
`pnpm install && pnpm build && pnpm test && pnpm lint` all succeed on a clean clone.

## Context

Everything else in the plan lands inside this structure (DESIGN.md §2, ADR-005).
Package skeletons must compile and publish-dry-run cleanly so later issues only add
modules, never fight tooling.

## Scope

- Root: `package.json` (private), `pnpm-workspace.yaml` (`packages/*`),
  `tsconfig.base.json` (strict), `biome.json`, `.gitignore`, `.editorconfig`,
  `.nvmrc` (`24`), `LICENSE` (MIT, holder "Saber5656").
- `packages/create-mcp/`: `package.json`, `tsconfig.json`, `tsup.config.ts`,
  `src/index.ts` (placeholder bin printing name+version, exit 0),
  `templates/.gitkeep`, one placeholder vitest test.
- `packages/conformance-kit/`: same shape; placeholder bin `src/bin.ts`; `src/index.ts`
  exporting a `KIT_VERSION` constant; one placeholder test.

## Detailed Requirements

1. Root `package.json`: `"private": true`, `"type": "module"`,
   `"engines": {"node": ">=22"}`, `"packageManager": "pnpm@<exact>"` where `<exact>`
   is resolved at implementation time via `npm view pnpm version` (record the value
   in the PR description), scripts `build` / `test` / `lint` / `typecheck` fanning
   out via `pnpm -r`.
2. Both package `package.json` files:
   - names `@saber5656/create-mcp` and `@saber5656/mcp-conformance-kit`; version `0.1.0`;
     `"type": "module"`; `"license": "MIT"`; `"engines": {"node": ">=22"}`;
     `"publishConfig": {"access": "public"}`.
   - `bin`: `{"create-mcp": "dist/index.js"}` and
     `{"mcp-conformance-kit": "dist/bin.js"}` respectively.
   - `files`: `["dist"]` for the kit; `["dist", "templates"]` for create-mcp.
   - `repository`, `bugs`, `homepage` pointing at `github.com/Saber5656/create-mcp`.
3. Dev tooling versions from `docs/research/2026-07-mcp-ecosystem-survey.md` §6:
   typescript ^5.9, tsup ^8.5.1, vitest ^4.1.10, @biomejs/biome ^2.5.3. TypeScript
   `strict: true`, `module: "nodenext"` + `moduleResolution: "nodenext"` (binding
   choice: these are Node-native ESM CLIs and the generated templates use NodeNext
   too — one resolution model everywhere; note this rationale in a comment in
   `tsconfig.base.json`).
4. tsup: ESM only, `target: "node22"`, shebang preserved for bins (`banner` or source
   shebang), `dist/` output, d.ts generation ON for the kit (its API is public),
   OFF for create-mcp.
5. Bin files start with `#!/usr/bin/env node` and are executable after
   `pnpm build` + `npm pack` (verify file mode via pack tarball listing).
6. `.gitignore`: `node_modules/`, `dist/`, `conformance-results/`, `.env`, `*.tgz`,
   coverage output.

## Acceptance Criteria

- [ ] Fresh clone: `pnpm install`, `pnpm build`, `pnpm test`, `pnpm lint`,
      `pnpm typecheck` all exit 0 on Node 22 and Node 24.
- [ ] `node packages/create-mcp/dist/index.js --version`-style placeholder run exits 0
      and prints the package version.
- [ ] `pnpm -r exec npm pack --pack-destination <tmp>` succeeds; `tar -tvf` of the
      create-mcp tarball shows `templates/` and an executable-mode `dist/index.js`;
      the kit tarball shows `dist/` (with executable `dist/bin.js`) and no `src/`.
- [ ] `git status` clean after full build (`dist/` ignored).
- [ ] LICENSE is exact MIT text with year 2026 and holder Saber5656.

## Validation

Run the five root scripts locally on Node 22 (`nvm use 22`) and 24; run the two
`npm pack --dry-run` listings and paste them into the PR description.

## Dependencies

None (first issue).

## Non-goals

CI workflow (issue 02), community docs (03), any real CLI/kit logic, changesets
setup (25).

## Design References

DESIGN.md §2, §2.1, §3.1, §6.1; ADR-003 (names), ADR-005 (toolchain);
research survey §6 (version pins).
