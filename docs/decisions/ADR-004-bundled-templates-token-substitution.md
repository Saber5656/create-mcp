# ADR-004: Bundled offline templates with literal token substitution

Date: 2026-07-10 — Status: Accepted

## Context

Scaffolders either bundle templates in the published package or fetch them from a
remote (GitHub) at run time. Remote fetching creates a runtime trust boundary
(tag pinning, TLS, tampering, availability) and breaks offline use. Separately,
expression-evaluating template engines (EJS & co.) put an eval-capable machine on the
path between package content and the user's disk.

## Decision

1. Templates ship **inside** `@saber5656/create-mcp` (`files: ["dist", "templates"]`).
   Generation performs no network I/O. The npm-audited artifact is the whole input.
2. Rendering is **literal string replacement** of a closed token set
   (`__MCP_TMPL_*`, DESIGN.md §4.3) on manifest-flagged files only. No template
   engine, no expressions, no conditionals inside file content.
3. Variance between transport variants is expressed **structurally** in the manifest
   (`when.transport` file conditions), not by in-file logic. If a file needs to differ
   beyond token values, it becomes two files.

## Consequences

- Reproducible, offline, supply-chain-minimal generation; trivially auditable diffs
  between template and output.
- Package tarball carries the templates (size cost: trivial for text files).
- Some duplication across variant files is accepted deliberately; the manifest
  coverage test (every template file referenced exactly once) keeps it honest.

## Alternatives rejected

- **Runtime GitHub fetch**: new attack surface + availability dependency for zero v1
  benefit (we control exactly one template set).
- **EJS/Handlebars rendering**: expressive power we don't need, at eval-adjacent risk
  and worse template readability (templates stop being valid TypeScript files).
