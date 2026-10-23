# Kami Marketplace

> Source: `packages/contracts/src/libraries/LibKamiMarket.sol` (L1–524),
> `packages/contracts/src/libraries/LibKamiMarketIndex.sol` (L1–111),
> `packages/contracts/src/systems/_KamiMarketRegistrySystem.sol` (L1–74),
> `packages/contracts/src/systems/KamiMarketListSystem.sol` (L1–58),
> `packages/contracts/src/systems/KamiMarketBuySystem.sol` (L1–92),
> `packages/contracts/src/systems/KamiMarketOfferSystem.sol` (L1–70),
> `packages/contracts/src/systems/KamiMarketAcceptOfferSystem.sol` (L1–172),
> `packages/contracts/src/systems/KamiMarketCancelSystem.sol` (L1–59),
> `packages/contracts/src/tokens/KamiMarketVault.sol` (L1–56)

## Overview

The Kami marketplace is an on-chain **orderbook** for trading Kamis. It
supports three order types: **listings** (sell for ETH), **specific offers**
(buy a specific Kami for WETH), and **collection offers** (buy any Kami for
WETH). There is no escrow — a listed Kami stays in the seller's **account**,
and WETH stays in the buyer's wallet during offers.

**Kamis never leave the world.** Only in-world Kamis trade: listing requires
`RESTING`, accepting an offer requires `RESTING` or `LISTED`, so an external
(`721_EXTERNAL`) Kami can be neither listed nor sold. A sale reassigns the
Kami's `IDOwnsKami` to the buyer's account — there is no ERC-721 transfer, so
sellers need no Kami721 approval. See [Kami Ownership on a Sale](#kami-ownership-on-a-sale).

**Signers:** buying a listing is signed by the buyer account's **owner**
wallet, which pays the ETH (`KamiMarketBuySystem.sol:29`); listing, offering,
accepting an offer and cancelling are signed by the **operator**. See
[accounts.md → Who May Sign](../world/accounts.md#who-may-sign).

> Source: `LibKamiMarket.sol:368–383` (`verifyKamiResting`,
> `verifyKamiOwnedRestingOrListed`), `KamiMarketListSystem.sol:25–27`,
> `KamiMarketAcceptOfferSystem.sol:55, 63, 121, 140`

The marketplace can be globally enabled/disabled via the `KAMI_MARKET_ENABLED`
config flag.

## Order Types

### Listings (Sell)

Sellers list a specific Kami at a fixed ETH price. The Kami is marked as
`LISTED` state (prevents harvesting, bridging, etc.) but stays in the
seller's account (`IDOwnsKami` unchanged) — no escrow.

| Component | Description |
|---|---|
| `EntityType` | `"KAMI_LISTING"` |
| `State` | `"ACTIVE"` → `"FILLED"` or `"CANCELLED"` |
| `IDOwnsKamiOrder` | Seller's account ID |
| `IndexKamiListing` | Kami token index |
| `Value` | Price in wei (ETH) |
| `TimeStart` | Creation timestamp |
| `TimeEnd` | (optional) Expiry timestamp (0 = never) |

> Source: `LibKamiMarket.sol:49–65`

### Specific Offers (Buy)

Buyers offer WETH for a specific Kami. The WETH is held via approval to the
`KamiMarketVault` contract — no transfer until the offer is accepted.

| Component | Description |
|---|---|
| `EntityType` | `"KAMI_OFFER"` |
| `State` | `"ACTIVE"` → `"FILLED"` or `"CANCELLED"` |
| `IDOwnsKamiOrder` | Buyer's account ID |
| `IndexKami` | Target Kami token index |
| `Value` | Offer price in wei (WETH) |
| `TimeStart` | Creation timestamp |
| `TimeEnd` | (optional) Expiry timestamp |

> Source: `LibKamiMarket.sol:70–86`

### Collection Offers (Buy Any)

Buyers offer WETH per Kami for **any** Kami, up to a specified quantity. Sellers
can accept partially (selling one Kami at a time from the offer).

| Component | Description |
|---|---|
| `EntityType` | `"KAMI_COLLECTION_OFFER"` |
| `State` | `"ACTIVE"` → `"FILLED"` or `"CANCELLED"` |
| `IDOwnsKamiOrder` | Buyer's account ID |
| `Value` | Price per Kami in wei (WETH) |
| `Balance` | Remaining quantity (decrements on each fill) |
| `Max` | Original quantity |
| `TimeStart` | Creation timestamp |
| `TimeEnd` | (optional) Expiry timestamp |

When `Balance` reaches 0, the order is automatically marked as FILLED.

> Source: `LibKamiMarket.sol:89–106`

## Listing Flow

### Creating a Listing

`KamiMarketListSystem.execute(kamiIndex, price, expiry)`:

1. Verify marketplace is enabled
2. Verify Kami is RESTING and owned by the caller
3. Verify Kami is not soulbound (`LibSoulbound.verify`)
4. **Force-unequip all items** back to the seller's inventory (`LibEquipment.unequipAll`)
5. Set Kami state to `LISTED`
6. Create listing entity
7. Emit `KAMI_MARKET_LIST` event, log `KAMI_MARKET_LIST`

> Source: `KamiMarketListSystem.sol:18–44`

### Buying a Listing

`KamiMarketBuySystem.execute(listingIDs)` — supports batch buying:

1. First pass: verify all listings (active, not expired, not self-trade),
   calculate total price
2. Verify `msg.value >= totalPrice`
3. Second pass for each listing:
   a. Reassign Kami ownership to buyer, set state to `RESTING`
   b. Apply purchase cooldown (default: 1 hour)
   c. Calculate fee: `fee = price × numerator / 10^precision` (from `KAMI_MARKET_FEE_RATE`)
   d. Transfer `price - fee` ETH to seller
   e. Feed sale price into TWAP oracle (`LibTWAP.poke`)
4. Transfer total fees to fee recipient
5. Refund excess ETH to buyer

**Payment**: ETH (sent as msg.value). Buyer pays listing price + gas.

> Source: `KamiMarketBuySystem.sol:28–86`

## Offer Flow

### Creating an Offer

`KamiMarketOfferSystem.execute(isCollection, kamiIndex, price, quantity, expiry)`:

- **Specific offer**: targets one Kami by index, quantity=1
- **Collection offer**: targets any Kami, quantity > 0

No WETH is transferred — the offer relies on pre-approval of the
`KamiMarketVault` contract to spend the buyer's WETH.

> Source: `KamiMarketOfferSystem.sol:17–51`

### Accepting an Offer

`KamiMarketAcceptOfferSystem.execute(isBatch, offerID, kamiIndex, kamiIndices)`:

For **specific offers**:
1. Verify seller owns the Kami and it is RESTING or LISTED
2. Verify the Kami is **not soulbound** (`LibSoulbound.verify`)
3. Verify Kami index matches the offer's target
4. If Kami is LISTED, cancel all its listings first
5. **Force-unequip all items** back to the seller's inventory (`LibEquipment.unequipAll`)
6. Reassign Kami ownership, set RESTING, apply cooldown
7. Pull WETH from buyer via vault: `vault.transferWETH(buyer, seller, price - fee)`
8. Transfer fee to fee recipient

For **collection offers** (single or batch):
1. Verify quantity remaining is sufficient
2. For each Kami: verify ownership and that it is **not soulbound**
   (`LibSoulbound.verify`), cancel listings if needed, **force-unequip
   all items to the seller**, reassign
3. Decrement offer balance (auto-fill if balance reaches 0)
4. Batched WETH transfers for efficiency: a single per-Kami fee is computed once
   and multiplied by the batch count (`feePerKami × batchCount`)

The soulbound check (`KamiMarketAcceptOfferSystem.sol:66, 141`) blocks Kamis
still under the [Newbie Vendor](newbie-vendor.md)'s 3-day soulbind from being
sold into offers.

All three fill paths (`fillOffer`, `fillCollectionOffer`, `fillListing`) call
`unequipAll` before reassigning ownership — equipment never transfers with a
sold Kami. See [Equipment → Force-Unequip on Ownership Change](../economy/equipment.md#force-unequip-on-ownership-change).

WETH moves between **owner wallets**: the buyer address the vault pulls from
and the seller address it pays are the two accounts' owners
(`LibAccount.getOwner`), whichever address signed the offer or the
acceptance — so the WETH approval must come from the buyer's owner wallet.

Accept-offer validation now reverts with **custom errors**
(`KamiMarketAcceptKamiMismatch`, `KamiMarketAcceptInvalidOrderType`,
`KamiMarketAcceptEmptyBatch`) instead of string messages.

All sales feed the TWAP oracle via `LibTWAP.poke(price)`.

> Source: `KamiMarketAcceptOfferSystem.sol:54–171`, `LibKamiMarket.sol:145–207`

### Kami Ownership on a Sale

The three fill functions in `LibKamiMarket` move the Kami entirely in-world:

| Step | `fillListing` (buy) | `fillOffer` (specific) | `fillCollectionOffer` |
|---|---|---|---|
| Pre-check | Kami still `LISTED` and still owned by the seller's account (`IDOwnsKami`) | — (the system checked owned + `RESTING`/`LISTED` + not soulbound) | — (same) |
| Open listings | — (the Kami is the listing) | cancelled if `LISTED` | cancelled if `LISTED` |
| Equipment | `unequipAll` → seller | `unequipAll` → seller | `unequipAll` → seller |
| Ownership | `IDOwnsKami` → buyer's account | `IDOwnsKami` → buyer's account | `IDOwnsKami` → buyer's account |
| State | `RESTING` | `RESTING` | `RESTING` |
| Cooldown | purchase cooldown | purchase cooldown | purchase cooldown |
| Order | `FILLED` | `FILLED` | `Balance − 1`; `FILLED` at 0 |

No fill path calls the ERC-721 contract, and none handles a `721_EXTERNAL`
Kami (those cannot reach a fill — see [Overview](#overview)). The buyer
receives the Kami `RESTING`, under the [purchase cooldown](#purchase-cooldown).

> Source: `LibKamiMarket.sol:112–143` (`fillListing`), `:145–172`
> (`fillOffer`), `:174–207` (`fillCollectionOffer`), `:263–277`

## Cancellation

Any active order can be cancelled by its owner:

- **Listing cancel**: Restores Kami to RESTING state (if still LISTED)
- **Offer/collection cancel**: Simply marks as CANCELLED (no funds to return)

Listings are also auto-cancelled when a Kami is transferred via offer acceptance
or other mechanisms (`cancelListingsForKami`).

Additionally, `KamiMarketCancelSystem.executeAdmin(ids[])` lets an admin cancel
**any** active order (batch, no owner check — state is restored on behalf of
the order's owner).

> Source: `LibKamiMarket.sol:209–258`, `KamiMarketCancelSystem.sol:31–38`

## Active-Listing Index

`EntityType` is a bare component, so listings are not enumerable on chain by
type. To make the orderbook's live floor readable inside a transaction, the
marketplace maintains an explicit index: a `uint256[]` of listing IDs stored on
the `ValuesComponent` of a single well-known entity.

| Property | Value |
|---|---|
| Entity | `keccak256("kami.market.listing.index")` |
| Storage | `ValuesComponent` (`uint256[]`) |
| Cap | `MAX_ACTIVE_LISTING_INDEX = 100` |
| Contents | Listing IDs not yet filled or cancelled — **expired ones may linger** |

The index lives in `LibKamiMarketIndex`, a **deployed (linked) library** rather
than inlined code: `KamiMarketAcceptOfferSystem` sits at the EIP-170 contract
size limit, which Yominet enforces on chain, so the order-teardown code was
moved out of system bytecode. It runs via `DELEGATECALL` in the calling
system's context, so component write authority is unchanged.

### Maintenance

| Event | Effect |
|---|---|
| `createListing` | Appends the new listing ID (`LibKamiMarket.sol:67`) |
| Any order settlement (`_cleanup`) | Removes the ID by shifting the tail down — a **tolerant no-op** when the ID is absent, which is the normal case for offers and for listings created before the index existed |
| `rebuildListingIndex(ids[])` | Admin-only wholesale overwrite (backfill or repair) |

**At the cap the index degrades rather than blocks.** When 100 entries are
already held, `add` scans for a *dead* entry — one whose state is not `ACTIVE`,
or whose `TimeEnd` has passed — and reuses that slot. If every slot is
genuinely live, the newcomer is simply **not indexed**: the listing still
exists and still trades, it is only invisible to floor pricing until a rebuild
re-syncs it. Listing creation never fails because the index is full.

**Cost.** Every list, fill and cancel rewrites the whole array, so index
maintenance is O(N) storage writes. The 100-entry cap is what keeps both that
rewrite and the vendor's in-transaction floor scan inside Yominet's per-tx gas
lane.

### `rebuildListingIndex`

`_KamiMarketRegistrySystem.rebuildListingIndex(uint256[] ids)` is `onlyAdmin`
and overwrites the index wholesale. It validates every entry first:

1. `ids.length <= MAX_ACTIVE_LISTING_INDEX` — else `"exceeds index cap"`
2. each ID is a `KAMI_LISTING` shape — else `"not a listing"`
3. each ID is in `State == "ACTIVE"` — else `"not active"`
4. no ID appears twice — else `"duplicate id"`. Duplicates would leave phantom
   entries, because index removal drops only the first occurrence.

The only consumer of the index today is the
[Newbie Vendor](newbie-vendor.md#cheapest-eligible-listing), which reads it
through `LibKamiMarket.getListingIndex` and filters expired, vendor-owned and
absurdly-priced entries at read time.

> Source: `LibKamiMarketIndex.sol:22–110` (entity, cap, `add`, `_isDead`,
> `cleanup`), `LibKamiMarket.sol:67` (append), `:283–303` (accessors and the
> `isOrderActive` / `isOrderExpired` predicates), `:319–321` (`_cleanup`
> delegating to the library), `_KamiMarketRegistrySystem.sol:31–51`

## Fee Calculation

```
fee = price × numerator / 10^precision
```

Fee parameters come from `KAMI_MARKET_FEE_RATE` config array: `[precision, numerator, ...]`

Fees are paid in the same currency as the trade (ETH for listings, WETH for
offers) and sent to the configured `KAMI_MARKET_FEE_RECIPIENT` address.

> Source: `LibKamiMarket.sol:326–331`

## Purchase Cooldown

After any Kami purchase, a cooldown is applied to the Kami:

```
cooldown = KAMI_MARKET_PURCHASE_COOLDOWN (3600 seconds / 1 hour if unset; 0 = none)
LibCooldown.modify(kami, +cooldown)   // extends a cooldown already running
```

The key is admin-set via `_KamiMarketRegistrySystem.setPurchaseCooldown`; no
init script sets it. For the buyer, the cooldown blocks harvesting with the
Kami, using items on it and liquidating with it (`"kami on cooldown"`); it
does **not** block re-listing, sending or equipping it. See
[cooldowns.md → What a Cooldown Blocks](../utility/cooldowns.md#what-a-cooldown-blocks).

> Source: `LibKamiMarket.sol:271–277`, `_KamiMarketRegistrySystem.sol:70–73`

## KamiMarketVault

The `KamiMarketVault` is a persistent relay contract for **WETH**. Buyers
approve the vault to spend their WETH, and authorized systems call
`transferWETH(from, to, amount)` when an offer is accepted.

The contract also carries `transferKami` (an ERC-721 `transferFrom`) and a
header comment saying sellers approve it for Kamis, but **no system calls
`transferKami` at the pin** — sales reassign `IDOwnsKami` instead (see
[Kami Ownership on a Sale](#kami-ownership-on-a-sale)), so sellers need no
Kami721 approval. Upstream's `packages/contracts/docs/KamiMarketplace.md`,
rewritten at `04cb25f8`, describes the same in-world model.

The vault's address persists across system upgrades, meaning buyer approvals
remain valid even when marketplace system contracts are redeployed; the owner
re-authorizes each redeployed system (`authorizeCaller`).

> Source: `KamiMarketVault.sol:1–56` (`transferWETH` `:37–39`, unused
> `transferKami` `:43–45`, `onlyAuthorized` `:25–28`, `authorizeCaller`
> `:47–50`), `KamiMarketAcceptOfferSystem.sol:94–99, 160–164` (the only vault
> calls)

## TWAP Integration

Every marketplace sale (listing buy or offer acceptance) feeds the sale price
into the TWAP oracle via `LibTWAP.poke(price)`. Nothing reads the accumulator
back: the [Newbie Vendor](newbie-vendor.md#price-calculation) prices off the
cheapest active listing, not the TWAP.

> Source: `KamiMarketBuySystem.sol:65`, `KamiMarketAcceptOfferSystem.sol:103, 167`

## Logging

| Data Key | Description |
|---|---|
| `KAMI_MARKET_LIST` | Listings created |
| `KAMI_MARKET_BUY` | Listings purchased |
| `KAMI_MARKET_OFFER` | Offers created |
| `KAMI_MARKET_ACCEPT` | Offers accepted |
| `KAMI_MARKET_CANCEL` | Orders cancelled |

> Source: `LibKamiMarket.sol:420–438`
