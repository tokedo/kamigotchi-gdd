# Item Usage (Use, Cast, Burn, Transfer)

> Source: `packages/contracts/src/systems/KamiUseItemSystem.sol` (L1–64),
> `packages/contracts/src/systems/KamiCastItemSystem.sol` (L1–51),
> `packages/contracts/src/systems/AccountUseItemSystem.sol` (L1–41),
> `packages/contracts/src/systems/ItemBurnSystem.sol` (L1–44),
> `packages/contracts/src/systems/ItemTransferSystem.sol` (L1–45),
> `packages/contracts/src/systems/DroptableRevealSystem.sol` (L1–50)

## Overview

Items can be **used** (applied to a target for effects), **cast** (applied to
enemy Kamis), **burned** (permanently destroyed), or **transferred** between
accounts. Each use case has its own system with distinct validation and
targeting rules.

## Item Target Shapes

Items define which target they can be used on via a `ForShape` component:

| Shape | System | Description |
|---|---|---|
| `"KAMI"` | `KamiUseItemSystem` | Use on your own Kami (e.g., healing snacks) |
| `"ENEMY_KAMI"` | `KamiCastItemSystem` | Cast on another player's Kami |
| `"ACCOUNT"` | `AccountUseItemSystem` | Use on your account |

> Source: `KamiUseItemSystem.sol:38`, `KamiCastItemSystem.sol:27`,
> `AccountUseItemSystem.sol:23`

## Use on Own Kami

`KamiUseItemSystem.execute(kamiID, itemIndex)`:

1. Verify Kami is owned by caller and in same room
2. Verify Kami cooldown has expired
3. Verify item is for shape `"KAMI"` and enabled
4. Verify item USE requirements against the Kami
5. Reset harvest-action bonuses (unless item bypasses this)
6. Sync Kami state (apply pending health regen, etc.)
7. Deduct 1 item from inventory
8. Apply item's allocations to the Kami
9. Reset Kami intensity
10. Log usage

**Special case**: Room 19 ("Temple of the Wheel") is restricted to account
index 833 only.

> Source: `KamiUseItemSystem.sol:19–58`

## Cast on Enemy Kami

`KamiCastItemSystem.execute(targetID, itemIndex)`:

1. Verify target is a Kami and in same room as caster
2. Verify item is for shape `"ENEMY_KAMI"` and enabled
3. Verify item USE requirements against the target
4. Sync caster's account and deplete **10 stamina**
5. Sync target Kami state
6. Deduct 1 item from inventory
7. Apply item's allocations to the target Kami
8. Emit `CAST` event

> Source: `KamiCastItemSystem.sol:18–46`

## Use on Account

`AccountUseItemSystem.execute(itemIndex, amount)`:

1. Verify item is for shape `"ACCOUNT"`
2. Verify item USE requirements against the account
3. Sync account state
4. Deduct `amount` items from inventory
5. Apply item's allocations × amount to the account
6. Log usage

Unlike Kami-targeted items, account items can be used in batches (`amount > 1`).

> Source: `AccountUseItemSystem.sol:18–36`

## Item Effects (Allocations)

When an item is used, `LibItem.applyAllos()` distributes all allocations
attached to the item's `"USE"` action. These can include:

- Stat modifications (healing, buffing)
- Temporary bonuses
- Droptable rolls (commit for later reveal)
- Other items

See [allocations.md](../utility/allocations.md) for the distribution framework.

## Burning

`ItemBurnSystem.execute(indices[], amounts[])`:

1. Verify all items are flagged as **burnable** (`ITEM_BURNABLE` flag)
2. Deduct items from inventory (batch operation)
3. Log burn amounts per item

Burning permanently removes items with no effects applied. Used for quest
turn-ins ("give item" quests).

> Source: `ItemBurnSystem.sol:18–36`

## Transferring

`ItemTransferSystem.execute(indices[], amounts[], targetAccountID)`:

1. Verify arrays match in length
2. Verify all items are flagged as **transferable**
3. Transfer items from sender to target account
4. Deduct **transfer fee**: `15 MUSU per item type transferred`

The fee is charged per distinct item type in the transfer, not per unit.
Currency: MUSU (item index 1).

| Constant | Value | Description |
|---|---|---|
| `MUSU_INDEX` | 1 | MUSU currency item index |
| `TRANSFER_FEE` | 15 | MUSU cost per item type transferred |

> Source: `ItemTransferSystem.sol:15–38`, `LibInventory.sol:24–29`

## Droptable Reveal

`DroptableRevealSystem.execute(commitIDs[])`:

When items or rewards include a droptable allocation, a commit is created.
The reveal system resolves these commits:

1. Verify commits are of type `DROPTABLE_COMMIT`
2. Filter out already-revealed commits
3. Reveal loot using blockhash-based randomness

Includes `forceReveal()` for admin use when the 256-block window is missed
(same pattern as gacha force reveal).

> Source: `DroptableRevealSystem.sol:19–45`
