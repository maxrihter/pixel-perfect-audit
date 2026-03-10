# Changelog

All notable changes to this project are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/).

## [2.2.0] - 2026-03-10

### Added
- **Token table persistence** (Phase 2G) — save token table to `audit-tokens.md` immediately after Phase 2G, before Phase 3. Update on every cross-verification discovery
- **Selector priority list** (Phase 4.1) — ordered hierarchy: `[data-testid]` > semantic > stable class > positional > ❌ hashed. Filter-by-text fallback snippet for when selectors fail
- **Hover / Focus state measurement** (new Phase 4.3) — `:focus` via `el.focus()` + immediate `getComputedStyle`; `:hover` via `computer(hover)` + single-property JS call in 150ms window
- **Complete XLSX row population recipe** (Phase 6) — `enumerate(bugs)` loop, `Alignment(wrap_text, vertical='top')`, severity cell fills, `column_dimensions` auto-width with 60px cap, `freeze_panes = 'A2'`

### Changed
- **Phase numbering** — 4.3 (Cross-Verification Loop) renamed to 4.4; new 4.3 is Hover/Focus measurement
- **Cross-verification approach** (Phase 4.4) — now batch-first: accumulate all unknowns per page, group by element type, verify in one source pass instead of per-value checks
- **NEVER rule** updated — references Phase 4.4 instead of 4.3

## [2.1.0] - 2026-03-09

### Added
- **Multi-source token extraction** — Phase 2 now supports 6 source types: Figma, Reference site, Screenshots, PDF brandbook, Notion, Manual specs
- **Figma canvas clicking workflow** (Phase 2A) — click elements on canvas, read Typography/Fill/Layout panels. Exhaustive checklist with 15+ element types
- **Reference site as source** (Phase 2B) — use `getComputedStyle()` on approved production build as design truth
- **Screenshot workflow** (Phase 2C) — visual inspection with confidence tagging and user-confirmed values
- **PDF brandbook workflow** (Phase 2D) — structured extraction of color palettes, typography scales, component specs
- **Notion workflow** (Phase 2E) — fetch design system pages via Notion API
- **Universal token table format** (Phase 2F) — standardized table with Typography / Color / Component tokens, used by all source types
- **Completeness check** (Phase 2G) — minimum 10 entries, 5 colors, key component specs before proceeding
- **Cross-verification loop** (Phase 4.3) — for every "not in tokens" finding, go back to the original source and verify before logging a bug
- **Source-specific gotchas** (#12–15) — Figma `get_design_context` vs panel values, browser defaults on reference sites, screenshot compression artifacts, PDF OCR misreads
- **Source priority hierarchy** — Figma > PDF > Reference site > Screenshots > Manual
- **Confidence levels** — High (Figma, reference site), Medium (PDF, screenshots, Notion), Low (manual)
- **Incremental saving protocol** — save XLSX after every page audit, not just at the end
- **Session handoff file** — save full state (tokens, bugs, progress) when context runs low

### Changed
- **BREAKING: Audit phases restructured** — 9 phases (0–7 + 6.5) → 7 phases (0–6). Phase 4 (Component Grouping) and Phase 6.5 (Self-Review) merged into adjacent phases
- **Phase 2 expanded** — from single-source brandbook extraction to multi-source with sub-phases 2A–2G
- **Phase 4** — now combines measurement + cross-verification (was just "Systematic Audit")
- **Phase 5** — now covers verification + cleanup + false positive rules (was split across Phase 6 + 6.5)
- **Phase 6** — now focused on report generation with incremental saving (was "Documentation")
- **False positive rules** — reordered by real-world frequency: incomplete token extraction (~60%), color under different name (~15%), semantic mismatch (~10%), matches production (~5%), imperceptible diff (~5%), standard UX pattern (~5%)
- Reduced total line count from 680 → 571 while adding multi-source support
- Requirements: "brandbook required" → "at least one design reference required"

### Removed
- Phase 4 (Component Grouping) as standalone phase — absorbed into measurement workflow
- Phase 6.5 (12-Point Self-Review) as standalone phase — rules integrated into Phase 5 verification
- Chrome MCP tool reference table — removed verbose tool descriptions (users already have MCP docs)
- Redundant CSS knowledge sections that duplicated browser documentation

## [1.3.0] - 2026-03-08

### Changed
- **BREAKING: Moved SKILL.md to repo root** — `git clone` now works directly without nested paths. Previous: `pixel-perfect/SKILL.md` → Now: `SKILL.md`
- Excel template aligned to 7-column format (removed N, Screenshot, Route columns)
- Fixed Phase 5.0 JS code: added inline `rgbToHex`, changed from statement to expression (was returning `undefined`)
- Fixed sample report bug #9 wording that contradicted dedup rules
- Fixed README summary table to match sample report data
- Softened FAQ accuracy claim (was absolute "zero errors", now "catches majority")

### Added
- Session handoff protocol in Phase 3 — save/resume audit across sessions via `/tmp/pixel-perfect-handoff.md`
- Batch measurement guidance in Phase 5.1 — reduce tool calls from ~200 to ~30
- Hex normalization step in Phase 2 — prevents case-mismatch false positives
- SPA lazy-load guidance in Phase 3 — scroll to trigger all lazy content before audit
- CHANGELOG.md (this file)
- Versioning scheme in CONTRIBUTING.md
- Testing instructions in CONTRIBUTING.md

### Fixed
- `rgbToHex` undefined in Phase 5.0 mandatory code snippet
- Excel template `bug[5]` index mismatch with 7-column format
- LICENSE year: 2025 → 2025-2026

## [1.2.0] - 2026-03-08

### Added
- **Anti-hallucination protocol** (Phase 5.0) — DOM existence verification + textContent capture before any bug is logged
- **Design intent filter** (Phase 6) — different categories/roles/states having different styles is NOT a bug
- **Context disambiguation** (Phase 5.3) — verify parent context when multiple elements share same text
- **Self-review expanded to 12 points** (Phase 6.5) — DOM existence proof, context verification, design intent check, navigation reproducibility
- Gotchas #13 (same text, different context) and #14 (intentional category differentiation)
- 6 new MUST DO rules (DOM existence, textContent capture, double-measure, design intent filter, 3-level nav, parent context)
- 4 new NEVER DO rules

### Changed
- Strengthened navigation format — minimum 3 levels with URL route and quoted element text
- Strengthened dedup rules — systemic+page overlap, same element different wording
- Updated severity classification with context-dependent examples

## [1.1.0] - 2026-03-08

### Added
- Initial public release
- 9-phase audit workflow (Phase 0–7 + Phase 6.5)
- Chrome MCP tool reference table
- CSS measurement techniques with reusable JS snippets
- Excel report generation via openpyxl
- 12 known gotchas
- Small font scan and color inventory utilities

## [1.0.0] - 2025

- Internal prototype
