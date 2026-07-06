# Auctions (GDA)

> Source: `packages/contracts/src/libraries/LibAuction.sol` (L1–139),
> `packages/contracts/src/libraries/LibAuctionRegistry.sol` (L1–132),
> `packages/contracts/src/libraries/utils/LibGDA.sol` (L1–51),
> `packages/contracts/src/systems/AuctionBuySystem.sol` (L1–47),
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

Auction ID: `keccak256("auction", itemIndex)`

Only one auction exists per item (at most).

> Source: `LibAuctionRegistry.sol:36–66`

## Price Calculation

The auction price uses the GDA formula (see
[NPC shops](../economy/npc-shops.md) for the underlying `LibGDA.calc`):

```
spotPrice = targetPrice × decay^(timeDelta) / decay^(prevSold / rate)
batchCost = geometric series summation over qty units at spotPrice
```

Where:
- `timeDelta = (block.timestamp - startTs) / period`
- `prevSold = balance` (total sales so far)
- `decay` is stored as `int32 × 1e6`, then multiplied by `1e12` to reach WAD
  precision for LibGDA

The result is in WAD (×1e18) and rounded up:
```
cost = ceil(LibGDA.calc(params) / 1e18)
```

> Source: `LibAuction.sol:44–58`

## Buying from an Auction

`AuctionBuySystem.execute(itemIndex, amount)`:

1. Verify amount > 0 and item exists
2. Verify auction exists for this item
3. Verify purchase won't exceed max supply: `balance + amount ≤ max`
4. Verify auction has started: `block.timestamp ≥ startTs`
5. Verify account meets any requirements (`LibConditional`)
6. Calculate cost via GDA formula
7. Deduct payment from buyer's inventory (currency item)
8. Add purchased items to buyer's inventory
9. Increment auction balance (sales counter)
10. Log purchase and emit `AUCTION_BUY` event

> Source: `AuctionBuySystem.sol:18–42`

## Requirements

Auctions can have conditional requirements (e.g., quest completion, item
ownership) that must be met before purchasing.

Requirement anchor: `keccak256("auction.requirement", auctionID)`

> Source: `LibAuctionRegistry.sol:87–95, 114–116`

## Current Auctions

| Name | Sale Item | Currency | Supply | Target Price | Period | Decay | Rate | Start |
|---|---|---|---|---|---|---|---|---|
| Gacha ↔ Musu | Gacha Ticket (10) | MUSU (1) | 17,222 | 32,000 | 86,400s (1 day) | 0.75 | 32/day | 2025-05-16 |
| Reroll ↔ Onyx | Reroll Token (11) | ONYX (100) | 100,000 | 50 | 86,400s (1 day) | 0.5 | 16/day | 2025-10-16 |

> Source: `data/auctions/auctions.csv`

## Logging

| Data Key | Description |
|---|---|
| `ITEM_AUCTION_BUY` | Per-item purchase count from auctions |
| `ITEM_AUCTION_SPEND` | Per-currency amount spent at auctions |

> Source: `LibAuction.sol:117–119`
