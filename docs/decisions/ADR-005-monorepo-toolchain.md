# ADR-005: pnpm-workspaces monorepo with tsup / vitest / Biome

Date: 2026-07-10 — Status: Accepted

## Context

Two published packages (wizard, kit) share a repo, TypeScript config, CI, and a
release pipeline. Tooling should minimize dependency count and configuration drift
while staying mainstream enough for OSS contributors.

## Decision

- **pnpm workspaces** (`packages/*`), single lockfile, `engines.node >= 22`
  (Node 22 = maintenance LTS to 2028-04, Node 24 = active LTS; CI matrix runs both).
- **tsup** builds both packages (ESM, `dist/`), TypeScript ^5.9 strict mode,
  `"type": "module"` everywhere.
- **vitest** for all tests (also inside generated projects — one mental model).
- **Biome** for lint+format (single tool replaces eslint+prettier pair).
- **Changesets** for versioning/publishing (independent semver per package).

Baseline versions: research survey §6. Note: pnpm is the *repo's* toolchain; generated
projects default to the user's chosen package manager (npm or pnpm) and are
single-package, tool-light (`tsc` build, no tsup).

## Consequences

- One-command DX (`pnpm install && pnpm test`); contributors need pnpm (documented in
  CONTRIBUTING.md).
- Biome instead of eslint is a mild contributor-familiarity trade accepted for the
  dependency-surface reduction (aligns with the security posture).

## Alternatives rejected

- npm workspaces (weaker isolation/perf, no material dependency saving);
  separate repos (release/docs/CI duplication for tightly-coupled packages);
  eslint+prettier (two toolchains to configure and secure instead of one).
