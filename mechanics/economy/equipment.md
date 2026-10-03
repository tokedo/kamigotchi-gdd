# Equipment

> Source: `packages/contracts/src/libraries/LibEquipment.sol` (L1–279),
> `packages/contracts/src/systems/KamiEquipSystem.sol` (L1–44),
> `packages/contracts/src/systems/KamiUnequipSystem.sol` (L1–39)

## Overview

Equipment items are a special item type (`"EQUIPMENT"`) that are equipped to
**Kamis**, granting stat bonuses while worn. Each equipment item occupies a
named **slot**, and equipping/unequipping automatically manages bonuses.

`KamiEquipSystem` is the only caller of `LibEquipment.equip`
(`KamiEquipSystem.sol:32`) — no account-equip system exists, so only Kami
equipping is reachable in the current system set.

## Slot System

Each equipment item defines a **slot** via its `For` component. The slot string
prefix names an intended target type:

| Slot Pattern | Prefix Convention | Example |
|---|---|---|
| `Kami_Pet_Slot` | Kami-targeted | Pet slot equipment |
| `Account_Badge_Slot` | Account-targeted | Badge slot equipment |

The `Kami_`/`Account_` prefix is an **unenforced naming convention**:
`LibEquipment.equip` never validates the slot prefix against the holder type
(`LibEquipment.sol:91–124`), and the only equip entry point targets Kamis.

Only **one item** can occupy a given slot at a time. Equipping into an
**occupied slot swaps**: the equipped item is unequipped first — its
slot bonuses are cleared and the item is **returned to the account's
inventory** — and only then is the new item taken from the inventory and
equipped. A swap needs no free capacity. There is no "slot occupied" revert.

> Source: `LibEquipment.sol:27–41, 105–118, 244–255`

## Equipment Capacity

Each entity has a **maximum number of equipment slots** it can fill:

```
capacity = DEFAULT_CAPACITY + EQUIP_CAPACITY_SHIFT bonus
```

- **Default capacity**: `1`
- Capacity can be increased via the `EQUIP_CAPACITY_SHIFT` bonus (**currently
  unused** — no item or skill grants this bonus; all entities have default capacity)
- Capacity cannot go below 0

When replacing an item in an existing slot, capacity is not consumed (swap).
Capacity is only checked when adding equipment to a **new** slot.

> Source: `LibEquipment.sol:51–52, 202–207`

## System Entry Points

### KamiEquipSystem

`KamiEquipSystem.executeTyped(kamiID, itemIndex)`:

1. Resolve account from operator address
2. **Verify Kami ownership** — Kami must belong to caller's account
3. **Verify equip state** — Kami must be in `RESTING` state
4. **Verify item** — item must be enabled and have type `"EQUIPMENT"`
5. Equip the item (delegates to `LibEquipment.equip`)
6. Update account timestamp

> Source: `KamiEquipSystem.sol:19–37`

### KamiUnequipSystem

`KamiUnequipSystem.executeTyped(kamiID, slot)`:

1. Resolve account from operator address
2. **Verify Kami ownership** — Kami must belong to caller's account
3. **Verify equip state** — Kami must be in `RESTING` state
4. Unequip the item from the named slot (delegates to `LibEquipment.unequip`)
5. Update account timestamp

Note: unequip takes a **slot name** (string), not an item index.

> Source: `KamiUnequipSystem.sol:18–32`

## Equip Process

`LibEquipment.equip(world, components, holderID, inventoryID, itemIndex)`:

1. **Verify item type** — must be `"EQUIPMENT"`
2. **Get slot** — read the item's `For` component; revert `"Equipment: no slot"`
   if empty
3. **Check slot** — if occupied, run the full unequip for that slot first
   (clear `UPON_UNEQUIP_{SLOT}` bonuses, remove the instance, **return the old
   item to the inventory**); no capacity check. If the slot is free, require
   `equippedCount < capacity` (`"Equipment: at capacity"`)
4. **Consume from inventory** — remove 1 of the new item from the inventory
   (reverts on insufficient balance)
5. **Create equipment instance** — ECS entity linking holder + item + slot
6. **Assign bonuses** — apply the item's `EQUIP` use case bonuses to the holder

Kamis must be in `RESTING` state to equip (`"kami not RESTING"`).

> Source: `LibEquipment.sol:91–124, 158–160, 244–255`, `LibKami.sol:284–286`

## Unequip Process

`LibEquipment.unequip(world, components, holderID, inventoryID, slot)`:

1. **Get equipment instance** from the slot; revert `"Equipment: slot empty"`
   if nothing is equipped there
