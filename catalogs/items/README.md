# Items Catalog

> Source: `packages/contracts/deployment/world/data/items/`
> Commit: `a99b4b64`

## Files

| File | Entries | Description |
|---|---|---|
| `items.csv` | 190 items | Complete item catalog with stats, effects, and descriptions |
| `effects.csv` | 99 effects | Item effect definitions (what items do when used/equipped) |
| `droptables.csv` | 8 tables | Weighted loot pools for lootbox items |

### Source Files & Transformations

- `items.csv` ← source `data/items/items.csv` (column order rearranged; `Image`
  column dropped).
- `effects.csv` ← source `data/items/allos.csv`, all 99 rows (the source's
  leading blank `.` column dropped). Allos reach the chain only when an item's
  `Effects` column references them; since `a99b4b64` an `Effects` (or
  `Requirements`) key with no matching row stops the deployment with an error
  instead of being skipped.
- Item requirements come from source `data/items/requirements.csv` (see
  "Item Requirements" below).
- **Name/description normalization**: the GDD strips decorative curly quotes
  (`“ ”`) from the display names of items 11002, 11211–11214, 11305, 21204,
  100006 and 100011, and normalizes curly apostrophes/punctuation in some
  descriptions. In-game names include the original characters (e.g.,
  `“Melkarth’s Heroic Awakening” Spell Card`).

## items.csv Schema

| Column | Type | Description |
|---|---|---|
| Index | uint32 | Unique item ID (non-sequential, see ranges below) |
| Name | string | Display name |
| Type | enum | Item type category (see types below) |
| Rarity | enum | Common / Uncommon / Rare / Epic / Legendary |
| For | enum | Target entity: `Kami`, `Account`, `Enemy_Kami`, `Any_Kami`, `RESPEC` (11403 only, used by the skill respec), the equipment slot `Kami_Pet_Slot`, or empty (a basic, unusable item) |
| Flags | string | Comma-separated flags (e.g., `NOT_TRADABLE`, `BYPASS_BONUS_RESET`) |
| Effects | string | Comma-separated effect keys (references `effects.csv`) |
| Requirements | string | Comma-separated requirement keys (see Item Requirements below) |
| Status | enum | `In Game`, `Shelved`, `To Deploy`, `To Update` |
| Description | string | In-game flavor text |

## Item Requirements

Requirement keys in `items.csv:Requirements` resolve via source
`data/items/requirements.csv`, which defines five requirements (all `USE`):

| Key | Type | Preposition | Index | Value | Meaning | Items |
|---|---|---|---|---|---|---|
| `VIP_ROOM` | `ROOM` | `AT` | 64 | — | Usable only while the account is in room 64 (Burning Room) | VIPP (2) |
| `MOCHI_LIMIT` | `MOCHI_USED` | `MAX` | 0 | 1 | Usable only while the target Kami's `MOCHI_USED` counter is at most 1 | the four Mochi |
| `KAMI_CAN_EAT` | `KAMI_CAN_EAT` | `IS` | 0 | — | Target Kami is `RESTING` or `HARVESTING` | 49 Kami / enemy / any-Kami items |
| `KAMI_DEAD` | `STATE` | `IS` | 3 | — | Target Kami is `DEAD` | the revive items 11001–11004 |
| `KAMI_RESTING` | `STATE` | `IS` | 1 | — | Target Kami is `RESTING` | none |

