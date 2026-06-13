---
name: validate-design
description: Compare a running frontend implementation against design HTML files exported from Claude Design. Opens both the live app (at the caller-provided app URL) and the served design prototype in playwright-cli, captures screenshots and DOM snapshots, then reports structured discrepancies across layout, color, typography, spacing, component structure, and visual hierarchy. Validates one view per invocation — the calling pipeline loops for multiple views. Invoke when validating implementation fidelity — trigger phrases include "validate against the design", "compare implementation to design", "check design fidelity", "does this match the design", "visual QA against design HTML", or any request to diff what's running vs the design prototype files.
---



Compare a single frontend view against its design prototype exported from Claude Design. This skill is a **validation step** in a larger pipeline — it reports discrepancies but never fixes them. The caller (or pipeline) decides what to act on.

The skill uses two subagents for fresh-context isolation:
1. **Capture subagent** — uses playwright-cli to screenshot and snapshot both the design and the implementation
2. **Comparison subagent** — receives only the captured data and produces a structured discrepancy report

## How this skill adapts to your project

This skill is **stack-agnostic about the app under test**. It does not assume a framework, dev-server port, or font stack — those arrive as inputs from the caller (e.g. the `build-frontend-generic` pipeline, which resolves them per project). The design side assumes a **runtime-JSX export** (a Claude Design-style HTML prototype that compiles JSX via Babel) — that is the supported design-input format, matching the planning/implementation skills.

Where this document shows a concrete value (e.g. `http://localhost:3000`, `#__next`, `pnpm nx dev frontend`, the Inter font), treat it as an **example**, not a requirement — it comes from the caller's resolved settings. The browser mechanism (`playwright-cli`) and the two-subagent capture/compare structure are this skill's own internals and stay fixed.

## Inputs

The caller's prompt must include:

- **Design HTML file path** — absolute or relative to the workspace root (e.g., `signal-test/project/Signal.html`). Can be a directory — the skill picks the best HTML file.
- **App base URL** (optional) — where the running app is served (e.g., `http://localhost:3000`, `http://localhost:5173`). Defaults to `http://localhost:3000`. The caller (e.g. `build-frontend-generic`) passes its resolved dev-server URL here.
- **App route** — the URL path in the running app (e.g., `/feed`, `/projects/123`); appended to the base URL.
- **Design navigation instructions** — explicit steps to reach the target view in the design prototype (e.g., "click the Feed item in the sidebar"). The design uses client-side `useState` page switching, not URL routes. The skill does NOT auto-discover navigation.
- **App navigation instructions** (optional) — additional steps beyond navigating to the route (e.g., "click the first project row to open the detail panel")
- **Viewport** (optional) — width x height in pixels. Defaults to 1440x900.
- **Focus areas** (optional) — narrow the comparison (e.g., "focus on typography and spacing only")

To validate an **alternate theme context** (e.g. dark mode), the caller includes the context-switch steps in the Design navigation and App navigation (e.g. "click the theme toggle") so both sides render in that context — this skill has no separate theme parameter; it reaches the context the same way it reaches any view, through navigation.

## Workflow

### Phase 0 — Prerequisites (hard gates)

All checks must pass. Any failing is an immediate hard stop — no prompting, no workarounds.

**0a. Design files exist.** Resolve the design path from the caller's prompt.

```bash
design_path="<caller-provided-path>"

# If it's a directory, find HTML files
if [ -d "$design_path" ]; then
  # Prefer standalone variant (no CDN dependency)
  design_file=$(find "$design_path" -maxdepth 2 -name '*standalone*.html' | head -1)
  # Fall back to any HTML file
  if [ -z "$design_file" ]; then
    design_file=$(find "$design_path" -maxdepth 2 -name '*.html' ! -name '*print*' | head -1)
  fi
else
  design_file="$design_path"
fi
```

If no HTML file is found, bail with: "No design HTML files found at `<path>`. Provide a path to a Claude Design HTML export."

**0b. Dev server is running.** Resolve the app base URL from the caller's input (default `http://localhost:3000` if not provided), then check the app is accessible:

```bash
app_base_url="<caller-provided-app-base-url>"          # from Inputs; e.g. http://localhost:3000
app_base_url="${app_base_url:-http://localhost:3000}"  # default only when the caller omitted it
curl -s -o /dev/null -w "%{http_code}" "$app_base_url"
```

