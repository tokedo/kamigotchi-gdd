# Skill Catalog Data

71 skills organized into 4 skill trees with 6 tiers each, plus 16 effect types.

> Source: `packages/contracts/deployment/world/data/skills/`

## Files

- `skills.csv` — 71 skill definitions
- `effects.csv` — 16 effect type definitions

## Column Reference

### skills.csv

| Column | Description |
|---|---|
| Index | Unique skill index (e.g., 111, 121) |
| Name | Display name |
| Tree | Skill tree: Predator, Enlightened, Guardian, Harvester |
| Tier | Tier level (1-6) |
| Tree req | Tree points needed to unlock this tier (0, 5, 15, 25, 40, 55) |
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
| Harvester | 17 | Economy — harvest power, fertility, bounty, intensity |

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