Since `a99b4b64` these keys are the only state rules a Kami item has: the
deployment no longer adds a hidden rule derived from the item's `Type` (see
[item-usage.md](../../mechanics/economy/item-usage.md#state-rules-come-from-the-items-requirements)).

**VIPP (item 2)** carries `VIP_ROOM` — it can be used/burned only in the
Burning Room (`catalogs/rooms/rooms.csv` room 64).

**Mochi cap — two per Kami, any kind.** The four Mochi (11110 Gaokerena,
11120 Sunset Apple, 11130 Kami, 11140 Mana) carry `MOCHI_LIMIT` and the effect
`MOCHI_USED` (a basic allo: `+1` on the Kami's `LibData` counter
`MOCHI_USED`, index 0). The requirement is checked before the effects apply,
so the first and second Mochi pass (counter 0, then 1) and the third reverts
`"Item: requirements not met"`; the counter is shared by all four kinds.

> Before `a99b4b64` no Mochi carried the `MOCHI_USED` effect and the limit was
> `MAX 2`: nothing wrote the counter, so it read 0 and the cap never bound.

> Source: `deployment/world/data/items/requirements.csv:2–6`,
> `deployment/world/data/items/allos.csv` (`MOCHI_USED`),
> `deployment/world/state/items/allos.ts:37–56`, `LibSetter.sol:64–66`,
> `LibGetter.sol:67–68`, `LibConditional.sol:288–289`,
> `LibItem.sol:310–318`, `test/systems/Items/ItemKami.t.sol` (mochi tests)

## Item Types

`Type` is a display category; since `a99b4b64` the deployment reads it only
to recognise `Equipment` (see `mechanics/economy/items.md`).

| Type | Count | Description |
|---|---|---|
| Material | 42 | Raw and processed crafting ingredients (incl. Holy Dust, 11011) |
| Equipment | 36 | Equipable pet-slot items with passive stat bonuses |
| Food | 27 | Kami food: HP, XP and buff snacks, the four Mochi, Igors Candy |
| NFT | 14 | Passport items (cosmetics) |
| Potion | 14 | Brewed potions, incl. the enemy-Kami poisons and Pale Potion |
| Consumable | 13 | Other usable items (Red Ribbon Gummy, Djed Pillar, VIPP, jewels, talismans, Energy Drink, ...) |
| Key Item | 13 | Quest-related unique items (incl. Ring of Spirits 22802, Igors Amulet 100012) |
| Lootbox | 8 | Openable containers that roll on a droptable (incl. Silt-Caked Box 21007, Golden Box 21008) |
| Misc | 7 | Currencies and tickets (MUSU, Gacha Ticket, Ballot, Golden Ballot, ...) |
| Spell Card | 7 | Spell Cards (Cultivation I–IV, Paeon's Field, Neith's River, Melkarth's Awakening) |
| Cosmetic | 4 | Voter Badges I–IV (15–18) |
| Tool | 3 | Crafting tools (Grinder, Burner, Screwdriver) |
| ERC20 | 2 | Portal-bridged tokens (Onyx Shard, Ether Shard) |

## Rarity Distribution

| Rarity | Count | Numeric Value |
|---|---|---|
| Common | 11 | 1 |
| Uncommon | 52 | 2 |
| Rare | 61 | 3 |
| Epic | 48 | 4 |
| Legendary | 18 | 5 |

57 rarities were realigned to the design sheet in this sync's range (e.g. the
equipment sets now run Rare / Epic / Legendary by tier). Rarity is written only
when an item is created; the world-state command `items setRarity` rewrites it
on live items from the sheet.

## Index Ranges

| Range | Category | Examples |
|---|---|---|
| 1–33 | Currency, tickets, ballots, passports | MUSU (1), Gacha Ticket (10), Ballot (13), Golden Ballot (14), Voter Badges (15–18), Passports (20-33) |
| 100, 103 | Bridged tokens (ERC20) | Onyx Shard (100), Ether Shard (103) — see `mechanics/marketplace/token-portal.md` |
| 1001–1021 | Raw materials | Wooden Stick, Stone, Black Poppy, Bone Chunk |
| 1102–1303 | Processed materials | Empty Cup, Microplastics, Ashlar, Timber, Ingot |
| 6001–6007 | Essences | Six elemental essences + Pure Essence |
| 11001–11020 | Revives & special | Red Ribbon Gummy, Holy Dust, Cleaning Fluid |
| 11110–11140 | Mochi (permanent stat boost) | Gaokerena, Sunset Apple, Kami, Mana |
| 11201–11233 | XP items | XP Candies, Heart Crystals, Jewels |
| 11301–11314 | HP food | Ghost Gum, Cheeseburger, Golden Apple |
| 11401–11414 | Potions & buffs | XP/Grace/Bless/Respec Potions, Festival Chime, Igors Candy |
| 11501–11502 | Temporary stat potions | Toadstool Liquor, Sarcophagus Honey |
| 19001–19301 | Offensive potions (Enemy_Kami) | Spirit Glue, Animistic Poison, Curse Tablet |
| 21001–21206 | Account consumables | Lootboxes (incl. Silt-Caked Box 21007, Golden Box 21008), Ice Cream, Spell Cards |
| 23100–23102 | Tools | Spice Grinder, Portable Burner, Screwdriver |
| 30001–30036 | Equipment (Pet Slot) | 12 sets of 3 tiers (Rare/Epic/Legendary) |
| 100001–100012 | Key Items | Quest and co-op items (Astrolabe Disk, Data Chips, Igors Amulet, etc.) |

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

The `Terminator` column is the **sole** source of an effect's lifetime: it is
passed straight through to the bonus registry as the end-anchor type
(`deployment/world/state/items/allos.ts:58–65` → `LibAllo.createBonus`,
`LibAllo.sol:93–115`). Effect **key names carry no semantics** — see the `_KK`
note below.

| Terminator | When Effect is Removed |
|---|---|
| `UPON_HARVEST_ACTION` | After the next harvest action completes (collect / feed / stop) |
| `UPON_COOLDOWN_SET` | On the next cooldown (re)set — harvest start/stop/collect, liquidation |
| `UPON_UNEQUIP` | Intended: on unequip. **Inert as deployed** — the runtime clears `UPON_UNEQUIP_{SLOT}`, and these allos are registered under the `USE` case ([bonus-system.md](../../mechanics/combat/bonus-system.md#end-types-temporary-bonus-lifecycle)) |
| `UPON_DEATH` | Only when the holder is liquidated (a kill by the holder does not clear it) |
| `UPON_LIQUIDATION` | Only when the holder successfully liquidates another Kami (being killed does not clear it) |
| `UPON_KILL_OR_KILLED` | On **either** — the holder killing or being killed |
| (empty) | Permanent / instant effect |

Full lifecycle rules, including which resetter runs on which side of a
liquidation, are in
[mechanics/combat/bonus-system.md](../../mechanics/combat/bonus-system.md#combat-buff-reset--exact-semantics).

### Effect Key Naming Conventions

Effect keys are free-form strings matched by exact name between
`items.csv:Effects` and `effects.csv:Name`. Nothing in the contracts parses
them; prefixes and suffixes are a human convention in the sheet only.

| Convention | Meaning (by convention) | Authoritative field |
|---|---|---|
| `E_` prefix | Equipment bonus, meant to persist while equipped | `Terminator` = `UPON_UNEQUIP` |
| no prefix | Consumable bonus, one-shot | `Terminator` |
| `_KK` suffix | "kill or killed" — a kill-scoped variant of an otherwise identically-named effect | `Terminator` |

Three keys carry `_KK`:

| Key | Bonus type | Value | Terminator | Item |
|---|---|---|---|---|
| `ATR+10%_KK` | `ATK_THRESHOLD_RATIO` | +100 | `UPON_KILL_OR_KILLED` | Flash Talisman (11412) |
| `DTR-10%_KK` | `DEF_THRESHOLD_RATIO` | −100 | `UPON_KILL_OR_KILLED` | Flash Talisman (11412) |
| `ATS-30%_KK` | `ATK_THRESHOLD_SHIFT` | −300 | `UPON_DEATH` | Curse Tablet (19301) |

The suffix exists to disambiguate `ATR+10%_KK` from the plain `ATR+10%` key —
same bonus type and value, different terminator (`UPON_LIQUIDATION`, used by
the Inverted Teardrop Jewel, 11224).

> ⚠️ **The `_KK` suffix has drifted.** `ATS-30%_KK` now terminates on
> `UPON_DEATH`, not `UPON_KILL_OR_KILLED`, so the name no longer describes the
> lifetime. Always read the `Terminator` column, never the key name.

Neither the prefix nor the suffix affects scope or stacking. A `_KK` bonus
applies to whichever entity the item is used or cast on, and — like every
temporary bonus — one instance exists per (source item, bonus type, holder) at
level 1, so a second dose of the same item adds nothing. See
[bonus-system.md → Stacking Rules](../../mechanics/combat/bonus-system.md#stacking-rules).

### `BYPASS_BONUS_RESET` (items.csv Flags)

An item flag, not an effect. When **absent**, using the item on your own Kami
first clears that Kami's `UPON_HARVEST_ACTION` bonuses; when **present**, the
item can be fed mid-harvest without destroying an active buff
(`KamiUseItemSystem.sol:34–37`, `LibItem.sol:239–243`). It is read only on the
own-Kami `USE` path, so it is inert on casts and on account items — including
for an `Any_Kami` item, which honours the flag when used on your own Kami and
ignores it when cast. 19 items carry it. Full semantics:
[bonus-system.md](../../mechanics/combat/bonus-system.md#bypass_bonus_reset-item-flag).

## droptables.csv Schema

| Column | Type | Description |
|---|---|---|
| Name | string | Droptable identifier |
| Indices | int[] | Comma-separated item indices in the pool |
| Tiers | int[] | Comma-separated weights (higher = more common) |
| Notes | string | Human-readable summary |

Tier weights are exponential: each tier `t` weighs `2^(t−1)` (0 never drops), so
`P(item) = 2^(t−1) / Σ 2^(t_i−1)` — see
[droptables.md](../../mechanics/economy/droptables.md). The `Value` of an
`ITEM_DROPTABLE` effect is the number of rolls (1 on every row).

## Cross-References

- Items → Effects: `items.csv:Effects` references `effects.csv:Name`
- Items → Droptables: Lootbox items reference droptable effects (e.g., `DT OG`)
- Effects → Droptables: `effects.csv:Droptable` references `droptables.csv:Name`
- Effects → Items: `ITEM` type effects reference other items by index
- Droptables → Items: `droptables.csv:Indices` reference `items.csv:Index`
- Room droptables (in `catalogs/rooms/`) are separate from item droptables — room droptables define scavenging/node drops, while these define lootbox contents

## Known Discrepancies

None open at the pin. The earlier one — Cultivation III Spell Card (11213)
naming the effect `XP+10000` with no such row in the source — is closed: the
row exists since this sync's range (`XP`, value 10000), so the card grants its
`HP+100` and 10,000 XP as its description says.
