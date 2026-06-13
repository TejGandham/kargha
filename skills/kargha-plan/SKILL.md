---
name: kargha-plan
description: Analyze a design HTML export (e.g., from Claude Design) and decompose it into vertical-slice tickets for frontend implementation. Reads the design's components, tokens, navigation, and mock data; maps design components to the project's component library, tokens to its theme/token system, icons to its icon libraries, and cross-references its data layer (GraphQL schema, REST types, or stubbed). Emits self-contained tickets into any ticketing system (JIRA, Linear, GitHub Issues, …) OR as plain Markdown/JSON files. Each ticket includes a component-to-library mapping so implementers know which library components to use. Invoke when the user has a design prototype and wants implementation tickets — trigger phrases include "plan the frontend for this design", "break this design into tickets", "create frontend tickets from this design", or "slice this design into implementation work".
---



Turn a design export (e.g., a Claude Design HTML export) into vertical-slice tickets for frontend implementation. Each ticket is self-contained and independently implementable by an implementation agent/skill or a human.

The skill's core value is the **component-to-library mapping** — determining which design components map to the project's component library (and with which props/variants), which need thin wrappers, and which must be built custom. This mapping appears in every ticket.

## How this skill adapts to your project

This skill is stack-agnostic about the **project** side. It does **not** assume a specific component library, icon set, theme system, ticketing system, or repo layout. Instead, it resolves a small set of project settings up front (detect → ask), then maps the design against whatever it finds:

- **Component library:** any library, several libraries, or none. With no library, every component is "custom" — the mapping still tells you exactly what to build.
- **Icon libraries:** a primary source, optional fallbacks, or none.
- **Token/theme system:** a theme object, CSS custom properties, a design-tokens file (incl. W3C DTCG JSON — see references/dtcg-tokens.md), a utility-class framework, or plain CSS.
- **Data layer:** GraphQL schema, REST/OpenAPI types, generated TS types, or none (stub until available).
- **Ticket destination:** any ticketing system, or plain Markdown/JSON files written to a directory.

Where this document shows a concrete tool or library, treat it as an **example**, not a requirement.

**Design-input contract (the one thing this skill *does* assume).** The analysis path assumes the design is a **runtime-JSX HTML export** — a Claude Design-style export whose components are JSX (individual `.jsx` files, a `combined.jsx`, or an inline `<script type="text/babel">` block), with `useState`-driven view switching and inline styles. This is a precondition, not stack-agnosticism: a Vue/Svelte prototype, a Figma/Anima HTML export, or a plain static-HTML export will pass the "is it HTML" check and then silently mis-parse (empty component inventory, no detected views). Phase 0a gates on this format and bails with the supported-format list if it doesn't hold. Where the document states design facts ("the design uses `useState` navigation / inline styles / hardcoded mock data"), read them as properties of this supported export format.

## Project configuration (resolve once, up front)

Resolve each setting in this order: **explicit user input → detect from the repo → ask the user** (batch all unknowns into a single `AskUserQuestion`). Do not prompt for things you can detect. **When detection conflicts with explicit user input or the project's documented/blessed stack, the stated stack wins — confirm it rather than asserting what's merely present** (a repo can be mid-migration, so a detected framework/library shows the *current* state, not the intended target).

| Setting | What it is | How to resolve |
| ------- | ---------- | -------------- |
| **Frontend app dir** | Where components/routes/styles live | Detect the package that depends on the UI framework (e.g., a `package.json` with `react`/`next`/`vue`/`svelte`); in a monorepo with multiple frontend apps, **ask** which app the design targets (an export carries no app identifier, so it is not detectable). One planning run targets exactly one `<frontend-app>` |
| **Component library** | UI-primitive library/libraries the project uses (0..n) | Detect from `package.json` deps + existing import statements; may legitimately be none |
| **Icon libraries** | Primary icon source + optional fallbacks | Detect from deps/imports; may be none |
| **Token/theme system** | The source of truth for colors / spacing / radius / typography / shadows | Detect: a theme object, CSS custom properties in a global stylesheet, a design-tokens file (incl. W3C DTCG JSON — resolve the extra settings in references/dtcg-tokens.md), a utility-class config, or plain CSS |
| **Data layer** | The API the UI reads from | Detect: a GraphQL schema file, OpenAPI/REST types, generated TS client types; may be none |
| **Project rules** | Component / data conventions the repo documents | Detect: contributor docs, lint configs, or rules files; **verify the doc has real content before citing — a detected rules/design doc may be an unfilled template (placeholder headings, TODOs); cite only filled docs** |
| **Toolchain commands** | lint / test / build invocations | Detect from package scripts and the repo's task runner (npm/pnpm/yarn scripts, Nx, Turborepo, Make, etc.) |
| **Ticket destination** | Where tickets are emitted | Ask: a ticketing system (which one + how to call it) **or** plain files (output dir + `md`/`json`) — see Phase 0b |
| **Design-validation** | Optional fidelity-check tool/skill | Use if the environment provides one (e.g., a skill that screenshots the running app and diffs it against the design); otherwise the design-validation loop becomes a manual checklist |

