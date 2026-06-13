---
name: build-frontend-generic
description: Implement a frontend ticket (planned by plan-frontend-generic) end-to-end in an isolated git worktree with design-aware and optional data-layer-aware validation. Resolves the project's stack, ticketing system, and toolchain up front. Gates on ticket status and assignee, reads the plan with its Design Reference and Component-to-Library Map, picks up the ticket (assign + move to in-progress), spins up a worktree, implements the frontend components, runs lint/test, optionally validates data-layer (e.g. GraphQL) conformance in a loop (up to 3 rounds on modified files containing data operations), starts the dev server, optionally validates visual fidelity against the design prototype in a loop (up to 3 rounds of capture-compare-fix), and finally opens a PR via the project's mechanism. Invoke when the user points at a frontend implementation ticket from plan-frontend-generic — trigger phrases include "build this frontend ticket", "implement the frontend ticket", "pick up the frontend work", or any request to implement a ticket that has a Design Reference section. This skill should NOT trigger for non-frontend tickets (those go to a non-frontend implementation skill/agent).
---



Drive a frontend ticket from a pickup-eligible status (e.g. "Backlog"/"To Do") all the way to a review-pending status with an open PR, including visual design validation. The ticket must have been authored by `plan-frontend-generic` and contain a **Design Reference** section with the design file path, view ID, and navigation instructions, plus a **Component-to-Library Map**.

This skill combines a generic ticket-handling pipeline (Phases 0–4 — fetch, gate, pick up, worktree) with two frontend-specific validation loops — an optional data-layer conformance check (Phase 5b, only when the project has a data layer such as GraphQL) and an optional visual design comparison (Phase 7, only when a design-validation tool/skill is available) — then opens a PR via the project's VCS mechanism.

## How this skill adapts to your project

This skill is stack-agnostic and ticketing-agnostic. It does **not** assume a specific component library, icon set, theme system, frontend framework, data layer, ticketing system, branch convention, or repo layout. Instead, it resolves a small set of project settings up front (detect → ask), then implements against whatever it finds:

- **Ticket source:** the ticket may live in a ticketing system (JIRA, Linear, GitHub Issues, …) **or** be a plain Markdown/JSON ticket file produced by `plan-frontend-generic`. Both input modes are supported — Phase 0a resolves which mode is active.
- **Component library:** any library, several libraries, or none. With no library, the Component-to-Library Map's "custom" entries are the build list.
- **Theme/token system:** a theme object, CSS custom properties, a design-tokens file, a utility-class framework, or plain CSS. The "no hardcoded values that duplicate tokens" rule holds regardless.
- **Data layer:** GraphQL (with fragment colocation/codegen), REST/OpenAPI types, tRPC, generated TS types, or none. The data-conformance loop (Phase 5b) is **conditional** on the project actually using a data layer with documented conventions.
- **Toolchain:** lint / test / build / dev / install commands run via whatever package manager and task runner the project uses (npm/pnpm/yarn scripts, Nx, Turborepo, Make, …).
- **Design-validation:** an optional fidelity-check tool/skill. If the environment provides one, Phase 7 runs it as a loop; otherwise Phase 7 degrades to a manual visual checklist.

Where this document shows a concrete tool, command, library, or status name, treat it as an **example**, not a requirement.

## Project configuration (resolve once, up front)

Resolve each setting in this order: **explicit user input → detect from the repo → ask the user** (batch all unknowns into a single `AskUserQuestion`). Do not prompt for things you can detect.

| Setting | What it is | How to resolve |
| ------- | ---------- | -------------- |
| **Frontend app dir** | Where components / routes / styles live, and its task-target name | Detect the package depending on the UI framework (a `package.json` with `react`/`next`/`vue`/`svelte`); in a monorepo pick the app the ticket targets; else ask |
| **Component library** | UI-primitive library/libraries the project uses (0..n) and its installed path | Detect from `package.json` deps + existing imports; resolve the install path from the manifest; may legitimately be none |
| **Icon libraries** | Primary icon source + optional fallbacks | Detect from deps/imports; may be none |
| **Theme/token system** | Source of truth for colors / spacing / radius / typography / shadows | Detect: theme object, CSS custom properties, a design-tokens file (incl. W3C DTCG JSON — see "DTCG token systems" below), a utility-class config, or plain CSS |
| **Data layer** | The API the UI reads from, and how to detect operations in source | Detect: a GraphQL schema + codegen, OpenAPI/REST types, a tRPC router, generated TS client types; may be none. Note the detector `<data-layer-detector>` (e.g. files containing `graphql(`) and the generated-code directory `<generated-code-dir>` to exclude, both used in Phase 5b (a project with no codegen may have no `<generated-code-dir>`) |
| **Ticketing system access** | Where the ticket lives and how to read/update it | Ask/detect: a ticketing system (which one + how to call it — MCP, CLI like `gh`/`glab`, REST) **or** a plain Markdown/JSON ticket file path — Phase 0a resolves the mode |
| **Pickup statuses** | The set of statuses a ticket may be in to be picked up | Ask/detect; default to the project's first two backlog-ish states (e.g. "Backlog", "To Do"); for file tickets, the `status` front-matter field |
| **In-progress status** | The status to move the ticket to on pickup | Ask/detect (e.g. "In Progress") |
| **Review status** | The status to move the ticket to once the PR is open | Ask/detect (e.g. "In Review", "Ready for review") |
| **Toolchain commands** | install / lint / test / build / dev invocations | Detect from package scripts + task runner; record `<install command>`, `<lint command>`, `<test command>`, `<build command>`, `<dev command>` |
| **Dev server URL/port** | Where the running app is served (and optional backend service + port) | Detect from dev-server config / framework defaults; record `<dev-server-port>`, and note any backend the dev target starts transitively and its port `<backend-port>` |
| **Default branch** | Base branch for worktrees | Detect (`git symbolic-ref refs/remotes/origin/HEAD`, or fall back to `main`/`master`) |
| **Worktree root** | Parent dir for per-ticket worktrees | Ask/default to a sibling dir (e.g. `../<repo>-worktrees/`) |
| **Branch convention** | Branch-name template — must embed the ticket id | Ask/detect; default `<prefix>_<ticket-id>_<slug>`; the embedded ticket id lets the PR step recover it |
| **PR conventions** | Target repo, reviewers, labels, title prefix | Detect repo from the git remote; ask/detect reviewer-selection rule, label set, and ticket-id title prefix (any may be none). The reviewer-selection rule must exclude the PR owner from reviewing their own PR (e.g. pick the other member of a fixed reviewer set) |
| **Project rules** | Component-structure and data-layer convention docs | Detect: contributor docs, lint configs, or rules files; cite them during implementation/fixes if present, else fall back to inline generic conventions |
| **Design-validation** | Optional fidelity-check tool/skill | Use if the environment provides one (a skill that screenshots the running app and diffs it against the design); otherwise Phase 7 becomes a manual checklist |

