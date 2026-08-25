# Harvesting

> Source: `packages/contracts/src/libraries/LibHarvest.sol` (L1–439),
> `packages/contracts/src/systems/HarvestStartSystem.sol`,
> `packages/contracts/src/systems/HarvestStopSystem.sol`,
> `packages/contracts/src/systems/HarvestCollectSystem.sol`,
> `packages/contracts/src/systems/HarvestLiquidateSystem.sol`,
> `packages/contracts/src/libraries/LibKill.sol`,
> `packages/contracts/src/libraries/LibTax.sol`,
> `packages/contracts/src/libraries/utils/LibAffinity.sol`,
> `packages/contracts/deployment/world/state/configs/configs.ts` (L127–137)

## Overview

Harvesting is the primary resource-generation loop in Kamigotchi. A Kami is
assigned to a **node** (a resource spot within a room) and passively generates
**Musu** (the base resource) over time. Harvesting drains HP via **strain**,
and other players can **liquidate** (PvP raid) active harvests.

Strictly, the harvest pays out the node's configured yield item
(`LibNode.getItem`, applied in `LibHarvest.claim` — `LibHarvest.sol:97–99`),
set per node via the `YieldIndex` column in deployment data: 53 nodes yield
Musu (item 1) and 11 yield item 2 — see `catalogs/rooms/nodes.csv`.

## Harvest Lifecycle

```
START → [accruing bounty, taking strain] → COLLECT (partial) or STOP (full)
                                         ↘ LIQUIDATE (PvP death)
```

### Start (`HarvestStartSystem`)

**Inputs**: `kamiID`, `nodeIndex`, `taxerID`, `taxAmt`

**Prerequisites**:
1. Kami is owned by caller's account
2. Kami state is `RESTING`
3. Kami is not on cooldown
4. Kami is healthy (HP > 0)
5. Account and node are in the same room
6. Node-specific requirements are met (e.g., level, items)