Record the resolved values — every later phase references them. In this document, placeholders like `<frontend-app>`, `<component-lib>`, `<icon-lib>`, `<fallback-icon-lib>`, `<schema>`, `<lint command>`, `<test command>`, `<build command>` refer to these resolved settings.

### DTCG token systems (extra settings)

When the **Token/theme system** is a W3C DTCG design-token file (JSON leaves carrying `$value`/`$type`, usually alongside a token build tool such as Style Dictionary / Terrazzo), resolve the additional DTCG-only settings in **[references/dtcg-tokens.md](references/dtcg-tokens.md)** so Phases 2, 4a.2, 4c, and the ticket's **Token Changes** section can author tier-correct, build-actionable guidance. (These mirror `kargha-build`'s DTCG settings — keeping them in sync is what lets a plan-authored token addition be applied without re-asking at build time.)

## Workflow

### Phase 0 — Collect and validate inputs (hard gates)

Required inputs. If any is missing, use `AskUserQuestion` once to collect all missing values simultaneously (combine with any unresolved Project configuration questions).

**0a. Design path.** The user must provide a path to the design HTML or its parent directory. Resolve it:

```bash
design_path="<caller-provided-path>"

if [ -d "$design_path" ]; then
  # Prefer standalone variant (no CDN dependency); exclude print variants in BOTH
  # branches, and sort so the pick is deterministic rather than filesystem order.
  candidates=$(find "$design_path" -maxdepth 2 -name '*standalone*.html' ! -name '*print*' | sort)
  if [ -z "$candidates" ]; then
    candidates=$(find "$design_path" -maxdepth 2 -name '*.html' ! -name '*print*' | sort)
  fi
  design_file=$(echo "$candidates" | head -1)
elif [ -f "$design_path" ]; then
  design_file="$design_path"
else
  design_file=""   # nonexistent path — caught by the bail below
fi

# Normalize to an absolute path so downstream tickets (and the implementation
# skill, which resolves design paths against its worktree) can always find it.
# (POSIX-portable; `readlink -f` is GNU-only and fails on macOS/BSD.)
[ -n "$design_file" ] && design_file="$(cd "$(dirname "$design_file")" && pwd -P)/$(basename "$design_file")"
```

