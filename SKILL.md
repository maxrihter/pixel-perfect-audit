---
name: pixel-perfect
description: >
  Manual pixel-perfect design audit — compare live web app CSS against a design reference
  (Figma, PDF brandbook, screenshots, reference site, or manual specs).
  TRIGGER when: user mentions "pixel-perfect", "design audit", "UI audit", "brandbook compliance",
  "design QA", "check against design", "design implementation check",
  or wants to verify a live site matches the design system.
  DO NOT TRIGGER when: user wants automated screenshot comparison / visual regression testing,
  functional/behavioral testing, just wants a single screenshot,
  CSS performance/bundle audit, CSS linting, or CSS optimization.
license: MIT
metadata:
  category: technique
  author: maxrihter
  version: "2.1.0"
  triggers: pixel-perfect, design audit, UI audit, brandbook compliance, design QA, design check, design implementation, check against design, check against brandbook
---

# Pixel-Perfect Design Audit v2.1

Compare live web app against a design reference. Measure every CSS property, cross-verify against design tokens, output structured report.

**Core workflow:** Token extraction (from any source) → Live site measurement (JS) → Cross-verification → Report.

---

## Phase 0: Prerequisites

Ask the user for:

1. **Design reference** — determine the source type:

   | Source type | What to ask for | Priority |
   |-------------|-----------------|----------|
   | **Figma** | Link with specific page/frame node-id | Best — interactive, inspectable |
   | **Reference site** | Production URL (e.g. app.example.com) | Good — measure CSS directly |
   | **Screenshots** | PNG/JPG files with annotated specs or clean UI | Decent — visual extraction |
   | **PDF brandbook** | File path or URL | Decent — structured specs |
   | **Notion** | Page URL or database ID | Decent — structured specs |
   | **Manual specs** | Colors (hex), fonts (size+weight), spacing | Minimum viable |

2. **Target URL** — the build to audit. Stable build preferred (HMR invalidates measurements)
3. **Reference URL** (optional) — production for cross-validation
4. **Scope** — full audit or specific pages? For 7+ pages, split sessions
5. **Language** — report language
6. **Known exceptions** — intentional deviations?

> **HARD STOP:** No design reference = no audit. Minimum needed: at least one source from the table above.

---

## Phase 1: Browser Setup

```
1. tabs_context_mcp (createIfEmpty: true)
2. navigate → target URL (tab 1)
3. Design reference tab(s):
   - Figma → tabs_create_mcp → navigate to Figma URL (tab 2)
   - Reference site → tabs_create_mcp → navigate to production (tab 2)
   - Screenshots/PDF → use Read tool (no browser tab needed)
   - Notion → use notion-fetch (no browser tab needed)
4. Optional: additional tab for cross-validation
5. Dismiss cookie banners on all web tabs (find → click reject/dismiss)
6. read_page on target → verify loaded
```

---

## Phase 2: Token Extraction (CRITICAL — DO NOT RUSH)

**This phase prevents 80% of false positives.** Incomplete tokens = false bugs downstream.

Pick the workflow matching your source type. If multiple sources available — combine them (e.g. Figma for typography + reference site for component dimensions).

---

### 2A. From Figma (interactive canvas inspection)

Figma MCP's `get_design_context` returns code references, NOT exact typography values from the properties panel. The reliable workflow is **clicking elements on canvas**:

```
FOR EACH visually distinct text element on Figma canvas:
  1. computer(left_click, coordinate=[x,y], tabId=FIGMA_TAB)
     → Click the text element directly on canvas
  2. computer(screenshot, tabId=FIGMA_TAB)
     → Read the right-side panels:
        Typography: Font family | Weight dropdown | Size | Line height | Letter spacing
        Fill: color hex/token
        Appearance: opacity, corner radius
  3. Add to token table (see 2F below)
```

**Exhaustive click checklist:**

