# Rooms Catalog

> Source: `packages/contracts/deployment/world/data/rooms/rooms.csv`
> Commit: `d9b50091`

## Schema

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

## Statistics

- **Total rooms**: 70
- **In Game**: 68
- **To Update**: 2 (rooms 19, 59)
- **Rooms with special exits**: 14
- **Missing exit targets**: Rooms 20, 24, 28 are referenced as exits but not defined in the CSV (possibly planned rooms)

## Adjacency

Rooms on the same Z-plane are adjacent if they differ by exactly 1 in either X
or Y (no diagonals). Rooms on different Z-planes are only connected via special
exits. See [mechanics/world/rooms.md](../../mechanics/world/rooms.md) for full
movement rules.
