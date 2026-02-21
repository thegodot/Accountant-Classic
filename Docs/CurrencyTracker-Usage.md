# Accountant Classic Currency Tracker — Quick Usage Guide

This guide covers all available `/ct` slash commands for the headless Currency Tracker. These commands do not affect the Gold tracker and only operate on currency tracking.

Quick start:

- Run `/ct` or `/ct help` in chat to display the built-in help with all commands and timeframes.

## Table of Contents

- /ct show
- /ct show-all-currencies
- /ct meta show
- /ct debug
- /ct status
- /ct discover
- /ct repair

---

## /ct show

Show detailed data for a single currency and timeframe.

Syntax:

```
/ct show <timeframe> [currencyid]
```

Timeframes:

- `this-session` (alias: `session`)
- `today`
- `prv-day`
- `this-week` (alias: `week`)
- `prv-week`
- `this-month` (alias: `month`)
- `prv-month`
- `this-year` (alias: `year`)
- `prv-year`
- `total`

Examples:

```
/ct show this-week 3008
/ct show today 3284
/ct show total 2815
```

Notes:

- If `[currencyid]` is omitted, the tracker uses the last selected currency (when available).
- Output includes totals and a breakdown by source (numeric source codes are mapped to signed keys: positive for gains, negative for spends). 

---

## /ct show-all-currencies

Show a summary across all tracked currencies for a timeframe.

Syntax:

```
/ct show-all-currencies <timeframe>
```

Examples:

```
/ct show-all-currencies this-session
/ct show-all-currencies this-week
/ct show-all-currencies total
```

Notes:

- Prints Income, Outgoing, and Net per currency.
- Use `/ct show` for a deeper breakdown of a specific currency.

---

## /ct meta show

Inspect raw metadata captured from events (for research/diagnostics). Shows occurrence counts of raw gain and lost/destroy source codes, plus the last snapshot.

Syntax:

```
/ct meta show <timeframe> <currencyid>
```

Examples:

```
/ct meta show today 3008
/ct meta show this-week 3284
```

Notes:

- Metadata does not affect aggregates; it records raw source fields for analysis.
- "Gain" counts are occurrences of `quantityGainSource`.
- "Lost/Destroy" counts are occurrences of the negative-direction source field. On WoW 11.0.2+, this field is named `destroyReason` (previously `quantityLostSource`). We continue to store it under the `lost` bucket for backward compatibility.

---

## /ct debug

Toggle structured event logging in chat. When ON, each processed change prints a multi-line debug block.

Syntax:

```
/ct debug on
/ct debug off
```

Notes:

- Debug mode does not change storage behavior; it only reports.
- Helpful for verifying deltas, source keys, and write paths.

---

## /ct status

Print internal status.

Syntax:

```
/ct status
```

Notes:

- Useful to confirm module initialization and feature flags.

---

## /ct discover

Manage dynamically discovered currencies.

Syntax:

```
/ct discover list
/ct discover track <id> [on|off]
/ct discover clear
```

Examples:

```
/ct discover list
/ct discover track 3008 on
/ct discover clear
```

Notes:

- `list` shows all discovered currency IDs.
- `track` toggles whether a discovered currency is actively tracked.

---

## /ct repair

Repair tools to correct data or re-initialize currency storage (currency tracker only; gold data unaffected).

### /ct repair init

Reset currency tracker storage for the current character.

Syntax:

```
/ct repair init
```

Notes:

- Clears currency data structures and resets options for the current character.
- Gold tracker storage is not touched.

### /ct repair adjust

Apply a signed correction across aggregates for a single currency/source.

Syntax:

```
/ct repair adjust <id> <delta> [source]
```

Examples:

```
/ct repair adjust 3008 -157 35   # decrease totals by 157 for source 35
/ct repair adjust 2815 11 35     # increase totals by 11 for source 35
```

Notes:

- Positive `<delta>` increases Income; negative `<delta>` increases Outgoing by `abs(delta)`.
- Applies to Session, Day, Week, Month, Year, and Total.

### /ct repair remove

Remove previously recorded amounts from aggregates for a single currency/source and side, with 0-clamp safety.

Syntax:

```
/ct repair remove <id> <amount> <source> (income|outgoing)
```

Examples:

```
/ct repair remove 3008 157 35 income
/ct repair remove 3008 157 35 outgoing
```

Notes:

- Use `income` to remove from recorded Income; `outgoing` to remove from recorded Outgoing.
- Applies to Session, Day, Week, Month, Year, and Total.
- Designed for true repairs (cascades all periods) and clamps at zero.

---

## Additional Notes

- Source codes are stored as signed numeric keys. Positive values represent gain codes; negative values represent spend codes.
- Human-readable labels for sources are resolved at display-time using token maps and Locale translations. Unknown sources fall back to `S:<code>`.
- Rollover (Day/Week/Month/Year → previous) happens during initialization/login to keep time buckets current.
 - WoW 11.0.2 changes:
   - `quantityChange` can be negative when spending/losing a currency. The tracker handles signed values and also auto-corrects direction on older clients when needed.
   - The loss-side field was renamed from `quantityLostSource` to `destroyReason`. The tracker shows "Destroy/Lost sources" on modern clients, while internally keeping the SavedVariables schema unchanged (`lost` bucket).