Text elements:
- [ ] Page title / H1
- [ ] Section headings (every level)
- [ ] Subheadings / descriptions
- [ ] Table headers + table body cells
- [ ] Button text (every variant: primary, secondary, ghost)
- [ ] Input text / placeholder / labels
- [ ] Link text / navigation links
- [ ] Badge / tag / status text
- [ ] Tooltip text
- [ ] Timer / counter / ID text
- [ ] Footer text
- [ ] Modal titles
- [ ] Financial values (often unique sizes)
- [ ] Secondary values / metadata

Non-text elements (click the shape/frame):
- [ ] Button component → corner radius, padding, fill
- [ ] Card container → radius, fill, stroke
- [ ] Table header row bg → fill color, stroke
- [ ] Input field → radius, border, height
- [ ] Modal container → radius, padding, bg
- [ ] Dividers / lines

> **RULE: Click at least 15-20 distinct elements before proceeding.** If you later find a live value matching nothing in your table — return to Figma and click that element type first.

---

### 2B. From Reference Site (CSS measurement as source of truth)

When the production/reference site IS the design source (e.g. approved build, or no Figma available):

```
1. Navigate to reference site tab
2. Use the JS measurement kit (see Phase 4.1) to measure key elements
3. Build token table from measured values
4. These become the "expected" values for the target site
```

**Workflow:**
```javascript
// Run on REFERENCE site to extract tokens
const rgbToHex = (v) => {
  if (!v || v === 'transparent' || !v.startsWith('rgb')) return v;
  const m = v.match(/[\d.]+/g);
  if (!m || m.length < 3) return v;
  const hex = '#' + m.slice(0,3).map(x => Math.round(+x).toString(16).padStart(2,'0')).join('').toUpperCase();
  const a = m.length >= 4 ? parseFloat(m[3]) : 1;
  return a < 1 ? hex + ' (a:' + a + ')' : hex;
};

const measure = (sel, label) => {
  const el = document.querySelector(sel);
  if (!el) return { label, error: 'NOT FOUND' };
  const s = getComputedStyle(el);
  return {
    label, text: el.textContent.trim().slice(0, 60),
    fontSize: s.fontSize, fontWeight: s.fontWeight,
    color: rgbToHex(s.color), bg: rgbToHex(s.backgroundColor),
    borderRadius: s.borderRadius,
    padding: [s.paddingTop, s.paddingRight, s.paddingBottom, s.paddingLeft].join(' ')
  };
};

// Measure same element categories as Figma checklist
JSON.stringify([
  measure('h1', 'Page title'),
  measure('h2', 'Section heading'),
  measure('th', 'Table header'),
  measure('td', 'Table body'),
  measure('.btn-primary', 'Primary button'),
  // ... adapt selectors to the reference site
])
```

**Key difference vs Figma:** Reference site gives you computed CSS (already resolved), so values may include browser defaults. Cross-reference with at least 2-3 instances of each element to confirm the pattern.

---

### 2C. From Screenshots / Images

When user provides design screenshots (mockups, annotated specs, UI recordings):

```
1. Read(file_path="path/to/screenshot.png") — view the image
2. Visually identify:
   - Typography: approximate sizes, weights (Bold vs Regular visible), colors
   - Layout: spacing patterns, alignment, component structure
   - Colors: sample from visible areas (ask user for exact hex if ambiguous)
3. Build token table with APPROXIMATE values + confidence level
```

**Limitations:**
- Cannot extract exact px values from screenshots — ask user to annotate or provide specs
- Colors may be affected by screenshot compression — ask user for hex palette
- Weight (600 vs 700) often indistinguishable visually — flag as "needs confirmation"

**Best practice:** Use screenshots as visual reference + ask user for key specs:
> "I can see the screenshot. To build accurate tokens, I need: exact hex colors for text/backgrounds, font sizes for headings/body, and border-radius values. Can you provide these?"

---

### 2D. From PDF Brandbook

