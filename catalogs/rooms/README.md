# Rooms Catalog

> Source: `packages/contracts/deployment/world/data/rooms/`
> Commit: `d9b50091`

## Files

| File | Entries | Description |
|---|---|---|
| `rooms.csv` | 70 rooms | Room locations, coordinates, exits, lore text |
| `nodes.csv` | 64 nodes | Harvest/scavenge nodes per room with affinity and drops |
| `scavenge-droptables.csv` | 50 tables | Weighted loot pools for scavenging (with resolved item names) |

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
| YieldIndex | uint32 | Base yield item index (1 = MUSU for all current nodes) |
| Scav Cost | uint32 | Scavenge bar point cost (100–500) |

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
| 100 | 20 | Basic materials (sticks, stones, scrap) |
| 200 | 26 | Intermediate (cones, daffodils, resin, pansy) |
| 300 | 11 | Advanced (mint, amber, ooze, coins) |
| 500 | 7 | Premium (essences, screwdriver, rename dust) |

## Statistics

- **Total rooms**: 70
- **Rooms with nodes**: 64 (6 rooms have no harvest node)
- **In Game**: 70 rooms / 64 nodes (rooms 19 Temple of the Wheel + 59 Black Pool now live)
- **Rooms with special exits**: 14
- **Unique droptables**: 50
- **Missing exit targets**: Rooms 20, 24, 28 are referenced as exits but not defined in the CSV

## Rooms Without Nodes

Rooms 4 (Vending Machine), 11 (Temple by the Waterfall), 13 (Convenience Store),
54 (Plane Interior), 64 (Burning Room), 66 (Marketplace) have no harvest nodes.
These are typically interiors or special-purpose locations.

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
