# kargha Orchestration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Evolve kargha from a three-skill frontend plugin (`plan → build → validate`) into a stack-agnostic, ad-hoc orchestration framework that synthesizes a **binder** and delivers it in **parallel waves** onto a per-binder **integration branch**, each work item verified against its own acceptance check.

**Architecture:** The **binder** (a validated JSON artifact) is the spine and the cross-skill contract. `kargha-plan` (keel-refine light) synthesizes and commits it. `kargha-deliver` schedules its work items into parallel waves on the integration branch. `kargha-build` implements one item in an isolated worktree, ending at tagged commits (no PR). `kargha-verify` (behavioral) and `kargha-validate` (visual) are read-only acceptance gates that kick findings back to build and escalate to the human only on retry-exhaustion — keel's automated-gate pattern, registry-free. A single-work-item binder skips deliver (the "just this once" hatch).

**Tech Stack:** Anthropic Agent Skills (canonical `SKILL.md` + `references/` + `scripts/`), packaged as a Claude Code / Codex plugin. JSON Schema (draft 2020-12) for the binder. PEP 723 Python scripts run via `uv run` for validation and the existing playwright capture. git worktrees + tags + a `refs/kargha/` namespace for orchestration and resume. No new runtime service.

**Plan-level decisions (mechanisms the spec deferred here).** This plan nails down the three items §12 of the spec left open:
- **Commit/ref tagging scheme** — item commits carry a `[kargha:item-<id>]` subject marker; the integration tip is tagged `kargha/<slug>/wave-<N>-base` (pre-merge revert anchor) and `kargha/<slug>/wave-<N>` (post-check completed tip); per-item outcomes live under `refs/kargha/<slug>/item-<id>/{done,failed,in-progress}`.
- **Serial merge-queue** — items build concurrently; completed items pass one at a time through a serial merge step (FIFO by completion), each re-validating its oracle against the *current* integration tip before merging.
- **Env-injection contract** — a binder-level `env_contract {command, supports_isolation, isolation_params}`; the test env binds to the wave; items needing a stateful env get injected isolation params only if `supports_isolation` is true, else they serialize.

These appear as concrete schema fields (Task 1) and reference procedures (Tasks 5–7), so no later task carries a placeholder.

**Reuse of keel's agents (lean lane).** kargha's verification layer is modeled on — and ported registry-free from — keel's **lean (karta) lane** gates (keel is READ-ONLY this session; we copy/adapt prose into kargha-owned files, **no interop**). kargha ships two kargha-owned agents under `agents/`:
- **`kargha-acceptance-reviewer`** ← keel `karta-spec-reviewer`: per-assertion evidence disposition against the oracle + contract; read-only, fresh session, on the actual diff; **max 2 attempts**, halt-with-CTA on exhaustion (implementer picks fix-and-rerun / declared-debt — no human escalation). Verdict tokens `CONFORMANT | DEVIATION | BLOCKED | SPEC-SUSPECT`, mapped to a `pass | concerns | blocked` envelope.
- **`kargha-safety-auditor`** ← keel `safety-auditor`: boundary/destructive/sensitive scan on the actual diff; read-only; **max 3 attempts**, escalate to the human on exhaustion. Verdict `PASS | VIOLATION`. The **invariants registry is stripped** — kargha's smart-surfaced-review signals (Task 5) are its inline rule set.

keel's `implementer`, `test-writer`, `landing-verifier`, `pre-check`, and `code-reviewer` inform **skill prose** (build discipline, oracle synthesis, final-gate framing) but are **not** shipped as V1 gate agents — matching keel's lean lane, which itself drops test-writer and code-reviewer.

**Out of scope (per spec §10):** keel binder-schema interop; a separate `submit`/`integrate` skill; invariants registry / fail-closed safety; setup/adopt lane; persisted state *files* (resume is git-native); multi-binder partition (one binder/run in V1); a human review gate *during* delivery. Shipping the plainlanguage skill bundled in this repo is **post-V1** (recorded, not built here).

---

## File Structure

New and modified files, by responsibility. Paths are relative to the repo root (`.../kargha-orchestration/`).

**The contract (binder) — owned by kargha-plan:**
- Create: `skills/kargha-plan/references/binder-schema.json` — JSON Schema for the binder.
- Create: `skills/kargha-plan/references/example-binder.json` — one canonical multi-item binder (doc + fixture).
- Create: `skills/kargha-plan/scripts/validate_binder.py` — schema + cycle + dangling-dep + opt-out validator, with `--self-test`.
- Create: `skills/kargha-plan/references/binder-reference.md` — human-readable binder field guide.

**Repo-wide integrity test:**
- Create: `scripts/validate_plugin.py` — SKILL.md frontmatter + reference-link integrity + example-binder validation, with `--self-test`.

