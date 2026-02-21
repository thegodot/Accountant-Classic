# Gold Tracker Classification: Data-Driven Design and Safe Refactor Plan

This document proposes a data-driven architecture to make adding new Event → AC_LOGTYPE mappings safe, simple, and maintainable, without repeatedly editing `Core/Core.lua` logic.

## Goals

- Reduce code scattering of classification logic in `AccountantClassic_OnEvent()`.
- Make new mappings (e.g., Dragonflight Crafting Orders) a declarative table entry instead of imperative code.
- Improve correctness via explicit precedence, transient vs. sticky semantics, and time-bound expiry.
- Preserve current behavior and data model (backward compatibility).
- Add observability for field debugging without spamming users.

## Current State (Summary)

- File `Core/Core.lua` sets `AC_LOGTYPE` in a large `elseif` chain inside `AccountantClassic_OnEvent()`.
- `PLAYER_MONEY` triggers bookkeeping; classification is whatever `AC_LOGTYPE` is at that moment; empty string falls back to `OTHER` in `updateLog()`.
- Certain close events are intentionally not clearing context to avoid “UI closed before money lands” misclassification.
- `CHAT_MSG_MONEY` forces one-shot logging as `LOOT`.

## Problems Observed

- Classification rules are scattered and implicit; difficult to audit or extend.
- No explicit concept of transient vs. sticky contexts; handled ad-hoc.
- No consistent prioritization when multiple contexts could match.
- Hard to add new features (e.g., Crafting Orders) without touching core logic.

## Proposed Architecture

Introduce a new module: `Core/GoldClassification.lua` that defines a data-driven registry and a small dispatcher.

### Rule Schema

Each rule is a Lua table with the following shape:

- `event`: string or array of strings (event names)
- `type`: string (an `AC_LOGTYPE` value)
- `mode`: string — one of:
  - `"sticky"`: persists until overridden or explicitly cleared
  - `"transient"`: consumed by the next `PLAYER_MONEY` (with optional expiry)
  - `"clear"`: sets context to empty string
- `priority`: number — higher wins when multiple rules match
- `cond(args)`: optional function `(argsTable) -> boolean` to guard the rule (e.g., `InRepairMode()`; `invoiceType == "seller"`)
- `branch`: optional string — `"retail" | "classic"` (loader filters accordingly)
- `minTOC`/`maxTOC`: optional numbers to gate by client build
- `ttlSec`: optional number — expiry window for `transient` mode (default 5)

Example snippet (illustrative only):

```lua
-- Core/GoldClassification.lua
local Rules = {
  { event = {"MERCHANT_SHOW"}, type = "MERCH", mode = "sticky", priority = 50 },
  { event = {"MERCHANT_UPDATE"}, type = "REPAIRS", mode = "transient", priority = 80, cond = function()
      return InRepairMode() == true
    end,
    ttlSec = 5
  },
  { event = {"TAXIMAP_OPENED"}, type = "TAXI", mode = "transient", priority = 70, ttlSec = 5 },
  { event = {"MAIL_INBOX_UPDATE"}, type = "AH", mode = "transient", priority = 60, cond = isAhSellerInvoice },
  { event = {"GARRISON_MISSION_FINISHED","GARRISON_ARCHITECT_OPENED","GARRISON_MISSION_NPC_OPENED","GARRISON_SHIPYARD_NPC_OPENED"}, type = "GARRISON", mode = "sticky", priority = 40, branch = "retail" },
  { event = {"GARRISON_ARCHITECT_CLOSED","GARRISON_MISSION_NPC_CLOSED","GARRISON_SHIPYARD_NPC_CLOSED"}, type = "", mode = "clear", priority = 90, branch = "retail" },
  -- Future: Crafting Orders (retail)
  { event = {"CRAFTINGORDERS_SHOW_FRAME","CRAFTINGORDERS_ORDER_PLACED"}, type = "OTHER", mode = "transient", priority = 75, branch = "retail" },
}
return Rules
```

## Module Layout and Loader Plan (no Core/Core.lua edits yet)

