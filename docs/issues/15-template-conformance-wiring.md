# Title

Wire conformance config and scripts into templates

## Summary

Add `conformance.config.json.tmpl` and the per-variant `conformance` npm scripts so
every generated project runs the right verification for its transport with one
command: official suite via the kit for http/both, smoke checks for stdio, both for
`both`. Include an empty expected-failures baseline with usage documentation.

## Context

DESIGN.md §5.5 fixes the config content and the build-before-run ordering. The kit's
config schema is the contract from issue 04 (consumed as a file format — no code
import across packages). This wiring is what the CI workflow (16) and dogfood e2e
(24) execute.

## Scope

The three conformance config template files under `templates/ts/conformance/`
(DESIGN §4.1), the expected-failures baseline stub, the `conformance` npm script
rows in the three package.json variant tmpls (12), and their manifest entries. No
workflow file (16).

## Detailed Requirements

1. Config files — exact manifest entries (all `substitute: false`; these files
   carry no tokens):

   | source | target | when.transport |
   |---|---|---|
   | `conformance/config.stdio.json.tmpl` | `conformance.config.json` | `["stdio"]` |
   | `conformance/config.http.json.tmpl` | `conformance.config.json` | `["http"]` |
   | `conformance/config.both.json.tmpl` | `conformance.config.json` | `["both"]` |

   Content (complete JSON per variant):
   - **both**: `$schema` node_modules path (DESIGN §5.5), `specVersion:
     "2025-11-25"`, `http` section (`url http://127.0.0.1:3000/mcp`, `start:
     ["node","dist/http.js"]`, `readyTimeoutMs: 30000`, `suite: "active"`), `stdio`
     section (`command: ["node","dist/stdio.js"]`).
   - **http**: same minus `stdio`.
   - **stdio**: same minus `http` (only `$schema`, `specVersion`, `stdio`).
2. Scripts (update the three package.json.tmpl variants from issue 12):
   - all variants get exactly `"conformance": "mcp-conformance-kit run"` — bare,
     package-manager-agnostic, no `pre`/`post` lifecycle magic, no embedded build.
   - Building first is an explicit caller responsibility (DESIGN §5.5): the CI
     workflow (16) and the README quickstart (17) always run the build script
     immediately before `conformance`.
3. Baseline: source `conformance/expected-failures.yaml` → target
   `conformance-expected-failures.yaml`, copied for **all variants** (no `when`),
   shipped **commented-out only** (all-comments file explaining format + a link to
   the official suite's baseline docs, and the rule "every entry needs a date +
   evidence link"). NOT referenced from the config by default (`expectedFailures`
   omitted); README shows how to enable it.
4. Manifest entries exactly as tabled above; coverage test green.
5. Validate the config files against the kit's published JSON Schema in a repo
   test: each of the three files (as shipped — they contain no tokens) must pass
   the schema in `packages/conformance-kit/schema/conformance-config.schema.json`
   (path-based cross-package test inside this monorepo).

## Acceptance Criteria

- [ ] Schema-validation test: all three shipped configs parse under the kit schema
      with zero errors.
- [ ] Scripts test (from 12's invariant harness): `conformance` script identical
      across variants and equal to `mcp-conformance-kit run`.
- [ ] Baseline file contains only comment lines (`grep -v '^#' | grep -v '^$'` is
      empty).
- [ ] Manifest coverage green; the four new entries carry `substitute: false` and
      contain no `__MCP_TMPL_` strings.

## Validation

CI green; paste schema-validation test output in PR.

## Dependencies

04 (schema file exists), 13, 14.

## Non-goals

CI workflow (16), README documentation (17), kit behavior changes, non-default
suites/scenarios in generated config (users edit; documented in 17).

## Design References

DESIGN.md §5.1, §5.5, §7.3; ADR-001, ADR-004; ISSUE_PLAN §8 U3.
