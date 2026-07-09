# Title

Author generated README template with badge and security notes

## Summary

Author `templates/ts/base/README.md.tmpl`: the generated project's README with the
conformance badge, quickstart per transport, client-connection snippets, conformance
documentation (including expected-failures usage), and a security-notes section that
matches the template's actual defaults.

## Context

The README is where the badge lives (ADR-006) and where BND-6's residual risks are
communicated to users (DESIGN §5.4 auth non-goal, HOST warning). It must never
overclaim what the badge proves.

## Scope

Three README template variant files (one per transport) with disjoint
`when.transport` manifest entries, all rendering to `README.md`. Token-only
rendering cannot drop sections inside a single file (ADR-004: no in-file
conditionals), so per-variant files are the mechanism — same pattern issue 12 uses
for package.json.

## Detailed Requirements

1. Files `base/README.stdio.md.tmpl`, `base/README.http.md.tmpl`,
   `base/README.both.md.tmpl`, each a complete document; they stay under `base/`
   (the manifest `when`, not the directory, decides selection — DESIGN §4.2).
2. Common content (all variants):
   - `# __MCP_TMPL_PACKAGE_NAME__` heading;
   - badge line:
     `![MCP Conformance](https://github.com/__MCP_TMPL_BADGE_PATH__/actions/workflows/conformance.yml/badge.svg)`
     immediately followed by a "Activate this badge" callout: replace
     `your-github-user/your-repo` (the literal substituted value of
     `__MCP_TMPL_BADGE_PATH__`, DESIGN §4.3) with the real owner/repo after pushing;
   - "What the badge proves" paragraph: exact ADR-006 semantics (official suite
     `active` @ spec 2025-11-25 + smoke checks, on default branch) and what it does
     not prove;
   - Quickstart: install (PM-neutral wording: "npm install / pnpm install"), build,
     test, `conformance` (with the build-first rule from DESIGN §5.5);
   - Project structure table (files from DESIGN §4.1 for that variant);
   - "Extending the server": add a tool via `src/tools/`, re-run conformance;
   - Conformance section: what runs per variant, `conformance-results/report.json`,
     expected-failures baseline how-to (issue 15's file), link to the official suite
     repo and MCP Inspector;
   - Security notes: loopback-bind default and the 0.0.0.0 warning (http variants),
     no-auth-in-v1 statement with link to MCP authorization spec, `.env` hygiene
     (all variants), "logging goes to stderr" note (stdio variants).
3. stdio variants additionally: client-config snippet for a local MCP client
   (Claude Desktop/Cursor-style JSON with `command: "node"`,
   `args: ["<abs path>/dist/stdio.js"]` and a "paths must be absolute" warning).
4. http variants additionally: endpoint URL, session header note, curl example for
   initialize (copy-paste runnable), PORT/HOST env table mirroring `.env.example`.
5. A shared-content sync test (like issue 12's): the three files' common sections
   (badge line, badge semantics paragraph, security `.env` note) are byte-identical
   — extract-and-compare in a unit test to prevent drift.
6. No invented commands: every command shown must exist in the corresponding
   package.json variant (test greps script names against the variant tmpls).

## Acceptance Criteria

- [ ] Manifest coverage green; three entries with disjoint `when.transport`.
- [ ] Sync test and script-existence test pass.
- [ ] Badge URL renders correctly after substitution with a dummy owner/repo
      (string-level test).
- [ ] Every external link HTTPS and resolvable (link-check step or manual pass
      pasted in PR).
- [ ] The "what the badge proves / does not prove" wording matches ADR-006's two
      claims verbatim.

## Validation

Repo tests in CI + manual render preview (GitHub markdown) of the `both` variant
pasted as a screenshot in the PR.

## Dependencies

13, 14, 15, 16.

## Non-goals

Root repo README (03/27), CHANGELOG generation, localized READMEs, badge activation
automation (wizard fills `__MCP_TMPL_BADGE_PATH__` with the placeholder only —
detecting the real remote is a v2 idea).

## Design References

DESIGN.md §4.3 (BADGE_PATH token), §5.4, §5.5, §6.3; ADR-004 (variant files),
ADR-006.