If no HTML found (or the supplied file path doesn't exist), bail with: "No design HTML files found at `<path>`. Provide a path to a design HTML export."

**If a directory yields more than one candidate** (`echo "$candidates" | wc -l` > 1 — the normal multi-page case), do not silently pick the first: list the candidates and ask the user which export to plan from (batch this with the other Phase 0 questions). Anchoring every ticket on an arbitrary pick is worse than one question.

**Format gate (the design-input contract).** After locating the HTML, confirm it is a supported runtime-JSX export: verify either `.jsx` siblings exist **or** the HTML contains a `<script type="text/babel">` block — including the **external-src** form `<script type="text/babel" src="...jsx">`, which points at a sibling `.jsx` instead of holding inline JSX. If neither holds, bail with: "This HTML is not a supported runtime-JSX design export (no `.jsx` sources or `text/babel` script found). Supported: Claude Design-style JSX exports. Re-export in that format, or point me at the `.jsx` sources." Do not proceed to Phase 1 on an unsupported export — the analysis would silently mis-parse.

Also check for sibling source files — these are easier to parse than the monolithic HTML:

- Individual `.jsx` files (e.g., `contacts-page.jsx`, `settings-pages.jsx`) — logically grouped component modules
- `combined.jsx` — all modules merged with `// ===== src/<file>.jsx =====` section markers
- An extracted tokens stylesheet (e.g., `colors_and_type.css`)
- `src/` subdirectory — split-out component files
- A screenshots/`assets`/`uploads` directory, if present — **check both the HTML's own dir and one level up** (exports commonly put screenshots in a sibling/parent `screenshots/` dir, not next to the HTML). Record its path for the ticket's Design Reference; absent is fine (the template's Screenshots line is optional)

Prefer reading individual `.jsx` files when present. Fall back to `combined.jsx`, then the inline `<script type="text/babel">` in the standalone HTML. Throughout the rest of this skill, `<design_file>` refers to the resolved absolute HTML path computed here.

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

Read the design export at `<design_file>` (and its sibling sources in the same directory). Start with the individual `.jsx` files if present, then `combined.jsx`, then fall back to the `<script type="text/babel">` block in the HTML. Also read the extracted tokens stylesheet if it exists.

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

3. **Page/view inventory.** Every distinct "view" — a rendered state worth a slice. Often these come from `page === '<id>'` conditionals or a client-side router, but a single-screen export commonly drives views with `useState` **modes / panes / modals** instead (e.g. a read pane vs an edit pane, a settings pane, a confirm modal, a mobile-nav drawer). Treat each distinct rendered state as a view, whichever mechanism switches it:
    - Page ID (e.g., 'home', 'feed', 'projects')
    - Entry component name (e.g., `HomePage`, `Feed`, `ProjectsPipelinePage`)
    - Sub-views (e.g., projects has both pipeline and detail views)
    - Components used exclusively by this page vs shared across pages

4. **Design token inventory (with values).** Every CSS custom property from `:root` and the tokens stylesheet — report each token *with its value*, not just counts/naming conventions (Phase 4a.2 must classify each design token as Direct/Close/No-match within ~2px / ~1 shade, which is impossible without the actual values). A hand-built design may tokenize **only some categories** (commonly colors + fonts, maybe one radius) and use **raw px/hex literals** for the rest (spacing, type sizes); report both — the defined tokens *and* the recurring raw literals (with the values and where they appear), so Phase 4a.2 can map the literals onto the project's scale or flag them as isolated/non-systematic values:
    - Colors: every custom property with its hex/rgb value (note the naming convention too, e.g. `--gray-25` … `--gray-950`)
    - Spacing: scale values
    - Typography: font families, size scale, weight scale
    - Radius: scale values
    - Shadows: elevation levels
    - Semantic tokens (e.g., `--bg-primary`, `--fg-primary`, `--border-primary`)
    - **Theme contexts:** if the design defines alternate contexts (e.g. a `[data-theme="dark"]` / `.dark` block, or a separate dark stylesheet), report each token's value *per context*, not just `:root`. The implementation skill needs per-context values to author multi-context token additions correctly. Also report whether the design prototype can be *switched* into an alternate context at runtime (a theme-toggle component or `useState('theme'|'darkMode'|'colorScheme')`) — this decides whether a ticket can request a `full` second design-validation loop (needs a switchable prototype) or only the `smoke` check. This full-vs-smoke decision is recorded in the ticket-template's **Theme context(s)** line (references/ticket-template.md) and consumed by `kargha-build`'s design-validation step.

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

Read the library from its installed location (e.g., `node_modules/<component-lib>/`). Check the main export/type entry (e.g., `dist/index.d.ts` / `dist/index.js`) and any components directory. **If the install can't be enumerated** (minified/bundled, types-only, a remote/CDN design system, or a nested store layout like pnpm's), fall back to the package's `.d.ts` type surface, its published docs/README, or the library's documentation site, and mark the catalog as best-effort in the report.

**Large-catalog handling.** Real icon sets (1,000+ exports) and component libraries produce inventories too large to carry in the main thread's context through Phase 4. Write the full catalogs (component props, the complete icon export list, the token inventory) to scratch files (e.g. under a temp dir) and return a **summary plus the file paths**; Phase 4 reads targeted sections on demand (e.g. greps the icon file for the design's icon names) rather than holding everything in context.

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

    **DTCG token systems — read tier/path/name from the JSON source, resolved values from the generated output.** When the token system is DTCG (per the "DTCG token systems" settings), the generated stylesheet alone hides what tier-correct mapping needs (tier, path, `$extensions` name), but the JSON source alone can't give *resolved* values without re-implementing alias/composite/context resolution — which is the build tool's job, not the planner's. So use both: take **token path**, **tier** (primitive/semantic/component, per the file or the convention doc), **emitted CSS variable name** (via the name-resolution `$extensions` key or path-derivation), and **alias target** from the JSON source; take the **resolved value per theme context** from the generated output (run `<token-build-command>` and parse it, or read a build-derived manifest the same way the implementation skill does) — do not hand-resolve aliases/overrides from the JSON. Also read the token dir's README/conventions doc and report the consumption rule (e.g. "consume semantic, never primitives") and any documented transitional-debt carve-outs. Phase 4a.2 depends on the tier and name fields to map design tokens to the *consumable* tier.

- README / setup docs for integration instructions.

Report with `file:line` citations.

---

### Phase 3 — Map data needs to the data layer (Explore subagent, parallel with Phases 1-2)

Spawn a third **Explore subagent** in parallel. (If the project has no data layer yet, this phase instead documents the data each design entity needs, to be stubbed until a backend exists.)

**Subagent brief:**

