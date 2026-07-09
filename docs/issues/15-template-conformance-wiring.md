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

Template files `http/conformance.config.json.tmpl` (http/both content difference
handled per ADR-004 rules), baseline YAML stub, npm script rows in the three
package.json variant tmpls (12), manifest entries. No workflow file (16).

## Detailed Requirements

1. Config files (complete JSON per variant, disjoint `when`):
   - **both**: `$schema` node_modules path (DESIGN §5.5), `specVersion:
     "2025-11-25"`, `http` section (`url http://127.0.0.1:3000/mcp`, `start:
     ["node","dist/http.js"]`, `readyTimeoutMs: 30000`, `suite: "active"`), `stdio`
     section (`command: ["node","dist/stdio.js"]`).
   - **http**: same minus `stdio`.
   - **stdio**: config with only `stdio` section (file lives under a manifest entry
     with `when: ["stdio"]` — place source at `stdio/conformance.config.json` to
     keep directory semantics; adjust manifest accordingly).
2. Scripts (update the three package.json.tmpl variants from issue 12):
   - all variants get exactly `"conformance": "mcp-conformance-kit run"` — bare,
     package-manager-agnostic, no `pre`/`post` lifecycle magic, no embedded build.
   - Building first is an explicit caller responsibility (DESIGN §5.5): the CI
     workflow (16) and the README quickstart (17) always run the build script
     immediately before `conformance`.
3. Baseline: `conformance-expected-failures.yaml` shipped **commented-out only**
   (all-comments file explaining format + a link to the official suite's baseline
   docs, and the rule "every entry needs a date + evidence link"). NOT referenced
   from the config by default (`expectedFailures` omitted); README shows how to
   enable it.
4. Manifest entries with correct `when` sets; coverage test green.
5. Validate config tmpls against the kit's published JSON Schema in a repo test:
   after token substitution with dummy values, each variant config must pass the
   schema in `packages/conformance-kit/schema/conformance-config.schema.json`
   (path-based cross-package test inside this monorepo).

## Acceptance Criteria

- [ ] Schema-validation test: all three substituted configs parse under the kit
      schema with zero errors.
- [ ] Scripts test (from 12's invariant harness): `conformance` script identical
      across variants and equal to `mcp-conformance-kit run`.
- [ ] Baseline file contains only comment lines (`grep -v '^#' | grep -v '^$'` is
      empty).
- [ ] Manifest coverage green; no token residue rules hold (config tmpls carry
      tokens only where defined in DESIGN §4.3).

## Validation

CI green; paste schema-validation test output in PR.

## Dependencies

04 (schema file exists), 13, 14.

## Non-goals

CI workflow (16), README documentation (17), kit behavior changes, non-default
suites/scenarios in generated config (users edit; documented in 17).

## Design References

DESIGN.md §5.1, §5.5, §7.3; ADR-001, ADR-004; ISSUE_PLAN §8 U3.
