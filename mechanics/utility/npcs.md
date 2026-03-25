# NPC System

> Source: `packages/contracts/src/libraries/LibNPC.sol` (L1–94)

## Overview

NPCs are room-locked entities used as merchants, quest givers, and relationship
anchors. Each NPC has an index, a name, and a room assignment. NPCs with room
index 0 are **global** (accessible from any room).

## NPC Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"NPC"` |
| `IndexNPC` | Unique NPC index |
| `Name` | NPC display name |
| `IndexRoom` | Room where the NPC is located (0 = global) |

NPC ID: `keccak256("NPC", index)`

> Source: `LibNPC.sol:19–31, 91–93`

## Room Verification

Before interacting with an NPC, the system verifies the player is in the same
room:

```
sharesRoomWith(npcID, accountID):
  npcRoom = IndexRoom[npcID]
  return npcRoom == 0 || npcRoom == IndexRoom[accountID]
```

NPCs with `roomIndex == 0` are always accessible regardless of the player's
location.

> Source: `LibNPC.sol:41–50`

## Usage

NPCs are referenced by:
- **NPC shops** (listing/buying) — NPC is the merchant
- **Relationships** — NPC relationship flags per player
- **Quest givers** — NPCs that offer quests (tracked via quest requirements)

> Source: `LibNPC.sol:36–38`