Read the project's data layer — the GraphQL schema (e.g., a `schema.graphql`), OpenAPI/REST spec, or generated TS types — at `<schema>`. Also read the design's mock data structures from the design files at `<design_file>` (top-level `const` arrays like SIGNALS, PROJECTS, TEAM, CONTACTS, etc.). **When there is no single contract artifact** (a REST/FastAPI/tRPC app may not ship one), **reconstruct the contract by triangulating both sides**: the **client** (endpoint table + fetch call signatures + request/response usage) and the **server** (route handlers + their request/response types). Treat a loader/action data router (e.g. React Router loaders/actions, TanStack Router, Remix) as a **first-class fetch boundary**, not just component-level fetching — that's often where the real endpoints are wired. **If the project has no data layer**, document the data each design entity needs (entity name, fields, inferred types) — do not generate a mock layer and do not ask the user anything (this is a read-only Explore subagent; the main thread decides how to handle missing data in Phase 4d).

Report:

1. **Type coverage.** For each design data entity, map to the data layer's types using these buckets:
    - **Matched** — fields in both the design mock and the schema
    - **Gap** — fields in the design mock with no schema equivalent
    - **Unused** — schema fields not used by any design component (for awareness)
    - **Exists-but-differs** — present but semantically/format-mismatched (single-file export vs zip; list vs paginated cursor; epoch vs ISO timestamp; a value the client derives vs one the server stores)
    - **Field-level gap on an existing entity** — a missing field on an entity that's otherwise present (note if the missing field spans multiple in-scope views)

2. **Operation mapping.** For each page's data needs, which queries/endpoints serve them (e.g., a `searchSignals` query or a `GET /signals` endpoint for the feed; a `me`/`/me` for the current user).

3. **Schema gaps.** Entities in the design with NO corresponding type. List each with:
    - Entity name
    - Which pages depend on it
    - Approximate field count

4. **Fetch-boundary planning.** Derive the containment relationships yourself from the design files (this subagent runs in parallel with Phase 1 and does not receive its hierarchy tree), then report which components own data fetching / fragments (those directly consuming entity data) vs purely presentational. If the stack uses GraphQL fragment colocation, note which components would own fragments. Phase 4 reconciles this against Phase 1's full hierarchy.

Report with `file:line` citations.

---

### Phase 4 — Synthesize and scope (main thread)

Do this yourself — do not delegate. Read all three subagent reports, then work the steps below.

**Ask ordering and batching.** The steps below surface several `AskUserQuestion` points (4a.1 icon gaps, 4a.2 step 5 token mismatches, 4d data-layer gaps). Resolve **scope first** (4b), then batch the icon-gap, token-mismatch, and data-gap questions for the *in-scope* pages into as few `AskUserQuestion` calls as practical — don't interrupt the user about gaps on pages they're about to exclude, and don't ask four separate times when one batched question works. These decisions must be made in Phase 4 (Phase 5 drafts the tickets — incl. the Token Changes section — from them), so they cannot be deferred to the Phase 6 approval; only their *reporting* lands there (the coverage block). Do the mapping work of 4a/4a.1/4a.2 first, but defer their user-facing questions until after 4b.

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

Components that are typically **custom** (no library equivalent), regardless of library:

- Page-level layouts and view controllers
- Domain-specific cards (SignalCard, ProjectCard, ContactCard)
- Domain-specific panels (detail views, slideout content)
- App chrome (Sidebar, TopBar) — though they may compose library primitives
- Charts, timelines, kanban boards
- Domain-specific filters and search

#### 4a.1. Icon mapping (required step)

Every icon used in the design must be mapped to a concrete icon import. Designs usually reference icons by string name (e.g., `'radar'`, `'target'`, `'bell'`) via an `Icon` component — resolve each to a concrete icon in the project's UI framework (e.g., a React/Vue/Svelte icon component, or an SVG sprite reference, per the resolved `<icon-lib>`).

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

**Report to user:** After completing the icon mapping, surface any gaps via `AskUserQuestion` or in the Phase 6 approval message's coverage summary (the prose block beneath the table — see Phase 6). Example: "10 of 12 design icons mapped (8 primary, 2 fallback); 2 require custom SVGs: `kanban`, `drag-handle`."

#### 4a.2. Design token mapping (required step)

**The project's token/theme system is the source of truth for all design tokens in the implementation.** The design's extracted tokens — whatever form they take (CSS custom properties, or even a full DTCG token system the *design* itself ships) — are the **spec of intended values**, not the implementation mechanism; they must NOT be copied verbatim into the app. Every design token maps back to a project token (theme key, CSS variable, or utility class). **Don't conflate the design's token system with the project's:** the project side always decides the mechanism, and the two can differ in kind (a design can ship DTCG JSON while the build-target is a non-DTCG component-library theme object — a Vuetify / PrimeVue / MUI / Chakra theme — or a Tailwind/UnoCSS config, or vice versa). The DTCG-specific guidance below (tiers, Token Changes, `<token-build-command>`) is keyed to the *project's* token system; it does **not** apply when the project is non-DTCG, even if the design ships DTCG. Adopting the design's DTCG system into a non-DTCG project is a foundation decision to flag in 4c, not an assumption. See references/dtcg-tokens.md for the full asymmetry rules.

