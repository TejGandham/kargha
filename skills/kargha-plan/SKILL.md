---
name: kargha-plan
description: Analyze a problem/feature description and/or a design mock or non-functional prototype and synthesize a validated binder of work items for ad-hoc orchestration; stack-agnostic (frontend, backend, CLI, data, library/SDK, IaC, mobile, ML, docs — UI is one stack among many); emits .kargha/binders/<slug>.json. Trigger phrases: "plan this with kargha", "synthesize a binder", "break this work into a binder", "kargha-plan this feature".
---

kargha-plan is **keel-refine light**: it ingests intent and — without fail — synthesizes a binder. Give it a problem or feature description; optionally attach a design mock or non-functional prototype. It asks a minimal set of questions, runs a synthesis subagent to draft the binder, and commits on an explicit "commit" verb. The output lands at `.kargha/binders/<slug>.json` and is validated by `validate_binder.py` before any build run.

## How this skill adapts to your project

kargha-plan is **stack-agnostic**. It plans frontend, backend, CLI, data pipelines, libraries/SDKs, IaC, mobile, ML, and docs work in the same way — UI is one stack among many, not the default.

**What it drops from keel-refine:**

- Bootstrap-gate preflight → a light, never-blocking repo detect (see Project configuration below).
- Multi-binder partition → when the scope spans multiple natural binders, the skill suggests a breakdown into one binder's work items and plans one binder per run (V1).
- Roundtable decomposition → synthesis runs as a single subagent, not a panel.

**What it keeps:**

- Intent ingestion widened to design exports: when the input is a Claude Design or runtime-JSX export, the UI-analysis path applies (component inventory, token map, icon map). This is a **stack-specific aside**, not a precondition for the whole skill.
- The synthesis subagent that drafts the binder from ingested intent.
- A minimal interview loop — asks only what it cannot detect.
- Commit-on-verb: the binder is presented as an editable card and committed when you say "commit."

**What it replaces:**

- The mandatory per-card walk → smart-surfaced review: the synthesis subagent flags items that warrant human attention (`surface.flagged` + `surface.signals`); unflagged items don't require a walk.

## Project configuration (resolve once, never blocking)

Resolve in order: **explicit user input → detect from the repo → ask the user** (batch all unknowns into one question). Skip what you can detect. When detection conflicts with the project's stated stack, the stated stack wins — a repo can be mid-migration.

| Setting | What it is | How to resolve |
|-|-|-|
| **Stack** | The primary tech domain (frontend, backend, CLI, data, IaC, mobile, ML, docs, mixed) | Detect from repo layout, package manifests, and language files; ask when ambiguous |
| **Toolchain commands** | lint / test / build / typecheck invocations | Detect from package scripts, task runners (npm/pnpm/yarn, Makefile, Nx, Turborepo, Cargo, Poetry, etc.) |
| **CI-facing checks** | The subset of commands that gate CI (used as oracle `command` values) | Detect from CI workflow files; if absent, use the toolchain commands |
| **Env command** | The command that starts the dev/test environment (feeds `env_contract.command`) | Detect from package scripts or a `docker-compose.yml`; ask if not found |
| **Repo direction docs** | Architecture docs, ADRs, or decision records | Detect: `ARCHITECTURE.md`, `docs/architecture/`, `docs/decisions/`, `adr/`; cite only filled docs (skip placeholder templates); ask when context is thin |
| **Project rules** | Conventions the repo documents (lint configs, contributor guides, rules files) | Detect; verify the doc has real content before citing |
| **Repo policy** | CI/branch/deployment policy (only when in planning scope) | Read root/area `AGENTS.md`, workflows, and CI docs; load [references/ci-policy.md](references/ci-policy.md) and [references/policy-yagni.md](references/policy-yagni.md) |

Resolved values feed the binder's `design_facts.stack`, `env_contract`, and each oracle's `command`. Record them — every later phase references them.

### UI / design-token annex (conditional)

When the resolved stack has a design or token surface, also resolve:

- **Component library:** detect from package deps and import statements; may be none.
- **Icon libraries:** detect from deps/imports; may be none.
- **Token/theme system:** detect a theme object, CSS custom properties, a design-tokens file (incl. W3C DTCG JSON — see [references/dtcg-tokens.md](references/dtcg-tokens.md)), a utility-class config, or plain CSS.

When the **token system** is a W3C DTCG file (JSON leaves carrying `$value`/`$type`), resolve the additional DTCG-only settings in [references/dtcg-tokens.md](references/dtcg-tokens.md) so the binder's `token_manifest` and each work item's `token_changes` can carry tier-correct, build-actionable guidance.

