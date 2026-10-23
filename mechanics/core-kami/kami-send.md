# Kami Send (In-Game Transfer)

> Source: `packages/contracts/src/systems/KamiSendSystem.sol` (L1–88)

## Overview

The Kami Send system allows players to **transfer staked (in-game) Kamis** to
another player without unstaking. This is a direct in-world transfer that
bypasses the marketplace.

## Send Flow

`KamiSendSystem.execute(kamiIndices[], toAddress)`:

1. Reject an empty batch (`"KamiSend: empty batch"`)
2. Resolve the sender's account from the **operator** address that signs
   (`LibAccount.verifyOperator`; see
   [accounts.md → Who May Sign](../world/accounts.md#who-may-sign))
3. Resolve target account via `LibAccount.getByOperator(toAddress)`
   (`KamiSendSystem.sol:45`) — `toAddress` must be the recipient's **operator**
   address. Passing an owner wallet or any unregistered address reverts
   (`"Account: Operator not found"`, `LibAccount.sol:260`)
4. Verify sender is not sending to themselves (`"KamiSend: cannot send to self"`)
5. For each Kami:
   a. Verify Kami exists (`"KamiMarket: kami not found"`), is in `RESTING` or
      `LISTED` state (`"KamiMarket: kami not available"`) and is owned by the
      sender (`"kami not urs"`) — `LibKamiMarket.verifyKamiOwnedRestingOrListed`,
      `LibKamiMarket.sol:376–383`
   b. Verify the Kami is **not soulbound** (`LibSoulbound.verify`,
      `"kami is soulbound"`, `KamiSendSystem.sol:59`) — a Kami bought from the
      [Newbie Vendor](../marketplace/newbie-vendor.md#what-the-3-day-soulbind-blocks)
      cannot be sent for 3 days
   c. If Kami is `LISTED`, cancel all marketplace listings
   d. **Force-unequip all items** back to the sender's inventory
      (`LibEquipment.unequipAll`) — equipment stays with the sender, not the recipient
   e. Reassign ownership to target account
   f. Set state to `RESTING`
   g. Apply **purchase cooldown** (default: 1 hour, from
      `KAMI_MARKET_PURCHASE_COOLDOWN` config)
   h. Log `KAMI_SEND` and emit event

Supports batch sending of multiple Kamis in one transaction. See
[Equipment → Force-Unequip on Ownership Change](../economy/equipment.md#force-unequip-on-ownership-change).

> Source: `KamiSendSystem.sol:39–77`

## Cooldown

A purchase cooldown is applied to each transferred Kami (same cooldown used by
marketplace purchases):

```
cooldown = KAMI_MARKET_PURCHASE_COOLDOWN config (3600s / 1 hour if unset; 0 = none)
LibCooldown.modify(kami, +cooldown)   // extends a cooldown already running
```

For the recipient, the cooldown blocks **harvest start / collect / stop,
item use on the Kami (feeding), and liquidating with it** — all revert
`"kami on cooldown"`. It does **not** block sending the Kami on, listing it,
equipping it or rerolling it. See
[cooldowns.md → What a Cooldown Blocks](../utility/cooldowns.md#what-a-cooldown-blocks).

> Source: `KamiSendSystem.sol:47–50, 70`, `LibCooldown.sol:34–53`
