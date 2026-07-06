# Newbie Vendor

> Source: `packages/contracts/src/systems/NewbieVendorBuySystem.sol` (L1–189),
> `packages/contracts/src/libraries/LibTWAP.sol` (L1–60)

## Overview

The Newbie Vendor is a **one-time Kami purchase** available to newly created
accounts. It sells Kamis at a price derived from a **TWAP (Time-Weighted Average
Price) oracle** fed by marketplace sales, with a configurable minimum floor.

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

> Source: `NewbieVendorBuySystem.sol:45–60`

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

> Source: `NewbieVendorBuySystem.sol:83–113`

## Purchase Flow

`NewbieVendorBuySystem.executeTyped(kamiIndex)`:

1. Verify eligibility (enabled, not purchased, account age ≤ 24h)
2. Calculate price from TWAP oracle (see below)
3. Verify `msg.value >= price`
4. Set `NEWBIE_VENDOR_PURCHASED` flag on account
5. Verify Kami is in the current display window; remove from pool
6. Verify vendor account owns the Kami and it is `RESTING`
7. Reassign Kami ownership to buyer
8. Send ETH to the **marketplace fee recipient** (`KAMI_MARKET_FEE_RECIPIENT`),
   falling back to `NEWBIE_VENDOR_ADDRESS` if the fee recipient is unset.
   Vendor (Zevana) proceeds now flow to the shared marketplace fee wallet.
9. Refund excess ETH to buyer
10. **Soulbind** the purchased Kami for **3 days** (prevents listing, unstaking,
    or accepting offers)
11. Emit `NEWBIE_VENDOR_BUY` event

> Source: `NewbieVendorBuySystem.sol:42–78`

## Price Calculation

The vendor price is `max(twapPrice, minPrice)`.

### Minimum Price

```
minPrice = NEWBIE_VENDOR_MIN_PRICE config value
         (defaults to 0.005 ETH if not set)
```

> Source: `NewbieVendorBuySystem.sol:140–142`

### TWAP Price

The TWAP oracle stores 5 values on `TWAP_ENTITY`:

| Index | Name | Description |
|---|---|---|
| 0 | `cumulativePriceSeconds` | Running sum of price × time |
| 1 | `lastPrice` | Most recent sale price |
| 2 | `lastUpdateTime` | Timestamp of last update |
| 3 | `snapshotCumulative` | Cumulative at start of current window |
| 4 | `snapshotTimestamp` | Timestamp of current window start |

TWAP calculation:

```
windowTime = block.timestamp - snapshotTimestamp

liveCumulative = cumulativePriceSeconds + lastPrice × (block.timestamp - lastUpdateTime)

twapPrice = (liveCumulative - snapshotCumulative) / windowTime
```

If `windowTime == 0` (just after snapshot), falls back to `lastPrice`.

If the TWAP entity doesn't exist or has wrong data length, returns `minPrice`.

> Source: `NewbieVendorBuySystem.sol:140–167`

## TWAP Oracle (LibTWAP)

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

### TWAP Data Flow

```
Kami Marketplace Sale ──→ LibTWAP.poke(salePrice) ──→ TWAP accumulator
                                                          │
Newbie Vendor ←── calcPrice() reads TWAP ←────────────────┘
```

Both `KamiMarketBuySystem` and `KamiMarketAcceptOfferSystem` call
`LibTWAP.poke()` on every successful trade.

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
| `NEWBIE_VENDOR_MIN_PRICE` | `5000000000000000` (0.005 ETH) | Floor price |
| `NEWBIE_VENDOR_CYCLE` | `172800` (48 hours) | Display rotation period in seconds |
| `NEWBIE_VENDOR_TWAP_WINDOW` | `86400` (24 hours) | TWAP averaging window in seconds |

> Source: `configs.ts:195–201`
