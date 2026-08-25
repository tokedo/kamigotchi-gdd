# Newbie Vendor

> Source: `packages/contracts/src/systems/NewbieVendorBuySystem.sol` (L1–202),
> `packages/contracts/src/libraries/LibKamiMarketIndex.sol` (L1–111),
> `packages/contracts/src/libraries/LibTWAP.sol` (L1–60)

## Overview

The Newbie Vendor (Zevana) is a **one-time Kami purchase** available to newly
created accounts. It sells Kamis at **110% of the cheapest active
marketplace listing**, clamped below by a configurable minimum price.

> The vendor previously priced off a TWAP oracle fed by marketplace sales.
> That oracle is still maintained — every sale still pokes it — but it is no
> longer read for pricing. See [TWAP Oracle](#twap-oracle-libtwap-no-longer-priced-off)
> below.

The vendor maintains a **rotating pool** of Kami indices. At any given time, 3
Kamis from the pool are displayed for sale. The display window advances every
cycle, wrapping around the pool. When a Kami is bought, it is removed from the
pool.

## Eligibility

A player can buy from the Newbie Vendor if **all** of the following are true:

1. `NEWBIE_VENDOR_ENABLED` config flag is true
2. The account has **not** previously purchased from the vendor
   (`NEWBIE_VENDOR_PURCHASED` flag is unset)
3. The account was created within the last **24 hours**:
   `block.timestamp - accountCreated <= 86400`

After purchase, the `NEWBIE_VENDOR_PURCHASED` flag is set permanently.

> Source: `NewbieVendorBuySystem.sol:53–68`

## Display Pool & Cycling

The vendor holds a pool of Kami indices (stored as `uint256[]` on the
`VENDOR_ENTITY`). At any moment, 3 Kamis are visible (or fewer if the pool has
< 3 entries).

The display window is determined by a cycle counter:

```
cycleNumber = (block.timestamp - cycleStart) / NEWBIE_VENDOR_CYCLE
offset = (cycleNumber × 3) % pool.length
displayIndices = [pool[(offset+0) % len], pool[(offset+1) % len], pool[(offset+2) % len]]
```

- `cycleStart`: `TimeStart` component on `VENDOR_ENTITY`
- `NEWBIE_VENDOR_CYCLE`: config value (cycle duration in seconds)

When a Kami is purchased, it is removed from the pool via swap-with-last and
shrink.

> Source: `NewbieVendorBuySystem.sol:93–123`

## Purchase Flow

`NewbieVendorBuySystem.executeTyped(kamiIndex)`:

1. Verify eligibility (enabled, not purchased, account age ≤ 24h)
2. Calculate price from the marketplace floor (see below)
3. Verify `msg.value >= price`
4. Set `NEWBIE_VENDOR_PURCHASED` flag on account
5. Verify Kami is in the current display window; remove from pool
6. **Soulbind** the purchased Kami for **3 days** (prevents listing, unstaking,
   or accepting offers)
7. Verify vendor account owns the Kami and it is `RESTING`
8. Reassign Kami ownership to buyer
9. Send ETH to the **marketplace fee recipient** (`KAMI_MARKET_FEE_RECIPIENT`),
   falling back to `NEWBIE_VENDOR_ADDRESS` if the fee recipient is unset.
   Vendor (Zevana) proceeds flow to the shared marketplace fee wallet.
10. Refund excess ETH to buyer
11. Emit `NEWBIE_VENDOR_BUY` event

**Soulbind precedes the ETH transfers.** The excess refund at step 10 is a raw
`call` to `msg.sender`, which hands the buyer a re-entrancy callback while the
Kami is already theirs; binding first means an un-soulbound Kami can never be
listed or sent from inside that callback.

> Source: `NewbieVendorBuySystem.sol:50–88` (order),
> `:73–77` (soulbind-before-transfer comment), `:143–147` (refund)

## Price Calculation

```
price = max( ⌊floor × 11000 / 10000⌋ , minPrice )
```

where `floor` is the price of the cheapest eligible marketplace listing. If no
listing is eligible (`floor == 0`), the price is `minPrice` exactly.

| Constant | Value | Meaning |
|---|---|---|
| `FLOOR_PREMIUM_BPS` | `11_000` | 110% of the floor, in basis points |
| `MAX_CONSIDERED_PRICE` | `1_000_000 ether` | Listings above this are ignored |
| `DEFAULT_MIN_PRICE` | `0.004 ether` | Fallback when the config key is unset or 0 |

> Source: `NewbieVendorBuySystem.sol:23–27` (constants), `:152–161`
> (`calcPrice`)

### Minimum Price

```
minPrice = NEWBIE_VENDOR_MIN_PRICE config value
         (falls back to DEFAULT_MIN_PRICE = 0.004 ETH when unset or 0)
```

The clamp is what bounds manipulation: a listing cheap enough to drag the
vendor down is itself a real buyable offer that anyone may take, so the only
cost-free attack is list-and-cancel timing, and its worst case is the vendor
briefly selling at the admin-set floor — one Kami, to one account younger than
24 hours, soulbound for 3 days.

The floor is admin-tunable at run time through
`_NewbieVendorRegistrySystem.setMinPrice(uint256)`, so the live value is chain
state and may differ from the seed in `configs.ts`.

> Source: `NewbieVendorBuySystem.sol:153–154`,
> `_NewbieVendorRegistrySystem.sol:64`

### Cheapest Eligible Listing

`_cheapestListing()` walks the **active-listing index** (see
[Kami Marketplace → Active-Listing Index](kami-market.md#active-listing-index))
and takes the minimum price among entries that pass every filter:

| Filter | Skips |
|---|---|
| `isOrderActive` | Anything not in `State == "ACTIVE"` (filled, cancelled) |
| `isOrderExpired` | Listings whose `TimeEnd` is in the past |
| `getOwner == vendorAccID` | The vendor's own listings |
| `price > MAX_CONSIDERED_PRICE` | Decorative listings above 1,000,000 ETH — this also keeps the ×1.1 premium overflow-safe |

The scan is a single in-transaction loop over the index, which is capped at
100 entries.

> ⚠️ UNCERTAIN: the index is populated going forward from the moment the
> change is deployed; pre-existing listings only enter it through an admin
> `rebuildListingIndex` backfill. Whether that backfill has been run — and so
> whether the vendor is pricing off the true market floor or off an empty
> index (i.e. at `minPrice`) — is chain state and cannot be read from source.

> Source: `NewbieVendorBuySystem.sol:163–183`,
> `LibKamiMarket.sol:283–303` (index accessor + the two order predicates)

## TWAP Oracle (LibTWAP, no longer priced off)

The TWAP accumulator described below is **still written on every marketplace
sale** but is **not read** by `calcPrice`. It stays as a maintained oracle with
no current consumer in the contracts.

The TWAP oracle stores 5 values on `TWAP_ENTITY`:

| Index | Name | Description |
|---|---|---|
| 0 | `cumulativePriceSeconds` | Running sum of price × time |
| 1 | `lastPrice` | Most recent sale price |
| 2 | `lastUpdateTime` | Timestamp of last update |
| 3 | `snapshotCumulative` | Cumulative at start of current window |
| 4 | `snapshotTimestamp` | Timestamp of current window start |


The TWAP accumulator is fed by every marketplace sale via `LibTWAP.poke(price)`:

1. Accumulate time since last update: `cumulativePriceSeconds += lastPrice × elapsed`
2. Update `lastPrice` to the new sale price (if > 0)
3. Set `lastUpdateTime = block.timestamp`
4. If current window has elapsed (`block.timestamp - snapshotTimestamp >= NEWBIE_VENDOR_TWAP_WINDOW`):
   - Roll over snapshot: `snapshotCumulative = cumulativePriceSeconds`
   - Reset window: `snapshotTimestamp = block.timestamp`

The TWAP window is configured via `NEWBIE_VENDOR_TWAP_WINDOW` (expected: 86400 = 1 day).

`poke()` silently returns if the TWAP entity hasn't been initialized.

> Source: `LibTWAP.sol:18–59`

### Data Flow

```
Kami Marketplace Sale ──→ LibTWAP.poke(salePrice) ──→ TWAP accumulator (unread)

Kami Marketplace Listing ──→ active-listing index ──→ calcPrice() ──→ Newbie Vendor
```

Both `KamiMarketBuySystem` and `KamiMarketAcceptOfferSystem` still call
`LibTWAP.poke()` on every successful trade
(`KamiMarketBuySystem.sol:65`, `KamiMarketAcceptOfferSystem.sol:103, 167`);
nothing reads the accumulator back.

## Key Entities

| Entity | ID | Description |
|---|---|---|
| Vendor | `keccak256("newbie.vendor")` | Holds pool of Kami indices + cycle start time |
| TWAP | `keccak256("newbie.vendor.twap")` | Holds 5-value TWAP accumulator data |

## Config

| Key | Value | Description |
|---|---|---|
| `NEWBIE_VENDOR_ENABLED` | `true` | Boolean — enables/disables the vendor |
| `NEWBIE_VENDOR_ADDRESS` | (deployment address) | Vendor account; fallback ETH recipient if no fee recipient set |
| `KAMI_MARKET_FEE_RECIPIENT` | `0x3d7f111B3b69C657624b8633a997A56300212872` | Shared marketplace fee wallet; primary recipient of vendor proceeds |
| `NEWBIE_VENDOR_MIN_PRICE` | `4000000000000000` (0.004 ETH) | Floor price. Lowered from 0.005 to mirror the value already set on chain and to match the in-code `DEFAULT_MIN_PRICE` |
| `NEWBIE_VENDOR_CYCLE` | `172800` (48 hours) | Display rotation period in seconds |
| `NEWBIE_VENDOR_TWAP_WINDOW` | `86400` (24 hours) | TWAP averaging window in seconds (accumulator only — not read for pricing) |

The deploy script also seeds the TWAP accumulator at `0.005 ether`
(`api.vendor.initTWAP`), which no longer affects the sale price.

> Source: `configs.ts:195–201`
