# kargha

> **करघा** — *a loom.* Load a design, weave it into built, verified frontend, one slice at a time.

A three-skill pipeline that turns a **design prototype** into **reviewed, merged frontend code** — planning the work, implementing it, and validating visual fidelity. The skills are stack-agnostic: each resolves the project's component library, icon set, token/theme system, data layer, toolchain, and ticketing system up front (detect → ask), then works against whatever it finds.

```
design HTML export
   →  kargha-plan       emits vertical-slice tickets (ticketing system OR md/json files)
   →  kargha-build      implements ONE ticket in a git worktree, opens a PR
         →  invokes kargha-validate each round: compares the running view to the design,
            returns a discrepancy report that build fixes against and re-runs
```

`kargha-plan` writes self-contained tickets; `kargha-build` picks up one and drives it to an open PR, calling `kargha-validate` (or any configured design-validation tool) in a capture-compare-fix loop. Each skill is independently usable, but they share one ticket contract so the chain composes.

## Layout

```
kargha/
  .claude-plugin/     plugin.json + marketplace.json   (Claude Code packaging)
  .codex-plugin/      plugin.json                      (Codex packaging)
  .agents/plugins/    marketplace.json                 (repo-local Codex marketplace)
  README.md
  skills/
    kargha-plan/      SKILL.md  +  references/{ticket-template.md, dtcg-tokens.md}
    kargha-build/     SKILL.md  +  references/{dtcg-tokens.md, design-validation-loop.md}
    kargha-validate/  SKILL.md  +  scripts/{serve_design.py, capture_view.py}
```

Each skill is a self-contained canonical Agent Skill — a directory whose `SKILL.md` carries the frontmatter (`name` + `description`) and the workflow. Heavy, conditionally-needed material (the ticket template, the DTCG token machinery) lives one level deep in `references/` and is loaded on demand, so the always-resident `SKILL.md` bodies stay lean.

## Install

kargha ships as a self-contained Claude Code plugin + marketplace (the `.claude-plugin/` manifests). Add the marketplace and install — from the **public GitHub** repo, which needs no auth:

```bash
/plugin marketplace add https://github.com/TejGandham/kargha.git
/plugin install kargha@kargha
```

This registers all three skills, namespaced under the plugin: `kargha:kargha-plan`, `kargha:kargha-build`, `kargha:kargha-validate`. (Pre-1.0: the names keep the `kargha-` prefix; they may shorten to `kargha:plan` / `:build` / `:validate` at a stable release.)

## The three skills

### kargha-plan

**Decomposes a design export into vertical-slice tickets for frontend implementation.** Its core output is the **component-to-library mapping** — for every design component, whether it maps to a project library component (with which props/variants), needs a thin wrapper, or must be built custom. Every ticket also carries an icon mapping, a design-token map, a route/layout plan, a data-layer plan, acceptance criteria, and (for DTCG projects) a token-changes authorization.

- **In:** a path to a design HTML export, a natural-language scope ("the whole thing" / "just the feed page"), and a ticket destination.
- **Out:** one ticket per slice — emitted to a ticketing system (JIRA, Linear, GitHub Issues, …) **or** as Markdown/JSON files (`NN-slug.md` + an `index.md` manifest). The canonical ticket layout lives in [`kargha-plan/references/ticket-template.md`](skills/kargha-plan/references/ticket-template.md).
- **Invoke with:** "plan the frontend for this design", "break this design into tickets", "create frontend tickets from this design", "slice this design into implementation work".

### kargha-build

**Implements one ticket end-to-end in an isolated git worktree**, from a pickup-eligible status to an open PR. It gates on ticket status/assignee, picks the ticket up, spins up a per-ticket worktree, implements the components against the project's conventions, runs lint/test, optionally validates data-layer conformance (Phase 5b) and DTCG token conformance (Phase 5), starts the dev server, runs the design-validation loop (Phase 7, up to 3 rounds), and opens a PR.

- **In:** a single ticket authored by `kargha-plan` — a ticketing-system id/key/URL **or** a Markdown/JSON ticket file. The ticket must contain a **Design Reference** section.
- **Out:** implemented code in a worktree, an open PR (or pushed branch + compare URL on hosts without PRs), and the ticket moved to a review status.
- **Invoke with:** "build this frontend ticket", "implement the frontend ticket", "pick up the frontend work", or any request to implement a ticket with a Design Reference section.

### kargha-validate

**Compares one running view against its design prototype** and reports structured discrepancies — it never fixes them; the caller decides what to act on. It uses two fresh-context workers when available: a **capture** subagent OR host worker (drives `playwright-cli` to screenshot + snapshot both the design prototype and the live app) and a **comparison** subagent OR host worker (sees only the captured data, emits the report).

