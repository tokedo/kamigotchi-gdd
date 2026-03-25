# Quest Catalog

Complete quest data extracted from the Kamigotchi source code.

Source: `packages/contracts/deployment/world/data/quests/`

See `mechanics/progression/quests.md` for how the quest system works
(acceptance, tracking, completion, reward distribution).

---

## Summary

**155 quests total** across 4 categories:

| Category | Key Prefix | Index Range | Count | Primary Giver |
|----------|-----------|-------------|-------|---------------|
| Main Story | MSQ | 1-109 | 109 | MENU (52), MINA (54), DIMIDIATUS (3) |
| Mina Line | MIN | 2001-2016 | 16 | MINA |
| Side Quests | SQ | 3001-3998 | 24 | MENU (14), MINA (8), ROB (2) |
| Event/Misc | SQ999/SQ997 | 10002-10003 | 2 | MENU / unset |
| Test | test-* | 1000000-1000004 | 5 | unset |

> Note: Both MSQ and MIN quests have Type=MAIN in the data. The MIN quests
> form a distinct parallel storyline given exclusively by Mina, focused on
> crafting education and the Elders faction. They are separated here for
> clarity. Combined MAIN type count: 125. Combined SIDE type count: 24.
> Remaining 6 have no type set (event + test quests).

### By Status

| Status | Count |
|--------|-------|
| In Game | 139 |
| To Deploy | 5 |
| To Update Text | 4 |
| To Update | 2 |
| Test | 5 |

### By Giver

| Giver | Count |
|-------|-------|
| MENU | 79 |
| MINA | 65 |
| DIMIDIATUS | 3 |
| ROB | 2 |
| (unset) | 6 |

### Quest Features

- **1 daily quest** (test-0, test only)
- **1 time-gated quest** (SQ999, before 25/10/25 0 GMT)
- **1 zone-unlocking quest** (MSQ035 rewards the `FLAG_CAVES_UNLOCKED` flag)
- **6 cross-storyline gates** where the MSQ and MIN lines depend on each other

---

## Files

| File | Rows | Description |
|------|------|-------------|
| `quests.csv` | 155 quests | Full quest definitions: key, index, status, title, type, giver, dialogues, requirements, objectives, rewards |
| `objectives.csv` | 167 objectives | Objective definitions: description, operator, delta type, tracking type, index, value |
| `requirements.csv` | 159 requirements | Prerequisite definitions: quest completions, item ownership, room presence, time windows |
| `rewards.csv` | 55 rewards | Reward definitions: items, reputation, flags |
| `quest-lines.md` | — | Quest chain map showing all prerequisite links, storyline branches, and convergence points |
| `dialogues/` | — | NPC dialogue data (see below) |

### CSV Column Reference

**quests.csv**:
`Key, Index, Status, Title, Daily, Type, Giver, Introduction Dialogue, Resolution Dialogue, Requirements, Objectives, Rewards`

- `Key`: Human-readable identifier (MSQ001, MIN003, SQ015, test-0)
- `Index`: Numeric ID used on-chain (1-109, 2001-2016, 3001-3998, 10002-10003, 1000000-1000004)
- `Status`: Deployment state (In Game, To Deploy, To Update Text, To Update, Test)
- `Daily`: Yes/No — whether the quest is repeatable daily
- `Giver`: The NPC or system that presents the quest (MENU, MINA, DIMIDIATUS, ROB)
- `Introduction Dialogue` / `Resolution Dialogue`: Full NPC dialogue text with speaker tags
- `Requirements`: Comma-separated prerequisite IDs referencing `requirements.csv` entries
- `Objectives`: Comma-separated objective descriptions referencing `objectives.csv` entries
- `Rewards`: Comma-separated reward descriptions referencing `rewards.csv` entries

**objectives.csv**:
`(empty), Description, Operator, DeltaType, Type, Index, Value`

Tracking types include: `HARVEST_TIME`, `DROPTABLE_ITEM_TOTAL`, `ITEM_BURN`,
`ITEM_TOTAL`, `MOVE`, `ROOM`, `SCAV_CLAIM_NODE`, `CRAFT_ITEM`,
`LISTING_BUY_TOTAL`, `KAMI_*`, `SKILL_POINTS_USE`, `TRADE_EXECUTE`, `PHASE`,
`LIQUIDATE_TOTAL`, `LIQUIDATED_VICTIM`, and affinity-scoped scavenging
(`SCAV_CLAIM_AFFINITY_*`).

