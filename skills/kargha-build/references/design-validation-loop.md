# Phase 7 — Design validation loop (full procedure)

Loaded by `kargha-build` Phase 7 when a design-validation tool/skill is available (and for the manual-checklist fallback it describes). Skip the whole loop when `DESIGN_VIEW` is `none`. The exit bar — zero critical and zero major discrepancies, capped at 3 rounds — and the Phase 1 invocation parameters are summarized in the SKILL.md Phase 7 pointer; this file holds the per-round detail.

## Contents
- Per-round structure (tool-available case)
- 7a. Invoke the design-validation tool/skill
- 7b. Parse the report and decide
- 7c. Implement fixes (main thread)
- 7d. Loop control
- 7e. Edge cases

---

This is the core frontend differentiator. **If a design-validation tool/skill is available** (resolved in Project configuration), invoke it, parse its structured discrepancy report, fix issues, and re-run — up to 3 rounds maximum. **If no design-validation tool exists**, this phase degrades to a manual visual checklist: open the app route (the dev server from Phase 6 is still started for this), compare it against the design file/view per `DESIGN_NAVIGATION`, and fix obvious critical/major discrepancies, capped at 3 fix passes (the same hard cap as the automated loop).

**Per-round structure (tool-available case):**

#### 7a. Invoke the design-validation tool/skill

Invoke the configured design-validation skill/tool. It is stack-agnostic: it takes a design-file path, an app URL/route, and navigation instructions, and returns a structured discrepancy report. Pass the parameters extracted from the ticket in Phase 1, including the optional fields from `DESIGN_VALIDATION_PARAMS` when present:

```
Design file: <DESIGN_FILE_PATH>
App base URL: <dev-server-url>  (the resolved dev-server URL, e.g. http://localhost:3000)
App route: <APP_ROUTE>  (served at <dev-server-url><APP_ROUTE>)
Design navigation: <DESIGN_NAVIGATION>
App navigation: <app-side steps to reach the view, from DESIGN_VALIDATION_PARAMS — pass when present; REQUIRED for slideout/modal/detail sub-views, else the tool only captures the route's initial render>
Viewport: <from DESIGN_VALIDATION_PARAMS if specified, else the tool default>
Focus areas: <from DESIGN_VALIDATION_PARAMS if the ticket scoped them, else omit>
```

Prefer the pre-authored `DESIGN_VALIDATION_PARAMS` values (focus areas, app navigation, viewport, theme context) over the raw extracted ones. A well-built design-validation tool manages its own capture + comparison internals and any HTTP server it needs for the design file — invoke it and parse the returned report; do not replicate its internals.

**Theme contexts (when the resolved theme/token system declares them).** Run the validation loop against the base context the ticket specifies (default: base/light). For an alternate context the ticket lists, the default is a cheap **smoke check** after the loop passes — this needs nothing from the design prototype. Concretely, activate the context the way the **resolved theme-context selector** demands — an attribute selector (`[data-theme="dark"]`) → `document.documentElement.setAttribute(...)`; a class (`.dark`) → toggle that class on the scope element; `prefers-color-scheme` → emulate the media (e.g. Chrome DevTools `Emulation.setEmulatedMedia`), since attribute/class mutation won't trigger it; or use the app's own toggle if it has one. Then confirm (a) the route still renders without console errors and (b) a few key theme variables resolve non-empty (read them on the element the selector scopes to, not always `documentElement`) — read them with `getComputedStyle(document.documentElement).getPropertyValue('--<var>')` for representative variables (surface, text, accent) and check none come back empty. An empty value means an alternate-context override is missing. Run a **full second validation loop** only when `DESIGN_VALIDATION_PARAMS` marks the context `full` **and** supplies design + app navigation that switches both into it: the design-validation tool compares against the design *as navigated* and has no theme input of its own, so without a switchable design prototype a "full" dark loop would compare a dark app against a light design and burn all three rounds. Absent the `full` marker and switch navigation, do the smoke check only.

#### 7b. Parse the report and decide

Expected report structure:

```
STATUS: <match | partial | mismatch>
SUMMARY: <assessment>
DISCREPANCIES:
  - DIMENSION: <layout|colors|typography|spacing|components|hierarchy|interactive|content>
  - SEVERITY: <critical|major|minor|cosmetic>
  - ELEMENT: <what element or area>
  - DESIGN: <what the design shows>
  - APP: <what the app shows>
  - NOTES: <context>
TOKEN_DRIFT: <token differences>
MISSING_ELEMENTS: <present in design, absent in app>
EXTRA_ELEMENTS: <present in app, absent in design>
```

Decision logic:

| Condition                                                                | Action                                       |
| ------------------------------------------------------------------------ | -------------------------------------------- |
| `STATUS: match`                                                          | Exit loop — validation passed                |
| `STATUS: partial` with zero `critical` and zero `major` discrepancies    | Exit loop — good enough                      |
| `STATUS: partial` or `mismatch` with `critical` or `major` discrepancies | Fix and re-run                               |
| Round 3 reached with residual issues                                     | Stop — surface to user via `AskUserQuestion` |

The "good enough" threshold: zero critical and zero major. Minor and cosmetic issues are acceptable — they can be addressed in PR review or follow-up tickets.

#### 7c. Implement fixes (main thread)

When fixing discrepancies between rounds:

- Address `critical` discrepancies first, then `major`
- Use the `DESIGN` and `APP` values from the report to understand exactly what needs to change
- Cross-reference the `COMPONENT_LIBRARY_MAP` from the ticket — fixes must use the correct `<component-lib>` components and the project's theme/token values
- **DTCG token systems:** reverse-map **both sides** of each `TOKEN_DRIFT` entry through the token manifest before fixing — e.g. design `#f6f4ef` → `semantic.color.canvas`, app `#ffffff` → `semantic.color.surface`: the element consumes `--surface` where the design expects `--canvas`. That names the right consuming variable to swap, or reveals a missing token (which routes through the Phase 4d tier-gated creation rule — then rebuild tokens and the manifest before the next round)
- After fixes, re-run `<lint command> && <test command>` before the next validation round — fixes must not break compilation

#### 7d. Loop control

```
round = 1
while round <= 3:
  # Re-run lint/test if we made fixes (skip for round 1 — Phase 5 already passed)
  if round > 1:
    run <lint command> && <test command>
    if fail: fix, re-run lint/test

  invoke the design-validation tool/skill
  parse STATUS and DISCREPANCIES

  if STATUS == "match":
    break

  if STATUS == "partial":
    critical_count = count where SEVERITY == "critical"
    major_count = count where SEVERITY == "major"
    if critical_count == 0 and major_count == 0:
      break

  if round == 3:
    surface residual discrepancies to user via AskUserQuestion
    break

  implement fixes based on report
  round += 1
```

This loop validates **one** context — the base context the ticket specifies, using the base Design/App navigation. The alternate-context handling from 7a wraps it: after this loop passes, run the cheap **smoke check** for any alternate context the ticket lists, and run this loop a **second time** (against the alternate context) only when `DESIGN_VALIDATION_PARAMS` marks that context `full`. For the second loop, append the params' **Context switch** steps to the navigation (the base navigation stays base-only, so the first loop really validated base); if no Context switch is supplied, the alternate context can't be switched — fall back to the smoke check. **Guard against regressions:** if a fix made during the alternate-context loop touches shared (non-context-specific) CSS, re-run one base-context verification round before opening the PR — a dark-mode fix can break light. The 3-round cap applies per loop invocation.

#### 7e. Edge cases

- **The design-validation tool crashes or times out:** Treat as a failed round. Retry once. If it fails again, skip design validation entirely and surface the error to the user — don't block the PR on a tooling failure.
- **Dev server degrades during validation:** If the tool reports the app is unhealthy/degraded (or the route stops responding), restart the dev server (repeat Phase 6 steps) and retry the round.
- **Design file path is relative:** Resolve it against the worktree root, not the main checkout.
