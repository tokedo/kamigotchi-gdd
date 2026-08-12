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

> Source: `LibBonus.sol:228–244` (`calcSingle` + `get`)

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

## Stacking Rules

Whether two applications of "the same" bonus add up is decided entirely by the
**instance ID**, `keccak256("bonus.instance", regID, holderID, duration)`, where
`regID = keccak256("bonus.registry", anchorID, type)` and `anchorID` is the
*source* the bonus hangs off (a skill registry entry, a node's bonus anchor, an
item's allocation entry).

| Case | Stacks? | Why |
|---|---|---|
| Same non-timed source applied twice (e.g. eating a second Hostility Potion) | **No** | `duration = 0` makes the instance ID deterministic; `assign` early-returns at `LibBonus.sol:158` if the instance already exists, and `assignTemporary` then re-writes `Level = 1` regardless (`LibBonus.sol:177–192`). The second application is a no-op. |
| Two **different** sources granting the same bonus type (e.g. Flash Talisman's `ATK_THRESHOLD_RATIO` and Inverted Teardrop Jewel's) | **Yes** | Different `anchorID` → different `regID` → different instance. Both instances survive and `LibBonus.get` sums them. |
| Permanent bonuses (skills) | **Yes, by level** | `incBy` increments the `Level` component, and the total is `Σ registryValue × level` (`LibBonus.sol:194–211, 228–244`). |
| Timed bonuses | **Yes** | The end timestamp enters the instance ID, so each application is a fresh entity — but nothing removes them (see the timed caveat above). |

Using several copies of an item in one call does not help either: the
allocation multiplier `mult` is threaded through `LibAllo.distribute` but
`giveBonus` ignores it entirely (`LibAllo.sol:241–248`), and the Kami-facing
item systems pass `1` regardless (`KamiUseItemSystem.sol:42`,
`KamiCastItemSystem.sol:38`).

> Source: `LibBonus.sol:43–48` (header contract: "temp bonuses do not stack"),
> `140–174` (`assign`), `176–192` (`assignTemporary`), `194–211` (`incBy`),
> `228–244` (summation), `359–366` (`genInstanceID`), `LibAllo.sol:241–248`

## End Types (Temporary Bonus Lifecycle)

| End Type | Cleared When | Example |
|---|---|---|
| `UPON_HARVEST_ACTION` | Collect, feed, or stop harvest | Food buffs |
| `UPON_HARVEST_STOP` | Stop harvest or get liquidated | Harvest-duration bonuses |
| `UPON_DEATH` | Holder is **liquidated** (victim side only) | Curse Tablet's threshold debuff |
| `UPON_KILL_OR_KILLED` | Holder liquidates **or** is liquidated | Flash Talisman's `_KK` buffs |
| `UPON_LIQUIDATION` | Holder **successfully liquidates** another Kami (killer side only) | Hostility Potion, Inverted Teardrop Jewel |
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

### Combat Buff Reset — Exact Semantics

A liquidation resets **different end types on each side**, and the two combat
resetters overlap on `UPON_KILL_OR_KILLED`:

```
resetUponDeath(holder)        → clears UPON_DEATH        + UPON_KILL_OR_KILLED
resetUponLiquidation(holder)  → clears UPON_LIQUIDATION  + UPON_KILL_OR_KILLED
```

`HarvestLiquidateSystem` applies them asymmetrically:

| Side | Resetters called | Line |
|---|---|---|
| Killer | `resetUponLiquidation`, then `resetUponCooldownSet` | `HarvestLiquidateSystem.sol:79, 81` |
| Victim | `resetUponHarvestStop`, then `resetUponDeath` | `HarvestLiquidateSystem.sol:86–87` |

Consequences worth stating plainly:

- An **`UPON_LIQUIDATION`** buff is consumed only by the holder's own
  *successful* kill. A failed attempt reverts before any reset, and being
  liquidated does **not** consume it — the buff survives the holder's death and
  is still there after revival.
- An **`UPON_DEATH`** debuff is consumed only when the holder is liquidated. The
  holder killing someone else does **not** clear it.
- An **`UPON_KILL_OR_KILLED`** bonus is consumed by *either* event, since both
  resetters unassign it.
- Liquidation is the only death path that runs a bonus reset at all. Sacrifice
  also sets the Kami to `DEAD` (`LibSacrifice.sol:110` → `LibKami.kill`,
  `LibKami.sol:76–79`) but calls no `LibBonus` resetter — the entity leaves play
  with its bonuses attached.

Combined with the [stacking rules](#stacking-rules), a combat buff is therefore
strictly **one-shot and non-cumulative per source**: a second dose before the
trigger fires adds nothing, and the first dose persists indefinitely until its
specific trigger occurs.

> Source: `LibBonus.sol:324–334`, `HarvestLiquidateSystem.sol:76–87`,
> `LibSacrifice.sol:102–114`, `LibKami.sol:76–79`

### `BYPASS_BONUS_RESET` (item flag)

`BYPASS_BONUS_RESET` is a **flag on the item registry entry**, not a bonus end
type. It controls exactly one branch: whether using the item on your own Kami
first wipes that Kami's `UPON_HARVEST_ACTION` bonuses.

```solidity
// KamiUseItemSystem.sol:34–37
if (!LibItem.bypassBonusReset(components, itemIndex)) {
  LibBonus.resetUponHarvestAction(components, kamiID);
}
```

- Read via `LibItem.bypassBonusReset` → `LibFlag.has(genID(index),
  "BYPASS_BONUS_RESET")` (`LibItem.sol:239–243`; the doc comment there reads
  "bonus only resets upon USE").
- Semantics: **without** the flag, feeding a Kami counts as a harvest action and
  destroys any active `UPON_HARVEST_ACTION` buff (Bless Potion's bounty boost,
  Grace Potion's strain reduction, node bonuses' sibling end type, …). **With**
  the flag, the item can be fed mid-harvest without collateral damage. It does
  **not** exempt the item's own bonuses from their own end types.
- Scope: the flag is only consulted on the own-Kami `USE` path.
  `KamiCastItemSystem` (`ENEMY_KAMI` items) and `AccountUseItemSystem` never
  call the reset at all, so the flag is inert on those paths — including on
  Cthonic Blight (19201), which carries the flag but is an `Enemy_Kami` item.
- **17** deployed items carry the flag
  (`deployment/world/data/items/items.csv`) — 16 `Kami`-target consumables
  (11224–11226, 11401–11410, 11413, 11501–11502) plus the one `Enemy_Kami`
  item noted above.

> Source: `KamiUseItemSystem.sol:34–37`, `LibItem.sol:239–243`,
> `LibBonus.sol:308–311`, `KamiCastItemSystem.sol:18–46`,
> `AccountUseItemSystem.sol:18–36`

### The `_KK` Effect-Key Suffix

`_KK` is **catalog naming convention, not a code construct**. Nothing in
`packages/contracts/src/` reads or parses it. In
`deployment/world/data/items/allos.csv` the `Name` column is a free-form key
that `items.csv:Effects` references by exact string match; `_KK` was appended to
distinguish a *kill-scoped* variant of an effect from an otherwise identical
key. The real semantics live entirely in that row's `Terminator` column, which
becomes the bonus registry's end type.

| Effect key | Bonus type | Value | Terminator (end type) | Carried by |
|---|---|---|---|---|
| `ATR+10%_KK` | `ATK_THRESHOLD_RATIO` | +100 | `UPON_KILL_OR_KILLED` | Flash Talisman (11412) |
| `DTR-10%_KK` | `DEF_THRESHOLD_RATIO` | −100 | `UPON_KILL_OR_KILLED` | Flash Talisman (11412) |
| `ATS-30%_KK` | `ATK_THRESHOLD_SHIFT` | −300 | `UPON_DEATH` | Curse Tablet (19301) |

So, answering the three open questions directly:

- **Scope** — none of its own. `_KK` is a key-name suffix; the bonus applies to
  the target the item is used/cast on, exactly like any other bonus allo.
  It exists to disambiguate `ATR+10%_KK` from the plain `ATR+10%` key (same
  bonus type and value, `UPON_LIQUIDATION` terminator, used by the Inverted
  Teardrop Jewel).
- **Duration** — set by the `Terminator` column, not by the suffix. Two of the
  three `_KK` keys end on `UPON_KILL_OR_KILLED`; **`ATS-30%_KK` ends on
  `UPON_DEATH`**, so the suffix no longer reliably signals its own terminator.
- **Stacking** — none beyond the general rule: one instance per (source item,
  bonus type, holder), level fixed at 1. Flash Talisman's two effects are two
  distinct bonus types and so coexist; a second Flash Talisman adds nothing.

> ⚠️ **Naming drift.** `_KK` reads as "kill or killed", and `ATS-30%_KK`
> (Curse Tablet) was retargeted from `UPON_KILL_OR_KILLED` to `UPON_DEATH`
> without renaming the key. The debuff therefore survives the target *killing*
> someone and clears only when the target is itself liquidated. Trust the
> `Terminator` column, never the key name.

> Source: `deployment/world/data/items/allos.csv:22, 32, 38` (`_KK` rows; the
> plain `ATR+10%` row at 33), `deployment/world/data/items/items.csv:114, 121`
> (items 11412, 19301), `deployment/world/state/items/allos.ts:58–65`
> (`addBonus` maps `Descriptor` → bonus type and `Terminator` → end type),
> `LibAllo.sol:93–115` (`createBonus`), `LibBonus.sol:66–92` (`regCreate`)

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
