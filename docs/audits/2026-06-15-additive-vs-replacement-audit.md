# kargha Additive-vs-Replacement Audit

**Date:** 2026-06-15
**Branch:** `kargha-orchestration` vs merge-base `main` (`3eec67ab7ec8a90a0a44b3f37bfab5d2748c4449`)
**Question:** Did the orchestration augmentation *replace or thin* any pre-existing kargha capability, or is it strictly *additive*?
**Method:** 15-agent workflow — per-file BEFORE/AFTER audit → adversarial verify (bias: present-but-uncited = confirmed loss) → pipeline-reachability capstone. Read-only; `keel/` never touched.

## Bottom line

**No capability was stomped or replaced.** All 7 modified files are additive. Every frontend capability the user feared losing is still present and reachable through a `SKILL.md` citation chain. The three reductions that *look* like losses are the three design-sanctioned orchestration swaps (ticket→binder, PR→integration-branch merge, human-review→automated read-only gate).

The only blemish was housekeeping: one legacy file (`ticket-template.md`) was uncited dead weight. It has since been removed (see F1, **Resolved**) after migrating its one unique section into the cited `ui-analysis.md` — so nothing was lost.

## The diff shape (why the risk surface is small)

- **48 files changed, +4010 / −840.**
- **Zero deletions (`D`).** Every file that existed in `main` still exists on the branch.
- **Only 7 files modified (`M`).** Everything else is net-new (`A`) — additive by construction.
- So all 840 deletions live in those 7 files — the entire replacement-risk surface.
- `kargha-verify` and `kargha-deliver` are **new** skills (absent from `main`); the original three (`kargha-plan`, `kargha-build`, `kargha-validate`) were rewritten in place.

## Per-file verdict

|File|Audit|Adversarial verify|Confirmed losses|
|-|-|-|-|
|skills/kargha-build/SKILL.md|ADDITIVE|ADDITIVE_CONFIRMED|0|
|skills/kargha-plan/SKILL.md|ADDITIVE (sev low)|ADDITIVE_CONFIRMED|0|
|skills/kargha-validate/SKILL.md|ADDITIVE|ADDITIVE_CONFIRMED|0|
|README.md|ADDITIVE|ADDITIVE_CONFIRMED|0|
|.claude-plugin/plugin.json|ADDITIVE|ADDITIVE_CONFIRMED|0|
|.codex-plugin/plugin.json|ADDITIVE|ADDITIVE_CONFIRMED|0|
|.claude-plugin/marketplace.json|ADDITIVE|ADDITIVE_CONFIRMED|0|

## Frontend capability reachability (the user's watchlist)

|Capability|Status|Reached via|
|-|-|-|
|component-to-library classification (match/wrapper/custom/composite)|reachable|kargha-plan → references/ui-analysis.md §4|
|icon-mapping priority + table + Missing-Icons flag|reachable|kargha-plan → ui-analysis.md §5|
|design-token tier mapping (DTCG semantic; primitives deny-listed)|reachable|kargha-plan + kargha-build → references/dtcg-tokens.md|
|data-layer buckets + 3-round data-layer conformance loop|reachable|ui-analysis.md §3 + kargha-build Phase 5b (inline)|
|vertical-slice heuristics + S/M/L estimates|reachable|kargha-plan → ui-analysis.md §7|
|dev-server lifecycle (port→start→health-poll real route→auth gate→cleanup)|reachable|kargha-build Phases 6–7 (inline)|
|design-validation loop|reachable|kargha-build → references/design-validation-loop.md|
|visual capture / serve (capture_view.py, serve_design.py)|reachable|kargha-validate scripts (inline)|
|ci-policy.md / policy-yagni.md / worktree-safety.md|reachable|kargha-plan + kargha-build|

The conditional UI annex in kargha-build (`component_map`/`icon_map`/`token_changes`, Phase 5b, dev-server lifecycle, design-validation loop) fires only when the work item carries a UI surface or a `visual` oracle — exactly as the old `plan→build→validate` did. It is **gated, not removed**. Non-UI stacks skip it.

