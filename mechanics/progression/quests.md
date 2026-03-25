# Quests

> Source: `packages/contracts/src/libraries/LibQuest.sol` (L1–407),
> `packages/contracts/src/libraries/LibQuestRegistry.sol` (L1–179),
> `packages/contracts/src/systems/QuestAcceptSystem.sol` (L1–45),
> `packages/contracts/src/systems/QuestCompleteSystem.sol` (L1–39),
> `packages/contracts/src/systems/QuestDropSystem.sol` (L1–35),
> `packages/contracts/deployment/world/data/quests/`

## Overview

Quests are structured tasks with **requirements** (conditions to accept),
**objectives** (conditions to complete), and **rewards** (items, reputation,
flags distributed on completion). Quests use the registry-instance pattern: the
registry defines the quest template, and instances track per-account progress.

The quest system supports both one-time and **repeatable** quests (time-gated
re-acceptance). Objectives use a **snapshot-based** tracking system to measure
changes that occurred after quest acceptance.

See `catalogs/quests/` for the full quest data files.

## Registry Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"QUEST"` |
| `IsRegistry` | Marks as registry entry |
| `IndexQuest` | Unique quest index (uint32) |
| `Name` | Display name |
| `Description` | Quest description text |
| `DescriptionAlt` | Resolution/end text |
| `Type` | Quest type: `"MAIN"`, `"SIDE"`, `"FACTION"`, etc. |
| `Subtype` | Quest giver: `"MENU"`, `"MINA"`, etc. |
| `Disabled` | Initially `true` — must be enabled to accept |
| `Flag REPEATABLE` | (optional) Marks quest as repeatable |
| `Time` | (optional) Cooldown duration for repeatable quests |

Registry ID: `keccak256("registry.quest", questIndex)`

> Source: `LibQuestRegistry.sol:36–58`

## Instance Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"QUEST"` |
| `IDOwnsQuest` | Account ID this quest is assigned to |
| `IndexQuest` | Quest index reference |
| `TimeStart` | Timestamp when quest was accepted |
| `TimeLast` | (set on completion) Timestamp when quest was completed |
| `IsComplete` | (set on completion) Marks quest as done |

Instance ID: `keccak256("quest.instance", questIndex, accID)`

> Source: `LibQuest.sol:53–66, 388–389`

## Accepting Quests

`QuestAcceptSystem.execute(questIndex)`:

1. Verify quest exists and is enabled
2. Verify account meets all requirements (via `LibConditional`)
3. For repeatable quests:
   - If previously completed, verify cooldown has elapsed:
     `block.timestamp > timeStart + duration`
   - Overwrite existing instance (clear completion, re-snapshot objectives)
4. For non-repeatable quests:
   - Revert if already accepted (even if not completed)
5. Create quest instance with `TimeStart = block.timestamp`
6. Snapshot all INCREASE/DECREASE objectives (record baseline values)

> Source: `QuestAcceptSystem.sol:16–39`, `LibQuest.sol:53–86`

## Objective System

Objectives are conditions that must be met to complete a quest. Each objective
uses `LibConditional` with one of four **handler types**:

### Handler Types

| Handler | Description | Snapshot? |
|---|---|---|
| `CURRENT` | Value must currently meet the condition | No |
| `INCREASE` | Value must have increased by the required amount since acceptance | Yes |
| `DECREASE` | Value must have decreased by the required amount since acceptance | Yes |
| `BOOLEAN` | Condition must currently be true (e.g., in a specific room) | No |

### Snapshot Mechanism

For INCREASE and DECREASE handlers, the system records the account's current
value for the tracked data key at the moment of quest acceptance. On
completion, the delta is calculated:

```
INCREASE: (currentValue - snapshotValue) >= requiredValue
DECREASE: (snapshotValue - currentValue) >= requiredValue
```

Snapshot entity ID: `keccak256("quest.objective.snapshot", questID, logicType, type, index)`
Snapshot anchor: `keccak256("snapshot.anchor", questID)`

> Source: `LibQuest.sol:106–150, 201–262`

### Objective Data Types

From the objectives CSV, quests track a wide range of actions:

| Type | Description | Examples |
|---|---|---|
| `HARVEST_TIME` | Time spent harvesting at a node | `INC, nodeIndex, seconds` |
| `DROPTABLE_ITEM_TOTAL` | Items received from scavenging | `INC, itemIndex, count` |
| `ITEM_BURN` | Items consumed/given | `INC, itemIndex, count` |
| `ITEM_TOTAL` | Items collected | `INC, itemIndex, count` |
| `ITEM_SPEND` | MUSU spent at shops | `INC, itemIndex, amount` |
| `MOVE` | Times moved between rooms | `INC, 0, count` |
| `ROOM` | Currently in a specific room | `BOOL, roomIndex` |
| `SCAV_CLAIM_NODE` | Scavenge claims at a node | `INC, nodeIndex, count` |
| `SCAV_CLAIM_AFFINITY_{X}` | Scavenge claims by affinity | `INC, 0, count` |
| `CRAFT_ITEM` | Items crafted | `INC, itemIndex, count` |
| `KAMI_LEVELS_TOTAL` | Kami levels gained | `CURR, 0, count` |
| `KAMI_NUM_OWNED` | Kamis currently owned | `CURR, 0, count` |
| `KAMI_NAME` | Kamis named | `INC, 0, count` |
| `LIQUIDATE_TOTAL` | Liquidations performed | `INC, 0, count` |
| `LIQUIDATED_VICTIM` | Times liquidated | `INC, 0, count` |
| `LISTING_BUY_TOTAL` | NPC shop purchases | `INC, 0, count` |
| `TRADE_EXECUTE` | Trades executed | `INC, 0, count` |
| `SKILL_POINTS_USE` | Skill points spent | `CURR, 0, count` |
| `PHASE` | Game phase check (e.g., Moonside) | `BOOL` |
| `KAMI_GACHA_REROLL` | Gacha rerolls | `INC, 0, count` |
| `KAMI_ITEM_USE` | Items used on Kami | `INC, 0, count` |

> Source: `data/quests/objectives.csv`

## Requirements

Requirements are pre-conditions checked when accepting a quest. They use
`LibConditional` and include:

| Type | Description | Examples |
|---|---|---|
| `QUEST` (COMPLETE) | Must have completed a specific quest | `Complete MSQ001` |
| `GOAL` (COMPLETE) | Must have completed a community goal | `Complete Goal: A Hole in Reality` |
| `ITEM` (MIN) | Must own a minimum quantity of an item | `Own at least 1 Obol` |
| `ROOM` (IS) | Must be in a specific room | `In Room: Marketplace` |
| `BLOCKTIME` (MIN/MAX) | Must be after/before a specific time | `After 12/9/24 8pm GMT` |
| `LIQUIDATED_VICTIM` (MIN) | Must have been liquidated | `At least 1 Kami liquidated` |
| `MINA_LAUNCH_VICTIM` (IS) | Special event flag | `Missed Mina` |

Most quests require completion of the previous quest in the main story chain.

> Source: `data/quests/requirements.csv`, `LibQuestRegistry.sol:80–87`

## Completing Quests

`QuestCompleteSystem.execute(questID)`:

1. Verify quest entity is valid, enabled, owned by caller, not already completed
2. Verify all objectives are met (checks current values against snapshots)
3. Mark quest as completed (`IsComplete`), set `TimeLast = block.timestamp`
4. Remove snapshotted objective data
5. Distribute rewards via `LibAllo.distribute`
6. Log `QUEST_COMPLETE` (and `QUEST_REPEATABLE_COMPLETE` if applicable)

> Source: `QuestCompleteSystem.sol:16–33`, `LibQuest.sol:88–95`

## Dropping Quests

`QuestDropSystem.execute(questID)`:

1. Verify quest is valid, enabled, owned by caller, not completed
2. Remove quest instance entity and all snapshot data
3. The player can re-accept the quest later (if non-repeatable: only if not
   previously accepted — dropping does not reset acceptance)

> ⚠️ UNCERTAIN: The drop function removes the entity entirely (`LibEntityType.remove`),
> which may allow re-acceptance of non-repeatable quests since `getAccQuestIndex`
> checks entity type existence. The `QuestAcceptSystem` checks `questID != 0`
> which would return 0 after entity removal. Needs verification.

> Source: `QuestDropSystem.sol:16–29`, `LibQuest.sol:97–104`

## Rewards

Rewards are distributed via `LibAllo` (the generic allocation system). Reward
types from the data:

| Type | Description | Examples |
|---|---|---|
| `ITEM` | Items given to account | MUSU, crafting materials, spell cards |
| `REPUTATION` | Faction reputation gained | Agency, Elders, Nursery reputation |
| `FLAG_*` | Flags set on account | `FLAG_CAVES_UNLOCKED` |

Reward anchor: `keccak256("registry.quest.reward", questIndex)`

> Source: `data/quests/rewards.csv`, `LibQuestRegistry.sol:149–155, 170–173`

## Quest Data Summary

The game has **~130+ quests** across multiple categories:

| Type | Index Range | Giver | Description |
|---|---|---|---|
| MAIN | 1–108 | MENU | Main story quests (MSQ001–MSQ108) |
| FACTION (Mina) | 2001–2016 | MINA | Mina's faction quests (MIN001–MIN016) |
| SIDE | 3001–3024+ | Various | Side quests (SQ001–SQ024+) |
| TEST | 10001–10003 | — | Test quests (TTX01–TTX03) |

Most main quests are sequential — each requires completion of the previous one.
Faction quests (MINA) are unlocked through specific main quest requirements.

> Source: `data/quests/quests.csv`

## Logging

| Data Key | Description |
|---|---|
| `QUEST_COMPLETE` | Incremented on any quest completion |
| `QUEST_REPEATABLE_COMPLETE` | Incremented only for repeatable quest completions |

> Source: `LibQuest.sol:376–383`