Record the resolved values — every later phase references them. In this document, placeholders like `<frontend-app>`, `<component-lib>`, `<lint command>`, `<dev command>`, `<dev-server-url>`, `<dev-server-port>`, `<backend-port>`, `<data-layer-detector>`, `<generated-code-dir>`, `<default-branch>`, `<ticket-id>`, `<pickup-statuses>`, `<in-progress-status>`, `<review-status>` refer to these resolved settings.

The current user's identity is **not** hardcoded — resolve it dynamically at the start of every run (Phase 0).

### DTCG token systems (extra settings, resolved only when detected)

When the theme/token row detects a **design-token file system in W3C DTCG format** (first stable spec 2025.10) — JSON files whose leaves carry `$value`/`$type`, usually alongside a token build tool in devDependencies (Style Dictionary, Terrazzo, Dispersa, …) and a build script — resolve these additional settings:

| Setting | What it is | How to resolve |
|-|-|-|
| `<token-source-dir>` | The DTCG JSON files — the **only** editable token surface | Detect from the build script's `source` config or glob for `$value`-bearing JSON |
| `<token-build-command>` | Regenerates platform artifacts from the JSON | Detect from package scripts (e.g. `build:tokens`) |
| `<generated-token-artifacts>` | Build outputs (e.g. a generated `tokens.css`) — never hand-edit; excluded from the Phase 5 primitive/hardcode scans (checks 2–3), analogous to `<generated-code-dir>` | Detect from the build script's output path or a "GENERATED" file header |
| Tier convention | Which tiers exist (primitive/semantic/component) and which one components may consume | Read the token dir's README/conventions doc; a common rule is "consume semantic, never primitives" — including documented **transitional-debt carve-outs** (legacy code consuming primitives) that must be tolerated where not touched |
| Name-resolution rule | How a token path maps to the emitted variable name — often a vendor `$extensions` key (e.g. `$extensions["com.<project>.cssName"]`), falling back to path-derived names | Detect by scanning `$extensions` keys on a few tokens; the key is **project-specific — never assume a particular vendor key**. If ambiguous, confirm in the batched config `AskUserQuestion` |
| Theme-context selector | How alternate contexts are activated (e.g. `[data-theme="dark"]`, a `.dark` class, `prefers-color-scheme`) | Detect from the build config's per-context selectors |

**Token manifest (built in Phase 4 step 4c-bis, before any implementation lookup; rebuilt after any token edit).** Do **not** re-implement DTCG resolution (alias chains, composite serialization, multi-file merge order) — that is the token build tool's job and exactly where the bugs live. Instead, derive the manifest from the **generated artifacts**: run `<token-build-command>`, parse the generated output per context selector into `variable → resolved value → theme context`, then join the DTCG JSON for metadata only (token path, tier, resolved `$type` — walk ancestor groups for inherited types — and vendor name). This assumes the build emits **CSS custom properties** (the common case, and what the conformance check and `TOKEN_DRIFT` reverse-map consume). If `<token-build-command>` instead emits a JS theme object, a Tailwind config, or SCSS, adapt the parse to that output's name→value form per context; if that isn't practical, skip the manifest and the Phase 5 token-conformance check and fall back to the generic "no hardcoded values that duplicate tokens" rule, noting the reduced coverage in the PR body. The manifest serves three consumers: implementation lookups (Phase 4d), the token-conformance check (Phase 5), and reverse-mapping design-validation `TOKEN_DRIFT` from raw values to token paths (Phase 7). Also extract the **primitive-tier variable list** — the deny-list for new code. A stale manifest silently breaks all three consumers, so any re-run of `<token-build-command>` (e.g. after a Phase 4d token addition) re-derives it. If `<token-build-command>` fails, retry once; if it still fails, surface via `AskUserQuestion` before implementing (token-correct styling depends on it) or degrade explicitly to read-only parsing of the committed `<generated-token-artifacts>`, noting the limitation in the PR body.

If the token dir carries DTCG **resolver-module** files (`*.resolver.json`), record their presence but do not branch on them — the skill runs the project's own build command, so it inherits resolver support whenever the project's toolchain ships it.

---

## Workflow

### Phase 0a — Resolve the ticket source mode

Before resolving identity or fetching anything, determine which **ticket mode** is active — every later phase (0, 1, 2a, 3a, 3b, 9) branches on it.

- **Ticketing-system mode:** the input is a ticketing-system reference — an id/key (e.g. a project-prefixed key), or a ticket URL. Resolve which system it is (the one resolved in the "Ticketing system access" config row) and how to call it (MCP, a CLI like `gh`/`glab`, or REST).
- **File-ticket mode:** the input is a path to a Markdown/JSON ticket file produced by `plan-frontend-generic`. Note the path; the front-matter (or top-level JSON fields) and body are read directly.

Resolve the mode in this order: explicit user input → detect from the reference shape (a filesystem path that exists ⇒ file-ticket; an id/key/URL ⇒ ticketing-system) → ask the user. Record `TICKET_MODE` (`ticketing-system` | `file-ticket`) for the rest of the run.

**File-ticket lifecycle-field contract.** `plan-frontend-generic` file tickets carry front-matter `{id, title, type, parent, depends_on, estimate}` — there is **no** `status` and **no** `assignee` field by default. So in file-ticket mode the status and assignee gates (Phase 1) are informational-only and pass when the field is absent, the pickup assign/transition steps (Phase 3) are additive writes (they create the field if missing) or no-ops, and the review-status write (Phase 9) is additive. This is expected, not an edge case. If a file ticket *does* carry `status`/`assignee` fields (a custom producer), honor them as real gates.

### Phase 0 — Resolve the current user

Before any gates or ticket work, look up who "the current user" is from the **authoritative source for the configured ticket mode**.

