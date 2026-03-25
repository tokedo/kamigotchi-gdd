# Items Catalog

> Source: `packages/contracts/deployment/world/data/items/`
> Commit: `d9b50091`

## Files

| File | Entries | Description |
|---|---|---|
| `items.csv` | 177 items | Complete item catalog with stats, effects, and descriptions |
| `effects.csv` | 94 effects | Item effect definitions (what items do when used/equipped) |
| `droptables.csv` | 6 tables | Weighted loot pools for lootbox items |

## items.csv Schema

| Column | Type | Description |
|---|---|---|
| Index | uint32 | Unique item ID (non-sequential, see ranges below) |
| Name | string | Display name |
| Type | enum | Item type category (see types below) |
| Rarity | enum | Common / Uncommon / Rare / Epic / Legendary |
| For | enum | Target entity: `Kami`, `Account`, `Enemy_Kami`, `Passport_slot`, `Kami_Pet_Slot`, or empty |
| Flags | string | Comma-separated flags (e.g., `NOT_TRADABLE`, `BYPASS_BONUS_RESET`) |
| Effects | string | Comma-separated effect keys (references `effects.csv`) |
| Requirements | string | Comma-separated requirement keys |
| Status | enum | `In Game`, `To Deploy`, `To Update` |
| Description | string | In-game flavor text |

## Item Types

| Type | Count | Description |
|---|---|---|
| Food | 44 | Consumables that restore HP, grant XP, or apply buffs to Kami |
| Material | 41 | Raw and processed crafting ingredients |
| Equipment | 36 | Equipable pet-slot items with passive stat bonuses |
| NFT | 14 | Passport items (equippable cosmetics) |
| Potion | 11 | Consumables with targeted effects (Kami or Enemy_Kami) |
| Key Item | 10 | Quest-related unique items |
| Misc | 7 | Currencies and special items (MUSU, Gacha Ticket, etc.) |
| Lootbox | 6 | Openable containers that roll on a droptable |
| Tool | 3 | Crafting tools (Grinder, Burner, Screwdriver) |
| Consumable | 2 | Generic consumables (Djed Pillar, VIPP) |
| Revive | 2 | Items that resurrect liquidated Kami |
| ERC20 | 1 | On-chain token (Onyx Shard) |

## Rarity Distribution

| Rarity | Count | Numeric Value |
|---|---|---|
| Common | 21 | 1 |
| Uncommon | 52 | 2 |
| Rare | 66 | 3 |
| Epic | 32 | 4 |
| Legendary | 6 | 5 |

## Index Ranges

| Range | Category | Examples |
|---|---|---|
| 1–33 | Currency, tickets, passports | MUSU (1), Gacha Ticket (10), Passports (20-33) |
| 100 | Premium currency | Onyx Shard (ERC20) |
| 1001–1021 | Raw materials | Wooden Stick, Stone, Black Poppy, Bone Chunk |
| 1102–1303 | Processed materials | Empty Cup, Microplastics, Ashlar, Timber, Ingot |
| 6001–6007 | Essences | Six elemental essences + Pure Essence |
| 11001–11020 | Revives & special | Red Ribbon Gummy, Holy Dust, Cleaning Fluid |
| 11110–11140 | Mochi (permanent stat boost) | Gaokerena, Sunset Apple, Kami, Mana |
| 11201–11233 | XP items | XP Candies, Heart Crystals, Jewels |
| 11301–11314 | HP food | Ghost Gum, Cheeseburger, Golden Apple |
| 11401–11413 | Potions & buffs | XP/Grace/Bless/Respec Potions, Festival Chime |
| 11501–11502 | Temporary stat potions | Toadstool Liquor, Sarcophagus Honey |
| 19001–19301 | Offensive potions (Enemy_Kami) | Spirit Glue, Animistic Poison, Curse Tablet |
| 21001–21206 | Account consumables | Lootboxes, Ice Cream, Spell Cards |
| 23100–23102 | Tools | Spice Grinder, Portable Burner, Screwdriver |
| 30001–30036 | Equipment (Pet Slot) | 12 sets of 3 tiers (Common/Uncommon/Rare) |
| 100001–100010 | Key Items | Quest items (Astrolabe Disk, Data Chips, etc.) |

## effects.csv Schema

| Column | Type | Description |
|---|---|---|
| Name | string | Effect key (referenced by items.csv Effects column) |
| Type | enum | `BONUS`, `STAT`, `ITEM`, `ITEM_DROPTABLE`, `STATE`, `ROOM`, `VIP`, `COOLDOWN`, `XP` |
| Descriptor | string | Specific stat/bonus being modified |
| Index | uint32 | Item index (for ITEM type) or room index (for ROOM type) |
| Value | int | Magnitude of the effect (can be negative) |
| Terminator | enum | When the effect expires (see below) |
| Droptable | string | Droptable name (for ITEM_DROPTABLE type) |

### Effect Types

| Type | Description | Example |
|---|---|---|
| BONUS | Temporary combat/harvest bonuses | `BOUNTY+25%` → +250 HARV_BOUNTY_BOOST, removed on next harvest |
| STAT | Permanent or equipment stat changes | `HP+50` → restore 50 HP; `E_POWER+5` → +5 Power while equipped |
| ITEM | Gives an item as side effect | `ITEM1003` → gives 1x Plastic Bottle (empty container return) |
| ITEM_DROPTABLE | Rolls a droptable | `DT Mochibox` → random Mochi from DT Mochibox table |
| STATE | Changes Kami state | `STATE-RESTING` → sets state to RESTING (used by revive items) |
| ROOM | Moves to a room | `MOVE13` → teleports to Convenience Store |
| XP | Grants experience points | `XP+1000` → adds 1000 XP |
| VIP | Grants VIP status | `VIP1` → activates VIP |
| COOLDOWN | Adds cooldown time | `NEXT_COOLDOWN+180` → adds 180s to next cooldown |

### Terminators (Effect Expiry)

| Terminator | When Effect is Removed |
|---|---|
| `UPON_HARVEST_ACTION` | After the next harvest action completes |
| `UPON_UNEQUIP` | When the equipment is unequipped |
| `UPON_DEATH` | When the Kami is liquidated (killed) |
| `UPON_LIQUIDATION` | After performing a liquidation attack |
| `UPON_KILL_OR_KILLED` | After killing or being killed in combat |
| (empty) | Permanent / instant effect |

### Equipment vs Consumable Effects

Effects prefixed with `E_` are **equipment bonuses** (persist while equipped, removed `UPON_UNEQUIP`).
Effects without `E_` prefix are **consumable bonuses** (one-shot, removed after their trigger event).

## droptables.csv Schema

| Column | Type | Description |
|---|---|---|
| Name | string | Droptable identifier |
| Indices | int[] | Comma-separated item indices in the pool |
| Tiers | int[] | Comma-separated weights (higher = more common) |
| Notes | string | Human-readable summary |

Tier weights determine drop probability: `P(item) = tier / sum(all tiers)`.

## Cross-References

- Items → Effects: `items.csv:Effects` references `effects.csv:Name`
- Items → Droptables: Lootbox items reference droptable effects (e.g., `DT OG`)
- Effects → Droptables: `effects.csv:Droptable` references `droptables.csv:Name`
- Effects → Items: `ITEM` type effects reference other items by index
- Droptables → Items: `droptables.csv:Indices` reference `items.csv:Index`
- Room droptables (in `catalogs/rooms/`) are separate from item droptables — room droptables define scavenging/node drops, while these define lootbox contents
