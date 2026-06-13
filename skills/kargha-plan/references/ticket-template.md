# Ticket template (`kargha-plan` → `kargha-build` contract)

`kargha-plan` Phase 5 copies this template into each ticket and fills the placeholders (`<frontend-app>`, `<component-lib>`, … refer to the resolved Project configuration).

> **This is a parse contract.** `kargha-build` parses the *emitted ticket*, matching these section headings and line formats. Emit them **exactly as written** — including the em-dashes (`—`), backticks, the right-arrow (`→`), and the ` / ` separators in the Token Changes table. Do not rename sections, reword labels, or normalize the punctuation.

## Contents
- Context
- Design Reference
- Component-to-Library Map
- Icon Mapping
  - Missing Icons
- Component Plan
  - <ComponentName> (custom | wrapper)
- Route / Layout
- Data Layer
- Design Token Map
- Token Changes
  - Foundation Setup Checklist
- Acceptance Criteria
- Implementation Notes
- Files to Create
- Verification
  - Design Validation Loop (if available)

---

## Context

<1-2 sentences: what this ticket implements and why>

**Parent / group:** <epic key | project | group label | none>
**Depends on:** [ticket refs] | none

## Design Reference

- **Design file:** `<design_file>` (the absolute path resolved in Phase 0a — same value here and in the Validation parameters below)
- **View:** `<page id>` — navigate via sidebar > <label>  *(foundation/setup tickets: `none` — setup ticket, design validation not applicable)*
- **Components in scope:** <list>
- **Screenshots:** `<path-to-screenshots/>` (if a screenshots/assets dir was found in Phase 0a; omit otherwise)

## Component-to-Library Map

| Design Component | Library Mapping                                  | Notes                         |
| ---------------- | ------------------------------------------------ | ----------------------------- |
| Button           | `Button` (variant="primary", color="brand")      | Direct match                  |
| Badge            | `Badge` (variant="pill", size="sm")              | Direct match                  |
| SignalCard       | **Custom** — composes `Badge`, `Avatar`, `Tag`   | Domain-specific               |
| FilterBar        | **Composite** — `Tabs` + `Badge` buttons         | Tab navigation + filter chips |
| ...              | ...                                              | ...                           |

## Icon Mapping

(Only for tickets that use icons — per 4a.1. Omit the section if the ticket uses none.)

| Design Icon | Source | Import | Notes |
| ----------- | ------ | ------ | ----- |
| `radar`     | primary | `Signal01` from `<icon-lib>/…/Signal01` | Closest match |
| `sparkle`   | fallback | `{ Sparkles }` from `<fallback-icon-lib>` | No primary match |

### Missing Icons

(Only when one or more design icons have no library match — per 4a.1 step 4. Omit when all icons mapped.)

- `drag-handle` — **custom SVG needed** (no configured library has it); used in <where/why>

## Component Plan

### <ComponentName> (custom | wrapper)

- **File:** `<frontend-app>/<path>/<ComponentName>.<ext>`
- **Type:** presentational | data-owning | view-controller
- **Props:**
    ```typescript
    interface <ComponentName>Props {
      // key props with types
    }
    ```
- **Styles:** `<ComponentName>` styles per the project's styling approach
- **Library components used:** `Badge`, `Avatar`, `Tag` (list library components composed within)
- **Data ownership** (if applicable): the operation/fragment this component owns, e.g.
    ```graphql
    fragment <ComponentName>Fragment on <Type> {
      <fields>
    }
    ```
    (or the REST fields it reads, if not GraphQL)
- **Key behaviors:** <hover states, click handlers, state changes>
- **Design reference:** `<file>:<line-range>` in design source

(Omit component plan entries for pure library usage — the mapping table is sufficient.)

## Route / Layout

