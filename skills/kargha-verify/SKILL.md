---
name: kargha-verify
description: Verify one built work item against its behavioral acceptance check (oracle types unit/integration/e2e/smoke) in a read-only, fresh session on the actual diff; check acceptance + external-contract conformance + boundary crossings; kick findings back to build and escalate to the human only on retry-exhaustion.
triggers:
  - "verify this work item"
  - "run the behavioral gate"
  - "check acceptance for <id>"
---

`kargha-verify` is the thin orchestrator for the behavioral acceptance gate. It dispatches two kargha-owned agents — `agents/kargha-acceptance-reviewer` and `agents/kargha-safety-auditor` — aggregates their verdicts, and drives the kickback and escalation loop per `references/verification-gate.md`. This skill is read-only throughout: the agents read the diff and the binder; this skill never edits code, tests, or the binder.

Visual oracles (`type: visual`) are not this skill's concern — those belong to `kargha-validate`. Opt-out oracles bypass this gate entirely; the build step skips kargha-verify for items where `oracle.opt_out` is true.

## Inputs

The caller must supply:

- **Worktree path** — the checked-out tree holding the item's branch.
- **Binder path + work item id** — locates the `oracle`, `assertions`, and optional `contract` for the item. The binder is a JSON file at `.kargha/binders/<slug>.json` by default; see `references/binder-reference.md`.
- **Diff range** — the item's branch versus the integration tip (e.g. `kargha/<slug>/WI01..kargha/<slug>/integration`).

## Phase 0 — Prerequisites

Before dispatching either agent:

1. Confirm a fresh, thin context: only the worktree path, the binder path, the work item id, and the diff range travel to each agent. No build-session state.
2. Resolve the **pre-verify env command** bound to this wave: read `env_contract.command` from the binder (see `references/verification-gate.md` and `references/binder-reference.md`). If the oracle's assertions need an environment to run and no env command is present, halt with a clear message naming the missing contract — this is a hard gate.
3. Check the floor: if the diff cannot clear compile / type-check / lint, do not dispatch the agents. Surface the item for human review and halt. The floor is defined in `references/definition-of-done.md`.

## Phase 1 — Acceptance + contract conformance

Dispatch **`agents/kargha-acceptance-reviewer`** with the worktree path, binder path, work item id, and diff range.

The agent reads the binder on disk, dispositions each `oracle.assertions[i]` as inspection-verifiable or execution-required, and checks contract conformance against an external artifact (type-checker, schema, or contract test). It returns one of:

- `CONFORMANT` — all assertions disposed, contract confirmed or absent.
- `DEVIATION` — one or more unresolved assertions or a missing contract artifact.
- `BLOCKED` — a required input is unreadable.
- `SPEC-SUSPECT` — the code diverges intentionally and the binder looks stale; halts for human adjudication.

**On DEVIATION:** kick the findings back to kargha-build for bounded self-correction and re-dispatch the agent on the corrected diff. Cap: **max 2 attempts total**. On the second attempt still returning DEVIATION, halt with a call to action — no human escalation at this gate. The implementer chooses: fix-and-rerun, or place a declared-debt marker naming the unresolved assertion and re-run.

**On SPEC-SUSPECT:** halt for human adjudication immediately. Do not loop; do not kick back. The binder is amended through kargha-plan, never by this gate.

**On BLOCKED:** halt with the blocking reason; do not proceed to Phase 2.

## Phase 2 — Boundary scan

Dispatch **`agents/kargha-safety-auditor`** with the same inputs: worktree path, binder path, work item id, and diff range.

The agent re-runs the seven smart-surfaced-review signals (see `references/smart-surfaced-review.md`) on the actual diff and returns:

- `PASS` — no undeclared crossings.
- `VIOLATION` — one or more undeclared boundary crossings.
- `BLOCKED` — a required input is unreadable.

**On VIOLATION:** kick findings back to kargha-build and re-dispatch the agent on the corrected diff. Cap: **max 3 attempts total**. After the third attempt still returning VIOLATION, escalate to the human — an unjustified boundary crossing is a safety question that requires a person's decision.

**On BLOCKED:** halt with the blocking reason.

Phase 2 runs after Phase 1 resolves to CONFORMANT (or is skipped on SPEC-SUSPECT/BLOCKED halt). The two agents run sequentially in the common path; if Phase 1 loops, Phase 2 does not start until Phase 1 clears or exhausts its cap.

## Phase 3 — Aggregate verdict

Combine both agents' return envelopes into a single verdict:

| Acceptance result | Safety result | Aggregate |
|-|-|-|
| CONFORMANT | PASS | `pass` |
| CONFORMANT | VIOLATION (cap not exhausted) | loop Phase 2 |
| DEVIATION (cap not exhausted) | — | loop Phase 1 |
| Either cap exhausted | — | `blocked` (halt-with-CTA or escalate per cap rules) |
| SPEC-SUSPECT or BLOCKED | — | `blocked` (halt for human) |

Report the aggregate verdict to the caller (kargha-build or kargha-deliver):

- `pass` — both agents cleared; report PASS with a one-paragraph summary.
- `concerns` — findings kicked back to build (intermediate, not a terminal state).
- `blocked` — cap exhausted, SPEC-SUSPECT, or BLOCKED input; include the exact agent output and the next required human action.

This skill is read-only throughout all phases.

## Gotchas

- **Floor first.** A change that cannot clear compile / type-check / lint never reaches the agents. See `references/definition-of-done.md`.
- **Read-only.** This skill never edits code, tests, the binder, or any other file. Neither do the agents. If an edit is needed, it goes back to kargha-build.
- **Fresh session per dispatch.** Each agent dispatch is a new session with no build-session context. Pass only the four inputs; nothing else travels.
- **The agents do the reading.** `kargha-acceptance-reviewer` and `kargha-safety-auditor` read the diff and the binder directly. This skill does not pre-read those files for them.
- **Escalate only on exhaustion.** No human review gate fires during delivery except safety-auditor cap exhaustion (3 attempts). The acceptance gate (2 attempts) halts with a call to action, not a human escalation.
- **Caps are per-agent, not shared.** The acceptance cap (2) and the safety cap (3) are independent. Exhausting one does not reset the other.
- **Opt-out items skip this gate.** Items with `oracle.opt_out: true` are not dispatched here. The build step reports the opt-out; kargha-verify is not invoked.
- **Visual oracles belong to kargha-validate.** If a work item arrives here with `oracle.type: visual`, return `blocked` with a note redirecting to `kargha-validate`.
