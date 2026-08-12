# Rooms Catalog

> Source: `packages/contracts/deployment/world/data/rooms/`
> Commit: `8302734d`

## Files

| File | Entries | Description |
|---|---|---|
| `rooms.csv` | 71 rooms | Room locations, coordinates, exits, lore text |
| `nodes.csv` | 64 nodes | Harvest/scavenge nodes per room with affinity and drops |
| `scavenge-droptables.csv` | 50 tables | Weighted loot pools for scavenging (with resolved item names) |
| `gates.csv` | 11 gates | Room access conditions (extracted from `deployment/world/state/rooms/gates.ts`) |

## rooms.csv Schema

| Column | Type | Description |
|---|---|---|
| Index | uint32 | Unique room identifier (non-sequential, range 1–90) |
| Name | string | Display name |
| Status | enum | `In Game` or `To Update` |
| X | int | East-west coordinate |
| Y | int | North-south coordinate |
| Z | int | Depth layer (see below) |
| Exits | int[] | Comma-separated special exit room indices (non-adjacent connections) |
| Description | string | In-game flavor text |

## nodes.csv Schema

| Column | Type | Description |
|---|---|---|
| Index | uint32 | Node index (matches room index — 1:1 mapping) |
| Name | string | Node display name (usually matches room name) |
| Status | enum | `In Game` |
| Drops | string | Droptable name (references `scavenge-droptables.csv`) |
| Affinity | enum | `Normal`, `Eerie`, `Insect`, `Scrap`, or multi (e.g., `Eerie, Scrap`) |
| Level Limit | uint32 | Max kami level for harvesting at this node (empty = no limit) |
| YieldIndex | uint32 | Base yield item index: 1 = MUSU (53 nodes) or 2 = VIPP (11 nodes, e.g., Cave Crossroads 18, Scrap Trees 60, Treasure Hoard 88) |
| Scav Cost | uint32 | Scavenge bar point cost (100–500) |

The source CSV carries one further column, **`💥 Node Effects`**, which is
dropped here: it is **empty on all 64 source rows** and is never read by the
node initialiser (`deployment/world/state/rooms/nodes.ts:6–19, 44–47`).

### No per-node bonus catalog