Accept any non-`000` response (a redirect or an auth gate still means the server is up); only `000`/connection-refused is a failure. If it fails, bail with: "The frontend dev server is not reachable at `$app_base_url`. Start it (the project's dev command) and re-run this validation."

Do NOT attempt to start the dev server — the caller owns its lifecycle.

**0c. playwright-cli is available.** Verify the CLI is installed:

```bash
playwright-cli --version
```

If not found, bail with: "`playwright-cli` is not installed. Install it with `npm install -g @playwright/cli@latest` and re-run."

### Phase 1 — Serve the design HTML

The design HTML files load React/Babel from CDN and reference relative font/asset paths (`fonts/`, `assets/`, `uploads/`). They must be served over HTTP — `file://` fails due to CORS restrictions.

Start a local HTTP server from the design file's **parent directory** so relative paths resolve correctly:

```bash
design_dir=$(dirname "$design_file")
design_filename=$(basename "$design_file")

# Clean up any prior run's temp directory
rm -rf /tmp/validate-design
mkdir -p /tmp/validate-design

# Start HTTP server on an OS-assigned port
python3 -m http.server 0 --bind 127.0.0.1 --directory "$design_dir" > /tmp/validate-design/server.log 2>&1 &
DESIGN_SERVER_PID=$!

# Wait for the server to start and extract the port
sleep 1
DESIGN_SERVER_PORT=$(grep -oE 'port [0-9]+' /tmp/validate-design/server.log | grep -oE '[0-9]+')
DESIGN_BASE_URL="http://127.0.0.1:${DESIGN_SERVER_PORT}"
DESIGN_HTML_URL="${DESIGN_BASE_URL}/${design_filename}"
```

Verify the design page loads:

```bash
curl -s -o /dev/null -w "%{http_code}" "$DESIGN_HTML_URL"
# Must return 200
```

If the server fails to start or the page doesn't load, bail and kill the background process.

### Phase 2 — Capture subagent

Spawn a **general-purpose subagent** to capture both the design prototype and the live app using playwright-cli. This subagent is purely mechanical — it navigates, waits, screenshots, and extracts data. It does NOT analyze or compare.

The subagent uses a named browser session `validate-design` for all commands. It interacts with the browser exclusively through `Bash` tool calls running `playwright-cli -s=validate-design ...` commands.

**Subagent prompt:**

```
You are a browser capture agent. Your job is to navigate to two web pages using playwright-cli, capture screenshots and DOM snapshots, and extract computed CSS values. You do NOT analyze or compare — just capture and return structured data.

You interact with the browser exclusively through Bash commands using playwright-cli with session name "validate-design". Every browser command follows the pattern: playwright-cli -s=validate-design <command> [args]

## Setup

Open a browser session and set the viewport:

playwright-cli -s=validate-design open
playwright-cli -s=validate-design resize ${viewport_width} ${viewport_height}

## Target 1: Design prototype

URL: ${DESIGN_HTML_URL}

This is a React prototype that compiles JSX at runtime via Babel. It takes 2-5 seconds to render after page load.

Steps:
1. Navigate to the URL:
   playwright-cli -s=validate-design goto "${DESIGN_HTML_URL}"

2. Wait for React/Babel to finish rendering — the page is blank until #root has child content. This is critical:
   playwright-cli -s=validate-design run-code "async page => await page.waitForSelector('#root > *', { timeout: 15000 })"

3. ${design_navigation_instructions — e.g., "Take a snapshot to find the sidebar navigation, then click the 'Feed' button:
   playwright-cli -s=validate-design snapshot
   Then use the ref from the snapshot output:
   playwright-cli -s=validate-design click <ref>
   "}

4. Take a screenshot — this is the DESIGN screenshot:
   playwright-cli -s=validate-design screenshot --filename=design.png
   Describe what you see in the screenshot.

5. Capture a DOM snapshot WITH bounding boxes — this is the DESIGN DOM snapshot:
   playwright-cli -s=validate-design snapshot --boxes --filename=design-snapshot.yaml
   The --boxes flag adds [box=x,y,width,height] annotations to each element — the comparison subagent uses these for quantitative layout and spacing comparison. Save the full output.

6. Check for rendering failures:
   playwright-cli -s=validate-design console error
   playwright-cli -s=validate-design requests
   If you see console errors mentioning Babel, React, or module loading failures, report as DESIGN_HEALTH: DEGRADED with the error text. If you see font or asset requests returning 404, report as DESIGN_ASSET_FAILURES: [list of failed URLs]. If everything looks clean, report DESIGN_HEALTH: OK and DESIGN_ASSET_FAILURES: None. Continue capturing regardless — partial data is better than no data.

7. Extract design tokens and element styles using eval with --raw for clean JSON output:
   playwright-cli --raw -s=validate-design eval "(() => { const cs = getComputedStyle(document.documentElement); const tokens = {}; const allProps = Array.from(cs).filter(p => p.startsWith('--')); allProps.forEach(p => tokens[p] = cs.getPropertyValue(p).trim()); const headings = Array.from(document.querySelectorAll('h1,h2,h3,h4,h5,h6')).slice(0, 20).map(el => { const s = getComputedStyle(el); return { tag: el.tagName, text: el.textContent.trim().substring(0, 80), fontSize: s.fontSize, fontWeight: s.fontWeight, color: s.color, fontFamily: s.fontFamily.substring(0, 50) }; }); const buttons = Array.from(document.querySelectorAll('button')).slice(0, 20).map(el => { const s = getComputedStyle(el); return { text: el.textContent.trim().substring(0, 40), bg: s.backgroundColor, color: s.color, borderRadius: s.borderRadius, padding: s.padding, fontSize: s.fontSize }; }); return JSON.stringify({ tokens, headings, buttons }); })()"
   The --raw flag strips page status and snapshot noise — the output is ONLY the JSON string. Save this as the DESIGN extracted data.

## Target 2: Live implementation

URL: ${app_base_url}${app_route}

Steps:
1. Navigate to the URL:
   playwright-cli -s=validate-design goto "${app_base_url}${app_route}"

2. Wait for the main content to be visible (the selector list is framework-agnostic — `#__next` is the Next.js example; `#root`, `#app`, etc. are covered by the `body > *` fallback):
   playwright-cli -s=validate-design run-code "async page => { for (const sel of ['main', '#__next > *', '#root > *', '#app > *', 'body > *']) { try { await page.waitForSelector(sel, { timeout: 5000 }); return; } catch {} } }"