```
1. Read(file_path="path/to/brandbook.pdf", pages="1-5") — start with first pages
2. Look for sections: Colors, Typography, Spacing, Components
3. Extract structured values into token table
4. If PDF is image-heavy (scanned), ask user for key values directly
```

**Common PDF brandbook structures:**
- Color palette page → hex values with names
- Typography page → font family, scale (sizes + weights + line-heights)
- Components page → button variants, input specs, card specs
- Spacing page → grid system, padding/margin standards

---

### 2E. From Notion

```
1. notion-fetch(id="page_url_or_id") → get Markdown content
2. Parse token tables from Markdown
3. For databases: notion-search → notion-fetch each relevant page
```

---

### 2F. Token Table Format (universal — use regardless of source)

**MANDATORY — keep accessible throughout entire audit:**

```
Source: [Figma / Reference site / PDF / Screenshots / Manual]
Confidence: [High — interactive inspection | Medium — measured from reference | Low — visual estimate]

Typography Tokens:
| Style Name        | Weight      | Size | Line-H | LS   | Where (source element)       |
|-------------------|-------------|------|--------|------|------------------------------|
| Display           | Bold 700    | 40px | Auto   | 0%   | "$12,340.56" main balance    |
| Section heading   | SemiBold 600| 28px | Auto   | 0%   | "Overview" heading           |
| Table header      | SemiBold 600| 15px | Auto   | 0%   | "Name" column header         |

Color Tokens:
| Name           | Hex      | Where                    |
|----------------|----------|--------------------------|
| Text/primary   | #FFFFFF  | Main text                |
| Text/secondary | #6B7280  | Labels, metadata         |

Component Tokens:
| Component     | Corner R | Padding     | Fill        | Other           |
|---------------|----------|-------------|-------------|-----------------|
| Button (M)    | 8px      | 16h × 12v  | #2563EB     | text: #FFFFFF   |
| Card          | 12px     | 24px        | #F9FAFB     | border: #E5E7EB |
```

### 2G. Completeness Check

Before proceeding:
- [ ] Token table has **10+ entries minimum**
- [ ] All major element types covered (headings, body, buttons, table, cards)
- [ ] At least 5 colors identified
- [ ] Key component specs (border-radius, padding) documented
- [ ] Confidence level noted (High/Medium/Low per source)

> **If confidence is Low (screenshots/manual), warn the user:** "Token extraction is approximate. False positive rate may be higher. Consider providing Figma or exact specs for critical values."

---

## Phase 3: Site Discovery

Navigate the target site. Build a checklist:

```
- Page A: default view, dropdown X open, modal Y open, tooltip Z
- Page B: default, table sorted, filter applied
```

**Interactive states to check:**
| Type | Action |
|------|--------|
| Dropdowns | Open each, scroll to see all items |
| Modals | Open each, check internal forms/dropdowns |
| Tooltips | Hover every info icon |
| Expandable | Click show more, accordions |
| Hover/Focus | Buttons, links, cards, inputs |

**SPA lazy loading:** Scroll entire page before auditing. Wait for `scrollHeight` to stabilize.

---

## Phase 4: Systematic Measurement

### 4.1 JS Measurement Kit

**Always include rgbToHex.** Always verify element exists. Always capture textContent.

