# Title

Implement interactive wizard prompts

## Summary

Implement `src/prompts.ts`: the @clack/prompts wizard implementing the S0–S7 state
flow of DESIGN.md §3.3 — asking only for unresolved fields, validating inline via
issue 19's functions, and ending with a confirm-summary step. Cancellation at any
point aborts with no writes and exit 130.

## Context

The wizard is the product's face ("A wizard for generating MCP server templates…").
It plugs into options resolution (18) as the injected `promptFn`, so all logic here
is presentational + flow; decisions about *what* needs asking were made in 18.

## Scope

`prompts.ts` + unit tests (clack is driven through an injected prompt-adapter
interface so tests run TTY-less; a thin default adapter binds real clack calls).
No generation logic.

## Detailed Requirements

1. Implement `promptForMissing(unresolved: FieldSet, partial: Partial<ResolvedOptions>):
   Promise<PromptResult>` with these exact types (shared in `context.ts`):
   ```ts
   type Field = "targetDir" | "packageName" | "transport" | "packageManager" | "git" | "install";
   type FieldSet = ReadonlySet<Field>;   // issue 18: every field NOT explicitly
                                         // provided via argv/flags — defaultable
                                         // fields are included when not explicit
   type PromptAnswers = Partial<{
     targetDir: string; packageName: string;
     transport: "stdio" | "http" | "both";
     packageManager: "npm" | "pnpm";
     git: boolean; install: boolean;
   }>;                                    // exactly the prompted keys are present
   type PromptResult = { cancelled: true } | { cancelled: false; answers: PromptAnswers };
   ```
   Defaults from issue 18's table appear as the pre-selected choice of each prompt.
2. Flow and copy (exact strings, EN):
   - intro: `create-mcp — MCP server project generator`;
   - S1 targetDir (text): "Where should the project be created?" placeholder
     `./my-mcp-server`; inline-validate via `checkTargetDir` (message table from 19);
   - S2 packageName (text): default = sanitized basename (19); validate via
     `validatePackageName`;
   - S3 transport (select): `both` (hint: "stdio + Streamable HTTP — recommended"),
     `stdio` ("local process servers"), `http` ("remote-capable HTTP server");
   - S4 packageManager (select): detected one first with "(detected)" hint;
   - S5 git (confirm, default yes); S6 install (confirm, default yes);
   - S7 summary: render resolved table (dir, name, transport, pm, git, install) via
     `note()`, then confirm "Create project?" — decline → cancelled.
3. Each prompt is skipped when its field is not in `unresolved` (18 computed that);
   S7 always runs when any prompt was shown, never in flag-only runs.
4. Cancellation: clack cancel symbol at any step → single outro line
   `Cancelled — nothing was written.` and `{ cancelled: true }`; no partial state
   leaks (function-local only).
5. Non-TTY guard: if called with `!process.stdin.isTTY`, throw a kit-internal error
   (18's matrix should have prevented the call) — assert-style, tested.
6. All user-echoed values pass through a display-escape helper (strip control
   chars) before rendering in the summary — the user's own input isn't hostile to
   themselves, but paste artifacts (ANSI) must not corrupt the summary.

## Acceptance Criteria

- [ ] Adapter-driven unit tests: full-ask flow, partial-ask flow (only transport
      missing), inline validation retry path (bad name then good name), cancel at
      each of S1/S3/S7, summary shows exactly the resolved values.
- [ ] Non-TTY call throws (test).
- [ ] Copy strings match requirement 2 (snapshot).
- [ ] Manual TTY pass on macOS Terminal recorded in the PR (screenshot or pasted
      transcript) — arrows/enter/ctrl-c behave.

## Validation

Unit tests in CI + the manual TTY checklist in the PR description.

## Dependencies

18, 19.

## Non-goals

Flag parsing (18), any fs writes, remembering previous answers (v2), i18n (EN only
in v1), fancy theming beyond clack defaults.

## Design References

DESIGN.md §3.1, §3.3, §3.4 (message table reuse), §3.2 (exit 130).
