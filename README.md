<div align="center">

# claude-pixel-perfect-agent

**Manual design audit skill for Claude Code — verify live web apps against a design system / brandbook**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Skill-6B48FF?style=flat-square&logo=anthropic&logoColor=white)](SKILL.md)
[![Version](https://img.shields.io/badge/version-1.3.0-green.svg?style=flat-square)](SKILL.md)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](CONTRIBUTING.md)

</div>

---

> Systematically inspects every CSS property of a live web app via Chrome DevTools, compares values against a design system (brandbook), and produces a structured bug report. Catches what automated tools miss.

## What It Catches

| Category | Bug | Current | Expected | Severity |
|:---|:---|:---|:---|:---|
| Typography | Wrong font-weight on ALL headings | `600` | `700` (Bold) | Critical |
| Typography | Font-size below minimum (15px) | `12px / 600` | `15px / 500` | Critical |
| Typography | Font-size outside type scale | `28px` | `24px` or `36px` | High |
| Colors | Off-palette color | `#8B9DAD` | `#8996A3` (Second Font) | High |
| Colors | Off-palette card background | `#323D47` | `#2B343F` (Gray) | Medium |
| UX / Bug | Dropdown items hidden on overflow | items cut off | scrollable | Critical |
| Consistency | CTA button color differs across pages | `#3B7BF6` (blue) | `#A684FF` (Purple) | High |
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

- *"Run a pixel-perfect audit on staging.example.com against our brandbook"*
- *"Design audit this page against the design system"*
- *"Check if the implementation matches the brandbook"*

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
| **Design system / brandbook** | Source of truth for colors, typography, spacing |
| **Target URL** | Live web app or staging environment to audit |

> [!IMPORTANT]
> A brandbook is **required**. Without a design system as the source of truth, there is nothing to compare against — every measurement would be subjective.

---

## When to Use

| Use this skill | Don't use this skill |
|:---|:---|
| New build ready for design QA | Automated CI screenshot diffing → use [visual regression](https://github.com/maxrihter/claude-skill-visual-regression) |
| Redesign needs brandbook verification | Functional / behavioral testing → use Playwright |
| Checking implementation matches design specs | CSS performance / bundle audit |
| Finding cross-component inconsistencies | CSS linting or code-level optimization |
| Pre-launch design sign-off | Just need a single screenshot |

---

## How It Works

The audit runs in **9 phases** (matching [SKILL.md](SKILL.md)):

```
Phase 0    →  Prerequisites     Collect brandbook URL, target URL, scope, viewport
Phase 1    →  Browser Setup     Get tab, navigate, set viewport 1440×900
Phase 2    →  Token Extraction  Extract colors, typography, spacing from brandbook
Phase 3    →  Site Discovery    Map all pages, modals, dropdowns, states
Phase 4    →  Component Group   Build registry of shared components
Phase 5    →  Systematic Audit  DOM verification + getComputedStyle() measurement
Phase 6    →  Verification      Design intent filter, dedup, severity alignment
Phase 6.5  →  Self-Review       12-point quality gate: hallucinations, duplicates, nav paths
Phase 7    →  Documentation     Generate Excel/Markdown report, present verdict
```

Each element is measured via `getComputedStyle()` — font-size, font-weight, line-height, color, background, padding, margin, border-radius, gap, shadows, opacity. Deviations from the brandbook are logged with severity, screenshot, and reproduction path.

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

Verdict logic: **SHIP AS-IS** if 0 Critical + 0 High. Otherwise **FIX → RE-AUDIT**. See the full [sample report](examples/sample-report.md) for these numbers in context.

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
Collect inputs: brandbook URL, target URL, audit scope (full site or specific pages), device targets. Default viewport: 1440×900.

### Phase 1: Browser Setup
Get browser tab via `tabs_context_mcp`, navigate to target URL, set viewport size. Confirm page loads correctly.

### Phase 2: Design Token Extraction
Open the design system. Extract canonical palette (hex values), typography scale (font-family, sizes, weights, line-heights), spacing system, border-radius values, shadow definitions.

### Phase 3: Site Discovery & Navigation Map
Discover all pages and interactive states. Build a checklist: main pages, modals, dropdowns, hover states, empty states, error states, loading states.

### Phase 4: Component Grouping
Build a registry of shared components (buttons, cards, inputs, headers, footers). Map which components appear on which pages — deviations here multiply.

### Phase 5: Systematic Audit
For each page: verify DOM existence of every element before measuring → capture textContent to prove correct element → measure via batch JS snippets → compare against brandbook → log deviations with severity and 3-level navigation path.

### Phase 6: Verification & Cleanup
Design intent filter: different categories/roles/states with different styles ≠ bug. Cross-page consistency: same component must match across pages. Strict deduplication: systemic bugs absorb page-level duplicates. Severity alignment: same violation class = same severity.

### Phase 6.5: Self-Review (12-Point Quality Gate)
Audit the audit before delivery. 12 checks including: DOM existence proof (catch hallucinations), context verification (catch same-text-wrong-element), design intent check, navigation reproducibility, duplicate scan, palette cross-reference, and format standardization. In production, catches 10-15% defective entries per session.

### Phase 7: Documentation
Compile all findings into the chosen format. Attach screenshots. Sort by severity. Present summary with verdict.

</details>

---

<details>
<summary><strong>FAQ</strong></summary>

**Can I use this without a brandbook?**
No. The brandbook is the source of truth. Without it, there's nothing objective to compare against.

**How long does a full audit take?**
Depends on site complexity. Plan ~5-10 min per page. A typical 5-page app takes 20-40 minutes including report generation.

**Does it work with dark mode?**
Yes. Audit each theme separately — switch themes via the app's toggle, then measure. The brandbook should define both light and dark palettes.

**What browsers does it support?**
Chrome only — the skill requires the Chrome MCP browser extension.

**Can I audit mobile layouts?**
Yes. Set the viewport to a mobile size (e.g., 375×812) in your request.

**What's the color tolerance?**
Per-channel RGB difference of ≤3 is auto-dismissed (e.g., `#8996A3` vs `#8996A4`). Larger deviations are flagged.

**How accurate is the report?**
v1.3.0 includes an anti-hallucination protocol (DOM existence verification + textContent capture) and a 12-point self-review gate. Phase 6.5 catches the majority of hallucinations, duplicates, and factual errors before delivery. Without these safeguards, expect 10-15% of entries to be duplicates, non-bugs, or misidentified elements.

</details>

---

## See Also

| | This skill (design audit) | [visual regression](https://github.com/maxrihter/claude-skill-visual-regression) |
|:---|:---|:---|
| **Purpose** | Verify implementation matches design specs | Catch unintended visual changes after code updates |
| **How** | Manual CSS inspection via Chrome MCP | Automated Playwright screenshot diffing |
| **Input** | Design system / brandbook | Baseline screenshots |
| **Output** | Structured bug report (Excel/Markdown) | HTML diff report (pass/fail) |
| **When** | Pre-launch design QA | CI/CD pipeline, PR checks |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

[MIT](LICENSE)
