# Health & Healing

> Source: `packages/contracts/src/libraries/LibKami.sol` (L52–170),
> `packages/contracts/src/libraries/utils/LibCooldown.sol` (L1–109),
> `packages/contracts/deployment/world/state/configs/configs.ts` (L105–118)

## Overview

Health is a **depletable stat** tracked in the `sync` field of the Health
`Stat` struct. It decreases from harvest strain and increases from resting
(metabolism). When health reaches 0, the Kami dies.

Health is **not updated in real-time** — it is computed lazily via `sync()` calls
whenever the Kami performs an action.

## Kami States

A Kami is always in exactly one state:

| State | Enum Index | Description |
|---|---|---|
| `RESTING` | 1 | Default state. HP passively regenerates. |
| `HARVESTING` | 2 | Actively farming. HP drains from strain. |
| `DEAD` | 3 | HP reached 0. Cannot act until revived. |
| `721_EXTERNAL` | 4 | ERC-721 unstaked (outside game world). |

> Source: `LibKami.sol:37–43`

## Sync Mechanism

Every time a Kami acts, `LibKami.sync(id)` is called to update health based
on elapsed time:

```
if state == HARVESTING:
    deltaBalance = HarvestSystem.sync(harvestID)   // accrue harvest yield
    damage = calcStrain(kamiID, deltaBalance)      // compute HP cost
    drain(kamiID, damage)                          // subtract from sync

else if state == RESTING:
    recovery = calcRecovery(kamiID)                // compute HP gain
    heal(kamiID, recovery)                         // add to sync (capped at max)
```

After syncing, `TimeLast` is updated to `block.timestamp`.

> Source: `LibKami.sol:82–102`

## Resting Recovery (Healing)

### Metabolism Rate

The **metabolism rate** determines HP recovered per second while resting:

```
Metabolism = precision × (Harmony + nudge) × ratio × boost / 3600
```

The result is a **1e9 fixed-point** value (9 decimal places).

Config `KAMI_REST_METABOLISM` = `[nudge, n_prec, ratio, r_prec, shift, s_prec, boost, b_prec]`

**Production values**: `[20, 0, 600, 3, 0, 0, 1000, 3]`

Expanded:
```
nudge  = 20     (additive to Harmony)
ratio  = 600    (precision 10^3 = 1000 → effective 0.6)
boost  = 1000   (precision 10^3 = 1000 → effective 1.0x, before bonuses)

precision_divisor = 10^(9 - (3 + 3)) = 10^3 = 1000

Metabolism(HP/s) = 1000 × (Harmony + 20) × 600 × (1000 + bonusBoost) / 3600
```

The `REST_METABOLISM_BOOST` bonus can modify the boost multiplier.

> Source: `LibKami.sol:134–145`, `configs.ts:114`

### Recovery Calculation

Total HP recovered over a time interval:

```
recovery = floor(elapsed_seconds × metabolism / 1e9)
```

Where `elapsed_seconds = block.timestamp - TimeLast`.

> Source: `LibKami.sol:148–152`

### Example

A resting Kami with Harmony = 10, no bonuses, idle for 1 hour (3600s):

```
Metabolism = 1000 × (10 + 20) × 600 × 1000 / 3600
           = 18,000,000,000 / 3600
           = 5,000,000  (in 1e9 precision → 0.005 HP/s)

recovery = 3600 × 5,000,000 / 1,000,000,000 = 18

HP recovered in 1 hour = 18 HP → capped at max HP (total Health stat)
```

> At 0.005 HP/s, a Harmony-10 Kami recovers its full 50 HP in approximately
> 10,000 seconds (~2.8 hours). Higher Harmony significantly speeds recovery.

## Harvest Strain (HP Drain)

While harvesting, Kamis take HP damage proportional to resources gathered:

```
strain = ceil(harvestedAmount × core × boost / (harmony + denomBase))
```

Config `KAMI_HARV_STRAIN` = `[denomBase, d_prec, core, c_prec, shift, s_prec, boost, b_prec]`

**Production values**: `[20, 0, 6500, 3, 0, 0, 1000, 3]`

```
denomBase = 20     (added to Harmony in denominator — hijacked nudge field)
core      = 6500   (precision 10^3)
boost     = 1000   (precision 10^3, before bonuses)
precision = 10^(3 + 3) = 10^6

strain = ceil(amt × 6500 × (1000 + strainBoost) / (10^6 × (Harmony + 20)))
```

The `STND_STRAIN_BOOST` bonus modifies the boost value.

> Source: `LibKami.sol:155–170`, `configs.ts:129`

## Cooldown System

After certain actions, a Kami enters **cooldown** and cannot act again until it
expires.

```
cooldown_duration = KAMI_STANDARD_COOLDOWN + STND_COOLDOWN_SHIFT bonus
cooldown_end = block.timestamp + cooldown_duration
```

- **Base cooldown**: config `KAMI_STANDARD_COOLDOWN` = **180 seconds** (3 minutes)
- **Bonus**: `STND_COOLDOWN_SHIFT` can reduce (negative) or increase cooldown
- **Minimum**: cooldown cannot go below 0

A Kami is "on cooldown" when `block.timestamp < TimeNext`.

> Source: `LibCooldown.sol:25–31, 82–108`, `configs.ts:117`

## Drain & Heal Operations

Both operations modify the Health stat's `sync` value:

- **`drain(id, amt)`**: `sync(HEALTH, -amt, id)` — reduces current HP
- **`heal(id, amt)`**: `sync(HEALTH, +amt, id)` — increases current HP

The `sync` value is always clamped: `0 ≤ sync ≤ Total` (where Total includes
bonuses).

A Kami is considered "healthy" when `Health.sync > 0`.

> Source: `LibKami.sol:64–73, 202–203`
