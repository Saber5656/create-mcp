# Title

Implement kit config schema and report data model

## Summary

Implement `src/config.ts` (zod schema + loader for `conformance.config.json`) and
`src/report/model.ts` (Report/Check types + helpers) in the conformance kit, plus
build-time export of the config JSON Schema to `schema/conformance-config.schema.json`.
These are the kit's two public data contracts; all later kit issues consume them.

## Context

DESIGN.md §7.3 defines the config schema (zod, `.strict()`, loopback URL refinement)
and §7.4 the report model. The JSON Schema file is referenced by generated projects'
`$schema` field for editor validation (DESIGN §5.5).

## Scope

`packages/conformance-kit/src/config.ts`, `src/report/model.ts`, a small
`scripts/emit-schema.ts` (or tsup hook) that writes the JSON Schema during build, and
unit tests. No process spawning, no CLI, no rendering.

## Detailed Requirements

1. Implement `KitConfig` exactly as DESIGN.md §7.3: `HttpTarget`, `StdioTarget`,
   `.strict()` objects, `specVersion: z.literal("2025-11-25")`, top-level refinement
   "at least one of http/stdio". zod v4.
2. Loopback refinement `isLoopbackOrExplicitOptIn`: accept URLs whose hostname is
   `127.0.0.1`, `::1`, or `localhost`; anything else fails schema validation with
   message "non-loopback URL requires --allow-remote". Export a second parse entry
   `parseConfig(json, { allowRemote: boolean })` that relaxes only this refinement —
   the CLI flag (issue 09) feeds it. **No DNS resolution** — string hostname check only.
3. Loader `loadConfig(path?)`:
   - default path `conformance.config.json` in cwd;
   - reads file (max 1 MiB, reject larger with a config error), `JSON.parse`, zod
     parse; zod issues are re-rendered as `<jsonPath>: <message>` lines.
   - Returns discriminated result `{ ok: true, config } | { ok: false, errors: string[] }`
     — never throws for user-input problems; throws only on kit bugs.
4. Report model per DESIGN.md §7.4: export the TS types, plus
   `emptyReport(kitVersion, specVersion)`, `summarize(report)` (recomputes `summary`
   from checks; single source of truth), and `mergeSections(report, official?, smoke?)`.
   `summarize` counts `expected_fail` and `unexpected_pass` per §7.4 statuses;
   `unexpected_pass` counts as fail in `summary.fail`.
5. JSON Schema emission: use zod v4's native JSON Schema conversion
   (`z.toJSONSchema`); write to `schema/conformance-config.schema.json`; wire into the
   package build script so `pnpm build` refreshes it; commit the generated file; add
   `schema/` to the package `files` array.
6. All exports flow through `src/index.ts` (public API surface).

## Acceptance Criteria

- [ ] Unit tests cover: valid http-only / stdio-only / both configs; unknown key
      rejection at every level (strict); missing both targets; non-loopback URL
      rejected by default and accepted with `allowRemote: true`; `readyTimeoutMs`
      bounds; oversized file rejection; malformed JSON produces `ok: false` with a
      pointerful message (not a throw).
- [ ] `summarize` unit tests: mixed statuses produce correct totals;
      `unexpected_pass` counted in `fail`.
- [ ] `schema/conformance-config.schema.json` exists after build, is valid JSON
      Schema (draft 2020-12), and `git diff --exit-code` passes after a rebuild
      (generation is deterministic).
- [ ] The example config in DESIGN.md §5.5 parses successfully verbatim.
- [ ] 100% of exported functions have explicit return types; no `any` in the two files.

## Validation

`pnpm --filter @saber5656/mcp-conformance-kit test` green; paste the emitted JSON
Schema top-level into the PR for eyeball review.

## Dependencies

01.

## Non-goals

Spawning (05), adapter arg-mapping (06), rendering (08), CLI flags (09), widening
`specVersion` beyond the 2025-11-25 literal (v1.1).

## Design References

DESIGN.md §5.5, §6.1, §7.3, §7.4, §9 BND-4 (loopback default).