**Process:**

1. Read the project's token inventory from Phase 2: colors (with shades), spacing scale, radius scale, font sizes, line heights, shadows, and any custom/semantic tokens. Map **what the design tokenized**; for categories the design left as **raw literals** (common in hand-built designs — read them directly from the design source, not the token inventory), map each recurring literal onto the project's scale (`16px` → the `md` spacing token) or, if it has no scale fit and isn't reused, flag it as a one-off to confirm with the user.

2. For each design token category, map design values to project equivalents. The exact form depends on the token system (a theme prop, a CSS variable, or a utility class):

    | Category  | Design Token Example       | Project Token Equivalent (form varies by system)              | How to Use                                                  |
    | --------- | -------------------------- | ------------------------------------------------------------- | ---------------------------------------------------------- |
    | Color     | `var(--blue-700)`          | brand/primary shade 7 — theme prop, CSS var, or class          | Use the library color prop on components; the token in CSS  |
    | Spacing   | `var(--space-16)` / `16px` | the `md` (or equivalent) spacing token                         | Use the spacing prop or token, not a raw px value           |
    | Radius    | `8px` / `var(--radius-md)` | the `md` radius token                                          | Use the radius prop or token                                |
    | Font size | `14px`                     | the `sm` font-size token                                       | Use the typography prop or token                            |
    | Shadow    | `var(--shadow-sm)`         | the `sm` shadow token                                          | Use the shadow token                                        |

    **Tiered (DTCG) token systems — map to the *consumable* tier.** When the project's token system has tiers (per the "DTCG token systems" settings), the project-token side of every mapping must be a token in the tier new code is allowed to consume — typically **semantic**, never primitives (e.g. design `--blue-700` → `semantic.color.accent`, *not* a raw ramp shade like `--p600`). Shade-ramp names are primitive-tier, and the implementation skill `kargha-build` runs a deterministic check that flags primitive consumption in new code — so a map that resolves a design value to a primitive shade sends the implementer straight into a guaranteed conformance failure. If a design value matches *only* a primitive with no semantic equivalent, that is a **"no semantic match"** (classified "No match" in step 3, then handled as a candidate new semantic token in step 5 / the Token Changes section), not a direct mapping. Capture the tier of each project token mapped (the Phase 2 DTCG inventory provides it).

3. Classify each design token:

    | Classification   | Meaning                                                | Action                                                                                          |
    | ---------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
    | **Direct match** | Design value maps exactly to a project token           | Use the project token/prop — document the mapping                                               |
    | **Close match**  | Design value is within ~2px / ~1 shade of a token      | Use the closest token — note the minor difference in the ticket                                 |
    | **No match**     | Design value has no equivalent in the consumable tier  | **Flag to user.** For a tiered (DTCG) system, offer the options in the implementation skill's vocabulary (next paragraph) so the decision is build-actionable, not a vague "extend the theme" |

4. Include a **Design Token Map** section in the foundation ticket (or the first ticket if no foundation ticket — see Phase 4c for which case applies). So that page tickets stay self-contained (the implementation skill reads one ticket file at a time, and this skill guarantees — see the intro and Phase 4e — that any unblocked ticket is independently implementable), do **not** make later tickets depend on reading the foundation ticket. Instead, in Mode B write the full map once as a sibling artifact `00-token-map.md` and have each ticket's Design Token Map line cite it by path *and* carry the slim subset of tokens that ticket actually uses; in Mode A, carry the slim per-ticket subset inline. Any token the ticket proposes to add goes in that ticket's **Token Changes** section (see the Token Changes section in references/ticket-template.md), not only the foundation ticket.

