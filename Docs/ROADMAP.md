# Accountant Classic — Roadmap

This document outlines planned and potential improvements for Accountant Classic. Items are not promises; they reflect current priorities and ideas.

## Near-term

- Track all in-game currencies (beyond gold)
  - Record gains/spends of currencies (e.g., Emblems, Badges, Tokens) similar to money In/Out by source
  - Display currency breakdowns alongside gold: Session, Day, Week, Month, Year, Total
  - Optional per-zone/per-activity attribution where feasible
  - Consider separate tabs or a unified currency panel with filters
- UX polish and accessibility
  - Keyboard navigation improvements, larger fonts option, frame scaling presets
  - Improved tooltips with sources, last updated timestamps
- Options
  - Toggle for priming chat message
  - Export/Import settings and saved data (per character / per account)

## Medium-term

- Enhanced categorization logic
  - More granular sources (e.g., dungeon vs. raid vs. world quests where applicable)
  - Smarter fallbacks when `AC_LOGTYPE` is unset
- Historical analytics
  - Simple trend charts (weekly/monthly net changes)
  - Top 5 sources of income/outgoing lists over a selected range
- Data maintenance
  - Archive older data into compact summaries
  - Data integrity checks and self-repair prompts

## Long-term / Ideas

- API surface for other addons
  - Lightweight API to query Accountant Classic totals and deltas by period/source
- Multi-profile support
  - Allow multiple named configurations per account with quick switching
- Localization improvements
  - Keep all new UI/messages fully localized

## Notes & Constraints

- Currency tracking feasibility varies by client version; we will use official APIs where available (e.g., `C_CurrencyInfo`) and degrade gracefully on older clients.
- We will avoid performance-heavy operations in combat and during rapid money/currency updates.
- SavedVariables size should remain modest; we will add settings to cap retention where needed.

If you have suggestions or want to help, please open an issue or PR. Contributions are welcome!
