# Title

Implement template manifest schema and copy-plan resolver

## Summary

In `@saber5656/create-mcp`, implement `engine/manifest.ts` (zod schema + loader for
`templates/ts/template.json`) and `engine/plan.ts` (pure resolver: manifest +
ResolvedOptions → CopyPlan), enforcing the path-safety rules of DESIGN.md §4.2 and
the "every template file referenced exactly once" invariant.

## Context

The manifest is the sole mechanism deciding what lands on the user's disk (ADR-004:
structural variance, no in-file logic). Path rules are security boundary BND-2's
first half (the copier in issue 21 is the second half).

## Scope

Two modules + unit tests + one repo-level test (manifest ↔ template dir coverage).
The manifest file itself ships with content in issues 12–17; here it may contain a
minimal placeholder entry set used by tests.

## Detailed Requirements

1. Schema per DESIGN §4.2, zod v4, `.strict()` at every level:
   `{ version: literal(1), files: [{ source, target, substitute?: boolean,
   when?: { transport?: array(enum(["stdio","http","both"])).nonempty(),
   packageManager?: array(enum(["npm","pnpm"])).nonempty() } }] }` — a `when`
   object must contain at least one key (refinement).
2. Path rules, validated per entry at parse time (schema refinements):
   relative; no `..` segments; no leading `/` or drive letters; no backslashes;
   after `path.resolve(templateRoot, source)` the result must start with
   `templateRoot + sep` (same for `target` against a hypothetical root) — resolution
   check implemented in the loader, not just regex. Additionally: `source` basenames
   must not begin with a dot (DESIGN §4.1 packaging rule — `dot-` prefixed sources
   map to dotfile targets), `source` may end in `.tmpl`, and `target` must never
   end in `.tmpl`.
3. Duplicate `target` detection at parse time: two entries may share a `target` only
   if no single `(transport, packageManager)` combination satisfies both `when`
   conditions. Implement by enumerating the 3×2 combination space and checking each
   pair of same-target entries; co-occurring duplicates → parse error listing the
   colliding sources and the offending combination.
4. `plan.ts` — `buildPlan(manifest, options, templateRoot, targetRoot, mustCreate):
   CopyPlan` (DESIGN §7.2: the plan carries `targetRoot` and `mustCreate` so the
   copier receives one self-contained value): include an entry iff every present
   `when` key matches (`options.transport ∈ when.transport` AND
   `options.packageManager ∈ when.packageManager`; absent keys always match);
   output absolute source/target paths + `substitute` flag; pure (no fs access —
   existence checks belong to the copier).
5. Repo coverage test: walk `templates/ts/**` files (excluding `template.json`);
   assert the set equals the set of manifest `source` values exactly (both
   directions), so dead files and unshipped files fail CI from issue 12 onward.
6. Loader errors are user-facing strings prefixed `template manifest:` — a broken
   bundled manifest is a packaging defect (CLI maps it to exit 4 in issue 21's
   integration).

## Acceptance Criteria

- [ ] Unit tests: valid manifest parses; each path rule rejects (`../x`, `/abs`,
      `a\\b`, empty string, duplicate co-occurring targets); duplicate targets with
      disjoint transport sets accepted; duplicate targets with disjoint
      packageManager sets accepted (the two workflow variants case); empty `when`
      object rejected; unknown keys rejected; dot-leading `source` rejected;
      `.tmpl`-suffixed `target` rejected.
- [ ] `buildPlan` tested for all six `(transport, packageManager)` combinations
      against a fixture manifest (entry counts and exact target lists asserted;
      fixture includes a packageManager-conditioned pair sharing one target).
- [ ] Coverage test wired and passing with the current (possibly placeholder)
      template content.
- [ ] No fs writes anywhere in the two modules (plan is pure; loader reads only).

## Validation

`pnpm --filter @saber5656/create-mcp test` in CI.

## Dependencies

01.

## Non-goals

Token substitution and file writing (21), template content (12–17), multi-template
support (`templates/<other>/` — v2), condition keys beyond `transport` and
`packageManager` (schema stays closed).

## Design References

DESIGN.md §4.1, §4.2, §7.2, §9 BND-2; ADR-004.
