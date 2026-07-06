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
1. Skip `DISPLAY_ONLY` entries (`LibAllo.sol:202`)
2. Skip entries whose allocation ID is 0 — `if (alloIDs[i] == 0) continue;`
   (`LibAllo.sol:203`)
3. Based on type:
   - **ITEM_DROPTABLE**: Create a commit for later reveal; returns commit ID
   - **STAT**: Modify target's stat by the packed value × multiplier
   - **BONUS**: Assign temporary bonus to target via `LibBonus.assignTemporary`
   - **CLEAR_BONUS**: Remove all bonuses from target
   - **Basic** (ITEM, etc.): Call `LibSetter.update(type, index, amount × mult, target)`

The multiplier parameter allows scaling rewards (e.g., proportional community
goal rewards multiply base values by contribution amount).

> ⚠️  UNCERTAIN: the zero-ID guard at `LibAllo.sol:203` appears intended to
> make unmatched allocation references silently no-op, but it sits *after*
> the type lookup at `:202`, and `TypeComponent.get` reverts for an entity
> with no `Type` value (`abi.decode` of empty bytes,
> `solecs/components/StringBareComponent.sol:27–29`). A zero ID would
> therefore revert at `:202` before reaching the skip. In practice reward ID
> arrays come from reverse-mapping queries (`LibAllo.sol:313–325`), which
> only return existing entities.

> Source: `LibAllo.sol:188–225`

## Creation Helpers

| Function | Description |
|---|---|
| `createBasic(...)` | Item/score/simple value reward |
| `createBonus(...)` | Temporary bonus reward (requires non-empty end type; duration unvalidated) |
| `createDT(...)` | Droptable reward with keys, weights, and roll count |
| `createStat(...)` | Stat modification reward (base/shift/boost/sync) |
| `createEmpty(...)` | Display-only reward |

Bonus allocations only validate that the end type is non-empty
(`require(!endType.eq(""), "Allo: bonus must be temporary")`,
`LibAllo.sol:105`); the `duration` value passes through to
`LibBonus.regCreate` unvalidated (`LibAllo.sol:106–114`).

> Source: `LibAllo.sol:75–161`

## Droptable Rewards

When an allocation is a droptable, distribution creates a commit that must be
revealed later (via `DroptableRevealSystem`). This uses the same commit-reveal
pattern as gacha.

> Source: `LibAllo.sol:252–261`
