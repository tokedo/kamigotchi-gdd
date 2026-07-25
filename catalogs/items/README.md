# Items Catalog

> Source: `packages/contracts/deployment/world/data/items/`
> Commit: `8302734d`

## Files

| File | Entries | Description |
|---|---|---|
| `items.csv` | 178 items | Complete item catalog with stats, effects, and descriptions |
| `effects.csv` | 93 effects | Item effect definitions (what items do when used/equipped) |
| `droptables.csv` | 6 tables | Weighted loot pools for lootbox items |

### Source Files & Transformations

- `items.csv` ← source `data/items/items.csv` (column order rearranged; `Image`
  column dropped).
- `effects.csv` ← source `data/items/allos.csv`. The GDD carries **94 of the
  112 source rows**: the 18 excluded rows are empty placeholders (a `Name` key
  only — no Type, Descriptor, or Value), are referenced by no item, and never
  reach the chain (allos are deployed only when an item's `Effects` column
  references them).
- Item requirements come from source `data/items/requirements.csv` (see
  "Item Requirements" below).
- **Name/description normalization**: the GDD strips decorative curly quotes
  (`“ ”`) from the display names of items 11002, 11211–11214, 11305, 21204,
  and 100006, and normalizes curly apostrophes/punctuation in some
  descriptions. In-game names include the original characters (e.g.,
  `“Melkarth’s Heroic Awakening” Spell Card`).

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
| Requirements | string | Comma-separated requirement keys (see Item Requirements below) |
| Status | enum | `In Game`, `To Deploy`, `To Update` |
| Description | string | In-game flavor text |

## Item Requirements

Requirement keys in `items.csv:Requirements` resolve via source
`data/items/requirements.csv`, which defines two requirements:

| Key | Type | Preposition | Index | Value | Meaning |
|---|---|---|---|---|---|
| `VIP_ROOM` | `ROOM` | `AT` | 64 | — | Usable only while the account is in room 64 (Burning Room) |
| `MOCHI_LIMIT` | `MOCHI_USED` | `MAX` | 0 | 2 | Usable only while the target's `MOCHI_USED` counter is at most 2 |

**VIPP (item 2)** carries `VIP_ROOM` — it can be used/burned only in the
Burning Room (`catalogs/rooms/rooms.csv` room 64).

The four **Mochi** items (11110 Gaokerena, 11120 Sunset Apple, 11130 Kami,
11140 Mana) carry `MOCHI_LIMIT`, capping permanent stat mochis at 2 per
target.

> ⚠️ UNCERTAIN: `MOCHI_USED` is not one of `LibGetter.getBal`'s named types, so
> it resolves to a generic `LibData` counter
> (`LibGetter.sol:67–68`). No system in `packages/contracts/src/` writes that
> key, so the counter reads `0` and the `MAX 2` check always passes — the cap
> does not bind under this source. Verify live on-chain behavior before
> publishing it as an enforced limit.

## Item Types

| Type | Count | Description |
|---|---|---|
| Food | 44 | Consumables that restore HP, grant XP, or apply buffs to Kami |
| Material | 41 | Raw and processed crafting ingredients |
| Equipment | 36 | Equipable pet-slot items with passive stat bonuses |
| NFT | 14 | Passport items (equippable cosmetics) |
| Potion | 11 | Consumables with targeted effects (Kami or Enemy_Kami) |
| Key Item | 11 | Quest-related unique items (incl. Ring of Spirits, 22802) |
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
| Uncommon | 51 | 2 |
| Rare | 66 | 3 |
| Epic | 34 | 4 |
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
| Type | enum | `BONUS`, `STAT`, `CLEAR_BONUS`, `ITEM`, `ITEM_DROPTABLE`, `STATE`, `ROOM`, `VIP`, `COOLDOWN`, `XP` |
| Descriptor | string | Specific stat/bonus being modified |
| Index | uint32 | Item index (for ITEM type) or room index (for ROOM type) |
| Value | int | Magnitude of the effect (can be negative) |
| Terminator | enum | When the effect expires (see below) |
| Droptable | string | Droptable name (for ITEM_DROPTABLE type) |

### Effect Types

| Type | Description | Example |
|---|---|---|
| BONUS | Temporary combat/harvest bonuses | `BOUNTY+25%` → +250 HARV_BOUNTY_BOOST, removed on next harvest |
| STAT | Permanent or equipment stat changes; HP/SP point restores | `HP+50` → restore 50 HP; `SP+80` → restore 80 Stamina; `E_POWER+5` → +5 Power while equipped |
| CLEAR_BONUS | Clears all temporary bonuses on the target | `CLEARALL` → wipes active temporary effects (Cleaning Fluid) |
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

## Known Discrepancies

> ⚠️ **UNCERTAIN / source bug — Cultivation III Spell Card (11213).** Its
> `Effects` column references the effect key `XP+10000`, but **no `XP+10000`
> row exists in `effects.csv`** (`allos.csv` in source). Allo resolution is an
> exact name-match (`LibBonus`/`addAllos`): an unmatched key is logged as an
> error and **skipped** at deploy. As written, Cultivation III therefore grants
> only its `HP+100` effect — the intended 10,000 XP would not apply until an
> `XP+10000` effect/allo is added. The item description ("grant 10000 XP")
> reflects the design intent. Source: `items.csv` row 11213 vs `allos.csv`
> (no `XP+10000`); `deployment/world/state/items/allos.ts:30–33`.