```javascript
// === CORE: Copy into every javascript_tool call ===
const rgbToHex = (v) => {
  if (!v || v === 'transparent' || !v.startsWith('rgb')) return v;
  const m = v.match(/[\d.]+/g);
  if (!m || m.length < 3) return v;
  const hex = '#' + m.slice(0,3).map(x => Math.round(+x).toString(16).padStart(2,'0')).join('').toUpperCase();
  const a = m.length >= 4 ? parseFloat(m[3]) : 1;
  return a < 1 ? hex + ' (a:' + a + ')' : hex;
};

const measure = (sel, label) => {
  const el = document.querySelector(sel);
  if (!el) return { label, error: 'NOT FOUND' };
  const s = getComputedStyle(el);
  return {
    label,
    text: el.textContent.trim().slice(0, 60),
    visible: el.offsetParent !== null && el.offsetWidth > 0,
    fontSize: s.fontSize, fontWeight: s.fontWeight,
    fontFamily: s.fontFamily.split(',')[0].trim().replace(/['"]/g, ''),
    lineHeight: s.lineHeight, letterSpacing: s.letterSpacing,
    color: rgbToHex(s.color), bg: rgbToHex(s.backgroundColor),
    padding: [s.paddingTop, s.paddingRight, s.paddingBottom, s.paddingLeft].join(' '),
    borderRadius: s.borderRadius, border: s.border,
    gap: s.gap, opacity: s.opacity
  };
};

// Batch: measure 5-10 elements per call
JSON.stringify([
  measure('[data-testid="title"]', 'Title'),
  measure('.btn-primary', 'CTA'),
  measure('.card-header', 'Card header')
])
```

### 4.2 Specialized Scans

**Small font scan (catch violations across entire page):**
```javascript
const MIN = 13; // Brandbook minimum
const small = [...document.querySelectorAll('*')].filter(el =>
  el.children.length === 0 && el.offsetParent !== null &&
  el.textContent.trim().length > 0 &&
  parseFloat(getComputedStyle(el).fontSize) < MIN
).map(el => {
  const s = getComputedStyle(el);
  return { text: el.textContent.trim().slice(0, 40), fontSize: s.fontSize, fontWeight: s.fontWeight };
});
JSON.stringify({ count: small.length, items: small.slice(0, 30) })
```

**Color inventory (catch off-palette values):**
```javascript
const rgbToHex = (v) => { if (!v || v === 'transparent' || !v.startsWith('rgb')) return v; const m = v.match(/[\d.]+/g); if (!m || m.length < 3) return v; return '#' + m.slice(0,3).map(x => Math.round(+x).toString(16).padStart(2,'0')).join('').toUpperCase(); };
const colors = { text: new Set(), bg: new Set() };
[...document.querySelectorAll('*')].filter(el => el.offsetParent !== null).forEach(el => {
  const s = getComputedStyle(el);
  const c = rgbToHex(s.color); if (c && c !== 'transparent') colors.text.add(c);
  const b = rgbToHex(s.backgroundColor); if (b && b !== 'transparent') colors.bg.add(b);
});
JSON.stringify({ textColors: [...colors.text], bgColors: [...colors.bg] })
```

**Dropdown item count:**
```javascript
const c = document.querySelector('DROPDOWN_SELECTOR');
c ? JSON.stringify({
  totalItems: c.querySelectorAll('li, [role="option"], [role="menuitem"]').length,
  texts: [...c.querySelectorAll('li, [role="option"]')].slice(0, 20).map(i => i.textContent.trim()),
  scrollable: (() => { const s = c.querySelector('ul, [role="listbox"]'); return s ? s.scrollHeight > s.clientHeight : false; })()
}) : JSON.stringify({ error: 'not found' })
```

### 4.3 Cross-Verification Loop

**For EVERY measured value that doesn't match a known token:**

| Source type | Cross-verification action |
|-------------|--------------------------|
| **Figma** | Go back to Figma tab → click the corresponding element → read panel |
| **Reference site** | Switch to reference tab → measure same element with JS |
| **Screenshots** | Re-examine screenshot → ask user if value is visible/annotated |
| **PDF** | Re-read relevant PDF section → look for the value |
| **Manual specs** | Ask user: "Is {value} intentional for {element}?" |

```
1. Is this value in the token table? → match → NOT a bug
2. Not in table? → GO BACK TO SOURCE (see table above)
3. Check the corresponding element in the source
4. Source shows this value? → YES → add to token table, NOT a bug
5. Source shows different value? → NOW it's a confirmed bug
6. Source doesn't have this element? → Flag with reduced confidence, note in bug
```

