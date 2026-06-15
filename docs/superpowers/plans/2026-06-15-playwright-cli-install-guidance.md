# Gentle playwright-cli Install Guidance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When `playwright-cli` is missing, kargha surfaces an actionable two-step install CTA instead of a terse failure — across the capture script, both skills, and the README.

**Architecture:** Keep the dependency a hard gate (validation cannot run without it). The literal CTA lives once in `capture_view.py` as a module constant raised via `SystemExit`; the two skills and README describe the behavior and restate the commands. No install-time enforcement (not supported by Claude Code / Codex plugin systems), no SessionStart hook, no auto-install.

**Tech Stack:** Python 3.12 PEP 723 standalone scripts run via `uv run`; in-script `--self-test` is the test harness (no pytest in this repo). Markdown skills + README.

**Spec:** `docs/superpowers/specs/2026-06-15-playwright-cli-install-guidance-design.md`

**Branch:** `playwright-cli-install-guidance` (already created off `main`).

---

### Task 1: capture_view.py — CTA message + self-test

**Files:**
- Modify: `skills/kargha-validate/scripts/capture_view.py` — add module constant; rewire `resolve_playwright_command()` (currently lines 64–68); extend `self_test()` (currently lines 190–195)
- Test: same file, via `--self-test`

- [ ] **Step 1: Write the failing test**

In `self_test()` (currently lines 190–195), add CTA-content assertions plus a deterministic check that `resolve_playwright_command()` raises the CTA when the binary is absent (monkeypatching `shutil.which` — no fragile PATH manipulation). The function becomes:

```python
def self_test() -> None:
    width, height = parse_viewport("1440x900")
    assert (width, height) == (1440, 900)
    sample = {"compare_ready": False, "app": {"health": "DEGRADED_AUTH"}}
    assert sample["app"]["health"] == "DEGRADED_AUTH"
    # CTA message content
    assert PLAYWRIGHT_CLI_MISSING_HELP.startswith("playwright-cli is not available on PATH")
    assert "npm install -g @playwright/cli@latest" in PLAYWRIGHT_CLI_MISSING_HELP
    assert "playwright-cli install --skills" in PLAYWRIGHT_CLI_MISSING_HELP
    assert "https://github.com/microsoft/playwright-cli" in PLAYWRIGHT_CLI_MISSING_HELP
    # resolve_playwright_command raises the CTA when the binary is absent
    original_which = shutil.which
    shutil.which = lambda _cmd: None
    try:
        raised = None
        try:
            resolve_playwright_command()
        except SystemExit as exc:
            raised = str(exc)
        assert raised == PLAYWRIGHT_CLI_MISSING_HELP
    finally:
        shutil.which = original_which
    print("capture_view self-test passed")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run skills/kargha-validate/scripts/capture_view.py --self-test`
Expected: FAIL — `NameError: name 'PLAYWRIGHT_CLI_MISSING_HELP' is not defined`

- [ ] **Step 3: Write minimal implementation**

Add the module-level constant immediately above `resolve_playwright_command()` (just before its current line 64):

```python
PLAYWRIGHT_CLI_MISSING_HELP = (
    "playwright-cli is not available on PATH, so I can't capture the app/design "
    "for visual validation.\n"
    "\n"
    "To enable it (one-time):\n"
    "  1. npm install -g @playwright/cli@latest\n"
    "  2. playwright-cli install --skills   # adds its agent skill\n"
    "\n"
    "Docs: https://github.com/microsoft/playwright-cli\n"
    "Then re-run the validation and I'll pick up from here."
)
```

Then change the raise inside `resolve_playwright_command()` from:

```python
        raise SystemExit("playwright-cli is not available on PATH")
```

to:

```python
        raise SystemExit(PLAYWRIGHT_CLI_MISSING_HELP)
```

