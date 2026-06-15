# NPCs & Shop Listings Catalog

> Source: `packages/contracts/deployment/world/data/npc/npc.csv`,
> `packages/contracts/deployment/world/data/listings/listings.csv`
> Commit: `91f69796` (npc/listings data unchanged since `0af5d9f0`)

## Files

| File | Entries | Description |
|---|---|---|
| `npcs.csv` | 2 NPCs | NPC registry with room locations |
| `listings.csv` | 19 listings | Shop inventories with prices and GDA pricing models |

## npcs.csv Schema

| Column | Type | Description |
|---|---|---|
| Index | uint32 | Unique NPC identifier |
| Name | string | Display name |
| Room Index | uint32 | Room where this NPC is located |
| Room Name | string | Room name for reference |

## listings.csv Schema

| Column | Type | Description |
|---|---|---|
| Name | string | Listing identifier |
| Status | enum | `In-game` or `Shelved` |
| NPC Index | uint32 | Which NPC sells this (references `npcs.csv`) |
| NPC Name | string | NPC name for reference |
| Item Index | uint32 | What item is sold (references `catalogs/items/items.csv`) |
| Item Name | string | Item name for reference |
| Currency Index | uint32 | Currency used: `1` = MUSU, `100` = Onyx Shard |
| Base Price | number | Starting price in the currency |
| Buy Price Model | string | Pricing formula (see below) |
| Requirements | string | Gating conditions (e.g., quest completion) |

## NPCs

| NPC | Location | Listings | Specialty |
|---|---|---|---|
| **Mina** | Convenience Store (Room 13) | 12 | General goods: materials, food, ice cream, tools |
| **Vending Machine** | Cave Crossroads (Room 18) | 7 | Food and ice cream (cave supply) |

## Pricing Models

### GDA (Gradual Dutch Auction)

Most items use dynamic pricing: `GDA {supply} {period} {decay}%`

- **Supply**: Total units available per period (e.g., 300, 1500)
- **Period**: Reset interval (`Daily` = 24h, `Bidaily` = 48h)
- **Decay**: Price decay rate per period (always 50%)

As items are bought, price increases. Price decays back to base over time.
See [mechanics/marketplace/auctions.md](../../mechanics/marketplace/auctions.md) for GDA math.

### FIXED

Some items (Spice Grinder, Portable Burner) have fixed prices — no supply limit
or dynamic pricing.

### Onyx Shard Pricing

> **Currently unused.** Mina has two Onyx Shard listings registered on-chain
> (Wooden Stick at 0.05 Onyx, Stone at 1 Onyx), but **no pricing strategy is
> assigned** — these listings have no buy or sell side, so `calcBuyPrice()`
> reverts. Players cannot purchase these items from Mina. The deployment script
> contains a commented-out `initLocalListings()` that was used for local ERC-20
> testing of these listings.

All active shop items are priced in MUSU.

## Statistics

- **Total listings**: 19 (18 in-game, 1 shelved)
- **Active buyable listings**: 16 (2 Onyx listings are registered but non-functional)
- **Unique items sold**: 11 distinct items
- **Currency split**: 2 Onyx Shard listings (dormant), 17 MUSU listings
- **Pricing models**: 15 GDA, 2 FIXED, 2 dormant (no pricing assigned)

## Cross-References

- NPCs → Rooms: `npcs.csv:Room Index` references `catalogs/rooms/rooms.csv:Index`
- Listings → NPCs: `listings.csv:NPC Index` references `npcs.csv:Index`
- Listings → Items: `listings.csv:Item Index` references `catalogs/items/items.csv:Index`

## Notes

- Item 21100 (Mina Shop Scroll) is referenced in a shelved listing but does not
  exist in the current items catalog — likely a planned but undeployed item
- The Vending Machine (Room 18) has lower GDA supply than Mina (Room 13) but
  uses `Bidaily` periods, making items available less frequently but at lower
  competition
