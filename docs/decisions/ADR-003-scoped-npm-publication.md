# ADR-003: Scoped npm publication under @saber5656

Date: 2026-07-10 — Status: Accepted (user-confirmed)

## Context

The unscoped npm name `create-mcp` is taken (zueai, v0.4.17, Cloudflare-Workers
scaffolder, unmaintained since 2025-02). `@saber5656/create-mcp` returned 404
(available) on 2026-07-10. `npm create @saber5656/mcp` resolves to the package
`@saber5656/create-mcp` per npm initializer rules.

## Decision

Publish two scoped packages from this repo:

| Package | Bin | Role |
|---|---|---|
| `@saber5656/create-mcp` | `create-mcp` | wizard CLI |
| `@saber5656/mcp-conformance-kit` | `mcp-conformance-kit` | conformance orchestrator |

Both `publishConfig.access: "public"`. The GitHub repo stays `Saber5656/create-mcp`.
Invocation documented as `npm create @saber5656/mcp` / `pnpm create @saber5656/mcp` /
`npx @saber5656/create-mcp`.

Scope creation, npm account setup, and Trusted Publishing configuration are **manual
user steps** (credentials are never handled by agents).

## Consequences

- Zero collision/squatting risk; reversible — an additional unscoped alias package can
  be added later if a good name is secured.
- Slightly weaker discoverability than an unscoped name; mitigated by README, GitHub
  topics, and keywords.

## Alternatives rejected

- **Find another unscoped name**: naming search becomes a blocking task; repo/package
  name mismatch confuses identity.
- **GitHub-only distribution**: high friction (`npx github:`), no npm discoverability.
