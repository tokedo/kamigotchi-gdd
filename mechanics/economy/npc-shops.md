# NPC Shop Listings

> Source: `packages/contracts/src/libraries/LibListing.sol` (L1–218),
> `packages/contracts/src/libraries/LibListingRegistry.sol` (L1–188),
> `packages/contracts/src/libraries/LibNPC.sol` (L1–94),
> `packages/contracts/src/libraries/utils/LibGDA.sol` (L1–51),
> `packages/contracts/src/systems/ListingBuySystem.sol` (L1–66),
> `packages/contracts/src/systems/ListingSellSystem.sol` (L1–62)

## Overview

NPC merchants sell and buy items at configurable prices. Players must be in the
same room as the NPC to transact (unless the NPC's room index is 0, making it
globally accessible). Each listing ties one item to one NPC with a defined
currency and pricing strategy.

See `catalogs/npcs/listings.csv` for the full NPC shop catalog.

## NPC Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"NPC"` |
| `IndexNPC` | Unique NPC index (uint32) |
| `Name` | Display name |
| `IndexRoom` | Room where the NPC resides (0 = global) |

Entity ID: `keccak256("NPC", index)`

> Source: `LibNPC.sol:18–31, 91–93`

## Listing Entity Shape

Each listing links an NPC to a specific item it trades:

| Component | Description |
|---|---|
| `EntityType` | `"LISTING"` |
| `IndexNPC` | Merchant index |
| `IndexItem` | Item being traded |
| `IndexCurrency` | Currency item index |
| `Value` | Target/base price |
| `Balance` | Running count of units bought/sold (for GDA) |
| `TimeStart` | Timestamp when listing was created/reset (for GDA) |

Entity ID: `keccak256("listing", merchantIndex, itemIndex)`

Additionally, each listing has **pricing sub-entities** for buy and sell sides:
- Buy: `keccak256("listing.buy", listingID)`
- Sell: `keccak256("listing.sell", listingID)`

> Source: `LibListingRegistry.sol:25–43, 52–67, 173–187`

## Pricing Strategies

### FIXED

Simple flat price: `price = Value × amount`

Both buy and sell sides can use FIXED pricing. The price per unit is the
`Value` component on the listing entity.

> Source: `LibListing.sol:111–112, 138–139`

### GDA (Gradual Dutch Auction)

Dynamic pricing that increases with demand and decays over time. Used for
**buy side only**.

Parameters stored on the buy pricing entity:

| Parameter | Component | Precision | Description |
|---|---|---|---|
| `targetPrice` | `Value` | 1e0 | Base price when supply matches demand |
| `period` | `Period` | seconds | Time unit for decay/rate calculations |
| `decay` | `Decay` | 1e6 (stored) → 1e18 (calc) | Price decay factor per period with no buys |
| `rate` | `Rate` | 1e0 | Expected purchases per period for steady pricing |
| `prevSold` | `Balance` | 1e0 | Total units sold so far |
| `startTs` | `TimeStart` | 1e0 | Epoch when tracking began |

**Formula** (perpetual discrete VRGDA):

```
timeDelta = (now - startTs) / period

spotPrice = targetPrice × decay^(timeDelta - prevSold / rate)
```

For quantity > 1, cost is summed via geometric series:

```
c = decay^(-1/rate)              (per-unit price compound)
cost = spotPrice × (c^quantity - 1) / (c - 1)
```

The result is in WAD (1e18) precision, then rounded up:
`finalPrice = ceil(costWad / 1e18)`

**Behavior**: Price rises when buying outpaces the `rate` per `period`. Price
decays back toward `targetPrice` when buying slows. This creates natural
supply/demand equilibrium.

> Source: `LibGDA.sol:28–50`, `LibListing.sol:113–126`

### SCALED (sell only)

> **Currently unused.** The SCALED pricing type is fully implemented in contract
> code but no NPC listings use it — all sell-side pricing is currently FIXED.

Sell price is a fraction of the current buy price:

```
sellPrice = calcBuyPrice(amount) × scale / 1e9
```

Scale is stored with 1e9 precision (e.g., `500000000` = 50% of buy price).
Must be in range `[0, 1e9]`.

> Source: `LibListing.sol:140–143`, `_ListingRegistrySystem.sol:90–99`

## Buy Process

`ListingBuySystem.execute(merchantIndex, itemIndices[], amounts[])`:

1. **Resolve account** — get caller's account entity
2. **Resolve NPC** — look up merchant by index, verify it exists
3. **Check room** — player must be in the same room as NPC (or NPC room = 0)
4. **For each item**:
   a. Look up listing by (merchantIndex, itemIndex)
   b. Verify listing requirements via `LibConditional`
   c. Calculate buy price, increment listing balance
   d. Add items to player inventory, deduct currency from player
   e. Log buy event, track spending in score system

> Source: `ListingBuySystem.sol:21–57`, `LibListing.sol:44–59`

## Sell Process

`ListingSellSystem.execute(merchantIndex, itemIndices[], amounts[])`:

Same flow as buy, but reversed:

1. **Room check** — same as buy
2. **For each item**:
   a. Look up listing, verify requirements
   b. Calculate sell price, decrement listing balance
   c. Remove items from player inventory, add currency to player
   d. Log sell event

> Source: `ListingSellSystem.sol:20–53`, `LibListing.sol:62–77`

## Requirements

Listings can have **conditional requirements** that must be met before buying
or selling. Requirements apply equally to both buy and sell sides.

Requirements are anchored to: `keccak256("listing.requirement", listingID)`

Checked via `LibConditional.check()` against the player's account entity.

> Source: `LibListing.sol:82–94`, `LibListingRegistry.sol:106–115`

## Balance Tracking

The listing's `Balance` component tracks cumulative units bought/sold:
- **Buy** increments balance (more purchased → higher GDA price)
- **Sell** decrements balance (selling back → lowers GDA price)

For FIXED pricing, balance is tracked but doesn't affect price. For GDA
pricing, balance directly feeds into the VRGDA formula as `prevSold`.

The balance and time-start can be **reset** by an admin to re-calibrate
dynamic pricing without removing the listing.

> Source: `LibListing.sol:151–159`, `LibListingRegistry.sol:70–73`

## Data Files

Listing definitions are in:
- `data/listings/listings.csv` — NPC merchant listings with pricing strategies