**Shared reference content (authored once, copied into each consuming skill per the repo's existing duplication convention):**
- Create: `skills/_shared/smart-surfaced-review.md` *(authoring home; copied — see note below)*
- Create: `skills/_shared/verification-gate.md`
- Create: `skills/_shared/integration-branch.md`
- Create: `skills/_shared/parallelism-gates.md`
- Create: `skills/_shared/declared-debt.md`
- Create: `skills/_shared/secret-scan.md`
- Create: `skills/_shared/definition-of-done.md`

> **Note on `skills/_shared/`:** the repo currently *duplicates* shared references into each skill (`dtcg-tokens.md`, `ci-policy.md`, `policy-yagni.md` live in both `kargha-plan/references/` and `kargha-build/references/`). We keep that convention so each skill stays self-contained, but we author each new shared reference **once** under `skills/_shared/` (not shipped as a skill — it has no `SKILL.md`), then each consuming skill task copies the files it needs into its own `references/`. `_shared/` is the single source of truth for edits; the copies are what skills cite. `validate_plugin.py` checks that each skill's cited copy exists; a later hardening task (Task 15) verifies copies match their `_shared/` source.

**Agents (kargha-owned, ported registry-free from keel's lean lane — authored in Task 6):**
- Create: `agents/kargha-acceptance-reviewer.md` — behavioral acceptance + contract conformance gate (← keel `karta-spec-reviewer`).
- Create: `agents/kargha-safety-auditor.md` — boundary/destructive/sensitive scan (← keel `safety-auditor`).

**The five skills:**
- Modify: `skills/kargha-plan/SKILL.md` — frontend-ticket emitter → stack-agnostic binder synthesizer (keel-refine light).
- Modify: `skills/kargha-build/SKILL.md` — frontend-only, opens PR → stack-agnostic, item-from-binder, integration-branch merge, acceptance loop, tagging, secret scan, declared debt; no PR.
- Create: `skills/kargha-deliver/SKILL.md` — wave scheduler over the integration branch.
- Create: `skills/kargha-verify/SKILL.md` — behavioral read-only acceptance gate with kickback/escalate.
- Modify: `skills/kargha-validate/SKILL.md` — pass/fail → acceptance-grounded against the oracle; kickback framing. (Scripts unchanged.)

**Packaging & docs:**
- Modify: `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json` — description, version, keywords.
- Modify: `.claude-plugin/marketplace.json`, `.agents/plugins/marketplace.json` — skill enumeration if present.
- Modify: `README.md` — orchestration framing, five skills, the binder, the pipeline.
- Create: `docs/how-to/parallelism-and-review.md` — user-facing how-to carrying the two blessed verbatim copy blocks.

---

## Task 1: Binder JSON Schema + canonical example

**Files:**
- Create: `skills/kargha-plan/references/binder-schema.json`
- Create: `skills/kargha-plan/references/example-binder.json`

- [ ] **Step 1: Write the schema**

Write `skills/kargha-plan/references/binder-schema.json` exactly:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/TejGandham/kargha/binder-schema.json",
  "title": "kargha binder",
  "type": "object",
  "required": ["slug", "motivation", "scope", "work_items"],
  "additionalProperties": false,
  "properties": {
    "slug": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]*$",
      "description": "kebab-case; names the integration branch kargha/<slug>/integration and the wave tags"
    },
    "motivation": { "type": "string", "minLength": 1 },
    "scope": {
      "type": "object",
      "required": ["included"],
      "additionalProperties": false,
      "properties": {
        "included": { "type": "array", "items": { "type": "string" } },
        "excluded": { "type": "array", "items": { "type": "string" } }
      }
    },
    "design_facts": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "source": { "type": ["string", "null"], "description": "path to the source design/prototype, or null" },
        "stack": { "type": "string", "description": "resolved stack, recorded once" }
      }
    },
    "token_manifest": {
      "type": ["object", "null"],
      "description": "shared design-token map; present only when a token system exists"
    },
    "env_contract": {
      "type": "object",
      "required": ["command", "supports_isolation"],
      "additionalProperties": false,
      "properties": {
        "command": { "type": "string", "description": "the project's own test/dev env command" },
        "supports_isolation": { "type": "boolean", "description": "whether the command accepts injectable isolation params" },
        "isolation_params": { "type": "array", "items": { "type": "string" }, "description": "e.g. PORT, COMPOSE_PROJECT_NAME" }
      }
    },
    "work_items": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/$defs/workItem" }
    }
  },
  "$defs": {
    "workItem": {
      "type": "object",
      "required": ["id", "title", "oracle"],
      "additionalProperties": false,
      "properties": {
        "id": { "type": "string", "pattern": "^[a-z0-9][a-z0-9-]*$" },
        "title": { "type": "string", "minLength": 1 },
        "estimate": { "enum": ["S", "M", "L"] },
        "depends_on": { "type": "array", "items": { "type": "string" }, "default": [] },
        "design_reference": { "type": "string", "description": "view/route id, or the literal 'none'" },
        "component_map": { "type": "array" },
        "icon_map": { "type": "array" },
        "token_changes": { "type": "array" },
        "contract": {
          "type": ["object", "string", "null"],
          "description": "the open-shape interface this item exposes/consumes"
        },
        "serialize": { "type": "boolean", "default": false },
        "shared_resources": { "type": "array", "items": { "type": "string" } },
        "surface": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "flagged": { "type": "boolean" },
            "signals": { "type": "array", "items": { "type": "string" } }
          }
        },
        "oracle": { "$ref": "#/$defs/oracle" }
      }
    },
    "oracle": {
      "oneOf": [
        {
          "type": "object",
          "required": ["type"],
          "additionalProperties": false,
          "properties": {
            "type": { "enum": ["unit", "integration", "e2e", "smoke", "visual"] },
            "assertions": { "type": "array", "items": { "type": "string" } },
            "command": { "type": "string", "description": "the CI-facing check kargha runs; floor is compile/type-check/lint" }
          }
        },
        {
          "type": "object",
          "required": ["opt_out", "reason"],
          "additionalProperties": false,
          "properties": {
            "opt_out": { "const": true },
            "reason": { "type": "string", "minLength": 1 }
          }
        }
      ]
    }
  }
}
```

- [ ] **Step 2: Write the canonical example binder**

Write `skills/kargha-plan/references/example-binder.json` — a 3-item binder exercising deps, an opt-out, a serialized item, a visual oracle, and `env_contract`:

```json
{
  "slug": "notifications-redesign",
  "motivation": "Replace the legacy alerts panel with the new notifications center from the design prototype.",
  "scope": {
    "included": ["notifications list view", "notification detail pane", "mark-all-read action"],
    "excluded": ["push delivery backend", "email digests"]
  },
  "design_facts": {
    "source": "design-export/notifications.html",
    "stack": "Next.js app router + internal component library + DTCG tokens + GraphQL"
  },
  "token_manifest": null,
  "env_contract": {
    "command": "npm run dev",
    "supports_isolation": true,
    "isolation_params": ["PORT"]
  },
  "work_items": [
    {
      "id": "shell",
      "title": "Notifications center shell + routing",
      "estimate": "M",
      "depends_on": [],
      "design_reference": "none",
      "contract": "exports <NotificationsCenter/> mount point at /notifications",
      "oracle": { "type": "smoke", "assertions": ["/notifications route renders without error"], "command": "npm run lint && npm test" }
    },
    {
      "id": "list-view",
      "title": "Notifications list view",
      "estimate": "L",
      "depends_on": ["shell"],
      "design_reference": "notifications-list",
      "oracle": { "type": "visual", "assertions": ["matches notifications-list view at 1440x900, zero critical/major discrepancies"] }
    },
    {
      "id": "schema-migration",
      "title": "Add read_at column to notifications table",
      "estimate": "S",
      "depends_on": [],
      "serialize": true,
      "shared_resources": ["db/migrations"],
      "contract": "adds nullable read_at timestamptz to notifications",
      "oracle": { "opt_out": true, "reason": "migration verified by the team's existing migration test suite in CI, not re-run here" }
    }
  ]
}
```

- [ ] **Step 3: Sanity-check JSON validity**

Run: `uv run python -c "import json,sys; json.load(open('skills/kargha-plan/references/binder-schema.json')); json.load(open('skills/kargha-plan/references/example-binder.json')); print('OK')"`
Expected: `OK` (both files parse).

- [ ] **Step 4: Commit**

```bash
git add skills/kargha-plan/references/binder-schema.json skills/kargha-plan/references/example-binder.json
git commit -m "feat(plan): add binder JSON schema and canonical example"
```

---

## Task 2: Binder validator (`validate_binder.py`) — TDD

The validator does what JSON Schema alone cannot: detect dependency **cycles** and **dangling** dep ids, and summarize **opt-outs** — on top of schema validation.

**Files:**
- Create: `skills/kargha-plan/scripts/validate_binder.py`

- [ ] **Step 1: Write the script with an embedded self-test**

Write `skills/kargha-plan/scripts/validate_binder.py`:

```python
# /// script
# requires-python = ">=3.11"
# dependencies = ["jsonschema>=4.21"]
# ///
"""Validate a kargha binder: JSON Schema + dependency-graph + opt-out checks.

Usage:
  uv run validate_binder.py --binder <path>     # validate one binder, exit 0/1
  uv run validate_binder.py --self-test          # run embedded fixtures, exit 0/1
"""
from __future__ import annotations
import argparse, json, sys
from pathlib import Path
from jsonschema import Draft202012Validator

SCHEMA_PATH = Path(__file__).resolve().parent.parent / "references" / "binder-schema.json"


def _load_schema() -> dict:
    return json.loads(SCHEMA_PATH.read_text())


def validate_binder(binder: dict) -> list[str]:
    """Return a list of human-readable errors; empty list == valid."""
    errors: list[str] = []
    validator = Draft202012Validator(_load_schema())
    for e in sorted(validator.iter_errors(binder), key=lambda e: list(e.path)):
        loc = "/".join(str(p) for p in e.path) or "(root)"
        errors.append(f"schema: {loc}: {e.message}")
    if errors:
        return errors  # graph checks assume a schema-valid shape

    items = binder.get("work_items", [])
    ids = [it["id"] for it in items]
    if len(ids) != len(set(ids)):
        errors.append("graph: duplicate work-item id(s)")
    id_set = set(ids)
    for it in items:
        for dep in it.get("depends_on", []):
            if dep not in id_set:
                errors.append(f"graph: item '{it['id']}' depends_on unknown id '{dep}'")

    # cycle detection (DFS over depends_on)
    graph = {it["id"]: list(it.get("depends_on", [])) for it in items}
    WHITE, GRAY, BLACK = 0, 1, 2
    color = {i: WHITE for i in graph}

    def visit(node: str, stack: list[str]) -> None:
        color[node] = GRAY
        for nxt in graph.get(node, []):
            if nxt not in color:
                continue  # dangling already reported
            if color[nxt] == GRAY:
                cyc = " -> ".join(stack + [nxt])
                errors.append(f"graph: dependency cycle: {cyc}")
            elif color[nxt] == WHITE:
                visit(nxt, stack + [nxt])
        color[node] = BLACK

    for i in graph:
        if color[i] == WHITE:
            visit(i, [i])
    return errors


def opt_out_summary(binder: dict) -> list[str]:
    return [f"{it['id']}: {it['oracle']['reason']}"
            for it in binder.get("work_items", [])
            if isinstance(it.get("oracle"), dict) and it["oracle"].get("opt_out")]


def _run_self_test() -> int:
    valid = json.loads((SCHEMA_PATH.parent / "example-binder.json").read_text())
    cyclic = {
        "slug": "c", "motivation": "x", "scope": {"included": ["x"]},
        "work_items": [
            {"id": "a", "title": "A", "depends_on": ["b"], "oracle": {"type": "unit"}},
            {"id": "b", "title": "B", "depends_on": ["a"], "oracle": {"type": "unit"}},
        ],
    }
    dangling = {
        "slug": "d", "motivation": "x", "scope": {"included": ["x"]},
        "work_items": [{"id": "a", "title": "A", "depends_on": ["ghost"], "oracle": {"type": "unit"}}],
    }
    no_oracle = {
        "slug": "n", "motivation": "x", "scope": {"included": ["x"]},
        "work_items": [{"id": "a", "title": "A"}],
    }
    optout_no_reason = {
        "slug": "o", "motivation": "x", "scope": {"included": ["x"]},
        "work_items": [{"id": "a", "title": "A", "oracle": {"opt_out": True}}],
    }
    cases = [
        ("valid example", valid, True),
        ("cyclic deps", cyclic, False),
        ("dangling dep", dangling, False),
        ("missing oracle", no_oracle, False),
        ("opt-out without reason", optout_no_reason, False),
    ]
    failures = 0
    for name, binder, should_pass in cases:
        errs = validate_binder(binder)
        passed = not errs
        ok = passed == should_pass
        print(f"[{'PASS' if ok else 'FAIL'}] {name}: "
              f"{'valid' if passed else 'invalid (' + '; '.join(errs) + ')'}")
        if not ok:
            failures += 1
    # opt-out summary must be detected on the valid example
    summ = opt_out_summary(valid)
    ok = len(summ) == 1
    print(f"[{'PASS' if ok else 'FAIL'}] opt-out summary on example: {summ}")
    failures += 0 if ok else 1
    print(f"\n{len(cases) + 1 - failures}/{len(cases) + 1} checks passed")
    return 1 if failures else 0