> **GOLDEN RULE: Never log "value X not in design tokens" without checking the source for that specific element type first.** Your token table may be incomplete.

---

## Phase 5: Verification & Cleanup

### False Positive Rules (by frequency in real audits)

| # | Type | Rule | Frequency |
|---|------|------|-----------|
| 1 | **Incomplete token extraction** | Value exists in source but you didn't inspect that element. GO CHECK. | ~60% of FPs |
| 2 | **Color under different name** | Cross-ref hex against FULL palette. A color may serve multiple roles | ~15% of FPs |
| 3 | **Semantic mismatch, not absence** | Color IS in palette but used for wrong element type → rewrite as consistency bug, not "off-palette" | ~10% of FPs |
| 4 | **Matches production** | Same value in production → likely intentional | ~5% of FPs |
| 5 | **Imperceptible color diff** | R,G,B each ≤3 difference → dismiss | ~5% of FPs |
| 6 | **Standard UX pattern** | Dropdown trigger bold, options regular → by design | ~5% of FPs |

**Source-specific considerations:**
- **Reference site as source:** Both sites may have the same bug. Cross-reference with a third source (Figma, brandbook) when possible
- **Screenshots as source:** Values are approximate. Mark bugs with "~" prefix and note confidence: "~14px (estimated from screenshot)"
- **Multiple sources disagree:** Figma > PDF > Reference site > Screenshots > Manual. Use the highest-priority source as tiebreaker

### Design Intent Filter

NOT a bug unless design reference explicitly says otherwise:
- Different categories/roles → different colors (Admin=blue, Member=green)
- Different hierarchy levels → different sizes (H1=28, H2=18)
- Different states → different styles (active bold, inactive regular)

### Deduplication

- Systemic bug covers specific → remove specific
- Same page + same element = duplicate → keep one
- Current Value = Expected Value → not a bug, remove
- "Repeats bug #N" → should not exist

### Severity

| Severity | When |
|----------|------|
| Critical | Content hidden, actions blocked, layout collapse |
| High | Clear spec violation on primary/interactive elements |
| Medium | Deviation on secondary elements, minor inconsistencies |
| Low | Cosmetic, only visible with measurement tools |

Same violation class = same severity (unless context changes impact).

---

## Phase 6: Report

### Format Standards

| Field | Format | Example |
|-------|--------|---------|
| Current (typography) | `{size}px / {weight}` | `13px / 600` |
| Current (color) | `{hex}` | `#323D47` |
| Current (mixed) | `{size}px / {weight} / {hex}` | `12px / 500 / #A8F4FF` |
| Current (spacing) | `{value}px` | `16px 24px` |
| Expected | value + token name | `15px / 600 (Table header)` |
| Page/Section | `Page (route) → Section → Element "text"` | `Products (/products) → Table → Header "Price"` |

**Categories (strict enum):** Typography · Colors · Consistency · UX / Bug

### XLSX Generation

Use openpyxl. Customize for user's language and style. **Save incrementally** — don't wait until the end.

```python
import openpyxl
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side

wb = openpyxl.Workbook()
ws = wb.active
ws.title = 'Pixel-Perfect Audit'

# Headers — adapt language to user's preference
headers = ['Page / Section', 'Element', 'Issue', 'Category',
           'Severity', 'Current Value', 'Expected Value']

header_fill = PatternFill(start_color='1F2937', end_color='1F2937', fill_type='solid')
header_font = Font(name='Inter', size=11, bold=True, color='FFFFFF')

for i, h in enumerate(headers, 1):
    c = ws.cell(row=1, column=i, value=h)
    c.font = header_font
    c.fill = header_fill
    c.alignment = Alignment(horizontal='center', vertical='center')

# Severity fills — apply ONLY to severity column (E)
sev_fills = {
    'Critical': (PatternFill(start_color='FF0000', fill_type='solid'),
                 Font(name='Inter', size=10, color='FFFFFF')),
    'High':     (PatternFill(start_color='FF6B35', fill_type='solid'),
                 Font(name='Inter', size=10, color='FFFFFF')),
    'Medium':   (PatternFill(start_color='FFD700', fill_type='solid'),
                 Font(name='Inter', size=10, color='000000')),
    'Low':      (PatternFill(start_color='90EE90', fill_type='solid'),
                 Font(name='Inter', size=10, color='000000')),
}

# Add bugs, then create Summary sheet with category×severity matrix
# Save to project folder, not /tmp
```

