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

A destination with no gates is always accessible.

> Source: `LibRoom.sol:51–68, 121–131, 211–246`

### Move Check Order and Revert Strings

`AccountMoveSystem.execute(toIndex)` (operator-signed) checks
**reachability before accessibility**:

| Order | Check | Revert |
|---|---|---|
| 1 | Destination is adjacent to the current room or one of its special exits (`LibRoom.isReachable`) | `"AccMove: unreachable room"` |
| 2 | Every gate on the destination — generic plus any specific to entering from the current room — passes (`LibRoom.isAccessible`) | `"AccMove: inaccessible room"` |
| 3 | Stamina covers the move cost (after the stamina sync) | `"Account: insufficient stamina"` |

Because reachability is checked first, a move to a room that is not adjacent
and not a special exit reverts `unreachable` **whether or not that room is
gated**; `"AccMove: inaccessible room"` only ever means "reachable from here,
but a gate fails". The system takes only a destination, so a gate can be
tested only from a room adjacent to (or exiting into) the gated room — a
multi-hop route meets each gate on the hop that enters it.

> Source: `AccountMoveSystem.sol:22–45`, `LibRoom.sol:103–131`,
> `LibAccount.sol:80–85, 101–107`

### Deployed Gates vs. Source (Rooms 19 and 59)

The checked-in gate list (`gates.ts`, the source of
[`catalogs/rooms/gates.csv`](../../catalogs/rooms/gates.csv)) gates **room 19**
(Temple of the Wheel) on `getGoalID(999)`, a goal no seed script defines.

> ⚠️ SOURCE ≠ DEPLOYED WORLD: a read of every in-game room's gates from the
> live world on 2026-08-27 found **no gate on room 19** and instead a
> `COMPLETE_COMP` gate on **room 59** (Black Pool) whose condition value is
> `getGoalID(13)` — goal 13, "Secret of the Ooze", which is contributed in
> room 19 and whose display reward reads "Fast Travel unlocked between Room 19
> and Room 59" (`goals.ts:149–159`). Rooms 19 and 59 are each other's special
> exit (`rooms.csv`); room 19 is otherwise reached from room 74, room 59 from
> room 58. The other ten gated destinations matched the checked-in list.
> Whether the room-59 gate applies to every entrance of room 59 or only to
> entry from room 19 was not determined by that read. Gates are created by
> admin transactions and `gates.ts` is labelled a placeholder, so the deployed
> gate set is chain state and must be read from the world.

> Source: `deployment/world/state/rooms/gates.ts:4–27`,
> `deployment/world/state/goals.ts:149–159`, `deployment/world/data/rooms/rooms.csv`
> (rows 19, 59), `deployment/world/state/utils.ts:85–91` (`getGoalID`)

## Room Sharing

Many systems check whether two entities are in the same room:

```
sharesRoom = roomComponent.get(entityA) == roomComponent.get(entityB)
```

Used by: NPC shops (player must be in NPC's room), harvesting liquidation
(attacker must be in correct room), etc.

> Source: `LibRoom.sol:97–99, 144–147`

## World Data

The current world has **70 player-reachable rooms** across 4 z-planes (plus a
debug placeholder at index 0, `deadzone`, which cannot be entered), including:
- Overworld areas: Misty Riverside, Torii Gate, Scrapyard, Forest paths
- Interiors: Convenience Store, Plane Interior, Burning Room
- Caves: Temple Cave, Cave Crossroads, Fungus Garden, Sacrarium
- Special: Marketplace (room 66 — the Trade Room), Treasure Hoard (room 88)

Notable rooms:
- **Room 1** (Misty Riverside) — starting room for all new accounts
- **Room 11** (Temple by the Waterfall) — required location for every Kami
  naming/renaming; the **Kami** must be in room 11 and each naming consumes
  1 Holy Dust (`KamiNameSystem.sol:16–17, 29–33`)
- **Room 66** (Marketplace) — trade room (delivery fee waived)
- **Room 31** (Scrapyard Exit) — the crossroads fountain here opens the item
  pool / liquidity interface (see
  [item-pools.md](../marketplace/item-pools.md))

> Source: `data/rooms/rooms.csv`
