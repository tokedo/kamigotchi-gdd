# Next Step

This file is the single source of truth for what to do next. The Scribe Agent
reads this at the start of every session to know where to pick up.

## Current Phase

**Phase 2: Economy**

## What to extract next

1. Harvesting — `LibHarvest.sol`, `HarvestStart/Stop/Collect/LiquidateSystem.sol`, configs
2. Items — `LibItem.sol`, `items.csv`, `allos.csv`
3. Inventory — `LibInventory.sol`
4. Equipment — `LibEquipment.sol`, `KamiEquipSystem.sol`
5. Crafting & recipes — `LibRecipe.sol`, `CraftSystem.sol`, `recipes.csv`
6. Droptables & loot — `LibDroptable.sol`, `droptables.csv`
7. Liquidation — `HarvestLiquidateSystem.sol`, `LibHarvest.sol`

## After this phase

Proceed to Phase 3 (Combat/PvP) per the extraction order in `survey/codebase-map.md`.

## Completed phases

- Phase 1: Core Kami (6/6) — creation, stats, experience/leveling, health/healing, death/revival, naming