## The three sanctioned swaps (not losses)

|Was|Now|Why acceptable|
|-|-|-|
|ticket contract / `ticket-template.md` sections|binder JSON fields (`component_map`, `icon_map`, `token_changes`, `design_reference`)|design-sanctioned plumbing; field shapes relocated into cited `ui-analysis.md`|
|open a PR|merge into per-binder integration branch (`refs/kargha/<slug>/…`)|design-sanctioned; no capability attached to the PR step|
|human review during delivery / mandatory per-card walk|automated read-only verify/validate gate + smart-surfaced review|design-sanctioned; human still reviews the integration branch|

## Findings

### F1 — `ticket-template.md` was a citation orphan (RESOLVED)

`skills/kargha-plan/references/ticket-template.md` existed on disk but **no `SKILL.md` cited it** (it was cited 4× by kargha-plan in `main`) — present-on-disk but unreachable.

A deletion-safety pass classified every section: most were obsolete ticket-parse plumbing or already reproduced in cited files (the four table formats live in `ui-analysis.md`; the Design Validation Loop in `design-validation-loop.md`; the Token Changes legend in `ui-analysis.md` §6 + `dtcg-tokens.md`; the CI matrix in `ci-policy.md`). **One section was unique and unreachable: the Foundation Setup Checklist** (provider bootstrap, base stylesheet, icon imports, fonts, token-coverage, supplemental-token rule).

**Resolution (additive-preserving):**
1. Migrated the Foundation Setup Checklist verbatim into `ui-analysis.md` §7 heuristic 1 (the Foundation-slice bullet), which is cited by `kargha-plan/SKILL.md:131` — so the guidance is now reachable.
2. Reworded the 3 descriptive mentions in `ui-analysis.md` (lines 13/164/231) so the table formats read as native to that file.
3. Deleted `ticket-template.md`.

Post-change: zero dangling references; `validate_plugin.py` → PLUGIN INTEGRITY: PASS; `check_shared_copies.py` → SHARED COPIES: IN SYNC; `validate_binder.py --self-test` → 6/6. Reachability is now clean (no on-disk frontend file is uncited).

### F2 — `design-validation-loop.md` internal "Phase 7" header (RESOLVED)

The file carried the old kargha-build numbering ("Phase 7" title + `7a–7e` sub-steps), while the rewritten kargha-build cites it from the Phase 6 acceptance gate. **Resolution:** dropped the stale `Phase 7` identity (title + intro), relabeled the sub-steps with stable semantic labels addressable by colon-path from the doc root `dvl` (`dvl:invoke`, `dvl:parse`, `dvl:fix`, `dvl:loop`, `dvl:edge`, plus `dvl:invoke:auth` / `dvl:invoke:theme`), aligned `Phase 6c`→`6-dev-c`, and updated the `kargha-build/SKILL.md:302` back-reference from `(7a)` to the precise `(dvl:invoke:auth)`. The procedure body (capture/compare/fix loop, exit bar, theme-context handling) is unchanged. Validators green. (The file still uses some old ticket-model vocabulary — `DESIGN_VIEW`, `DESIGN_VALIDATION_PARAMS`, "PR", "ticket" — which is a separate, larger vocabulary migration, not a phase-label issue.)

### Non-findings (reachable, listed for completeness)

- `binder-schema.json`, `example-binder.json` — no direct `SKILL.md` citation but loaded at runtime by the cited `validate_binder.py`; functionally live. (Both are net-new files anyway.)
- `skills/_shared/*.md` (7 files) — intentional authoring source-of-truth, kept byte-identical to the per-skill copies by `check_shared_copies.py`. Not shipped via a `SKILL.md`; by design.

## Conclusion

The augmentation honors the additive contract: orchestration (binder, integration-branch merge, parallel waves, verify/validate gates, non-UI stacks) was added **on top of** the frontend depth, which was relocated intact into cited references rather than thinned. F1 (the one orphan) is resolved — its unique content migrated into a cited file, the dead file removed, all validators green. F2 is optional polish.