### Incremental Saving Protocol

**Save the bug list to a file AFTER every page audit, not just at the end.**

```
After auditing each page/section:
  1. Append new bugs to running list (Python list or temp file)
  2. Save XLSX with current state
  3. Print progress: "Page X done: N new bugs, M total"
```

This prevents data loss when context runs out.

### Session Handoff

When context is getting low (~20% remaining), save handoff file:

```
File: {project_folder}/pixel-perfect-handoff.md
Contents:
  1. Source type + confidence level
  2. Full token table (Phase 2)
  3. Completed pages with bug counts
  4. Current bug list (all columns)
  5. Remaining pages to audit
  6. Known systemic issues
```

---

## Known Gotchas

**Critical (will cause wrong bugs):**
1. **Incomplete token extraction** — You didn't inspect that element in the source ≠ token doesn't exist. Go back and check
2. **text-transform: uppercase** — DOM text is "Project" but shows "PROJECT". Search original text, not visual
3. **Same text, different context** — "APY" in table header vs chart legend. Always capture textContent + parent
4. **Multiple identical elements** — SPA renders mobile + desktop versions. Check `el.offsetParent !== null`

**Important:**
5. **line-height: normal** — browsers compute ~1.2×. If design specifies explicit value, this is a deviation
6. **getComputedStyle returns px** — even if source is rem/em. Multiply by root font-size if comparing
7. **Gradients** — check `backgroundImage` not `backgroundColor`
8. **Pseudo-elements** — use `getComputedStyle(el, '::before')`
9. **CSS-in-JS hashed classes** — unstable selectors. Prefer `[data-testid]`, `[role]`, semantic tags
10. **Sticky/fixed elements** — audit separately, check z-index
11. **Hex case** — normalize to UPPERCASE. `getComputedStyle` returns lowercase rgb → `rgbToHex` outputs uppercase

**Source-specific gotchas:**
12. **Figma: get_design_context ≠ panel values** — Code output may not match what the Typography panel shows. Always verify by clicking
13. **Reference site: browser defaults** — Some values (line-height, letter-spacing) may be browser defaults, not intentional design. Cross-ref with 2+ instances
14. **Screenshots: compression artifacts** — Colors in JPEG screenshots may shift by 2-5 per channel. Always ask for hex palette separately
15. **PDF: scanned pages** — OCR may misread values (0 vs O, l vs 1). Double-check suspicious numbers

---

## Critical Rules

### MUST
- Extract tokens from the design source BEFORE measuring the target site
- Build token table with 10+ entries before starting measurement
- For every "not in tokens" finding → go back to source and verify
- Capture textContent with every measurement
- Save incrementally — don't wait for report completion
- Cross-reference every "off-palette" color against FULL palette
- Verify Current ≠ Expected for every bug
- Open every dropdown, modal, tooltip on every page
- Note confidence level when source is screenshots or manual specs

### NEVER
- Never log "token X doesn't exist" without checking the source for that element type
- Never start measuring before token table has 10+ entries
- Never flag different categories/roles having different styles as "inconsistency"
- Never create a bug where Current = Expected
- Never skip the cross-verification loop (Phase 4.3)
- Never enter credentials on behalf of the user
- Never treat screenshot-extracted values as exact without user confirmation
