# Murder & Kill

> Source: `packages/contracts/src/libraries/LibKill.sol` (L1–375),
> `packages/contracts/src/systems/HarvestLiquidateSystem.sol`,
> `packages/contracts/test/systems/Murder/Murder.t.sol`,
> `packages/contracts/test/systems/Murder/HiredHitman.t.sol`

## Overview

Killing (also called "murder" or "liquidation") is the core PvP mechanic.
An attacker's Kami can kill a victim's Kami that is harvesting on the same node,
if the victim's health has dropped below a calculated threshold. The kill
steals a portion of the victim's harvest bounty, rewards the killer with an
Obol, and sets the victim's Kami to DEAD state.

Kill mechanics are executed through the `HarvestLiquidateSystem` (see
[harvesting.md](../economy/harvesting.md#liquidation-pvp) for the system-level
flow). This document covers the **combat math** that determines outcomes.

## Kill Eligibility

A victim is killable by an attacker when:

```
threshold > currentHealth
```

Where `threshold` is derived from attacker Violence vs. victim Harmony, and
`currentHealth` is the victim's HP after harvest strain has been applied.

> Source: `LibKill.sol:74–82`

## Animosity (Base Threshold)

Animosity measures how aggressive the attacker is relative to the victim's
defense. It uses a **Gaussian CDF** over the log ratio of stats:

```
imbalance = lnWad(sourceViolence × 1e18 / targetHarmony)  (WAD precision)
base = Φ(imbalance)                                     (Gaussian CDF)
animosity = (base × ratio) / precision
```

Where:
- `sourceViolence` = attacker's total Violence stat
- `targetHarmony` = victim's total Harmony stat
- `ratio` = `KAMI_LIQ_ANIMOSITY[2]` (core animosity baseline)
- `precision` = `10^(18 + config[3] - 6)`

Result is in 1e6 precision (proportion of total health).

**Interpretation**: Higher attacker Violence relative to victim Harmony produces
a higher animosity value, making it easier to kill.

> Source: `LibKill.sol:89–105`

## Attack Efficacy

Efficacy modifies the threshold based on **affinity type matchups** between
attacker and victim:

```
efficacy = base + affinityShift
```

Where:
- `base` = `KAMI_LIQ_THRESHOLD[2]`
- `affinityShift` = effectiveness of attacker's **hand affinity** vs victim's
  **body affinity** (see [harvesting.md](../economy/harvesting.md) affinity
  table), modified by ATK/DEF bonus shifts

Bonus integration:
```
atkBonus = ATK_THRESHOLD_RATIO bonus on attacker
defBonus = DEF_THRESHOLD_RATIO bonus on victim
bonusShift = atkBonus - defBonus
```

> Source: `LibKill.sol:108–135`

## Kill Threshold (Final)

The final absolute HP threshold:

```
threshold = (animosity × efficacy + shift) × totalHealth / precision
```

Where:
- `shift = (ATK_THRESHOLD_SHIFT - DEF_THRESHOLD_SHIFT) × shiftPrecision`
- `totalHealth` = victim's max HP
- If `animosity × efficacy + shift < 0`, threshold = 0 (unkillable)

> Source: `LibKill.sol:139–160`

## Karma (Recoil Damage to Attacker)

The attacker takes HP damage ("karma") when killing. Karma is based on the
**victim's violence vs attacker's harmony** — a reversal of the animosity check:

```
rawKarma = nudge + victimViolence - attackerHarmony
karma = (rawKarma × efficacy × boost) / precision
```

Where:
- `nudge` = `KAMI_LIQ_KARMA[0]` (baseline offset)
- `efficacy` = calculated with roles reversed (victim attacking, attacker defending)
- `boost` = `KAMI_LIQ_KARMA[6]`

If `nudge + victimViolence - attackerHarmony < 0`, karma = 0.

> Source: `LibKill.sol:163–178`

## Recoil (Total Attacker HP Loss)

Total attacker HP damage combines karma and harvest strain:

```
core = strain × ratio + karma × 10^config[3]
boost = config[6] + ATK_RECOIL_BOOST bonus
recoil = (core × boost) / precision
```

Where:
- `strain` = attacker's harvest strain from their own harvest output
- `ratio` = `KAMI_LIQ_RECOIL[2]`
- `boost` = `KAMI_LIQ_RECOIL[6]` + `ATK_RECOIL_BOOST` bonus (combined into single multiplier)

> Source: `LibKill.sol:181–195`

## Loot Distribution

When a kill succeeds, the victim's harvest bounty is split:

### Salvage (to victim's account)

```
scaleFactor = 10^(config[3] - config[1])
ratio = config[2] + (config[0] + power) × scaleFactor + DEF_SALVAGE_RATIO bonus
salvage = bounty × ratio / precision
```

- `power` = victim's Power stat (higher Power = more salvage)
- Capped at 100% of bounty

The victim's account receives the salvage as MUSU, plus the victim Kami gets
XP equal to the salvage amount.

> Source: `LibKill.sol:49–54, 199–213`

### Spoils (to killer's harvest)

```
scaleFactor = 10^(config[3] - config[1])
ratio = config[2] + (config[0] + power) × scaleFactor + ATK_SPOILS_RATIO bonus
spoils = (bounty - salvage) × ratio / precision
```

- `power` = attacker's Power stat
- Spoils are added to the killer's **harvest bounty** (not directly to inventory)
- Capped at 100% of remaining bounty

> Source: `LibKill.sol:57–60, 217–231`

### Killer Reward

The killer's account receives **1 Obol** (item index `OBOL_INDEX`) per kill.

> Source: `LibKill.sol:63–66`

## Kill Constraints (from tests)

| Constraint | Error |
|---|---|
| Killer Kami must be HARVESTING | `"kami not HARVESTING"` |
| Killer must not be starving (HP=0) | `"kami starving.."` |
| Killer must own the attacking Kami | `"kami not urs"` |
| Killer's account must be in same room as node | `"node too far"` |
| Both Kamis must be on the same node | `"target too far"` |
| Killer must have passed cooldown | `"kami on cooldown"` |
| Victim HP must be below threshold | `"kami lacks violence (weak)"` |

> Source: `Murder.t.sol`, `HiredHitman.t.sol`

## Hired Hitman (Quest Integration)

"Hired Hitman" is not a separate system — it refers to **quests** that track
kill-related data. Kill events log data that quest objectives can check:

| Data Key | Description |
|---|---|
| `LIQUIDATE_TOTAL` | Total kills by account |
| `LIQUIDATE_AT_NODE` | Kills at a specific node |
| `LIQ_WHEN_{phase}` | Kills during a specific time phase |
| `LIQUIDATED_VICTIM` | Times killed (on victim's account) |
| `LIQ_TARGET_ACC` | Kills of a specific target account |

Quests can require killing a specific number of targets, or killing a specific
player's Kami — enabling "hitman contract" style quest objectives.

> Source: `LibKill.sol:236–278`, `HiredHitman.t.sol`
