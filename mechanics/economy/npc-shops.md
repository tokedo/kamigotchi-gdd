# NPC Shop Listings

> Source: `packages/contracts/src/libraries/LibListing.sol` (L1–276),
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

> Source: `LibListing.sol:128–129, 196–197`

### GDA (Gradual Dutch Auction)

Dynamic pricing that increases with demand and decays over time. Used for
**buy side only**.

Parameters are split between the **listing entity** and its **buy pricing
sub-entity** — only `Period`/`Decay`/`Rate` live on the buy sub-entity:

| Parameter | Component | Stored on | Precision | Description |
|---|---|---|---|---|
| `targetPrice` | `Value` | listing entity | 1e0 | Base price when supply matches demand |
| `period` | `Period` | buy sub-entity | seconds | Time unit for decay/rate calculations |
| `decay` | `Decay` | buy sub-entity | 1e6 (stored) → 1e18 (calc) | Price decay factor per period with no buys |
| `rate` | `Rate` | buy sub-entity | 1e0 | Expected purchases per period for steady pricing |
| `prevSold` | `Balance` | listing entity | 1e0 | Total units sold so far |
| `startTs` | `TimeStart` | listing entity | 1e0 | Epoch when tracking began |

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

The result is in WAD (1e18) precision, then rounded up and floored at one
currency unit per item:

```
finalPrice = max( ceil(costWad / 1e18), amount )
```

**Behavior**: Price rises when buying outpaces the `rate` per `period` and
decays when buying lags. It equals `targetPrice` only when cumulative sales
exactly track `rate` per period. Decay is **bounded** — see below.

> Source: `LibGDA.sol:28–50`, `LibListing.sol:120–153`,
> `LibListingRegistry.sol:64–66, 83–89`

#### Price Floor (Deficit Clamp)

Left unbounded, a dormant listing's deficit grows forever and its price decays
toward zero; the exact batch integral then lets an entire accumulated backlog
clear at dust prices. Two constants bound this.

| Constant | Value | Meaning |
|---|---|---|
| `MAX_DEFICIT_PERIODS` | `3` | How many periods a listing may run behind its sales schedule |
| `MAX_BATCH_PERIODS` | `100` | Maximum batch size, in periods of supply |

The clamp works on **elapsed seconds**. The schedule-seconds already covered by
sales, plus the allowance, is:

```
cap = ⌊prevSold × period / rate⌋ + MAX_DEFICIT_PERIODS × period
```

If `now − startTs > cap`, the elapsed term is treated as exactly `cap`. The
exponent in the spot-price formula then bottoms out at `MAX_DEFICIT_PERIODS`,
giving an implicit **price floor**:

```
minSpotPrice = targetPrice × decay^MAX_DEFICIT_PERIODS
```

With every deployed listing using `decay = 0.5` and `MAX_DEFICIT_PERIODS = 3`,
the floor is `targetPrice / 8` — **12.5% of target**.

**Batch bound**: `calcBuyPrice` requires `amount ≤ rate × MAX_BATCH_PERIODS`,
reverting `"LibListing: batch too large"`. Beyond roughly 190 periods the
batch integral's `decay^(−q/rate)` term overflows the fixed-point math and
reverts with a raw error; the explicit bound produces a legible message
instead.

The clamp is applied in `LibListing`, **not** in `LibGDA` — auctions share
that library and deliberately want unbounded decay. See
[Auctions](../marketplace/auctions.md).

> Source: `LibListing.sol:33, 38` (constants), `:140–152` (batch bound, clamp,
> unit floor), `:174–180` (`gdaCapSeconds`)

#### Deficit Settlement on Buy

The clamp above is a view-side calculation. `buy` additionally **settles the
stored deficit** so state and display agree, calling `settleGDA` before
`calcBuyPrice`:

```
if now − startTs > cap:
    startTs ← now − cap
```

`settleGDA` is a no-op unless the listing's buy side is typed `GDA`.

