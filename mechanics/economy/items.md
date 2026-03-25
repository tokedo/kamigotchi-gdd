# Items

> Source: `packages/contracts/src/libraries/LibItem.sol` (L1–519),
> `packages/contracts/src/libraries/LibInventory.sol` (L1–414)

## Overview

Items are **fungible** game objects held in inventories. Each item type is defined
once in a **registry** and can be held by any number of accounts. Items can be
consumed (used on Kamis/accounts), burned, traded, crafted, and some are backed
by ERC-20 tokens.

See `catalogs/items/items.csv` for the full item catalog.

## Item Registry Shape

Each item type is a registry entity with:

| Component | Description |
|---|---|
| `EntityType` | `"ITEM"` |
| `IsRegistry` | Marks as registry entry |
| `IndexItem` | Unique item index (uint32) |
| `Type` | Item type string (behavior category) |
| `Name` | Display name |
| `Description` | Item description |
| `MediaURI` | Image/media reference |
| `Rarity` | Rarity weight (for droptables) |
| `TokenAddress` | *(optional)* Linked ERC-20 token address |
| `Scale` | *(optional)* ERC-20 conversion scale |

Entity ID: `keccak256("registry.item", index)`

> Source: `LibItem.sol:40–65, 79–100`

## Known Item Constants

| Constant | Index | Description |
|---|---|---|
| `MUSU_INDEX` | 1 | Musu — primary currency (harvested resource) |
| `GACHA_TICKET_INDEX` | 10 | Gacha mint ticket |
| `REROLL_TICKET_INDEX` | 11 | Gacha reroll ticket |
| `ONYX_INDEX` | 100 | Onyx Shards — premium currency |
| `OBOL_INDEX` | 1015 | Obols |

> Source: `LibInventory.sol:24–28`

## Item Types

Items are categorized by their `Type` component, which determines behavior:

- **Base items** — simple holdable items (e.g., Musu, crafting materials)
- **EQUIPMENT** — equippable items that grant bonuses (see [equipment.md](equipment.md))
- **LOOTBOX** — items that resolve via `LibDroptable` when opened
- **Consumables** — items used on Kamis or accounts with stat effects

> Source: `LibItem.sol:53–65`

## Use Case System

Items can have different behaviors depending on how they are used. Each use case
is a separate reference entity:

```
Item Registry → Use Case (e.g., "USE", "BURN", "EQUIP") → Requirements + Effects
```

- **Requirements**: conditions that must be met (level, stats, location, etc.)
  validated via `LibConditional`
- **Effects (Allos)**: what happens when the item is used — distributes items,
  modifies stats, etc. via `LibAllo`

> Source: `LibItem.sol:102–148`

## Item Effects on Use

When a consumable item is applied to a target:

1. **Stat effects** — `LibItem.applyStats()` applies the item's stat deltas to the
   target. The registry item's `base` → target's `shift` (permanent). The registry
   item's `sync` → target's `sync` (e.g., healing).
2. **XP effects** — if the item has an Experience component, that XP is added to the target
3. **Move effects** — `applyMove()` can teleport the target to a specific room
4. **Allocation effects** — `applyAllos()` distributes items/resources per the use case's allo rules

> Source: `LibItem.sol:196–234`

## Item Flags

Items can have flags that modify behavior:

| Flag | Effect |
|---|---|
| `ITEM_UNBURNABLE` | Cannot be burned |
| `NOT_TRADABLE` | Cannot be transferred between players |
| `BYPASS_BONUS_RESET` | Using this item doesn't reset harvest bonuses |
| Item type as flag | Stored for reverse querying (e.g., `"EQUIPMENT"`) |

Items can also be **disabled** (via `LibDisabled`), preventing use.

> Source: `LibItem.sol:241–289`

## ERC-20 Backed Items

Some items are linked to on-chain ERC-20 tokens via `TokenAddress` and `Scale`
components. These items:
- Cannot be removed from the registry while a token address is set
- Have special handling for inventory increases (cannot use `incFor` directly —
  must go through the Token Portal system)
- Can be transferred between accounts

> Source: `LibItem.sol:122–137`

---

# Inventory

> Source: `packages/contracts/src/libraries/LibInventory.sol` (L1–414)

## Overview

Inventory is the **fungible balance tracking** system. Each account holds a
separate inventory instance per item type, storing a simple balance (quantity).

## Inventory Instance Shape

| Component | Description |
|---|---|
| `EntityType` | `"INVENTORY"` |
| `IndexItem` | Item index this inventory tracks |
| `IDOwnsInventory` | Holder entity ID (usually an account) |
| `Value` | Current balance |

Entity ID: `keccak256("inventory.instance", holderID, itemIndex)`

> Source: `LibInventory.sol:43–48, 360–361`

## Key Operations

### Increase (`incFor`)
- Creates inventory instance if it doesn't exist (lazy creation)
- Adds amount to balance
- **Blocks ERC-20 items** — token-backed items cannot be increased directly
- Logs global item count and per-account totals

### Decrease (`decFor`)
- Subtracts amount from balance (reverts on underflow)
- **Removes the inventory entity entirely if balance reaches 0** (prevents state bloat)
- Logs global item count decrease

### Transfer (`transferFor`)
- Decreases from sender, increases to recipient
- Allows ERC-20 items (unlike raw `incFor`)
- Transfer fee constant: **15** (defined but usage context is system-specific)

> Source: `LibInventory.sol:123–276`

## Transfer Restrictions

Items flagged `NOT_TRADABLE` cannot be transferred between accounts.
Checked via `verifyTransferable()`.

> Source: `LibInventory.sol:295–299`

## Balance Queries

- `getBalanceOf(holderID, itemIndex)` — get balance of specific item
- `getAllForHolder(holderID)` — get all inventory instances for a holder

> Source: `LibInventory.sol:304–355`
