# Title

Implement report renderers (terminal, JSON, GitHub summary)

## Summary

Implement `report/json.ts`, `report/terminal.ts`, and `report/github-summary.ts`:
persist `conformance-results/report.json`, print a readable terminal table with
strict sanitization of server-controlled strings, and append a markdown summary to
`$GITHUB_STEP_SUMMARY` when running in GitHub Actions.

## Context

DESIGN.md §6.1 module layout, §7.4 report shape, §6.3 (the GitHub summary is the
human-readable proof behind the badge), and §9 BND-4: `detail` strings originate from
arbitrary servers and must never reach the user's terminal unsanitized (ANSI/control
injection).

## Scope

Three renderer modules + unit tests. Input is always a complete `Report` (04). No
process management, no CLI parsing.

## Detailed Requirements

1. `json.ts` — `writeReport(report, reportDir)`: mkdir -p; write
   `<reportDir>/report.json` (2-space indent, trailing newline, UTF-8); returns the
   absolute path. Never merges with previous runs (overwrite).
2. `terminal.ts` — `renderTerminal(report, { color: boolean }): string`:
   - Sections: header (kit version, spec version, timestamps, target), official
     table (scenario / id / status), smoke table (id / level / status), summary line
     (`total/pass/fail/expected_fail/skip`), final verdict line.
   - `color` auto-off when `!process.stdout.isTTY` or `NO_COLOR` env is set (decided
     by the caller in 09; this module just obeys the flag). Status glyphs: pass `✓`,
     fail `✗`, expected_fail `≈`, unexpected_pass `!`, skip `-` (ASCII fallbacks not
     required in v1).
   - **Sanitization** `sanitizeDetail(s)`: strip all C0 controls except `\n`→space,
     strip C1 (0x80–0x9F), strip ESC-initiated sequences (regex for CSI/OSC/SS3),
     collapse to ≤200 chars per rendered detail. Applied to every string that
     originated outside the kit (scenario ids and details from checks.json, smoke
     details). A dedicated unit test feeds ANSI/OSC/BEL/newline-bomb payloads and
      asserts the output is inert plain text.
3. `github-summary.ts` — `appendGithubSummary(report)`:
   - no-op unless `process.env.GITHUB_STEP_SUMMARY` is a writable path;
   - appends: H2 title with verdict emoji, a markdown table per section, the summary
     row, and a footnote naming engine + version + spec version (badge provenance,
     ADR-006). Markdown cell content passes the same sanitizer plus `|`/backtick
     escaping.
4. All three renderers are pure w.r.t. the Report (no mutation) and covered by
   snapshot tests with a canonical fixture Report containing every status value.

## Acceptance Criteria

- [ ] Snapshot tests for terminal (color on/off) and GitHub summary against the
      canonical fixture Report.
- [ ] Injection test: details containing `\x1b]0;pwned\x07`, `\x1b[2J`, raw `\r`,
      and a 10k-char string render as plain, truncated, single-line text in both
      terminal and markdown outputs.
- [ ] `report.json` round-trips: `JSON.parse` of the written file deep-equals the
      input Report.
- [ ] With `GITHUB_STEP_SUMMARY` unset, `appendGithubSummary` performs zero fs calls
      (spied in test).

## Validation

Kit unit tests in CI; paste one rendered terminal block (color off) into the PR.

## Dependencies

04.

## Non-goals

HTML reports, artifact uploading (the workflow template does that — issue 16), badge
SVG generation (ADR-006: GitHub renders the badge), i18n.

## Design References

DESIGN.md §6.1, §6.3, §7.4, §9 BND-4; ADR-006.
