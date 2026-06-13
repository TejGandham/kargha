# kargha

> **करघा** — *a loom.* Load a design, weave it into built, verified frontend, one slice at a time.

A three-skill pipeline that turns a **design prototype** into **reviewed, merged frontend code** — planning the work, implementing it, and validating visual fidelity. The skills are stack-agnostic: each resolves the project's component library, icon set, token/theme system, data layer, toolchain, and ticketing system up front (detect → ask), then works against whatever it finds.

```
design HTML export
   →  plan-frontend-generic     emits vertical-slice tickets (ticketing system OR md/json files)
   →  build-frontend-generic    implements ONE ticket in a git worktree, opens a PR
         →  invokes validate-design each round: compares the running view to the design,
            returns a discrepancy report that build fixes against and re-runs
```

`plan` writes self-contained tickets; `build` picks up one and drives it to an open PR, calling `validate-design` (or any configured design-validation tool) in a capture-compare-fix loop. Each skill is independently usable, but they share one ticket contract so the chain composes.

## The three skills

### plan-frontend-generic

**Decomposes a design export into vertical-slice tickets for frontend implementation.** Its core output is the **component-to-library mapping** — for every design component, whether it maps to a project library component (with which props/variants), needs a thin wrapper, or must be built custom. Every ticket also carries an icon mapping, a design-token map, a route/layout plan, a data-layer plan, acceptance criteria, and (for DTCG projects) a token-changes authorization.

- **In:** a path to a design HTML export, a natural-language scope ("the whole thing" / "just the feed page"), and a ticket destination.
- **Out:** one ticket per slice — emitted to a ticketing system (JIRA, Linear, GitHub Issues, …) **or** as Markdown/JSON files (`NN-slug.md` + an `index.md` manifest).
- **Invoke with:** "plan the frontend for this design", "break this design into tickets", "create frontend tickets from this design", "slice this design into implementation work".

### build-frontend-generic

**Implements one ticket end-to-end in an isolated git worktree**, from a pickup-eligible status to an open PR. It gates on ticket status/assignee, picks the ticket up, spins up a per-ticket worktree, implements the components against the project's conventions, runs lint/test, optionally validates data-layer conformance (Phase 5b) and DTCG token conformance (Phase 5), starts the dev server, runs the design-validation loop (Phase 7, up to 3 rounds), and opens a PR.

- **In:** a single ticket authored by `plan-frontend-generic` — a ticketing-system id/key/URL **or** a Markdown/JSON ticket file. The ticket must contain a **Design Reference** section.
- **Out:** implemented code in a worktree, an open PR (or pushed branch + compare URL on hosts without PRs), and the ticket moved to a review status.
- **Invoke with:** "build this frontend ticket", "implement the frontend ticket", "pick up the frontend work", or any request to implement a ticket with a Design Reference section.

### validate-design

**Compares one running view against its design prototype** and reports structured discrepancies — it never fixes them; the caller decides what to act on. It uses two fresh-context subagents: a **capture** agent (drives `playwright-cli` to screenshot + snapshot both the design prototype and the live app) and a **comparison** agent (sees only the captured data, emits the report).

- **In:** the design HTML file, the app base URL + route, design/app navigation instructions, optional viewport and focus areas.
- **Out:** a structured report — `STATUS` (match/partial/mismatch), severity-rated `DISCREPANCIES` across layout/color/typography/spacing/components/hierarchy, `TOKEN_DRIFT`, and missing/extra elements.
- **Invoke with:** "validate against the design", "compare implementation to design", "check design fidelity", "does this match the design".
- One view per invocation — the calling pipeline loops for multiple views.

## How they connect (the ticket contract)

`plan` emits, and `build` extracts and implements, a fixed set of ticket sections:

| Ticket section | Produced by `plan` | Consumed by `build` |
|-|-|-|
| Design Reference (design file, view, navigation) | Phase 5 template | Gate 3 + Phase 1 extraction |
| Component-to-Library Map | Phase 4a | Phase 4d implementation |
| Icon Mapping / Missing Icons | Phase 4a.1 | Phase 4d (exact imports) |
| Design Token Map (+ `00-token-map.md`) | Phase 4a.2 | existing-token decisions |
| Token Changes (DTCG) | Phase 4a.2 / template | Phase 4d tier-gated creation |
| Route / Layout (+ Served URL) | Phase 5 template | `APP_ROUTE`, health check |
| Acceptance Criteria | Phase 5 template | done-definition |
| Validation parameters | Phase 5 template | Phase 7 design-validation |

`build` then invokes `validate-design` (Phase 7a) with the design file, app base URL, app route, navigation, and viewport, and parses its `STATUS`/`DISCREPANCIES`/`TOKEN_DRIFT` report — fixing critical/major issues and re-running until **zero critical and zero major** remain (the same bar all three skills agree on), capped at 3 rounds.

## Cross-cutting concepts

- **Stack-agnostic.** No skill assumes a specific component library, icon set, theme system, framework, data layer, ticketing system, or repo layout. Concrete tools shown in the docs (Style Dictionary, Next.js, JIRA, `playwright-cli`, `localhost:3000`, …) are **examples**, resolved per project.
- **Design-input contract.** The design side assumes a **runtime-JSX HTML export** (a Claude Design-style prototype: `.jsx` sources or an inline `text/babel` script, `useState` view switching, inline styles). `plan` gates on this format; other export formats are out of scope.
- **W3C DTCG design tokens.** When the token system is in [W3C DTCG](https://www.designtokens.org/) format (2025.10), the skills map design tokens to the **consumable (semantic) tier only**, derive a token manifest from the generated CSS (never re-implementing DTCG resolution), and let `plan` author a **Token Changes** section that pre-authorizes `build` to create additive semantic tokens autonomously (a deterministic Phase 5 check enforces no primitive-tier consumption and no hand-edited generated artifacts). Non-DTCG token systems use the generic "no hardcoded values that duplicate tokens" rule.
- **Theme contexts.** Alternate contexts (e.g. dark mode) get a cheap render/empty-variable smoke check by default; a full second design-validation loop is opt-in and requires a switchable design prototype, reached through navigation.

## Requirements

- **`build`** needs `git` (uses per-ticket worktrees), the project's package manager + toolchain (lint/test/build/dev), and access to the ticket source (a ticketing MCP/CLI/REST, or a ticket file).
- **`validate-design`** needs [`playwright-cli`](https://playwright.dev) (`npm install -g @playwright/cli@latest`), a browser (Chromium), and `python3` (serves the design HTML over HTTP). The running app must already be up — the caller owns the dev server's lifecycle.
- **`plan`** needs read access to the design export and the project repo; it writes only tickets/files, never implementation code.

## Usage

Invoke a skill by pointing it at its input — e.g. "plan the frontend for `./design-export/`", then "build ticket `03-signal-feed.md`". `build` calls `validate-design` automatically when a design-validation tool is configured; you can also run `validate-design` standalone to spot-check fidelity. See each skill's own `.md` file for the full phase-by-phase workflow.
