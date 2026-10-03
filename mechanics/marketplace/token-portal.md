# Token Portal (ERC-20 Bridge)

> Source: `packages/contracts/src/libraries/LibTokenPortal.sol` (L1–416),
> `packages/contracts/src/systems/TokenPortalSystem.sol` (L1–270),
> `packages/contracts/src/libraries/utils/LibERC20.sol` (L40–52),
> `packages/contracts/deployment/world/data/portal/tokens.csv`,
> `packages/contracts/deployment/world/state/configs/configs.ts` (L37–51, L158–176)

## Overview

The Token Portal enables **bridging ERC-20 tokens** between the external
blockchain and the in-game item system. Deposits convert ERC-20 tokens into
game items (with import tax), and withdrawals convert game items back to ERC-20
tokens (with export tax and a time delay).

The portal uses a **receipt system** for withdrawals — items are immediately
removed from the player's inventory, but the ERC-20 transfer is delayed behind
a configurable timelock. Admins can pause or cancel pending withdrawals.

Two tokens are registered (see [Registered Tokens](#registered-tokens)):

| Item | Index | Token | Portal scale | 1 item unit = |
|---|---|---|---|---|
| Onyx Shard | 100 | ONYX | 2 | 0.01 ONYX (`10^16` wei) |
| Ether Shard | 103 | ETH | 5 | 0.00001 ETH (`10^13` wei); 1 ETH = 100,000 shards |

The portal is the **only** way a bridged token enters the item space. Once
deposited, an Onyx Shard or Ether Shard is an ordinary fungible item and can be
traded, spent, or supplied as pool liquidity like any other — see
[Item Pools](item-pools.md). Pools never mint, so shards held in a pool
reserve remain backed one-for-one by tokens in portal custody.

Withdrawals have two payout routes: the **owner lane** (`withdraw`, paid to the
account's owner wallet) and the **operator lane** (`withdrawToOperator`, paid
to the account's operator wallet, enabled per item). See
[Operator Lane](#operator-lane).

## Enabled / Disabled Toggle

The whole portal is gated by a single `isEnabled` boolean (stored in the
system's local storage, default `false`). An `onlyEnabled` modifier guards
**`deposit`, `withdraw`, `withdrawToOperator`, `claim`, and `cancel`** — when
disabled, all five revert with `"Token Portal: disabled"`. The owner flips it
via `adminToggleEnabled(bool)`.

> ⚠️ UNCERTAIN: `isEnabled` is mutable runtime state that defaults to `false`
> on deployment and is not pinned by any artifact in this repo — the current
> on-chain value cannot be verified from source.

> Source: `TokenPortalSystem.sol:30, 35–38, 178–180` (`isEnabled`,
> `onlyEnabled`, `adminToggleEnabled`)

## Player Entrypoints and Signers

All five player entrypoints live on `TokenPortalSystem` (system ID
`keccak256("system.erc20.portal")`).

| Entrypoint | Who may sign | Account resolved by |
|---|---|---|
| `deposit(itemIndex, itemAmt)` | Account **owner** only | `LibAccount.getByOwner(msg.sender)` |
| `withdraw(itemIndex, itemAmt)` | Account **owner** only | `LibAccount.getByOwner(msg.sender)` |
| `withdrawToOperator(itemIndex, itemAmt)` | Account **owner or operator** | Own account first: if the sender is itself an account owner, that account; otherwise the account that names the sender as operator |
| `claim(receiptID)` | Owner receipts: the receipt account's **owner**. Operator-lane receipts: that account's owner **or its current operator** | The receipt's `IDOwnsWithdrawal` |
| `cancel(receiptID)` | Same rule as `claim` | The receipt's `IDOwnsWithdrawal` |

A non-owner calling `deposit`/`withdraw` reverts `"Account: no account
detected"`; a `withdrawToOperator` sender that is neither an owner nor a
registered operator reverts `"Account: Operator not found"`. For `claim` and
`cancel`, a caller outside the rule above — or a receipt that does not exist,
including one already settled — reverts `"not receipt owner"`.

The own-account-first rule in `withdrawToOperator` means an address that owns
an account is never routed to a different account that merely names it as
operator.

> Source: `TokenPortalSystem.sol:14, 42–53, 57–79, 84–106, 113–151, 238–261`;
> `LibAccount.sol:253–267` (`getByOperator`, `getByOwner` reverts)

## Deposit Flow

`TokenPortalSystem.deposit(itemIndex, itemAmt)`:

1. Resolve the account from the owner wallet (`msg.sender`)
2. Look up the token address and scale for the item in the portal's local
   registry; revert `"Token Portal: item not registered"` if absent
3. Calculate import tax:
   ```
   taxAmt = (itemAmt × taxRate) / 10000 + flatTax
   ```
   Revert `"TokenPortal: tax exceeds item amount"` unless `taxAmt < itemAmt`
4. Pull `toTokenUnits(itemAmt, scale)` tokens from the **owner wallet** into
   `TokenHolderComponent` (the game's token custody contract)
5. Increase the account's inventory by `itemAmt − taxAmt`
6. Credit `taxAmt` items to the reserve account (`0x3d7f…2872`)
7. Log the deposit and emit `PORTAL_TOKEN_DEPOSIT`

`itemAmt` is denominated in **items** (shards), not tokens: depositing 1 ETH
means `itemAmt = 100,000` Ether Shards.

> Source: `TokenPortalSystem.sol:42–53`, `LibTokenPortal.sol:28–29, 106–131`

## Withdrawal Flow (3-step)

### Step 1: Initiate Withdrawal

`TokenPortalSystem.withdraw(itemIndex, itemAmt)` (owner lane) or
`withdrawToOperator(itemIndex, itemAmt)` (operator lane):

1. Resolve the account (see [signers](#player-entrypoints-and-signers)) and
   check the item is registered on the portal (`"Token Portal: item not
   registered"`); `withdrawToOperator` additionally requires the item to be on
   the operator lane (`"Token Portal: item not on the operator lane"`)
2. Calculate export tax on the gross amount:
   ```
   taxAmt = (itemAmt × taxRate) / 10000 + flatTax
   ```
   Revert `"TokenPortal: tax exceeds item amount"` unless `taxAmt < itemAmt`
3. Token amount owed: `tokenAmt = toTokenUnits(itemAmt − taxAmt, scale)`
4. Delay end time: `endTime = block.timestamp + PORTAL_TOKEN_EXPORT_DELAY`
5. Create a **Receipt** entity (pending withdrawal) holding `tokenAmt` and
   `endTime`
6. Remove the full `itemAmt` from the account's inventory and credit `taxAmt`
   items to the reserve account
7. Log and emit `PORTAL_TOKEN_WITHDRAW`
8. Operator lane only: set the receipt's `PORTAL_TO_OPERATOR` flag

`endTime` is written onto the receipt at creation, so a later change to
`PORTAL_TOKEN_EXPORT_DELAY` does not move the end time of receipts already
pending.

> Source: `TokenPortalSystem.sol:57–106`, `LibTokenPortal.sol:69–89, 134–159,
> 219–221`

### Step 2: Claim (after delay)

`TokenPortalSystem.claim(receiptID)` runs its checks in this order:

1. **Caller** — resolve the receipt's account; apply the
   [signer rule](#player-entrypoints-and-signers) (`"not receipt owner"`)
2. **Not paused** — `LibDisabled.verifyEnabled` (`"entity not enabled"`)
3. **Delay elapsed** — `block.timestamp ≥ endTime` (`"withdrawal not ready"`)
4. **Item index** — the receipt's item index must be nonzero (`"Item Registry:
   item not registered"`)
5. **Portal registration** — re-read the token address from the portal's
   **local registry** (`itemAddrs[itemIndex]`), which **overrides** the
   address stored on the receipt; revert `"Token Portal: item not
   registered"` if the item is no longer registered
6. **Payout address** — owner receipt: the account's owner wallet. Operator-
   lane receipt: the account's operator **as it stands at claim time**; revert
   `"Token Portal: no operator"` if that operator is the zero address
7. **Settle** — delete the receipt entity, **then** transfer `tokenAmt` from
   `TokenHolderComponent` to the payout address; log and emit
   `PORTAL_TOKEN_CLAIM`

Because the receipt is removed before the token transfer, and a removed
receipt fails the caller check in step 1, a receipt can be claimed at most
once.

> **Token migration semantics**: because claim reads the address from the
> Portal's local registry (not the receipt), an admin can migrate a portal
> item to a new token address via `setItem` — pending receipts then claim
> against the new address. Migration preserves claims **only** in that case.
> Unsetting a portal item (`unsetItem`) makes `deposit`, `withdraw`,
> `withdrawToOperator`, **and `claim`** revert with `"Token Portal: item not
> registered"` (`TokenPortalSystem.sol:121–122`) — receipts on an unset item
> cannot be claimed until the item is re-registered. Only `cancel` still
> works, but it converts the receipt's token amount using the Portal's current
> scale entry (deleted → `0`), returning a wrong item amount; a dev note in the
> code acknowledges this scale-deletion edge case
> (`TokenPortalSystem.sol:111–112, 139–140, 148–149`).

> Source: `TokenPortalSystem.sol:113–136`, `LibTokenPortal.sol:166–188,
> 245–247, 254–257`

### The Pending Queue Is Public

A receipt is an ordinary world entity, so the set of unclaimed withdrawals is
readable by anyone — it is not scoped to its owner. The client's portal view
splits into a Deposit tab and a Withdraw tab with a toggleable queue panel
beneath them, and that panel lists **other players' open withdrawals** next to
your own, labelling operator-lane receipts as such. A pending exit is therefore
publicly observable for the whole `PORTAL_TOKEN_EXPORT_DELAY` window before it
can be claimed.

> Source: `packages/client/src/app/components/modals/tokenPortal/TokenPortal.tsx`
> (`getOpenWithdrawals` with an empty account filter, minus the caller's own;
> per-receipt lane lookup), `queue/table/Table.tsx` (mine / others' split)

### Step 3: Cancel (optional, before claim)

`TokenPortalSystem.cancel(receiptID)`:

1. **Caller** — same signer rule as `claim` (`"not receipt owner"`)
2. **Not paused** — `"entity not enabled"`
3. **Item index** — nonzero (`"Item Registry: item not registered"`)
4. Return `toGameUnits(tokenAmt, scale)` items — that is, `itemAmt − taxAmt` —
   to the account's inventory. The export tax is **not** refunded
5. Delete the receipt entity; log and emit `PORTAL_TOKEN_CANCEL`

Cancel has no delay check: a receipt can be cancelled at any time before it is
claimed.

> Source: `TokenPortalSystem.sol:141–151`, `LibTokenPortal.sol:192–212`

## Operator Lane

`withdrawToOperator(itemIndex, itemAmt)` lets an account turn a bridged item
into tokens in its **operator** wallet — the address that signs day-to-day
gameplay — without the owner wallet taking part.

- **Same economics as `withdraw`**: same export tax, same delay, same receipt
  shape, same data logs and events. The only difference is the
  `PORTAL_TO_OPERATOR` flag on the receipt.
- **The flag**: a bare flag (`LibFlag.set`) — a separate entity
  `keccak256("has.flag", receiptID, "PORTAL_TO_OPERATOR")` carrying `HasFlag`.
  Settling a receipt removes only the receipt's own components, so the flag
  survives claim and cancel and a settled receipt's payout route stays
  readable.
- **Payout resolved at claim time**: the destination is the account's
  operator when `claim` executes, not when the withdrawal was created.
  Rotating the operator during the delay redirects the payout to the new
  operator; setting the operator to the zero address makes `claim` revert
  `"Token Portal: no operator"` (the receipt can still be cancelled).
- **Who settles**: the account owner or its current operator may `claim` or
  `cancel` a lane receipt. Owner receipts stay owner-only and pay the owner.
- **Per-item gate**: `laneItems[itemIndex]` (public getter) must be `true`.
  The owner sets it with `setLaneItem(index, enabled)`, which requires the
  item to be registered on the portal; `unsetItem` clears it. The bit lives in
  the system's local storage, so it is chain state, not source data.
- **Admin controls apply unchanged**: pause, admin cancel and the portal-wide
  switch cover lane receipts exactly as they cover owner receipts.

> ⚠️ UNCERTAIN: which items are on the operator lane is runtime state
> (`laneItems`), not pinned by any checked-in artifact. The client offers the
> operator route only for the ETH token (its `isGasToken` check against the
> client's ETH address), and labels that token as the chain's gas token.

> Source: `TokenPortalSystem.sol:16–17, 29, 84–106, 113–151, 183–186, 232,
> 248–261`; `LibFlag.sol:37–45, 66–69, 177–179`; `LibTokenPortal.sol:91–100`;
> client `TokenPortal.tsx:112–113, 301–303`, `constants/tokens.ts:3–12`

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

Operator-lane receipts additionally have the `PORTAL_TO_OPERATOR` flag
entity described above (not a component on the receipt itself).

> Source: `LibTokenPortal.sol:69–100`, `TokenPortalSystem.sol:104`

## Tax System

Taxes are calculated in **basis points** (1/10000) plus a flat amount:

```
tax = (amount × taxRate) / 10000 + flatTax
```

Config format: `[flatTax, taxRate, ...]`

| Direction | Config Key | Applied to |
|---|---|---|
| Import (deposit) | `PORTAL_ITEM_IMPORT_TAX` | The deposited item amount |
| Export (withdrawal) | `PORTAL_ITEM_EXPORT_TAX` | The gross withdrawn item amount |

Tax is always denominated in the game item's units and credited to the
reserve account. The flat part is **one item unit of whichever item is
bridged**: 1 Onyx Shard (0.01 ONYX) or 1 Ether Shard (0.00001 ETH). Both
directions and both lanes share the same two configs; there are no per-item
rates.

Worked example at the production values `[1, 50]`, bridging 1 ETH:

```
deposit  100,000 shards:  tax = 100,000 × 50 / 10,000 + 1 = 501  →  99,499 shards credited
withdraw 100,000 shards:  tax = 501  →  receipt for 99,499 × 10^13 wei = 0.99499 ETH
```

> Source: `LibTokenPortal.sol:225–239`, `configs.ts:158–168`

## Withdrawal Delay

| Config Key | Value | Description |
|---|---|---|
| `PORTAL_TOKEN_EXPORT_DELAY` | 43200 (12 hours) | Time before a withdrawal can be claimed |

> Source: `configs.ts:159`, `LibTokenPortal.sol:148, 219–221`

## Token Scale

Each portal item has a **scale** (int32) that converts between game item units
and ERC-20 token units (wei, 18 decimals):

```
tokenUnits = itemUnits × 10^(18 − scale)   (deposit/withdraw: game → token)
gameUnits = tokenUnits / 10^(18 − scale)   (claim/cancel: token → game)
```

For ONYX (scale 2), 1 game unit corresponds to `10^16` wei of the token. For
ETH (scale 5), 1 game unit corresponds to `10^13` wei, so 1 ETH = 100,000
Ether Shards. Scale must be 0–18. Negative scales are not supported.

> Source: `LibERC20.sol:42–52` (`toTokenUnits`/`toGameUnits`), applied at
> `LibTokenPortal.sol:120, 147, 177, 202`; scale bounds:
> `TokenPortalSystem.sol:214–215`

## Registered Tokens

| Name | Status | Item Index | Token Address | Scale |
|---|---|---|---|---|
| ONYX | In Game | 100 | `0x4BaDFb501Ab304fF11217C44702bb9E9732E7CF4` | 2 |
| ETH | In Game | 103 | `0xE1Ff7038eAAAF027031688E1535a055B2Bac2546` | 5 |

The items themselves are catalogued in `catalogs/items/items.csv` (100 Onyx
Shard, 103 Ether Shard, both type `ERC20`).

> Source: `data/portal/tokens.csv`

## Admin Controls

| Function | Description |
|---|---|
| `adminToggleEnabled(enabled)` | Enables/disables the entire portal (owner only) |
| `adminPause(receiptID)` | Disables a pending withdrawal — blocks both `claim` and `cancel` (both check `LibDisabled.verifyEnabled`) |
| `adminUnpause(receiptID)` | Re-enables a paused withdrawal (owner only) |
| `adminCancel(receiptID)` | Force-cancels a withdrawal (either lane), returning `itemAmt − taxAmt` items to the account; reverts `"Token Portal: no receipt"` if the receipt does not exist; clears any pause first |
| `setLaneItem(index, enabled)` | Adds or removes an item on the operator lane (owner only; item must be registered) |
| `initItem(index)` / `setItem(index, tokenAddr, scale)` / `unsetItem(index)` | Register, re-register after a system redeploy, or remove a portal item (owner only); `unsetItem` also clears the lane bit |

> Source: `TokenPortalSystem.sol:157–186, 193–233`; enabled checks in
> `claim`/`cancel` at `TokenPortalSystem.sol:115, 143`

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

| World event | Fields |
|---|---|
| `PORTAL_TOKEN_DEPOSIT` | `ts`, `accID`, `itemIndex`, `itemAmt`, `taxAmt`, `token`, `tokenAmt` |
| `PORTAL_TOKEN_WITHDRAW` | `ts`, `accID`, `receiptID`, `itemIndex`, `itemAmt`, `taxAmt`, `token`, `tokenAmt` |
| `PORTAL_TOKEN_CLAIM` | `ts`, `accID`, `receiptID` |
| `PORTAL_TOKEN_CANCEL` | `ts`, `accID`, `receiptID` |

Both lanes emit the same events; nothing in an event distinguishes an
operator-lane withdrawal — the route is readable only from the receipt's
`PORTAL_TO_OPERATOR` flag.

> Source: `LibTokenPortal.sol:273–326` (data), `:332–415` (events)

## Config

| Key | Value | Description |
|---|---|---|
| `PORTAL_TOKEN_EXPORT_DELAY` | `43200` (12 hours) | Timelock before token withdrawals can be claimed |
| `PORTAL_ITEM_EXPORT_TAX` | `[1, 50]` | Item export tax: 1 flat item + 50 basis points (0.5%) |
| `PORTAL_ITEM_IMPORT_TAX` | `[1, 50]` | Item import tax: 1 flat item + 50 basis points (0.5%) |
| `ERC20_RECEIVER_ADDRESS` | `0x6a2350...1aAd40` | Used only by the deprecated pre-bridge `LibERC20.spend` path (`LibERC20.sol:57–62`) — portal deposits instead transfer tokens to the `TokenHolderComponent` custody contract (`LibTokenPortal.sol:123`) |
| `ONYX_BURNER_ADDRESS` | `0x4A8B41...2465Ec` | Address where burned Onyx tokens are sent |

> Source: `configs.ts:158–176`

### Local/Test Overrides

In test environments, `PORTAL_TOKEN_EXPORT_DELAY` is reduced to `60` seconds
(1 minute) for faster iteration; the tax arrays match production (`[1, 50]`).

> Source: `configs.ts:37–51`
