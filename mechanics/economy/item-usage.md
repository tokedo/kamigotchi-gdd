# Item Usage (Use, Cast, Burn, Transfer)

> Source: `packages/contracts/src/systems/KamiUseItemSystem.sol` (L1–57),
> `packages/contracts/src/systems/KamiCastItemSystem.sol` (L1–59),
> `packages/contracts/src/systems/AccountUseItemSystem.sol` (L1–41),
> `packages/contracts/src/systems/ItemBurnSystem.sol` (L1–44),
> `packages/contracts/src/systems/ItemTransferSystem.sol` (L1–45),
> `packages/contracts/src/systems/DroptableRevealSystem.sol` (L1–50)

## Overview

Items can be **used** (applied to a target for effects), **cast** (applied to
enemy Kamis), **burned** (permanently destroyed), or **transferred** between
accounts. Each use case has its own system with distinct validation and
targeting rules.

**Use and cast are disjoint doors.** `KamiUseItemSystem` is the only self-use
path (it alone applies the cooldown check and the bonus reset);
`KamiCastItemSystem` is strictly an enemy action and rejects your own Kamis
outright.

## Item Target Shapes

Items define which target they can be used on via a `For` component
(`ForComponent`, read through `LibFor`):

| Shape | System | Description |
|---|---|---|
| `"KAMI"` | `KamiUseItemSystem` | Use on your own Kami (e.g., healing snacks) |
| `"ENEMY_KAMI"` | `KamiCastItemSystem` | Cast on another player's Kami |
| `"ANY_KAMI"` | **both** | Accepted by either door — usable on your own Kami *and* castable on another player's |
| `"ACCOUNT"` | `AccountUseItemSystem` | Use on your account |

Each system checks the shape with one of two helpers on `LibItem`:

| Helper | Behaviour |
|---|---|
| `verifyForShape(index, shape)` | Exact match; reverts `"not for {shape}"` |
| `verifyForShapeOr(index, a, b)` | Matches either shape; reverts `"not for {a} or {b}"` |

`KamiUseItemSystem` calls `verifyForShapeOr(itemIndex, "KAMI", "ANY_KAMI")` and
`KamiCastItemSystem` calls `verifyForShapeOr(itemIndex, "ENEMY_KAMI",
"ANY_KAMI")`, so an `ANY_KAMI` item is the only kind that reaches both systems.
`AccountUseItemSystem` still uses the exact-match form.

> Source: `KamiUseItemSystem.sol:31`, `KamiCastItemSystem.sol:35`,
> `AccountUseItemSystem.sol:23`, `LibItem.sol:292–296` (`verifyForShape`),
> `:299–307` (`verifyForShapeOr`)

The only deployed `ANY_KAMI` item is **Flash Talisman (11412)** — see
[items catalog](../../catalogs/items/items.csv).

## Use on Own Kami

`KamiUseItemSystem.execute(kamiID, itemIndex)`:

1. Verify Kami is owned by caller and in same room
2. Verify Kami cooldown has expired
3. Verify item is enabled and for shape `"KAMI"` **or** `"ANY_KAMI"`
4. Verify item USE requirements against the Kami
5. Reset harvest-action bonuses **unless** the item carries the
   `BYPASS_BONUS_RESET` flag (`KamiUseItemSystem.sol:34–37`) — this is the only
   place in the codebase that flag is read; see
   [bonus-system.md → `BYPASS_BONUS_RESET`](../combat/bonus-system.md#bypass_bonus_reset-item-flag)
6. Sync Kami state (apply pending health regen, etc.)
7. Deduct 1 item from inventory
8. Apply item's allocations to the Kami
9. Reset Kami intensity
10. Log usage

> Source: `KamiUseItemSystem.sol:19–51`

## Cast on Enemy Kami

`KamiCastItemSystem.execute(targetID, itemIndex)`:

1. Verify target is a Kami and in same room as caster
2. Verify the target is **not** the caster's own Kami — reverts
   `"cannot cast on own kami"`
3. Verify the target's state is `HARVESTING`
4. Verify item is for shape `"ENEMY_KAMI"` or `"ANY_KAMI"`, that its USE
   requirements hold against the target, and that it is enabled
5. Sync caster's account and deplete **10 stamina**
6. Sync target Kami state
7. Deduct 1 item from inventory
8. Apply item's allocations to the target Kami
9. Log the use as `"ENEMY_KAMI"` and emit the `CAST` event

Three behaviors worth noting:

- **Own Kamis are unreachable through cast** — `checkAccount` is asserted
  false before any item check (`KamiCastItemSystem.sol:29`). Self-targeting is
  rejected outright, so the `"ENEMY_KAMI"` log line stays truthful and there is
  no cast-shaped path around the use-path cooldown and bonus reset. An
  `ANY_KAMI` item used on your own Kami must go through `KamiUseItemSystem`.
- **Targets must be harvesting** — `verifyState(targetID, "HARVESTING")` is an
  explicit gate (`KamiCastItemSystem.sol:32`): resting Kamis are unreachable by
  design, since only a Kami placed on a node is exposed. The `CAST` event
  independently derives the node index from the target's harvest via reverting
  getters (`LibItem.sol:501–503`), so the gate and the event agree.
- **No bonus reset** — unlike the own-Kami path, `KamiCastItemSystem` never
  calls a `LibBonus` resetter, so casting neither clears the target's
  harvest-action buffs nor consults `BYPASS_BONUS_RESET`. Cthonic Blight
  (19201) and Flash Talisman (11412) both carry the flag; on the cast path it
  is inert for both, and it takes effect for Flash Talisman only when the item
  is used on your own Kami.

> Source: `KamiCastItemSystem.sol:18–54`

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
attached to the item's `"USE"` use case via `LibAllo.distribute()`, which
dispatches on each allo's `Type`:

| Allo type | Effect |
|---|---|
| `STAT` | Stat modification (healing, buffing) |
| `BONUS` | Temporary bonuses |
| `ITEM_DROPTABLE` | Droptable roll — creates a commit for later reveal |
| `CLEAR_BONUS` | Clears all of the target's temporary bonuses via `LibBonus.clearAll` (`LibAllo.sol:212`) |
| `DISPLAY_ONLY` | Skipped at distribution — display-only entry (`LibAllo.sol:202, 285–287`) |
| *(any other)* | Basic grant via `LibSetter` (items, currencies, XP, etc.) |

> Source: `LibAllo.sol:188–215`

See [allocations.md](../utility/allocations.md) for the distribution framework.

## Burning

`ItemBurnSystem.execute(indices[], amounts[])`:

1. Verify no items have the `ITEM_UNBURNABLE` flag (items are burnable by default)
2. Deduct items from inventory (batch operation)
3. Log burn amounts per item

Burning permanently removes items with no effects applied. Used for quest
turn-ins ("give item" quests).

> Source: `ItemBurnSystem.sol:18–36`

## Transferring

`ItemTransferSystem.execute(indices[], amounts[], targetAccountID)`:

Unlike every other item system (which resolves the account from the **operator**
address), `ItemTransferSystem` authenticates via
`LibAccount.getByOwner(msg.sender)` — the transfer must be signed by the
account's **owner** wallet (`ItemTransferSystem.sol:21`).

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

1. Verify commits are of type `ITEM_DROPTABLE_COMMIT` (`LibDroptable.sol:38, 137`)
2. Filter out already-revealed commits
3. Reveal loot using blockhash-based randomness

Includes `forceReveal()` for admin use when the 256-block window is missed
(same pattern as gacha force reveal).

> Source: `DroptableRevealSystem.sol:19–45`