2. **Clear bonuses** — unassign all bonuses with end type `UPON_UNEQUIP_{SLOT}`
3. **Remove equipment instance** — delete the ECS entity
4. **Return to inventory** — add 1 of the item back to the inventory

Single-slot unequip and bulk `unequipAll` share the internal `_unequipByID`
helper (reads the item index + slot, clears bonuses, removes the instance,
returns the item).

> Source: `LibEquipment.sol:132–142, 244–255`

## Force-Unequip on Ownership Change

`LibEquipment.unequipAll(components, holderID, inventoryID)` strips **every**
equipped item from a Kami and returns each to the given inventory. It is called
on every path that changes a Kami's ownership or removes it from the world, so
equipment never travels with a Kami across owners. Returned items always go to
the **previous owner / seller**.

| Path | System / Library | When |
|---|---|---|
| Direct transfer | `KamiSendSystem` | Sending a Kami to another account |
| Marketplace list | `KamiMarketListSystem` | Listing a Kami for sale |
| Marketplace sale | `LibKamiMarket.fillOffer` / `fillCollectionOffer` / `fillListing` | Offer/listing filled (ownership → buyer) |
| Bridge out | `Kami721UnstakeSystem` | Unstaking the ERC-721 out of the world |
| Sacrifice | `LibSacrifice` | Burning a Kami in the sacrifice ritual |
| Gacha reroll | `KamiGachaRerollSystem` | Depositing Kamis into the gacha pool |

> Source: `LibEquipment.sol:232–255`, `KamiSendSystem.sol:63`,
> `KamiMarketListSystem.sol:34`, `LibKamiMarket.sol:129, 158, 187`,
> `Kami721UnstakeSystem.sol:47`, `LibSacrifice.sol:85`,
> `KamiGachaRerollSystem.sol:33`

## Equipment Instance Shape

| Component | Description |
|---|---|
| `EntityType` | `"EQUIPMENT"` |
| `IDOwnsEquipment` | Holder entity ID (kami or account) |
| `IndexItem` | Which item is equipped |
| `For` | Slot string |

Entity ID: `keccak256("equipment.instance", holderID, slot)` — one per holder per slot.

> Source: `LibEquipment.sol:59–72, 271–273`

## Bonus Lifecycle

Equipment bonuses use the end type `UPON_UNEQUIP_{SLOT}`
(`END_TYPE_PREFIX = "UPON_UNEQUIP_"`, `LibEquipment.sol:48`). This means:
- On equip: the item's `EQUIP` use-case bonus allo is assigned as temporary
  bonuses to the holder (`LibEquipment.sol:122–123`)
- On unequip: all bonuses tagged with that slot's end type
  (`"UPON_UNEQUIP_" + slot`, `LibEquipment.sol:276–278`) are cleared
  (`LibEquipment.sol:250–252`)
- Replacing equipment in the same slot properly clears old and applies new bonuses

The bonus allocation ID is derived deterministically
(`LibEquipment.getEquipBonusAlloID`, `LibEquipment.sol:216–225`):
```
refAnchor = keccak256("item.usecase", itemIndex)
refID = LibReference.genID("EQUIP", refAnchor)
alloAnchor = keccak256("item.allo", refID)
alloID = LibAllo.genID(alloAnchor, "BONUS", 1)
```

> Source: `LibEquipment.sol:48, 122–123, 216–225, 250–252, 276–278`

> ⚠️ **CHECKED-IN CATALOG DATA DOES NOT MATCH THE LIVE WORLD**: equipment
> bonuses **do** apply — equipping attaches the item's bonuses and unequipping
> removes them, confirmed by observation in the live world. The checked-in
> deployment data reads otherwise: `deployment/world/state/items/allos.ts:64`
> calls `api.bonus(itemIndex, 'USE', descriptor, terminator, 0, value)`,
> registering equipment bonuses under the **`USE`** use case with the
> terminator taken verbatim from `data/items/allos.csv`, whose equipment rows
> all carry the bare `UPON_UNEQUIP` with no slot suffix — whereas equip reads
> the **`EQUIP`** use-case anchor (`LibEquipment.getEquipBonusAlloID`,
> `LibEquipment.sol:216–225`) and unequip clears the slot-suffixed end type
> `UPON_UNEQUIP_{SLOT}` (`LibEquipment.sol:48, 277`). The live registration
> therefore differs from the checked-in catalog rows; the lifecycle documented
> above is the behavior that holds. The test suite registers `EQUIP` use-case
> bonuses with slot-suffixed terminators directly
> (`test/systems/Equipment.t.sol:66–95`), matching the observed behavior.
