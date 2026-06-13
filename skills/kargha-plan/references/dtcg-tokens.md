# DTCG token systems — planning settings

Loaded by `kargha-plan` only when the **Token/theme system** is a W3C DTCG design-token file. Resolve these extra settings so Phases 2, 4a.2, 4c, and the ticket's **Token Changes** section author tier-correct, build-actionable guidance.

### DTCG token systems (extra settings, resolved only when the token system is DTCG)

When the **Token/theme system** is a design-token file in W3C DTCG format (JSON leaves carrying `$value`/`$type`, usually alongside a token build tool such as Style Dictionary / Terrazzo and a build script), resolve these additional settings so Phases 2, 4a.2, 4c, and the ticket's **Token Changes** section can author tier-correct, build-actionable token guidance. (These mirror the implementation skill `kargha-build`'s DTCG settings — keeping the two in sync is what lets a plan-authored token addition be applied without re-asking the user at build time.)

| Setting | What it is | How to resolve |
|-|-|-|
| `<token-source-dir>` | The DTCG JSON files — the only editable token surface | Detect from the build script's `source` config, or glob for `$value`-bearing JSON |
| `<token-build-command>` | Regenerates artifacts from the JSON | Detect from package scripts (e.g. `build:tokens`) |
| `<generated-token-artifacts>` | Build outputs (e.g. a generated `tokens.css`) — read-only; tickets must never hand-edit them | Detect from the build script's output path or a "GENERATED" header |
| Tier convention | Which tiers exist (primitive/semantic/component) and which one new code may consume | Read the token dir's README/conventions doc; a common rule is "consume semantic, never primitives", including documented transitional-debt carve-outs |
| Name-resolution rule | How a token path maps to the emitted variable name — often a vendor `$extensions` key (e.g. `com.<project>.cssName`), else path-derived | Scan `$extensions` keys on a few tokens; the key is project-specific — never assume one |
| Theme-context selector | How alternate contexts are activated (e.g. `[data-theme="dark"]`, a `.dark` class, `prefers-color-scheme`) | Detect from the build config's per-context selectors and the source files (e.g. a `semantic.dark.json`) |
