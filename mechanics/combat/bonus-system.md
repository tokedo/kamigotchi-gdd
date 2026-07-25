# Bonus System

> Source: `packages/contracts/src/libraries/LibBonus.sol` (L1–374)

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

- Anchored to a specific entity (skill instance)
- Stack via **level** — assigning the same bonus again increments level
- Removed when the source is removed (e.g., respecing skill)
- Equipment bonuses are **not** permanent: `LibEquipment.equip` assigns them
  via `LibBonus.assignTemporary` with an `UPON_UNEQUIP_{SLOT}` end anchor
  (`LibEquipment.sol:121–124`) — see the end-type table below. (The
  `LibBonus.sol:30` header comment lists "equip" as a permanent anchor, but
  the code path does not.)

### Temporary

- Anchored to an **end type** (when the bonus expires)
- Do **not** stack — level is always 1
- Automatically cleared on the triggering event

### Timed

- Like temporary, but expire after a **duration** (stored as end timestamp)
- Each instance gets a unique ID (based on expiration time)

> **Partially implemented.** Timed bonuses can be created and queried, but the
> cleanup function (`unassignTimed`) has its unassign logic **commented out** in
> source — timed bonuses are not automatically removed on expiry.

> Source: `LibBonus.sol:30–52, 176–192, 219–223`

## End Types (Temporary Bonus Lifecycle)

| End Type | Cleared When | Example |
|---|---|---|
| `UPON_HARVEST_ACTION` | Collect, feed, or stop harvest | Food buffs |
| `UPON_HARVEST_STOP` | Stop harvest or get liquidated | Harvest-duration bonuses |
| `UPON_DEATH` | Kami dies | Death-triggered effects |
| `UPON_KILL_OR_KILLED` | Kill or get killed | Combat-round effects |
| `UPON_LIQUIDATION` | Liquidate another Kami | Post-kill effects |
| `UPON_COOLDOWN_SET` | Cooldown is (re)set — harvest start/stop/collect, liquidation | Energy Drink's cooldown shift |
| `UPON_UNEQUIP_{SLOT}` | Unequip from slot | Equipment stat bonuses |
| `TIMED` | Duration expires | Timed consumable buffs |

> Every cooldown-reset path calls `resetUponCooldownSet`: harvest start
> (`HarvestStartSystem.sol:54`), collect (`HarvestCollectSystem.sol:89`), stop
> (`HarvestStopSystem.sol:99`), and liquidation
> (`HarvestLiquidateSystem.sol:81`). **Energy Drink**'s `STND_COOLDOWN_SHIFT`
> −30 buff is the one deployed bonus that uses this terminator
> (`deployment/world/data/items/allos.csv:11`), so it lasts until the Kami's
> next cooldown is set rather than until its next harvest action. Item 11409
> also carries the `BYPASS_BONUS_RESET` flag, so feeding it does not clear
> other temporary bonuses (`KamiUseItemSystem.sol:35–37`).

> ⚠️ **SUSPECTED UPSTREAM DATA BUG**: the deployment pipeline registers *all*
> item bonus allos — including equipment bonuses — under the `USE` use case
> with the bare terminator string from the catalog (`UPON_UNEQUIP`, no slot
> suffix) (`deployment/world/state/items/allos.ts:64`,
> `deployment/world/data/items/allos.csv:6–35`). Equipping reads the `EQUIP`
> use-case anchor (`LibEquipment.getEquipBonusAlloID`,
> `LibEquipment.sol:216–225`) and finds no registry entries there, so
> `assignTemporary` attaches nothing (`LibBonus.sol:182–183`); even if
> attached, the bare `UPON_UNEQUIP` terminator would not match the
> `UPON_UNEQUIP_{SLOT}` end type cleared on unequip (`LibEquipment.sol:48,
> 252, 276–278`). Per source data, catalog equipment bonuses neither attach
> on equip nor clear on unequip.

> Source: `LibBonus.sol:308–350`, `LibEquipment.sol:48` (`END_TYPE_PREFIX`)

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
| `ATK_RECOIL_BOOST` | Kill | Modifies recoil boost (attacker-side, added to boost total) |
| `DEF_RECOIL_BOOST` | Kill | Modifies recoil boost (defender-side, added to boost total). **Currently unused** — no item or skill grants this bonus. |
| `EQUIP_CAPACITY_SHIFT` | Equipment | Increases equipment slot capacity. **Currently unused** — no item or skill grants this bonus. |
| `STND_COOLDOWN_SHIFT` | Cooldown | Modifies standard cooldown duration |

> Source: `LibKill.sol`, `LibStat.sol`, `LibEquipment.sol`, `LibCooldown.sol`

## Clear All

The `clearAll()` function removes all **temporary** bonuses from a holder. It
clears exactly these end types: `UPON_HARVEST_ACTION` and `UPON_HARVEST_STOP`
(both via `resetUponHarvestStop`, `LibBonus.sol:314–317`),
`UPON_COOLDOWN_SET`, `UPON_DEATH`, `UPON_KILL_OR_KILLED`, `UPON_LIQUIDATION`,
and `TIMED`. Permanent bonuses and `UPON_UNEQUIP_{SLOT}` bonuses are not
affected. Used by the "Cleaning Fluid" item to reset active temporary effects.

> Source: `LibBonus.sol:341–350`

## Query Patterns

- **By parent**: all instances sharing an anchor (e.g., all bonuses from one skill)
- **By type**: all instances of a specific bonus type on a holder
  (e.g., all `STAT_HEALTH_SHIFT` bonuses on a Kami)

> Source: `LibBonus.sol:282–303`
