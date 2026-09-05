# Title

Add Changesets release pipeline with npm provenance

## Summary

Set up Changesets for independent versioning of the two packages and the standard
Changesets **version-PR release workflow** (the action maintains a "Version
Packages" PR on `main`; merging it publishes to npm with **provenance via OIDC
Trusted Publishing** and pushes the release tags it creates) — no long-lived npm
tokens anywhere — plus the documented manual user prerequisites (npm account/scope,
Trusted Publisher registration).

## Context

DESIGN §11 and §9 BND-7. Credentials policy (repo operating rules): agents never
create or handle npm tokens; the npm-side setup is the user's manual step, and this
issue must make that step a precise checklist.

## Scope

Changesets config + CONTRIBUTING section, `.github/workflows/release.yml`, package
metadata final pass (`publishConfig`, `files`, `exports`, `repository.directory`),
and `docs/RELEASING.md`. First actual publish is executed by the user following
RELEASING.md — not by CI on this issue's PR.

## Detailed Requirements

1. Changesets: `@changesets/cli` ^2.31, default config with
   `access: public`; `pnpm changeset` documented in CONTRIBUTING (one paragraph +
   example); linked-packages NOT used (independent semver per DESIGN §11); the
   create-mcp devDependency on the kit (DESIGN §4.3 mechanism) gets a note in
   `.changeset/config.json` `updateInternalDependencies: "patch"` so kit releases
   propagate into create-mcp's next release automatically.
2. `release.yml` (the Changesets version-PR model per DESIGN §11 — tag-push
   triggers are not used; the action itself pushes tags after publishing):
   - trigger: `push` to `main` (plus `workflow_dispatch` for recovery) — the
     changesets/action opens/updates the "Version Packages" PR; when that PR
     merges, the same workflow run publishes;
   - `permissions: { contents: write, pull-requests: write, id-token: write }`
     scoped to the release job only; top-level stays `contents: read`;
   - publish step: `pnpm changeset publish` with `NPM_CONFIG_PROVENANCE=true`;
     **no `NPM_TOKEN` secret** — Trusted Publishing (OIDC) is the auth path;
   - guard: publishing is gated by branch protection — RELEASING.md instructs the
     user to add `quality`, `kit-integration`, `cli-e2e`, and `dogfood` to the
     required status checks of `main` (ruleset), so the "Version Packages" PR
     cannot merge (and thus nothing publishes) unless the full release gate is
     green on its merge commit; the publish job itself re-runs no tests;
   - changesets/action pinned by SHA like every other action.
3. Package metadata final pass (both packages):
   - kit: `exports` map (`.` → dist ESM entry + types) and `"sideEffects": false`;
   - create-mcp: no `exports` field (bin-only package, nothing importable);
   - both: `repository: { type, url, directory }`, `keywords` (mcp,
     modelcontextprotocol, scaffold, conformance, wizard, badge), `description`
     strings matching README pitches.
4. `docs/RELEASING.md` — the user's manual checklist:
   - npm account + `@saber5656` scope availability confirmation (U1);
   - npm → GitHub Trusted Publisher registration for BOTH package names with this
     repo + `release.yml` as the trusted workflow (exact npm UI path documented);
   - repo settings: default-branch protection already assumed; environments not
     required in v1;
   - first publish: one Changesets publish run releases both packages together —
     no manual ordering is needed (the devDependency reference does not gate
     publishing; it only needs to resolve when generated projects install, i.e.
     after both are live). RELEASING.md instructs verifying both packages are
     visible on the registry before announcing;
   - post-publish QA: `npm create @saber5656/mcp@latest` smoke on a clean machine
     (U5 check), badge proof if 16 deferred it.
5. Dry-run proof on this issue's PR: `pnpm changeset version` on a scratch branch +
   `pnpm -r publish --dry-run` output pasted; workflow lint (actionlint) green.

## Acceptance Criteria

- [ ] `pnpm changeset` → version → `publish --dry-run` chain works locally
      (transcript in PR); tarball contents re-verified (templates included).
- [ ] `release.yml` passes actionlint; SHA-pinned; `id-token: write` present only
      in the publish job; zero secrets referenced.
- [ ] RELEASING.md checklist complete enough that the user can execute first
      publish without asking anything (self-audit against U1/U5).
- [ ] `.changeset/config.json` committed with the settings from requirement 1.
- [ ] CONTRIBUTING.md updated (changeset-per-PR rule activated, replacing the
      "after release pipeline exists" placeholder from issue 03).

## Validation

Dry-run transcripts + actionlint in CI. Real publish intentionally deferred to the
user (documented exit criterion: RELEASING.md exists and dry-runs pass).

## Dependencies

24 (release gate must exist before enabling a publish path).

## Non-goals

Executing the first real publish (user-manual), GitHub Releases/changelog
automation beyond Changesets defaults, provenance for the dogfood tarballs,
canary/pre-release channels (v2).

## Design References

DESIGN.md §9 BND-7, §11; ADR-003; ISSUE_PLAN §8 U1, U5.
