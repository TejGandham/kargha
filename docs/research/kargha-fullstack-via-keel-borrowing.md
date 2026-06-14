# kargha ↔ KEEL — Thought Experiment B: Borrowing KEEL/karta to make kargha full-stack

> **Status: thought experiment / exploratory analysis — NOT a committed plan or active work.** Recorded 2026-06-13.
> **Method:** 6-reader surface-map workflow → 7-model roundtable (deliberate) → 6-skeptic adversarial verification against source. Findings below reflect the verification-corrected synthesis.
> **Companion:** [`kargha-in-keel-integration.md`](kargha-in-keel-integration.md) (the reverse direction).
> **User constraints honored:** (1) do NOT port KEEL's **Binder** — kargha keeps its self-contained ticket model; (2) kargha stays a **skill plugin**, not an orchestrated agent framework.

## Question
Which KEEL/karta mechanisms should kargha (a frontend-only skill pipeline) **borrow** to become a safe full-stack tool (backend APIs, data, persistence, auth)?

## Bottom line
Yes — kargha can become full-stack *while staying a skill plugin*, by importing KEEL/karta **disciplines** (not its orchestration) as ticket sections, build phases, and one new read-only gate skill. The Binder is **not** needed: a per-ticket `contract` + a project-level invariants registry recover the coherence the Binder provided. The one genuine strain is **globally-ordered stateful resources (DB migrations)** — that needs serialization + test-DB isolation, not a Binder.

## What to import (tiered) — the backend safety floor imports as INDIVISIBLE bundles; `layer` field selects which activate (frontend path needs none)
- **Safety:** read-only **safety-auditor** scanning the worktree diff against a project **Domain Invariants** registry (auth/SQL/secrets/GET-immutability/idempotency), **fail-closed** if unconfigured + **KARTA-GUARD/PLACEHOLDER** write-time human-OK halts + the verbatim refusal protocol.
- **Coherence:** per-ticket open-shape **`contract`** (backend ticket publishes route+auth+response; consumer verified against it) + **`depends_on` as a gate** + contract-conformance.
- **Quality floor:** real **green gate** (rejects no-op commands, fail-closed) + **bones-clean admissibility** (cuts only in leaf code, never data model / contracts / auth) + **declared-debt markers + scanner**.
- **Disposition:** **spec-first** + per-assertion classification (inspection-verifiable vs execution-required → test-or-`KARTA-DEFER`).
- **Plus:** a structured **`oracle`** ticket section (type ∈ unit|integration|e2e|smoke + assertions), a **`layer`** field, a backend **design brief** mode in kargha-plan.

**Do NOT import (would clone KEEL):** the Binder/resolver/`resolved-work-item.json`, `routing.json` gate-state, the SPEC-SUSPECT→refine loop, arch-advisor + the multi-lens panel, landing-verifier-as-agent, handoff dirs/WI##/JSON-Pointer addressing, karta-drive's depth-2 sequencer.

## Resolved judgment calls (T1–T6)
- **T1 — stays a skill.** The line is *persisted inter-run state + multi-agent scheduling*, not prompt length or whether it uses subagents OR host workers. Full-stack does not cross it — but pushes it to the edge (see depends_on correction).
- **T3 — spec-first**, not RED→GREEN (test-first needs two mutator personas = orchestration inside a skill). Preserves spec>test>code via classification ordering + green-gate.
- **T2 — bundles are indivisible.** Half-importing is strictly worse than no backend support.
- **T4/T6 — 4 skills:** `plan` (extended + a plan-time coherence-review subagent OR host worker), `build` (layer-dispatcher), **`verify` (NEW** read-only behavioral/safety gate; behavioral analog of `validate`; auto-invoked by build), `validate` (unchanged, visual). The gate must be independent of the implementer.
- **T5 — pairwise coherence suffices, with two additions.** A **project Domain Invariants registry** gives global rule enforcement without a Binder (*the invariant is the global mechanism; the Binder was just a delivery vehicle*). Gaps: plan-time transitive consistency → a coherence-review subagent OR host worker; globally-ordered migrations → below.

## Corrections the adversarial verification forced
| Claim | Verdict vs source | Correction |
|-|-|-|
| GUARD/PLACEHOLDER can be **deterministically** script-enforced via a PreToolUse hook | **refuted** | KEEL's own safety hook is **advisory (exit 0, never denies)**. The floor is **prompt-discipline + the read-only auditor gate**, not a hard interceptor. A plugin may add an opt-in Bash tripwire for syntactically-obvious destructive commands as coarse defense-in-depth. Drop "deterministically enforced." |
| build adds a backend branch, **stays near current size** | partial | No `<500` line target actually exists in the repo. Several phases are frontend-inlined (Gate 3, Phase 6 dev-server/health, Phase 10 report). Reframe as a **layer-dispatch refactor**: extract frontend specifics into `references/frontend-build.md` mirroring `references/backend-build.md`, leaving a thin resident spine. Single-skill/no-orchestrator holds. |
| `depends_on`-gate is a **build-only, free** change | partial | Split it. **Ordering** gate is build-only but needs build to read sibling tickets + manifest — new cross-ticket reading in a deliberately single-ticket skill (mild orchestration creep). **Merge-state** gate isn't in plan's emitted data → needs a runtime git/forge probe. |
| kargha-verify is a **clean swap** of validate's two subagents OR host workers | partial | Only the ~20-line skeleton (collector → fresh-context judge → report → caller decides) transfers; ~95% by volume (Phase 0/1, both prompt bodies, output schema) is **net-new**. Budget it as a near-from-scratch skill; real design work = the safety-capture contract + the invariant/oracle judge schema. validate's "independence" is only contextual (data pasted into a fresh prompt). |
| debt **scanner ships verbatim** | **confirmed** | Stdlib-only, zero KEEL deps, runs over any tree. Two cosmetic tweaks: invoke as plain `python3` (relax 3.14 `requires-python`); reword KEEL-specific CTA strings. |
| DB-migration **global-ordering gap** is real | **confirmed (strengthened)** | kargha issues no migrations itself, but Phase 5 tests + the Phase 6 backend hit a **shared user-managed DB**; parallel worktrees collide. Because depends_on is non-binding today, foundation tickets don't serialize. Full-stack needs **both** the depends_on gate **and** ephemeral per-worktree DB isolation (DATABASE_URL-per-worktree / transactional rollback / container-per-ticket). |

## Named gaps and fixes
- **Bootstrap deadlock** — fail-closed safety means a repo with no invariants registry can't build any backend ticket. Ship a **foundation/readiness ticket** first (invariants, green-gate command, test-DB isolation, contract locations).
- **Plan-time coherence** — a read-only review subagent OR host worker in kargha-plan (the only point all sibling tickets are visible) catches transitive schema/auth inconsistencies.
- **Migrations** — serialized foundation tickets + ephemeral test-DB isolation.
- *(Optional)* an append-only contract/surface index for collision detection; a `kargha-audit` skill for invariant-registry decay.

## Honest scope boundary
Pairwise `contract` + `depends_on` + a project invariant registry covers the majority of full-stack work without a Binder. The exception is **globally-ordered stateful resources** (migrations, shared schema) — there, the self-contained-ticket model strains, and the answer is **serialization + isolation**, not a global plan artifact. And the safety floor is honestly **prompt-discipline + read-only gate**, weaker than a deterministic interceptor — acceptable for a skill, but state it plainly.
