# Kami Send (In-Game Transfer)

> Source: `packages/contracts/src/systems/KamiSendSystem.sol` (L1–88)

## Overview

The Kami Send system allows players to **transfer staked (in-game) Kamis** to
another player without unstaking. This is a direct in-world transfer that
bypasses the marketplace.

## Send Flow

`KamiSendSystem.execute(kamiIndices[], toAddress)`:

1. Verify sender has an account (via operator address)
2. Resolve target account from `toAddress`
3. Verify sender is not sending to themselves
4. For each Kami:
   a. Verify Kami is owned by sender and in `RESTING` or `LISTED` state
   b. If Kami is `LISTED`, cancel all marketplace listings
   c. **Force-unequip all items** back to the sender's inventory
      (`LibEquipment.unequipAll`) — equipment stays with the sender, not the recipient
   d. Reassign ownership to target account
   e. Set state to `RESTING`
   f. Apply **purchase cooldown** (default: 1 hour, from
      `KAMI_MARKET_PURCHASE_COOLDOWN` config)
   g. Log `KAMI_SEND` and emit event

Supports batch sending of multiple Kamis in one transaction. See
[Equipment → Force-Unequip on Ownership Change](../economy/equipment.md#force-unequip-on-ownership-change).

> Source: `KamiSendSystem.sol:39–77`

## Cooldown

A purchase cooldown is applied to each transferred Kami (same cooldown used by
marketplace purchases). This prevents immediate re-listing or other actions.

```
cooldown = KAMI_MARKET_PURCHASE_COOLDOWN config (default: 3600s / 1 hour)
```

> Source: `KamiSendSystem.sol:47–49, 66`
