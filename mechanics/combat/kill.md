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

> Source: `LibKill.sol:76–84`

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
- `precision` = `10^(18 + config[3] - 6)` (the `6` is the `ANIMOSITY_PREC`
  constant, `LibKill.sol:26`)

Deployed config (standard 8-slot layout
`[nudge, n_prec, ratio, r_prec, shift, s_prec, boost, b_prec]`, but only the
first four slots are set):

```
KAMI_LIQ_ANIMOSITY = [0, 0, 400, 3]
```

so `ratio = 400` and `precision = 10^(18 + 3 − 6) = 10^15`.

`Gaussian.cdf` returns a WAD value in `[0, 1e18]`, so animosity spans
`[0, 400 × 1e18 / 1e15] = [0, 400,000]` at 1e6 precision — i.e. the base
threshold saturates at **0.40 (40%) of the victim's max HP**, reached only as
attacker Violence overwhelms victim Harmony. Efficacy and the threshold shifts
(below) then scale that base.

> Source: `LibKill.sol:26, 91–107`,
> `deployment/world/state/configs/configs.ts:142–146` (`initLiquidation`,
> array at 145)

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
- `base` = `KAMI_LIQ_THRESHOLD[2]` = **1000** (i.e. 1.0x)
- `affinityShift` = **attack-triangle** effectiveness of attacker's **hand
  affinity** vs victim's **body affinity** (EERIE→SCRAP→INSECT→EERIE; see
  [affinity.md](../utility/affinity.md)), from config `KAMI_LIQ_EFFICACY` =
  `[3, 0, 500, 500, 200]` (format `[prec, neut, +, -, n-n]`; the `-` value is
  negated in code):
  - Advantage: `+500` → 1.5x threshold
  - Disadvantage: `−500` → 0.5x threshold
  - NORMAL vs NORMAL: `+200` → 1.2x
  - Neutral: `0` → 1.0x

  This is a **single check** (unlike harvest efficacy's separate body+hand
  checks), modified by ATK/DEF bonus shifts

Bonus integration:
```
atkBonus = ATK_THRESHOLD_RATIO bonus on attacker
defBonus = DEF_THRESHOLD_RATIO bonus on victim
```

The combined bonus `atkBonus − defBonus` is written **only** into the `up` and
`special` slots of the bonus-shift struct — `base` and `down` are hardcoded 0:
`Shifts{base: 0, up: atkBonus − defBonus, down: 0, special: atkBonus − defBonus}`.
`calcEfficacyShift` then selects exactly **one** slot by matchup effectiveness,
so the THRESHOLD_RATIO bonuses only take effect on some matchups:

| Matchup | Efficacy shift applied |
|---|---|
| Advantaged | `+500 + (atkBonus − defBonus)` |
| NORMAL vs NORMAL | `+200 + (atkBonus − defBonus)` |
| Neutral | `0` — bonuses have **no effect** |
| Disadvantaged | `−500` — bonuses have **no effect** |

> Source: `LibKill.sol:110–137` (bonus shift struct at 119–124),
> `LibAffinity.sol:49–58`

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
- `precision` = `10^(18 + config[3] - 3)` (result in 1e3 precision; the `3` is
  the `KARMA_PREC` constant, `LibKill.sol:27`, marked "don't change this")

Deployed config:

```
KAMI_LIQ_KARMA = [0, 0, 2000, 3, 0, 0, 0, 0]
```

so `ratio = 2000` and `precision = 10^(18 + 3 − 3) = 10^18`. With `Gaussian.cdf`
in `[0, 1e18]`, karma spans `[0, 2000]` at 1e3 precision — a multiplier of
**0 to 2.0×**. Only slots 2 and 3 are non-zero; the nudge, shift and boost
slots are unused by `calcKarma`, which reads `config[2]` and `config[3]` only.