5. **When tokens don't map:** Use `AskUserQuestion` to surface mismatches before filing tickets, and **record the user's decision in the ticket's Token Changes section** so the implementation skill can act on it without re-asking. For a tiered (DTCG) system, frame the options in the implementation skill's vocabulary:
    - **(a) Use the nearest existing token in the consumable tier** (note the delta in the ticket).
    - **(b) Add an additive semantic-tier token** — alias to a named existing primitive when one matches, else a literal value; record per theme context. Do **not** alias a primitive that sits in a documented transitional-debt block (per the token dir's README/conventions) — it's re-pointed per context and the alias would resolve wrong; use a literal (with explicit per-context values) instead. This is the one option the build skill can apply autonomously, so record it as pre-authorization (token name, tier, alias-or-literal, per-context values, target source file).
    - **(c) Adjust the design** to use an existing token.
    - **(d) A new primitive, or mutating an existing token's value** — only when (a)/(b) won't do. These can *never* be applied autonomously by the build skill; mark them "requires build-time confirmation" or land them in the foundation ticket under explicit user approval.

    Example: "The design uses `--gray-25` (#FAFAFA); the project's `gray` ramp starts at `--n50` (#F8F9FA) and there is no semantic token for this surface. Options: (a) use `semantic.color.surface` (Δ ~2 in lightness), (b) add a semantic token `semantic.color.surface-subtle` = `#FAFAFA` (alias none — literal; dark context: `<value>`), (c) nudge the design to `semantic.color.surface`?" Avoid "one-off override" — a hardcoded literal in component CSS collides with the Key rule and the build skill's conformance check.

**Key rule:** Custom components must reference project theme tokens (theme variables / utility classes) — never hardcoded hex values or px that duplicate what the token system already provides. This ensures theme changes propagate everywhere. For a tiered (DTCG) system, "project theme tokens" means the **consumable tier only** (typically semantic); primitives are deny-listed for new code.

#### 4b. Scope the work

Parse the user's prompt against the page inventory from Phase 1. If ambiguous, use `AskUserQuestion`:

> "The design contains these pages: [list]. Which ones should I plan tickets for?"

Also check: does the prompt mention related work already in progress? If so, note dependencies.

**Architectural-reversal check.** While scoping, watch for design elements that **conflict with or reverse a deliberate existing architectural decision** — a pattern the app intentionally removed or replaced (the Phase 2 codebase survey, plus any architecture/decision docs the repo exposes, are the signal). Re-adding it is not a neutral implementation detail; surface it as an **explicit scope decision** via `AskUserQuestion` (implement as designed / keep the current architecture and adapt the design / descope), rather than silently planning the reversal.

#### 4c. Foundation ticket (conditional)

Check whether the component library + theme system are integrated in the frontend app (Phase 2, Part A, item 7):

- **Not integrated yet:** Create a foundation ticket covering: provider/theme bootstrap with the project token system, icon library setup, and font setup. The project token system IS the source of truth — do NOT copy the design's tokens stylesheet into the app. Configure the library's theme/token resolver so all tokens are available as the project's variables/props. The foundation ticket carries the full **Design Token Map** and (if any) the **Token Changes** section. Only add supplemental tokens for design values that have no project equivalent (and flag these per 4a.2). **DTCG edit-path:** supplemental tokens are added by editing the DTCG JSON in `<token-source-dir>` (naming the tier file, e.g. `semantic.json`, plus the per-context file such as `semantic.dark.json` for any theme-context override) and running `<token-build-command>`; the `<generated-token-artifacts>` are read-only and must never be hand-edited. If the project has no component library, the foundation ticket instead establishes the token system, base primitives, and fonts the rest of the work builds on.
- **Already integrated:** Skip the foundation ticket. There is then **no foundation ticket**, so the **first page ticket** carries the Design Token Map (Mode B: a sibling `00-token-map.md` plus each ticket's slim subset, per 4a.2 step 4). If design tokens differ from what's configured, note "verify/update existing tokens" in the relevant page tickets.
- **Partially integrated:** Create a scoped foundation ticket covering only the missing pieces.

Also check whether shared app chrome (Sidebar, TopBar, root layout) exists. If not and the design includes it, either include it in the foundation ticket or as a separate app-shell ticket.

**Foundation and app-shell tickets and the Design Reference.** A pure foundation ticket has no design *view* to validate. It still gets a Design Reference (the implementation skill hard-gates on the section's presence), but with `**View:** \`none\` — setup ticket, design validation not applicable`. When drafting from the ticket template (references/ticket-template.md) for such a ticket, apply the foundation carve-outs the template marks: the Acceptance Criteria's "Design validation passes" line and the whole Design Validation Loop are N/A. The implementation skill's Gate 3 accepts `View: \`none\`` and skips design validation (and the dev server) for these — so this is a wired contract, not a request. An app-shell ticket (Sidebar/TopBar) *does* have a view — give it a real `**View:**` and route.

#### 4d. Data-layer gaps (always ask)

For every data entity in the design that has no equivalent in the data layer, surface it to the user via `AskUserQuestion`. Offer per-entity options:

- **Stub data:** Plan the frontend with mocked data, note as blocked until backend work is done
- **Exclude:** Skip pages that depend on this entity for now
- **Include backend work:** Add schema/resolver/endpoint work to the frontend ticket (makes it larger)
- **Plan a canonical mock layer** (when the project has *no* data layer at all): emit a ticket to establish a mock data layer (e.g. OpenAPI-derived or another user-chosen standard) that the page tickets build against. This is the planning-time home for the mock-layer decision — Phase 3's subagent only *documents* the needed entity shapes; it never generates or asks.

Group related entities in a single question when practical.

**Prototype-vs-target reconciliation.** Design prototypes are often **single-user, localStorage/in-memory mocks**; the build target may be **multi-user with real auth and server persistence**. Reconcile the *semantics*, not just field shapes: ownership/visibility (whose data is this?), concurrency, identity/auth, and persistence boundaries are gaps even when the data shape matches — surface them like other data-layer gaps. Also handle **interactive-but-dataless** features (e.g. a validation-only form, a calculator, a filter UI with no backing entity): these need no schema work — note them as such so they aren't mistaken for data gaps or blocked on backend work.

#### 4e. Vertical slice boundaries

Apply these heuristics:

1. **Foundation slice** (only if needed per 4c): library/theme setup + design tokens + app shell
2. **One slice per page/view** when the page has unique components. For a **one-page app** (no routes — views are `useState` modes/panes/modals), this heuristic gives nothing to split on; fall back to slicing by **mode/pane/component boundary** (e.g. a read-pane slice, an edit-pane slice, a settings-pane slice, the confirm modal bundled per heuristic 6)
3. **Split list + detail** when both exist for an entity (e.g., Projects list vs Project detail)
4. **Complex reusable components** get their own slice when >100 lines in design and used across pages
5. **Cross-cutting features** (AI Assistant, Copilot, global search) are separate slices
6. **Dialogs/modals** that serve a single page bundle with that page

Each slice must be independently implementable and verifiable — it should render something meaningful on its own.

**Estimate each slice (S/M/L).** Derive a coarse effort size from the slice's component count and the per-component complexity from Phase 1 (`trivial <30 lines / moderate 30–100 / complex >100`): **S** = mostly library matches, ≤2 custom components, none complex; **M** = a few custom components or one complex one; **L** = many custom components or multiple complex ones, or list+detail in one slice. This `estimate` is shown in the Phase 6 table and emitted in the ticket (Phase 7 Mode B front-matter / Mode A as the system's estimate field), satisfying the implementation skill's documented `{id, title, type, parent, depends_on, estimate}` file-ticket contract.

---

### Phase 5 — Draft ticket descriptions

For each identified vertical slice, draft a ticket description by copying the canonical template in **[references/ticket-template.md](references/ticket-template.md)** verbatim and filling its placeholders (`<frontend-app>`, `<component-lib>`, etc. refer to the resolved Project configuration). **The section headings and line formats in that template are the plan→build parse contract — `kargha-build` parses the emitted ticket, so emit them exactly as written** (em-dashes, backticks, and the ` / ` cell separators included).

**Size management.** After drafting, estimate each description's size. If it's large (over ~12 KB, or near your ticketing system's payload limit):

1. Collapse detailed inline data operations/fragments to names only — cite the project's data-layer conventions for the pattern
2. Collapse token lists into counts with category summaries
3. Remove implementation notes that merely restate the rules files — cite the path instead
4. If still over, split the ticket further

---

### Phase 6 — Present plan and get approval

Print the breakdown as normal output (the multi-row table is too wide for `AskUserQuestion` option labels), then use `AskUserQuestion` for the decision. Present this before emitting anything:

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

Coverage:
- Library: X of Y design components mapped to library components; Z custom
- Icons: X of Y mapped (A primary, B fallback); Z need custom SVGs: <list>
- Tokens: X of Y mapped to the consumable tier; Z proposed as Token Changes
```

This coverage block beneath the table is where 4a.1's icon-gap line and 4a.2's token-gap line land (their other channel is `AskUserQuestion`).

Options (as the `AskUserQuestion`):
- "Proceed as planned"
- "Adjust scope (merge/split/reorder)"
- "Add more detail to a specific ticket"

Do not emit until the user approves. On **Adjust scope**, collect the changes, revisit 4e (re-slice) and Phase 5 (re-draft) for the affected tickets, and re-present this approval step. On **Add more detail**, expand the named ticket and re-present. Loop until the user picks "Proceed as planned".

---

### Phase 7 — Emit tickets

Emit in implementation order. Use the mode resolved in Phase 0b.

#### Mode A — Ticketing system

Create each ticket via the system's MCP / CLI / REST API:

- **Title:** under ~70 characters, verb-first (e.g., "Implement Signal Feed page with filtering")
- **Type:** the system's standard work item type (e.g., Task / Story / Issue)
- **Parent / grouping:** attach to the container resolved in Phase 0b (epic / project / milestone / parent issue), using whatever field that system requires
- **Estimate:** write the Phase 4e S/M/L estimate to the system's estimate/story-points field if it has one (else fold it into a label or the body); this is the Mode A counterpart to Mode B's `estimate` front-matter
- **Body:** the full description from Phase 5, in the format the system expects (Markdown, ADF, etc.)

After creating each ticket, record the returned id/key/URL for cross-referencing in subsequent tickets' "Depends on" fields. **Emit in an order that topologically respects "Depends on"** — every ticket's dependencies must be created before it — otherwise the id needed for a "Depends on" reference won't exist yet. **On a create failure mid-batch** (auth expiry, rate limit, invalid field), stop; report which tickets were created (with ids) and which remain; ask whether to retry the failed one and resume, rather than re-emitting the whole set from scratch. (The JIRA WAF/oversize note below is a specific instance — compress and retry that one.)

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
    estimate: M
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
      "estimate": "M",
      "body": "<full Phase 5 ticket body as markdown>"
    }
    ```

- **Manifest:** also write an `index.md` (or `tickets.json`) listing the tickets in implementation order with their `depends_on`, so a reader (or an implementation agent) sees the build order at a glance.
- **Token map:** when a Design Token Map exists, also write it in full as a sibling `00-token-map.md`, which each ticket's Design Token Map cites by path (per 4a.2 step 4). This keeps page tickets self-contained without repeating the full map.

The `depends_on` ids are informational ordering hints for readers and whoever schedules implementation; the implementation skill does not gate on them. Resolve actual build order from them at implementation time.

---

### Phase 8 — Report back

Tell the user:

- Number of tickets emitted, and where (ticketing system + container URL, or the output directory)
- Each ticket in implementation order: id/key, location (URL or file path), title
- Dependency chain summary
- Schema/data gaps or blockers noted
- Library coverage summary: "X of Y design components mapped to library components; Z require custom implementation"
- Icon coverage: "X of Y design icons mapped (A primary, B fallback); Z require custom SVGs" — list the missing icons explicitly so the user can plan for them
- Token coverage: "X of Y design tokens mapped to the consumable tier; Z proposed as Token Changes (semantic additions)" — list any gaps that were flagged and how they were resolved
- Reminder: each ticket is self-contained — its Component-to-Library Map, the token subset it uses (plus, in file mode, a pointer to the sibling `00-token-map.md`), and any Token Changes travel with the ticket, so an implementation agent or human can pick up any unblocked ticket independently

---

## Gotchas

- **The design input must be a runtime-JSX export.** The whole analysis path assumes a Claude Design-style export (`.jsx` sources or a `text/babel` script, `useState` views, inline styles). Phase 0a gates on this; a Vue/Svelte/Figma/static-HTML export passes the "is it HTML" check and then silently mis-parses. The "design uses X" statements throughout (inline styles, `useState` navigation, hardcoded mock data) are properties of this supported format, not universal truths.
- **Design uses inline React/Babel.** A standalone export's `<script type="text/babel">` block compiles JSX at runtime. Parse it as JSX source — it's standard React with `useState` for state and inline styles for layout.
- **Individual `.jsx` files are preferred.** When present alongside the HTML, these are the split-out source modules — easier to parse, with logical grouping and comments. A `combined.jsx` merges them with `// ===== src/<file>.jsx =====` markers.
- **The project's token system is the token authority.** It defines the canonical color palette, spacing scale, radius scale, shadows, and typography tokens. Design prototypes (the extracted tokens stylesheet) are approximations — do NOT copy them into the app. All implementation must use project tokens (props or variables/classes). When a design token doesn't map, flag it to the user and ask how to resolve it before emitting tickets.
- **Tiered (DTCG) token systems have a consumable tier.** Map design tokens to the tier new code may consume (typically semantic, never primitives — shade ramps like `--p600` are primitives). A map that resolves a design value to a primitive shade sends the implementer straight into the build skill's deterministic primitive-deny check. Inventory tiers from the JSON source (not the generated CSS, which hides tier/path/`$extensions`/context), and route any needed-but-missing token through the ticket's Token Changes section (semantic, additive, per-context) so the build skill can add it without re-asking. Never hand-edit generated token artifacts; edit `<token-source-dir>` JSON and run `<token-build-command>`.
- **Icons have a priority order.** Map every design icon to the primary icon library first; fall back to the configured secondary; only flag "custom SVG needed" when no configured library has a suitable icon — the user needs to decide how to handle those gaps.
- **Don't delegate synthesis.** Phase 4 is the skill's highest-value step — cross-referencing design components, icons, and tokens against the library catalogs requires judgment. Do it yourself, don't farm it to a subagent.
- **Mock data vs real data.** Design prototypes use hardcoded arrays (SIGNALS, PROJECTS, etc.). Tickets must translate these into the project's data layer (queries/fragments or REST calls) per the project's conventions.
- **Design navigation vs the app router.** Designs use `useState('page')` for navigation. Implementation uses the project's router — each design "page" becomes a route.
- **Ticketing is pluggable.** Resolve the destination up front (Phase 0b). Everything from approval (Phase 6) to emission (Phase 7) is the same regardless of backend; only the create/link calls differ. Plain-file mode needs no system access at all.
