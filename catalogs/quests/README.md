# Quest Catalog

Complete quest data extracted from the Kamigotchi source code.

Source: `packages/contracts/deployment/world/data/quests/`

See `mechanics/progression/quests.md` for how the quest system works
(acceptance, tracking, completion, reward distribution).

---

## Summary

**195 quests total** across 5 categories:

| Category | Key Prefix | Index Range | Count | Primary Giver |
|----------|-----------|-------------|-------|---------------|
| Main Story | MSQ | 1-109 | 109 | MENU (65), MINA (41), DIMIDIATUS (3) |
| Mina Line | MIN | 2001-2016, 2100-2101 | 18 | MINA (18) |
| Side Quests | SQ | 3001-3998 | 60 | MENU (27), MINA (18), ROB (9), ZEVANA (5), DIMIDIATUS (1) |
| Event/Misc | SQ999/SQ997/TTX01 | 10001-10003 | 3 | MENU / unset |
| Test | test-* | 1000000-1000004 | 5 | unset |

> Note: MSQ quests and MIN001-MIN016 have Type=MAIN in the data; MIN100-MIN101
> are Type=SIDE. The MIN quests form a distinct parallel storyline given
> exclusively by Mina, focused on crafting education and the Elders faction.
> They are separated here for clarity. Combined MAIN type count: 128. Combined
> SIDE type count: 60 (58 SQ + MIN100-MIN101). Remaining 6 have no type set
> (event + test quests).

### By Status

| Status | Count |
|--------|-------|
| In Game | 187 |
| Defunct | 3 |
| Test | 5 |

**`Defunct`** is a retirement marker, not a deployment state. The deployment
pipeline recognises `To Deploy`, `In Game`, `Test`, `To Update` /
`Revise Deployment` (revise) and `To Remove` (delete); `Defunct` matches none
of them, so a `Defunct` row is never created, revised or deleted by a bulk
run. Removing an already-deployed quest requires calling the delete path with
an explicit index.

> ⚠️ UNCERTAIN: because no bulk run acts on the marker, whether SQ802 and
> SQ999 have actually been deleted from the live world — or merely flagged as
> retired in the sheet — is chain state and cannot be read from source.

> Source: `deployment/world/state/quests/quests.ts:52–54, 80–101`,
> `deployment/world/state/utils.ts:40–49`

### By Giver

| Giver | Count |
|-------|-------|
| MENU | 93 |
| MINA | 77 |
| ROB | 9 |
| ZEVANA | 5 |
| DIMIDIATUS | 4 |
| (unset) | 7 |

### Quest Features

- **1 daily quest** (test-0, test only)
- **1 time-gated quest** (SQ999, before 25/10/25 0 GMT — now `Defunct`)
- **3 retired quests** (SQ802, SQ999, TTX01 — all `Defunct`)
- **1 zone-unlocking quest** (MSQ035 rewards the `FLAG_CAVES_UNLOCKED` flag)
- **6 cross-storyline gates** where the MSQ and MIN lines depend on each other

---

## Files

| File | Rows | Description |
|------|------|-------------|
| `quests.csv` | 195 quests | Full quest definitions: key, index, status, title, type, giver, dialogues, requirements, objectives, rewards |
| `objectives.csv` | 202 objectives | Objective definitions: description, operator, delta type, tracking type, index, value |
| `requirements.csv` | 201 requirements | Prerequisite definitions: quest completions, item ownership, room presence, time windows |
| `rewards.csv` | 68 rewards | Reward definitions: items, reputation, flags |
| `quest-lines.md` | — | Quest chain map showing all prerequisite links, storyline branches, and convergence points |
| `dialogues/` | — | NPC dialogue data (see below) |

### CSV Column Reference

**quests.csv**:
`Key, Index, Status, Title, Daily, Type, Giver, Introduction Dialogue, Resolution Dialogue, Requirements, Objectives, Rewards`

- `Key`: Human-readable identifier (MSQ001, MIN003, SQ015, test-0)
- `Index`: Numeric ID used on-chain (1-109, 2001-2016, 3001-3998, 10002-10003, 1000000-1000004)
- `Status`: Deployment state (In Game, To Deploy, To Update Text, To Update, Test) or the retirement marker `Defunct`
- `Daily`: Yes/No — whether the quest is repeatable daily
- `Giver`: The NPC or system that presents the quest (MENU, MINA, DIMIDIATUS, ROB, ZEVANA)
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
4. **Act IV — Temple of the Wheel** (MSQ105-MSQ109): Now **In Game**.
   Introduces Dimidiatus as quest giver. Titles: "The Turning of the Wheel",
   "Two Faces Under One Hood", "Get a Foot In The Door", "Treat Yourself",
   "Remain Unburdened of Attachments"

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
- **Annfwn / Rob quests** (SQ017-SQ022): Other-world treasure room exploration,
  introduces Rob as quest giver
- **Adoption / trading line** (SQ028-SQ045): ZEVANA's Kami Adoption Agency
  errands, trading chains ("Trading Lunches", "Container Deposit"), Rookie
  Training, and the "Resonant" cave-bell quests
- **Spirit / Ring of Spirits line** (SQ100-SQ118): speaking with lost souls —
  "Get the Story Straight I-III", "Call Your Grandparents", "Look Who's
  Talking", "My Ears Are Burning". Tied to the **Ring of Spirits** key item
  (22802). SQ113-SQ118 ("Airing it Out", "Conditioned Environment", "Dry
  Conversation", "Lost and Found", "Trash Pickers", "Janitorial Supplies") are
  **To Deploy**
- **Diagnostics** (SQ802-SQ803): "Quest Diagnostics" (now `Defunct`),
  "Never Brought to Mind"
- **Special** (SQ998, SQ997, SQ999): Conditional/event quests. SQ999 is now
  `Defunct`
- **Retired stub** (TTX01, index 10001): "Proof of Honesty" — carries only the
  `Missed Mina` requirement (`MINA_LAUNCH_VICTIM`); no type, giver, dialogue,
  objectives or rewards, and `Defunct` from the row's first appearance

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