Resolved UI values feed the binder's `design_facts` and each work item's `component_map`, `icon_map`, and `token_changes` (see [references/binder-reference.md](references/binder-reference.md) — these fields are UI-optional and omitted when the stack has no design surface).

## Workflow

### Phase 0 — Ingest intent (light gate)

Accept a problem or feature description as the only hard requirement. Optionally accept a path to a design mock or non-functional prototype. Some statement of intent is required — if the user provides nothing, ask once.

**Repo detect (light, never blocking).** Check for a recognizable repo layout to resolve the binder's landing path and later feed Phase 1. Never fail on an unfamiliar structure.

**UI-format gate (conditional).** When the input includes a design file path, check whether it is a Claude Design or runtime-JSX export: verify that `.jsx` siblings exist or the HTML contains a `<script type="text/babel">` block. If the check fails, tell the user what was found and ask them to re-export or point at the `.jsx` sources. Only run this check when a design file is provided; a plain text description has no format to gate.

**Resolve the binder location.** In order: explicit user input → detect an existing `.kargha/binders/` directory in the repo → default to `.kargha/binders/<slug>.json`. Ask the user only when you cannot determine `<slug>` from the intent.

---

### Phase 1 — Best-effort repo + stack understanding (Explore subagent)

Use an **Explore subagent** OR an inline read-only pass to survey the repo.

**Subagent brief:**

Survey the repo at `<root>`. Identify the primary tech domain (frontend, backend, CLI, data pipeline, library/SDK, IaC, mobile, ML, docs, or mixed). Identify the toolchain: how the project lints, tests, builds, and type-checks (look at package scripts, task runners — npm/pnpm/yarn, Makefile, Nx, Turborepo, Cargo, Poetry, Gradle, etc. — and any CI workflow files). Determine the env command: what starts the dev or test environment. Read any architecture, ADR, or decision docs you find (`ARCHITECTURE.md`, `docs/architecture/`, `docs/decisions/`, `adr/`, `AGENTS.md`) — cite only docs with real content; skip placeholders. Read project rule and contributor convention files when present.

Report:
1. Resolved stack (one phrase, e.g. "Python/FastAPI backend + Postgres").
2. Toolchain commands for lint, test, build, and typecheck — with the exact invocations.
3. CI-facing checks: the subset of toolchain commands that gate CI (from workflow files, or fall back to the toolchain commands).
4. Env command.
5. Conventions and rules the repo documents.
6. Architecture or decision-record notes relevant to the stated intent (if any docs exist).

When context is thin on a point, say so — the main thread will interview the user for the missing piece.

Report with `file:line` citations.

**UI/design-token annex (conditional).** When the resolved stack has a design or token surface, also report:

- Component library (from package deps and import statements; may be none).
- Icon libraries (from deps/imports; may be none).
- Token/theme system: detect a theme object, CSS custom properties, a design-tokens file (including W3C DTCG JSON — see [references/dtcg-tokens.md](references/dtcg-tokens.md)), a utility-class config, or plain CSS. When the token system is DTCG, also resolve the additional settings in [references/dtcg-tokens.md](references/dtcg-tokens.md) so the binder's `token_manifest` and each work item's `token_changes` can carry tier-correct, build-actionable guidance.

---

### Phase 2 — Synthesize the binder (synthesis subagent; main thread owns judgment)

Decompose the stated intent into work items. Do not delegate this judgment — the synthesis subagent drafts; you review and own the output.

**Subagent brief:**

Given the intent `<intent>` and the repo survey from Phase 1, draft a binder JSON that conforms to [references/binder-reference.md](references/binder-reference.md).

For the binder level, populate:
- `slug` (kebab-case, derived from the feature name)
- `motivation` (one sentence)
- `scope.included` and `scope.excluded`
- `design_facts.source` (path to the design, or null) and `design_facts.stack` (from Phase 1)
- `env_contract.command`, `env_contract.supports_isolation`, and `env_contract.isolation_params` (from Phase 1)
- `token_manifest` only when the stack has a token system

For each work item, set:
- `id` (kebab-case, unique)
- `title` (short human label)
- `estimate` (`S`, `M`, or `L`)
- `depends_on` (IDs of items that must land first)
- `contract` (the open-shape interface this item exposes or consumes — be specific; vague contracts produce unverifiable oracles)
- `oracle` per [references/definition-of-done.md](references/definition-of-done.md): a real CI-facing check oracle (`type`, `command`, `assertions`) OR an explicit opt-out (`opt_out: true`, `reason: "…"`). The floor is compile + type-check + lint. Use opt-out only when a genuinely better check already exists and recording it here would be redundant — always provide a reason.
- `serialize: true` and `shared_resources` for items that must not run in parallel (e.g. DB migrations, lock-file changes).
- UI-only fields (`design_reference`, `component_map`, `icon_map`, `token_changes`) only when the stack has a design surface. Omit them entirely for backend, CLI, data, and other non-UI stacks.