(Adapt to the project's router — file-based or config-based. *Foundation/setup tickets with `View: none`: omit this section — there is no route; record any root-layout/provider wiring under Foundation Setup Checklist instead.*)

- **Route:** `<how this view's route is declared>`
- **Served URL:** `</exact/live/path>` — the concrete URL path the running app serves this view at (the build skill uses this verbatim for its dev-server health check and the validator; it falls back to deriving the URL from the route declaration only if this line is absent, which is error-prone — always fill it)
- **Layout changes:** <shared layout modifications if any>
- **Navigation:** <how users reach this view in the running app>

## Data Layer

(*Foundation/setup tickets: omit unless the setup itself touches the data layer, e.g. bootstrapping the API client.*)

- **Operation:** the query/endpoint this view calls, e.g.
    ```graphql
    query <OperationName> {
      <fields with fragment spreads>
    }
    ```
    (or the REST endpoint + response shape, if not GraphQL)
- **Fetch policy / caching:** <default, or note any deviation with reason>
- **Schema gaps:** <list fields needed but missing, with strategy: stub/blocked>
- **Mock handlers:** <list mock handlers needed for testing>

## Design Token Map

(Full map in the foundation ticket; if 4c skipped the foundation, in the first page ticket. In Mode B the full map also lives in a sibling `00-token-map.md`, and each page ticket carries the slim subset of tokens it uses plus a pointer to that file — so every ticket stays self-contained.)

The project's token/theme system is the **source of truth** for all design tokens. Do NOT copy the design's tokens stylesheet into the app — use project tokens via the library's props and the project's variables/classes. For a tiered (DTCG) system, map to the **consumable tier only** (typically semantic, never primitives — the Tier column makes this explicit).

| Design Token             | Project Token (tier)              | Usage                                      |
| ------------------------ | --------------------------------- | ------------------------------------------ |
| `--blue-700` / `#0E5ECF` | `semantic.color.accent` (semantic) | Color prop `color="brand"` or `var(--accent)` in CSS |
| `--gray-500` / `#667085` | `semantic.color.text-2` (semantic) | `var(--text-2)`                            |
| ...                      | ...                               | ...                                        |

**Tokens with no consumable-tier match:** <list any — each becomes a Token Changes entry below or a resolved user decision>

## Token Changes

(DTCG / tiered token systems only. Lists the tokens this ticket needs that **don't yet exist in the consumable tier** — the Design Token Map above covers tokens that already exist; this section covers tokens to **add**. It is the implementation skill's authorization to create them. **Omit the whole section** when every design token maps to an existing consumable-tier token — the common case. Carry the rows in every ticket that consumes the token; don't make a page ticket depend on reading the foundation ticket.)

One row per proposed token (example rows shown — the project's name-resolution rule here is a vendor `cssName` extension, so paths map to `--<name>`):

| Token (path → var) | Op | Value — base / <context> | Source file(s) | Covers | Auth |
|-|-|-|-|-|-|
| `semantic.color.surface-info` → `--surface-info` | `add` | `#eef4fb` / dark `#1c2430` (literal) | `semantic.json`, `semantic.dark.json` | InfoBanner background | autonomous |
| `semantic.color.accent-quiet` → `--accent-quiet` | `add` | `{color.burgundy.600}` (alias; one value across contexts — safe because `burgundy.600` isn't re-pointed in dark) | `semantic.json` | muted accent on FilterChip | autonomous |
| `color.burgundy.950` → `--p950` | `add-primitive` | `#2a0410` (net-new stop beyond the existing `…900`) | `primitives.json` | darkest ramp stop the design introduces | **requires build-time confirmation** |

Column legend:

- **Token (path → var):** the DTCG token path and the emitted CSS variable, named per the project's name-resolution rule (vendor `$extensions` cssName, else path-derived). The name must be unambiguous and must not collide with an existing token — an ambiguous or duplicate name forces a build-time ask instead of an autonomous add.
- **Op (this encodes tier):** `add` = an additive **semantic** token — the *only* case the implementation skill applies autonomously; `add-primitive` = a new primitive ramp entry; `mutate` = change an existing token's `$value`. `add-primitive` and `mutate` are never autonomous — put **"requires build-time confirmation"** in their Auth cell, or land them in the foundation ticket under explicit user approval recorded here.
- **Value — base / <context>:** an **alias** to a named existing primitive (`{group.token}`) when one matches the design value, else a **literal**. Give a value **per theme context the design defines**: omitting a context override makes that context silently inherit the **base** value (the project's per-context file, e.g. `semantic.dark.json`, is where overrides live), so specify the dark (etc.) value whenever the design's differs — alias or literal alike. An alias is context-stable only when the aliased token isn't itself re-pointed in that context; if it is (e.g. a primitive inside a transitional-debt dark block), give an explicit override.
- **Source file(s):** the file(s) in `<token-source-dir>` to edit — the semantic-tier file plus the per-context file (e.g. `semantic.json` + `semantic.dark.json`) for any context override. The implementation skill edits these and runs `<token-build-command>`; it never hand-edits `<generated-token-artifacts>`.
- **Covers:** which design values / components need the token (the "why").
- **Auth:** `autonomous` (an `add` row whose value the design supplies — the implementation skill applies it without asking) or `requires build-time confirmation`. A row appearing here at all **is** the recorded plan-time user decision from 4a.2 step 5; the Auth cell says whether the build skill may act on it unattended.

Never propose touching documented transitional-debt blocks — flag those to the user instead.

### Foundation Setup Checklist

(Foundation ticket only — omit in page tickets, and in the "already integrated" first-ticket case from 4c.)

- [ ] Bootstrap the component library provider with the project theme/token resolver
- [ ] Import the library's base stylesheet (if required)
- [ ] Set up icon library imports
- [ ] Configure fonts
- [ ] Verify project tokens cover the design's color/spacing/radius/shadow needs
- [ ] Add supplemental tokens ONLY for design values with no project equivalent (document why). **DTCG:** edit the DTCG JSON in `<token-source-dir>` (the right tier file, plus the per-context file for any context override) and run `<token-build-command>`; the generated artifacts are read-only

## Acceptance Criteria

- [ ] <Specific, verifiable criterion per component>
- [ ] Components render matching design layout
- [ ] Library components used wherever mapped (see Component-to-Library Map)
- [ ] Interactive states work (hover, active, focus, disabled)
- [ ] Accessible: semantic HTML, keyboard navigation, ARIA where needed
- [ ] Co-located test files exist with smoke tests
- [ ] `<lint command>` passes
- [ ] `<test command>` passes
- [ ] (DTCG token systems) Token-conformance passes: no primitive-tier variable consumption in new code, no hand-edits to generated token artifacts, no hardcoded values duplicating tokens
- [ ] Design validation passes (match, or partial with only cosmetic/minor issues) — see Verification  *(omit for foundation/setup tickets with `View: none` — design validation is N/A)*

## Implementation Notes

- Follow the project's component conventions <cite rules file if present>
- Follow the project's data-layer conventions <cite rules file if present>
- Use `<component-lib>` components per the mapping table — do not rebuild primitives the library provides
- **All styling must reference project theme tokens** — use library props (`color="brand"`, `spacing="md"`, `radius="sm"`) for library components and project token variables/classes in custom styles. Never hardcode hex colors, px spacing, or px radii that duplicate project token values. For a tiered (DTCG) system, consume the **semantic tier only** — primitives are deny-listed for new code, and any needed-but-missing token is in this ticket's Token Changes section.
- Design uses inline styles — implementation must use the project's styling approach (or library props)
- Design uses hardcoded mock data — implementation must use the project's data layer
- Design uses client-side `useState` navigation — implementation uses the project's router
- **If a design-validation tool/skill is available:** run it in a loop (up to 3 rounds) to verify fidelity against the design prototype — see Verification

## Files to Create

- `<frontend-app>/<path>/<Component>.<ext>`
- `<frontend-app>/<path>/<Component>` styles
- `<frontend-app>/<path>/<Component>` test
- (list all files)

## Verification

```bash
<lint command>
<test command>
<build command>
```

### Design Validation Loop (if available)

**Foundation/setup tickets (`View: none`): this whole loop is N/A — omit it.** There is no view to validate; the ticket's done-definition is lint/test + token-conformance + PR. For all other tickets:

If the environment provides a design-validation tool/skill, run it to compare the running app against the design prototype. Treat it as a **required loop** — up to 3 rounds. Each round fixes `critical` then `major` discrepancies; the exit bar is **zero critical and zero major** (a `match`, or a `partial` with only minor/cosmetic left). Minor and cosmetic issues are acceptable — do not burn rounds chasing them; list any residuals in the PR body.

1. **Round 1:** Validate with the design file, app route, and navigation. Fix critical, then major, discrepancies.
2. **Round 2:** Re-validate after fixes. Fix any remaining critical/major.
3. **Round 3:** Final pass. Exit when zero critical and zero major remain; document any minor/cosmetic residuals as follow-up.

Stop early once zero critical and zero major remain. Do not exceed 3 rounds — if major issues persist, document them as follow-up work. If no validation tool exists, this becomes a manual visual checklist against the design file. (This matches the implementation skill's exit criteria, so a build agent executing the ticket and the acceptance criterion above agree on the bar.)

**Validation parameters for this ticket:**

- Design file: `<design_file>` (same absolute path as the Design Reference)
- App route: `<route>`
- Design navigation: `<instructions to reach this view in the design prototype>`
- App navigation: `<steps beyond loading the route to reach the view in the running app, e.g. "click the first row to open the detail panel" — REQUIRED for slideout/modal/detail sub-view tickets; omit when the route's initial render is the view>`
- Viewport: `<WxH, e.g. 1440x900; omit for the tool's default>`
- Theme context(s): `<base only | base + <context> (smoke) | base + <context> (full)>`. Emit a non-default context only when the token system declares one. **smoke** (the safe default) → the implementation skill renders the app in that context and checks no theme variable resolves empty; needs nothing from the design side. **full** → a complete second design-validation loop, only meaningful when the *design prototype itself* supports the context (per Phase 1's theme-switching report). The Design/App navigation lines above stay **base-only** (so the base loop really validates base); supply the context switch separately in the next line so the build applies it only for the second loop.
- Context switch (only for a `full` alternate context): `<the extra steps that switch BOTH the design prototype and the app into the alternate context, e.g. "click the theme toggle in the top bar">`. The validator has no theme input of its own — it drives the switch through this navigation, appended to the base navigation for the second loop only.
- Focus areas: `<relevant dimensions if scoped, otherwise omit>`