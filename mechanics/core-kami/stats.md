# Stat System

> Source: `packages/contracts/src/libraries/LibStat.sol` (L1–411),
> `packages/contracts/src/solecs/components/types/Stat.sol` (L1–116)

## Overview

Every entity that has stats (Kamis, items, equipment, traits) uses the same
`Stat` struct. LibStat manages reading, writing, modifying, and computing
effective stat values with bonuses applied.

## The Stat Struct

```solidity
struct Stat {
    int32 base;   // permanent base value
    int32 shift;  // fixed +/- additive modifier
    int32 boost;  // percentage multiplier (3 decimals of precision, i.e. 1000 = +100%)
    int32 sync;   // current depletable value (for HP, slots, stamina)
}
```

> Source: `Stat.sol:8–13`

## Stat Types

There are **6 stat types**, each stored in its own ECS component:

| Stat | Index | Component | Depletable? |
|---|---|---|---|
| Health | 1 | `HealthComponent` | Yes — `sync` tracks current HP |
| Harmony | 2 | `HarmonyComponent` | No |
| Power | 3 | `PowerComponent` | No |
| Slots | 4 | `SlotsComponent` | Yes — `sync` tracks available slots |
| Stamina | 5 | `StaminaComponent` | Yes — `sync` tracks current stamina |
| Violence | 6 | `ViolenceComponent` | No |

> Source: `LibStat.sol:22–27`

## Formulas

### Total (effective) stat value

```
Total = ((1000 + boost) × (base + shift)) / 1000
if Total < 0 then Total = 0
```

The boost is in **per-mille** (1/1000): a boost of `500` means +50%, `1000` means
+100% (double), `-500` means -50%.

> Source: `LibStat.sol:139–142`, `Stat.sol:36–37`

### Sync (depletable value) update

```
sync = clamp(current_sync + delta, 0, max)
where max = Total (with bonuses)
```

> Source: `LibStat.sol:144–148`

## Bonus System Integration

When computing a stat's effective value, the system queries two bonus types:

| Bonus Key Pattern | Modifies | Example |
|---|---|---|
| `STAT_{TYPE}_SHIFT` | `shift` field | `STAT_HEALTH_SHIFT` |
| `STAT_{TYPE}_BOOST` | `boost` field | `STAT_POWER_BOOST` |

These bonuses come from the Bonus system (`LibBonus`) and can be applied by
equipment, skills, items, or temporary effects.

**Effective stat with bonuses:**
```
effectiveStat.base  = baseStat.base          (never modified by bonus)
effectiveStat.shift = baseStat.shift + shiftBonus
effectiveStat.boost = baseStat.boost + boostBonus
effectiveStat.sync  = baseStat.sync          (carried through)
```

> Source: `LibStat.sol:210–234`

## Stat Modification

### `modify(delta, targetID)`

Adds a `Stat` delta to an existing entity's stat:

```
result.base  = baseStat.base     (base is never changed by modify)
result.shift = baseStat.shift + delta.shift
result.boost = baseStat.boost + delta.boost
result.sync  = baseStat.sync + delta.sync
```

If `delta.sync > 0`, the sync value is also clamped to the new total (with
bonuses applied).

> Source: `LibStat.sol:65–96`

### `applyAll(deltaID, baseID)`

Applies all stat deltas from one entity (e.g., a consumable item) to another
(e.g., a Kami). Iterates over all 6 stat types.

> Source: `LibStat.sol:100–118`

## How Different Entity Types Use Stats

### Traits (on creation)
- Only `base` values are set on the trait registry entry
- On Kami instantiation, trait `base` values are **added to the Kami's `base`**
- Depletable stats (health, slots) have `sync` set to the total base value

### Consumable Items (on use)
- `base` field → updates target's `shift` (permanent stat boost)
- `sync` field → updates target's `sync` (e.g., healing potions)

### Equipment (nonfungible items)
- Track their own `base`, `shift`, `boost`, and `sync`
- `shift` and `boost` start at 0, are upgradable
- `sync` is used for depletable equipment stats (slots, durability)

> Source: `Stat.sol:15–28`

## Stat Encoding

Stats are packed into a single `uint256` for on-chain storage:

```
uint256 = (uint32(base) << 192) | (uint32(shift) << 128) | (uint32(boost) << 64) | uint32(sync)
```

Each field occupies 64 bits (though they are `int32`, they are cast to `uint32`
for packing).

> Source: `Stat.sol:99–115`