**Process**:
1. Sync Kami health (pre-bonus)
2. Assign node bonuses to the Kami
3. Sync again (apply bonuses)
4. Create or reuse harvest entity, link to node
5. Set up tax (if `taxAmt > 0`, the `taxerID` receives `taxAmt` **basis
   points** of output, capped at 2000 bp = 20% — see [Tax System](#tax-system))
6. Set Kami state → `HARVESTING`
7. Start cooldown (consumes `UPON_COOLDOWN_SET` bonuses, e.g. Energy Drink)

> Source: `HarvestStartSystem.sol:24–57`

### Collect (`HarvestCollectSystem`)

Withdraws accrued bounty **without stopping** the harvest.

**Prerequisites**: ownership, state `HARVESTING`, cooldown clear, healthy, same room

**Process**:
1. Sync Kami health
2. Claim accrued balance (split by tax)
3. Transfer items to account (and tax recipients)
4. Grant **XP equal to the post-tax output** to the Kami
5. Add the post-tax output to the account's scavenge bar for the node — no
   droptable roll here (see `mechanics/world/scavenging.md`)
6. Increment the account's leaderboard score by the post-tax output
   (`LibScore.incFor`, `HarvestCollectSystem.sol:100`)
7. Reset cooldown (consumes `UPON_COOLDOWN_SET` bonuses, e.g. Energy Drink)
8. Reset harvest-action bonuses

> Source: `HarvestCollectSystem.sol:84–113`

### Stop (`HarvestStopSystem`)

Collects all accrued bounty **and ends** the harvest.

**Prerequisites**: same as Collect

**Process**:
1. Sync Kami health
2. Claim full accrued balance (split by tax)
3. Stop harvest (set `INACTIVE`, zero balance)
4. Set Kami state → `RESTING`
5. Grant XP equal to the post-tax output
6. Add the post-tax output to the account's scavenge bar for the node (no
   droptable roll here)
7. Increment the account's leaderboard score by the post-tax output
   (`LibScore.incFor`, `HarvestStopSystem.sol:110`)
8. Reset all harvest-stop bonuses
9. Reset cooldown (consumes `UPON_COOLDOWN_SET` bonuses, e.g. Energy Drink)
10. Log harvest time

> Source: `HarvestStopSystem.sol:88–124`

### Liquidate (`HarvestLiquidateSystem`)

A PvP action: one harvesting Kami raids another's harvest, stealing resources
and killing the victim. See [Liquidation](#liquidation-pvp) below.

## Bounty Formula

The total accrued bounty since last sync:

```
Bounty = (Fertility + Intensity) × Duration × Boost / Precision
```

Where:
- **Fertility** = power-based, steady rate (see below)
- **Intensity** = violence-based, grows over time (see below)
- **Duration** = seconds since last sync
- **Boost** = config `KAMI_HARV_BOUNTY[6]` + `HARV_BOUNTY_BOOST` bonus
- **Precision** = `10^(RATE_PREC + config[3] + config[7])`

Config `KAMI_HARV_BOUNTY` = `[0, 9, 0, 0, 0, 0, 1000, 3]`

> **Important**: Fertility and Intensity are intermediate scaled values (internal
> precision), NOT direct Musu/s. The bounty formula's Precision divisor (10^9)
> converts them to actual Musu. Always compute the full bounty formula for
> real-world harvest rates.

> Source: `LibHarvest.sol:156–175`

### Starve Cutoff (Bounty Cap by HP)

The raw bounty above is **capped** so a Kami can never accrue more Musu than its
**current HP** can physically sustain. This prevents a long-running harvest from
generating bounty that would have starved the Kami to death before it could be
collected.

```
Bounty = min(rawBounty, MaxMusu)
```

`MaxMusu` is the inverse of the strain formula — the largest Musu amount whose
strain damage would not exceed current HP:

```
MaxMusu = floor(HP × Divisor / (core × boost))
        = floor(HP × Precision × (Harmony + config[0]) / (core × boost))
```

Where (from `KAMI_HARV_STRAIN` = `[20, 0, 6500, 3, 0, 0, 1000, 3]`):
- **core** = `config[2]` = 6500
- **boost** = `config[6]` + `STND_STRAIN_BOOST` bonus = 1000 (+ bonus)
- **Harmony** = Kami's total HARMONY stat
- **config[0]** = 20 (denominator base — "hijacked" nudge, added to Harmony)
- **Precision** = `10^(config[3] + config[7])` = `10^(3+3)` = 10^6

If current HP ≤ 0, `MaxMusu` = 0. If `core × boost` = 0, no cap is applied.

This is the exact inverse of strain (rounded the opposite way): collecting Musu
inflicts strain damage, and the cap guarantees that damage tops out at current HP.

> Source: `LibHarvest.sol:172–174` (cap), `LibHarvest.sol:179–195`
> (`calcMaxMusu`), `LibKami.sol:155–170` (`calcStrain`), `configs.ts:136`

#### Strain (HP cost of harvesting)

Each Musu collected inflicts HP **strain** (rounded up):

```
Strain = ceil(amt × core × boost / Divisor)
       = ceil(amt × core × boost / (Precision × (Harmony + config[0])))
```

Higher **Harmony** increases the divisor, reducing strain per Musu (more Harmony
= more Musu before starving). The starve cutoff above is this equation solved
for `amt` at `Strain = HP`.

> Source: `LibKami.sol:155–170`, `configs.ts:136`

### Worked Example

A Kami with Power=10, Violence=10, neutral affinity, no bonuses, harvesting
for 1 hour (3600s), 60 minutes of intensity:

```
Fertility = 1 × 10 × 1500 × 1000 / 3600 = 4,166        (intermediate value, integer division)
Intensity = 1,000,000 × (10×5 + 60) × 10 / (480 × 3600) = 636   (intermediate)

Rate      = 4,166 + 636 = 4,802
Duration  = 3,600 seconds
Boost     = 1,000 (no bonuses)
Precision = 10^(6 + 0 + 3) = 10^9

Bounty = 4,802 × 3,600 × 1,000 / 1,000,000,000
       = 17 Musu in 1 hour (integer division)
```

For comparison — a Power=20 Kami with perfect affinity match (efficacy 2000):
```
Fertility = 1 × 20 × 1500 × 2000 / 3600 = 16,666
Bounty = 16,666 × 3,600 × 1,000 / 1,000,000,000 = 59 Musu/hr (integer division; before Intensity)
```

### Fertility (Power-Based Rate)

Steady harvest rate based on the Kami's **Power** stat:

```
Fertility = Precision × Power × ratio × Efficacy / 3600
```

Config `KAMI_HARV_FERTILITY` = `[0, 0, 1500, 3, 0, 0, 1000, 3]`

```
ratio = 1500 (precision 10^3)
Efficacy = base boost (1000) + affinity adjustments (see below)
Precision = 10^(6 - (3 + 3)) = 10^0 = 1
```

Effective: `Fertility = Power × 1500 × Efficacy / 3600`

> Source: `LibHarvest.sol:260–272`, `configs.ts:133`

### Intensity (Violence-Based Rate)

Time-ramping bonus rate based on **Violence** stat:

```
Intensity = Precision × (Violence × nudge + minutesElapsed) × boost / (ratio × 3600)
```

Config `KAMI_HARV_INTENSITY` = `[5, 0, 480, 0, 0, 0, 10, 0]`

```
nudge = 5 (multiplied by Violence stat)
minutesElapsed = floor((now - intensityResetTime) / 60)
ratio = 480 (inverted — this is the divisor)
boost = 10 + HARV_INTENSITY_BOOST bonus
```

Intensity **grows linearly** with time spent harvesting (minutes elapsed),
incentivizing longer harvest sessions but increasing strain risk.

Intensity resets when the harvest starts (`LibHarvest.sol:111`) and when an
item is used on the Kami (`KamiUseItemSystem.sol:45` → `LibKami.resetIntensity`,
`LibKami.sol:125–129`).

> Source: `LibHarvest.sol:276–290`, `configs.ts:134`

## Affinity & Efficacy System

Efficacy is a **multiplier on Fertility** determined by how well the Kami's
trait affinities match the node's affinity.

### Affinity Types

Four affinities exist: `EERIE`, `SCRAP`, `INSECT`, `NORMAL`

### Harvest Effectiveness (Trait vs Node)

| Trait Affinity | Node Affinity | Result |
|---|---|---|
| Same as node | — | **Strong** (+bonus) |
| Different (non-NORMAL) | Different (non-NORMAL) | **Weak** (−penalty) |
| `NORMAL` | Any | **Neutral** (half of `HARV_FERTILITY_BOOST` bonus, no config shift) |
| Any | `NORMAL` | **Neutral** |

> Source: `LibAffinity.sol:82–90`

### Two Affinity Checks Per Kami

Each Kami has two relevant affinities for harvesting:
1. **Body affinity** — stronger impact
2. **Hand affinity** — weaker impact

Nodes can have 1 or 2 affinities (e.g., `"EERIE"` or `"EERIE-SCRAP"`).

The system picks the most favorable matchup order when a node has dual affinities
(body gets the better match since it has higher config impact).

Config values:
- `KAMI_HARV_EFFICACY_BODY` = `[3, 0, 650, 250, 0]` → strong +650, weak −250
- `KAMI_HARV_EFFICACY_HAND` = `[3, 0, 350, 100, 0]` → strong +350, weak −100

Format: `[precision, neutral, strong, weak, special]`

The shifts are applied to the base Efficacy (config boost value, default 1000):
```
Efficacy = baseBoost + bodyShift + handShift
```

A perfectly matched Kami gets `1000 + 650 + 350 = 2000` (2x harvest).
A poorly matched Kami gets `1000 - 250 - 100 = 650` (0.65x harvest).

The `HARV_FERTILITY_BOOST` bonus enters the efficacy calc only as the
**up-shift**: it is added in full on a **Strong** matchup, at half value for a
`NORMAL` trait, and not at all on Weak or Neutral results
(`LibHarvest.sol:206–211, 252–256`, `LibAffinity.sol:49–58`).

Total efficacy is clamped at a minimum of 0 (`LibHarvest.sol:240`).

> Source: `LibHarvest.sol:199–257`, `LibAffinity.sol:34–58`, `configs.ts:129–130`

## Tax System

When starting a harvest, a **taxer** can be specified (e.g., a guild leader or
referrer). The tax rate is denominated in **basis points** (`PRECISION` = 1e4):
each recipient receives `original × rate / 10⁴` of the collected output
(`LibTax.sol:14, 92`). The rate is capped at 2000 basis points (20%) —
`LibTax.create` reverts with `"LibTax: cannot be more than 20%"` above that
(`LibTax.sol:36`).

On collection:
1. Tax bill is calculated via `LibTax.getBillFor(balance, harvestID)`
2. Tax recipients receive their share of the harvested items
3. The remainder goes to the Kami's account

Taxes are removed and re-set each time a harvest starts.

> Source: `LibHarvest.sol:83–101`, `LibTax.sol:14, 36, 85–95`

## Liquidation (PvP)

One harvesting Kami can raid another's active harvest if they share the same node.

### Requirements

1. Both Kamis must be `HARVESTING` on the **same node**
2. The victim's harvest must be `ACTIVE` (`HarvestLiquidateSystem.sol:33`)
3. The killer's account must be in the node's room (`LibRoom.sharesRoom`,
   `HarvestLiquidateSystem.sol:43`)
4. Killer must pass `LibKill.isLiquidatableBy()` — violence threshold check
5. Killer must be healthy and off cooldown

### Process

1. Sync both Kamis' health
2. **Salvage** — victim retains a portion: `salvage = calcSalvage(victimID, bounty)`.
   It is paid as MUSU to the victim's account inventory, and the victim Kami
   gains XP equal to the salvage (`LibKill.sendSalvage`, `LibKill.sol:51–56`)
3. **Spoils** — killer steals a portion: `spoils = calcSpoils(killerID, bounty - salvage)`.
   Spoils are credited to the **killer's own harvest balance**
   (`LibKill.sendSpoils` → `LibHarvest.incBounty`, `LibKill.sol:59–62`), not to
   inventory — they must still be collected and then pass through the killer's
   own tax/XP path like any other bounty
4. **Recoil** — killer takes HP damage:
   `strain = LibKami.calcStrain(killerID, spoils)` — the killer's would-be
   strain on the stolen spoils — then
   `recoil = LibKill.calcRecoil(victimID, killerID, strain)`
   (`HarvestLiquidateSystem.sol:62–63`); karma is computed inside `calcRecoil`
   via `calcKarma` (`LibKill.sol:202–220`; `calcKarma` at `LibKill.sol:165–179`)
5. **Killer reward** — the killer's account receives 1 Obol
   (`LibKill.rewardKiller`, `LibKill.sol:65–68`, called at
   `HarvestLiquidateSystem.sol:78`)
6. Killer's cooldown resets (consumes `UPON_COOLDOWN_SET` bonuses, e.g. Energy Drink), and `UPON_LIQUIDATION` bonuses reset
7. **Victim dies** — state → `DEAD`, HP → 0, harvest stops, all bonuses reset
8. **Score** — the killer's account `LIQUIDATE` score increments by 1
   (`HarvestLiquidateSystem.sol:90`)

The victim's **unsalvaged, unspoiled bounty is destroyed** (not transferred).

> Source: `HarvestLiquidateSystem.sol:24–94`, `LibKill.sol:51–68`

> The detailed liquidation formulas (salvage, spoils, karma, recoil) are
> documented in `mechanics/combat/kill.md`.

## Harvest Entity Shape

Each harvest is an ECS entity with:

| Component | Value |
|---|---|
| `EntityType` | `"HARVEST"` |
| `IdHolder` | Kami entity ID |
| `IdSource` | Node entity ID |
| `State` | `"ACTIVE"` or `"INACTIVE"` |
| `TimeStart` | Timestamp when harvest started |
| `TimeLast` | Timestamp of last sync |
| `TimeReset` | Timestamp of last intensity reset |
| `Value` | Accrued but unclaimed bounty |

Entity ID: `keccak256("harvest", kamiID)` — one harvest per Kami.

> Source: `LibHarvest.sol:70–79, 108–113, 436–438`

## Side Effects on Collection

When bounty is collected (via Collect or Stop):

1. **XP** — Kami gains XP equal to the **post-tax output** (`LibHarvest.claim`
   returns the after-tax remainder, `LibHarvest.sol:83–101`;
   `LibExperience.inc` at `HarvestCollectSystem.sol:87` /
   `HarvestStopSystem.sol:97`)
2. **Scavenge** — the post-tax output is added to the account's scavenge-bar
   points for the node (`LibNode.scavenge` → `LibScavenge.incFor`,
   `LibNode.sol:125–133`, `LibScavenge.sol:87–96`). No droptable roll happens
   here — rewards are rolled in a separate `ScavengeClaimSystem` transaction,
   which extracts whole tiers and carries the remainder over
   (`LibScavenge.sol:99–114`). See `mechanics/world/scavenging.md`
3. **Score** — the account's leaderboard score is incremented by the
   post-tax output (`LibScore.incFor`, `HarvestCollectSystem.sol:100` /
   `HarvestStopSystem.sol:110`). The score is **epoch-scoped** — keyed by
   the current `SCORE_EPOCH` config value, the MUSU item index and the type
   string `COLLECT` (`LibScore.sol:71–80, 145–147`) — so it is a resettable
   leaderboard (an admin can start a new epoch), not a lifetime counter

> Source: `HarvestStopSystem.sol:97–110`, `HarvestCollectSystem.sol:87–100`