- **In:** the design HTML file, the app base URL + route, design/app navigation instructions, optional viewport and focus areas.
- **Out:** a structured report — `STATUS` (match/partial/mismatch), severity-rated `DISCREPANCIES` across layout/color/typography/spacing/components/hierarchy, `TOKEN_DRIFT`, and missing/extra elements.
- **Invoke with:** "validate against the design", "compare implementation to design", "check design fidelity", "does this match the design".
- One view per invocation — the calling pipeline loops for multiple views.

## How they connect (the ticket contract)

`kargha-plan` emits, and `kargha-build` extracts and implements, a fixed set of ticket sections. The contract lives in the **emitted ticket** (build parses the ticket payload, not plan's `SKILL.md`), so the section headings and line formats in [`ticket-template.md`](skills/kargha-plan/references/ticket-template.md) are byte-significant — em-dashes, backticks, and ` / ` separators included.

| Ticket section | Produced by `kargha-plan` | Consumed by `kargha-build` |
|-|-|-|
| Design Reference (design file, view, navigation) | ticket template | Gate 3 + Phase 1 extraction |
| Component-to-Library Map | Phase 4a | Phase 4d implementation |
| Icon Mapping / Missing Icons | Phase 4a.1 | Phase 4d (exact imports) |
| Design Token Map (+ `00-token-map.md`) | Phase 4a.2 | existing-token decisions |
| Token Changes (DTCG) | Phase 4a.2 / template | Phase 4d tier-gated creation |
| Route / Layout (+ Served URL) | ticket template | `APP_ROUTE`, health check |
| Acceptance Criteria | ticket template | done-definition |
| Validation parameters | ticket template | Phase 7 design-validation |

`kargha-build` then invokes `kargha-validate` (Phase 7a) with the design file, app base URL, app route, navigation, and viewport, and parses its `STATUS`/`DISCREPANCIES`/`TOKEN_DRIFT` report — fixing critical/major issues and re-running until **zero critical and zero major** remain (the same bar all three skills agree on), capped at 3 rounds.

## Cross-cutting concepts

- **Stack-agnostic.** No skill assumes a specific component library, icon set, theme system, framework, data layer, ticketing system, or repo layout. Concrete tools shown in the docs (Style Dictionary, Next.js, JIRA, `playwright-cli`, `localhost:3000`, …) are **examples**, resolved per project.
- **Design-input contract.** The design side assumes a **Claude Design OR runtime-JSX HTML export** (`.jsx` sources or an inline `text/babel` script, `useState` view switching, inline styles). `kargha-plan` gates on this format; other export formats are out of scope.
- **W3C DTCG design tokens.** When the token system is in [W3C DTCG](https://www.designtokens.org/) format (2025.10), the skills map design tokens to the **consumable (semantic) tier only**, derive a token manifest from the generated CSS (never re-implementing DTCG resolution), and let `kargha-plan` author a **Token Changes** section that pre-authorizes `kargha-build` to create additive semantic tokens autonomously (a deterministic Phase 5 check enforces no primitive-tier consumption and no hand-edited generated artifacts). The DTCG-specific machinery lives in each skill's `references/dtcg-tokens.md`. Non-DTCG token systems use the generic "no hardcoded values that duplicate tokens" rule.
- **Theme contexts.** Alternate contexts (e.g. dark mode) get a cheap render/empty-variable smoke check by default; a full second design-validation loop is opt-in and requires a switchable design prototype, reached through navigation.

## Requirements

- **`kargha-build`** needs `git` (uses per-ticket worktrees), the project's package manager + toolchain (lint/test/build/dev), and access to the ticket source (a ticketing MCP/CLI/REST, or a ticket file).
- **`kargha-validate`** needs [`uv`](https://docs.astral.sh/uv/), [`playwright-cli`](https://playwright.dev) (`npm install -g @playwright/cli@latest`), and a browser (Chromium). Its bundled PEP 723 scripts serve the design HTML and capture views; the running app must already be up because the caller owns the dev server's lifecycle.
- **`kargha-plan`** needs read access to the design export and the project repo; it writes only tickets/files, never implementation code.

## Usage

Invoke a skill by pointing it at its input — e.g. "plan the frontend for `./design-export/`", then "build ticket `03-signal-feed.md`". `kargha-build` calls `kargha-validate` automatically when a design-validation tool is configured; you can also run `kargha-validate` standalone to spot-check fidelity. See each skill's `SKILL.md` for the full phase-by-phase workflow, and its `references/` for the ticket template, the DTCG token machinery, and the design-validation loop procedure.
