# Skill Catalog Data

72 skills (18 per tree: 3 per tier × 6 tiers) organized into 4 skill trees,
plus 16 effect types.

> Source: `packages/contracts/deployment/world/data/skills/`

## Files

- `skills.csv` — 72 skill definitions
- `effects.csv` — 16 effect type definitions

> **Deliberate divergence from source — `Tree req` column.** This is the one
> column in `skills.csv` that is **not** a verbatim copy of the source CSV.
> The source file still carries the pre-retune display values `20/30/40` for
> tiers 4–6; the values enforced on chain are `25/40/55`, from the
> `KAMI_TREE_REQ` config `[0, 5, 15, 25, 40, 55, 75, 95]`
> (`deployment/world/state/configs/configs.ts:185–187`), read by
> `LibSkill.getTreeTierPoints` (`LibSkill.sol:226–228`) and checked in
> `LibSkill.meetsTreePrerequisites` (`LibSkill.sol:165–180`).
>
> The column is safe to correct because **the deployment script never reads
> it** — `initSkill` takes only `Index`, `Name`, `Description`, `Tree`,
> `Cost`, `Max` and `Tier` (`deployment/world/state/skills.ts:62–86`), passing
> `tier - 1` (line 84) as the config index. `Tree req` is display data in the
> source sheet and nothing else. This catalog therefore carries the enforced
> values so that catalog and mechanic agree; every other column remains
> byte-identical to source.
>
> Tier 6 needing 55 tree points means a mono-tree build reaches its ultimate
> at 56 skill points invested in that tree.

## Column Reference

### skills.csv

| Column | Description |
|---|---|
| Index | Unique skill index (e.g., 111, 121) |
| Name | Display name |
| Tree | Skill tree: Predator, Enlightened, Guardian, Harvester |
| Tier | Tier level (1-6) |
| Tree req | Tree points to unlock this tier, **as enforced on chain** (0, 5, 15, 25, 40, 55) — corrected against `KAMI_TREE_REQ`; the source CSV's own column shows 20/30/40 for tiers 4–6 and is unread by deployment (see note above) |
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