**requirements.csv**:
`(empty), Description, Operator, Type, Index, Value`

Requirement types: `QUEST` (quest completion), `ITEM` (item ownership),
`ROOM` (location check), `BLOCKTIME` (time window), `MINA_LAUNCH_VICTIM`
(event-specific), `GOAL` (goal completion), `LIQUIDATED_VICTIM`.

**rewards.csv**:
`(empty), Description, Type, Index, Value`

Reward types: `ITEM` (grant items/currency), `REPUTATION` (faction rep),
`FLAG_CAVES_UNLOCKED` (zone unlock).

---

## Quest Lines Overview

See `quest-lines.md` for the complete chain map with ASCII graphs.

### Main Story Arc

The main story (MSQ) progresses through four acts:

1. **Act I — Tutorial & Surface** (MSQ001-MSQ020): Core mechanic tutorials,
   material identification, world exploration, harvesting data
2. **Act II — Investigation & Convergence** (MSQ021-MSQ036): The "Squaring
   the Circle" investigation series, convergence with Mina's line, unlocking
   the Sanctuary Caves
3. **Act III — Sanctuary Caves** (MSQ037-MSQ104): Deep cave exploration with
   parallel branches exploring different cave areas, the Crystal Set sub-arc,
   Dowsing Rod discovery quests, and the "Ordinary Intermediate Potion
   Crafting" series
4. **Act IV — Temple of the Wheel** (MSQ105-MSQ109): Not yet deployed.
   Introduces Dimidiatus as quest giver

### Mina's Parallel Line

The MIN line (MIN001-MIN016) runs parallel to the main story, teaching
crafting fundamentals and building Elders faction reputation. It gates the
main story at two critical junctures:

- MIN013 is required for MSQ021
- MIN015 is required for MSQ031

### Side Quest Groups

- **Tutorial sides** (SQ001-SQ006): Kami management tutorials
- **Exploration** (SQ007-SQ008): Movement milestones
- **Crafting education** (SQ010-SQ014): Advanced hex/potion crafting
- **Economy** (SQ009, SQ012-SQ013): Spending at Mina's shop
- **Obols chain** (SQ015-SQ016): Mystery currency investigation
- **Annfwn quests** (SQ017-SQ022): Other-world treasure room exploration,
  introduces Rob as quest giver
- **Special** (SQ998, SQ997, SQ999): Conditional/event quests

---

## Dialogues

NPC dialogue data is stored in `dialogues/`:

| File | Description |
|------|-------------|
| `npcs.csv` | NPC roster: 9 NPCs with index, name, default image, room, text color, ritual role |
| `npcDialogues.csv` | Full dialogue trees: 106 lines across 5 NPCs with mood images, branching choices |

### NPCs with Dialogues

| Index | Name | Room | Role |
|-------|------|------|------|
| 0 | Menu | (system) | Quest guide, main narrator |
| 1 | Mina | 13 (Convenience Store) | Shop 1, crafting teacher |
| 2 | Dimidiatus | 19 (Temple of the Wheel) | Kami Sacrifice ritual |
| 3 | Zevana | 3 (Torii Gate) | Kami Adoption Agency |
| 4 | Vending Machine | 18 (Cave Crossroads) | Shop 2 |
| 5 | Rob | 5 (Restricted Area) | Treasure room NPC |
| 6 | Nurse Why | — | (no dialogue data yet) |
| 7 | Dolores | — | (no dialogue data yet) |
| 8 | Artie Zbirak | — | Equipment/Gear Shop (no dialogue data yet) |

Dialogue trees use a choice-based branching system. Each NPC has a main
conversation loop with topic choices that return to the menu. See
`npcDialogues.csv` for the full dialogue graph with next-dialogue pointers.

Note: Quest-specific dialogues (introduction and resolution) are embedded
directly in `quests.csv`, not in the dialogue files. The `dialogues/` files
contain ambient NPC conversations triggered by visiting their rooms.
