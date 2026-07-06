# Token Portal (ERC-20 Bridge)

> Source: `packages/contracts/src/libraries/LibTokenPortal.sol` (L1–403),
> `packages/contracts/src/systems/TokenPortalSystem.sol` (L1–192),
> `packages/contracts/deployment/world/data/portal/tokens.csv`

## Overview

The Token Portal enables **bridging ERC-20 tokens** between the external
blockchain and the in-game item system. Deposits convert ERC-20 tokens into
game items (with import tax), and withdrawals convert game items back to ERC-20
tokens (with export tax and a time delay).

The portal uses a **receipt system** for withdrawals — items are immediately
removed from the player's inventory, but the ERC-20 transfer is delayed behind
a configurable timelock. Admins can pause or cancel pending withdrawals.

## Enabled / Disabled Toggle

The whole portal is gated by a single `isEnabled` boolean (stored in the
system's local storage, default `false`). An `onlyEnabled` modifier guards
**`deposit`, `withdraw`, `claim`, and `cancel`** — when disabled, all four
revert with `"Token Portal: disabled"`. The owner flips it via
`adminToggleEnabled(bool)`.

> ⚠️ UNCERTAIN: `isEnabled` is mutable runtime state that defaults to `false`
> on deployment and is not pinned by any artifact in this repo — the current
> on-chain value cannot be verified from source.

> Source: `TokenPortalSystem.sol:25, 30–33, 135–137` (`isEnabled`, `onlyEnabled`,
> `adminToggleEnabled`)

## Deposit Flow

`TokenPortalSystem.deposit(itemIndex, itemAmount)`:

1. Look up the token address and scale for the item
2. Calculate import tax:
   ```
   taxAmount = (itemAmount × taxRate) / 10000 + flatTax
   ```
3. Transfer ERC-20 tokens from player to `TokenHolderComponent` (the game's
   token custody contract)
4. Increase player's inventory by `itemAmount - taxAmount`
5. Send tax amount to reserve account (`0x3d7f...2872`)
6. Log deposit and emit `PORTAL_TOKEN_DEPOSIT` event

> Source: `LibTokenPortal.sol:103–128`, `TokenPortalSystem.sol:29–40`

## Withdrawal Flow (3-step)

### Step 1: Initiate Withdrawal

`TokenPortalSystem.withdraw(itemIndex, itemAmount)`:

1. Calculate export tax:
   ```
   taxAmount = (itemAmount × taxRate) / 10000 + flatTax
   ```
2. Calculate token amount after tax and scaling
3. Calculate delay end time: `block.timestamp + PORTAL_TOKEN_EXPORT_DELAY`
4. Create a **Receipt** entity (pending withdrawal)
5. Immediately remove items from player's inventory
6. Send tax amount to reserve account

### Step 2: Claim (after delay)

`TokenPortalSystem.claim(receiptID)`:

1. Verify caller owns the receipt
2. Verify receipt is not paused/disabled
3. Verify delay has elapsed: `block.timestamp >= endTime`
4. Re-read the token address from the Portal's **local registry**
   (`itemAddrs[itemIndex]`), which **overrides** the address stored on the
   receipt; reverts if the item is no longer registered
5. Transfer ERC-20 tokens from `TokenHolderComponent` to player's wallet
6. Remove the receipt entity

> **Token migration semantics**: because claim reads the address from the
> Portal's local registry (not the receipt), an admin can migrate a portal
> item to a new token address via `setItem` — pending receipts then claim
> against the new address. Migration preserves claims **only** in that case.
> Unsetting a portal item (`unsetItem`) makes `deposit`, `withdraw`, **and
> `claim`** revert with `"Token Portal: item not registered"`
> (`TokenPortalSystem.sol:88–89`) — receipts on an unset item cannot be
> claimed until the item is re-registered. Only `cancel` still works, but it
> converts the receipt's token amount using the Portal's current scale entry
> (deleted → `0`), returning a wrong item amount; a dev note in the code
> acknowledges this scale-deletion edge case
> (`TokenPortalSystem.sol:97–98, 104–108`).

### Step 3: Cancel (optional, before claim)

`TokenPortalSystem.cancel(receiptID)`:

1. Verify caller owns the receipt
2. Verify receipt is not paused
3. Return items to player's inventory (no tax refund)
4. Remove the receipt entity

> Source: `LibTokenPortal.sol:131–195`, `TokenPortalSystem.sol:44–96`

## Receipt Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"TOKEN_RECEIPT"` |
| `IDOwnsWithdrawal` | Account that owns this withdrawal |
| `IndexItem` | Item index being withdrawn |
| `TokenAddress` | ERC-20 contract address |
| `Value` | Token amount to transfer (after tax + scaling) |
| `Tax` | Tax amount (in item units, stored for potential refunds) |
| `TimeStart` | Creation timestamp |
| `TimeEnd` | Earliest claim timestamp |

> Source: `LibTokenPortal.sol:66–86`

## Tax System

Taxes are calculated in **basis points** (1/10000):

```
tax = (amount × taxRate) / 10000 + flatTax
```

Config format: `[flatTax, taxRate, ...]`

| Direction | Config Key | Description |
|---|---|---|
| Import (deposit) | `PORTAL_ITEM_IMPORT_TAX` | Tax on tokens entering the game |
| Export (withdrawal) | `PORTAL_ITEM_EXPORT_TAX` | Tax on tokens leaving the game |

Tax is always denominated in the game item's units and sent to the reserve
account.

> Source: `LibTokenPortal.sol:208–222`

## Withdrawal Delay

| Config Key | Value | Description |
|---|---|---|
| `PORTAL_TOKEN_EXPORT_DELAY` | 86400 (1 day) | Time before withdrawal can be claimed |

> Source: `configs.ts:159`, `LibTokenPortal.sol:211–213`

## Token Scale

Each portal item has a **scale** (int32) that converts between game item units
and ERC-20 token units (wei, 18 decimals):

```
tokenUnits = itemUnits × 10^(18 − scale)   (deposit/withdraw: game → token)
gameUnits = tokenUnits / 10^(18 − scale)   (claim/cancel: token → game)
```

For ONYX (scale 2), 1 game unit corresponds to `10^16` wei of the token.
Scale must be 0–18. Negative scales are not supported.

> Source: `LibERC20.sol:42–52` (`toTokenUnits`/`toGameUnits`), applied at
> `LibTokenPortal.sol:120, 147, 174, 194`; scale bounds:
> `TokenPortalSystem.sol:165–166`

## Registered Tokens

| Name | Status | Item Index | Token Address | Scale |
|---|---|---|---|---|
| ONYX | In Game | 100 | `0x4BaD...7CF4` | 2 |
| ETH | Test | 103 | `0xE1Ff...546` | 5 |

> Source: `data/portal/tokens.csv`

## Admin Controls

| Function | Description |
|---|---|
| `adminToggleEnabled(enabled)` | Enables/disables the entire portal (owner only) |
| `adminPause(receiptID)` | Disables a pending withdrawal — blocks both `claim` and the owner's own `cancel` (both check `LibDisabled.verifyEnabled`) |
| `adminUnpause(receiptID)` | Re-enables a paused withdrawal (owner only) |
| `adminCancel(receiptID)` | Force-cancels a withdrawal, returning items to player |

> Source: `TokenPortalSystem.sol:115–137`; enabled checks in `claim`/`cancel`
> at `TokenPortalSystem.sol:82, 102`

## Logging

| Data Key | Scope | Description |
|---|---|---|
| `PORTAL_ITEM_DEPOSIT_TOTAL` | Per account + global | Total items deposited |
| `PORTAL_ITEM_WITHDRAW_TOTAL` | Per account + global | Total items withdrawn |
| `PORTAL_ITEM_CLAIM_TOTAL` | Per account + global | Total items claimed |
| `PORTAL_ITEM_CANCEL_TOTAL` | Per account + global | Total items cancelled |
| `PORTAL_ITEM_TAX_TOTAL` | Per account + global | Total tax collected |
| `PORTAL_TOKEN_DEPOSIT_TOTAL` | Per token address | Token units deposited |
| `PORTAL_TOKEN_WITHDRAW_TOTAL` | Per token address | Token units withdrawn |
| `PORTAL_TOKEN_CLAIM_TOTAL` | Per token address | Token units claimed |
| `PORTAL_TOKEN_CANCEL_TOTAL` | Per token address | Token units cancelled |

> Source: `LibTokenPortal.sol:251–304`

## Config

| Key | Value | Description |
|---|---|---|
| `PORTAL_TOKEN_EXPORT_DELAY` | `86400` (1 day) | Timelock before token withdrawals can be claimed |
| `PORTAL_ITEM_EXPORT_TAX` | `[1, 100]` | Item export tax: 1 flat + 100 basis points (1%) |
| `PORTAL_ITEM_IMPORT_TAX` | `[1, 100]` | Item import tax: 1 flat + 100 basis points (1%) |
| `ERC20_RECEIVER_ADDRESS` | `0x6a2350...1aAd40` | Used only by the deprecated pre-bridge `LibERC20.spend` path (`LibERC20.sol:57–62`) — portal deposits instead transfer tokens to the `TokenHolderComponent` custody contract (`LibTokenPortal.sol:123`) |
| `ONYX_BURNER_ADDRESS` | `0x4A8B41...2465Ec` | Address where burned Onyx tokens are sent |

> Source: `configs.ts:158–176`

### Local/Test Overrides

In test environments, `PORTAL_TOKEN_EXPORT_DELAY` is reduced to `60` seconds
(1 minute) for faster iteration.

> Source: `configs.ts:42–50`
