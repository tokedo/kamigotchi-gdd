# Trading (P2P)

> Source: `packages/contracts/src/libraries/LibTrade.sol` (L1–477),
> `packages/contracts/src/systems/TradeCreateSystem.sol` (L1–70),
> `packages/contracts/src/systems/TradeExecuteSystem.sol` (L1–40),
> `packages/contracts/src/systems/TradeCompleteSystem.sol` (L1–54),
> `packages/contracts/src/systems/TradeCancelSystem.sol` (L1–51)

## Overview

Player-to-player trading uses an **orderbook model** with a three-step flow:
Create → Execute → Complete. A maker posts a trade offer (what they want to buy
and what they're selling), a taker fulfills it, and the maker confirms receipt.
Trades can optionally target a specific taker.

## Trade Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"TRADE"` |
| `IDOwnsTrade` | Maker's account ID |
| `State` | `"PENDING"` → `"EXECUTED"` |
| `IdTarget` | (optional) Designated taker account ID |

Sub-entities for orders:
- **Buy order**: `keccak256("trade.buy", tradeID)` — `Keys` (item indices) + `Values` (amounts)
- **Sell order**: `keccak256("trade.sell", tradeID)` — `Keys` (item indices) + `Values` (amounts)

Trade ID: unique entity ID assigned by the world.

> Source: `LibTrade.sol:30–46, 62–79, 349–355`

## Structure Constraints

Trades are currently limited to:

- **Exactly one item** on the buy side
- **Exactly one item** on the sell side
- **One side must be MUSU** (the base currency, index 1) — no pure barter
- **Both sides cannot be MUSU**
- All items must not have the `NOT_TRADABLE` flag

> Source: `LibTrade.sol:277–290, 304–313`

## Trade Lifecycle

### Step 1: Create (Maker)

`TradeCreateSystem.execute(buyIndices, buyAmts, sellIndices, sellAmts, targetID)`:

1. Verify maker has not exceeded `MAX_TRADES_PER_ACCOUNT`
2. Verify trade structure (one item each side, one must be MUSU)
3. Verify all items are tradable
4. Deduct **creation fee** (MUSU)
5. Deduct **delivery fee** (MUSU) — waived if maker is in the **Trade Room** (room 66)
6. Create trade entity with state `PENDING`
7. **Escrow sell-side items**: maker's items are transferred from their inventory
   to the trade entity's inventory

> Source: `TradeCreateSystem.sol:15–54`, `LibTrade.sol:62–98, 358–368`

### Step 2: Execute (Taker)

`TradeExecuteSystem.execute(tradeID)`:

1. Verify trade exists, is `PENDING`, caller is valid taker (not maker)
2. If trade has a designated target, verify caller matches
3. Deduct **delivery fee** from taker (waived in Trade Room)
4. **Execute buy order**: transfer buy-side items from taker to trade entity
   (held in escrow). Log amounts for both parties.
5. **Execute sell order**: compute the tax on each sell-side item, **burn** it
   from the trade entity's escrow, and transfer the rest to the taker. Log
   amounts (pre-tax) and the taker's `TRADE_TAX`.
6. Set trade state to `EXECUTED`, record taker ID

> Source: `TradeExecuteSystem.sol:15–34`, `LibTrade.sol:104–153`

### Step 3: Complete (Maker)

`TradeCompleteSystem.execute(tradeID)`:

1. Verify trade is `EXECUTED` and caller is maker
2. Deduct **delivery fee** from maker (waived in Trade Room)
3. **Complete buy order**: compute the tax on each buy-side item, **burn** it
   from the trade entity's escrow, and transfer the rest to the maker; log the
   maker's `TRADE_TAX`. Clean up order data.
4. **Complete sell order**: clean up remaining data
5. **Burn residue**: any MUSU still held by the trade entity is burned, so no
   inventory outlives the trade
6. Remove all trade entity data

> Source: `TradeCompleteSystem.sol:16–34`, `LibTrade.sol:160–207`

### Cancel (Maker only, while PENDING)

`TradeCancelSystem.execute(tradeID)`:

1. Verify trade is `PENDING` and caller is maker
2. Deduct delivery fee
3. **Return escrowed items**: sell-side items are transferred back from trade
   entity to maker's inventory
4. Remove all trade entity data

> Source: `TradeCancelSystem.sol:16–34`, `LibTrade.sol:213–241`

## Fees

| Fee | Config Key | When Charged | To Whom |
|---|---|---|---|
| Creation fee | `TRADE_CREATION_FEE` | On trade create | Maker |
| Delivery fee | `TRADE_DELIVERY_FEE` | On create, execute, complete, cancel | Whoever calls (waived in room 66) |

Both fees are paid in MUSU.

> Source: `LibTrade.sol:358–368`

## Trade Tax

Tax is applied when items leave escrow (on execute for sell-side, on complete
for buy-side):

```
tax = (amount × TRADE_TAX_RATE[1]) / 10^TRADE_TAX_RATE[0]
```

`TRADE_TAX_RATE` is a config array where index 0 is the precision exponent and
index 1 is the rate numerator. Deployed value `[3, 10]`: `10 / 10^3` = **1%**,
floored.

Tax only applies to **MUSU** transfers. Non-MUSU items are not taxed. Since one
side of every trade is MUSU (see [Structure Constraints](#structure-constraints)),
exactly one side is taxed: the taker pays it on a MUSU sell side, the maker on
a MUSU buy side.

**The tax is burned, not distributed.** In both tax branches the tax is
removed from the trade entity's escrow with `LibInventory.decFor` before the
payout transfer, so escrow ends at exactly the payout and the transfer empties
it. Because `decFor` also decrements the global `ITEM_COUNT` for MUSU, the
burn shows up as a supply reduction. `complete()` then calls `burnResidue`,
which burns any MUSU still left in the trade's escrow — this catches the
sell-side tax of trades **executed before** this burn existed (which kept
their tax in escrow) and any MUSU sent to the trade entity directly.

Before the burn was added, the tax was subtracted from the payout but left in
the trade entity's escrow; once the trade entity was stripped, that inventory
belonged to a dead ID that no system can reach — out of circulation, but still
counted in `ITEM_COUNT`. Those remnants are not swept by the new code.

> Source: `LibTrade.sol:140–149, 164, 180–189, 195–200, 339–347`,
> `LibInventory.sol:183–198` (`_decFor`, `ITEM_COUNT`), `configs.ts:181`

## Trade Room

Room 66 is the designated **Trade Room**. Players in this room are exempt from
the delivery fee. This incentivizes using a specific in-game location for
trading.

> Source: `LibTrade.sol:28, 247–249, 364–368`

## Admin Operations

Both `TradeCompleteSystem` and `TradeCancelSystem` have admin-only batch
functions that can complete or cancel trades without the maker's action. These
are used for maintenance (e.g., clearing stuck trades).

> Source: `TradeCompleteSystem.sol:37–48`, `TradeCancelSystem.sol:37–45`
