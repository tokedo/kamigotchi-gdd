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
| `Rarity` | Rarity value — written at registration (`LibItem.sol:97`) but not read by the item-droptable system (droptable weights live on the droptable/allo entities); its readers are trait/gacha-mint code (`LibTraitRegistry.sol`, `_721BatchMinterSystem.sol`) |
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

All three item-use systems apply effects through a single path:
`LibItem.applyAllos(world, components, itemIndex, "USE", amt, targetID)`, which
looks up the allos registered under the item's `"USE"` use case and passes them
to `LibAllo.distribute()`:

- `KamiUseItemSystem` → `applyAllos(..., "USE", 1, kamiID)` (`KamiUseItemSystem.sol:42`)
- `KamiCastItemSystem` → `applyAllos(..., "USE", 1, targetID)` (`KamiCastItemSystem.sol:38`)
- `AccountUseItemSystem` → `applyAllos(..., "USE", amt, accID)` (`AccountUseItemSystem.sol:29`)

`LibAllo.distribute()` dispatches each allo by its `Type` — stat deltas,
temporary bonuses, droptable commits, bonus clears, and basic grants (items,
XP, etc.). See [item-usage.md](item-usage.md) for the per-type breakdown.

**Stat allos**: the deployment writes an item's stat effect into the allo's
`Stat` struct with stat totals (HEALTH/POWER/VIOLENCE/HARMONY/STAMINA) in
`shift` and current-point effects (HP/SP, e.g. healing) in `sync`; `base` is
never set (`deployment/world/state/items/allos.ts:97–115`). On application,
`LibStat.add` maps `shift` → `shift`, `boost` → `boost`, `sync` → `sync` and
**discards the delta's `base` field** (`LibStat.sol:344–347`); a positive
`sync` delta is clamped so current points cannot exceed the stat's
bonus-inclusive total (`LibStat.sol:87–93`).

`LibItem` also defines `applyStats()` (`LibItem.sol:209`), `applyMove()`
(`LibItem.sol:219`), and `droptableCommit()` (`LibItem.sol:225`) helpers, but
no system calls them — all live item effects flow through the allo path above.

> Source: `LibItem.sol:196–206`, `LibAllo.sol:188–215`

## Item Flags

Items can have flags that modify behavior:

| Flag | Effect |
|---|---|
| `ITEM_UNBURNABLE` | Cannot be burned |
| `NOT_TRADABLE` | Cannot be transferred between players |
| `BYPASS_BONUS_RESET` | Using this item doesn't reset harvest bonuses |
| Item type as flag | Stored for reverse querying (e.g., `"EQUIPMENT"`) |

Items can also be **disabled** (via `LibDisabled`). The disabled flag blocks
Kami-use (`KamiUseItemSystem.sol:23`), cast (`KamiCastItemSystem.sol:29`), and
equip (`KamiEquipSystem.sol:28`) — but `AccountUseItemSystem` never calls
`verifyEnabled`, so account-targeted use of a disabled item still succeeds.

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
