# Auctions (GDA)

> Source: `packages/contracts/src/libraries/LibAuction.sol` (L1–150),
> `packages/contracts/src/libraries/LibAuctionRegistry.sol` (L1–163),
> `packages/contracts/src/libraries/utils/LibGDA.sol` (L1–51),
> `packages/contracts/src/systems/AuctionBuySystem.sol` (L1–61),
> `packages/contracts/src/systems/_AuctionRegistrySystem.sol` (L1–100),
> `packages/contracts/src/components/TimePauseComponent.sol` (L1–11),
> `packages/contracts/deployment/world/data/auctions/auctions.csv`

## Overview

Auctions are **global item sales** using a **Discrete Gradual Dutch Auction
(GDA)** pricing model. Unlike NPC shop listings, auctions are not merchant-run
— they are standalone, system-level mechanisms for distributing items at
dynamically adjusting prices.

Each auction sells a specific item in exchange for a specific currency item.
The price starts high and decays over time, but increases with each purchase.
This creates a VRGDA (Variable Rate Gradual Dutch Auction) equilibrium where
the price self-adjusts based on demand.

## Auction Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"AUCTION"` |
| `IndexItem` | Item being sold |
| `IndexCurrency` | Payment item accepted |
| `Value` | Target price (initial/reference price) |
| `Max` | Total supply available for auction |
| `Balance` | Number of units sold so far |
| `Period` | Reference time period in seconds |
| `Decay` | Price decay factor per period (stored as int32, ×1e6) |
| `Rate` | Target sales per period to counteract decay |
| `TimeStart` | Auction start timestamp |
| `TimePause` | Pause window the price skips, packed `uint64 start << 64 \| uint64 end`; optional, unset means none (see [Pause Windows](#pause-windows)) |

Auction ID: `keccak256("auction", itemIndex)`

Only one auction exists per item (at most).

`create` writes every component above except `TimePause`; only `setPause`
writes the window.

> Source: `LibAuctionRegistry.sol:37–69` (shape comment, `create`), `:48–49`
> (`TimePause` in the shape comment), `:91–96` (`setPause`), `:145–154`
> (`packPause` / `unpackPause`); `TimePauseComponent.sol:6–11`

## Price Calculation

The auction price uses the GDA formula (see
[NPC shops](../economy/npc-shops.md) for the underlying `LibGDA.calc`):

```
spotPrice = targetPrice × decay^(timeDelta) / decay^(prevSold / rate)
batchCost = geometric series summation over qty units at spotPrice
```

Where:
- `timeDelta = (block.timestamp - priceStart) / period`, where `priceStart`
  is `startTs` pushed forward past any pause window (see
  [Price Start](#price-start)); with no window, `priceStart = startTs`
- `prevSold = balance` (total sales so far)
- `decay` is stored as `int32 × 1e6`, then multiplied by `1e12` to reach WAD
  precision for LibGDA

The result is in WAD (×1e18) and rounded up; a non-positive result reverts
`"LibAuction: negative GDA cost"`:
```
cost = ceil(LibGDA.calc(params) / 1e18)
```

**Auction decay is unbounded.** `LibAuction` calls `LibGDA.calc` directly with
no deficit clamp, so a dormant auction's price decays toward zero without
limit and there is no per-unit minimum. This is deliberate — the clamp and the
price floor live in `LibListing`, which shares the same GDA library, so NPC
shop listings are floored while auctions are not. See
[NPC shops → Price Floor](../economy/npc-shops.md#price-floor-deficit-clamp).

> Source: `LibAuction.sol:44–58`, `LibListing.sol:33, 142–152` (clamp applied
> in `LibListing`, not `LibGDA`)

### Price Start

`calcBuy` passes `LibAuction.getPriceStart` to the GDA as its start time, not
the stored `startTs`. An auction can carry one pause window
`(pauseStart, pauseEnd)` (see [Pause Windows](#pause-windows)) whose time the
price does not count:

```
if pauseEnd > pauseStart:            (a window is set)
    priceStart = startTs + clamp(block.timestamp − pauseStart, 0, pauseEnd − pauseStart)
else:                                (no window: reads as (0, 0))
    priceStart = startTs

elapsed = block.timestamp − priceStart
```

Nothing is skipped before the window, time stands still during it, and the
whole window is skipped after it. Because `setPause` only accepts a window
that has already ended (`pauseEnd ≤ block.timestamp`), every on-chain price
read after a window is set falls in the last case:
`priceStart = startTs + (pauseEnd − pauseStart)`. The before / during cases
matter only when the rule is evaluated at past times, as the client's price
history does with a mirror of the same function. The "has started" check does
not use `priceStart`: it reads `startTs` (step 5 below).

> Source: `LibAuction.sol:47` (`getPriceStart` as the GDA start), `:60–69`
> (`getPriceStart`), `LibAuctionRegistry.sol:140–143` (`getPause`: unset reads
> `(0, 0)` via `safeGet`), `_AuctionRegistrySystem.sol:71` (window must have
> ended); client mirror `packages/client/src/network/shapes/Auction/pricing.ts:16–25`

## Buying from an Auction

`AuctionBuySystem.execute(itemIndex, amount)`:

1. Resolve the buyer's account — the account's owner wallet **or** its
   operator may sign (`LibAccount.getByOwnerOrOperator`; see
   [accounts.md → Who May Sign](../world/accounts.md#who-may-sign))
2. Verify amount > 0 and item exists
3. Verify auction exists for this item
4. Verify purchase won't exceed max supply: `balance + amount ≤ max`
5. Verify auction has started: `block.timestamp ≥ startTs` — the stored
   start, not the pause-adjusted `priceStart`; else
   `"LibAuction: auction has not started"`
6. Verify account meets any requirements (`LibConditional`)
7. **Gacha-pool check (ticket auctions only)** — buying Gacha Tickets
   (item 10) requires `getNumFree() + getSupplyHeadroom() > 0`; buying Reroll
   Tokens (item 11) requires `getNumFree() > 0`; otherwise revert
   `"gacha pool exhausted"`. The check stops sales once nothing can be drawn
   but does not ration the amount bought — see
   [gacha.md → Supply Cap and Pool Drawdown](../gacha/gacha.md#supply-cap-and-pool-drawdown)
8. Calculate cost via GDA formula, counting elapsed time from
   `getPriceStart` (see [Price Start](#price-start))
9. Deduct payment from buyer's inventory (currency item)
10. Add purchased items to buyer's inventory
11. Increment auction balance (sales counter)
12. Log purchase and emit `AUCTION_BUY` event

> Source: `AuctionBuySystem.sol:20–56` (resolver at 23); `LibAuction.sol:111–115`
> (start check on `startTs`), `:44–69` (cost from `getPriceStart`)

## Requirements

Auctions can have conditional requirements (e.g., quest completion, item
ownership) that must be met before purchasing.

Requirement anchor: `keccak256("auction.requirement", auctionID)`

> Source: `LibAuctionRegistry.sol:98–106, 125–127, 160–162`

## Pause Windows

Each auction can carry one pause window — a span such as a chain halt that GDA
pricing skips, so time in which nobody could buy does not count as time with
no sales. It is stored in `TimePause` as `uint64 pauseStart << 64 |
uint64 pauseEnd`; the arithmetic is in [Price Start](#price-start).

### Setting a Window

`_AuctionRegistrySystem.setPause(itemIndex, pauseStart, pauseEnd)` is **admin
only** (`onlyAdmin`: the caller needs `ROLE_ADMIN`, else `"Auth: not an
admin"`; see [admin-operations.md → Role Gating](../utility/admin-operations.md#role-gating)).
Checks, in source order:

1. The auction exists, else `"AuctionRegistry: auction does not exist"`
2. Unless both bounds are 0:
   1. `pauseStart < pauseEnd`, else `"AuctionRegistry: pause must end after it starts"`
   2. `pauseEnd ≤ block.timestamp`, else `"AuctionRegistry: pause has not ended"`
   3. `pauseStart ≥ startTs`, else `"AuctionRegistry: pause starts before the auction"`

Together: `startTs ≤ pauseStart < pauseEnd ≤ now`, so a bad window fails at
the admin transaction instead of in later buys. The source comment names the
intended bounds: `pauseStart` = the last block timestamp before the halt,
`pauseEnd` = the first block timestamp after it.

### Window Rules

| Case | Effect |
|---|---|
| `setPause(itemIndex, 0, 0)` | Clears the window (`TimePause` removed) |
| `setPause` on an auction that already has a window | Replaces it — one window per auction, no list |
| `remove(itemIndex)` | Deletes the auction; its window is removed with it |
| `create(...)` | Sets no window; a new auction has none until `setPause` |
| No window set | Reads as `(0, 0)`: `priceStart = startTs` |

**Merchant listings are not paused.** The window exists only on auctions:
`LibListing` passes a GDA listing's stored `TimeStart` to `LibGDA` unchanged.
How far a listing's price can fall behind schedule is bounded instead by its
deficit clamp (`MAX_DEFICIT_PERIODS = 3`) — see
[NPC shops → Price Floor](../economy/npc-shops.md#price-floor-deficit-clamp).

**`create` returns the new auction ID.** `_AuctionRegistrySystem.create`
returns the ID `LibAuctionRegistry.create` assigns
(`keccak256("auction", itemIndex)`). It previously returned its pre-create
lookup, which the `"auction already exists"` check forces to 0, so it always
returned 0.

> Source: `_AuctionRegistrySystem.sol:59–76` (`setPause`; intended bounds in
> the comment at 59–61), `:28–29, 49` (`create`), `:52–57` (`remove`);
> `LibAuctionRegistry.sol:91–96` (`setPause`: `(0, 0)` removes, else
> `packPause` + set), `:84` (`remove` clears `TimePause`), `:56–69` (`create`
> sets no window), `:140–143` (`getPause`), `:156–158` (`genID`);
> `LibAuction.sol:63–69`; `AuthRoles.sol:20–26`; `LibListing.sol:28–33, 131–147`
> (listing GDA reads `TimeStart`; deficit clamp);
> `TimePauseComponent.sol:6–11`; tests `test/systems/Auction.t.sol:125–192`

## Current Auctions

| Name | Sale Item | Currency | Supply | Target Price | Period | Decay | Rate | Start |
|---|---|---|---|---|---|---|---|---|
| Gacha ↔ Musu | Gacha Ticket (10) | MUSU (1) | 17,222 | 32,000 | 86,400s (1 day) | 0.75 | 32/day | 2025-05-16 |
| Reroll ↔ Onyx | Reroll Token (11) | ONYX (100) | 100,000 | 50 | 86,400s (1 day) | 0.5 | 16/day | 2025-10-16 |

> Source: `data/auctions/auctions.csv`

> **2026-11-02 — pause window on both live auctions (chain state, not
> source).** After the chain halt of 2026-10-23 → 2026-11-02, both auctions
> above — Gacha Ticket (10) and Reroll Token (11) — carry the window
> `pauseStart = 1792770590` (2026-10-23T15:49:50Z) → `pauseEnd = 1793582921`
> (2026-11-02T01:28:41Z), 812,331 s (≈ 9.4 days), set by two `setPause` calls
> to the auction registry system on 2026-11-02. Their prices skip that span:
> the halt does not count toward their decay. `auctions.csv` carries no
> window; it is set only by `setPause` (see [Pause Windows](#pause-windows)).

## Logging

| Data Key | Description |
|---|---|
| `ITEM_AUCTION_BUY` | Per-item purchase count from auctions |
| `ITEM_AUCTION_SPEND` | Per-currency amount spent at auctions |

> Source: `LibAuction.sol:128–130`
