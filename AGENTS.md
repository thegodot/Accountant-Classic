# AGENTS.md

This file provides guidance to Qoder (qoder.com) when working with code in this repository.

## Project Overview

Accountant Classic is a World of Warcraft addon that tracks gold income/expenses by source across multiple time ranges. Built on ACE3 framework with Lua 5.1.

## Testing & Development

**No traditional build/test/lint commands** - this is a WoW addon loaded directly by the game client.

**Testing**: Manual in-game testing only
- Copy addon folder to WoW's `Interface/AddOns/` directory
- Test with different WoW client versions (Classic Era, TBC, Wrath, Cata, Retail)
- Test money events: merchant transactions, repairs, quest rewards, shared loot
- Test cross-character and cross-realm data validation
- Slash commands: `/accountant` or `/acc`

**Release Packaging**: Uses CurseForge packager with `pkgmeta.yaml` configuration

## Architecture

### Event-Driven Money Tracking

The addon uses a context-based tracking system:
1. UI events set the current transaction context (category)
   - `MERCHANT_SHOW` → `MERCH` category
   - `TAXIMAP_OPENED` → `TAXI` category
   - etc. (see `Core/Constants.lua`)
2. `PLAYER_MONEY` event fires on balance change, delta is attributed to current context
3. `CHAT_MSG_MONEY` handles shared loot parsing to prevent double-counting
4. First-session baseline priming prevents initial balance from being counted as income

**Core flow**: `updateLog()` in `Core/Core.lua` handles all money changes and categorization

### Module Structure

**Gold Tracking** (`Core/`)
- `Core.lua`: Event handling, money tracking logic, UI rendering (`AccountantClassic_OnShow()`)
- `Constants.lua`: WoW version detection, category constants, defaults
- `Config.lua`: ACE3 configuration interface
- `MoneyFrame.lua`: Currency formatting
- `Template.xml`: UI frame definitions

**Currency Tracking** (`CurrencyTracker/`)
- `CurrencyCore.lua`: Main tracking logic
- `CurrencyDataManager.lua`: Data storage/retrieval
- `CurrencyEventHandler.lua`: Event processing
- `CurrencyStorage.lua`: SavedVariables management
- `CurrencyUIController.lua`: UI integration

**Data Storage**
- `Accountant_ClassicSaveData`: Per-character money tracking
- `Accountant_ClassicZoneDB`: Optional zone-level breakdown
- `Accountant_Classic_CurrencyDB`: Currency tracking
- Uses AceDB for profile management

### Multi-Client Support

Version detection in `Core/Core.lua` sets globals: `WoWClassicEra`, `WoWClassicTBC`, `WoWWOTLKC`, `WoWCataC`, `WoWRetail`

Client-specific TOC files:
- `Accountant_Classic.toc` (Retail)
- `Accountant_Classic-Classic.toc`
- `Accountant_Classic-BCC.toc`
- `Accountant_Classic-WOTLKC.toc`
- `Accountant_Classic-Cata.toc`

## Code Conventions

- Lua 5.1 syntax with WoW API
- ACE3 addon patterns (AceAddon, AceEvent, AceDB, AceConfig)
- Localization via AceLocale-3.0 (`Locale/` directory)
- Spaces for indentation, not tabs
- Third-party libraries in `Libs/` (do not modify)

## Common Modifications

**Adding new money source category**:
1. Add constant in `Core/Constants.lua`
2. Update `updateLog()` in `Core/Core.lua`
3. Add localization strings in all `Locale/localization.*.lua` files

**UI changes**:
- Frame templates: `Core/Template.xml`
- Rendering: `Core/Core.lua:AccountantClassic_OnShow()`

**Localization**:
- Edit `Locale/localization.xx.lua` files
- Follow existing key/value patterns
