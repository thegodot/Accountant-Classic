# Accountant Classic — Workspace Memory Dump

This document consolidates key memories, conventions, and task directives relevant to this repository and the broader active workspaces. It is intended to be a living reference for ongoing development.

## Project Overview
- Name: `Accountant Classic`
- Purpose: Track monetary and currency changes in World of Warcraft with robust UI for Gold tracker and a headless-first Currency tracker.
- Key Modules:
  - `Core/` — Gold Tracker UI and logic (Retail via `Core.xml`, Classic via `Core-Classic.xml`).
  - `CurrencyTracker/` — Currency tracking (headless-first) with optional UI we are currently implementing for Retail.
  - `Libs/` — Ace3 and UI libraries.
  - `Locale/` — Localizations.
  - `Docs/` — Documentation.

## Active Workspaces (for context)
- `c:\Users\kamus\CascadeProjects\Accountant_Classic` — Primary addon repo (this file belongs here).
- `d:\Games\World of Warcraft\_retail_\Interface\AddOns\SavedInstances` — Separate addon workspace (reference only).
- `c:\Users\kamus\OneDrive\Documents\WoW\Koolbox` — Another addon workspace (reference only).
- `c:\Users\kamus\OneDrive\Documents\pop-os-notes\Wow` — Notes and tutorials (reference only).

## Global Development Rules (User Global Memory)
- Language:
  - Use the same language as the user for prose responses.
  - All source code (comments, logs, UI strings, errors, prompts) must be in English.
- Documentation:
  - Always include docstrings for functions and classes.
  - Add meaningful comments for complex logic.
- Support tooling:
  - When you have an API syntax question, use the Context7 MCP server to retrieve the latest documentation.
- Git hygiene:
  - Always add all the directories starting with `.` (e.g., `.windsurf`, `.vscode`, `.qoder`, `.kiro`, etc.) to `.gitignore`.
  - Include standard ignore patterns for the project’s language/stack.
- Editing workflow:
  - Always use the built-in editor (apply_patch/inline edits) to update files, including code.

## Currency Tracker Task Directives and Decisions
- Headless-first optimization (Memory ID: 99b3ddbc):
  - Make Currency Tracker headless (no UI) by default.
  - Focus on `CURRENCY_DISPLAY_UPDATE` with legacy `BAG_UPDATE` fallback.
  - Add `/ct` slash commands for data display.
  - Preserve the data model and backward compatibility.
  - Do not modify Gold tracker logic.

- Help text consistency (Memory ID: b1b4b479):
  - When adding/removing/modifying any `/ct` slash commands, update `ShowHelp()` in `CurrencyTracker/CurrencyCore.lua` to keep help in sync.

- CLI design pattern (Memory ID: c748cf65):
  - For CLI-like commands, implement shared detection/compute logic once (e.g., `Preview` helper).
  - The non-preview “apply” command must call the same preview logic before writes.
  - Avoid duplicated detection logic between preview and apply.

## UI Parity Notes (Retail)
- Gold Tracker Retail Tabs:
  - Uses atlas textures `uiframe-tab-*` and `uiframe-activetab-*` with HIGHLIGHT layer (`alpha=0.4`, `ADD`).
  - Two-row layout: first row from bottom-left offset (15, -20), horizontal spacing 5; second row starts from Tab1’s RIGHT with y = -32.
  - Runtime adjustments via PanelTemplates: `PanelTemplates_TabResize`, `PanelTemplates_SetNumTabs`, `PanelTemplates_SetTab`, `PanelTemplates_UpdateTabs`.
- Currency Tracker Retail UI (current work):
  - Template and runtime behavior replicated to match Gold’s Retail UI while keeping business logic unchanged.

## Packaging and Metadata
- `pkgmeta.yaml`:
  - Packaged as `Accountant_Classic`.
  - Excludes `CLAUDE.md` and `Docs/` from release builds.

## Conventions and Reminders
- When adding/adjusting `/ct` commands:
  - Update `CurrencyTracker/CurrencyCore.lua::ShowHelp()`.
  - Follow CLI preview/apply pattern to avoid logic duplication.
- Keep UI code changes isolated from business logic.
- Maintain backward compatibility in currency storage schema.

## Known Integration Points
- Retail vs Classic loading is decided by separate TOC files.
  - Retail: `Accountant_Classic.toc` (loads `Core/Core.xml`, etc.).
  - Classic variants: `Accountant_Classic-Classic.toc`, `Accountant_Classic-BCC.toc`, `Accountant_Classic-WOTLKC.toc`, `Accountant_Classic-Cata.toc`.
- Currency Tracker is included via `CurrencyTracker/CurrencyTracker.xml` under the relevant TOC (Retail, per current focus).

## Last Updated
- Timestamp: 2025-09-16T16:04:46.428846+09:00
