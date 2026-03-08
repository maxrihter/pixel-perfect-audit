---
name: pixel-perfect
description: >
  Manual pixel-perfect design audit — compare live web app CSS against a design system / brandbook.
  TRIGGER when: user mentions "pixel-perfect", "design audit", "UI audit", "brandbook compliance",
  "design QA", "check against design", "design implementation check",
  or wants to verify a live site matches the design system.
  DO NOT TRIGGER when: user wants automated screenshot comparison / visual regression testing,
  functional/behavioral testing, just wants a single screenshot,
  CSS performance/bundle audit, CSS linting, or CSS optimization (those are code-level tasks, not visual design QA).
license: MIT
metadata:
  category: technique
  author: maxrihter
  version: "1.3.0"
  triggers: pixel-perfect, design audit, UI audit, brandbook compliance, design QA, design check, design implementation, check against design, check against brandbook
---

# Pixel-Perfect Design Audit

Systematic manual audit of a live web application against a design system (brandbook). Measure every CSS property via browser inspection, flag deviations, output a structured Excel report with screenshots and reproduction paths.

**When to use:** A new build, redesign, or feature is ready for design QA. You need to verify that the implementation matches the brandbook — colors, typography, spacing, component consistency. The output is a detailed bug report for the designer/developer.

**Core strength:** Catches what automated tools cannot — wrong font-weight (600 vs 700), colors close but not matching the palette (#8B9DAD vs #8996A3), dropdown overflow bugs, typography outside the type scale, cross-component inconsistencies. Requires a brandbook as source of truth.

---

## Tool Quick Reference

Every action in this audit maps to a specific Chrome MCP tool. **This skill requires Chrome MCP browser extension.**

> **`javascript_tool` critical note:** Every call requires three parameters: `action: "javascript_exec"`, `text` (the JS code string), and `tabId`. Omitting `action` will cause the call to fail.

| Action | Tool | Notes |
|--------|------|-------|
| Get browser tab ID | `tabs_context_mcp` | **MUST call first** before any browser tool |
| Create new tab | `tabs_create_mcp` | For opening reference site alongside target |
| Navigate to URL | `navigate` | Requires `url` + `tabId`. Supports `"back"`/`"forward"` |
| Discover page structure | `read_page` | Accessibility tree with `ref` IDs. Params: `tabId` (required), `filter` (`"interactive"`), `depth`, `ref_id` (subtree), `max_chars` (default 50000) |
| Find elements by purpose | `find` | Natural language query + `tabId`. Returns max 20 matches |
| Measure CSS properties | `javascript_tool` | See critical note above |
| Extract all page text | `get_page_text` | Quick text inventory for typography audit |
| Take viewport screenshot | `computer` | `action: "screenshot"` + `tabId` (no coordinate needed) |
| Click elements | `computer` | `action: "left_click"` + `coordinate: [x,y]` or `ref` + `tabId` |
| Hover elements | `computer` | `action: "hover"` + `coordinate: [x,y]` or `ref` + `tabId` |
| Scroll to element | `computer` | `action: "scroll_to"` + `ref` from read_page/find + `tabId` |
| Scroll viewport | `computer` | `action: "scroll"` + `coordinate` + `scroll_direction` + `tabId` |
| Wait for transitions | `computer` | `action: "wait"` + `duration` (seconds, max 30) + `tabId` |
| Resize viewport | `resize_window` | `width` + `height` + `tabId` (all required) |
| Read Figma brandbook | `get_design_context` / `get_screenshot` (Figma MCP) | Optional — extract tokens from Figma |
| Generate Excel report | openpyxl via Python script | See Phase 7 for complete template |

---

## Phase 0: Request Prerequisites

**Before any work, ask the user for:**

1. **Design system / Brandbook** — Figma link, PDF, Notion page, or explicit list of tokens:
   - Color palette (hex + names), typography scale (sizes, weights, line-heights, letter-spacing, font-family)
   - Button variants and states, spacing system, border-radii, shadows, component specs

2. **Target URL** — the site or build to audit
   - If localhost/dev server: warn that HMR may invalidate measurements — use a stable build or disable HMR

3. **Reference implementation** (optional) — production site or approved build for cross-validation

4. **Scope** — full audit or specific pages/components? For 7+ pages, suggest splitting into sessions

5. **Device targets** — desktop only? Mobile? Responsive breakpoints? Default: desktop 1440x900

6. **Known exceptions** — any intentional deviations from brandbook?

7. **Authentication** — if login required: ask user to log in manually. Never enter credentials on behalf of the user

> **Critical:** Do NOT start without a brandbook. If the user cannot provide a formal one, ask for at minimum: brand colors (hex), font family, type scale (sizes + weights). If they cannot provide even this, **decline the audit** — otherwise every finding would be arbitrary opinion, not a verifiable bug.

---

## Phase 1: Browser Setup

### Step 1: Get tab context
Call `tabs_context_mcp` to get available tabs. If no MCP tab group exists, set `createIfEmpty: true`.

### Step 2: Open target site
Use `navigate` with the target URL. If a reference site is provided, use `tabs_create_mcp` for a second tab.

### Step 3: Handle cookie consent / overlays
Dismiss cookie banners before auditing — use `find(query="cookie banner")` or `find(query="accept cookies button")`. Click the reject/dismiss button (prefer privacy-preserving option).

### Step 4: Verify page loaded
Call `read_page` to confirm rendering. If output exceeds limit, reduce `depth` or use `ref_id` to drill into sections.

### Step 5: Handle authentication (if needed)
- Inform user, ask them to log in manually in the browser
- **HTTP Basic Auth** (browser-native popups): cannot be interacted with via MCP — ask user to enter directly

### SPA / Client-side routing
- Use `navigate` for full URL changes; `computer` (left_click) for internal links (preserves app state)
- After navigation: `read_page` to verify; `computer` (wait, duration: 2) for lazy content

---

## Phase 2: Design Token Extraction

Extract the full token set from the brandbook. Build a quick-reference lookup.

### From Figma
Use Figma MCP. Extract from URL `https://figma.com/design/{fileKey}/{fileName}?node-id={nodeId}` — replace dash with colon in nodeId; no node-id -> use "0:1"; branch URLs -> use branchKey as fileKey.
1. `get_design_context(fileKey, nodeId)` — reference code with tokens
2. `get_screenshot(fileKey, nodeId)` — visual reference
3. Extract hex colors, font sizes, weights, spacing

### From PDF
Use `Read` tool with `pages` parameter for large PDFs. URL PDFs: open in Chrome via `navigate` + `get_page_text`. Image-heavy: ask user for key values directly.

### From Notion
`notion-fetch` with URL/ID -> Markdown with tokens. For databases: `notion-search` -> `notion-fetch` each page.

### Manual specs
Build reference tables from whatever the user provides:

```
Colors:
Name            | Hex
Primary         | #XXXXXX
Text Primary    | #XXXXXX
Text Secondary  | #XXXXXX

Typography Scale:
Style  | Size | Weight | Line-Height | Letter-Spacing
H1     | XXpx | 700    | ...         | ...
Body   | XXpx | 400    | ...         | ...
```

Note buttons, inputs, border-radius, padding per variant. If CSS custom properties are defined (e.g., `--color-primary`), note them for checking: `getComputedStyle(document.documentElement).getPropertyValue('--color-primary')`.

> **Units:** `getComputedStyle()` returns **px** even if source uses rem/em/%. If brandbook uses rem: multiply by root font-size (default 16px).

**Normalize all hex values to UPPERCASE** (e.g., `#a684ff` → `#A684FF`). `getComputedStyle()` returns lowercase rgb — the `rgbToHex` helper outputs uppercase. If the brandbook mixes cases, normalize during extraction to avoid false mismatches.

**Keep these tables accessible throughout the entire audit.**

---

## Phase 3: Site Discovery & Navigation Map

Navigate the entire site and build an exhaustive map.

### Step 1: Map all pages
- Visit every route using `navigate` and internal link clicks
- `read_page` on each page (reduce `depth` if truncated); `filter="interactive"` for buttons/links/inputs
- Identify tabs, sub-views, nested navigation

### Step 2: Identify ALL interactive states

| Element Type | What to Open/Trigger |
|---|---|
| **Dropdowns** | Open each. Scroll to see ALL items (verify count via JS) |
| **Modals** | Open each. Test ALL internal dropdowns and forms |
| **Expandable sections** | Click "show more", accordions, collapsible cards |
| **Tooltips** | Hover over every info icon, truncated text |
| **Form states** | Empty, filled, error, success, disabled |
| **Hover/Focus/Active** | Buttons, links, cards, inputs — check visual changes |
| **Loading / Empty** | Skeleton and empty data views if accessible |
| **Themes / Dark mode** | Detect: `window.matchMedia('(prefers-color-scheme: dark)').matches` or UI toggle. Each theme = separate audit pass |

### Step 3: Create audit checklist
```
- Page A: default, card expanded, settings modal, tooltip
- Page B: default, period dropdown, export modal
```

**Lazy-loaded content:** SPA apps may render content on scroll (infinite scroll, virtual lists, lazy sections). Before auditing, scroll the entire page to trigger all lazy loads: `computer` (scroll, direction: down) repeatedly until `document.documentElement.scrollHeight` stabilizes. For virtual lists, the visible items represent all items — measure visible ones only.

For 5+ page sites, maintain a progress log. If context runs low, summarize completed pages before continuing.

**Session handoff protocol:** When context runs low or splitting across sessions, save a handoff file to `/tmp/pixel-perfect-handoff.md` containing:
1. **Design tokens** — full color palette + typography scale tables from Phase 2
2. **Component registry** — from Phase 4 with expected styles
3. **Completed pages** — list with bug count per page
4. **Bug list so far** — all bugs in report format (7 columns)
5. **Systemic issues** — bugs that apply globally
6. **Remaining pages** — unchecked items from Phase 3 checklist

Start the new session with: *"Continue pixel-perfect audit. Handoff file: /tmp/pixel-perfect-handoff.md"*

---

## Phase 4: Component Grouping

Identify repeating UI patterns. Build a component registry:

```
Component  | Variant   | Expected Styles (from brandbook)
Button     | Primary   | bg: #XXXX, radius: 12px, weight: 700
Button     | Secondary | bg: transparent, border: 1px #XXXX
Card       | Default   | bg: ?, radius: ?, padding: ?
Text       | Heading   | size: 36px, weight: 700, color: #XXXX
Text       | Body      | size: 16px, weight: 400, color: #XXXX
Input      | Default   | bg: ?, border: ?, radius: ?
Dropdown   | Trigger   | weight may differ from options (standard UX)
Modal      | Container | bg: ?, radius: ?, padding: ?
```

For `?` values, measure one canonical instance during Phase 5 — use as expected value for cross-page consistency.

**Purpose:** Components within the same group MUST have identical styling. Any inconsistency = bug.

---

## Phase 5: Systematic Audit

Process one page at a time.

### 5.0 DOM Existence Verification (MANDATORY)

**Before logging ANY bug, the element MUST be confirmed to exist in the DOM.**

Every measurement MUST follow this sequence:
1. **Find** the element via `find(query=...)` or `read_page` — get a `ref` or selector
2. **Verify existence** via `javascript_tool`: `document.querySelector(SELECTOR) !== null`
3. **Capture textContent** alongside CSS: `el.textContent.trim().slice(0, 60)` — proves you measured the RIGHT element
4. **Double-measure** if the value seems unexpected: use a DIFFERENT selector or `ref` click + re-measure

```javascript
// MANDATORY: existence check + textContent capture
const rgbToHex = (v) => { if (!v || !v.startsWith('rgb')) return v; const m = v.match(/[\d.]+/g); return '#' + m.slice(0,3).map(x => (+x).toString(16).padStart(2,'0')).join('').toUpperCase(); };
const el = document.querySelector(SELECTOR);
!el ? JSON.stringify({ error: 'ELEMENT DOES NOT EXIST', selector: SELECTOR })
: (() => { const s = getComputedStyle(el); return JSON.stringify({
    exists: true,
    textContent: el.textContent.trim().slice(0, 60),
    visible: el.offsetParent !== null && el.offsetWidth > 0,
    fontSize: s.fontSize, fontWeight: s.fontWeight,
    color: rgbToHex(s.color), bg: rgbToHex(s.backgroundColor)
  }); })()
```

**If `exists: false` or `visible: false` → the element is NOT on the page. Do NOT log a bug.**

> **Why this exists:** In a production audit, "Export PDF" and "Submit report" buttons were flagged as having wrong font-size — but they didn't exist on the page at all. This was a pure hallucination from training data, not a measurement.

### 5.1 Element discovery
Use `read_page` for the accessibility tree. Use `find` to locate elements by purpose (max 20 per call). Use `computer` (scroll_to) to reach elements below the fold.

**Batch measurements:** Combine 5-10 element measurements into a single `javascript_tool` call using the `measure()` helper array pattern (see CSS Measurement Techniques). This reduces tool calls from ~200 to ~30 for a typical page. Example: `JSON.stringify([measure('#h1','H1'), measure('.btn','CTA'), measure('.card','Card')])`

### 5.2 Take viewport screenshot
`computer` (action: screenshot) captures **visible viewport only**. Scroll and take multiple screenshots for long pages. Screenshots live in conversation context — cannot be saved to disk or embedded in Excel.

### 5.3 Typography audit (most common issue source)
For every text element, measure via `javascript_tool`:
- `fontSize` — must match type scale
- `fontWeight` — watch for 600 vs 700!
- **Context disambiguation:** When multiple elements share the same visible text (e.g., "Price" appears in a table header AND a chart legend), ALWAYS capture `textContent` + parent context to confirm you measured the correct one. Use: `el.closest('section, [class*="table"], [class*="chart"]')?.className` to verify context.
- `fontFamily` — strip quotes and fallbacks for comparison
- `lineHeight`, `letterSpacing`, `color`

**Font loading:** Verify before measuring: `document.fonts.check('1px "Inter"')` (any pixel size — checks availability, not size). Wait via `document.fonts.ready` if needed.

### 5.4 Color audit
- Convert rgb/rgba to hex (see Techniques section)
- Compare against palette; flag any hex not in brandbook
- Check alpha/opacity separately
- **Gradients:** check `backgroundImage` for `linear-gradient()` — won't appear in `backgroundColor`
- Check CSS custom properties vs hardcoded values

### 5.5 Spacing & layout
- **Padding/Margin:** use individual properties (`paddingTop`, etc.) — shorthand may return empty when sides differ
- **Gap:** `gap`, `rowGap`, `columnGap` for flex/grid
- **Alignment:** `textAlign`, `justifyContent`, `alignItems`
- **Width constraints:** `maxWidth`, `minWidth`

### 5.6 Interactive state audit
- **Hover:** `computer` (hover, coordinate), then immediately screenshot/measure
- **Focus:** `computer` (left_click) on inputs — check outline, border changes
- **Wait for transitions:** `computer` (wait, duration: 1) before measuring if CSS transitions exist
- **Dropdowns in modals:** check if items overflow beyond modal boundary (see Techniques)

### 5.7 Responsive audit (if specified in Phase 0)
- `resize_window(width=BREAKPOINT, height=HEIGHT, tabId=TAB_ID)` per breakpoint
- Re-audit key elements; check text overflow, element overlap
- Note: minimum window width ~500px on macOS; verify with `window.innerWidth`

### 5.8 Consistency cross-check
Compare against Component Registry from Phase 4: same component on different pages = same styles? Same button type = same padding, font, border-radius?

---

## Phase 6: Verification & Cleanup

### Remove false positives
| Type | Rule |
|------|------|
| **Imperceptible color** | Per-channel decimal difference <= 3 (each R,G,B in 0-255) -> dismiss |
| **Standard UX pattern** | Dropdown trigger bold, options regular -> by design |
| **Matches production** | Production has same value -> likely intentional |
| **Color in palette** | Before flagging ANY color as "not in palette", check it against EVERY color in the Phase 2 reference table. A color may have multiple names (e.g. #A8F4FF = Cyan). If the hex exists anywhere in the palette — it is NOT a bug |

**Color threshold detail:** Compare R, G, B independently as decimals (0-255). Example: `#8B9DAD` vs `#8996A3` — R: 139 vs 137 (2), G: 157 vs 150 (7), B: 173 vs 163 (10) — G and B exceed 3, this IS a bug. Also compare alpha separately.

### Design intent filter

Not every visual difference is a bug. Apply this filter BEFORE logging:

| Pattern | Decision | Rationale |
|---------|----------|-----------|
| **Different categories → different colors** | NOT a bug | e.g., "Admin" badge = blue, "Member" badge = green — intentional differentiation |
| **Different hierarchy levels → different sizes** | NOT a bug | e.g., section header 24px vs subsection 18px — intentional hierarchy |
| **Different states → different styles** | NOT a bug | e.g., active tab bold, inactive regular — standard UX |
| **Role vs account type distinction** | NOT a bug | e.g., "Admin" role has no badge but "Editor" account type does |
| **Element doesn't exist on page** | NOT a bug | If you can't find it in DOM — it's a hallucination, not a bug |

> **Rule:** If the design system does NOT explicitly define a single style for all instances of a pattern, assume the variation is intentional. Only flag it if the brandbook says "all badges must be the same color" or similar.

### Deduplicate strictly

Duplicate bugs waste developer time and undermine report credibility. Apply these rules:

| Rule | Example |
|------|---------|
| **Systemic covers specific** | If global bug says "28px used in Overview, Activity feed" — do NOT add a page-level bug for "Overview heading 28px" |
| **Self-declared duplicates** | A bug that says "repeats systemic bug #N" should not exist — remove it |
| **Same page + same element** | Two entries for the same element on the same page = always a duplicate |
| **Current = Expected** | If "Current Value" equals "Expected Value" — it is NOT a bug, remove |
| **Systemic + page-level overlap** | Bug #5 says "28px headings across all pages" AND bug #29 says "28px on Dashboard heading" → remove #29 |
| **Same element, different wording** | "font-size 12px below minimum" and "12px too small for body text" on same element → keep one |
| **Single category per bug** | Never create "Typography + Colors" — split into two bugs or pick the primary |

**Allowed categories (strict enum):**
- `Typography` — font-size, font-weight, line-height, letter-spacing, font-family
- `Colors` — background, text color, border color, off-palette hex values
- `Consistency` — same component differs across pages/states
- `UX / Bug` — functional issues (overflow, hidden content, broken interactions)

### Severity classification

Severity must be **consistent for the same class of violation**. Use this decision tree:

| Severity | Criteria | Decision Rule |
|----------|----------|---------------|
| **Critical** | Functional impact — content hidden, actions blocked | Dropdown overflow hiding items; CTA button unreadable (≤10px); layout collapse |
| **High** | Clear spec violation on primary elements, visible to users | Wrong font-weight on headings; brand color mismatch; font-size below minimum on interactive elements |
| **Medium** | Minor deviation on secondary elements | Off-palette color on inner cards; font-size below minimum on passive text; padding differs |
| **Low** | Cosmetic, only visible with measurement tools | Letter-spacing -0.5% vs -1%; border-radius 11px vs 12px; color diff 4-6 per channel |

**Consistency rule:** The same type of violation (e.g., "12px below 15px minimum") should get the same severity UNLESS context changes impact:
- 12px on CTA button → **Critical** (user can't interact)
- 12px on tooltip text → **High** (visible spec violation)
- 12px on footer fine-print → **Medium** (secondary element)

Renumber sequentially after removals (1..N, no gaps).

---

## Phase 6.5: Self-Review (Quality Gate)

**Before writing the report, audit the audit.** This phase catches 10-15% of issues that degrade report quality.

### Checklist (run every time)

1. **Duplicate scan** — For each page-specific bug, check: does a global systemic bug already cover this exact element? If yes → remove
2. **Palette cross-reference** — For each "color not in palette" bug, verify the hex against the FULL Phase 2 palette table. Check all color names, not just the obvious one
3. **Current ≠ Expected** — No bug should have Current Value = Expected Value
4. **Category validation** — Every bug uses exactly one allowed category (no mixed, no ad-hoc)
5. **Severity consistency** — Same violation class → same severity (unless impact context differs)
6. **Format standardization** — Apply the report format template (Phase 7) uniformly:
   - Typography current values: `{size}px / {weight}` (e.g., `13px / 600`)
   - Color current values: `{hex}` (e.g., `#323D47`)
   - Mixed: `{size}px / {weight} / {hex}` (e.g., `12px / 500 / #A8F4FF`)
7. **Numbering** — Sequential 1..N, no gaps after removals
8. **Page name consistency** — Same page should always use the same name format across all bugs
9. **DOM existence proof** — Every bug must have been measured from a confirmed-existing DOM element. If any bug references an element you cannot re-find via `find` or `read_page` → it's a hallucination, remove it
10. **Context verification** — For bugs on elements with common text (e.g., "Price", "Total", "Status"), verify the textContent + parent context matches the intended element, not a similarly-named one elsewhere on page
11. **Design intent check** — For any "inconsistency" bug between different categories/types/roles, ask: "Does the brandbook explicitly require these to be identical?" If no → remove
12. **Navigation reproducibility** — For each bug, mentally trace the navigation path. Can a developer follow `Page (URL) → Section → Element "text"` to find it? If ambiguous → rewrite the path

> **Why this phase exists:** In production audits (5 rounds, 85+ bugs), this phase consistently caught 9-12 duplicates, 1-2 factual errors, and 5+ formatting inconsistencies per session. Skipping it delivers a 10-15% defective report.

---

## Phase 7: Documentation

Output in the user's preferred language. Column headers may stay English for developer compatibility.

### Report field format standards

Apply these formats uniformly across ALL bugs. Inconsistency looks unprofessional.

| Field | Format | Examples |
|-------|--------|---------|
| **Current Value** (typography) | `{size}px / {weight}` | `13px / 600`, `28px / 700` |
| **Current Value** (color) | `{hex}` | `#323D47`, `#8B9DAD` |
| **Current Value** (mixed) | `{size}px / {weight} / {hex}` | `12px / 500 / #A8F4FF` |
| **Current Value** (spacing) | `{value}px` or `{top} {right} {bottom} {left}` | `16px 20px`, `12px` |
| **Expected Value** | value + brandbook term | `15px SubText`, `#A684FF (Purple)`, `18px / 500 (Btn)` |
| **Page / Section** | `Page (route) → Section → Element "text"` | `Products (/products) → Data table → Column header "Price"`, `Settings (/settings) → Edit project modal → Input "Project name"` |

**Never use:** CSS property notation in Current Value (e.g., `font-weight: 600`), verbose `fontWeight 600`, or inconsistent slash spacing.

**Navigation path rules:**
- MINIMUM 3 levels: Page (with URL/route) → Section/Area → Specific element with quoted text
- Include the URL path or route (e.g., `/products`, `/settings`) in the Page level
- Quote the element's visible text: `Button "Submit"`, `Header "Overview"`, `Tab "Analytics"`
- For modal/dropdown bugs: include the trigger: `Settings → "Edit project" modal → Input "Discount"`
- **Test:** A developer who has never seen the site should be able to find the element in <30 seconds using only your navigation path

### Excel report

Write Python to a temp file and execute. `openpyxl` is auto-installed if missing.

> **Output:** `/tmp/pixel-perfect-audit.xlsx`. Inform user of path and suggest moving to a permanent location after delivery. Alternative formats: **CSV** (`csv.writer`), **Markdown** (in conversation for <20 bugs).

```python
# Write to /tmp/generate_audit_report.py, then run: python3 /tmp/generate_audit_report.py
import subprocess, sys
from datetime import datetime
try:
    import openpyxl
except ImportError:
    subprocess.check_call([sys.executable, '-m', 'pip', 'install', '-q', 'openpyxl'])
    import openpyxl

from openpyxl.styles import PatternFill, Font, Alignment
from collections import Counter

wb = openpyxl.Workbook()
ws = wb.active
ws.title = 'Pixel-Perfect Audit'

headers = ['Page / Section', 'Element', 'Issue', 'Category',
           'Severity', 'Current Value', 'Expected Value']
ws.append(headers)

for cell in ws[1]:
    cell.font = Font(bold=True)
    cell.alignment = Alignment(horizontal='center')

severity_fills = {
    'Critical': PatternFill(start_color='FF4444', end_color='FF4444', fill_type='solid'),
    'High': PatternFill(start_color='FF8800', end_color='FF8800', fill_type='solid'),
    'Medium': PatternFill(start_color='FFCC00', end_color='FFCC00', fill_type='solid'),
    'Low': PatternFill(start_color='88CC44', end_color='88CC44', fill_type='solid'),
}

# Bug data — fill from audit results
bugs = [
    # ['Page (route) → Section → Element "text"', 'Element', 'Issue', 'Category', 'Severity', 'Current', 'Expected']
]

for bug in bugs:
    ws.append(bug)
    row = ws.max_row
    severity = bug[4]
    if severity in severity_fills:
        for cell in ws[row]:
            cell.fill = severity_fills[severity]

for col in ws.columns:
    max_len = max(len(str(cell.value or '')) for cell in col)
    ws.column_dimensions[col[0].column_letter].width = min(max_len + 2, 50)

# Summary sheet
ws2 = wb.create_sheet('Summary')
ws2.append(['Audit Date', datetime.now().strftime('%Y-%m-%d')])
ws2.append(['Target URL', 'https://...'])  # Replace with actual URL
ws2.append(['Reference URL', 'https://...'])  # Replace if applicable
ws2.append([])

ws2.append(['Severity', 'Count'])
sev_counts = Counter(bug[4] for bug in bugs)
for sev in ['Critical', 'High', 'Medium', 'Low']:
    ws2.append([sev, sev_counts.get(sev, 0)])
ws2.append(['Total', len(bugs)])
ws2.append([])

ws2.append(['Category', 'Count'])
cat_counts = Counter(bug[3] for bug in bugs)
for cat, count in cat_counts.most_common():
    ws2.append([cat, count])

output_path = '/tmp/pixel-perfect-audit.xlsx'
wb.save(output_path)
print(f'Report saved to {output_path}')
```

### Screenshots
Take `computer` (screenshot) for every visually evident bug. For hover states: hover first, then immediately screenshot. Screenshots live in conversation context only — reference them by description in the Issue column (e.g., "see screenshot: hero section misaligned CTA").

---

## CSS Measurement Techniques

### Measure elements (primary technique)

Copy-paste template with `rgbToHex` + `measure()`. **Use selectors from `read_page`/`find`** — CSS-in-JS hashed classes are unstable; prefer `[data-testid]`, `[role]`, semantic tags:

```javascript
const rgbToHex = (val) => {
  if (!val || val === 'transparent') return 'transparent';
  if (!val.startsWith('rgb')) return val;
  const m = val.match(/[\d.]+/g);
  if (!m || m.length < 3) return val;
  const hex = '#' + m.slice(0,3).map(x => Math.round(+x).toString(16).padStart(2,'0')).join('').toUpperCase();
  const a = m.length >= 4 ? parseFloat(m[3]) : 1;
  return a < 1 ? hex + ' (a:' + a + ')' : hex;
};
const measure = (sel, label) => {
  const el = document.querySelector(sel);
  if (!el) return { label, error: 'not found' };
  const s = getComputedStyle(el);
  const ff = s.fontFamily.split(',')[0];
  return {
    label, fontSize: s.fontSize, fontWeight: s.fontWeight,
    fontFamily: ff ? ff.trim().replace(/['"]/g, '') : s.fontFamily,
    lineHeight: s.lineHeight, letterSpacing: s.letterSpacing,
    color: rgbToHex(s.color), bg: rgbToHex(s.backgroundColor),
    padding: s.padding || [s.paddingTop, s.paddingRight, s.paddingBottom, s.paddingLeft].join(' '),
    margin: s.margin || [s.marginTop, s.marginRight, s.marginBottom, s.marginLeft].join(' '),
    gap: s.gap, borderRadius: s.borderRadius, border: s.border,
    boxShadow: s.boxShadow, opacity: s.opacity, backgroundImage: s.backgroundImage
  };
};
// Replace with ACTUAL selectors from read_page/find:
JSON.stringify([
  measure('[data-testid="page-title"]', 'Page title'),
  measure('h2.subtitle', 'Subtitle'),
  measure('button[type="submit"]', 'Primary button')
])
```

### Specialized patterns

**Form/input values:**
```javascript
const el = document.querySelector('input[name="fieldname"]');
el ? JSON.stringify({ value: el.value, placeholder: el.placeholder, disabled: el.disabled, type: el.type })
   : JSON.stringify({ error: 'Input not found' })
```

**Pseudo-elements (::before/::after):**
```javascript
const el = document.querySelector('SELECTOR');
el ? JSON.stringify({
    beforeContent: getComputedStyle(el, '::before').content,
    beforeColor: getComputedStyle(el, '::before').color,
    afterContent: getComputedStyle(el, '::after').content
  }) : JSON.stringify({ error: 'not found' })
```

**Dropdown items (verify count):**
```javascript
const c = document.querySelector('DROPDOWN_SELECTOR');
c ? JSON.stringify({
    totalItems: c.querySelectorAll('li, [role="option"], [role="menuitem"]').length,
    texts: [...c.querySelectorAll('li, [role="option"], [role="menuitem"]')].slice(0, 20).map(i => i.textContent.trim()),
    scrollable: (() => { const s = c.querySelector('ul, [role="listbox"]'); return s ? s.scrollHeight > s.clientHeight : false; })()
  }) : JSON.stringify({ error: 'not found' })
```

**Modal overflow check:**
```javascript
const dd = document.querySelector('.dropdown-container');
const modal = document.querySelector('[role="dialog"]');
(dd && modal) ? JSON.stringify({
    overflows: dd.getBoundingClientRect().bottom > modal.getBoundingClientRect().bottom,
    hiddenPx: Math.max(0, Math.round(dd.getBoundingClientRect().bottom - modal.getBoundingClientRect().bottom))
  }) : JSON.stringify({ error: 'Elements not found' })
```

### Small font scan (catch ALL violations)

Scan the entire page for visible text elements below the brandbook minimum font-size. Catches instances you might miss when auditing section by section:

```javascript
const MIN = 15; // Replace with brandbook minimum
const small = [...document.querySelectorAll('*')].filter(el =>
  el.children.length === 0 && el.offsetParent !== null &&
  el.textContent.trim().length > 0 &&
  parseFloat(getComputedStyle(el).fontSize) < MIN
).map(el => {
  const s = getComputedStyle(el);
  return { text: el.textContent.trim().slice(0, 40), fontSize: s.fontSize, fontWeight: s.fontWeight, tag: el.tagName };
});
JSON.stringify({ count: small.length, items: small.slice(0, 30) })
```

### Color inventory (catch ALL off-palette values)

Build a complete set of unique text colors and background colors on the page. Compare against palette in one pass:

```javascript
// rgbToHex — reuse from "Measure elements" section above
const rgbToHex = (val) => { if (!val || val === 'transparent' || !val.startsWith('rgb')) return val; const m = val.match(/[\d.]+/g); if (!m || m.length < 3) return val; return '#' + m.slice(0,3).map(x => Math.round(+x).toString(16).padStart(2,'0')).join('').toUpperCase(); };
const colors = { text: new Set(), bg: new Set() };
[...document.querySelectorAll('*')].filter(el => el.offsetParent !== null).forEach(el => {
  const s = getComputedStyle(el);
  const c = rgbToHex(s.color); if (c && c !== 'transparent') colors.text.add(c);
  const b = rgbToHex(s.backgroundColor); if (b && b !== 'transparent') colors.bg.add(b);
});
JSON.stringify({ textColors: [...colors.text], bgColors: [...colors.bg] })
```

### Element discovery
Use `find` before writing selectors:
```
find(query="period selector dropdown", tabId=TAB_ID)
find(query="submit button", tabId=TAB_ID)
```
Then `read_page` with `ref_id` to drill into subtrees, or `javascript_tool` with a derived selector.

---

## Known Gotchas

> Items covered in Phase 5/6 or Techniques are not repeated here. Only unique edge cases.

1. **line-height: normal** — browsers compute as ~1.2x font-size. If brandbook specifies explicit values, `normal` is a deviation (usually low severity).
2. **Text overflow** — `text-overflow: ellipsis` + `overflow: hidden` clips text. Detect: `el.scrollWidth > el.clientWidth`.
3. **SVG currentColor** — `fill="currentColor"` inherits parent's CSS `color`. Measure parent instead.
4. **SVG className** — returns `SVGAnimatedString`, not string. Use `el.getAttribute('class')` when iterating.
5. **Shadow DOM** — `querySelector` can't pierce shadow roots. Use `el.shadowRoot.querySelector('inner')` (only works for `mode: "open"`; `mode: "closed"` is inaccessible).
6. **Canvas/WebGL** — elements inside `<canvas>` can't be measured via CSS. Note: "canvas-rendered — not auditable."
7. **Iframes** — `javascript_tool` operates main frame only. Content inside iframes not measurable.
8. **Sticky/fixed elements** — persist across scroll, may overlap content. Audit separately, check z-index.
9. **text-transform: uppercase** — DOM text contains "Project" but displays "PROJECT". Searching for the displayed text returns 0 matches. Always search for the original DOM text, not the visual representation.
10. **Multiple identical elements** — SPA apps often render duplicate elements (e.g., mobile + desktop buttons). When measuring, verify the element is visible: `el.offsetParent !== null && el.offsetWidth > 0`. Finding the wrong one gives wrong measurements.
11. **`.closest()` parent capture** — When finding a badge's parent container, `.closest('div')` can return a much larger ancestor. Use precise selectors or verify dimensions match expectations.
12. **Excel merged cells** — When updating summary sheets with `openpyxl`, merged cells throw `'MergedCell' object attribute 'value' is read-only`. Write only to the top-left cell of each merged range, or unmerge before editing.
13. **Same text, different context** — "Price" may appear as a table column header, a chart axis label, a tooltip, and a card stat — all with different (correct) styles. When measuring, ALWAYS capture `el.textContent` + parent selector to confirm which "Price" you measured. If you report a bug on the wrong one, the developer can't reproduce it.
14. **Intentional category differentiation** — Different account types, roles, statuses, or tiers often have intentionally different visual treatments (colors, badges, icons). "Admin" may be blue while "Member" is green — this is differentiation, not inconsistency. Only flag if the brandbook explicitly mandates uniform styling.

---

## Critical Rules

### MUST DO
- Initialize browser via `tabs_context_mcp` BEFORE any inspection
- Pass `action: "javascript_exec"` in every `javascript_tool` call
- Cache `getComputedStyle(el)` — never call multiple times on same element
- Use `read_page` as PRIMARY element discovery tool on every page
- Measure EVERY value from live CSS — never assume
- Open EVERY dropdown, modal, tooltip, expandable on every page
- Verify dropdown item counts via JS — never trust visual count
- Cross-reference with production site when available
- Document navigation path to reproduce each bug
- Group systemic issues — don't report the same pattern 15 times
- Clean up false positives before delivering the report
- Ask the user for the brandbook BEFORE starting
- Run Phase 6.5 Self-Review before EVERY report delivery
- Cross-reference EVERY "off-palette" color against the FULL palette before flagging
- Verify Current Value ≠ Expected Value for every bug
- Use exactly one allowed category per bug (Typography / Colors / Consistency / UX Bug)
- Check `el.offsetParent !== null` when multiple DOM elements match a selector
- Verify DOM existence (`document.querySelector(sel) !== null`) before logging ANY bug
- Capture `textContent` with every measurement to prove the correct element was measured
- Double-measure any value that seems unexpected using an alternative selector
- Apply design intent filter: different categories/types having different styles is NOT a bug unless brandbook says otherwise
- Write navigation paths with 3+ levels including URL route and quoted element text
- For elements with common text ("Price", "Total", "Status"), verify parent context before reporting

### NEVER DO
- Never start without a design system reference
- Never flag color differences where ALL channels differ by <= 3 (decimal 0-255)
- Never flag standard UX patterns as bugs without brandbook evidence
- Never skip a page, dropdown, modal, or interactive state
- Never enter passwords or credentials on behalf of the user
- Never create a bug that says "repeats bug #N" — if it repeats, it should not exist
- Never use mixed categories ("Typography + Colors") — split or pick primary
- Never search for visually-displayed text when `text-transform` is applied — search original DOM text
- Never log a bug for an element you cannot find in the DOM — if `querySelector` returns null, it does not exist
- Never flag different visual treatments for different categories/types/roles as "inconsistency" without explicit brandbook evidence
- Never write a navigation path with fewer than 3 levels or without the element's visible text
- Never report a measurement without capturing the element's textContent alongside it