- **Ticketing-system mode:** call the system's "who am I" operation (e.g. a JIRA user-info call, `gh api user`, the Linear `viewer` query) to get the current user's id and email/handle. This is the only authoritative identity for the assignee gate and the assignment side-effect.
- **File-ticket mode:** there is no remote identity service. Fall back to the local git/config identity (`git config user.email`, `git config user.name`) for PR ownership and — only when the file format actually carries an `assignee` field — the assignment field; otherwise the git identity is informational. If `git config` has no `user.email`/`user.name`, ask the user once for an identity to record, or proceed with an explicit "unassigned" note — do not silently invent one.

Cache the resolved values for the rest of the run:

- `CURRENT_USER_EMAIL` — used for the assignee gate (Phase 1) and error messages
- `CURRENT_USER_ID` — the system's account/user identifier, used for the assignment side-effect (Phase 3)

**Identity normalization (provider-specific).** Different systems return identity fields under different shapes and casings — e.g. one system may return a snake_case `account_id` from its "who am I" call while its issue payloads use camelCase `assignee.accountId`. Both refer to the same identity; normalize them into the single `CURRENT_USER_ID` variable and compare that throughout the skill. Keep this normalization note only where the configured provider needs it.

Don't fall back to the OS `$USER` or the harness email for a **ticketing-system** identity — those are the OS/VCS identity, which often doesn't match the ticketing one. Only file-ticket mode legitimately uses the git/config identity.

If the identity lookup fails in ticketing-system mode, bail — the whole skill is predicated on knowing who "me" is.

### Phase 1 — Fetch, gate, and validate it's a frontend ticket

Resolve the ticket reference from the user. Depending on mode:

- **Ticketing-system mode:** take the ticket id/key (or extract it from a ticket URL), then fetch the ticket with its description, comments, status, assignee, and type. Parse the response per that system's payload shape (e.g. JIRA nests fields under `fields`, with comments under `fields.comment.comments[]`; GitHub Issues returns `body`/`state`/`assignees`; Linear returns the issue node). Render the description/body as Markdown.
- **File-ticket mode:** read the Markdown/JSON ticket file produced by `plan-frontend-generic`. The "status"/assignee live in the front-matter (or top-level JSON fields); the body is the ticket description. There are no remote comments — any review notes live inline or in sibling files the ticket references.

Three gates, **all must pass**. Any failing is an immediate hard stop — no `AskUserQuestion`, no "want me to continue anyway?", just report and exit.

**Gate 1 — Status.** The ticket's status/state must be in the configured **pickup-eligible set** `<pickup-statuses>` (e.g. exactly `"Backlog"` or `"To Do"`).

If it's any other status (e.g. an in-progress, review, or done state), bail with:

- The ticket id and current status
- Who it's assigned to (if anyone)
- A note that this skill only picks up tickets in the pickup-eligible statuses

For file tickets with no `status` field, treat the gate as passing (there is no workflow to violate) and note it. **This is the norm for `plan-frontend-generic` file tickets** — their front-matter carries no `status` field (see Phase 0a), so this gate is informational in the common file-ticket case. Apply it as a real gate only when the file actually carries a `status` field.

**Gate 2 — Assignee.** The ticket must be either **unassigned** **or** **assigned to the current user**.

Apply the comparison in this order (identity-normalized, id first, then email/handle fallback):

1. If the assignee is empty/null, the gate passes.
2. If the assignee has an account/user id, compare to `CURRENT_USER_ID`. Match passes; mismatch bails.
3. Only if the assignee id is missing, fall back to comparing the assignee email/handle against `CURRENT_USER_EMAIL`.

On failure, bail with the ticket id, current assignee, and `CURRENT_USER_EMAIL`. For file tickets with no assignee field, the gate passes — and **`plan-frontend-generic` file tickets carry no `assignee` field by default** (see Phase 0a), so this gate is informational in the common file-ticket case. Apply it as a real gate only when the file actually carries an `assignee` field.

**Gate 3 — Frontend ticket validation.** Parse the description/body for the **"Design Reference"** section. This section must contain at minimum:

- `**Design file:**` with a file path
- `**View:**` with a view/page ID and navigation instructions — or the literal `none` (optionally followed by an annotation, e.g. `\`none\` — setup ticket, design validation not applicable`) for a setup/foundation ticket (see below)

If the Design Reference section is missing, bail with: "This ticket does not contain a Design Reference section. It was likely not planned by `plan-frontend-generic`. Use a non-frontend implementation skill/agent to implement it instead."

When matching these field labels (here and in the extraction below), tolerate formatting variations — a ticketing system may have round-tripped the Markdown to rich text and back, turning `**Design file:**` into `*Design file:*`, `<b>Design file:</b>`, or plain `Design file:`. Match the label text case-insensitively, ignoring surrounding emphasis/markup; don't bail on a present-but-reformatted section.

**Setup/foundation tickets.** `plan-frontend-generic` emits foundation/setup tickets (library + theme/token bootstrap, fonts) with `**View:** \`none\` — setup ticket, design validation not applicable`. These pass Gate 3 (the Design Reference section is present), but they have no view to validate: when `DESIGN_VIEW` resolves to `none`, **skip the design-validation loop (Phase 7) and its dev-server prerequisite (Phase 6)** — there is nothing to capture-and-compare — and don't fail the missing "Design validation passes" acceptance criterion. Implement, lint/test, run the token-conformance check, and open the PR. (A setup ticket may still touch the data layer and tokens, so Phases 5/5b still apply.)

**Extract and cache** the following from the ticket description for use in later phases:

- `DESIGN_FILE_PATH` — from the `**Design file:**` line
- `DESIGN_VIEW` — view/page ID from the `**View:**` line: the backtick-delimited identifier before the em-dash separator (`plan-frontend-generic` emits `**View:** \`<page id>\` — <navigation>`). If it isn't backtick-delimited, take the page-id token between `**View:**` and the em-dash.
- `DESIGN_NAVIGATION` — navigation instructions from the `**View:**` line (text after the `—`)
- `APP_ROUTE` — prefer the **`Served URL`** line in the "Route / Layout" section (`plan-frontend-generic` emits the concrete live path there); use it verbatim. Only if that line is absent, fall back to deriving the URL from the route declaration in a framework-appropriate way — file-based routers from the file path (e.g. a page at `<frontend-app>/.../{{EXAMPLE_ROUTE}}/page.tsx` yields `/{{EXAMPLE_ROUTE}}`), config-based routers from the declared path. If the derived path has a dynamic segment (`[id]`, `:id`), substitute a representative value from the design's mock data (e.g. `/projects/1`) — the health check and validator can't load a literal `[id]`.
- `COMPONENT_LIBRARY_MAP` — the full "Component-to-Library Map" table (the binding contract for fidelity fixes)
- `ICON_MAPPING` — the "Icon Mapping" table (design icon → Source → concrete import) and any "Missing Icons" list, if present. Implement icons with exactly these imports; for a Missing-Icons entry, add the custom SVG the plan flagged (don't silently substitute a different library icon)
- `DESIGN_TOKEN_MAP` — the "Design Token Map" table (design token → project token, with tier), and the sibling `00-token-map.md` if the ticket cites one. This is the binding mapping for *existing* tokens — use it for token decisions rather than re-deriving the design→project mapping; `TOKEN_CHANGES` covers only the tokens to add. `TICKET_ID` — the ticket id (the reference/key in ticketing mode; the front-matter `id` in file-ticket mode), needed for the Phase 4a branch name
- `ACCEPTANCE_CRITERIA` — the full "Acceptance Criteria" checklist
- `DESIGN_VALIDATION_PARAMS` — the pre-authored design-validation invocation parameters (design file, app route, design navigation, **app navigation**, **viewport**, **theme context(s)**, focus areas), if present. `plan-frontend-generic` emits these under a "Design Validation Loop (if available)" heading inside "Verification", with the values under a "Validation parameters for this ticket:" sub-label — fuzzy-match on "Design Validation Loop" so the "(if available)" suffix doesn't break extraction. The optional `App navigation` (steps beyond the route to reach the view), `Viewport`, and `Theme context(s)` lines feed Phase 7a's invocation and the theme-context decision
- `TOKEN_CHANGES` — the "Token Changes" section, if present: a table, one row per proposed token, with columns matching the plan's header exactly — **Token (path → var)**, **Op** (`add` = additive semantic / `add-primitive` / `mutate`), **Value — base / <context>** (alias `{group.token}` or literal), **Source file(s)** (in `<token-source-dir>`), **Covers**, and **Auth** (`autonomous` or `requires build-time confirmation`). This is the **pre-authorization** the Phase 4d tier-gated rule reads: an `add` row with `Auth: autonomous` and a supplied value is the "ticket's plan implies it" signal — apply it without re-asking. A row marked `requires build-time confirmation` (every `add-primitive`/`mutate`), a needed token with **no** row, or an absent section all route to `AskUserQuestion`

### Phase 2 — Sanity-check the plan and comments against the codebase

Read **both** the plan in the ticket description **and any comments/review notes** on the ticket. Comments may contain bug fixes, corrections, additional requirements, or cross-ticket ordering notes. When a comment contradicts the plan, the comment wins.

**2a. Read and incorporate comments.**

- Bug fixes/corrections override the corresponding plan section
- Additional requirements add to the implementation checklist
- Unresolved questions get flagged via `AskUserQuestion`

In file-ticket mode there are no remote comments; treat any inline "Notes"/"Updates" the file contains the same way, then proceed.

**2b. Verify the plan against the codebase:**

- **Do the referenced/reused files still exist?** Glob/Read the paths the ticket cites as existing — reuse targets, critical-files, and any "depends on" file paths. (Do **not** existence-check the "Files to Create" section — those are new files that by definition don't exist yet; see the conflict check below.)
- **Do "Files to Create" conflict with existing files?** For each path in the ticket's "Files to Create" section (which `plan-frontend-generic` emits), flag any that already exist with different content — a conflict to resolve, not the expected greenfield case.
- **Do citations still resolve?** If the plan says "reuse `<some/shared/path>`", grep for it.
- **Has the surrounding code drifted?** Confirm function signatures and data shapes match.

**2c. Frontend-specific checks:**

- **Design file exists.** Verify the file at `DESIGN_FILE_PATH` exists. If it's been moved or deleted, hard stop and ask the user.
- **Component library spot-check.** Pick 2–3 entries from the `COMPONENT_LIBRARY_MAP` and verify the referenced `<component-lib>` components still exist at the library's installed path (resolved in Project configuration). Library updates since the ticket was filed could have renamed or removed components. If the library's components can't be enumerated at the installed path (minified/bundled/types-only, or a remote/CDN design system), spot-check via the library's exported type surface or import resolution instead, and treat the check as best-effort rather than a hard mismatch. (Skip if the project has no component library — the mapping's entries are all "custom".)
- **Route conflict check.** Verify the route path from the plan doesn't already exist with conflicting content.

If a mismatch is load-bearing, use `AskUserQuestion` to flag it. Minor drift gets silently adapted and noted in the PR body later.

### Phase 3 — Pick up the ticket

Two side-effects, in this order. The mechanism depends on the ticket mode.

**3a. Assign to the current user.** Skip if already assigned to the current user.

- **Ticketing-system mode:** set the assignee field to `CURRENT_USER_ID` via the system's edit/update operation.
- **File-ticket mode:** write the current user into the ticket file's `assignee` front-matter field (or no-op if the file format has no assignee concept).

**3b. Move to the in-progress status.**

- **Ticketing-system mode:** look up the available transitions/states dynamically (transition ids are typically not stable), find the one whose name matches `<in-progress-status>`, and apply it.
- **File-ticket mode:** set the ticket file's `status` field to `<in-progress-status>` (or no-op if the file has no status field).

Always look transitions up dynamically — never hard-code transition ids.

### Phase 4 — Create an isolated worktree and implement

**Every ticket gets its own git worktree.** This isolates implementation from whatever is happening in the main checkout.

**4a. Pick the branch name** per the configured branch convention. It must embed the ticket id so the PR step can recover it. Default template:

```
<prefix>_<ticket-id>_<kebab-slug-from-summary>
```

Example: `<prefix>_{{EXAMPLE_TICKET}}_signal-feed-page`. Keep the slug short (~30 chars). **Sanitize the slug** to `[a-zA-Z0-9_-]` only (the summary is untrusted input — e.g. a public issue title — and the branch/path are interpolated into shell commands; strip everything else, e.g. `tr -cd 'a-zA-Z0-9_-'`). The PR step re-extracts the ticket id from the branch name (e.g. via a regex matching the configured ticket-id pattern), so the id must remain literally present in the branch name.

**4b. Create the worktree** from the default branch, into the configured worktree root:

```bash
branch="<prefix>_<ticket-id>_<slug>"
worktree="<worktree-root>/${branch}"
mkdir -p "<worktree-root>"
git worktree add "$worktree" -b "$branch" "<default-branch>"
cd "$worktree"
```

If `git worktree add` fails because the branch or path already exists, don't clobber it — stop and ask the user. An existing worktree usually means a prior run they may want to resume.

**4c. Install dependencies.** Run `<install command>` in the worktree before any build/lint/test/dev commands — worktrees need their own dependency links.

**4c-bis. Build the token manifest (DTCG token systems only).** Before any implementation lookup, run `<token-build-command>` and derive the token manifest per the "DTCG token systems" section (variable → resolved value → theme context, joined with the JSON for tier/path/name metadata; plus the primitive-tier deny-list). 4d, Phase 5's conformance check, and Phase 7's `TOKEN_DRIFT` reverse-map all read it. Skip this step when the project has no DTCG/tiered token system.

**4d. Implement the plan's steps** in the worktree. Follow the ticket's "Component Plan" and "Files to Create" sections. Key rules:

- Follow the project's component-structure conventions — cite the project's component-rules doc if one was resolved in Project configuration (e.g. PascalCase files, named exports, the project's styling approach, co-located test files); otherwise apply sensible inline conventions.
- Follow the project's data-layer conventions — cite the project's data-layer-rules doc if one exists (e.g. fragment colocation, the project's preferred query hook, network-mocking for tests).
- Use `<component-lib>` components per the `COMPONENT_LIBRARY_MAP` — do not rebuild primitives the library provides. (If there is no library, build the "custom" entries the map lists.)
- Use the exact icon imports from `ICON_MAPPING`; for each "Missing Icons" entry, add the custom SVG the plan flagged rather than substituting a different library icon.
- All styling must reference the project's theme/token system — component props for library components, theme variables / utility classes in custom styles. Never hardcode hex colors or px values that duplicate what the token system already provides. Use `DESIGN_TOKEN_MAP` for which existing project token each design value maps to (don't re-derive the mapping the plan already made); `TOKEN_CHANGES` covers only tokens to add.
- **DTCG token systems:** consume only the tier the project's convention allows (typically semantic, never primitives) — look variables up in the token manifest, not by grepping generated CSS. The only edit path for token values is the DTCG JSON in `<token-source-dir>` followed by `<token-build-command>` (then rebuild the manifest, and if a dev server is running verify it serves the regenerated values — many watchers hot-reload generated CSS; restart it if values are stale); never edit `<generated-token-artifacts>`. When the design needs a value with no token: prefer the nearest existing semantic token. Creating a new token is **tier-gated, and the autonomous case is exhaustive**: an *additive semantic-tier token* may be added autonomously when the ticket's plan implies it and the design supplies the value — aliasing an existing primitive when one matches, literal-valued otherwise. The plan signals this through the ticket's **`TOKEN_CHANGES`** section (extracted in Phase 1): an entry with operation `add`, tier semantic, a name, and per-context value(s) is pre-authorized — apply it without re-asking, mechanically:

  0. **Idempotency preflight.** The same `add` row is carried in every ticket that consumes the token, so an earlier ticket may already have created it. Before adding, check `<token-source-dir>`/the manifest: if the token already exists with the same path, variable, and per-context values, it's a **no-op** (skip); if it exists but differs, **ask** (don't silently overwrite); only add when absent.
  1. **Parse the Value cell.** Split the cell on ` / ` into segments and strip any trailing parenthetical annotation (`(literal)`, `(alias…)`) — those are informational. The first segment is the base value, each later segment is a `<context> <value>` override (e.g. `dark #1c2430`). Within a segment, `{group.token}` is an alias to that token path; a bare hex/dimension is a literal. Never write the `/`, a context label, or an annotation into the JSON value.
  2. **Insert at the nesting the token path indicates** — `semantic.color.surface-info` goes under `semantic` → `color`, never at the JSON root (root insertion breaks the build tool's grouping/tier resolution). Ensure the token has a resolved `$type` (color/dimension/…): inherited from its group if that group declares one, else set `$type` explicitly on the new token.
  3. **Name it to match the promised variable.** When the project's rule is a vendor `$extensions` cssName key, set that key to the promised name **without the leading `--`** — the build tool adds the prefix, so writing `--surface-info` yields `----surface-info`. When names are path-derived (no key), the token path must derive to the promised variable.
  4. **For an alias, confirm context-stability against the manifest.** An alias inherits the aliased token's per-context resolution: if that token resolves to different values across the contexts the design needs (e.g. it's a primitive re-pointed in a transitional-debt dark block), the alias is **not** stable — emit explicit per-context overrides, or route to `AskUserQuestion`, rather than trusting it. The conformance check won't catch a wrong-but-non-empty dark value.
  5. **Run `<token-build-command>`, rebuild the manifest, and verify it contains the exact promised variable.** If not, fix the cause (extension name, `--` prefix, nesting) and rebuild; for a path-derived name the plan mis-guessed, align the consuming component CSS to the actually-emitted variable and note the correction in the PR body. Never leave `var(--x)` resolving empty — lint, test, and the conformance check won't catch it.

  Add the matching override for each theme context the entry lists, and note the addition in the PR body. If the project declares an alternate context (e.g. dark) but the row gives no override for a new **color or shadow** token, don't silently let it inherit the base value — confirm with `AskUserQuestion` that base-inheritance is intended (a light-only color usually looks wrong in dark and would still pass the non-empty smoke check). **Everything else asks**: a new primitive, a mutation of an existing `$value`, an ambiguous semantic name, a tierless token file (no semantic/primitive distinction to gate on), any `TOKEN_CHANGES` entry marked "requires build-time confirmation", or any touch of a documented transitional-debt block requires `AskUserQuestion`. A needed token with **no** `TOKEN_CHANGES` entry also falls here — ask rather than invent one.
- Translate design mock data into the project's data layer (e.g. GraphQL queries with fragment colocation, REST calls, or typed-client calls per the resolved data layer).
- Translate the design's client-side (`useState`) page navigation into the project's router (file-based or config-based routing).

All subsequent phases run from inside the worktree directory. Stay `cd`'d in there until the skill finishes.

### Phase 5 — Lint + test (deterministic gate)

Run from the worktree before attempting design validation. The code must compile for the dev server to work.

```bash
<lint command>
<test command>
```

These are the deterministic gates. If the project also defines a `typecheck`/`build` target, run it too. (Some projects have no standalone typecheck target — in that case lint + test are the gates.)

If any fails, fix the issues in the main thread (you own the code). Do not proceed to the dev server or design validation until they pass. If fixes take more than ~2 attempts, surface to the user via `AskUserQuestion`.

**Token-conformance check (single pass — only when the DTCG token settings were resolved; see "DTCG token systems").** A non-DTCG token file has none of the inputs below — for those projects the generic "no hardcoded values that duplicate tokens" rule in Phase 4d is the only token check. Token violations are mechanical, so this is a deterministic scan folded into this phase — **not** a validation loop like 5b/7, and it never spawns a subagent. Two preconditions: (a) **stage new files first** (`git add -A` in the worktree) — newly created component files are untracked and won't appear in `git diff <default-branch>` otherwise, so checks 2–3 would silently skip them; (b) this check reads the **token manifest**, so it requires a **fresh** one — if `<token-build-command>` failed and the manifest is degraded/stale (Phase 4c-bis / 4d), **skip the check** and note `TOKEN_CONFORMANCE: skipped — manifest unavailable` in the PR body rather than scanning against stale state (which would mis-flag correctly-added tokens). Three checks, scoped to files changed relative to `<default-branch>` (checks 2–3 additionally exclude `<generated-token-artifacts>`, whose hunks legitimately contain primitive definitions and raw values):

1. **Generated artifacts exactly reproducible, never hand-edited:** an artifact diff vs `<default-branch>` is *expected* whenever `<token-source-dir>` was edited through the sanctioned Phase 4d path — commit it. The violation is a **hand-edit**, detected as either an artifact change with no corresponding `<token-source-dir>` change, or artifact residue a rebuild doesn't reproduce: re-run `<token-build-command>` and require `<generated-token-artifacts>` to come out clean (no uncommitted residue). Revert hand-edits and route the change through the DTCG JSON instead.
2. **No primitive-tier consumption in new code:** extract `var(--name)` references (and bare `--name` re-declarations) from added lines and compare by **string equality** against the manifest's primitive-tier list — never substring grep (legacy names as short as `--r` make substring matching unusable). In a theme-prop / utility-class system where components consume tokens by name rather than `var()` (e.g. `color="p600"`, `bg="n800"`), also compare added prop/class values against the primitive names (strip the `--`). Tolerate pre-existing primitive usage in lines the ticket didn't touch (documented transitional debt) — only added lines count.
3. **No hardcoded token duplicates:** flag added color/dimension literals whose value matches a manifest entry; replace with the corresponding **consumable-tier (semantic)** variable. If the only manifest match is a primitive, do **not** swap in the primitive (that just trades a hardcode for a check-2 violation) — treat it as a "no semantic match" and route it through the Phase 4d token-creation rule. This check is best-effort (composite values and near-misses won't match exactly) — treat non-obvious hits as flags to review, not auto-fixes.

Fix violations inline, re-run `<lint command>`, and move on — one pass, no rounds.

### Phase 5b — Data-layer conformance loop (conditional, up to 3 rounds)

**This phase runs whenever the project has a data layer** (e.g. GraphQL with fragment colocation/codegen, or REST/OpenAPI/tRPC). **Skip this entire phase and jump to Phase 6 only when there is no data layer at all, or when no changed file contains a data operation** (computed in 5b-1). A missing conventions doc is **not** a skip trigger: when a data layer exists but no rules doc was resolved, still run the loop and fall back to the inline read-only pass described in 5b-3 (against whatever conventions the repo documents; if truly none, check only that data operations are typed — no `any` — and not duplicated, and note the thin coverage in the PR body).

Validate that created or modified components follow the project's data-layer conventions (for GraphQL: fragment colocation, fragment/operation naming, imports per the project's GraphQL rules, query/mutation tier boundaries; for other layers: schema conformance, typed-client usage, etc.) before proceeding to the dev server and design validation.

#### 5b-1. Identify target files

Only validate files that were created or modified in this ticket **and** contain data-layer operations. Skip this phase entirely if no files qualify. Use the resolved `<data-layer-detector>` from Project configuration — the literal `graphql(` below is just the GraphQL example; for REST/tRPC the detector is "files importing the client or calling the typed endpoints," etc. Anchor on the resolved detector, not the example token. Exclude generated code (the `<generated-code-dir>` resolved in Project configuration; drop that `grep -v` if the project has no generated-code directory) and test files.

```bash
# Stage new files first so untracked, just-created files are included in the diff.
git add -A
# From the worktree, find changed source files relative to the default branch,
# excluding generated code and tests, that contain data-layer operations.
# The extension set should match the resolved framework's source files, not just React's.
git diff --name-only "<default-branch>" -- '<frontend-app>/**' \
  | grep -E '\.(tsx?|jsx?|vue|svelte)$' \
  | grep -v '<generated-code-dir>' \
  | grep -v '\.test\.\|\.spec\.' \
  | while read f; do grep -l '<data-layer-detector>' "$f" 2>/dev/null; done \
  | sort -u
```

This produces a list of modified frontend source files (excluding generated code and tests) that contain data-layer operations. If the list is empty, log "No data-layer files modified — skipping data-layer validation" and jump to Phase 6.

#### 5b-2. Per-round structure

```
round = 1
while round <= 3:
  # Re-run lint/test if we made fixes (skip for round 1 — Phase 5 already passed)
  if round > 1:
    run <lint command> && <test command>
    if fail: fix, re-run lint/test

  invoke the data-layer conformance validator (see below)
  parse the structured report

  issues_count = count of Issues (not Warnings) across all files

  if issues_count == 0:
    break — validation passed

  if round == 3:
    surface residual issues to user via AskUserQuestion
    break

  implement fixes based on report
  round += 1
```

#### 5b-3. Invoke the data-layer conformance validator

Run a **read-only conformance check** scoped to the target file list from 5b-1. **Strongly prefer a separate read-only subagent so the check runs in an isolated context** — the implementer should not grade its own work. If the project provides a dedicated data-layer conformance validator (a subagent or skill — e.g. a GraphQL-conventions checker when the stack is GraphQL/Apollo, or the project's REST/OpenAPI/tRPC schema-conformance equivalent), use it. Pass it the file list and ask it to check each file against the project's data-layer rules.

Only when the environment provides no subagent/skill mechanism, fall back to an inline read-only pass against the project's data-layer-rules doc, noting that this loses context isolation (the implementer is reviewing its own output). Either way the validator must be **read-only** — it reports, it does not edit — and **MUST** return a per-file `STATUS: PASS | ISSUES_FOUND` line plus a summary containing an explicit issue count (e.g. `Issues found: N across M files`), so the loop can parse the exit condition deterministically.

#### 5b-4. Parse the report and decide

The report MUST include a per-file `STATUS: PASS | ISSUES_FOUND` line and a summary line with an explicit issue count (e.g. `Issues found: N across M files`); the loop parses that count as its exit condition. Since 5b-1 already excludes generated code and tests, only Issues in non-generated files count. Decision logic:

| Condition                            | Action                                                       |
| ------------------------------------ | ------------------------------------------------------------ |
| `Issues found: 0` (all files PASS)   | Exit loop — validation passed                                |
| Issues found, round < 3              | Fix issues in the main thread, re-run lint/test, re-validate |
| Round 3 reached with residual issues | Stop — surface to user via `AskUserQuestion`                 |

**Warnings are acceptable** — only Issues (clear rule violations) trigger fixes. Do not attempt to fix Warnings.

#### 5b-5. Implement fixes (main thread)

When fixing issues between rounds:

- Use the report's category and line hints to locate each violation
- Cross-reference the project's data-layer-rules doc for the correct pattern if the category isn't self-explanatory
- After fixes, re-run `<lint command> && <test command>` before the next validation round — fixes must not break compilation

#### 5b-6. Edge cases

- **The validator crashes or returns no output:** Treat as a failed round. Retry once. If it fails again, skip data-layer validation entirely and note the failure in the PR body — don't block the PR on a tooling failure.
- **All issues are in generated code:** This shouldn't happen (generated code is excluded in 5b-1), but if it does, treat as a pass.
- **A file was deleted between rounds:** Re-compute the target file list before each round to avoid passing stale paths.

### Phase 6 — Start the dev server

The design-validation step requires the frontend app running at the configured dev-server URL `<dev-server-url>`.

**6a. Check port availability** (`lsof` is the example mechanism; where it's unavailable — minimal CI containers, non-Unix dev envs — use an equivalent like `ss -ltnp`, `fuser`, `netstat`, or a `curl`-based liveness check):

```bash
lsof -i :<dev-server-port> -t 2>/dev/null
```

If something is already on the dev-server port (or the backend port, when the dev target starts one), bail with a message asking the user to stop it first. Do NOT kill another process's dev server. (This guards against *other* processes — when an instruction elsewhere says to **restart** your own dev server, e.g. after a token rebuild or a degraded server in Phase 7e, first `kill $DEV_SERVER_PID` to free the port, then repeat these Phase 6 steps; otherwise this check sees your own still-running server and bails.)

**6b. Start the dev server in the background:**

```bash
<dev command> &
DEV_SERVER_PID=$!
```

If the dev target transitively starts a backend/API service (resolved in Project configuration), that service also comes up on its port — both are needed when the frontend depends on the backend for data. Note its port for cleanup in Phase 8. (If the project has no local backend dependency, there is just the one dev server.)

**6c. Wait for the server to be ready.** Poll until the **specific app route** responds (not just `/` — many dev servers compile/warm pages on demand):

```bash
for i in $(seq 1 30); do
  http_code=$(curl -s -o /dev/null -w "%{http_code}" "<dev-server-url>${APP_ROUTE}" 2>/dev/null)
  if echo "$http_code" | grep -qE "^(200|307|308)$"; then
    echo "Dev server ready"
    break
  fi
  sleep 2
done
```

If the server is not responding after ~60 seconds (30 retries), kill the process and bail with the error. Common causes: port conflict, build error that lint/test didn't catch, or missing environment variables.

**6d. Store `DEV_SERVER_PID`** for cleanup in Phase 8.

### Phase 7 — Design validation loop (conditional, up to 3 rounds)

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

### Phase 8 — Kill the dev server

**Always runs, regardless of outcome.** Whether the skill succeeds, fails at a gate, or hits an error after starting the dev server, this cleanup must happen.

```bash
kill $DEV_SERVER_PID 2>/dev/null || true
# Only if the port is STILL held after killing our own PID, clean up an orphan we spawned.
# Guard the port-kill so it can't take out an unrelated process that happens to bind the port
# (matches our dev command; lsof is the example mechanism — substitute ss/fuser/netstat).
for pid in $(lsof -i :<dev-server-port> -t 2>/dev/null); do
  ps -o command= -p "$pid" | grep -q "<dev command>" && kill "$pid" 2>/dev/null || true
done
# Same guard for the backend port — only kill a process the dev command spawned, never a
# pre-existing shared backend/DB the user was already running on that port.
for pid in $(lsof -i :<backend-port> -t 2>/dev/null); do
  ps -o command= -p "$pid" | grep -q "<backend service command>" && kill "$pid" 2>/dev/null || true
done
```

This frees only the server *this run* started — consistent with Phase 6a's "do not kill another process's dev server." If the skill exits before Phase 6 (e.g. at a gate in Phase 1), this phase is a no-op. If the project has no backend service, only the dev-server port needs freeing.

### Phase 9 — Open the PR

Run this from inside the worktree directory (you should already be `cd`'d there from Phase 4b).

Open a change request (PR/MR) via the project's mechanism — the VCS CLI (`gh` for GitHub, `glab` for GitLab — note GitLab calls them Merge Requests via `glab mr create`, …) or the ticketing/VCS integration the project uses. If the project has a dedicated PR-creation skill, delegate to it rather than re-implementing commit/push/PR logic; otherwise drive the CLI directly. Honor the configured branch/reviewer/label conventions. Treat reviewers, labels, and a review-status transition as each "if the host supports it; otherwise omit." For a host with no change-request concept at all (push-only remote, email-patch flow), the terminal state is: push the branch and report the branch name + compare URL. The flow:

- Commit with the ticket-prefixed subject line (per the configured title-prefix pattern, if any)
- Push the branch (the ticket id is recoverable from the branch name per Phase 4a)
- Open the change request against the repo resolved from the git remote, with the configured title prefix (e.g. `<ticket-id>:`), label set (if the host supports labels), assignee, and reviewer(s) per the configured reviewer-selection rule (if any) — the rule must exclude the PR owner from being their own reviewer
- Move the ticket to the **review status** `<review-status>`:
    - **Ticketing-system mode:** transition/update the ticket to the review status (look the transition up dynamically).
    - **File-ticket mode:** set the ticket file's `status` field to `<review-status>` and report it.

Include this additional context in the PR body:

- **Data-layer validation status** from Phase 5b (e.g., "data-layer conformance: passed after 2 rounds", or "skipped — no data-layer files modified", or "1 residual issue after 3 rounds", or "skipped — project has no data layer")
- **Design validation status** from the final round (e.g., "design validation: partial — 2 minor cosmetic issues remaining", or "manual checklist — no automated tool available")
- **Design file reference** so reviewers can compare (e.g., "Design: `{{EXAMPLE_DESIGN_FILE}}` > {{EXAMPLE_ROUTE}} view")
- **Residual discrepancies** (if any minor/cosmetic issues remain, list them)

Because the branch was pre-created with the ticket id, the PR step does the full create flow (new feature branch, no open PR yet).

### Phase 10 — Report back

Brief summary to the user (~8 lines):

- Ticket id + URL (or file path, in file-ticket mode)
- PR URL
- **Worktree path** — so the user knows where the checkout lives for iterating on review feedback
- **Data-layer validation summary:** passed/failed, rounds completed, residual issues (if any), or "skipped" (no qualifying files / no data layer)
- **Design validation summary:** final STATUS, rounds completed, residual discrepancies (if any), or "manual checklist"
- Acceptance criteria summary — a **self-assessment** from the automated gates (lint, test, token-conformance, design validation), not a full verification. Explicitly flag criteria nothing automatically checked (e.g. accessibility, "co-located test files exist") as needing manual review rather than implying they passed
- If `AskUserQuestion` was used during the run, a one-liner noting what got decided

Leave the worktree in place — PR review rounds frequently need it back.

---

## Gotchas

- **All three Phase 1 gates are hard stops.** Status, assignee, and Design Reference presence. Don't prompt for confirmation — bail immediately on failure.
- **Don't hardcode the user.** Resolve identity dynamically in Phase 0 from the configured source. In ticketing-system mode, don't fall back to `$USER`, git config, or the harness email — those are the OS/VCS identity. Only file-ticket mode uses the git/config identity.
- **Ticketing is pluggable.** Map every ticketing concept to both modes: gating reads a status/assignee field (or front-matter `status` / no-op for files), pick-up assigns + transitions (or updates front-matter / no-op), and completion transitions to the review status (or updates front-matter + reports).
- **Always work in the worktree.** After Phase 4b, every file path must resolve under `<worktree-root>/<branch>/`. Double-check with `pwd` or `git rev-parse --show-toplevel`.
- **Don't clobber an existing worktree.** If `git worktree add` fails, stop and ask.
- **The branch name must embed the ticket id.** The PR step recovers the id from the branch name — never strip it.
- **Transitions are looked up dynamically.** Don't hard-code transition/state ids; query the system for them.
- **Parse the ticket payload per the configured provider.** Field locations and casing differ across JIRA / Linear / GitHub Issues / file front-matter — normalize before comparing.
- **Always read ticket comments, not just the description.** Comments contain plan-review feedback and corrections. When a comment contradicts the plan, the comment wins. (File tickets: treat inline notes the same way.)
- **Port conflicts are the #1 failure mode.** Another dev server (or backend) on the configured port will silently cause failures. Always check before starting; never kill another process's server.
- **The dev target may start a backend too.** If the project's dev command transitively starts a backend/API service, Phase 8 must kill both ports.
- **Resolve the design file path against the worktree root**, not the main checkout. Design files typically live in the repo alongside the app code.
- **Design validation is expensive.** Each round can spawn capture + comparison subagents plus a browser session. The 3-round cap is a mandatory cost control — don't exceed it.
- **Always run lint/test before design validation.** If the code doesn't compile, the dev server serves a broken page and a whole validation round is wasted on a build error.
- **Hit the actual app route in the health check, not just `/`.** Dev servers often compile/warm pages on demand; the root might respond while the specific route is still compiling.
- **The Component-to-Library Map is the contract.** If design validation reports a component fidelity issue, check the map first — the implementation should use exactly the library components specified.
- **Kill the dev server on every exit path.** Whether the skill succeeds, fails at a gate, or errors after Phase 6, the dev-server port (and backend port, if any) must be freed. Structure cleanup to run on every exit path.
- **Data-layer validation is conditional and scoped.** Run it only when the project has a data layer with conventions, and only against modified files containing data operations — never the whole source tree, and never generated code. Both the data-layer loop and the design loop are capped at 3 rounds.
- **Warnings don't block.** Only Issues (clear rule violations) trigger fix-and-retry rounds. Warnings are informational.
- **Generated token artifacts are read-only ground truth.** In a DTCG setup the generated CSS is what the manifest and conformance checks read — and the one thing never edited. The editable surface is the DTCG JSON; the bridge is `<token-build-command>`.
- **Compare composite tokens as resolved CSS, never as JSON.** Shadows, typography, and font stacks are objects/arrays in DTCG source but strings in the build output; value comparisons against the JSON shape will always miss. The same goes the other way: token build tools can corrupt multi-layer arrays under naive deep-merge — another reason not to re-implement resolution.
- **`$type` inherits from groups.** A per-token scan for `$type` misses tokens that inherit it from an ancestor group — read types through the manifest, not by spot-checking leaves.
- **Vendor `$extensions` keys are project-specific.** The CSS-name extension (e.g. `com.<project>.cssName`) varies per project and may be absent (path-derived names). Detect, don't assume.
- **Token edits don't reach the page by themselves.** After editing DTCG JSON, run `<token-build-command>`, rebuild the token manifest, and verify the dev server serves the regenerated values (many watchers hot-reload generated CSS; restart the server if values are stale) — otherwise Phase 7 validates against stale tokens.
- **Transitional-debt blocks are read-only.** Documented legacy carve-outs (e.g. dark mode re-pointing primitives because old CSS consumes them directly) are tolerated where untouched and never "cleaned up" opportunistically — that's its own ticket.
- **Don't re-enter plan mode.** The plan is already written and lives in the ticket. Your job is execution, not re-planning.
- **`AskUserQuestion` is for real decisions**, not status updates. Don't interrupt the user to say "starting implementation now."
- **The planning counterpart is `plan-frontend-generic`.** A ticket without a Design Reference section wasn't authored by it — route non-frontend tickets to a non-frontend implementation skill/agent.
