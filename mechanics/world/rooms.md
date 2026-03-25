# Rooms & Exits

> Source: `packages/contracts/src/libraries/LibRoom.sol` (L1–281),
> `packages/contracts/src/systems/AccountMoveSystem.sol` (L1–50),
> `packages/contracts/deployment/world/data/rooms/rooms.csv`

## Overview

Rooms are the primary spatial units of the game world. Each room has a 3D
coordinate, a name, description, and optionally special exits to non-adjacent
rooms. Rooms can have **gates** — conditional requirements that restrict access.
Players move between rooms by spending stamina.

See `catalogs/rooms/rooms.csv` for the full room catalog.

## Room Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"ROOM"` |
| `IndexRoom` | Unique room index (uint32) |
| `Location` | 3D coordinate `(x, y, z)` |
| `Name` | Display name |
| `Description` | Flavor text |
| `Exits` | (optional) Array of special exit room indices |
| Flags | (optional) Room-specific flags |

Entity ID: `keccak256("room", roomIndex)`

> Source: `LibRoom.sol:34–48, 266–268`

## Movement Rules

### Adjacency

Two rooms are **adjacent** if they share the same `z` plane and differ by
exactly 1 in either `x` or `y` (but not both — no diagonal movement):

```
isAdjacent = (a.z == b.z) && (
  (a.x == b.x && |a.y - b.y| == 1) ||
  (a.y == b.y && |a.x - b.x| == 1)
)
```

Adjacent rooms can always be reached (subject to gates).

> Source: `LibRoom.sol:134–141`

### Special Exits

Rooms can define **special exits** — non-adjacent rooms that are directly
reachable (e.g., portals, doors, tunnels). These are stored in the `Exits`
component as an array of room indices.

A room is reachable if it is **adjacent OR listed as a special exit**.

> Source: `LibRoom.sol:103–118, 162–168`

### Z-Plane

The `z` coordinate acts as a layer/floor system. Rooms on different `z` planes
are never adjacent — they can only be connected via special exits. This enables
multi-level areas (overworld z=1, caves z=3, interiors z=2, etc.).

From the room data:
- z=1: Overworld (forests, scrapyard, paths)
- z=2: Interiors (convenience store, burning room, plane interior)
- z=3: Underground caves
- z=4: Special areas (treasure hoard, castle)

## Gates (Room Access Conditions)

Gates are conditional requirements that must be met to enter a room. Gates
can be:

- **Generic**: apply to all incoming movement to the room
- **Source-specific**: only apply when coming from a specific room

Gate entity:

| Component | Description |
|---|---|
| `IDTo` | Pointer to destination room: `keccak256("room.gate.to", roomIndex)` |
| `IDFrom` | Pointer to source room: `keccak256("room.gate.from", sourceIndex)` or 0 (generic) |
| `IdSource` | Source room index (for client display) |
| Conditional data | Standard `LibConditional` condition components |

Gate checking:
```
conditions = queryGates(fromIndex, toIndex)  // combines generic + source-specific
accessible = LibConditional.check(conditions, accountID)
```

> Source: `LibRoom.sol:51–68, 121–131, 211–246`

## Room Sharing

Many systems check whether two entities are in the same room:

```
sharesRoom = roomComponent.get(entityA) == roomComponent.get(entityB)
```

Used by: NPC shops (player must be in NPC's room), harvesting liquidation
(attacker must be in correct room), etc.

> Source: `LibRoom.sol:97–99, 144–147`

## World Data

The current world has **70 rooms** across 4 z-planes, including:
- Overworld areas: Misty Riverside, Torii Gate, Scrapyard, Forest paths
- Interiors: Convenience Store, Plane Interior, Burning Room
- Caves: Temple Cave, Cave Crossroads, Fungus Garden, Sacrarium
- Special: Marketplace (room 66 — the Trade Room), Treasure Hoard (room 88)

Notable rooms:
- **Room 1** (Misty Riverside) — starting room for all new accounts
- **Room 11** (Temple by the Waterfall) — location for first Kami naming
- **Room 66** (Marketplace) — trade room (delivery fee waived)

> Source: `data/rooms/rooms.csv`
