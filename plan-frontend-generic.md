---
name: plan-frontend-generic
description: Analyze a design HTML export (e.g., from Claude Design) and decompose it into vertical-slice tickets for frontend implementation. Reads design files (JSX components, CSS tokens, page navigation, mock data, screenshots), maps design components to the project's component library, design tokens to the project's theme/token system, design icons to the project's icon libraries, and cross-references the project's data layer (mocked or stubbed, GraphQL schema, REST types, etc.). Emits self-contained tickets into any ticketing system (JIRA, Linear, GitHub Issues,...) OR as plain Markdown/JSON ticket files. Each ticket includes a component-to-library mapping so implementers know which library components to use. Invoke when the user has a design prototype and wants implementation tickets — trigger phrases include "plan the frontend for this design", "break this design into tickets", "create frontend tickets from this design", "plan frontend from the HTML export", "slice this design into implementation work", or any request that combines a design HTML path with ticket/planning intent.
---



Turn a design export (e.g., a Claude Design HTML export) into vertical-slice tickets for frontend implementation. Each ticket is self-contained and independently implementable by an implementation agent/skill or a human.

The skill's core value is the **component-to-library mapping** — determining which design components map to the project's component library (and with which props/variants), which need thin wrappers, and which must be built custom. This mapping appears in every ticket.

## How this skill adapts to your project

This skill is stack-agnostic. It does **not** assume a specific component library, icon set, theme system, ticketing system, or repo layout. Instead, it resolves a small set of project settings up front (detect → ask), then maps the design against whatever it finds:

- **Component library:** any library, several libraries, or none. With no library, every component is "custom" — the mapping still tells you exactly what to build.
- **Icon libraries:** a primary source, optional fallbacks, or none.
- **Token/theme system:** a theme object, CSS custom properties, a design-tokens file, a utility-class framework, or plain CSS.
- **Data layer:** GraphQL schema, REST/OpenAPI types, generated TS types, or none (stub until available).
- **Ticket destination:** any ticketing system, or plain Markdown/JSON files written to a directory.

Where this document shows a concrete tool or library, treat it as an **example**, not a requirement.

## Project configuration (resolve once, up front)

Resolve each setting in this order: **explicit user input → detect from the repo → ask the user** (batch all unknowns into a single `AskUserQuestion`). Do not prompt for things you can detect.

| Setting | What it is | How to resolve |
| ------- | ---------- | -------------- |
| **Frontend app dir** | Where components/routes/styles live | Detect the package that depends on the UI framework (e.g., a `package.json` with `react`/`next`/`vue`/`svelte`); in a monorepo pick the app the design targets; else ask |
| **Component library** | UI-primitive library/libraries the project uses (0..n) | Detect from `package.json` deps + existing import statements; may legitimately be none |
| **Icon libraries** | Primary icon source + optional fallbacks | Detect from deps/imports; may be none |
| **Token/theme system** | The source of truth for colors / spacing / radius / typography / shadows | Detect: a theme object, CSS custom properties in a global stylesheet, a design-tokens file, a utility-class config, or plain CSS |
| **Data layer** | The API the UI reads from | Detect: a GraphQL schema file, OpenAPI/REST types, generated TS client types; may be none |
| **Project rules** | Component / data conventions the repo documents | Detect: contributor docs, lint configs, or rules files; cite them in tickets if present |
| **Toolchain commands** | lint / test / build invocations | Detect from package scripts and the repo's task runner (npm/pnpm/yarn scripts, Nx, Turborepo, Make, etc.) |
| **Ticket destination** | Where tickets are emitted | Ask: a ticketing system (which one + how to call it) **or** plain files (output dir + `md`/`json`) — see Phase 0b |
| **Design-validation** | Optional fidelity-check tool/skill | Use if the environment provides one (e.g., a skill that screenshots the running app and diffs it against the design); otherwise the design-validation loop becomes a manual checklist |

Record the resolved values — every later phase references them. In this document, placeholders like `<frontend-app>`, `<component-lib>`, `<icon-lib>`, `<theme>`, `<schema>`, `<lint command>` refer to these resolved settings.

## Workflow

### Phase 0 — Collect and validate inputs (hard gates)

Required inputs. If any is missing, use `AskUserQuestion` once to collect all missing values simultaneously (combine with any unresolved Project configuration questions).