> ⚠️ **Node bonuses are runtime world state, not source data.** Nodes can grant
> temporary `UPON_HARVEST_STOP` bonuses to Kamis harvesting on them
> (`LibNode.sol:72–88`), but **which node grants what is not present in the
> source repo at the pin**, so no catalog file can be extracted. The empty
> `💥 Node Effects` column is the only trace in the data sheets, and the admin
> helper that would write them (`api.node.bonus.add`,
> `deployment/world/api/nodes.ts:35–42`) has zero callers. The values are set
> by admin calls to `_NodeRegistrySystem.addBonus`
> (`_NodeRegistrySystem.sol:48–54`) made outside the deployment pipeline and
> are only readable on chain, under the anchor
> `keccak256("node.bonus", nodeIndex)`. See
> [mechanics/world/nodes.md](../../mechanics/world/nodes.md#which-nodes-grant-what--runtime-world-state).

## scavenge-droptables.csv Schema

| Column | Type | Description |
|---|---|---|
| Name | string | Droptable identifier (referenced by `nodes.csv:Drops`) |
| Indices | int[] | Comma-separated item indices (references `catalogs/items/items.csv`) |
| Tiers | int[] | Comma-separated weights (higher = more common) |
| Items (resolved names) | string | Human-readable item names for reference |

Drop probability: `P(item) = tier / sum(all tiers)`

## Z-Planes (Depth Layers)

| Z | Layer | Room Count | Examples |
|---|---|---|---|
| 1 | Overworld | 39 | Forests, scrapyard, paths, riverside |
| 2 | Interiors | 3 | Convenience Store (13), Plane Interior (54), Burning Room (64) |
| 3 | Underground / Caves | 25 | Temple Cave (15), Cave Crossroads (18), Sacrarium (87) |
| 4 | Castle | 3 | Treasure Hoard (88), Trophies (89), Scenic View (90) |

## Notable Rooms

- **Room 1** (Misty Riverside) — starting room for new accounts
- **Room 12** (Scrap Confluence) — ERC-721 bridge room (stake/unstake)
- **Room 66** (Marketplace) — trade room (delivery fee waived)
- **Room 11** (Temple by the Waterfall) — first Kami naming location

## Affinity Distribution

| Affinity | Node Count | Description |
|---|---|---|
| Normal | 19 | No elemental bonus |
| Insect | 15 | Bug elemental |
| Eerie | 14 | Ghost/spirit elemental |
| Scrap | 12 | Metal/junk elemental |
| Dual affinity | 4 | Eerie+Scrap (2), Insect+Scrap (1), Eerie+Insect (1) |

## Scavenge Cost Tiers

| Cost | Node Count | Typical Drops |
|---|---|---|
| 100 | 18 | Basic materials (sticks, stones, scrap) |
| 200 | 26 | Intermediate (cones, daffodils, resin, pansy) |
| 300 | 12 | Advanced (mint, amber, ooze, coins) |
| 500 | 8 | Premium (essences, screwdriver, rename dust) |

## Statistics

- **Total rooms**: 71, of which 70 are player-reachable — room `0` (`deadzone`)
  is a debug placeholder whose own description states it cannot be entered
- **Rooms with nodes**: 64 (7 rooms have no harvest node, counting the deadzone)
- **In Game**: 71 rooms / 64 nodes
- **Rooms with special exits**: 14
- **Unique droptables**: 50 defined, but only 49 referenced by nodes — the
  `Bottle Scrap Burger` table is orphaned (referenced by no node; same in source)
- **Missing exit targets**: Rooms 20, 24, 28 are referenced as exits but not defined in the CSV
- **Name normalization**: room 85 is stored here as `Giant's Palm` (straight
  apostrophe); the source/in-game name uses a curly apostrophe (`Giant’s Palm`)

## Rooms Without Nodes

Rooms 4 (Vending Machine), 11 (Temple by the Waterfall), 13 (Convenience Store),
54 (Plane Interior), 64 (Burning Room), 66 (Marketplace) have no harvest nodes.
These are typically interiors or special-purpose locations.

## Room Gates (gates.csv)

Gates are **not deployed from a CSV** — they are hardcoded in
`packages/contracts/deployment/world/state/rooms/gates.ts` (marked "placeholder
until notion is up"). `gates.csv` here is extracted from that file: each row is
one `createGate(roomIndex, sourceIndex, conditionIndex, conditionValue, type,
logicType, for)` call, mapped to the columns Room Index / Source Room Index /
Condition Index / Condition Value / Condition Type / Logic / For.

- **Source Room Index** `0` = the gate applies from any entrance; a nonzero
  value (room 88's gate, source 72) applies only when entering from that room.
- **QUEST + BOOL_IS** (room 15) — the account must have completed the quest in
  `Condition Index` (quest 35, "Steel Your Heart").
- **COMPLETE_COMP + BOOL_IS** (9 rooms) — the entity in `Condition Value` must
  be marked complete. These use `getGoalID(n)` = `keccak256("goal", n)`,
  gating rooms behind community goal completion. Goal names are from
  `deployment/world/state/goals.ts`.
- **ITEM + CURR_MIN** (room 88) — the account must hold at least
  `Condition Value` (1) of item `Condition Index` (100004, Aetheric Sextant).

> ⚠️ UNCERTAIN: room 19's gate references goal 999, which is not defined in
> `goals.ts` (the gates.ts comment says "was coop 8 before"). Whether goal 999
> exists on-chain (created by other means) is not determinable from the
> deployment scripts alone.

A commented-out test gate for room 1 in `gates.ts` is not deployed and is
excluded. See [mechanics/world/rooms.md](../../mechanics/world/rooms.md)
("Gates (Room Access Conditions)") for the gate mechanic and on-chain checks.

## Cross-Reference Chain

```
Room (rooms.csv) ←→ Node (nodes.csv) → Droptable (scavenge-droptables.csv) → Items (catalogs/items/items.csv)
                                       ↘ Affinity → affects harvest effectiveness (see mechanics/utility/affinity.md)
```

## Adjacency

Rooms on the same Z-plane are adjacent if they differ by exactly 1 in either X
or Y (no diagonals). Rooms on different Z-planes are only connected via special
exits. See [mechanics/world/rooms.md](../../mechanics/world/rooms.md) for full
movement rules.