(The `shutil.which("playwright-cli")` detection and the `return [resolved]` line are unchanged.)

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run skills/kargha-validate/scripts/capture_view.py --self-test`
Expected: PASS — prints `capture_view self-test passed`. (This now deterministically exercises the missing-binary branch via the monkeypatched `shutil.which`, so no separate live-PATH check is needed.)

- [ ] **Step 5: Commit**

```bash
git add skills/kargha-validate/scripts/capture_view.py
git commit -m "feat(validate): actionable CTA when playwright-cli is missing"
```

---

### Task 2: kargha-validate/SKILL.md — describe the gentle guidance

**Files:**
- Modify: `skills/kargha-validate/SKILL.md` — the external-dependency note (line 15) and Phase 0 prerequisite #4 (line 48)

- [ ] **Step 1: Append to the external-dependency note**

Find this paragraph (line 15):

```
`playwright-cli` is an external dependency. This skill uses the installed `playwright-cli` command; it does not patch, wrap, bypass, or maintain Playwright CLI behavior. If a Playwright action cannot be performed, fail with the command, exit code, stdout, and stderr.
```

Append one sentence to the end of it:

```
 When `playwright-cli` is not installed at all, `capture_view.py` fails with a one-time install CTA (`npm install -g @playwright/cli@latest`, then `playwright-cli install --skills` for its companion agent skill); relay that message to the user and stop — do not auto-install or hand-drive Playwright.
```

- [ ] **Step 2: Expand Phase 0 prerequisite #4**

Find this line (line 48):

```
4. **`playwright-cli` is available.** `capture_view.py` checks this before capture.
```

Replace it with:

```
4. **`playwright-cli` is available.** `capture_view.py` checks this before capture. If it is missing, the script exits non-zero with an actionable install CTA — the two-step `npm install -g @playwright/cli@latest` then `playwright-cli install --skills`, plus the docs link. Surface that message and stop; this stays a hard gate (no prompting, no auto-install, no degraded capture).
```

- [ ] **Step 3: Verify wording**

Run: `grep -n "playwright-cli install --skills" skills/kargha-validate/SKILL.md`
Expected: two matches (the two edits above).

- [ ] **Step 4: Commit**

```bash
git add skills/kargha-validate/SKILL.md
git commit -m "docs(validate): document missing-playwright install CTA"
```

---

### Task 3: kargha-build/SKILL.md — Phase 7 missing-dependency nudge

**Files:**
- Modify: `skills/kargha-build/SKILL.md` — Phase 7 description (line 391)

- [ ] **Step 1: Add the missing-dependency clause**

Find this paragraph (line 391), which ends:

```
...fix obvious critical/major discrepancies, capped at the same 3 passes. Re-run lint/test between rounds (round 1 reuses the Phase 5 pass); the Phase 6 dev server must be up.
```

Append one sentence to the end of that paragraph:

```
 If the validation tool reports its capture dependency is missing (e.g. `playwright-cli` not installed — distinct from no validation tool being configured), surface its install CTA to the user once, treat this round as degraded/manual rather than a fidelity failure, and continue; don't silently fall back as if no validator existed.
```

- [ ] **Step 2: Verify wording**

Run: `grep -n "capture dependency is missing" skills/kargha-build/SKILL.md`
Expected: one match.

- [ ] **Step 3: Commit**

```bash
git add skills/kargha-build/SKILL.md
git commit -m "docs(build): surface playwright-cli CTA instead of silent degrade in Phase 7"
```

---

### Task 4: README.md — Requirements line

**Files:**
- Modify: `README.md` — the `kargha-validate` Requirements bullet (line 98)

- [ ] **Step 1: Add the companion-skill install step**

Find this fragment in the `kargha-validate` requirements bullet (line 98):

```
[`playwright-cli`](https://playwright.dev) (`npm install -g @playwright/cli@latest`), and a browser (Chromium).
```

Replace it with:

```
[`playwright-cli`](https://playwright.dev) (`npm install -g @playwright/cli@latest`, then `playwright-cli install --skills` for its companion agent skill), and a browser (Chromium).
```

- [ ] **Step 2: Verify wording**

Run: `grep -n "playwright-cli install --skills" README.md`
Expected: one match.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add playwright-cli --skills install step to requirements"
```

---

## Notes for the implementer

- Run all commands from the repo root (`/Users/tej/src/kargha`) on branch `playwright-cli-install-guidance`.
- The four tasks are independent and may be done in any order, but Task 1 is the only one with executable verification — do it first to lock the canonical wording, then keep Tasks 2–4 textually consistent with it.
- Do NOT add a SessionStart hook, auto-install logic, version checks, or any plugin-manifest dependency field — all explicitly out of scope per the spec.
- There is no CI in this repo; verification is the `--self-test` run plus the `grep` checks above.
