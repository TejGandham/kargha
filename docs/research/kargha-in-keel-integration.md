# kargha ↔ KEEL — Thought Experiment A: Using kargha inside KEEL

> **Status: thought experiment / exploratory analysis — NOT a committed plan or active work.** Recorded 2026-06-13.
> **Method:** 6-reader surface-map workflow → 7-model roundtable (deliberate) → 6-skeptic adversarial verification against source. Findings below reflect the verification-corrected synthesis.
> **Companion:** [`kargha-fullstack-via-keel-borrowing.md`](kargha-fullstack-via-keel-borrowing.md) (the reverse direction).

## Question
How can the **kargha** frontend skill-pipeline (plan → build → validate; a Claude Code OR Codex-compatible skill plugin) be used effectively *inside* the **KEEL** SDLC framework (and its lean **karta** lane)?

## The situation
kargha and KEEL are two pipelines that overlap on frontend. The overlap is **incompatible by construction** on building/publishing but **complementary** on design-intelligence and visual evidence.

| kargha skill | KEEL analog | Verdict |
|-|-|-|
| kargha-plan | `/keel-refine` + backlog-drafter + frontend-designer | Redundant decomposition, additive intelligence. KEEL already turns a design export into work units — as a JSON **Binder** (single input canon, P6 authority), not a ticket store. kargha-plan's component-to-library / icon / **DTCG** maps have no KEEL equivalent (KEEL has zero DTCG concept; its prototype manifest is deliberately thin per P4). |
| kargha-build | keel/karta-pipeline + karta-drive worktrees + keel-submit | Most incompatible. Monolithic build+auto-PR vs KEEL's split agent-pipeline (TDD→review→landing) + **repo-local-done, PR deferred to keel-submit**. kargha-build's auto-push violates karta-drive's "never pushes." |
| kargha-validate | landing-verifier served-bundle drive | Genuine gap-filler. KEEL has no running-view-vs-prototype comparator; the lean lane *defers* browser evidence to `KARTA-DEFER`. |

## Verdict: B + A (re-scoped) + D; reject C; lane-aware
- **B — kargha-plan as a `/keel-refine` mapping-donor (adopt).** Run kargha-plan unmodified to a throwaway temp dir; a `/keel-refine` adapter parses the ticket files and lifts the maps into the Binder + WI## backlog; the DTCG Token-Changes are authored into `docs/design-docs/ui-design.md` by frontend-designer (kargha *donates* proposed values). Ticket store discarded. Token governance gets teeth by becoming `oracle.assertions[]` → real tests (P6: tests>code). DTCG home must be the authored `ui-design.md`, **not** a new Binder field (`binder.schema.json` is `additionalProperties:false`; NORTH-STAR.md forbids schema changes).
- **A — kargha-validate as KEEL's served verify command (adopt, re-scoped).** Wires into `landing-verifier`'s served-bundle Stage-4 drive (fires when `oracle.type ∈ {e2e,smoke}` + a served command is configured; no new agent/stage).
- **C — kargha-build (reject as a drop-in).** Monolithic build + auto-publish bypasses KEEL's split TDD/review/landing apparatus + the repo-local-done / keel-submit boundary. Its salvageable pieces are already covered by A+B.
- **D — honest boundary (adopt).** kargha stays standalone for non-KEEL repos. Add a guardrail: kargha-build/-plan should detect a KEEL repo (`.keel/`, `schemas/binder.schema.json`) and HALT with a CTA to use `/keel-refine` then `/keel-submit`.

## The deep tension, resolved
Is kargha-validate a *richer realization* of KEEL's structural served-bundle reviewer, or a *smuggling-back* of the golden-screenshot / prototype-as-authority mechanism KEEL adversarially killed (P4/P5/P1)? **Richer realization** — provided its PASS/FAIL is grounded in `oracle.assertions[]` (the 4-axis fidelity checklist + token-drift vs the authored token spec), **not** raw "matches the mock." The committed prototype is a *capture comparand*; the **oracle is the spec**. A prototype-vs-app delta not covered by an oracle predicate → evidence + a `/keel-refine` CTA to add a predicate, never a gate failure.

## Corrections the adversarial verification forced
| Assumption (roundtable) | Verdict vs source | Correction |
|-|-|-|
| validate plugs in as an **advisory** reviewer | wrong word | KEEL has **no advisory channel**; as the served command it's the **binding** served verdict (VERIFIED-on-pass / HALT). It IS authoritative over the served surface — but oracle-grounded (above). Only `landing-verifier` runs it (spec-reviewer has no Bash; "landing-review" is an orchestrator step, not an agent). |
| validate "stores nothing, no change needed" | partial | Loop/fix/exit-bar live in the caller (good), but validate writes 4 capture files to repo-relative `.playwright-cli/`, cleaned only best-effort. Fix: abs temp path / gitignore / trap teardown. |
| prototype-serving is the **sharpest unresolved risk** | confirmed-resolved | KEEL *already* `mv`'s the prototype into `docs/exec-plans/binders/<slug>/prototype/`, commits it, designates it the "canonical, locally-runnable home"; implementer is forbidden from reading that in-repo path. validate serves from a local file. **P3-safe and P4-clean — worry evaporates.** |
| **C** "CANNOT be made admissible" | 2 of 3 reasons false | Branch naming is configurable (only rule: embed the ticket id, which `keel/WI##-<slug>` satisfies). File-ticket status gates are no-ops. The *one* real blocker is Phase 9's unconditional push. Honest framing: "rejected as a drop-in; admissible only via a Phase-9 amputation" — and even then it duplicates karta-pipeline while bypassing its gate independence. |
| DTCG home exists somewhere | confirmed | KEEL has zero structured token system; home must be the authored `ui-design.md`, not a Binder field. |

## Lane split
- **keel lane (full):** B + A.
- **karta lean lane:** B only (refine is upstream of lane choice). A is incompatible — the lean lane drops the served stage and karta-spec-reviewer never runs a browser. Optional knob: let karta-spec-reviewer invoke validate as its browser-evidence source instead of forcing a `KARTA-DEFER`. Default off.

## Knobs (two-org)
`design_token_format: prose | dtcg-donor` (Salesforce/Shopify vs a no-governed-tokens app). Served visual evidence is *derived* (oracle e2e/smoke + harness), not a knob. `karta_visual_evidence: off | validate`.

## Recommended build order
1. D + the KEEL-repo-detection HALT guardrail in kargha (cheap, prevents footguns).
2. B — the `/keel-refine` ticket-file adapter + the `ui-design.md` token-donation path.
3. A — the served-command adapter with oracle-grounded re-scoping + artifact hygiene.
4. (optional) the karta browser-evidence knob.
