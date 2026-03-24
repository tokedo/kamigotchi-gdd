# Next Step

This file is the single source of truth for what to do next. The Scribe Agent
reads this at the start of every session to know where to pick up.

## Current Phase

**Phase 2: Economy** (in progress)

## What to extract next

1. Items — `LibItem.sol`, `items.csv`, `allos.csv`
2. Inventory — `LibInventory.sol`
3. Equipment — `LibEquipment.sol`, `KamiEquipSystem.sol`
4. Crafting & recipes — `LibRecipe.sol`, `CraftSystem.sol`, `recipes.csv`
5. Droptables & loot — `LibDroptable.sol`, `droptables.csv`
6. NPC shop listings — `LibListing.sol`, `listings.csv`
7. Trading (P2P) — `LibTrade.sol`, `TradeCreate/Execute/CompleteSystem.sol`

## After this phase

Proceed to Phase 3 (Combat/PvP) per the extraction order in `survey/codebase-map.md`.

## Completed phases

- Phase 1: Core Kami (6/6) — creation, stats, experience/leveling, health/healing, death/revival, naming
- Phase 2 partial: Harvesting + Liquidation (2/7 done)
