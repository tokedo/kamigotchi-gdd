# Skill Catalog Data

72 skills (18 per tree: 3 per tier × 6 tiers) organized into 4 skill trees,
plus 16 effect types.

> Source: `packages/contracts/deployment/world/data/skills/`

## Files

- `skills.csv` — 72 skill definitions
- `effects.csv` — 16 effect type definitions

> ⚠️ **Stale column warning**: the `Tree req` column in `skills.csv` carries
> outdated display values for tiers 4–6 (`20/30/40`) — this matches the game
> repo's own CSV, but on-chain enforcement reads the `KAMI_TREE_REQ` config
> `[0, 5, 15, 25, 40, 55, ...]` (configs.ts, checked in `LibSkill.sol`).
> The real gates are **25/40/55** for tiers 4/5/6 (player-verified: the tier-6
> ultimate needs 55 tree points → level 56 for a mono-tree build). Use the
> table below, not the CSV column.

## Column Reference

### skills.csv

| Column | Description |
|---|---|
| Index | Unique skill index (e.g., 111, 121) |
| Name | Display name |
| Tree | Skill tree: Predator, Enlightened, Guardian, Harvester |
| Tier | Tier level (1-6) |
| Tree req | Tree points to unlock this tier **as shipped in the source CSV** (0, 5, 15, 20, 30, 40) — stale for tiers 4–6; see warning above and use the Tier Unlock table instead |
| Max | Maximum ranks purchasable |
| Cost | Skill points per rank |
| Effect | Effect key (e.g., SVS, HFB) — see effects.csv |
| Value | Effect magnitude per rank |
| Units | Unit type (Stat, Percent, MUSU/hr, Seconds) |
| Exclusion | Mutually exclusive skill (if any) |
| Description | Flavor text |

### effects.csv

| Column | Description |
|---|---|
| Context | Effect category: Stat, Harvest, Attack, Defense, Resting, Standard |
| Key | Short key used in skills.csv Effect column |
| Name | Full effect name |
| Type | System type (STAT, HARV, ATK, DEF, REST, STND) |
| AsphoAST | Target parameter (Health, Power, Fertility, etc.) |
| Operation | How effect applies: Shift (additive) or Boost (multiplicative) or Ratio |
| Units | Display units |
| Baseline | Default value before skill |
| Precision | Decimal precision (0 or 3) |

## Skill Trees

| Tree | Skills | Focus |
|---|---|---|
| Predator | 18 | Combat offense — violence, attack threshold, spoils, cooldown |
| Enlightened | 18 | Sustain — resting recovery, strain reduction, cooldown |
| Guardian | 18 | Defense — harmony, health, defense threshold, salvage, strain reduction |
| Harvester | 18 | Economy — harvest power, fertility, bounty, intensity |

## Tier Unlock Requirements

| Tier | Tree Points Required |
|---|---|
| 1 | 0 (unlocked from start) |
| 2 | 5 |
| 3 | 15 |
| 4 | 25 |
| 5 | 40 |
| 6 | 55 |

See `mechanics/progression/skills.md` for skill system details, point allocation,
and effect formulas.
