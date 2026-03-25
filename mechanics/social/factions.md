# Factions

> Source: `packages/contracts/src/libraries/LibFaction.sol` (L1–117),
> `packages/contracts/deployment/world/data/factions/factions.csv`

## Overview

Factions are named groups that entities (primarily NPCs, but potentially
players) can belong to. Players build **reputation** with factions through
quest rewards and other systems. Reputation is tracked via `LibScore`,
enabling leaderboard integration.

See `catalogs/factions/factions.csv` for the full faction catalog.

## Faction Entity Shape (Registry)

| Component | Description |
|---|---|
| `EntityType` | `"FACTION"` |
| `IsRegistry` | Marks as registry entry |
| `IndexFaction` | Unique faction index (uint32) |
| `Name` | Display name |
| `Description` | Faction description |
| `MediaURI` | Faction image/icon |

Faction ID: `keccak256("faction", factionIndex)`

Only one faction entity exists per index per world.

> Source: `LibFaction.sol:39–55`

## Faction Assignment

Entities (typically NPCs) can be assigned to a faction:

```solidity
LibFaction.assign(components, targetID, factionIndex)
```

This sets the `IndexFaction` component on the target entity. Currently used
for NPCs, but the code notes it could be extended to any entity.

> Source: `LibFaction.sol:72–74`

## Reputation System

Faction reputation is tracked per-account per-faction using `LibScore`:

### Incrementing Reputation

```solidity
LibFaction.incRep(components, targetID, factionIndex, amount)
```

Reputation ID: `keccak256("faction.reputation", holderID, factionIndex)`

This calls `LibScore.incFor` with the faction ID as the type, which:
- Creates the score entity if needed
- Increments individual reputation
- Increments total reputation for the faction

### Decrementing Reputation

```solidity
LibFaction.decRep(components, targetID, factionIndex, amount)
```

Same mechanism in reverse.

### Reading Reputation

```solidity
LibFaction.getRep(components, targetID, factionIndex) → uint256
```

Returns 0 if no reputation exists.

> Source: `LibFaction.sol:76–98`

## Current Factions

| Index | Name | Key | Description |
|---|---|---|---|
| 1 | The Agency | Agency | Relationship with the world administrators (the Menu) |
| 2 | The Elders | Mina | Relationship with Mina and her business/investors |
| 3 | The Nursery | Nursery | Relationship with the Nursery and its mysterious forces |

Reputation is gained primarily through quest rewards:
- **Agency Reputation**: Awarded for completing main story quests (2/4/6 per quest)
- **Elders Loyalty**: Awarded for completing Mina's quests (2/4/6 per quest)
- **Nursery Dedication**: Awarded for specific quests (4 per quest)

> Source: `data/factions/factions.csv`, `data/quests/rewards.csv`

## Reputation as Leaderboard

Since reputation uses `LibScore`, faction reputation is automatically
leaderboard-compatible. The `IdHolder` component enables reverse-mapping for
front-end leaderboard queries, and the `IDType` component groups all
reputation entries for a given faction.

> Source: `LibFaction.sol:18–34` (comment block)
