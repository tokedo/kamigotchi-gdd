# Skill Tree

> Source: `packages/contracts/src/libraries/LibSkill.sol` (L1–273),
> `packages/contracts/src/libraries/LibSkillRegistry.sol` (L1–215),
> `packages/contracts/src/systems/SkillUpgradeSystem.sol` (L1–59),
> `packages/contracts/src/systems/SkillRespecSystem.sol` (L1–59),
> `packages/contracts/deployment/world/data/skills/skills.csv`,
> `packages/contracts/deployment/world/data/skills/effects.csv`

## Overview

Skills are permanent stat modifiers that players invest **skill points** into.
Each skill belongs to a **skill tree**, has a **tier** that gates access, and
grants one or more **bonuses** (via `LibBonus.incBy`). Skills use the
**registry-instance** pattern: the registry defines the skill template, and
instances track per-holder investment.

Skills can target either a **Kami** or an **Account** (determined by the
registry's `for` field). Kami must be in `RESTING` state to upgrade skills.

See `catalogs/skills/skills.csv` for the full skill catalog and
`catalogs/skills/effects.csv` for effect type definitions.

## Registry Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"SKILL"` |
| `IsRegistry` | Marks as registry entry |
| `IndexSkill` | Unique skill index (uint32) |
| `Name` | Display name |
| `Description` | Flavor text |
| `Cost` | Skill point cost per upgrade |
| `Max` | Maximum investment level |
| `MediaURI` | Skill icon/image |
| `For` | Target entity type: `"KAMI"` or `"ACCOUNT"` |
| `Type` | (optional) Skill tree name (e.g., `"Predator"`) |
| `Level` | (optional) Tier within the tree |

Registry ID: `keccak256("registry.skill", skillIndex)`

> Source: `LibSkillRegistry.sol:56–87`

## Instance Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"SKILL"` |
| `IDOwnsSkill` | Holder entity ID |
| `IndexSkill` | Skill index reference |
| `SkillPoint` | Current investment level in this skill |

Instance ID: `keccak256("skill.instance", holderID, skillIndex)`

> Source: `LibSkill.sol:36–44, 270–272`

## Skill Trees

Skills are organized into 4 trees, each with 6 tiers of 3 skills:

| Tree | Focus |
|---|---|
| **Predator** | Combat (violence, attack threshold, spoils, cooldown reduction) |
| **Enlightened** | Sustain (resting recovery, harvest bounty, strain reduction) |
| **Guardian** | Defense (harmony, defense threshold, salvage, health, harvest intensity) |
| **Harvester** | Harvest (power, harvest fertility/bounty, strain reduction, defense) |

Each tree has 18 skills (3 per tier × 6 tiers = 18, total 72 skills).

### Tier Structure

Each tier requires a minimum number of **tree points** invested in the same
tree before skills at that tier become available:

| Tier | Tree Points Required | Config |
|---|---|---|
| 0 | 0 | (no gate) |
| 1 | 5 | `KAMI_TREE_REQ[1]` |
| 2 | 15 | `KAMI_TREE_REQ[2]` |
| 3 | 25 | `KAMI_TREE_REQ[3]` |
| 4 | 40 | `KAMI_TREE_REQ[4]` |
| 5 | 55 | `KAMI_TREE_REQ[5]` |
| 6 | 75 | `KAMI_TREE_REQ[6]` |
| 7 | 95 | `KAMI_TREE_REQ[7]` |

> Source: `configs.ts:173`, `LibSkill.sol:226–228`

Tree points are tracked via bonuses with type `SKILL_TREE_{TreeName}` (e.g.,
`SKILL_TREE_Predator`). Each skill upgrade increments the tree bonus by the
skill's cost, so tree points = total skill points spent in that tree.

> Source: `LibSkillRegistry.sol:80–84, 212–214`

### Tier 3 and Tier 6 Exclusions

At tiers 3 and 6, skills have **mutual exclusions** — you can only invest in
one of the three options. The exclusion field in the CSV lists which other
skill indices are excluded when this skill is chosen.

Example: Predator Tier 3 — Warmonger (131), Vampire (132), and Bandit (133)
are mutually exclusive.

> Source: `skills.csv` (Exclusion column)

## Upgrading Skills

`SkillUpgradeSystem.execute(holderID, skillIndex)`:

1. Verify the skill registry entry exists
2. Verify entity type matches the skill's `for` field
3. If Kami: verify ownership, verify `RESTING` state, sync stats
4. Verify prerequisites:
   a. **Point balance**: holder has enough unspent skill points ≥ skill cost
   b. **Max check**: current investment in this skill < max
   c. **Tree tier**: total tree points ≥ tier requirement
   d. **Additional requirements**: pass any `LibConditional` checks
5. Deduct skill points from holder, increment skill instance level
6. Increment bonuses via `LibBonus.incBy` (applies the skill's effect)
7. Log `SKILL_POINTS_USE` for the account

> Source: `SkillUpgradeSystem.sol:22–53`, `LibSkill.sol:47–66, 127–164`

### Skill Point Cost

All current skills have `Cost = 1` (one skill point per upgrade level).

Skill points are gained from leveling up (see
[experience-leveling.md](../core-kami/experience-leveling.md)).

> Source: `skills.csv`

## Resetting Skills (Respec)

`SkillRespecSystem.execute(targetID)`:

1. Verify ownership (account or Kami)
2. Consume 1 **Respec Potion** (item index `11403`) from account inventory
3. Apply potion allocations via `LibItem.applyAllos`
4. Call `LibSkill.resetAll(targetID)`:
   a. Query all skill instances for the holder
   b. Calculate total refund: `sum(instanceLevel × skillCost)` for each skill
   c. Remove all bonuses assigned by those skill instances
   d. Delete all skill instance entities
   e. Add refunded points back to holder
5. If Kami, sync stats after reset

> Source: `SkillRespecSystem.sol:24–53`, `LibSkill.sol:69–87, 104–116`

## Skill Effects

Each skill grants a specific bonus effect type. Effects modify different game
systems:

| Key | Name | Context | Operation | Units |
|---|---|---|---|---|
| `SHS` | Stat Health Shift | Stat | Health Shift | Stat points |
| `SPS` | Stat Power Shift | Stat | Power Shift | Stat points |
| `SVS` | Stat Violence Shift | Stat | Violence Shift | Stat points |
| `SYS` | Stat Harmony Shift | Stat | Harmony Shift | Stat points |
| `HFB` | Harvest Fertility Boost | Harvest | Fertility Boost | Percent (×1000) |
| `HIB` | Harvest Intensity Boost | Harvest | Intensity Boost | MUSU/hr |
| `HBB` | Harvest Bounty Boost | Harvest | Bounty Boost | Percent (×1000) |
| `ATS` | Attack Threshold Shift | Attack | Threshold Shift | Percent (×1000) |
| `ATR` | Attack Threshold Ratio | Attack | Threshold Ratio | Percent (×1000) |
| `ASR` | Attack Spoils Ratio | Attack | Spoils Ratio | Percent (×1000) |
| `DTS` | Defense Threshold Shift | Defense | Threshold Shift | Percent (×1000) |
| `DTR` | Defense Threshold Ratio | Defense | Threshold Ratio | Percent (×1000) |
| `DSR` | Defense Salvage Ratio | Defense | Salvage Ratio | Percent (×1000) |
| `RMB` | Resting Recovery Boost | Resting | Metabolism Boost | Percent (×1000) |
| `SB` | Strain Boost | Standard | Strain Boost | Percent (×1000) |
| `CS` | Cooldown Shift | Standard | Cooldown Shift | Seconds |

Percent effects have precision 3 (multiplied by 1000 in storage). Stat effects
are integer values applied directly to the stat's Shift component.

> Source: `effects.csv`

## Logging

| Data Key | Description |
|---|---|
| `SKILL_POINTS_USE` | Incremented each time the account upgrades a skill |

> Source: `LibSkill.sol:263–265`