To minimize risk and allow a clean review phase, we will introduce new modules without touching `Core/Core.lua` initially:

- `Core/GoldClassification.lua`
  - Pure data table of rules and small helper predicates.
  - No side effects; safe to load in any order.
- `Core/GoldClassifier.lua`
  - Tiny dispatcher that can index rules, evaluate conditions, manage sticky/transient/clear contexts, and expose a minimal API (documented below).
  - Also no side effects by default. It does not attempt to set `AC_LOGTYPE` nor hook events on its own.

Add both files to the `.toc` so they load, but do not import from `Core/Core.lua` yet. This keeps the runtime behavior 100% unchanged while we finalize the data model and tests.

### Minimal API of `Core/GoldClassifier.lua`

- `init(rules, env)`
  - `rules`: the table from `Core/GoldClassification.lua`.
  - `env`: optional table with helpers and environment (e.g., build info, branch flags, time provider, verbose flag).
  - Filters rules by `branch/minTOC/maxTOC` and builds `rulesByEvent` index.
- `onEvent(event, ...)`
  - Evaluates applicable rules for the event and updates internal context state (sticky or transient stacks) without changing `AC_LOGTYPE`.
- `useContextForMoney()`
  - Returns `{ type = "...", mode = "transient|sticky" }` if a valid context exists (consuming transient if present), otherwise `nil` to allow fallback to existing behavior (`AC_LOGTYPE` → `OTHER`).
- `getDebugState()`
  - Returns the last match information, context stack, and TTLs for verbose tooling.

> Important: Because `Core/Core.lua` remains untouched, simply loading these modules will NOT alter behavior. Wiring is deferred to a later step and is deliberately a small, reversible change.

## Deferring Core/Core.lua Wiring (design choice)

We explicitly avoid editing `Core/Core.lua` during the design and documentation phase. When you are ready to test the classifier, the minimal wiring consists of two optional calls:

1) At the very start of `AccountantClassic_OnEvent(self, event, ...)`:

```lua
GoldClassifier:onEvent(event, ...)
```

2) Inside the `PLAYER_MONEY` branch, before computing classification:

```lua
local ctx = GoldClassifier:useContextForMoney()
-- if ctx then prefer ctx.type; otherwise keep existing AC_LOGTYPE / OTHER fallback
```

These lines can be added under a feature flag (e.g., profile option or a debug cvar) to allow A/B validation.

### Dispatcher Responsibilities

Implemented as a small utility inside `Core/Core.lua` (or a new `Core/GoldClassifier.lua`) wired into `AccountantClassic_OnEvent()`:

1. Build an index: `rulesByEvent[event] = { ...rules }` at load time.
2. On any event:
   - Fetch rules for `event`.
   - Evaluate `cond` (if present). Collect matches.
   - Choose highest `priority` rule; break ties by insertion order.
   - Apply per `mode`:
     - `sticky`: `AC_LOGTYPE = rule.type`
     - `transient`: push a context `{ type, expiresAt = now + (ttlSec or 5) }`
     - `clear`: `AC_LOGTYPE = ""`
3. On `PLAYER_MONEY`:
   - If transient context exists and not expired, use it for this bookkeeping and then pop it.
   - Else use `AC_LOGTYPE` (sticky), with `""` still falling back to `OTHER` inside `updateLog()`.
4. Optional: auto-expire sticky contexts after a large timeout (configurable), but default to current behavior.

### Helper Predicates (Optional)

Centralize common conditions for reuse:

- `isAhSellerInvoice()` — scans inbox via `GetInboxInvoiceInfo()` to detect `seller` invoice.
- `isInRepairMode()` — thin wrapper over `InRepairMode()` with pcall guard.
- `isRetail()` / `isClassic()` — environment helpers.
- `isCraftingOrderContext()` — future placeholder when Blizzard exposes signals.

### Observability / Debugging

- Reuse `AccountantClassic_Verbose` to emit structured logs:
  - Event received; matched rule; mode; type; priority; remaining TTL.
  - On `PLAYER_MONEY`, log which context (transient/sticky/other) was used.