**0a. Design path.** The user must provide a path to the design HTML or its parent directory. Resolve it:

```bash
design_path="<caller-provided-path>"

if [ -d "$design_path" ]; then
  # Prefer standalone variant (no CDN dependency)
  design_file=$(find "$design_path" -maxdepth 2 -name '*standalone*.html' | head -1)
  # Fall back to any HTML
  if [ -z "$design_file" ]; then
    design_file=$(find "$design_path" -maxdepth 2 -name '*.html' ! -name '*print*' | head -1)
  fi
else
  design_file="$design_path"
fi
```

If no HTML found, bail with: "No design HTML files found at `<path>`. Provide a path to a design HTML export."

Also check for sibling source files — these are easier to parse than the monolithic HTML:

- Individual `.jsx` files (e.g., `contacts-page.jsx`, `settings-pages.jsx`) — logically grouped component modules
- `combined.jsx` — all modules merged with `// ===== src/<file>.jsx =====` section markers
- An extracted tokens stylesheet (e.g., `colors_and_type.css`)
- `src/` subdirectory — split-out component files

Prefer reading individual `.jsx` files when present. Fall back to `combined.jsx`, then the inline `<script type="text/babel">` in the standalone HTML.

**0b. Ticket destination.** Determine where tickets go before doing the work — it shapes Phase 7. Two modes:

- **Ticketing system.** Identify the system (JIRA, Linear, GitHub Issues, GitLab, Azure Boards, …) and how to call it (an MCP server, a CLI like `gh`/`tea`, or a REST API). Resolve the container the tickets attach to (epic / project / milestone / parent issue) and validate it:
    1. Fetch the container by id/key and confirm it exists.
    2. Confirm its type matches what the system expects for grouping (e.g., an Epic in JIRA, a Project/Milestone elsewhere). If it's the wrong type, tell the user and ask again.
    3. Sanity-check thematic fit — does the container plausibly cover frontend implementation work? Surface a mismatch briefly.
    4. If the user doesn't know the container id, query the system for open epics/projects and offer a short list.
- **Plain files.** Get an output directory and a format (`md` default, or `json`). No container is required, but accept an optional group/parent label to stamp on each ticket. Confirm the directory (create it if absent). See Phase 7, Mode B, for the file format.

**0c. Implementation prompt.** Natural-language scope describing what to implement from the design. Can be broad ("the whole thing") or narrow ("just the feed page and signal cards"). May include context about work already in progress ("the sidebar is being built in <ticket-ref>").

---

### Phase 1 — Analyze design structure (Explore subagent)

Spawn an **Explore subagent** to read the design HTML and all sibling source files.

**Subagent brief:**

Read the design export at `<design_path>`. Start with the individual `.jsx` files if present, then `combined.jsx`, then fall back to the `<script type="text/babel">` block in the HTML. Also read the extracted tokens stylesheet if it exists.

Report:

1. **Component inventory.** Every named component (function or const with PascalCase name):
    - Name
    - Props (list prop names and inferred types from usage)
    - Approximate complexity (trivial <30 lines, moderate 30-100, complex >100)
    - Dependencies on other design components (which components it renders)
    - Local state (`useState` calls — what state and what drives it)

2. **Component hierarchy tree.** Containment from the root component down:

    ```
    App > Sidebar > [NavItem, UserMenu, Avatar]
    App > Feed > [SignalCard > [Badge, Avatar, Icon], FilterChip]
    ```

3. **Page/view inventory.** Every distinct "page" (typically from `page === '<id>'` conditionals or a client-side router):
    - Page ID (e.g., 'home', 'feed', 'projects')
    - Entry component name (e.g., `HomePage`, `Feed`, `ProjectsPipelinePage`)
    - Sub-views (e.g., projects has both pipeline and detail views)
    - Components used exclusively by this page vs shared across pages

4. **Design token summary.** CSS custom properties from `:root` and the tokens stylesheet, organized:
    - Colors: how many, naming convention (e.g., `--gray-25` through `--gray-950`)
    - Spacing: scale values
    - Typography: font families, size scale, weight scale
    - Radius: scale values
    - Shadows: elevation levels
    - Semantic tokens (e.g., `--bg-primary`, `--fg-primary`, `--border-primary`)

