# Gentle playwright-cli install guidance — design

- Date: 2026-06-15
- Status: approved (brainstorming) — pending implementation plan
- Topic: guide kargha users to install `playwright-cli` + its agent skill when missing

## Problem

`kargha-validate` shells out to the external binary `playwright-cli` to capture the
running app and the design prototype. When it is absent, the bundled
`capture_view.py` fails with a terse `SystemExit("playwright-cli is not available on
PATH")` and no remediation. The user is left to discover the install path themselves.

`playwright-cli` (https://github.com/microsoft/playwright-cli, active as of v0.1.14,
June 2026) installs in two steps: the global npm binary, then its own coding-agent
skill. Both should be surfaced when the dependency is missing.

## Goals

- When `playwright-cli` is missing, the failure carries an actionable call-to-action:
  why it is needed, the two install commands, the docs link, and a re-invoke nudge.
- Keep the dependency a **hard gate** — validation genuinely cannot run without it.
  This is gentler *messaging*, not a softer gate. No prompting, no auto-install.
- Keep guidance consistent across the script, both skills, and the README.

## Non-goals (YAGNI)

- No install-time enforcement. Confirmed against Claude Code / Codex docs: plugin
  systems have **no install/enable lifecycle hook**, **no manifest field** for system
  binaries, and **cannot block install** on a missing binary. Out of scope.
- No SessionStart hook (the rejected "always warn" option — fires every session even
  when kargha is unused; not gentle).
- No auto-install and no consent prompt to install.
- No version/upgrade checks.
- No attempt to verify the `--skills` companion actually installed (no reliable check;
  we recommend the command, we do not assert its result).

## Canonical message

Single source of truth: the `SystemExit` payload in `capture_view.py`. The skills and
README describe the behavior and restate the commands; the literal wording lives in the
script.

```
playwright-cli is not available on PATH, so I can't capture the app/design for visual validation.

To enable it (one-time):
  1. npm install -g @playwright/cli@latest
  2. playwright-cli install --skills   # adds its agent skill

Docs: https://github.com/microsoft/playwright-cli
Then re-run the validation and I'll pick up from here.
```

The leading clause keeps the existing `"playwright-cli is not available on PATH"` text
so any existing matching still works; the CTA is appended.

## Changes

| # | File | Change |
|-|-|-|
| 1 | `skills/kargha-validate/scripts/capture_view.py` | `resolve_playwright_command()` (currently lines 64–68) keeps `raise SystemExit(...)` but the message becomes the canonical multi-line block above. `shutil.which("playwright-cli")` detection is unchanged; exit stays non-zero. |
| 2 | `skills/kargha-validate/SKILL.md` | Phase 0 prerequisite #4 and the top "external dependency" note: state that a missing `playwright-cli` yields the actionable install guidance (both commands + the `--skills` companion + docs link) and a clean stop. Remains a hard gate; the *report* is now the CTA. No hand-driving Playwright, no auto-install. |
| 3 | `skills/kargha-build/SKILL.md` | Phase 7: when the design-validation tool reports its capture dependency is missing (distinct from *no tool configured*), surface the same install nudge **once** and note the round used the degraded/manual path — explicitly **not** a fidelity failure. |
| 4 | `README.md` | Requirements: add `playwright-cli install --skills` after the existing `npm install -g @playwright/cli@latest`, so setup docs match the in-skill guidance. |

## Behavior

- `capture_view.py` invoked with `playwright-cli` absent → prints the canonical block to
  stderr (via `SystemExit`) and exits non-zero. The agent running `kargha-validate`
  relays it and stops.
- `kargha-validate` treats this as the Phase 0 hard-gate failure path — clean stop with
  an actionable report, no fallback capture.
- `kargha-build` Phase 7 distinguishes "tool present but dependency missing" (surface the
  CTA, mark the round degraded/manual, not a failure) from "no validation tool at all"
  (existing degrade-to-manual-checklist behavior).

## Verification

- `uv run skills/kargha-validate/scripts/capture_view.py --self-test` still passes
  (unchanged path).
- Confirm the missing-binary branch: with `playwright-cli` not on PATH, running the
  script prints the canonical block (including both commands and the docs link) and
  exits non-zero. Cover via a small added self-test assertion or a documented manual
  check.
- Skim the two SKILL.md edits and README for consistent command wording against the
  canonical block.