- Add a diagnostic slash command (optional): `/accountant rules` to print active rules and priorities for the current client branch.

## Systematic, Safe Refactor Steps

1. **Scaffold**
   - Create `Core/GoldClassification.lua` (read-only rules table + helper predicates) and `Core/GoldClassifier.lua` (dispatcher with minimal API: `init`, `onEvent`, `useContextForMoney`, `getDebugState`).
   - Add both files to the `.toc` so they load. Do not edit `Core/Core.lua` at this stage; runtime behavior remains unchanged.

2. **Wire the Dispatcher (non-invasive)**
   - When ready to test, import `Core/GoldClassifier.lua` in `Core/Core.lua`, and at the start of `AccountantClassic_OnEvent()` call `GoldClassifier:onEvent(event, ...)` to record a candidate context (no changes to `AC_LOGTYPE`).
   - Do not remove existing assignments; let the classifier run in parallel and, under verbose mode, log what it would have chosen.

3. **Flip Read Path First**
   - In the `PLAYER_MONEY` branch only, call `GoldClassifier:useContextForMoney()` and, if it returns a context, prefer that type; otherwise fall back to existing `AC_LOGTYPE`.
   - This is low-risk and does not interfere with how `AC_LOGTYPE` is currently set; guard with a feature flag if desired.

4. **Migrate a Few Rules**
   - Add rules mirroring the safest, well-understood mappings (e.g., `MERCHANT_SHOW → MERCH`, `MERCHANT_UPDATE + InRepairMode → REPAIRS`, `TAXIMAP_OPENED → TAXI`).
   - Confirm parity under verbose logs.

5. **Remove Redundant Branches Gradually**
   - For each mapping that matches 1:1 with a rule and is stable under testing, remove the corresponding `elseif` setter from `AccountantClassic_OnEvent()`.
   - Keep comments pointing to the registry file for future maintenance.

6. **Introduce Transient Expiry Windows**
   - For contexts like TAXI, LOOT, MAIL/AH, set `ttlSec` (default 5) and validate that delayed money changes still classify correctly.

7. **Version/Branch Gating**
   - Tag retail-only features (GARRISON/BARBER/LFG/TRANSMO) with `branch = "retail"` in the rules.
   - Loader filters out irrelevant rules at init.

8. **Crafting Orders (Future)**
   - Add `CRAFTINGORDERS_*` rules as `transient` with a short TTL.
   - Optionally, add a SecureHook on the placement API (if safe and allowed) to set a transient context right before the call.

9. **Testing & Rollout**
   - Testing matrix:
     - Merchant buy/sell, repairs on/off, taxi, quest turn-in, loot, trade, mail/auction, garrison/barber/transmog (retail).
     - Ensure OTHER fallback remains intact.
     - Verify that classification matches the existing addon behavior on stable paths.
   - Ship with verbose off by default.

10. **Documentation & Contributor Guide**
   - Update `Docs/GoldTracker-Event-Classification.md` with a “How to add a rule” section.
   - Provide `RegisterGoldRule(rule)` and `RegisterGoldRules(rules)` helpers to allow third-party module extensions if desired.

## Backward Compatibility

- `PLAYER_MONEY` diffing remains the same; only the *source selection* becomes rule-driven.
- `OTHER` fallback behavior is unchanged.
- Existing SavedVariables structures and data model remain valid; no migration required.

## Risks and Mitigations

- **Rule conflicts**: resolved by `priority`; provide a default priority range and docs.
- **Transient never consumed**: use `ttlSec` expiry; if expired, falls back to sticky or OTHER.
- **Missing rules**: default is no-op → preserves behavior.
- **Unexpected events**: ignored safely (no-op).

## Minimal Public API (Optional)

```lua
-- For internal and optional external use
function RegisterGoldRule(rule) ... end
function RegisterGoldRules(rules) ... end
```

These helpers validate schema (fields present, types) and insert rules with deterministic priority order.

---

With this in place, adding support for new features (like Crafting Orders) is reduced to: add a rule entry in `Core/GoldClassification.lua`, test under verbose logs, ship. No core event logic surgery required.
