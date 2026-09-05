# ADR-001: Wrap the official conformance suite instead of building a custom runner

Date: 2026-07-10 — Status: Accepted (user-confirmed)

## Context

The original concept ("independent conformance runner") predated the discovery that
`@modelcontextprotocol/conformance` exists (v0.1.16, active, official GitHub Action,
expected-failures baselines, `checks.json` output). It is becoming normative:
Standards-Track SEPs must land scenarios in it, and official SDKs are tiered against
it. It tests servers over Streamable HTTP only (`server --url`); it cannot test stdio
servers. Evidence: `docs/research/2026-07-mcp-ecosystem-survey.md` §2.

## Decision

`@saber5656/mcp-conformance-kit` is an **orchestrator**, not a test suite:
build/launch/stop the server under test, execute the exactly-pinned official suite
against the HTTP endpoint, parse `checks.json`, render reports/badge summaries. stdio
servers get a clearly-labeled **smoke-check catalog** of our own (DESIGN.md §6.5),
which never claims to be official conformance.

An adapter layer (`adapter/official.ts` + contract tests) isolates upstream 0.x drift;
the engine version is an exact dependency executed via its installed bin — never `npx`
at run time.

## Consequences

- Badge claims inherit official credibility ("passes the official suite") and spec
  updates arrive by bumping one pinned dependency.
- We accept coupling to a 0.x tool; mitigated by the adapter, contract tests, and
  Dependabot-gated upgrades.
- stdio verification depth is limited to our smoke checks in v1.

## Alternatives rejected

- **Custom full runner**: duplicates official work, produces a weaker
  "self-certified" badge, and carries permanent spec-chasing maintenance.
- **Hybrid (own runner primary + official optional)**: maximizes v1 effort and doubles
  maintenance for marginal benefit.