def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("--binder", type=Path)
    ap.add_argument("--self-test", action="store_true")
    args = ap.parse_args()
    if args.self_test:
        return _run_self_test()
    if not args.binder:
        ap.error("provide --binder <path> or --self-test")
    binder = json.loads(args.binder.read_text())
    errs = validate_binder(binder)
    if errs:
        print("INVALID:")
        for e in errs:
            print(f"  - {e}")
        return 1
    summ = opt_out_summary(binder)
    print(f"VALID. {len(binder['work_items'])} work items; {len(summ)} opted out of acceptance checks.")
    for s in summ:
        print(f"  opt-out: {s}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 2: Run the self-test (expect PASS)**

Run: `uv run skills/kargha-plan/scripts/validate_binder.py --self-test`
Expected: every line `[PASS ...]`, final `6/6 checks passed`, exit 0.

- [ ] **Step 3: Run it against the canonical example**

Run: `uv run skills/kargha-plan/scripts/validate_binder.py --binder skills/kargha-plan/references/example-binder.json`
Expected: `VALID. 3 work items; 1 opted out of acceptance checks.` and the opt-out line for `schema-migration`.

- [ ] **Step 4: Commit**

```bash
git add skills/kargha-plan/scripts/validate_binder.py
git commit -m "feat(plan): add binder validator with schema, cycle, and opt-out checks"
```

---

## Task 3: Plugin integrity test (`validate_plugin.py`) — TDD

This is the repo's plugin-wide test: every `SKILL.md` has valid frontmatter, every relative reference path a `SKILL.md` cites exists, and every `agents/*.md` (the kargha-owned agents added in Task 6) has valid frontmatter. It is the "run the test" gate for every later prose task. (Binder *content* is validated separately by `validate_binder.py`.)

**Files:**
- Create: `scripts/validate_plugin.py`

- [ ] **Step 1: Write the script**

Write `scripts/validate_plugin.py`:

```python
# /// script
# requires-python = ">=3.11"
# dependencies = []
# ///
"""Plugin integrity check: SKILL.md frontmatter + reference-link existence.

Usage:
  uv run scripts/validate_plugin.py --self-test   # check this repo, exit 0/1
"""
from __future__ import annotations
import argparse, re, sys
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent
SKILLS = ROOT / "skills"
LINK_RE = re.compile(r"\(([^)]+\.(?:md|json|py))\)")          # markdown links
PATH_RE = re.compile(r"`(references/[^`]+|scripts/[^`]+)`")    # backticked paths


def _frontmatter(text: str) -> dict[str, str]:
    if not text.startswith("---"):
        return {}
    end = text.find("\n---", 3)
    if end == -1:
        return {}
    fm: dict[str, str] = {}
    for line in text[3:end].splitlines():
        if ":" in line:
            k, _, v = line.partition(":")
            fm[k.strip()] = v.strip()
    return fm


def check() -> list[str]:
    errors: list[str] = []
    skill_dirs = [p.parent for p in SKILLS.glob("*/SKILL.md")]
    if not skill_dirs:
        errors.append("no skills found under skills/*/SKILL.md")
    for sd in sorted(skill_dirs):
        text = (sd / "SKILL.md").read_text()
        fm = _frontmatter(text)
        for field in ("name", "description"):
            if not fm.get(field):
                errors.append(f"{sd.name}: SKILL.md missing frontmatter '{field}'")
        cited = set(LINK_RE.findall(text)) | set(PATH_RE.findall(text))
        for rel in sorted(cited):
            if rel.startswith(("http://", "https://")):
                continue
            target = (sd / rel).resolve()
            if not str(target).startswith(str(ROOT)):
                continue  # out-of-tree example path, not a repo file
            if not target.exists():
                errors.append(f"{sd.name}: SKILL.md cites missing path '{rel}'")
    # kargha-owned agents (ported from keel): frontmatter only (no SKILL-style links)
    for agent in sorted((ROOT / "agents").glob("*.md")):
        fm = _frontmatter(agent.read_text())
        for field in ("name", "description"):
            if not fm.get(field):
                errors.append(f"agents/{agent.name}: missing frontmatter '{field}'")
    return errors


def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("--self-test", action="store_true")
    ap.parse_args()
    errors = check()
    if errors:
        print("PLUGIN INTEGRITY: FAIL")
        for e in errors:
            print(f"  - {e}")
        return 1
    print("PLUGIN INTEGRITY: PASS")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 2: Run it (expect PASS against the current repo)**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PLUGIN INTEGRITY: PASS`, exit 0. (The three existing skills already have valid frontmatter and resolvable references.)

- [ ] **Step 3: Prove the link check bites (temporary negative test)**

Add a line citing a missing file to any SKILL.md, e.g. append `` See `references/does-not-exist.md`. `` to `skills/kargha-validate/SKILL.md`, run the self-test, confirm it FAILs naming that path, then revert the line.
Run: `uv run scripts/validate_plugin.py --self-test`
Expected: FAIL listing `references/does-not-exist.md`. After revert: PASS.

- [ ] **Step 4: Commit**

```bash
git add scripts/validate_plugin.py
git commit -m "test: add plugin integrity check (frontmatter + reference links)"
```

---

## Task 4: Binder field guide (`binder-reference.md`)

**Files:**
- Create: `skills/kargha-plan/references/binder-reference.md`

- [ ] **Step 1: Author the field guide**

Write `skills/kargha-plan/references/binder-reference.md` documenting every field in `binder-schema.json` for a reader who has never seen a binder. Required sections and content:

| Section | Required content |
|-|-|
| Intro | One paragraph: the binder is kargha's spine and the cross-skill contract; it is kargha-owned (keel concepts, own shape, **no keel interop**); validated by `validate_binder.py`. |
| Binder-level fields | A table: `slug`, `motivation`, `scope.included/excluded`, `design_facts.source/stack`, `token_manifest`, `env_contract.command/supports_isolation/isolation_params`, `work_items` — each with type, required?, and one-line meaning. |
| Per-work-item fields | A table: `id`, `title`, `estimate`, `depends_on`, `design_reference`, `component_map`/`icon_map`/`token_changes` (only-where-relevant), `contract`, `serialize`, `shared_resources`, `surface.flagged/signals`, `oracle`. |
| The oracle | Explain the two shapes (a check `{type, assertions, command}` or an explicit `{opt_out, reason}`). State the **floor**: a non-opted-out change must at least compile/type-check/lint; below that kargha surfaces instead of auto-merging. State that opt-out is **explicit and recorded**, never silent. Cite `definition-of-done.md`. |
| On disk + resume | Default location `.kargha/binders/<slug>.json`; committed only at run boundaries; **read-only to build steps**; resume is git-native (cite `integration-branch.md` for tags/refs). |
| Example | Embed or link `example-binder.json`; walk its three items (a smoke-oracle shell, a visual-oracle list view, an opted-out serialized migration). |

Keep `design_reference`, `component_map`, `icon_map`, `token_changes`, `token_manifest` described as **optional, UI-relevant** fields — present only when the stack has a design/token surface, absent for non-UI work.

- [ ] **Step 2: Run the integrity test**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS` (this file is not yet cited by a SKILL.md, so no new link obligations; it must not break parsing).

- [ ] **Step 3: Commit**

```bash
git add skills/kargha-plan/references/binder-reference.md
git commit -m "docs(plan): add binder field guide"
```

---

## Task 5: Shared reference — smart-surfaced review

**Files:**
- Create: `skills/_shared/smart-surfaced-review.md`

- [ ] **Step 1: Author the reference**

Write `skills/_shared/smart-surfaced-review.md`. It defines the **objective boundary signals** used at plan time (triage) and re-checked at build time (the real gate). Required content:

- **Principle:** review is front-loaded at plan time and **optional**; surfacing is by objective signals computed from the declared change, **not** self-assessed confidence. (Cite principle 6.)
- **The seven signals** — surface a card if **any** holds:
  1. **Contract mutation** — public API/SDK signature, data/wire/DB schema, CLI flags/args/defaults, config keys. Distinguish *new* surface from a change to an *existing* one.
  2. **Destructive op** — drop/delete/truncate/overwrite/migrate/force/revert.
  3. **Sensitive zone** — by path-convention + optional repo setting (not a wordlist).
  4. **Capability/resource escalation** — new dependency, new IO, new infra, new integration.
  5. **Blast radius** — file/context thresholds, or a file edited by >1 item in this binder.
  6. **Genuine architectural novelty** — a new pattern, not merely new-to-repo.
  7. **Explicit open question / conflict / ambiguous scope.**
- **When stack detection is weak:** ask rather than guess; **log which signals were not computed** (records them in the item's `surface.signals` as e.g. `not-computed:contract`).
- **Two-pass model:** plan-time pass writes `surface {flagged, signals}` into the work item (advisory triage; user picks review all / some / flagged). Build-time pass re-runs the same signals on the **actual diff** (cite `verification-gate.md`) — that is the real gate, because an implementer can pivot into a boundary the plan never predicted.
- **Output for plan time:** how to present the surfaced list (review all / some / flagged) without making refinement feel like a chore.

- [ ] **Step 2: Run the integrity test**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS`.

- [ ] **Step 3: Commit**

```bash
git add skills/_shared/smart-surfaced-review.md
git commit -m "docs(shared): add smart-surfaced review signals reference"
```

---

## Task 6: The verification gate — reference + two kargha-owned agents (ported from keel's lean lane)

This task authors the verification-gate reference AND ports the two kargha-owned gate agents it describes. keel source is **READ-ONLY** — read `keel/.claude/agents/karta-spec-reviewer.md` and `keel/.claude/agents/safety-auditor.md`, then write *adapted, kargha-owned* copies. Do not edit, move, or run anything under `keel/`. No keel interop — the agents are standalone kargha files.

**Files:**
- Create: `skills/_shared/verification-gate.md`
- Create: `agents/kargha-acceptance-reviewer.md` (← keel `karta-spec-reviewer`)
- Create: `agents/kargha-safety-auditor.md` (← keel `safety-auditor`)

- [ ] **Step 1: Author the reference**

Write `skills/_shared/verification-gate.md` — the registry-free version of keel's automated-gate pattern, shared by `kargha-verify`, `kargha-validate`, `kargha-build`, and orchestrated by `kargha-deliver`. Required content:

- **Shape:** a read-only gate runs on the **actual diff** in a **fresh AI session** (only the worktree, the binder, and the acceptance check — no build-session context); it is independent of the implementer. **It is realized by dispatching the two kargha-owned agents below** (`kargha-acceptance-reviewer`, `kargha-safety-auditor`) plus `kargha-validate` for visual oracles.
- **Three checks (run on the actual diff):**
  1. **Acceptance** — the oracle's assertions (visual fidelity for `kargha-validate`; `unit/integration/e2e/smoke` via `kargha-acceptance-reviewer`).
  2. **Contract conformance** — against an **external artifact** (type-checker, schema, contract test), *not* the binder's own declaration (also `kargha-acceptance-reviewer`).
  3. **Boundary scan** — does the diff cross a sensitive/destructive/contract boundary the item didn't justify? (`kargha-safety-auditor`, re-running the `smart-surfaced-review.md` signals on real code.)
- **The loop:** on any finding, **kick back to build for bounded self-correction** and re-run. **Only on retry-exhaustion halt-with-CTA.** State the caps explicitly, mirroring keel's lean lane: **`kargha-safety-auditor` max 3 attempts → escalate to the human; `kargha-acceptance-reviewer` max 2 attempts → halt-with-CTA (implementer picks fix-and-rerun / declared-debt, no human escalation).** There is **no human review gate during delivery**.
- **The floor:** if the change does not even compile/type-check/lint, the gate does not pass and kargha does not auto-merge — it surfaces (cite `definition-of-done.md`).
- **Advisory-hook note:** any pre-tool hook stays advisory (exit 0) — the gate, not the hook, blocks; the gate self-corrects then escalates. (Mirrors keel's safety hook.)

- [ ] **Step 2: Port `agents/kargha-acceptance-reviewer.md`**

Read `keel/.claude/agents/karta-spec-reviewer.md` (read-only). Write `agents/kargha-acceptance-reviewer.md` as an adapted kargha-owned copy. Keep:
- frontmatter shape `name`/`description`/`tools`/`model` (set `name: kargha-acceptance-reviewer`, `tools: Read, Glob, Grep, Bash` — read-only; `model: opus`);
- the **read-only / inspection-only** discipline and "do not edit code" rule;
- the **per-assertion evidence disposition** mechanism — for each `oracle.assertions[i]`: evidence kind (inspection-verifiable / execution-required) and disposition (`CONFORMS | DEVIATION | covered-by-test | declared-debt | UNDISPOSED`);
- **contract conformance** against an external artifact (type-checker/schema/contract test), not the binder's own claim;
- the verdict tokens `CONFORMANT | DEVIATION | BLOCKED | SPEC-SUSPECT` and the `pass | concerns | blocked` envelope;
- the **max-2-attempts → halt-with-CTA** contract (no human escalation; implementer picks fix-and-rerun or a declared-debt marker).

Adapt/strip: replace all keel **invariants-registry / resolved-feature-pointer / stored-state** references with kargha's **binder + work-item `oracle`/`contract`** (read from the binder JSON on disk); drop keel's test-writer/code-reviewer lane assumptions; "declared debt" uses kargha's marker family from `skills/_shared/declared-debt.md` (Task 7). Input it receives: worktree path, binder path + item id, diff range (item branch vs integration tip).

- [ ] **Step 3: Port `agents/kargha-safety-auditor.md`**

Read `keel/.claude/agents/safety-auditor.md` (read-only). Write `agents/kargha-safety-auditor.md` as an adapted kargha-owned copy. Keep: frontmatter (`name: kargha-safety-auditor`, `tools: Read, Glob, Grep, Bash`, `model: opus`); the read-only scan-and-report frame; the verdict `PASS | VIOLATION` + `pass | concerns | blocked` envelope; the **max-3-attempts → escalate-to-human** contract. **Strip the invariants registry / fail-closed-on-unconfigured rule entirely** — the rule set is the **smart-surfaced-review signals** (`skills/_shared/smart-surfaced-review.md`, Task 5): destructive ops, sensitive zones, contract mutations, capability/resource escalation, blast radius — scanned on the **actual diff**, flagging crossings the work item didn't justify. Input: worktree path, binder path + item id, diff range.

- [ ] **Step 4: Run the integrity test**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS` (the new `agents/*.md` frontmatter is now linted; both must have `name` + `description`).

- [ ] **Step 5: Commit**

```bash
git add skills/_shared/verification-gate.md agents/kargha-acceptance-reviewer.md agents/kargha-safety-auditor.md
git commit -m "feat(verify): add verification-gate reference + kargha acceptance/safety agents (ported from keel lean lane)"
```

---

## Task 7: Shared references — integration mechanics, parallelism gates, debt, secrets, done

Five reference files capturing the orchestration mechanics and the build-time disciplines. Author each, then run the integrity test once, then commit together.

**Files:**
- Create: `skills/_shared/integration-branch.md`
- Create: `skills/_shared/parallelism-gates.md`
- Create: `skills/_shared/declared-debt.md`
- Create: `skills/_shared/secret-scan.md`
- Create: `skills/_shared/definition-of-done.md`

- [ ] **Step 1: Author `integration-branch.md`**

Required content:
- **Per-binder integration branch** `kargha/<slug>/integration`, kept in its own worktree, accumulates completed work; it is the resume record and the single reviewable assembled result; it resolves "an item depends on two parents" (child builds off the tip that already contains both).
- **Wave loop** (verbatim structure):
  1. Re-derive the ready **frontier** — items whose `depends_on` are all merged into integration. (`depends_on` is a **scheduling constraint**; cycles already rejected at binder creation.)
  2. Build the wave's items **concurrently**, each in its own worktree off the current integration tip.
  3. **Barrier, then serial merge:** each passing item, *before* merging, **re-validates its oracle against the current integration tip** (which may have advanced as wave-mates merged); on conflict/failure it rebuilds (bounded) or halts.
  4. **Post-wave integration check:** run the project's build/type-check on the new tip; on failure **revert to the wave's pre-merge tag and halt-with-CTA** (catches semantic collisions text-clean merges miss).
- **Serial merge-queue (concrete):** completed items enter a FIFO queue by completion time; the orchestrator processes one merge at a time (the "lock" is sequential processing). For each: rebase/merge the item branch onto the current integration tip → on conflict, bounded rebuild against the new tip, else halt → re-run the item oracle on the merged result → merge (ff or no-ff) → tag and write the done ref.
- **Tagging / ref scheme (concrete):**
  - item commits carry the `[kargha:item-<id>]` subject marker;
  - before a wave merges: tag `kargha/<slug>/wave-<N>-base` on the pre-merge tip (the revert anchor);
  - after the post-wave check passes: tag `kargha/<slug>/wave-<N>` on the completed tip;
  - per-item outcomes: `refs/kargha/<slug>/item-<id>/done` (→ merge commit), `.../failed` (→ failing branch tip), `.../in-progress`.
  - **Resume** reads these refs/tags; **no separate state file**.
  - **Revert-the-wave** = `git reset --hard kargha/<slug>/wave-<N>-base` on the integration branch.
- **Env-injection contract (concrete):** the test env **binds to the wave** (started once, torn down once). An item needing a stateful env gets kargha-injected isolation params from `env_contract.isolation_params` (e.g. `PORT`, `COMPOSE_PROJECT_NAME`) **only if `env_contract.supports_isolation` is true**; otherwise the item **serializes** (a "do not parallelise" trigger).
- **Honest framing:** build-parallel, **merge-serial-with-revalidation** — fast because building dominates wall-clock, not "free."

- [ ] **Step 2: Author `parallelism-gates.md` with the gates table and the blessed copy**

Include the gate table verbatim:

```markdown
| Gate | Trigger |
|-|-|
| Dependency edge | dep not yet merged (correctness) |
| Shared / order-sensitive resource | wave-mates touch the same stateful resource — inferred file-overlap or a declared annotation |
| Stateful env without injectable isolation | the repo's env command can't be parameterized |
| File-collision risk | wave-mates likely edit the same files |
| Explicit `serialize` | the binder marks items must-serialize |
```

Then include the **blessed user-facing copy** verbatim under a "How to explain parallelism to users" heading (this is the memory-blessed block — reproduce word-for-word):

```
kargha builds work items at the same time by default, to save time. It only drops back to one-at-a-time when running two together would produce a wrong or broken result. There are four such cases — everything else runs in parallel.

1. One item needs another finished first.
Item B uses something Item A creates, so B can't be built until A exists. B waits for A. This isn't a special rule — it's just correctness; B literally won't work without A. Example: the profile page can't be built until the app shell and routing exist.

2. Two items change the same order-sensitive shared thing.
A few things can only be changed one at a time, in a set order — a database migration, a shared lock file, a generated file. If two items both change one of these at once, they clobber each other. kargha catches this either by noticing both items plan to edit the same such file, or because the binder says these items share that resource. Example: two items that each add a database migration must run one after the other.

3. Two items in the same batch would edit the same files.
Even when neither item needs the other, if both edit the same file, building them side by side means their changes collide when kargha stacks the results together. So kargha runs them one at a time to avoid the merge mess. Example: two items that both edit the global stylesheet.

4. You said so.
You can mark items in the binder as "don't run these together." This is your override for interference kargha can't see — you know two items will step on each other for a reason that isn't visible in the file list.

The difference between #2 and #3, since they sound alike: #2 is about order (these changes must happen in sequence), #3 is about collision (these changes would conflict if merged). #2 and #1 keep results correct; #3 keeps the stacking clean; #4 is your manual escape hatch.
```

- [ ] **Step 3: Author `declared-debt.md`**

Required content: the declared-debt marker family (keel's `KARTA-DEFER`-style markers) for inline-declared deferrals during build — marker syntax, what a deferral records (what, why, follow-up), and that deferrals are surfaced (they do not silently disappear). Snapshot voice (principle 8): the marker states what is deferred and why, not a history.

- [ ] **Step 4: Author `secret-scan.md`**

Required content: a pre-commit secret scan run **before each commit** in build, with an **allow-list for benign matches**; what it scans (the staged diff), what it does on a hit (block the commit, surface), and how the allow-list is expressed (path/pattern entries recorded in-repo). Note it is a floor safety check, not a replacement for the project's own secret tooling.

- [ ] **Step 5: Author `definition-of-done.md`**

Required content:
- "Done" = the oracle passes on the merged result. The **definition of done is the minimum check you'd want CI to run** — the project's *real* CI-facing checks (its tests/build/type-check) plus item-specific assertions, **not** a check the model grades itself on.
- **Floor:** the change must at least compile/type-check/lint clean; below that kargha does **not** auto-merge — it surfaces.
- **Opt-out:** explicit and **recorded** (the binder's `oracle.opt_out` + `reason`), never silent; kargha reports what it leaves unchecked.
- Include the **blessed user-facing copy** verbatim under a "How to explain review to users" heading (memory-blessed — reproduce word-for-word):

```
You don't have to review everything. kargha shows you what's worth a look.

Every work item carries a check that proves it's done — a real one, the kind you'd want CI to run, not a box to tick. kargha writes it and runs it for you.

When planning is done, kargha hands you a short list: the items worth your eyes — anything risky, unusual, or touching something sensitive — and stays quiet about the routine ones. Review is a short list, not a slog.

You keep the controls:
- Want to see everything? You can.
- A check doesn't fit an item? Turn it off — kargha tells you what that leaves unchecked.

The idea is simple: kargha puts the decisions that matter in front of you early, then gets out of your way.
```

- [ ] **Step 6: Run the integrity test**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS`.

- [ ] **Step 7: Commit**

```bash
git add skills/_shared/integration-branch.md skills/_shared/parallelism-gates.md skills/_shared/declared-debt.md skills/_shared/secret-scan.md skills/_shared/definition-of-done.md
git commit -m "docs(shared): add integration, parallelism, debt, secret-scan, and done references"
```

---

## Task 8: kargha-plan — frontmatter, intro, and project configuration (keel-refine light)

Transform `kargha-plan` from a frontend-ticket emitter into a stack-agnostic binder synthesizer. This task does the head of the file (through Project configuration); Task 9 does the workflow phases.

**Files:**
- Modify: `skills/kargha-plan/SKILL.md:1-48` (frontmatter + intro + adaptation + project-config head)

- [ ] **Step 1: Rewrite the frontmatter**

Replace the `name`/`description` frontmatter (`skills/kargha-plan/SKILL.md:1-4`) so `description` says: analyze a problem/feature description and/or a design mock or non-functional prototype (keel-refine inputs) and **synthesize a validated binder** of work items for ad-hoc orchestration; **stack-agnostic** (UI is one stack among many); emits `.kargha/binders/<slug>.json`. Trigger phrases: "plan this with kargha", "synthesize a binder", "break this work into a binder", "kargha-plan this feature". Keep `name: kargha-plan`. Keep it within the existing description-length style.

- [ ] **Step 2: Rewrite the intro + "How this skill adapts"**

Replace the frontend-specific intro (`:6-24`) with: kargha-plan is **keel-refine light** — it ingests intent (a problem/feature description, optionally a design mock or non-functional prototype) and, **without fail, synthesizes a binder**. State the four keel-refine batteries it **drops** (bootstrap-gate preflight → a light never-blocking repo detect; multi-binder partition → suggest a breakdown into one binder's items, one binder/run in V1; roundtable decomposition) and what it **keeps** (intent ingestion widened to design exports; the synthesis subagent; a minimal interview loop; commit-on-verb) and **replaces** (the mandatory per-card walk → smart-surfaced review). Keep the design-input contract note **only** as a stack-specific aside (when the input is a Claude Design / runtime-JSX export, the UI-analysis path applies) — not as a precondition for the whole skill.

- [ ] **Step 3: Rewrite Project configuration to be stack-agnostic**

Replace the frontend-only config table (`:26-48`) with a **best-effort, never-required** repo-context resolution: detect the stack (frontend, backend, CLI, data pipeline, library/SDK, IaC, mobile, ML, docs — no domain assumption), the toolchain commands (lint/test/build/typecheck), the CI-facing checks (for oracles), the env command (for `env_contract`), and any repo direction docs (architecture/decisions) — **ask when context is thin, never block on missing docs.** Keep the DTCG/token and component-library settings as a **conditional UI annex** ("when the resolved stack has a design/token surface"), pointing at `references/dtcg-tokens.md`. State that resolved values feed the binder's `design_facts`, `env_contract`, and each oracle's `command`.

- [ ] **Step 4: Run the integrity test**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS` (frontmatter still valid; any new `references/...` citations must resolve — only cite files that exist).

- [ ] **Step 5: Commit**

```bash
git add skills/kargha-plan/SKILL.md
git commit -m "feat(plan): stack-agnostic intro + project config (keel-refine light)"
```

---

## Task 9: kargha-plan — workflow phases (synthesis → review → emit → validate → commit)

Rewrite the workflow body so the output is a **validated, committed binder**, not frontend tickets. Copy the shared references this skill needs.

**Files:**
- Modify: `skills/kargha-plan/SKILL.md:49-554` (Workflow through Gotchas)
- Create (copy): `skills/kargha-plan/references/smart-surfaced-review.md`, `skills/kargha-plan/references/definition-of-done.md`, `skills/kargha-plan/references/binder-reference.md` *(already in this skill's references from Task 4)*

- [ ] **Step 1: Copy the shared references this skill cites**

```bash
cp skills/_shared/smart-surfaced-review.md skills/kargha-plan/references/smart-surfaced-review.md
cp skills/_shared/definition-of-done.md skills/kargha-plan/references/definition-of-done.md
```
(`binder-reference.md`, `binder-schema.json`, `example-binder.json`, and `scripts/validate_binder.py` already live in this skill from Tasks 1–4.)

- [ ] **Step 2: Rewrite the Workflow phases**

Replace `:49-540` with these phases (keep the existing "Phase N — …" heading idiom; preserve the subagent-brief pattern the skill already uses):

| Phase | Content |
|-|-|
| **Phase 0 — Ingest intent (light gate)** | Accept keel-refine inputs: a problem/feature description, and/or a design mock or non-functional prototype path. The only hard requirement is *some* statement of intent. Light repo detect (never-blocking). If the input is a Claude Design / runtime-JSX export, additionally run the UI-format gate (keep the existing format-gate logic as a **conditional** branch, citing the design-input contract aside). Resolve where the binder lands: detect → ask → default `.kargha/binders/<slug>.json`. |
| **Phase 1 — Best-effort repo + stack understanding** | Reuse the Explore-subagent pattern, but stack-agnostically: understand the stack, conventions, toolchain, CI-facing checks, and env command. Interview the user only where context is genuinely thin. Never require architecture/decision docs; read them if present. (UI annex: when a component library/token system exists, run the existing library/token inventory passes — cite `references/dtcg-tokens.md`.) |
| **Phase 2 — Synthesize the binder (synthesis subagent, main thread owns)** | Decompose the intent into work items. For each item set `id`, `title`, `estimate`, `depends_on`, `contract`, and the **oracle** (per `references/definition-of-done.md`: a real CI-facing check + assertions, or an explicit recorded `opt_out`+reason). Set `serialize`/`shared_resources` for order-sensitive/collision-prone items. Populate UI-only fields (`design_reference`, `component_map`, `icon_map`, `token_changes`) **only** when the stack has that surface. Fill binder-level `slug`, `motivation`, `scope`, `design_facts`, `env_contract`, and `token_manifest` (only if a token system exists). Do not delegate the synthesis judgment. **keel test-writer discipline (inspiration):** keep each oracle assertion traceable to the item's `contract` — if an assertion references a field/shape the item doesn't declare, flag the contract gap now rather than emitting an unverifiable assertion. |
| **Phase 3 — Smart-surfaced review (the one human-in-the-loop point)** | Per `references/smart-surfaced-review.md`: compute the seven boundary signals per item, write `surface {flagged, signals}`, and offer the user **review all / some / flagged**. Front-load decisions here so delivery is hands-off. Do not surface oracles for routine items. Log uncomputed signals. |
| **Phase 4 — Cost education** | When scope is large, tell the user plainly that this scope will cost time and money before they see anything tangible, and suggest a smaller first slice. Educate; do not forbid. |
| **Phase 5 — Emit + validate + commit** | Write the binder JSON to the resolved location. **Validate it:** run `uv run skills/kargha-plan/scripts/validate_binder.py --binder <path>` and do not proceed on failure (fix and re-validate). Then commit on the `commit` verb (the binder is committed only at run boundaries; it is read-only to build steps thereafter). A **single-work-item binder** is noted as eligible to skip deliver (the "just this once" hatch — go straight to build). |
| **Phase 6 — Report back** | Binder path, work-item count + ids in dependency order, the dependency chain, surfaced items, opted-out items (from the validator's opt-out summary), and the eligible first wave (items with no deps). |

- [ ] **Step 3: Rewrite the Gotchas**

Replace the frontend gotchas (`:542-554`) with binder-centric ones: the binder is always synthesized; UI fields are conditional, not universal; the oracle is a real CI-facing check, not self-grading; opt-out is explicit + recorded; don't delegate synthesis judgment; validate before commit; one binder/run in V1; the binder is read-only to build once committed.

- [ ] **Step 4: Validate the example + run the integrity test**

Run: `uv run skills/kargha-plan/scripts/validate_binder.py --binder skills/kargha-plan/references/example-binder.json`
Expected: `VALID. 3 work items; 1 opted out...`
Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS` (the new `references/smart-surfaced-review.md` and `definition-of-done.md` copies now resolve).

- [ ] **Step 5: Commit**

```bash
git add skills/kargha-plan/SKILL.md skills/kargha-plan/references/smart-surfaced-review.md skills/kargha-plan/references/definition-of-done.md
git commit -m "feat(plan): synthesize+validate+commit a binder (keel-refine light)"
```

---

## Task 10: kargha-build — stack-agnostic item builder, integration-branch merge, no PR

Rewrite `kargha-build` to consume **one work item from a binder** (not a frontend ticket), implement it stack-agnostically, end at **tagged commits merged into the integration branch** (no PR), run the acceptance loop via verify/validate, scan for secrets, and declare debt inline.

**Files:**
- Modify: `skills/kargha-build/SKILL.md` (whole file)
- Create (copy): `skills/kargha-build/references/{binder-reference.md, definition-of-done.md, declared-debt.md, secret-scan.md, integration-branch.md, smart-surfaced-review.md, verification-gate.md}`

- [ ] **Step 1: Copy the shared references this skill cites**

```bash
cp skills/kargha-plan/references/binder-reference.md skills/kargha-build/references/binder-reference.md
cp skills/_shared/definition-of-done.md skills/kargha-build/references/definition-of-done.md
cp skills/_shared/declared-debt.md skills/kargha-build/references/declared-debt.md
cp skills/_shared/secret-scan.md skills/kargha-build/references/secret-scan.md
cp skills/_shared/integration-branch.md skills/kargha-build/references/integration-branch.md
cp skills/_shared/smart-surfaced-review.md skills/kargha-build/references/smart-surfaced-review.md
cp skills/_shared/verification-gate.md skills/kargha-build/references/verification-gate.md
```
(`worktree-safety.md` already exists in this skill; keep it. `dtcg-tokens.md`, `ci-policy.md`, `policy-yagni.md`, `design-validation-loop.md` stay as the UI annex.)

- [ ] **Step 2: Rewrite the frontmatter + adaptation head**

Replace `:1-58`. New `description`: implement one work item from a kargha binder in an isolated git worktree — stack-agnostic (frontend, backend, CLI, data, IaC, …) — running the project's lint/test/build plus the item's acceptance check, tagging commits, and **merging into the per-binder integration branch (no PR)**. Trigger phrases: "build this binder item", "implement work item `<id>`", "kargha-build `<binder> <id>`". Replace the frontend-only "How this skill adapts" with stack-agnostic adaptation; demote the design-validation/data-layer rows to a **conditional UI/data annex**.

- [ ] **Step 3: Rewrite the input + gates (Phase 0–1)**

The input is now `(binder path, work-item id)` — extract the item from the validated binder, not a ticket file. Keep the **always-on mutation guard** (`references/worktree-safety.md`) verbatim. Replace the three ticket gates with binder-item gates: (1) the binder validates (`validate_binder.py`); (2) the item id exists; (3) the item's `depends_on` are all merged into integration (check the `refs/kargha/<slug>/item-*/done` refs per `references/integration-branch.md`) — unmet deps halt-with-CTA. Drop the assignee/status/ticketing identity machinery (no ticketing system); keep a light git identity resolution for commit authorship only. Preserve the **ticketless inspection-aid mode** as-is (it's still a valid escape hatch).

- [ ] **Step 4: Rewrite worktree + implement (Phase 4)**

Keep the worktree creation + mutation-guard discipline, but branch **off the current integration tip** (`kargha/<slug>/integration`), not the default branch, and name the branch to embed the item id. Implement the item against resolved conventions, stack-agnostically. Keep the UI-specific implementation rules (component map, icon imports, token rules) as a **conditional annex** that applies only when those binder fields are present. Add: **declare deferrals inline** per `references/declared-debt.md`. **keel-implementer discipline (inspiration):** never weaken or edit the item's `oracle`/acceptance to make it pass; on a genuine oracle-or-contract conflict, halt-with-CTA rather than silently diverging (principle 6: code/specs/tests win).

- [ ] **Step 5: Rewrite the gates → acceptance loop (Phase 5–7)**

Replace the lint/test + data-layer + design-validation phases with:
1. **Deterministic gate** — run the project's `<lint>`/`<test>`/`<typecheck>`/`<build>` (the floor; cite `references/definition-of-done.md`).
2. **Acceptance loop** — invoke the gate for the item's oracle: `kargha-validate` when `oracle.type == visual`, else `kargha-verify`. The gate is read-only and **kicks findings back to build** (this skill) for bounded self-correction per `references/verification-gate.md` (boundary gate max 3, contract gate max 2); only on exhaustion does it halt-with-CTA. Keep the conditional UI design-validation annex (`references/design-validation-loop.md`) as the mechanism `kargha-validate` uses.

- [ ] **Step 6: Rewrite completion (Phase 9–10): tag, secret-scan, merge — no PR**

Replace "open the PR" with:
- Before each commit, run the **secret scan** (`references/secret-scan.md`); block + surface on a hit.
- Commit with the `[kargha:item-<id>]` subject marker.
- **Merge into the integration branch** per `references/integration-branch.md` (re-validate the oracle against the current tip before merging; on conflict, bounded rebuild or halt), then write `refs/kargha/<slug>/item-<id>/done`. **No PR** — the user reviews and merges the integration branch.
- Report: worktree path, item id, the integration tip it merged to, acceptance result, declared-debt summary, secret-scan status. Preserve the **failing item's worktree** and print its path on halt.

- [ ] **Step 7: Rewrite the Gotchas** to match: no PR; merge into integration; commit marker; secret scan before commit; acceptance via verify/validate with kickback; UI rules are conditional; the binder is read-only to build; deps must be merged before pickup.

- [ ] **Step 8: Run the integrity test**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS` (all seven new `references/*` copies resolve; frontmatter valid).

- [ ] **Step 9: Commit**

```bash
git add skills/kargha-build/SKILL.md skills/kargha-build/references/
git commit -m "feat(build): stack-agnostic item build, integration merge, acceptance loop (no PR)"
```

---

## Task 11: kargha-verify — new behavioral acceptance gate

Author the new `kargha-verify` skill: the behavioral counterpart to `kargha-validate`, implementing the keel-faithful read-only gate.

**Files:**
- Create: `skills/kargha-verify/SKILL.md`
- Create (copy): `skills/kargha-verify/references/{verification-gate.md, definition-of-done.md, smart-surfaced-review.md, binder-reference.md}`

- [ ] **Step 1: Create the references**

```bash
mkdir -p skills/kargha-verify/references
cp skills/_shared/verification-gate.md skills/kargha-verify/references/verification-gate.md
cp skills/_shared/definition-of-done.md skills/kargha-verify/references/definition-of-done.md
cp skills/_shared/smart-surfaced-review.md skills/kargha-verify/references/smart-surfaced-review.md
cp skills/kargha-plan/references/binder-reference.md skills/kargha-verify/references/binder-reference.md
```

- [ ] **Step 2: Author the SKILL.md**

Write `skills/kargha-verify/SKILL.md`. Frontmatter `name: kargha-verify`; `description`: verify one built work item against its behavioral acceptance check (oracle types `unit/integration/e2e/smoke`) in a **read-only, fresh session** on the **actual diff**; check acceptance + external-contract conformance + boundary crossings; **kick findings back to build** and escalate to the human only on retry-exhaustion. Trigger phrases: "verify this work item", "run the behavioral gate", "check acceptance for `<id>`".

`kargha-verify` is the **thin orchestrator** for the behavioral gate: it **dispatches the two kargha-owned agents** (`agents/kargha-acceptance-reviewer`, `agents/kargha-safety-auditor` — authored in Task 6, dispatched by name, plugin-global, NOT copied into this skill's references) in fresh sessions, then aggregates their verdicts and drives the kickback/escalation loop (per `references/verification-gate.md`). It is read-only and never edits. Body:

| Phase | Content |
|-|-|
| Inputs | the worktree path, the binder + item id (for the oracle), and the diff range (item branch vs the integration tip). |
| Phase 0 — Prerequisites | fresh session with only the worktree/binder/oracle; resolve the repo-provided **pre-verify env command** bound to the wave; hard-gate on a missing env if the oracle needs one. |
| Phase 1 — Acceptance + contract (dispatch `kargha-acceptance-reviewer`) | pass it the worktree/binder/item-id/diff-range; it does per-assertion disposition of the oracle and external-contract conformance, returns `CONFORMANT \| DEVIATION \| BLOCKED \| SPEC-SUSPECT`. Loop: on DEVIATION, kick back to build (max 2), then halt-with-CTA. |
| Phase 2 — Boundary scan (dispatch `kargha-safety-auditor`) | it re-runs the `smart-surfaced-review` signals on the actual diff, returns `PASS \| VIOLATION`. Loop: on VIOLATION, kick back to build (max 3), then escalate to the human. |
| Phase 3 — Aggregate verdict | combine both agents' verdicts into a `pass \| concerns \| blocked` envelope; on pass, report PASS to the caller (build/deliver); on exhaustion, halt-with-CTA / escalate per the caps above. |
| Gotchas | floor = compile/type-check/lint (cite `references/definition-of-done.md`); read-only; fresh session (no build-session context); the agents — not this skill — do the reading; escalate only on exhaustion; no human gate during delivery. |

- [ ] **Step 3: Run the integrity test**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS` (new skill discovered; frontmatter + references resolve).

- [ ] **Step 4: Commit**

```bash
git add skills/kargha-verify/
git commit -m "feat(verify): add behavioral acceptance gate with kickback/escalate"
```

---

## Task 12: kargha-validate — acceptance-grounded, kickback framing

Light modification: `kargha-validate` now answers to the **oracle's assertions** and frames its report as kickback input to build. The capture/compare scripts are unchanged.

**Files:**
- Modify: `skills/kargha-validate/SKILL.md` (intro + Inputs + Phase 3 framing + Gotchas)
- Create (copy): `skills/kargha-validate/references/{verification-gate.md, definition-of-done.md}`

- [ ] **Step 1: Create the references**

```bash
mkdir -p skills/kargha-validate/references
cp skills/_shared/verification-gate.md skills/kargha-validate/references/verification-gate.md
cp skills/_shared/definition-of-done.md skills/kargha-validate/references/definition-of-done.md
```

- [ ] **Step 2: Reframe intro + Inputs (`:6-31`)**

State that `kargha-validate` is the **visual** acceptance gate (oracle `type: visual`): it is read-only and reports discrepancies as **kickback input** for `kargha-build` to self-correct; it never fixes. Add an optional input: the **oracle assertions** for this view (e.g. "zero critical/major discrepancies at 1440x900"), so the report can state pass/fail against the item's acceptance, not just describe drift. Cite `references/verification-gate.md` and `references/definition-of-done.md`. Keep the design-input contract and the playwright/uv prerequisites verbatim.

- [ ] **Step 3: Add an acceptance verdict to the report (Phase 3)**

In Phase 3, keep the existing `STATUS`/`DISCREPANCIES`/`TOKEN_DRIFT`/`MISSING_ELEMENTS`/`EXTRA_ELEMENTS` schema unchanged, and add a single `ACCEPTANCE: pass | fail` line derived from the oracle assertions when provided (pass = the assertions hold, e.g. zero critical/major). When no assertions are passed, omit the line (back-compatible). Do **not** add fixes/recommendations (the existing read-only rule stands).

- [ ] **Step 4: Update Gotchas** — add: validation is the **visual acceptance gate**; its report is kickback input, not a fix; the oracle's assertions set the bar. Keep the existing read-only/playwright gotchas.

- [ ] **Step 5: Run the integrity test**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS`.

- [ ] **Step 6: Sanity-check the unchanged capture script still self-tests**

Run: `uv run skills/kargha-validate/scripts/serve_design.py --self-test`
Expected: the existing self-test passes (we changed no scripts).

- [ ] **Step 7: Commit**

```bash
git add skills/kargha-validate/SKILL.md skills/kargha-validate/references/
git commit -m "feat(validate): acceptance-grounded visual gate with kickback framing"
```

---

## Task 13: kargha-deliver — wave scheduler over the integration branch

Author the new orchestration skill: schedule the binder's work items into parallel waves on the integration branch, calling `kargha-build` per item.

**Files:**
- Create: `skills/kargha-deliver/SKILL.md`
- Create (copy): `skills/kargha-deliver/references/{integration-branch.md, parallelism-gates.md, binder-reference.md, verification-gate.md, definition-of-done.md}`

- [ ] **Step 1: Create the references**

```bash
mkdir -p skills/kargha-deliver/references
cp skills/_shared/integration-branch.md skills/kargha-deliver/references/integration-branch.md
cp skills/_shared/parallelism-gates.md skills/kargha-deliver/references/parallelism-gates.md
cp skills/_shared/verification-gate.md skills/kargha-deliver/references/verification-gate.md
cp skills/_shared/definition-of-done.md skills/kargha-deliver/references/definition-of-done.md
cp skills/kargha-plan/references/binder-reference.md skills/kargha-deliver/references/binder-reference.md
```

- [ ] **Step 2: Author the SKILL.md**

Write `skills/kargha-deliver/SKILL.md`. Frontmatter `name: kargha-deliver`; `description`: deliver a kargha binder by building its work items in **parallel waves** onto a per-binder **integration branch**, serializing only where correctness or collision demands it; resume is git-native; ends at the assembled integration branch (no PR). Trigger phrases: "deliver this binder", "run the binder", "kargha-deliver `<binder>`". Body:

| Phase | Content |
|-|-|
| Phase 0 — Preflight | Validate the binder (`validate_binder.py`). **Single-work-item binder ⇒ skip deliver**, hand straight to `kargha-build` (the "just this once" hatch). Detect leftovers from a prior run (refs/tags) and offer to resume or clear. |
| Phase 1 — Integration branch | Create/locate `kargha/<slug>/integration` in its own worktree per `references/integration-branch.md`. |
| Phase 2 — Wave loop | Run the four-step wave loop verbatim from `references/integration-branch.md`: derive frontier → build wave concurrently (host's parallel primitive; serial fallback) → barrier + serial merge with re-validation → post-wave integration check (revert+halt on failure). Apply the **gates** from `references/parallelism-gates.md` to decide what serializes (dependency edge, shared/order-sensitive resource, stateful env without injectable isolation, file-collision risk, explicit `serialize`). |
| Phase 3 — Env binding | Start the project's env (`env_contract.command`) once per wave; inject `isolation_params` for items that need a stateful env **iff** `supports_isolation`; else serialize them. Tear down once. |
| Phase 4 — Lifecycle | Partial-wave failure: passing items merge; the failing item halts-with-CTA; only dependents wait; the rest of the frontier continues; user may revert-the-wave (`git reset --hard kargha/<slug>/wave-<N>-base`) or continue. Cleanup: remove successful/abandoned temp worktrees and tear down the env; **preserve the failing worktree** and print its path. |
| Phase 5 — Cost education | Echo the plan-time cost note when scope is large (cite `kargha-plan`); nudge toward a small first slice; no hard cap. |
| Phase 6 — Report | Waves run, items merged (with ids and the integration tip), items halted with causes + worktree paths, and that the integration branch is the single reviewable result. |
| Gotchas | build-parallel/merge-serial-with-revalidation (not free); the binder is **immutable while a wave runs**, mutable only between waves; backlog curation is the user's job; resume is git-native (no state file); the human enters only on escalation. |

- [ ] **Step 3: Run the integrity test**

Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS`.

- [ ] **Step 4: Commit**

```bash
git add skills/kargha-deliver/
git commit -m "feat(deliver): add parallel-wave scheduler over the integration branch"
```

---

## Task 14: Packaging — plugin manifests, marketplace, README

Update packaging so the plugin describes five skills **and the two kargha-owned agents**, with the orchestration framing.

**Files:**
- Modify: `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`, `.agents/plugins/marketplace.json`
- Modify: `README.md`

- [ ] **Step 1: Read the manifests + confirm agent discovery**

Read `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json`, `.claude-plugin/marketplace.json`, and `.agents/plugins/marketplace.json` to see whether skills/agents are enumerated (the Claude `plugin.json` does not enumerate skills — they're auto-discovered — but the marketplace/codex manifests may). **Confirm the top-level `agents/` directory (Task 6's `kargha-acceptance-reviewer.md` / `kargha-safety-auditor.md`) is discovered by the plugin** the same way top-level `skills/` is: if the manifest format requires an explicit `agents` path/array (some do), add it; if agents are auto-discovered from `agents/`, no manifest change is needed beyond confirming it. If unsure of the convention, check current Claude Code plugin docs (e.g. via the claude-code-guide agent) — do not guess silently.

- [ ] **Step 2: Update `.claude-plugin/plugin.json`**

Replace the `description` with the orchestration framing (synthesize a binder; deliver it in parallel waves onto an integration branch; verify each item; stack-agnostic; five skills `kargha-plan`/`kargha-deliver`/`kargha-build`/`kargha-verify`/`kargha-validate`). Bump `version` `0.1.0` → `0.2.0`. Update `keywords`: drop the frontend-only skew, add `orchestration`, `binder`, `parallel`, `worktree`, keep `stack-agnostic`, `design-validation`, `dtcg`.

- [ ] **Step 3: Mirror into `.codex-plugin/plugin.json`** with the same description/version/keywords (matching that file's shape).

- [ ] **Step 4: Update the marketplace manifests** — if either enumerates skills, add `kargha-deliver` and `kargha-verify`; update any description string to the new framing. If they only reference the plugin, just update the description.

- [ ] **Step 5: Rewrite `README.md`**

Rewrite for the orchestration framework. Keep the loom epigraph. Required sections: the pipeline (`plan → deliver → build`, with `verify`/`validate` as gates; mermaid or ascii); **the binder** (link `skills/kargha-plan/references/binder-reference.md`); the five skills (one para each); **the two kargha-owned agents** (`kargha-acceptance-reviewer`, `kargha-safety-auditor`) — one line each, noting they are the verification gates, ported registry-free from keel's lean-lane agents and dispatched by `kargha-verify`/`kargha-build`; the new connecting contract is the **binder** (not the ticket template); cross-cutting concepts (stack-agnostic; ad-hoc, repo-directed, no setup/invariants; parallel-by-default with the gates; git-native resume; no PR — ends at the integration branch); Requirements; Install (namespaced `kargha:kargha-plan` … `kargha:kargha-verify`). Remove the frontend-only "three skills"/"ticket contract" framing.

- [ ] **Step 6: Run the integrity test + JSON validity**

Run: `uv run python -c "import json; [json.load(open(p)) for p in ['.claude-plugin/plugin.json','.codex-plugin/plugin.json','.claude-plugin/marketplace.json','.agents/plugins/marketplace.json']]; print('json OK')"`
Expected: `json OK`.
Run: `uv run scripts/validate_plugin.py --self-test`
Expected: `PASS`.

- [ ] **Step 7: Commit**

```bash
git add .claude-plugin/ .codex-plugin/ .agents/ README.md
git commit -m "docs: repackage kargha as a five-skill orchestration framework"
```

---

## Task 15: How-to guide (blessed copy) + final hardening

Add the user-facing how-to with the blessed verbatim copy, verify the `_shared/` copies match their skill copies, and run all self-tests once more.

**Files:**
- Create: `docs/how-to/parallelism-and-review.md`
- Create: `scripts/check_shared_copies.py`

- [ ] **Step 1: Author the how-to guide**

Write `docs/how-to/parallelism-and-review.md` in kargha's plain-language register. It carries **both blessed blocks verbatim** — the parallelism-gates block and the review/definition-of-done block (copy them word-for-word from `skills/_shared/parallelism-gates.md` and `skills/_shared/definition-of-done.md`, which hold the blessed copy from Task 7). Add a one-paragraph intro framing them for users (when does kargha parallelize, and how does review work) and nothing that contradicts the blessed text.

- [ ] **Step 2: Write a shared-copy drift check**

Write `scripts/check_shared_copies.py`:

```python
# /// script
# requires-python = ">=3.11"
# dependencies = []
# ///
"""Assert each skills/*/references/<f> copied from skills/_shared/<f> matches its source.

Usage: uv run scripts/check_shared_copies.py --self-test
"""
from __future__ import annotations
import argparse, sys
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent
SHARED = ROOT / "skills" / "_shared"


def check() -> list[str]:
    errors: list[str] = []
    shared = {p.name: p.read_text() for p in SHARED.glob("*.md")}
    for ref in (ROOT / "skills").glob("*/references/*.md"):
        if ref.parent.parent.name == "_shared":
            continue
        if ref.name in shared and ref.read_text() != shared[ref.name]:
            errors.append(f"{ref.relative_to(ROOT)} drifted from skills/_shared/{ref.name}")
    return errors


def main() -> int:
    argparse.ArgumentParser().parse_known_args()
    errors = check()
    if errors:
        print("SHARED COPIES: DRIFT")
        for e in errors:
            print(f"  - {e}")
        return 1
    print("SHARED COPIES: IN SYNC")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 3: Run the drift check**

Run: `uv run scripts/check_shared_copies.py --self-test`
Expected: `SHARED COPIES: IN SYNC` (every copied reference still matches its `_shared/` source). If any drifted, re-copy from `_shared/` (the source of truth) and re-run.

- [ ] **Step 4: Run the full test suite**

Run all three, expecting success:
```bash
uv run scripts/validate_plugin.py --self-test
uv run scripts/check_shared_copies.py --self-test
uv run skills/kargha-plan/scripts/validate_binder.py --self-test
```
Expected: `PLUGIN INTEGRITY: PASS`, `SHARED COPIES: IN SYNC`, `6/6 checks passed`.

- [ ] **Step 5: Commit**

```bash
git add docs/how-to/parallelism-and-review.md scripts/check_shared_copies.py
git commit -m "docs: add parallelism+review how-to and shared-copy drift check"
```

---

## Self-Review (run after implementing all tasks)

- **Spec coverage:** map each spec section to a task — §2 principles (Tasks 4–7 references + skill bodies), §3 pipeline (README/plan/deliver), §4 binder (Tasks 1–4), §5 kargha-plan (Tasks 8–9), §6 smart-surfaced review (Task 5 + plan Phase 3 + the `kargha-safety-auditor` agent's boundary scan), §7 deliver (Task 13), §8 lifecycle (Task 13 Phase 4 + build halt/cleanup), §9 build+verify — the keel-lean-lane gate (Tasks 10–12 + the two ported agents in Task 6), §10 out-of-scope (header), §11 migration (Tasks 8–13), §12 deferred mechanisms (plan-level decisions → schema + `integration-branch.md`).
- **keel-agent reuse:** the two kargha-owned agents (`kargha-acceptance-reviewer` ← `karta-spec-reviewer`, `kargha-safety-auditor` ← `safety-auditor`) are ported in Task 6, dispatched by `kargha-verify` (Task 11), linted by `validate_plugin.py` (Task 3), and packaged in Task 14; inspiration-only borrowings (implementer, test-writer) are noted in Tasks 9–10.
- **Placeholder scan:** no "TBD"/"handle edge cases"; prose tasks carry section blueprints + verbatim contract content (schema, gates table, blessed copy).
- **Type consistency:** the field names used across tasks match `binder-schema.json` exactly (`slug`, `work_items`, `oracle.opt_out`/`reason`, `env_contract.supports_isolation`, `surface.flagged/signals`); the tag/ref scheme strings (`kargha/<slug>/wave-<N>-base`, `refs/kargha/<slug>/item-<id>/done`, `[kargha:item-<id>]`) are identical in `integration-branch.md`, build, and deliver.