5. **Mock data shapes.** For each top-level data constant (e.g., SIGNALS, PROJECTS, TEAM, CONTACTS), report the TypeScript-like shape of one item showing all fields and inferred types.

6. **Navigation structure.** How pages relate: sidebar items, breadcrumbs, drill-down patterns (list -> detail), modal/dialog/slideout patterns.

Report with exact `file:line` citations.

---

### Phase 2 — Inventory codebase + component library (Explore subagent, parallel with Phase 1)

Spawn a second **Explore subagent** in parallel.

**Subagent brief:**

Survey the frontend app at `<frontend-app>` and the project's component library/libraries (`<component-lib>`, if any).

**Part A — Frontend app:**

1. **Existing components.** List every component file under the app's components/routes directories, noting exports and purpose.
2. **Existing routes.** Map the router structure to routes. Whatever the router is (file-based like Next.js App Router, or config-based like React Router / TanStack Router), report how routes are declared and where layouts / loading / error boundaries live.
3. **Styles.** Stylesheet files, global tokens, font stack, and the styling approach (CSS Modules, CSS-in-JS, utility classes, etc.).
4. **Data layer.** Existing API access — GraphQL fragments/queries/mutations and codegen setup, REST clients, or generated types. Note where operations are defined and any codegen config for contracts e.g openapi.
5. **Client state.** Client-state stores (Redux, Zustand, Jotai, signals, context, …) and what they manage.
6. **Tests.** Test files, test-runner config, network-mocking setup.
7. **Library integration.** Is `<component-lib>` already imported anywhere? Is its provider/theme bootstrapped (e.g., a theme provider in the root layout or a providers file)? If there is no component library, note the project's local primitives instead.

**Part B — Component library catalog** (skip if the project has no component library; instead catalog any local design-system primitives):

Read the library from its installed location (e.g., `node_modules/<component-lib>/`). Check the main export/type entry (e.g., `dist/index.d.ts` / `dist/index.js`) and any components directory.

For each exported component, report:

- Name
- Category (the library's own grouping if it has one: atoms / molecules / organisms / blueprint / primitives, etc.)
- Key props and their types (especially `variant`, `size`, `color`, `type` enums)
- What it renders (button, card, input, badge, avatar, tooltip, table, menu, etc.)

Also catalog:

- **Icon libraries.** For each configured icon source (primary first, then fallbacks), list ALL available icon exports and the import pattern. Report the full set of icon component names (e.g., `AddBold`, `Bell01`, `Search01`, …) so Phase 4 can map design icons to concrete imports. Note each library's props convention (size, color, standard SVG props).
- **Theme/token inventory (critical).** Read however the project exposes its tokens — a theme object and its types, a CSS-variable map, a design-tokens file, or a utility-class config — and report the complete inventory:
    - **Colors:** every named color (and shade scale, e.g., 0–11) with hex values where readable
    - **Spacing:** the full spacing scale (named keys and px values)
    - **Radius:** the full radius scale
    - **Font sizes & line heights:** all named sizes (e.g., `text-xs`, `text-sm`, `display-xs`)
    - **Shadows:** named shadow/elevation levels
    - **Font family:** body and heading font families
    - **Any custom/semantic tokens** the system exposes (and how they resolve to CSS variables or classes)

    This inventory is essential — Phase 4 uses it to map design tokens to project tokens and flag gaps.

- README / setup docs for integration instructions.

Report with `file:line` citations.

---

### Phase 3 — Map data needs to the data layer (Explore subagent, parallel with Phases 1-2)

Spawn a third **Explore subagent** in parallel. (If the project has no data layer yet, this phase instead documents the data each design entity needs, to be stubbed until a backend exists.)

**Subagent brief:**

Read the project's data layer — the GraphQL schema (e.g., a `schema.graphql`), OpenAPI/REST spec, or generated TS types — at `<schema>`. Also read the design's mock data structures from the design files at `<design_path>` (top-level `const` arrays like SIGNALS, PROJECTS, TEAM, CONTACTS, etc.). If no data layer exists ask get user permission to generate a canonical mock layer based on openapi or other user suggested standard.

Report:

1. **Type coverage.** For each design data entity, map to the data layer's types:
    - Fields in both the design mock and the schema (matched)
    - Fields in the design mock with no schema equivalent (gaps)
    - Schema fields not used by any design component (unused, for awareness)

2. **Operation mapping.** For each page's data needs, which queries/endpoints serve them (e.g., a `searchSignals` query or a `GET /signals` endpoint for the feed; a `me`/`/me` for the current user).

3. **Schema gaps.** Entities in the design with NO corresponding type. List each with:
    - Entity name
    - Which pages depend on it
    - Approximate field count

4. **Fetch-boundary planning.** Based on the component hierarchy, which components own data fetching / fragments (those directly consuming entity data) vs purely presentational. If the stack uses GraphQL fragment colocation, note which components would own fragments.

Report with `file:line` citations.

---

### Phase 4 — Synthesize and scope (main thread)

Do this yourself — do not delegate. Read all three subagent reports, then:

#### 4a. Component-to-library mapping (critical step)

For every design component, determine if it maps to a `<component-lib>` component. Classify each:

| Classification        | Meaning                                                  | Ticket impact                                                     |
| --------------------- | -------------------------------------------------------- | ----------------------------------------------------------------- |
| **Library match**     | Maps directly to a library component                     | Use library component with props/variants — no custom code needed |
| **Library + wrapper** | Library covers 80%+ but needs a thin domain wrapper      | Wrapper component that composes the library component             |
| **Custom**            | No library equivalent (or no library at all)             | Full custom component plan                                        |
| **Composite**         | Assembled from multiple library components               | Document the composition                                          |

This mapping is the **core output** of the skill. It must appear in every ticket description.

When mapping, look for these common UI-primitive correspondences in `<component-lib>` (names vary by library — match by role, not by name):

- Design buttons → the library's `Button` (check variant: primary/secondary/tertiary, color)
- Design badges/tags → `Badge` / `Tag` / `Chip` (check style/type variants)
- Design avatars → `Avatar` / `AvatarGroup`
- Design tooltips → `Tooltip`
- Design inputs → `TextInput` / `Textarea` / `InputField`
- Design checkboxes/toggles → `Checkbox` / `Switch` / `Toggle`
- Design tabs → `Tabs` (horizontal/vertical)
- Design tables/lists → `Table` / `DataTable`
- Design dropdowns/selects → `Select` / `Combobox` / `MultiSelect`
- Design menus → `Menu`
- Design breadcrumbs → `Breadcrumbs`
- Design pagination → `Pagination`
- Design accordions → `Accordion`
- Design alerts/banners → `Alert` / `Banner` / `Notification`
- Design empty states → `EmptyState`
- Design slide-outs/drawers → `Drawer` / `SlideOut`
- Design command bar/search → `Spotlight` / `CommandBar` / `SearchBar`
- Design progress indicators → `Progress` / `ProgressRing` / `Stepper`
- Design date pickers → `DatePicker`
- Design section headers → `SectionHeader` / `Title`
- Design loading states → `Loader` / `Skeleton`
- Design icons → the project's icon libraries — see icon mapping rules below

If the project has **no** component library, every primitive is **Custom** — record that, and the mapping becomes a build list.

#### 4a.1. Icon mapping (required step)

Every icon used in the design must be mapped to a concrete icon import. Designs usually reference icons by string name (e.g., `'radar'`, `'target'`, `'bell'`) via an `Icon` component — resolve each to a specific React component import from one of the configured icon libraries.

**Priority order:**

1. **Primary icon library** (`<icon-lib>`) — check first.
2. **Fallback icon library** (if configured) — use when the primary has no suitable match.
3. **Custom SVG** — only when no configured library has a match.

**Process:**

1. Inventory every unique icon name used in the design (from `<Icon name="...">` calls and any inline SVGs).
2. For each design icon, check the primary library first; if no suitable match, check each fallback in order.
3. Include an **Icon Mapping** table in every ticket that uses icons. The table must include a **Source** column:

| Design Icon   | Source         | Import                                          | Notes                                       |
| ------------- | -------------- | ----------------------------------------------- | ------------------------------------------- |
| `radar`       | primary        | `Signal01` from `<icon-lib>/…/Signal01`         | Closest match to radar/signal concept       |
| `bell`        | primary        | `Bell01` from `<icon-lib>/…/Bell01`             | Direct match                                |
| `sparkle`     | fallback       | `{ Sparkles }` from `<fallback-icon-lib>`       | No primary match; fallback has Sparkles     |
| `drag-handle` | **Custom SVG** | —                                               | No configured library has a drag-handle     |

4. **When no library has a match:** Flag it prominently in the ticket under a "Missing Icons" warning section. List each missing icon with its design usage context. The implementer will need a custom SVG. Do NOT silently plan custom SVGs without calling this out — the user needs to know which icons are gaps.

**Report to user:** After completing the icon mapping, surface any gaps via `AskUserQuestion` or in the Phase 6 summary table. Example: "10 of 12 design icons mapped (8 primary, 2 fallback); 2 require custom SVGs: `kanban`, `drag-handle`."

#### 4a.2. Design token mapping (required step)

**The project's token/theme system is the source of truth for all design tokens in the implementation.** The design's extracted CSS custom properties are a prototype artifact — they must NOT be copied verbatim into the app. Instead, every design token must be mapped back to a project token (theme key, CSS variable, or utility class).

**Process:**

1. Read the project's token inventory from Phase 2: colors (with shades), spacing scale, radius scale, font sizes, line heights, shadows, and any custom/semantic tokens.

2. For each design token category, map design values to project equivalents. The exact form depends on the token system (a theme prop, a CSS variable, or a utility class):

    | Category  | Design Token Example       | Project Token Equivalent (form varies by system)              | How to Use                                                  |
    | --------- | -------------------------- | ------------------------------------------------------------- | ---------------------------------------------------------- |
    | Color     | `var(--blue-700)`          | brand/primary shade 7 — theme prop, CSS var, or class          | Use the library color prop on components; the token in CSS  |
    | Spacing   | `var(--space-16)` / `16px` | the `md` (or equivalent) spacing token                         | Use the spacing prop or token, not a raw px value           |
    | Radius    | `8px` / `var(--radius-md)` | the `md` radius token                                          | Use the radius prop or token                                |
    | Font size | `14px`                     | the `sm` font-size token                                       | Use the typography prop or token                            |
    | Shadow    | `var(--shadow-sm)`         | the `sm` shadow token                                          | Use the shadow token                                        |

3. Classify each design token:

    | Classification   | Meaning                                                | Action                                                                                          |
    | ---------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
    | **Direct match** | Design value maps exactly to a project token           | Use the project token/prop — document the mapping                                               |
    | **Close match**  | Design value is within ~2px / ~1 shade of a token      | Use the closest token — note the minor difference in the ticket                                 |
    | **No match**     | Design value has no reasonable project equivalent      | **Flag to user** — ask whether to extend the theme, use a one-off override, or adjust the design |

4. Include a **Design Token Map** section in the foundation ticket (or the first ticket if no foundation ticket). For subsequent tickets, reference "use project theme tokens per the foundation ticket" rather than repeating the full map.

5. **When tokens don't map:** Use `AskUserQuestion` to surface mismatches before filing tickets. Example: "The design uses `--gray-25` (#FAFAFA) but the project's `gray` scale starts at shade 0 (#F8F9FA). Should we: (a) use shade 0 as-is, (b) extend the theme with a custom shade, or (c) accept the 1-value difference?"

**Key rule:** Custom components must reference project theme tokens (theme variables / utility classes) — never hardcoded hex values or px that duplicate what the token system already provides. This ensures theme changes propagate everywhere.

Components that are typically **custom** (no library equivalent):

- Page-level layouts and view controllers
- Domain-specific cards (SignalCard, ProjectCard, ContactCard)
- Domain-specific panels (detail views, slideout content)
- App chrome (Sidebar, TopBar) — though they may compose library primitives
- Charts, timelines, kanban boards
- Domain-specific filters and search

#### 4b. Scope the work

Parse the user's prompt against the page inventory from Phase 1. If ambiguous, use `AskUserQuestion`:

> "The design contains these pages: [list]. Which ones should I plan tickets for?"

Also check: does the prompt mention related work already in progress? If so, note dependencies.

#### 4c. Foundation ticket (conditional)

Check whether the component library + theme system are integrated in the frontend app (Phase 2, Part A, item 7):

- **Not integrated yet:** Create a foundation ticket covering: provider/theme bootstrap with the project token system, icon library setup, and font setup. The project token system IS the source of truth — do NOT copy the design's tokens stylesheet into the app. Configure the library's theme/token resolver so all tokens are available as the project's variables/props. Only add supplemental tokens for design values that have no project equivalent (and flag these per 4a.2). If the project has no component library, the foundation ticket instead establishes the token system, base primitives, and fonts the rest of the work builds on.
- **Already integrated:** Skip the foundation ticket. If design tokens differ from what's configured, note "verify/update existing tokens" in relevant page tickets.
- **Partially integrated:** Create a scoped foundation ticket covering only the missing pieces.

Also check whether shared app chrome (Sidebar, TopBar, root layout) exists. If not and the design includes it, either include it in the foundation ticket or as a separate app-shell ticket.

#### 4d. Data-layer gaps (always ask)

For every data entity in the design that has no equivalent in the data layer, surface it to the user via `AskUserQuestion`. Offer per-entity options:

- **Stub data:** Plan the frontend with mocked data, note as blocked until backend work is done
- **Exclude:** Skip pages that depend on this entity for now
- **Include backend work:** Add schema/resolver/endpoint work to the frontend ticket (makes it larger)

Group related entities in a single question when practical.

#### 4e. Vertical slice boundaries

Apply these heuristics:

1. **Foundation slice** (only if needed per 4c): library/theme setup + design tokens + app shell
2. **One slice per page/view** when the page has unique components
3. **Split list + detail** when both exist for an entity (e.g., Projects list vs Project detail)
4. **Complex reusable components** get their own slice when >100 lines in design and used across pages
5. **Cross-cutting features** (AI Assistant, Copilot, global search) are separate slices
6. **Dialogs/modals** that serve a single page bundle with that page

Each slice must be independently implementable and verifiable — it should render something meaningful on its own.

---

### Phase 5 — Draft ticket descriptions

For each identified vertical slice, draft a ticket description using this template. (`<frontend-app>`, `<component-lib>`, etc. refer to the resolved Project configuration.)

````markdown
## Context

<1-2 sentences: what this ticket implements and why>

**Parent / group:** <epic key | project | group label | none>
**Depends on:** [ticket refs] | none

## Design Reference

- **Design file:** `<path-to-html>`
- **View:** `<page id>` — navigate via sidebar > <label>
- **Components in scope:** <list>
- **Screenshots:** `<path-to-screenshots/>` (if available)

## Component-to-Library Map

| Design Component | Library Mapping                                  | Notes                         |
| ---------------- | ------------------------------------------------ | ----------------------------- |
| Button           | `Button` (variant="primary", color="brand")      | Direct match                  |
| Badge            | `Badge` (variant="pill", size="sm")              | Direct match                  |
| SignalCard       | **Custom** — composes `Badge`, `Avatar`, `Tag`   | Domain-specific               |
| FilterBar        | **Composite** — `Tabs` + `Badge` buttons         | Tab navigation + filter chips |
| ...              | ...                                              | ...                           |

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

(Adapt to the project's router — file-based or config-based.)

- **Route:** `<how this view's route is declared>`
- **Layout changes:** <shared layout modifications if any>
- **Navigation:** <how users reach this view in the running app>

## Data Layer

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

(Foundation ticket — or first ticket if no foundation)

The project's token/theme system is the **source of truth** for all design tokens. Do NOT copy the design's tokens stylesheet into the app — use project tokens via the library's props and the project's variables/classes.

| Design Token             | Project Token                  | Usage                                      |
| ------------------------ | ------------------------------ | ------------------------------------------ |
| `--blue-700` / `#0E5ECF` | brand shade 7                  | Color prop `color="brand"` or token in CSS |
| `--gray-500` / `#667085` | gray shade 5                   | ...                                        |
| ...                      | ...                            | ...                                        |

**Tokens with no project match:** <list any, with resolution per user decision>

### Foundation Setup Checklist

- [ ] Bootstrap the component library provider with the project theme/token resolver
- [ ] Import the library's base stylesheet (if required)
- [ ] Set up icon library imports
- [ ] Configure fonts
- [ ] Verify project tokens cover the design's color/spacing/radius/shadow needs
- [ ] Add supplemental tokens ONLY for design values with no project equivalent (document why)

## Acceptance Criteria

- [ ] <Specific, verifiable criterion per component>
- [ ] Components render matching design layout
- [ ] Library components used wherever mapped (see Component-to-Library Map)
- [ ] Interactive states work (hover, active, focus, disabled)
- [ ] Accessible: semantic HTML, keyboard navigation, ARIA where needed
- [ ] Co-located test files exist with smoke tests
- [ ] `<lint command>` passes
- [ ] `<test command>` passes
- [ ] Design validation passes (match, or partial with only cosmetic/minor issues) — see Verification

## Implementation Notes

- Follow the project's component conventions <cite rules file if present>
- Follow the project's data-layer conventions <cite rules file if present>
- Use `<component-lib>` components per the mapping table — do not rebuild primitives the library provides
- **All styling must reference project theme tokens** — use library props (`color="brand"`, `spacing="md"`, `radius="sm"`) for library components and project token variables/classes in custom styles. Never hardcode hex colors, px spacing, or px radii that duplicate project token values.
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

If the environment provides a design-validation tool/skill, run it to compare the running app against the design prototype. Treat it as a **required loop** — up to 3 rounds:

1. **Round 1:** Validate with design file `<design_path>`, the app route, and navigation instructions ("click <page label> in the sidebar"). Fix any critical or major discrepancies.
2. **Round 2:** Re-validate after fixes. Fix remaining major/minor discrepancies.
3. **Round 3:** Final pass. Only cosmetic discrepancies should remain.

Stop early on a clean match. Do not exceed 3 rounds — if major issues persist, document them as follow-up work. If no validation tool exists, this becomes a manual visual checklist against the design file.

**Validation parameters for this ticket:**

- Design file: `<design_path>`
- App route: `<route>`
- Design navigation: `<instructions to reach this view in the design prototype>`
- Focus areas: `<relevant dimensions if scoped, otherwise omit>`
````

**Size management.** After drafting, estimate each description's size. If it's large (over ~12 KB, or near your ticketing system's payload limit):

1. Collapse detailed inline data operations/fragments to names only — cite the project's data-layer conventions for the pattern
2. Collapse token lists into counts with category summaries
3. Remove implementation notes that merely restate the rules files — cite the path instead
4. If still over, split the ticket further

---

### Phase 6 — Present plan and get approval

Use `AskUserQuestion` to present a summary table before emitting anything:

```
Here is the proposed ticket breakdown for <epic/group/dir>:

| #   | Ticket                               | Type | Key Components              | Library Components Used     | Depends on | Est. |
| --- | ------------------------------------ | ---- | --------------------------- | --------------------------- | ---------- | ---- |
| 1   | Foundation: library setup + tokens   | Task | Provider, theme             | (setup only)                | none       | S    |
| 2   | App Shell: sidebar + topbar + layout | Task | Sidebar, TopBar             | MenuItem, Avatar, Tooltip   | #1         | M    |
| 3   | Signal Feed page                     | Task | Feed, SignalCard, FilterBar | Badge, Tag, Tabs            | #2         | L    |
| 4   | Signal detail slideout               | Task | SignalDetail, ActionList    | Drawer, Badge, Button       | #2,#3      | M    |
| ... | ...                                  | ...  | ...                         | ...                         | ...        | ...  |

Total: N tickets.
```

Options:
- "Proceed as planned"
- "Adjust scope (merge/split/reorder)"
- "Add more detail to a specific ticket"

Do not emit until the user approves.

---

### Phase 7 — Emit tickets

Emit in implementation order. Use the mode resolved in Phase 0b.

#### Mode A — Ticketing system

Create each ticket via the system's MCP / CLI / REST API:

- **Title:** under ~70 characters, verb-first (e.g., "Implement Signal Feed page with filtering")
- **Type:** the system's standard work item type (e.g., Task / Story / Issue)
- **Parent / grouping:** attach to the container resolved in Phase 0b (epic / project / milestone / parent issue), using whatever field that system requires
- **Body:** the full description from Phase 5, in the format the system expects (Markdown, ADF, etc.)

After creating each ticket, record the returned id/key/URL for cross-referencing in subsequent tickets' "Depends on" fields.

**Link dependencies** if the system supports typed links (e.g., a "Blocks" relationship, linked issues, or a dependency field) so the chain is visible in the tracker.

*System-specific notes:*

- **JIRA (worked example):** team-managed projects attach children via the `parent` field (not a custom epic-link field). Issue data from a fetch is nested (e.g., under `fields`), not top-level. Payloads over a size limit may be rejected by an upstream WAF as an HTML error response — if that happens, compress the description (shrink code blocks first, then drop examples, then convert to tables/bullets) and retry; don't split a logical ticket just to bypass the limit unless no compression works.
- **GitHub Issues:** use `gh issue create`; express grouping via labels/milestone/project and dependencies via task lists or "Depends on #N" references.
- **Linear:** use its API/MCP; group under a Project and relate issues with blocked/blocking relations.

#### Mode B — Plain files

Write one file per ticket to the output directory, ordered by implementation order.

- **Filename:** `NN-slug.<md|json>` where `NN` is the two-digit implementation order and `slug` is a kebab-case title (e.g., `03-signal-feed.md`).
- **Markdown format:** YAML front-matter + the Phase 5 body:

    ```markdown
    ---
    id: 03-signal-feed
    title: Implement Signal Feed page with filtering
    type: task
    parent: <epic/group label or null>
    depends_on: [01-foundation, 02-app-shell]
    ---

    <full Phase 5 ticket body>
    ```

- **JSON format:** the same metadata plus the body as a Markdown string:

    ```json
    {
      "id": "03-signal-feed",
      "title": "Implement Signal Feed page with filtering",
      "type": "task",
      "parent": "<epic/group label or null>",
      "depends_on": ["01-foundation", "02-app-shell"],
      "body": "<full Phase 5 ticket body as markdown>"
    }
    ```

- **Manifest:** also write an `index.md` (or `tickets.json`) listing the tickets in implementation order with their `depends_on`, so a reader (or an implementation agent) sees the build order at a glance.

Dependencies are resolved at implementation time, the `depends_on` field referencing other tickets' `id`s is just for initial reference ONLY.

---

### Phase 8 — Report back

Tell the user:

- Number of tickets emitted, and where (ticketing system + container URL, or the output directory)
- Each ticket in implementation order: id/key, location (URL or file path), title
- Dependency chain summary
- Schema/data gaps or blockers noted
- Library coverage summary: "X of Y design components mapped to library components; Z require custom implementation"
- Icon coverage: "X of Y design icons mapped (A primary, B fallback); Z require custom SVGs" — list the missing icons explicitly so the user can plan for them
- Token coverage: "X of Y design tokens mapped to project theme tokens; Z require supplemental tokens or theme extensions" — list any gaps that were flagged and how they were resolved
- Reminder: each ticket is self-contained — an implementation agent or human can pick up any unblocked ticket independently

---

## Gotchas

- **Design uses inline React/Babel.** A standalone export's `<script type="text/babel">` block compiles JSX at runtime. Parse it as JSX source — it's standard React with `useState` for state and inline styles for layout.
- **Individual `.jsx` files are preferred.** When present alongside the HTML, these are the split-out source modules — easier to parse, with logical grouping and comments. A `combined.jsx` merges them with `// ===== src/<file>.jsx =====` markers.
- **The project's token system is the token authority.** It defines the canonical color palette, spacing scale, radius scale, shadows, and typography tokens. Design prototypes (the extracted tokens stylesheet) are approximations — do NOT copy them into the app. All implementation must use project tokens (props or variables/classes). When a design token doesn't map, flag it to the user and ask how to resolve it before emitting tickets.
- **Icons have a priority order.** Map every design icon to the primary icon library first; fall back to the configured secondary; only flag "custom SVG needed" when no configured library has a suitable icon — the user needs to decide how to handle those gaps.
- **Don't delegate synthesis.** Phase 4 is the skill's highest-value step — cross-referencing design components, icons, and tokens against the library catalogs requires judgment. Do it yourself, don't farm it to a subagent.
- **Mock data vs real data.** Design prototypes use hardcoded arrays (SIGNALS, PROJECTS, etc.). Tickets must translate these into the project's data layer (queries/fragments or REST calls) per the project's conventions.
- **Design navigation vs the app router.** Designs use `useState('page')` for navigation. Implementation uses the project's router — each design "page" becomes a route.
- **Ticketing is pluggable.** Resolve the destination up front (Phase 0b). Everything from approval (Phase 6) to emission (Phase 7) is the same regardless of backend; only the create/link calls differ. Plain-file mode needs no system access at all.