Without it, a dormant listing's stored deficit would let unlimited volume
clear at the floor until the whole backlog was bought. Settling caps the owed
backlog at `MAX_DEFICIT_PERIODS × rate` units, makes the price respond to
purchases immediately, and bounds recovery-to-target at that same volume.

`sell` deliberately does **not** settle: sells raise the deficit rather than
lowering a buyer's price, `calcBuyPrice` clamps whatever it returns anyway,
and the next buy settles.

> Source: `LibListing.sol:63` (call site), `:75–82` (sell rationale),
> `:185–200` (`settleGDA`)

### SCALED (sell only)

> **Currently unused.** The SCALED pricing type is fully implemented in
> contract code, but per the pinned deployment data **no listing has any
> sell-side pricing at all** — the `Sell Price` column is empty for every row
> of `deployment/world/data/listings/listings.csv`, and the deploy script only
> creates a sell pricing sub-entity when that column is set
> (`deployment/world/state/listings.ts:60–73`). With no sell pricing entity,
> `calcSellPrice` reverts (`LibListing.sol:189–202`), so selling to NPCs is
> effectively disabled under this data.
>
> ⚠️ UNCERTAIN: on-chain state created by earlier deployments is not visible
> in the repo — a sell pricing entity set in a prior deploy could exist
> on-chain without appearing in the pinned data files.

Sell price is a fraction of the current buy price:

```
sellPrice = calcBuyPrice(amount) × scale / 1e9
```

Scale is stored with 1e9 precision (e.g., `500000000` = 50% of buy price).
Must be in range `[0, 1e9]`.

> Source: `LibListing.sol:198–201`, `_ListingRegistrySystem.sol:90–99`

## Buy Process

`ListingBuySystem.execute(merchantIndex, itemIndices[], amounts[])`:

1. **Resolve account** — get caller's account entity
2. **Resolve NPC** — look up merchant by index, verify it exists
3. **Check room** — player must be in the same room as NPC (or NPC room = 0)
4. **For each item**:
   a. Look up listing by (merchantIndex, itemIndex)
   b. Verify listing requirements via `LibConditional`
   c. Settle the GDA deficit (`settleGDA`), calculate buy price, increment
      listing balance
   d. Add items to player inventory, deduct currency from player
   e. Log buy event, track spending in score system

> Source: `ListingBuySystem.sol:21–57`, `LibListing.sol:56–72`

## Sell Process

`ListingSellSystem.execute(merchantIndex, itemIndices[], amounts[])`:

Same flow as buy, but reversed:

1. **Room check** — same as buy
2. **For each item**:
   a. Look up listing, verify requirements
   b. Calculate sell price, decrement listing balance
   c. Remove items from player inventory, add currency to player
   d. Log sell event

> Source: `ListingSellSystem.sol:20–53`, `LibListing.sol:75–94`

## Requirements

Listings can have **conditional requirements** that must be met before buying
or selling. Requirements apply equally to both buy and sell sides.

Requirements are anchored to: `keccak256("listing.requirement", listingID)`

Checked via `LibConditional.check()` against the player's account entity.

> Source: `LibListing.sol:99–111`, `LibListingRegistry.sol:106–115`

## Balance Tracking

The listing's `Balance` component tracks cumulative units bought/sold:
- **Buy** increments balance (more purchased → higher GDA price)
- **Sell** decrements balance (selling back → lowers GDA price)

For FIXED pricing, balance is tracked but doesn't affect price. For GDA
pricing, balance directly feeds into the VRGDA formula as `prevSold`.

The balance and time-start can be **reset** by an admin to re-calibrate
dynamic pricing without removing the listing. `TimeStart` is also advanced
automatically by `settleGDA` on every buy against a GDA listing that has
fallen more than `MAX_DEFICIT_PERIODS` behind — see
[Deficit Settlement on Buy](#deficit-settlement-on-buy).

> Source: `LibListing.sol:209–220`, `LibListingRegistry.sol:70–73`

## Data Files

Listing definitions are in:
- `data/listings/listings.csv` — NPC merchant listings with pricing strategies
