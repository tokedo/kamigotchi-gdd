# Equipment

> Source: `packages/contracts/src/libraries/LibEquipment.sol` (L1–262)

## Overview

Equipment items are a special item type (`"EQUIPMENT"`) that can be equipped to
Kamis or accounts, granting stat bonuses while worn. Each equipment item occupies
a named **slot**, and equipping/unequipping automatically manages bonuses.

## Slot System

Each equipment item defines a **slot** via its `For` component. The slot string
encodes both the target type and slot name:

| Slot Pattern | Target | Example |
|---|---|---|
| `Kami_Pet_Slot` | Kami | Pet slot equipment |
| `Account_Badge_Slot` | Account | Badge slot equipment |

Only **one item** can occupy a given slot at a time. Equipping a new item into an
occupied slot automatically unequips the existing one first.

> Source: `LibEquipment.sol:27–41`

## Equipment Capacity

Each entity has a **maximum number of equipment slots** it can fill:

```
capacity = DEFAULT_CAPACITY + EQUIP_CAPACITY_SHIFT bonus
```

- **Default capacity**: `1`
- Capacity can be increased via the `EQUIP_CAPACITY_SHIFT` bonus
- Capacity cannot go below 0

When replacing an item in an existing slot, capacity is not consumed (swap).
Capacity is only checked when adding equipment to a **new** slot.

> Source: `LibEquipment.sol:51–53, 215–220`

## Equip Process

`LibEquipment.equip(world, components, holderID, inventoryID, itemIndex)`:

1. **Verify item type** — must be `"EQUIPMENT"`
2. **Get slot** — read the item's `For` component
3. **Check slot** — if occupied, unequip existing item first (no capacity check).
   If new slot, check `equippedCount < capacity`
4. **Consume from inventory** — remove 1 of the item from the inventory
5. **Create equipment instance** — ECS entity linking holder + item + slot
6. **Assign bonuses** — apply the item's `EQUIP` use case bonuses to the holder

Kamis must be in `RESTING` state to equip.

> Source: `LibEquipment.sol:91–124, 171–173`

## Unequip Process

`LibEquipment.unequip(world, components, holderID, inventoryID, slot)`:

1. **Get equipment instance** from the slot
2. **Clear bonuses** — unassign all bonuses with end type `ON_UNEQUIP_{SLOT}`
3. **Remove equipment instance** — delete the ECS entity
4. **Return to inventory** — add 1 of the item back to the inventory

> Source: `LibEquipment.sol:132–155`

## Equipment Instance Shape

| Component | Description |
|---|---|
| `EntityType` | `"EQUIPMENT"` |
| `IDOwnsEquipment` | Holder entity ID (kami or account) |
| `IndexItem` | Which item is equipped |
| `For` | Slot string |

Entity ID: `keccak256("equipment.instance", holderID, slot)` — one per holder per slot.

> Source: `LibEquipment.sol:253–256`

## Bonus Lifecycle

Equipment bonuses use the naming convention `ON_UNEQUIP_{SLOT}` as their end type.
This means:
- On equip: bonuses are assigned as temporary bonuses to the holder
- On unequip: all bonuses tagged with that slot's end type are cleared
- Replacing equipment in the same slot properly clears old and applies new bonuses

The bonus allocation ID is derived deterministically:
```
refAnchor = keccak256("item.usecase", itemIndex)
refID = LibReference.genID("EQUIP", refAnchor)
alloAnchor = keccak256("item.allo", refID)
alloID = LibAllo.genID(alloAnchor, "BONUS", 1)
```

> Source: `LibEquipment.sol:122–123, 147–148, 229–238`
