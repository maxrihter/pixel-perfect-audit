<div align="center">

# claude-pixel-perfect-agent

**Manual design audit skill for Claude Code — verify live web apps against any design reference**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Skill-6B48FF?style=flat-square&logo=anthropic&logoColor=white)](SKILL.md)
[![Version](https://img.shields.io/badge/version-2.1.0-green.svg?style=flat-square)](SKILL.md)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](CONTRIBUTING.md)

</div>

---

> Systematically inspects every CSS property of a live web app via Chrome DevTools, compares values against a design reference (Figma, PDF brandbook, screenshots, reference site, or manual specs), and produces a structured bug report. Catches what automated tools miss.

## Supported Design Sources

| Source | How it works | Confidence |
|:---|:---|:---|
| **Figma** | Click elements on canvas → read Typography / Fill / Layout panels | High |
| **Reference site** | Measure CSS with `getComputedStyle()` on approved production build | High |
| **PDF brandbook** | Extract colors, typography scale, component specs from PDF pages | Medium–High |
| **Screenshots** | Visual inspection + user-provided hex values / font specs | Medium |
| **Notion** | Fetch design system pages via Notion API | Medium |
| **Manual specs** | User provides colors, sizes, spacing directly | Minimum viable |

> [!TIP]
> Multiple sources can be combined — e.g. Figma for typography + reference site for component dimensions. Source priority: Figma > PDF > Reference site > Screenshots > Manual.

## What It Catches

| Category | Bug | Current | Expected | Severity |
|:---|:---|:---|:---|:---|
| Typography | Wrong font-weight on ALL headings | `600` | `700` (Bold) | Critical |
| Typography | Font-size below minimum (15px) | `12px / 600` | `15px / 500` | Critical |
| Typography | Font-size outside type scale | `28px` | `24px` or `36px` | High |
| Colors | Off-palette color | `#6B7280` | `#6C727F` (Secondary Text) | High |
| Colors | Off-palette card background | `#323D47` | `#2B343F` (Surface) | Medium |
| UX / Bug | Dropdown items hidden on overflow | items cut off | scrollable | Critical |
| Consistency | CTA button color differs across pages | `#2B6CB0` (blue) | `#2563EB` (Primary) | High |
| Consistency | Badge style inconsistency | plain text / badge / uppercase | unified style | Medium |

> [!TIP]
> All examples above are from real production audits. See a full [sample audit report](examples/sample-report.md) with 12 bugs across 3 pages.

---

## Installation

### One-liner (recommended)

```bash
git clone https://github.com/maxrihter/claude-pixel-perfect-agent.git ~/.claude/skills/pixel-perfect
```

### Project-level (committed to your repo)

```bash
git clone https://github.com/maxrihter/claude-pixel-perfect-agent.git .claude/skills/pixel-perfect
```

### Symlink (auto-updates via git pull)

```bash
git clone https://github.com/maxrihter/claude-pixel-perfect-agent.git ~/pixel-perfect-agent
ln -s ~/pixel-perfect-agent ~/.claude/skills/pixel-perfect
```

> [!NOTE]
> Claude Code auto-discovers skills in `~/.claude/skills/` on session start. No restart required — just start a new conversation.

**Trigger with:**

- *"Run a pixel-perfect audit on staging.example.com against our Figma"*
- *"Design audit this page against the brandbook PDF"*
- *"Check if the implementation matches the reference site"*
- *"Compare live build to these screenshots"*

**Verify installation:**

```bash
ls ~/.claude/skills/pixel-perfect/SKILL.md
```

---

## Requirements

| Requirement | Why |
|:---|:---|
| **Claude Code** | The skill runs as a Claude Code agent skill |
| **Chrome MCP extension** | All measurements use Chrome browser automation tools |
| **Design reference** | At least one source: Figma link, PDF, reference URL, screenshots, or manual specs |
| **Target URL** | Live web app or staging environment to audit |

> [!IMPORTANT]
> A design reference is **required** — at least one source from the table above. Without a source of truth, there is nothing to compare against.

---

## When to Use

| Use this skill | Don't use this skill |
|:---|:---|
| New build ready for design QA | Automated CI screenshot diffing → use [visual regression](https://github.com/maxrihter/claude-skill-visual-regression) |
| Verify implementation matches Figma / brandbook / specs | Functional / behavioral testing → use Playwright |
| Finding cross-component inconsistencies | CSS performance / bundle audit |
| Pre-launch design sign-off | CSS linting or code-level optimization |
| Multiple design sources need reconciliation | Just need a single screenshot |

---

## How It Works

The audit runs in **7 phases** (matching [SKILL.md](SKILL.md)):

```
Phase 0    →  Prerequisites       Collect design reference, target URL, scope
Phase 1    →  Browser Setup       Open tabs for target + design source
Phase 2    →  Token Extraction    Extract tokens from design source (2A–2G per source type)
Phase 3    →  Site Discovery      Map all pages, modals, dropdowns, interactive states
Phase 4    →  Measurement         JS measurement kit + cross-verification loop
Phase 5    →  Verification        False positive rules, design intent filter, dedup, severity
Phase 6    →  Report              XLSX/Markdown generation with incremental saving
```

### Token Extraction Workflows (Phase 2)

Phase 2 adapts to your design source:

| Sub-phase | Source | Method |
|:---|:---|:---|
| 2A | Figma | Click elements on canvas → read right-side panels |
| 2B | Reference site | `getComputedStyle()` measurement as source of truth |
| 2C | Screenshots | Visual inspection + user-confirmed values |
| 2D | PDF brandbook | Parse color palettes, typography scales, component specs |
| 2E | Notion | Fetch via Notion API, parse Markdown tables |
| 2F | — | Universal token table format (used by all sources) |
| 2G | — | Completeness check (10+ entries before proceeding) |

Each element is measured via `getComputedStyle()` — font-size, font-weight, line-height, color, background, padding, margin, border-radius, gap, shadows, opacity. Deviations from the design reference are logged with severity and reproduction path.

---

## Output Format

The default output is a **structured Excel file** (`.xlsx`) with columns:

`Page / Section` · `Element` · `Issue` · `Category` · `Severity` · `Current Value` · `Expected Value`

Alternative formats: Markdown table, CSV, or Notion page. See a full [sample report](examples/sample-report.md) with 12 bugs across 3 pages.

The report ends with a summary and verdict:

| Severity | Count |
|:---|:---|
| Critical | 2 |
| High | 5 |
| Medium | 4 |
| Low | 1 |
| **Verdict** | **FIX → RE-AUDIT** |

Verdict logic: **SHIP AS-IS** if 0 Critical + 0 High. Otherwise **FIX → RE-AUDIT**.

---

## Configuration

| Option | Default | Notes |
|:---|:---|:---|
| Output format | Excel (`.xlsx`) | Also supports Markdown, CSV, Notion |
| Language | User's preferred | Report outputs in any language |
| Viewport | 1440 × 900 | Adjustable for responsive audits (e.g., 375×812 for mobile) |
| Color tolerance | ≤3 per RGB channel | `#8996A3` vs `#8996A4` = auto-dismissed |
| Severity levels | Critical / High / Medium / Low | Critical = functional bugs, Low = nitpicks |

---

<details>
<summary><strong>Phases in Detail</strong></summary>

### Phase 0: Prerequisites
Collect inputs: design reference (Figma link, PDF path, reference URL, screenshots, or manual specs), target URL, audit scope (full site or specific pages), viewport size. Default: 1440×900.

### Phase 1: Browser Setup
Open browser tabs for target site and design reference (Figma, reference site). For non-browser sources (PDF, screenshots, Notion), use file/API tools directly. Dismiss cookie banners, verify pages load.

### Phase 2: Token Extraction (Multi-Source)
Extract design tokens using the workflow matching your source type (2A–2G). Build a universal token table with typography, color, and component tokens. Minimum 10 entries before proceeding. Note confidence level per source.

### Phase 3: Site Discovery & Navigation Map
Navigate the target site. Build a checklist of all pages and interactive states: modals, dropdowns, tooltips, hover states, expandable sections. Scroll entire page for SPA lazy loading.

### Phase 4: Systematic Measurement
For each page: measure elements via batch JS (`getComputedStyle()`), compare against token table. For every "not in tokens" finding — cross-verify against the original design source before logging a bug.

### Phase 5: Verification & Cleanup
Apply false positive rules (ordered by real-world frequency). Design intent filter: different categories/roles/states = not a bug. Dedup systemic vs page-level. Align severity. Verify Current ≠ Expected for every entry.

### Phase 6: Report
Generate XLSX (openpyxl) or Markdown report. Save incrementally after each page. Include summary with category×severity matrix and verdict. Save handoff file when context runs low.

</details>

---

<details>
<summary><strong>FAQ</strong></summary>

**Can I use this without a brandbook?**
Yes — v2.1 supports multiple design sources. You can use Figma, a reference site (approved production build), screenshots with annotated specs, or even manual values. A brandbook/PDF is just one of the options.

**What's the best design source?**
Figma gives the highest confidence — you can click individual elements and read exact values from the properties panel. Reference sites are next best (real CSS). Screenshots and manual specs work but may produce more false positives.

**Can I combine multiple sources?**
Yes. For example, use Figma for typography tokens and a reference site for component dimensions. The skill tracks confidence per source and flags when sources disagree.

**How long does a full audit take?**
Depends on site complexity. Plan ~5-10 min per page. A typical 5-page app takes 20-40 minutes including report generation.

**Does it work with dark mode?**
Yes. Audit each theme separately — switch themes via the app's toggle, then measure. The design reference should define both light and dark palettes.

**What browsers does it support?**
Chrome only — the skill requires the Chrome MCP browser extension.

**Can I audit mobile layouts?**
Yes. Set the viewport to a mobile size (e.g., 375×812) in your request.

**What's the color tolerance?**
Per-channel RGB difference of ≤3 is auto-dismissed (e.g., `#8996A3` vs `#8996A4`). Larger deviations are flagged.

**How accurate is the report?**
v2.1 includes a cross-verification loop (go back to source for every "not in tokens" finding), false positive rules ordered by real-world frequency, and incremental saving to prevent data loss. Source-specific gotchas help avoid common pitfalls per source type.

</details>

---

## See Also

| | This skill (design audit) | [visual regression](https://github.com/maxrihter/claude-skill-visual-regression) |
|:---|:---|:---|
| **Purpose** | Verify implementation matches design specs | Catch unintended visual changes after code updates |
| **How** | Manual CSS inspection via Chrome MCP | Automated Playwright screenshot diffing |
| **Input** | Design reference (Figma, PDF, screenshots, etc.) | Baseline screenshots |
| **Output** | Structured bug report (Excel/Markdown) | HTML diff report (pass/fail) |
| **When** | Pre-launch design QA | CI/CD pipeline, PR checks |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

[MIT](LICENSE)
