# Item Pools (Constant-Product AMM)

> Source: `packages/contracts/src/systems/PoolSystem.sol` (L1–127),
> `packages/contracts/src/systems/_PoolRegistrySystem.sol` (L1–132),
> `packages/contracts/src/libraries/LibPool.sol` (L1–330),
> `packages/contracts/src/libraries/LibPoolRegistry.sol` (L1–109),
> `packages/contracts/src/libraries/LibInventory.sol` (L252–274, L319–323),
> `packages/contracts/deployment/contracts/PoolCeremony.s.sol` (L1–129)

## Overview

An **item pool** is an automated market maker between **two fungible items**.
It holds a reserve of each item and prices swaps by the constant-product rule
`x × y = k`, exactly like a UniswapV2 pair — but the traded assets are game
item indices, not ERC-20 tokens.

Anything that exists as an item can be pooled. That includes **MUSU** (item
index `1`), the game's currency, and **ONYX** (item index `100`), which becomes
an item only after being bridged in through the token portal — see
[Token Portal](token-portal.md). A pool never mints: every unit in a reserve
was supplied by an account, so ONYX in a pool is backed by real bridged
tokens.

Three player actions exist — **swap**, **add liquidity**, **remove
liquidity** — all on `PoolSystem`. Liquidity providers (LPs) hold **share**
entities entitling them to a pro-rata slice of both reserves; the swap fee
accrues to the reserves themselves, so LPs earn it when they burn.

**In-world access point**: the fountain at the crossroads leading out of the
junkyard — room `31`, Scrapyard Exit — is a clickable hotspot that opens the
pool interface. It can also be reached from the Menu.

> Source: `LibPool.sol:22–35` (library overview),
> `packages/client/src/constants/rooms/31_scrapyard-exit.ts` (fountain
> clickbox → `triggerPoolModal`),
> `packages/client/src/app/triggers/triggerPoolModal.ts`

> ⚠️ **Swapping can be switched off world-side.** Beyond the per-pool
> `IsDisabled` component, pool systems are gated by world config flags
> (`POOL_SWAP_ENABLED` and siblings). While such a flag is `0`, every swap
> reverts: the fountain still opens and the entrypoints below still exist, but
> the transaction fails. These flags are chain state, not source constants —
> the live value is read from the world config by key (`0` = disabled) and
> cannot be derived from this document.

## Which Pools Exist Is World State, Not Code

The contracts define *how* a pool works; they do not define *which* pools
exist. Pools are created by an admin transaction after deployment, and the
item pair, seed amounts and fee are supplied as **run-time environment
parameters** to the `PoolCeremony` deploy script:

| Env var | Meaning |
|---|---|
| `TREASURY_PRIV_KEY` | Treasury account owner key — signs `create` |
| `POOL_ITEM_A` / `POOL_ITEM_B` | Item indices of the pair |
| `POOL_SEED_A` / `POOL_SEED_B` | Initial reserve amounts (each ≥ `MINIMUM_SEED`) |
| `POOL_FEE_BPS` | Swap fee in basis points (≤ `MAX_FEE_BPS`) |

No CSV or checked-in artifact lists the live pools, so **the set of deployed
pools cannot be derived from source**. The ceremony seeds from a treasury
account that already holds the items (nothing is minted or faucetted), grants
that account `ROLE_ADMIN` only for the duration of the call, and revokes it
afterwards; a rerun after a partial failure performs only the outstanding
revoke.

> ⚠️ UNCERTAIN: the live pool list is on-chain state. This document describes
> the mechanism only; any specific pair must be read from the world.

> Source: `PoolCeremony.s.sol:11–51` (parameters and doctrine), `:75–127`
> (ephemeral role grant/revoke, create, rerun self-heal)

## Entity Shapes

### Pool

| Component | Description |
|---|---|
| `EntityType` | `"POOL"` |
| `Keys` | `[itemIndexLo, itemIndexHi]` — the pair, canonically sorted `lo < hi` |
| `Rate` | Swap fee, in basis points |
| `Value` | Total LP shares outstanding |
| `TimeStart` | Creation timestamp |
| `IsDisabled` | Optional — pauses swaps and liquidity adds |

Entity ID:

```
poolID = keccak256("amm.pool", lo, hi)          where (lo, hi) = sort(indexA, indexB)
```

