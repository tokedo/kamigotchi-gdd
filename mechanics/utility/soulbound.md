# Soulbound System

> Source: `packages/contracts/src/libraries/LibSoulbound.sol` (L1–30)

## Overview

The soulbound system provides a **temporary transfer lock** for Kami entities.
Until the lock expires, a soulbound Kami cannot be listed, sold into an offer,
unstaked, sent, rerolled or sacrificed (table below).

## Mechanics

### Setting a Lock

```solidity
LibSoulbound.set(components, kamiID, duration)
```

Stores an expiry timestamp:
```
expiryEntity = keccak256(kamiID, "SOULBOUND")
expiry = block.timestamp + duration
```

The expiry is stored in `ValueComponent` on a derived entity key.

> Source: `LibSoulbound.sol:18–23`

### Verifying (Reverting if Locked)

```solidity
LibSoulbound.verify(components, kamiID)
```

Reverts with `"kami is soulbound"` if `block.timestamp < expiry`.
Equivalently, the source check is `require(block.timestamp >= expiry)`.

If never set, `safeGet` returns 0, and the check passes (`timestamp >= 0` is
always true).

> Source: `LibSoulbound.sol:26–29`

## Usage

| Context | Duration | Purpose |
|---|---|---|
| Newbie Vendor purchase | 3 days | Prevent immediate flip of discounted Kami — the **only** setter |
| Kami721 unstake | Checked | Prevents unstaking while locked |
| Kami marketplace listing | Checked | Cannot list soulbound Kami for sale |
| Kami marketplace offer acceptance | Checked | Cannot sell a soulbound Kami by accepting a direct or collection offer |
| Kami send | Checked | Cannot send a soulbound Kami to another player (per Kami, after the ownership/state check, before its listings are cancelled) |
| Gacha reroll | Checked | Cannot reroll a soulbound Kami (per Kami, after the `RESTING` check) |
| Sacrifice | Checked | Cannot sacrifice a soulbound Kami (after the ownership check, before the `RESTING` check) |

`LibSoulbound.set` is called only from the Newbie Vendor; a Kami obtained any
other way is never soulbound. See
[Newbie Vendor → What the 3-Day Soulbind Blocks](../marketplace/newbie-vendor.md#what-the-3-day-soulbind-blocks).

> Source: `NewbieVendorBuySystem.sol:77`, `Kami721UnstakeSystem.sol:44`,
> `KamiMarketListSystem.sol:33`, `KamiMarketAcceptOfferSystem.sol:66, 141`,
> `KamiSendSystem.sol:59`, `KamiGachaRerollSystem.sol:27`,
> `KamiSacrificeCommitSystem.sol:46`
