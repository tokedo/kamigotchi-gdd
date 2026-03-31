# Murder & Kill

> Source: `packages/contracts/src/libraries/LibKill.sol` (L1–400),
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
combatRatio = lnWad(attackerViolence × 1e18 / victimHarmony)  (WAD precision)
base = Φ(combatRatio)                                        (Gaussian CDF)
animosity = (base × ratio) / precision
```

Where:
- `attackerViolence` = attacker's total Violence stat
- `victimHarmony` = victim's total Harmony stat
- `ratio` = `KAMI_LIQ_ANIMOSITY[2]` (core animosity range)
- `precision` = `10^(18 + config[3] - 6)`

Result is in 1e6 precision (proportion of total health).

**Interpretation**: Higher attacker Violence relative to victim Harmony produces
a higher animosity value, making it easier to kill.

> Source: `LibKill.sol:91–107`

## Threshold Efficacy

Efficacy modifies the kill threshold based on **affinity type matchups** between
attacker and victim. Renamed from `calcEfficacy` to `calcThresholdEfficacy` to
distinguish from the newer [Recoil Efficacy](#recoil-efficacy) system.

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

> Source: `LibKill.sol:110–137`

## Kill Threshold (Final)

The final absolute HP threshold:

```
threshold = (animosity × efficacy + shift) × totalHealth / precision
```

Where:
- `shift = (ATK_THRESHOLD_SHIFT - DEF_THRESHOLD_SHIFT) × shiftPrecision`
- `totalHealth` = victim's max HP
- If `animosity × efficacy + shift < 0`, threshold = 0 (unkillable)

> Source: `LibKill.sol:141–162`

## Karma (Recoil Multiplier)

Karma is a **Gaussian CDF-based multiplier** that scales recoil damage. It
measures the defender's violence vs the attacker's harmony — a reversal of the
animosity check. The result is a multiplier (~0 to `ratio`), not raw HP damage.

```
combatRatio = lnWad(defenderViolence × 1e18 / attackerHarmony)  (WAD precision)
base = Φ(combatRatio)                                          (Gaussian CDF)
karma = (base × ratio) / precision
```

Where:
- `defenderViolence` = victim's total Violence stat
- `attackerHarmony` = attacker's total Harmony stat
- `ratio` = `KAMI_LIQ_KARMA[2]` (core karma range)
- `precision` = `10^(18 + config[3] - 3)` (result in 1e3 precision)

**Interpretation**: A high-violence victim inflicts more karma on the attacker,
making it riskier to kill strong opponents.

> Source: `LibKill.sol:165–179`

## Recoil Efficacy

Recoil efficacy is an **affinity-based nudge** that modifies recoil damage. It
uses the **defender's hand affinity** vs the **attacker's body affinity** — the
opposite direction from [Threshold Efficacy](#threshold-efficacy).

```
efficacy = max(0, baseEfficacy + affinityShift)
```

Where:
- `baseEfficacy` = `KAMI_LIQ_RECOIL[0] / 10^KAMI_LIQ_RECOIL[1]` (from recoil config nudge/n_prec)
- `affinityShift` = looked up from `KAMI_LIQ_KARMA_EFFICACY` config based on matchup

The `KAMI_LIQ_KARMA_EFFICACY` config uses the standard efficacy format `[prec, neut, +, -, n-n]`:

| Matchup | Shift |
|---|---|
| Advantaged (e.g., defender EERIE hand vs attacker SCRAP body) | `+1000` (increases recoil) |
| Disadvantaged (e.g., defender EERIE hand vs attacker INSECT body) | `+1000` (symmetric) |
| Neutral (different types, no triangle edge) | `0` |
| Same non-NORMAL | `0` |
| NORMAL vs NORMAL | `+400` (special case) |

No bonuses are applied to recoil efficacy yet (hardcoded zeroes in contract).
Result is floored at 0.

> Source: `LibKill.sol:182–199`

## Recoil (Total Attacker HP Loss)

Total attacker HP damage uses karma and recoil efficacy as **multiplicative
factors** with harvest strain:

```
karma   = calcKarma(defender, attacker)
nudge   = calcRecoilEfficacy(defender, attacker, config[0] / 10^config[1])
boost   = max(0, config[6] + DEF_RECOIL_BOOST + ATK_RECOIL_BOOST)
recoil  = (karma + nudge) × strain × boost / precision
```

Where:
- `strain` = attacker's harvest strain from their own harvest output
- `karma` = Gaussian CDF multiplier (see [Karma](#karma-recoil-multiplier))
- `nudge` = affinity-based efficacy (see [Recoil Efficacy](#recoil-efficacy))
- `boost` = `KAMI_LIQ_RECOIL[6]` + `DEF_RECOIL_BOOST` (defender) + `ATK_RECOIL_BOOST` (attacker)
- `precision` = `10^(config[1] + config[3] + config[7])`
- Boost is clamped to min 0
- `calcKarma` is called internally (not passed as a parameter)

**Design note**: Karma and nudge are **additive** with each other, then
**multiplicative** with strain and boost. This replaced an older formula where
karma was additive with `strain × ratio`.

> Source: `LibKill.sol:202–220`

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

> Source: `LibKill.sol:51–56, 224–240`

### Spoils (to killer's harvest)

```
scaleFactor = 10^(config[3] - config[1])
ratio = config[2] + (config[0] + power) × scaleFactor + ATK_SPOILS_RATIO bonus
spoils = (bounty - salvage) × ratio / precision
```

- `power` = attacker's Power stat
- Spoils are added to the killer's **harvest bounty** (not directly to inventory)
- Capped at 100% of remaining bounty

> Source: `LibKill.sol:59–62, 242–258`

### Killer Reward

The killer's account receives **1 Obol** (item index `OBOL_INDEX`) per kill.

> Source: `LibKill.sol:65–68`

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

> Source: `LibKill.sol:261–303`, `HiredHitman.t.sol`