Because the ID is a pure hash of the sorted pair, it is **order-insensitive**
(`swap(A,B)` and `swap(B,A)` address the same pool) and **precomputable by
anyone** before the pool exists — a fact the creation path has to defend
against (see [`create`](#createindexa-indexb-amta-amtb-feebps)).

The pool's **reserves** are ordinary inventory entities held by the pool ID
itself, so a reserve is read with `LibInventory.getBalanceOf(poolID, itemIndex)`.

> Source: `LibPoolRegistry.sol:19–30` (shape), `:37–53` (`create`),
> `:101–108` (`sortIndices`, `genID`)

### LP Share

| Component | Description |
|---|---|
| `EntityType` | `"POOL_SHARE"` |
| `IdHolder` | Account holding the position (or the pool itself, for locked shares) |
| `IDType` | The pool this position belongs to |
| `Value` | Share balance |

Entity ID:

```
shareID = keccak256("amm.pool.share", poolID, holderID)
```

One share entity per (pool, holder) pair. It is created on first mint and
**deleted outright** when the balance reaches zero. Total supply is not summed
from the share entities — it is mirrored on the pool's own `Value` component,
incremented on every mint and decremented on every burn.

Because `IDType` stores the pool, every position in a pool is reverse-indexable
by `getEntitiesWithValue(poolID)` — used by pool teardown.

> Source: `LibPool.sol:30–35` (shape), `:144–170` (`mintShares`, `burnShares`),
> `:221–223` (`getTotalSupply`), `:327–329` (`genShareID`)

## Constant-Product Math

All three formulas floor (round down) in the pool's favour, so `k` never
decreases through rounding.

### Swap Output

`BPS_DENOMINATOR = 10000`. The fee is taken off the **input** before it enters
the invariant, which is what leaves it in the reserves for LPs:

```
amountInWithFee = amountIn × (10000 − feeBps)

                       amountInWithFee × reserveOut
amountOut = ────────────────────────────────────────────────
             reserveIn × 10000 + amountInWithFee
```

Equivalently, with `f = feeBps / 10000`:

```
amountOut = (reserveOut × (1 − f) × amountIn) / (reserveIn + (1 − f) × amountIn)
```

Price impact is therefore the usual hyperbolic curve: output approaches
`reserveOut` asymptotically and can never drain a reserve.

> Source: `LibPool.sol:20` (`BPS_DENOMINATOR`), `:229–237` (`calcAmountOut`)

### Quote (Reserve-Ratio Valuation)

The value of `amountA` expressed in item B at the *current* ratio, with no
price impact and no fee. Used only by `addLiquidity` to settle the deposit
ratio:

```
quote(amountA, reserveA, reserveB) = ⌊amountA × reserveB / reserveA⌋
```

Reverts `"Pool: no reserves"` if `reserveA = 0`.

> Source: `LibPool.sol:240–247`

### Initial Liquidity (Admin Seed Only)

The very first mint — performed inside `_PoolRegistrySystem.create` — uses the
geometric mean of the seed amounts:

```
initialLiquidity = ⌊√(amtA × amtB)⌋
```

These shares are minted **to the pool itself** and are permanently locked (see
[Locked Seed Shares](#locked-seed-shares)).

> Source: `LibPool.sol:250–252` (`calcInitialLiquidity`),
> `_PoolRegistrySystem.sol:61`

### Subsequent LP Mint

Every later deposit mints pro-rata against existing supply, taking the
**smaller** of the two sides so an unbalanced deposit cannot dilute existing
LPs:

```
liquidity = min( ⌊amtA × supply / reserveA⌋ , ⌊amtB × supply / reserveB⌋ )
```

Computed with 512-bit `mulDiv` because `amt × supply` can exceed `2^256` at
extreme scales. Reverts `"Pool: insufficient liquidity minted"` if the result
floors to zero.

> Source: `LibPool.sol:95–101`

### LP Burn

```
amtA = ⌊shares × reserveA / supply⌋
amtB = ⌊shares × reserveB / supply⌋
```

> Source: `LibPool.sol:125–127`

## Player Entrypoints

All three are called by the account's **operator** address
(`LibAccount.getByOperator`) and refresh the account's last-action timestamp.

### `swap(indexIn, indexOut, amountIn, minAmountOut)`

1. Resolve the account from the operator
2. Resolve the pool from the pair; revert `"Pool does not exist"` if absent
3. **Enabled check** — `LibDisabled.verifyEnabled(poolID)`
4. **Tradability check** — both indices must be transferable
5. Require `amountIn > 0` (`"Pool: zero input"`)
6. Read both reserves, compute `amountOut` via `calcAmountOut` with the pool's
   fee
7. Require `amountOut > 0` (`"Pool: insufficient output"`)
8. **Slippage guard** — require `amountOut ≥ minAmountOut`
   (`"Pool: slippage exceeded"`)
9. Move `amountIn` account → pool (reverts if the account is short), then
   `amountOut` pool → account

Returns `amountOut`.

> Source: `PoolSystem.sol:22–48`, `LibPool.sol:41–60`

### `addLiquidity(indexA, indexB, amountADesired, amountBDesired, amountAMin, amountBMin)`

Deposits **both** items at the current reserve ratio, settling between the
desired and minimum bounds the same way the UniswapV2 router does:

1. Resolve account and pool; enabled check; tradability check
2. `amountBOptimal = quote(amountADesired, reserveA, reserveB)`
3. If `amountBOptimal ≤ amountBDesired`:
   - require `amountBOptimal ≥ amountBMin` (`"Pool: insufficient B amount"`)
   - settle on `(amountADesired, amountBOptimal)`
4. Otherwise:
   - `amountAOptimal = quote(amountBDesired, reserveB, reserveA)`
   - require `amountAOptimal ≤ amountADesired` (`"Pool: excessive A amount"`)
     — a rounding edge-case guard, so the caller can never overspend A
   - require `amountAOptimal ≥ amountAMin` (`"Pool: insufficient A amount"`)
   - settle on `(amountAOptimal, amountBDesired)`
5. Mint `liquidity` shares (formula above)
6. Move both settled amounts account → pool

Returns `(amtA, amtB, liquidity)`.

> Source: `PoolSystem.sol:51–86`, `LibPool.sol:73–106`

### `removeLiquidity(indexA, indexB, shares, amountAMin, amountBMin)`

1. Resolve account and pool — **no enabled check**: LPs can always exit a
   disabled pool
2. Require `shares > 0` (`"Pool: zero shares"`)
3. Compute the pro-rata slice of both reserves
4. Require `amtA > 0 **or** amtB > 0` (`"Pool: insufficient liquidity
   burned"`) — deliberately OR, not AND. At game-scale integers a small
   position in a skewed pool floors one side to zero; requiring both to be
   positive would freeze that position forever. The zeroed side is genuinely
   worth ~0 and is forfeited.
5. **Slippage guard** — require `amtA ≥ amountAMin` **and** `amtB ≥ amountBMin`
   (`"Pool: slippage exceeded"`)
6. Burn the shares (reverts if the holder is short), then move both amounts
   pool → account

Returns `(amtA, amtB)`.

> Source: `PoolSystem.sol:89–115`, `LibPool.sol:117–138`

### Tradability

`swap` and `addLiquidity` both call `LibInventory.verifyTransferable` on the
pair, which reverts `"Transfer includes untradeable item"` if either item
carries the `NOT_TRADABLE` flag. This mirrors the rules on trading and item
transfer; a pool of untradable items would strand its LPs. Pool creation
applies the same check.

`removeLiquidity` does **not** re-check tradability, so an item flagged
untradable after a pool already exists still lets its LPs exit.

> Source: `PoolSystem.sol:117–122`, `LibInventory.sol:319–323`,
> `_PoolRegistrySystem.sol:36, 122–127`

### Reserve Moves Are Not Acquisitions

All pool ↔ account movement uses `LibInventory.transferForNoLog`, a transfer
variant that deliberately **skips acquisition logging** (`ITEM_TOTAL` per
account and the global `ITEM_COUNT`). Crediting `ITEM_TOTAL` on a swap output
would corrupt earned-item leaderboards — MUSU earned is read from
`ITEM_TOTAL[MUSU]` — and `ITEM_COUNT` churn would be wrong regardless, since
a transfer moves supply rather than minting or burning it.

> Source: `LibInventory.sol:252–274`

## Locked Seed Shares

The initial `⌊√(amtA × amtB)⌋` shares are minted with the **pool itself** as
the holder. `burnShares` hard-rejects `holderID == poolID`
(`"Pool: shares locked"`), so those shares can never be redeemed by anyone.
Consequently total supply and both reserves can never fully drain, and the
per-share exchange rate is always well-defined.

The only path that clears them is `burnLockedShares`, reachable only from
admin pool removal.

> Source: `LibPool.sol:157–158` (lock), `:173–183` (`burnLockedShares`),
> `_PoolRegistrySystem.sol:19–21, 61`

## Admin Registry

`_PoolRegistrySystem`, gated by `onlyAdmin`.

| Constant | Value | Meaning |
|---|---|---|
| `MAX_FEE_BPS` | `1000` | Maximum swap fee — 10% |
| `MINIMUM_SEED` | `1000` | Minimum initial reserve per side, so integer rounding stays tolerable |

### `create(indexA, indexB, amtA, amtB, feeBps)`

Creates the pool and atomically seeds it. Validation, in order:

1. `indexA ≠ indexB` (`"Pool: identical items"`)
2. Both items are registered (`"Item: does not exist"`)
3. Both items are transferable
4. `feeBps ≤ MAX_FEE_BPS` (`"Pool: fee too high"`)
5. `amtA ≥ MINIMUM_SEED` and `amtB ≥ MINIMUM_SEED` (`"Pool: seed too small"`)
6. No pool already exists for the pair (`"Pool already exists"`)

Then:

7. **Pre-funding sweep** — because the pool ID is a precomputable hash, anyone
   can send items to it *before* it exists. Any pre-existing balance on either
   index is swept to the caller first. Reverting instead would let a 1-unit
   deposit permanently brick the pair (there is no `donate`/`remove` path
   before creation), and seeding on top of it would launch the pool at a
   skewed price. Sweeping keeps the launch price at exactly the seeded ratio.
8. Create the pool entity with the sorted pair and fee
9. Transfer `amtA`/`amtB` from the **caller's own inventory** into the pool —
   nothing is minted, so the caller must actually hold the items
10. Mint `⌊√(amtA × amtB)⌋` locked shares to the pool

Returns the pool ID.

> Source: `_PoolRegistrySystem.sol:15–16` (constants), `:26–62`

### `donate(indexA, indexB, amtA, amtB)`

Adds reserves from the caller's inventory **without minting shares**, which
raises the value of every existing LP share. Either amount may be zero.

> Source: `_PoolRegistrySystem.sol:65–76`

### `setFee(indexA, indexB, feeBps)`

Re-sets the swap fee. Re-checks `feeBps ≤ MAX_FEE_BPS`.

> Source: `_PoolRegistrySystem.sol:78–83`, `LibPoolRegistry.sol:69–71`

### `setDisabled(indexA, indexB, disabled)`

Pauses the pool. Blocks `swap` and `addLiquidity`; `removeLiquidity` still
works, so a pause can never trap LP value.

> Source: `_PoolRegistrySystem.sol:86–90`, `PoolSystem.sol:17, 32, 71`

### `remove(indexA, indexB)`

Deletes the pool. Teardown cannot be griefed by a leftover dust position:

1. **`forceExitAll`** — iterate every `POOL_SHARE` reverse-indexed to this
   pool, skipping the pool's own locked share, and settle each holder at the
   current reserve ratio (`amt = ⌊s × reserve / supply⌋`, recomputing supply
   each iteration as positions burn). Every LP is made whole before deletion.
2. **`burnLockedShares`** — clear the pool's own locked position
3. Return the residual reserves to the **calling admin** — these are real
   supplied items (including token-backed ONYX shards), so they are returned,
   not burned
4. Remove `EntityType`, `Keys`, `Rate`, `Value`, `TimeStart` and any
   `IsDisabled` entry

> Source: `_PoolRegistrySystem.sol:97–119`, `LibPool.sol:190–211`
> (`forceExitAll`), `LibPoolRegistry.sol:57–64`

## Logging & Events

| Data Key | Scope | Description |
|---|---|---|
| `POOL_SWAP_TOTAL` | Per account | Count of swaps performed |
| `POOL_VOLUME` | Per item index | Units of that item moved in or out by swaps |
| `POOL_LIQUIDITY_ADD` | Per account | Count of liquidity adds |
| `POOL_LIQUIDITY_REMOVE` | Per account | Count of liquidity removals |

> ⚠️ `POOL_SWAP_TOTAL` is farmable — a solo round-trip swap costs only the fee
> and increments it. The source explicitly warns never to key a quest
> objective or reward off it.

Emitted events:

| Event | Payload |
|---|---|
| `POOL_SWAP` | `accID`, `poolID`, `itemIn` (uint32), `itemOut` (uint32), `amountIn`, `amountOut` |
| `POOL_LIQUIDITY_ADD` | `accID`, `poolID`, `amtA`, `amtB`, `shares` |
| `POOL_LIQUIDITY_REMOVE` | `accID`, `poolID`, `amtA`, `amtB`, `shares` |

> Source: `LibPool.sol:261–322`

## System IDs

| System | ID string |
|---|---|
| `PoolSystem` | `system.pool` |
| `_PoolRegistrySystem` | `system.pool.registry` |

> Source: `PoolSystem.sol:13`, `_PoolRegistrySystem.sol:13`,
> `packages/contracts/systemIDs.json`
