# Bonus System

> Source: `packages/contracts/src/libraries/LibBonus.sol` (L1–368)

## Overview

The bonus system provides **stat modifiers and gameplay effects** to Kamis and
accounts. Bonuses come from skills, equipment, items, and temporary combat
effects. The system uses a **registry-instance** pattern: a registry entry
defines the bonus value, while instances track who has it and at what level.

## Architecture

### Registry (Definition)

Each bonus type is defined as a registry entry:

| Component | Description |
|---|---|
| `EntityType` | `"BONUS"` |
| `IsRegistry` | Marks as registry entry |
| `IDAnchor` | Parent registry ID (e.g., skill or equipment reference) |
| `IdSource` | Literal source entity ID (for client display) |
| `Type` | Bonus type string (e.g., `"STAT_HEALTH_SHIFT"`, `"ATK_THRESHOLD_RATIO"`) |
| `Value` | Bonus value (stored as uint256, interpreted as signed int256) |
| `Subtype` | (temporary only) End anchor type (e.g., `"UPON_HARVEST_ACTION"`) |
| `Time` | (temporary only) Duration in seconds for timed bonuses |

Registry ID: `keccak256("bonus.registry", anchorID, type)`

> Source: `LibBonus.sol:33–43, 66–92`

### Instance (Application)

When a bonus is assigned to a holder (Kami or account):

| Component | Description |
|---|---|
| `IdSource` | Points to the registry entry |
| `IDAnchor` | Permanent: parent entity. Temporary: end anchor hash |
| `IDType` | Hash for querying all bonuses of a type on this holder |
| `Level` | Multiplier (permanent bonuses stack via level increments) |
| `Time` | (timed only) Expiration timestamp |

Instance ID: `keccak256("bonus.instance", regID, holderID, duration)`

> Source: `LibBonus.sol:43–53, 140–174`

## Bonus Value Calculation

```
totalBonus = Σ(registryValue[i] × level[i])    for all instances of type on holder
```

Values are signed — bonuses can be negative (debuffs).

> Source: `LibBonus.sol:231–253`

## Permanent vs Temporary Bonuses

### Permanent

- Anchored to a specific entity (skill instance, equipment instance)
- Stack via **level** — assigning the same bonus again increments level
- Removed when the source is removed (e.g., unequipping, respecing skill)

### Temporary

- Anchored to an **end type** (when the bonus expires)
- Do **not** stack — level is always 1
- Automatically cleared on the triggering event

### Timed

- Like temporary, but expire after a **duration** (stored as end timestamp)
- Each instance gets a unique ID (based on expiration time)

> Source: `LibBonus.sol:30–52, 176–192`

## End Types (Temporary Bonus Lifecycle)

| End Type | Cleared When | Example |
|---|---|---|
| `UPON_HARVEST_ACTION` | Collect, feed, or stop harvest | Food buffs |
| `UPON_HARVEST_STOP` | Stop harvest or get liquidated | Harvest-duration bonuses |
| `UPON_DEATH` | Kami dies | Death-triggered effects |
| `UPON_KILL_OR_KILLED` | Kill or get killed | Combat-round effects |
| `UPON_LIQUIDATION` | Liquidate another Kami | Post-kill effects |
| `ON_UNEQUIP_{SLOT}` | Unequip from slot | Equipment stat bonuses |
| `TIMED` | Duration expires | Timed consumable buffs |

> Source: `LibBonus.sol:308–344`

## Known Bonus Types

Used across combat, harvesting, and stat systems:

| Bonus Type | Used By | Effect |
|---|---|---|
| `STAT_{TYPE}_SHIFT` | Stats | Adds to stat `shift` component |
| `STAT_{TYPE}_BOOST` | Stats | Adds to stat `boost` component (multiplicative) |
| `ATK_THRESHOLD_RATIO` | Kill | Modifies kill threshold efficacy (attacker) |
| `DEF_THRESHOLD_RATIO` | Kill | Modifies kill threshold efficacy (defender) |
| `ATK_THRESHOLD_SHIFT` | Kill | Flat shift to kill threshold (attacker) |
| `DEF_THRESHOLD_SHIFT` | Kill | Flat shift to kill threshold (defender) |
| `ATK_SPOILS_RATIO` | Kill | Modifies spoils percentage (attacker) |
| `DEF_SALVAGE_RATIO` | Kill | Modifies salvage percentage (defender) |
| `ATK_RECOIL_BOOST` | Kill | Modifies recoil damage (attacker) |
| `EQUIP_CAPACITY_SHIFT` | Equipment | Increases equipment slot capacity |
| `STND_COOLDOWN_SHIFT` | Cooldown | Modifies standard cooldown duration |

> Source: `LibKill.sol`, `LibStat.sol`, `LibEquipment.sol`, `LibCooldown.sol`

## Clear All

The `clearAll()` function removes all **temporary** bonuses from a holder
(UPON_HARVEST_STOP, UPON_DEATH, UPON_KILL_OR_KILLED, UPON_LIQUIDATION, TIMED).
Permanent bonuses and ON_UNEQUIP bonuses are not affected. Used by the
"Cleaning Fluid" item to reset active temporary effects.

> Source: `LibBonus.sol:337–344`

## Query Patterns

- **By parent**: all instances sharing an anchor (e.g., all bonuses from one skill)
- **By type**: all instances of a specific bonus type on a holder
  (e.g., all `STAT_HEALTH_SHIFT` bonuses on a Kami)

> Source: `LibBonus.sol:282–303`