> Source: `LibKill.sol:27, 165–179`,
> `deployment/world/state/configs/configs.ts:142–156` (`initLiquidation`,
> array at 154)

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
- `baseEfficacy` = `KAMI_LIQ_RECOIL[0]` = **1000**, passed **raw** — no division
  by `10^KAMI_LIQ_RECOIL[1]` occurs here (`LibKill.sol:210`). The `10^config[1]`
  scaling appears only inside the final combined recoil divisor (see
  [Recoil](#recoil-total-attacker-hp-loss))
- `affinityShift` = looked up from `KAMI_LIQ_KARMA_EFFICACY` config based on matchup

The `KAMI_LIQ_KARMA_EFFICACY` config uses the standard efficacy format `[prec, neut, +, -, n-n]`:

| Matchup | Shift |
|---|---|
| Advantaged (e.g., defender EERIE hand vs attacker SCRAP body) | `+1000` (increases recoil) |
| Disadvantaged (e.g., defender EERIE hand vs attacker INSECT body) | `−1000` (decreases recoil; efficacy floors at 0) |
| Neutral (different types, no triangle edge) | `0` |
| Same non-NORMAL | `0` |
| NORMAL vs NORMAL | `+400` (special case) |

> **Correction note**: the `-` slot of efficacy configs is stored positive but
> **negated** in `LibAffinity.getShifts` (`down: -1 * config[3]`). An earlier
> version of this table wrongly listed the disadvantaged case as `+1000
> (symmetric)`.

No bonuses are applied to recoil efficacy yet (hardcoded zeroes in contract).
Result is floored at 0.

> Source: `LibKill.sol:182–199`

## Recoil (Total Attacker HP Loss)

Total attacker HP damage uses karma and recoil efficacy as **multiplicative
factors** with harvest strain:

```
karma   = calcKarma(defender, attacker)
nudge   = calcRecoilEfficacy(defender, attacker, config[0])    (raw, unscaled)
boost   = max(0, config[6] + DEF_RECOIL_BOOST + ATK_RECOIL_BOOST)
recoil  = (karma + nudge) × strain × boost / 10^(config[1] + config[3] + config[7])
```

Where:
- `strain` = `LibKami.calcStrain(killerID, spoils)` — the strain the **killer**
  would incur harvesting the **spoils taken from this kill**
  (`HarvestLiquidateSystem.sol:62`). Recoil scales with the loot taken, not
  with the attacker's own harvest output
- `karma` = Gaussian CDF multiplier (see [Karma](#karma-recoil-multiplier))
- `nudge` = affinity-based efficacy (see [Recoil Efficacy](#recoil-efficacy)),
  seeded with raw `config[0]` = 1000
- `boost` = `KAMI_LIQ_RECOIL[6]` + `DEF_RECOIL_BOOST` (defender, **currently unused** — no source grants it) + `ATK_RECOIL_BOOST` (attacker)
- The divisor is a **single combined precision** `10^(config[1] + config[3] +
  config[7])` (`LibKill.sol:218–219`): it removes the nudge scale
  `10^config[1]` (= 10^3, matching karma's hardcoded 1e3 precision), a spare
  exponent `config[3]` (= 0), and the boost scale `10^config[7]` (= 10^3) in
  one step. Deployed `KAMI_LIQ_RECOIL = [1000, 3, 0, 0, 0, 0, 1000, 3]` gives
  divisor `10^6`
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
precision = 10^config[3]
salvage = ⌊bounty × ratio / precision⌋
```

- `power` = victim's Power stat (higher Power = more salvage)
- `config` = `KAMI_LIQ_SALVAGE` = `[0, 2, 0, 3, 0, 0, 0, 0]`
  (`deployment/world/state/configs/configs.ts:149`), so
  `ratio = 10 × power + DEF_SALVAGE_RATIO` and `precision = 1000`
- The cap check is `if (ratio / precision > 1) return amt` (`LibKill.sol:236`)
  — **integer division**, so it fires only at `ratio ≥ 2 × precision`
  (= 2000, i.e. 200%). For `ratio` in `(1000, 1999]` salvage **exceeds** the
  bounty and is not clamped

> ⚠️ **CONFIRMED UPSTREAM BUG (code-level, at the pin)**: at the deployed
> config a victim with Power **101–199** (and no `DEF_SALVAGE_RATIO` bonus)
> yields `ratio` 1010–1990. The clamp at `LibKill.sol:236` does not fire
> (`1010/1000 = 1`, not `> 1`), so `salvage = ⌊bounty × ratio / 1000⌋`, which
> **exceeds** the bounty. The subtraction `bounty - salvage` at
> `HarvestLiquidateSystem.sol:58` then underflows under Solidity ≥0.8 checked
> arithmetic (`LibKill.sol:2`, `HarvestLiquidateSystem.sol:2`) and the whole
> liquidation **reverts** — victims in this Power band are unliquidatable.
>
> Exact condition, since `salvage > bounty` requires the floor to clear an
> extra unit: `bounty × (ratio − 1000) ≥ 1000`, i.e.
> `bounty ≥ ⌈1000 / (10·power − 1000)⌉` — 100 MUSU at Power 101, 10 at Power
> 110, 2 from Power 150 up. Below that bounty the call still succeeds with
> `salvage = bounty` and zero spoils.
>
> Boundary values: at Power **100** (`ratio` exactly 1000) salvage equals the
> bounty exactly — no underflow, spoils computed on 0. At Power **≥ 200**
> (`ratio ≥ 2000`) the clamp fires and returns the entire bounty as salvage,
> again leaving 0 for spoils. So the reverting band is bounded on both sides.
>
> The band is far above deployed Kami Power in practice (base Power is 10,
> `configs.ts:115`), so it is a latent defect rather than an observed one.

The victim's account receives the salvage as MUSU, plus the victim Kami gets
XP equal to the salvage amount.

> Source: `LibKill.sol:51–56, 222–238`, `HarvestLiquidateSystem.sol:53–58`

### Spoils (to killer's harvest)

```
scaleFactor = 10^(config[3] - config[1])
ratio = config[2] + (config[0] + power) × scaleFactor + ATK_SPOILS_RATIO bonus
precision = 10^config[3]
spoils = ⌊(bounty - salvage) × ratio / precision⌋
```

- `power` = attacker's Power stat
- `config` = `KAMI_LIQ_SPOILS` = `[45, 2, 0, 3, 0, 0, 0, 0]`
  (`deployment/world/state/configs/configs.ts:150`), so
  `ratio = 450 + 10 × power + ATK_SPOILS_RATIO` and `precision = 1000`
- Spoils are added to the killer's **harvest bounty** (not directly to inventory)
- The cap uses the same integer-division check (`LibKill.sol:254`) and fires
  only at `ratio ≥ 2000`

> ⚠️ **CONFIRMED UPSTREAM BUG (code-level, at the pin)**: at the deployed
> config an attacker with Power **56–154** (and no `ATK_SPOILS_RATIO` bonus)
> yields `ratio` 1010–1990. The clamp at `LibKill.sol:254` uses the same
> integer division and does not fire, so
> `spoils = ⌊(bounty − salvage) × ratio / 1000⌋` — **101–199% of the remaining
> bounty**, uncapped. `sendSpoils` credits that straight onto the killer's
> harvest balance (`LibKill.sol:59–62`, `LibHarvest.incBounty`), so the killer
> gains more MUSU than the victim lost: net issuance, not a transfer.
>
> Unlike the salvage band this does **not** revert — nothing subtracts the
> spoils from a smaller quantity. Exact condition for a strict overpay:
> `(bounty − salvage) ≥ ⌈1000 / (450 + 10·power − 1000)⌉` — 100 MUSU at Power
> 56, 20 at Power 60, 3 at Power 100.
>
> Boundaries: at Power **55** `ratio` is exactly 1000 (spoils = 100% of the
> remainder, no overpay). At Power **≥ 155** (`ratio ≥ 2000`) the clamp fires
> and snaps spoils back to exactly 100% of the remainder. As with salvage, the
> band sits well above deployed Kami Power.

> ⚠️ UNCERTAIN: both functions fold the bonus in as
> `ratio = config[2] + powerTuning + ratioBonus.toUint256()`
> (`LibKill.sol:233, 251`), casting a **signed** bonus total to unsigned. A net
> negative `DEF_SALVAGE_RATIO` / `ATK_SPOILS_RATIO` would therefore fail the
> cast rather than reduce the ratio, reverting every liquidation involving that
> Kami. No deployed source grants a negative value for either type
> (`deployment/world/data/items/allos.csv`,
> `deployment/world/data/skills/skills.csv` — all `DSR`/`ASR` values are
> positive), so the path is unreachable at the pin. The cast's exact revert
> behaviour lives in `solady`'s `SafeCastLib`, an npm dependency not vendored
> in the source repo, so it is asserted from the call site only.

> Source: `LibKill.sol:59–62, 240–256`

### Killer Reward

The killer's account receives **1 Obol** (item index `OBOL_INDEX`) per kill.

> Source: `LibKill.sol:65–68`

## Kill Constraints

| Constraint | Error |
|---|---|
| Killer Kami must be HARVESTING | `"kami not HARVESTING"` |
| Killer must not be starving (HP=0) | `"kami starving.."` |
| Killer must own the attacking Kami | `"kami not urs"` |
| Killer's account must be in same room as node | `"node too far"` |
| Both Kamis must be on the same node | `"target too far"` |
| Killer must have passed cooldown | `"kami on cooldown"` |
| Victim's harvest must be ACTIVE | `"harvest inactive"` |
| Victim HP must be below threshold | `"kami lacks violence (weak)"` |

> Source: `HarvestLiquidateSystem.sol:28–50`, `Murder.t.sol`, `HiredHitman.t.sol`

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
