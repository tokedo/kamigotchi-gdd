# Allocation System (Rewards)

> Source: `packages/contracts/src/libraries/LibAllo.sol` (L1–357)

## Overview

The allocation system (LibAllo) provides a **generic reward distribution
framework**. Allocations are entities that describe what to give a player when
a reward is triggered. They are used by quests, community goals, scavenging,
and any system that distributes rewards.

Allocation ID: `keccak256("reward.instance", anchorID, type, index)`

## Allocation Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"ALLOCATION"` |
| `IDSource` | Source entity that created this allocation |
| `IDAnchor` | Parent entity this allocation belongs to |
| `Type` | Reward type (see below) |
| `Index` | Item/stat index (context-dependent) |
| `Value` | Amount or raw data |

> Source: `LibAllo.sol:56–73`

## Reward Types

| Type | Description | Value Field |
|---|---|---|
| `ITEM` | Grant items to inventory | Item amount |
| `ITEM_DROPTABLE` | Roll a droptable (commit-reveal) | Number of rolls |
| `STAT` | Modify a stat (health, power, etc.) | Packed Stat struct |
| `BONUS` | Apply a temporary bonus | — (bonus defined via LibBonus) |
| `CLEAR_BONUS` | Remove all active bonuses | — |
| `DISPLAY_ONLY` | Display-only, no distribution | — |
| Other (basic) | Delegated to `LibSetter.update` | Amount |

> Source: `LibAllo.sol:194–215`

## Distribution

`LibAllo.distribute(world, components, alloIDs, multiplier, targetID)`:

For each allocation:
1. Skip `DISPLAY_ONLY` entries
2. Based on type:
   - **ITEM_DROPTABLE**: Create a commit for later reveal; returns commit ID
   - **STAT**: Modify target's stat by the packed value × multiplier
   - **BONUS**: Assign temporary bonus to target via `LibBonus.assignTemporary`
   - **CLEAR_BONUS**: Remove all bonuses from target
   - **Basic** (ITEM, etc.): Call `LibSetter.update(type, index, amount × mult, target)`

The multiplier parameter allows scaling rewards (e.g., proportional community
goal rewards multiply base values by contribution amount).

> Source: `LibAllo.sol:188–225`

## Creation Helpers

| Function | Description |
|---|---|
| `createBasic(...)` | Item/score/simple value reward |
| `createBonus(...)` | Temporary bonus reward (must have duration + end type) |
| `createDT(...)` | Droptable reward with keys, weights, and roll count |
| `createStat(...)` | Stat modification reward (base/shift/boost/sync) |
| `createEmpty(...)` | Display-only reward |

> Source: `LibAllo.sol:75–161`

## Droptable Rewards

When an allocation is a droptable, distribution creates a commit that must be
revealed later (via `DroptableRevealSystem`). This uses the same commit-reveal
pattern as gacha.

> Source: `LibAllo.sol:252–261`
