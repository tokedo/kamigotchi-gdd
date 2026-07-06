# Soulbound System

> Source: `packages/contracts/src/libraries/LibSoulbound.sol` (L1–30)

## Overview

The soulbound system provides a **temporary transfer lock** for Kami entities.
When a Kami is soulbound, it cannot be unstaked, listed on the marketplace, or
transferred until the lock expires.

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
| Newbie Vendor purchase | 3 days | Prevent immediate flip of discounted Kami |
| Kami721 unstake | Checked | Prevents unstaking while locked |
| Kami marketplace listing | Checked | Cannot list soulbound Kami for sale |
| Kami marketplace offer acceptance | Checked | Cannot sell a soulbound Kami by accepting a direct or collection offer |

> Source: `NewbieVendorBuySystem.sol:71`, `Kami721UnstakeSystem.sol:44`,
> `KamiMarketListSystem.sol:33`, `KamiMarketAcceptOfferSystem.sol:66, 141`