3. ${app_navigation_instructions — if any, e.g., "Take a snapshot and click the first project row:
   playwright-cli -s=validate-design snapshot
   playwright-cli -s=validate-design click <ref>
   "}

4. Take a screenshot — this is the APP screenshot:
   playwright-cli -s=validate-design screenshot --filename=app.png
   Describe what you see in the screenshot.

5. Capture a DOM snapshot WITH bounding boxes — this is the APP DOM snapshot:
   playwright-cli -s=validate-design snapshot --boxes --filename=app-snapshot.yaml
   Save the full output.

6. Check for rendering failures:
   playwright-cli -s=validate-design console error
   playwright-cli -s=validate-design requests
   If you see console errors (React errors, uncaught exceptions), report as APP_HEALTH: DEGRADED with the error text. If you see font or asset requests returning 404, report as APP_ASSET_FAILURES: [list of failed URLs]. If everything looks clean, report APP_HEALTH: OK and APP_ASSET_FAILURES: None. Continue capturing regardless.

7. Extract element styles using the same eval command as Target 1 (with --raw). Save as the APP extracted data.

## Cleanup

Close the browser session when all captures are complete:
playwright-cli -s=validate-design close

## Return format

Return ALL captured data in this exact structure:

DESIGN_SCREENSHOT: <describe what you see in the design screenshot>
DESIGN_HEALTH: <OK | DEGRADED — if DEGRADED, include console errors>
DESIGN_ASSET_FAILURES: <list of failed asset URLs, or "None">
DESIGN_DOM_SNAPSHOT:
<paste the full snapshot output for the design — includes [box=x,y,width,height] for each element>
DESIGN_EXTRACTED_DATA:
<paste the --raw JSON result from eval for the design>

APP_SCREENSHOT: <describe what you see in the app screenshot>
APP_HEALTH: <OK | DEGRADED — if DEGRADED, include console errors>
APP_ASSET_FAILURES: <list of failed asset URLs, or "None">
APP_DOM_SNAPSHOT:
<paste the full snapshot output for the app — includes [box=x,y,width,height] for each element>
APP_EXTRACTED_DATA:
<paste the --raw JSON result from eval for the app>

Important:
- Do NOT skip any capture step
- Do NOT analyze or compare the results
- Do NOT suggest changes
- If a navigation step fails, report the failure and continue with what you can capture
- If the waitForSelector times out on the design page, try again with a longer timeout (30s) — Babel compilation can be slow on large prototypes
- If the session fails to open or crashes mid-capture, report the error immediately — do not retry
```

### Phase 3 — Comparison subagent

After the capture subagent returns, spawn a **separate general-purpose subagent** with ONLY the captured data. This subagent gets a fresh context — it has never seen the design files, the app code, or the pipeline state. It is purely analytical.

**Subagent prompt:**

```
You are a design fidelity comparison agent. You have captures from a design prototype and a live implementation of the same view. Compare them and produce a structured discrepancy report.

You must NOT suggest fixes or code changes. You are a reporter, not an implementer.

## Captured data

${paste the full output from the capture subagent}

## Rendering health context

Before comparing, check the DESIGN_HEALTH, APP_HEALTH, DESIGN_ASSET_FAILURES, and APP_ASSET_FAILURES fields.

- If either page is DEGRADED, note this at the top of your SUMMARY. Console errors may explain visual discrepancies (e.g., a React error boundary replacing content, or a failed Babel compilation leaving a blank page).
- If asset failures include font files (.woff, .woff2, .ttf), **deprioritize typography discrepancies to "minor"** — they are caused by the missing font, not by implementation error. Note the root cause in the NOTES field.
- If asset failures include images or icons, note this when reporting missing visual elements.

## Comparison dimensions

Compare across each dimension below. A discrepancy is any visible difference between the design and the implementation.

### 1. Layout & Structure
- Overall page structure (sidebar + main content, header placement, content sections)
- Section ordering and grouping
- Grid/flex layout patterns
- Presence or absence of major UI regions
- **Bounding box comparison**: The DOM snapshots include [box=x,y,width,height] for each element. Use these to compare:
  - Major container widths (sidebar, main content area, header)
  - Vertical ordering — elements should appear at similar y-positions
  - Element dimensions — width/height should be proportionally similar
  - Flag elements that differ by >20px in position or >10% in dimensions as layout issues

### 2. Colors & Theming
- Background colors of major surfaces (page, cards, sidebar, header)
- Text colors (primary, secondary, tertiary)
- Brand accent color usage
- State colors (success, warning, error)
- Compare the CSS token values from the extracted data

### 3. Typography
- Font family — compare what the design uses against what the app renders (read both from the extracted `fontFamily` data; don't assume a specific family)
- Font sizes for headings, body text, labels, captions
- Font weights
- Compare the heading data from the extracted data

### 4. Spacing
- Padding within cards, sections, containers
- Gap between elements
- Margins around major sections
- **Bounding box gap analysis**: Use the [box=x,y,width,height] data to compute gaps between adjacent elements:
  - Vertical gap = next_element.y - (current_element.y + current_element.height)
  - Horizontal gap = next_element.x - (current_element.x + current_element.width)
  - Compare computed gaps between design and app — differences >8px are significant

### 5. Component Fidelity
- Buttons: shape, color, padding, icon placement, border radius
- Cards: border, shadow, radius, internal padding
- Badges/tags: shape, color scheme
- Inputs: border, radius, placeholder text
- Compare the button data from the extracted data
- **Component sizing from bounding boxes**: Use [box] data to compare actual rendered sizes of buttons, cards, and inputs between design and app. A button that is 40px tall in the design but 32px in the app is a fidelity issue even if CSS properties look similar.

### 6. Visual Hierarchy
- What draws the eye first in each version
- Heading size progression
- Use of color and weight to direct attention
- Whitespace distribution

### 7. Interactive Elements
- Are all clickable elements from the design present in the app?
- Navigation items, action buttons, form controls

### 8. Content & Copy
- Labels, headings, placeholder text
- Information architecture (what info appears where)
- NOTE: Ignore differences in data content (names, numbers, dates) — the design uses hardcoded mock data while the app uses live data or empty states

${focus_areas — if the caller specified focus areas, add: "FOCUS: The caller has asked you to focus specifically on: <areas>. Prioritize these dimensions but still report critical issues in other dimensions."}

## Report format

Return your report in EXACTLY this format:

STATUS: <match | partial | mismatch>

SUMMARY: <2-3 sentence overall assessment of design fidelity>

DISCREPANCIES:
For each issue found, one block:
- DIMENSION: <layout | colors | typography | spacing | components | hierarchy | interactive | content>
- SEVERITY: <critical | major | minor | cosmetic>
- ELEMENT: <what specific element or area is affected>
- DESIGN: <what the design shows — cite specific values where possible>
- APP: <what the app shows — cite specific values where possible>
- NOTES: <any additional context>

Severity guide:
- critical: fundamentally different layout, missing major sections, broken page structure
- major: wrong colors, wrong font sizes, missing components, wrong visual hierarchy
- minor: small spacing differences, slightly wrong border radius, minor color shade differences
- cosmetic: negligible differences that would be hard to notice

TOKEN_DRIFT:
For each CSS custom property that differs significantly:
- TOKEN: <name> | DESIGN: <value> | APP: <value or "not defined">

MISSING_ELEMENTS:
Bulleted list of elements present in the design but absent in the app.

EXTRA_ELEMENTS:
Bulleted list of elements present in the app but not in the design.

If STATUS is "match", DISCREPANCIES can be empty. Always include TOKEN_DRIFT, MISSING_ELEMENTS, and EXTRA_ELEMENTS even if empty (write "None" for each).
```

### Phase 4 — Report and cleanup

1. **Format the report.** Take the comparison subagent's output and present it as the final validation report. Preserve the structured format exactly — downstream pipeline steps may parse it.

2. **Kill the design HTTP server:**
   ```bash
   kill $DESIGN_SERVER_PID 2>/dev/null || true
   ```

3. **Close the browser session** (defensive — in case the capture subagent failed to close it):
   ```bash
   playwright-cli -s=validate-design close 2>/dev/null || true
   ```

4. **Clean up playwright-cli artifacts:**
   ```bash
   rm -f .playwright-cli/design.png .playwright-cli/app.png .playwright-cli/design-snapshot.yaml .playwright-cli/app-snapshot.yaml
   ```

5. **Return the report.** This is the skill's output. Do NOT:
   - Suggest code changes or fixes
   - Attempt to implement any corrections
   - Re-run the comparison
   - Modify any files

The pipeline caller decides what to do with the discrepancy report.

## Gotchas

- **React/Babel render delay.** The design HTML compiles JSX at runtime. The capture subagent MUST use `run-code` with `waitForSelector('#root > *')` before screenshotting. Without this, you get a blank page screenshot.
- **Design navigation is client-side.** The prototype uses `useState` for page switching — navigate by clicking sidebar elements, not by changing the URL. The caller must provide explicit navigation instructions.
- **Named sessions enable isolation.** The skill uses a `validate-design` named session. Both captures (design + app) still happen sequentially in one subagent for simplicity, but if parallel captures are ever needed, use distinct session names (e.g., `validate-design-target1`, `validate-design-target2`).
- **HTTP server directory matters.** The server MUST be started from the design file's parent directory. If you serve from a parent or sibling directory, relative paths to `fonts/`, `assets/`, `uploads/` will 404.
- **Prefer standalone HTML variants.** Files matching `*standalone*` bundle React/Babel inline and have no CDN dependency. They're larger but more reliable.
- **The comparison subagent is read-only.** It reports findings. If it accidentally suggests fixes, strip that from the output before returning to the pipeline.
- **Ignore data content.** Design prototypes use hardcoded mock data. Differences in names, numbers, or dates between design and app are NOT design discrepancies.
- **Clean `/tmp/validate-design/` at the start** of each run to prevent stale data from a prior invocation contaminating the current one.
- **Session cleanup on failure.** If the capture subagent crashes or times out, the named session may be left open. Phase 4 defensively runs `playwright-cli -s=validate-design close` to handle this. If stale sessions accumulate, `playwright-cli list` shows all active sessions and `playwright-cli close-all` cleans them up.
- **eval requires JSON.stringify for complex return values.** Unlike Playwright MCP's `browser_evaluate` which auto-serialized objects, `playwright-cli eval` returns the raw expression result. Wrap complex return values in `JSON.stringify()` to ensure structured data is captured correctly.
- **Use `--raw` when you need just the eval result.** Without `--raw`, `eval` output includes page status, generated code, and a snapshot section alongside the actual return value. The capture subagent must use `--raw` so the extracted JSON is the only output — this prevents the subagent from accidentally pasting snapshot text into the EXTRACTED_DATA fields.
- **`--boxes` increases snapshot size.** Each element gets a `[box=x,y,width,height]` annotation, which roughly doubles the snapshot text. On very large pages, this could approach context limits for the comparison subagent. If you encounter truncation, use `snapshot --boxes --depth=5` to limit the tree depth while still getting bounding boxes for the top-level layout structure.
- **Filter console output to errors only.** Use `playwright-cli -s=validate-design console error`, not bare `console`. Design prototypes routinely emit Babel deprecation warnings and React development-mode noise that would overwhelm the capture subagent's context.