**Oracle traceability rule.** Each oracle assertion must be traceable to the item's `contract`. If an assertion references a field, shape, or behavior the `contract` does not declare, flag the gap now — emit the work item with a note that the contract needs expanding rather than writing an assertion that cannot be verified.

Return the draft binder JSON.

---

After the subagent returns, review the draft. Check:
- Every work item has an `id`, a `contract`, and an `oracle`.
- `depends_on` references resolve to real IDs in this binder (no dangling refs).
- Oracle assertions trace to the item's contract.
- `serialize`/`shared_resources` are set for any items that write shared state (migrations, lock files, config that multiple items touch).
- UI fields are present only on items with a UI surface.

Fix gaps in the main thread before proceeding.

---

### Phase 3 — Smart-surfaced review (the one human-in-the-loop point)

Per [references/smart-surfaced-review.md](references/smart-surfaced-review.md): compute the seven boundary signals for each work item and write `surface { flagged, signals }` into the binder. When a signal cannot be computed yet (no diff, no path conventions), record `not-computed:<signal-name>` in `surface.signals` rather than giving a clean pass.

Present the results. Tell the user which items are flagged and why. Offer three options:

- **Review all** — walk every work item.
- **Review flagged only** — walk only surfaced items.
- **Accept as-is** — proceed without a walk.

Do not surface oracle details for routine, unflagged items — keep the list short. Front-load any decisions here so the deliver and build steps can run hands-off.

---

### Phase 4 — Cost education

When the binder contains many work items or several large (`L`) estimates, tell the user plainly: this scope will take time and real money before anything tangible lands. Suggest a smaller first slice — the items with no `depends_on` that form the first wave — as a lower-risk starting point.

Educate; do not forbid. If the user wants to proceed with the full scope, move on.

---

### Phase 5 — Emit, validate, and commit

**Write the binder** to the resolved location (`.kargha/binders/<slug>.json`).

**Validate it.** Run:

```
uv run skills/kargha-plan/scripts/validate_binder.py --binder <path>
```

Do not proceed on a validation failure. Fix the binder and re-validate until it passes.

**Single-work-item hatch.** A binder with exactly one work item is eligible to skip the deliver phase and go straight to build. Tell the user this option exists; do not make the choice for them.

**Commit on the `commit` verb.** Present the binder as an editable card. Commit only when the user says "commit." The binder is read-only to all subsequent build steps once committed.

---

### Phase 6 — Report back

Tell the user:

- Binder path and the total work-item count.
- Work item IDs in dependency order (topological sort).
- The dependency chain.
- Surfaced items: IDs and the signals that triggered review.
- Opted-out items: IDs and recorded reasons (from the validator's opt-out summary).
- The eligible first wave: items with no `depends_on` (these can start immediately).

---

## Gotchas

- **The binder is always synthesized.** kargha-plan never exits early with a plan outline or a list of tickets. The output is a validated `.json` binder or nothing.
- **UI fields are conditional, not universal.** `design_reference`, `component_map`, `icon_map`, and `token_changes` belong only on items with a UI surface. Emitting them on a migration, a CLI command, or a data pipeline item is a schema error — the validator will catch it, but don't write it in the first place.
- **The oracle is a real CI-facing check.** It is not a self-grading statement ("implementation looks correct") or a description of what the item does. It is the command you'd want CI to run and the assertions you'd want to confirm.
- **Opt-out is explicit and recorded.** There is no silent opt-out. Every `opt_out: true` requires a `reason`. kargha reports opted-out items after every run so nothing slips through unnoticed.
- **Don't delegate synthesis judgment.** The synthesis subagent drafts; the main thread reviews, corrects, and owns the binder. Cross-referencing contracts, oracle traceability, and dependency order requires judgment — do not hand that off.
- **Validate before commit.** A binder that fails `validate_binder.py` is not a valid binder. Fix it before presenting it to the user for commit.
- **One binder per run in V1.** When the scope spans multiple natural binders, suggest a breakdown and plan one binder this run. Multi-binder partitioning is not supported in V1.
- **The binder is read-only once committed.** Build steps read the binder; they do not modify it. A build step that tries to rewrite its own work item's oracle or estimate is corrupting its own governance.
