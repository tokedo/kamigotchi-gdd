# Gacha System (Mint / Reroll / Reveal)

> Source: `packages/contracts/src/libraries/LibGacha.sol` (L1–197),
> `packages/contracts/src/systems/KamiGachaMintSystem.sol` (L1–53),
> `packages/contracts/src/systems/KamiGachaRerollSystem.sol` (L1–64),
> `packages/contracts/src/systems/KamiGachaRevealSystem.sol` (L1–55),
> `packages/contracts/src/systems/GachaBuyTicketSystem.sol` (L1–143),
> `packages/contracts/src/libraries/LibKamiCreate.sol` (L72–101, L164–174),
> `packages/contracts/src/systems/AuctionBuySystem.sol` (L1–61)

## Overview

The gacha system is the primary mechanism for players to **obtain new Kamis**.
It uses a commit-reveal pattern based on future block hashes for verifiable
randomness. Players spend **Gacha Tickets** (item 10) to mint, or **Reroll
Tokens** (item 11) to exchange existing Kamis for new random ones.

The gacha operates on a **pool** — a set of Kami entities owned by the
`GACHA_ID` sentinel. When a player mints, new Kamis are created and added to
the pool **while the ERC-721 supply cap leaves room**; then on reveal, random
Kamis are drawn from the pool and assigned to the player. Once the cap is
reached, minting creates nothing and every claim draws the pool down — see
[Supply Cap and Pool Drawdown](#supply-cap-and-pool-drawdown).

## Gacha Pool

Entity ID: `keccak256("gacha.id")`

The pool contains all Kamis whose `IDOwnsKami` component points to `GACHA_ID`.
Pool size is queried dynamically via `ownerComp.size(abi.encode(GACHA_ID))`
(`getNumInGacha`).

### Pending Commits and Free Kamis

A global counter tracks gacha commits that have not yet been revealed:

| Item | Definition |
|---|---|
| `GACHA_COMMITS_PENDING` | `LibData` counter on holder `0`, index `0`. **Incremented** by the commit amount on every gacha commit (mint and reroll); **decremented** by the number of commits revealed (normal or forced reveal), saturating at 0 |
| `getNumPending()` | The counter's value |
| `getNumFree()` | `max(0, poolSize − pending)` — pool Kamis not already spoken for by an unrevealed commit |

The decrement saturates because commits made before the counter existed were
never counted and must still reveal.

> ⚠️ UNCERTAIN: the counter starts at zero when this code is deployed; the
> source comment says it should be seeded at deploy from the unrevealed commits
> already outstanding. Whether that seeding was done is chain state. Until those
> legacy commits are revealed, an unseeded counter under-counts pending claims,
> so `getNumFree()` can overstate the free pool.

> Source: `LibGacha.sol:20, 22–24` (`GACHA_ID`, `PENDING_COMMITS_KEY`),
> `:31–40` (`commit` increments), `:75, 80–83` (`releasePending`),
> `:140–154` (`getNumInGacha`, `getNumPending`, `getNumFree`)

### Initial Seed

The pool is **pre-seeded at world deployment** by batch-minting Kamis straight
into `GACHA_ID`. The production branch of the world init seeds
**2,222 Kamis**:

```
await initGachaPool(api, 2222);
```

`initGachaPool` drives `_721BatchMinterSystem.batchMint` in batches of 40
(gas-capped at 60M per call), each minted Kami getting `IDOwnsKami = GACHA_ID`
alongside its ERC-721 token.

| Environment | Seed count |
|---|---|
| `production` | 2,222 |
| default (unnamed env) | 2,222 |
| `local` | 88 |
| `testing` | none — the call is commented out ("deployment unreliable. gacha autocreates upon mint") |

> Source: `deployment/world/state/index.ts:64–80` (local 66, testing 71–72,
> production 76, default 78), `deployment/world/state/gacha.ts:3–12`,
> `_721BatchMinterSystem.sol:312–336, 390–394`

### Supply Cap and Pool Drawdown

`Kami721.MAX_SUPPLY` is **22,222**, a contract constant enforced on both 721
mint entrypoints (`"Kami721: max supply reached"`, `Kami721.sol:44, 88–97`).
Both Kami creation paths mint a 721 against that counter — `LibKamiCreate`
derives the next index from `totalSupply() + 1` and calls `nft.mint`; the batch
minter calls `mintBatch` — so the pool seed consumes 2,222 of the 22,222 token
IDs before any player mints.

Creation **stops at the cap instead of reverting**:

- `LibKamiCreate.create()` returns `0` without creating anything when
  `totalSupply() ≥ MAX_SUPPLY`
- `LibKamiCreate.create(amt)` clamps `amt` to the remaining headroom and
  returns only the Kamis actually created
- `getSupplyHeadroom()` = `max(0, MAX_SUPPLY − totalSupply())`
- The admin batch minter clamps its batch the same way and returns an empty
  array at zero headroom

So the pool has two regimes:

| Regime | Mint | Reroll | Pool size per claim |
|---|---|---|---|
| **Headroom > 0** | creates one new Kami into the pool per ticket; the reveal draws one out | deposits one, the reveal draws one out | unchanged — the pool stays at its seed size, and every mint raises total Kami supply by one |
| **Headroom = 0** (cap reached) | creates nothing; the reveal still draws one out | deposits one, the reveal draws one out | **−1 per mint** — the pool is a drawdown; rerolls stay neutral |

In the drawdown regime the pool can empty. A reveal against an empty pool
cannot select a Kami (the no-replacement draw takes `seed mod poolSize`), so
claims are refused **up front**, before any ticket is spent, with the revert
`"gacha pool exhausted"`:

| Entrypoint | Refused unless |
|---|---|
| `KamiGachaMintSystem` (mint `amount`) | `getNumFree() + getSupplyHeadroom() ≥ amount` |
| `KamiGachaRerollSystem` (reroll) | `getNumFree() > 0` |
| `AuctionBuySystem`, buying Gacha Tickets (item 10) | `getNumFree() + getSupplyHeadroom() > 0` |
| `AuctionBuySystem`, buying Reroll Tokens (item 11) | `getNumFree() > 0` |

The auction checks stop **selling** tickets once nothing can be drawn, but do
not ration tickets against the pool: tickets already held, or bought while a
single free Kami remains, can outnumber what the pool can serve, and surplus
tickets then fail the mint check.

Before the cap, the seeded Kamis stay resident and the ceiling on gacha-minted
Kamis is `22,222 − 2,222 = 20,000` over the world's lifetime. After the cap,
the 2,222 seeded Kamis (plus any later top-ups) are what mint claims draw down.

> ⚠️ UNCERTAIN: whether the cap has been reached is chain state
> (`Kami721.totalSupply()`), not source. The upstream change that introduced
> the drawdown regime was made because the production Kami721 had reached
> 22,222.

> ⚠️ The 2,222 figure is the seed the **deployment script** passes; it is not
> asserted on chain, and the pool can be topped up afterwards at any size via
> `mintToGachaPool` (`gacha.ts:14–21`, exposed as `world.ts:99`) — but only
> while headroom remains, since the batch minter is capped too. Unrevealed mint
> commits made with headroom leave their new Kami resident in the pool until
> revealed.

> Source: `Kami721.sol:44, 88–97`, `LibKamiCreate.sol:75–101, 164–174`,
> `_721BatchMinterSystem.sol:312–336, 390–394`, `KamiGachaMintSystem.sol:30–42`,
> `KamiGachaRerollSystem.sol:28–29`, `AuctionBuySystem.sol:27–37`,
> `LibGacha.sol:89–121`, `LibRandom.sol:78–91`

## Commit-Reveal Pattern

All gacha operations (mint and reroll) use a **two-step commit-reveal** pattern
powered by `LibCommit`:

### Commit (Step 1)

Creates one or more commit entities with:

| Component | Value |
|---|---|
| `BlockReveal` | Target block number for randomness |
| `IdHolder` | Account ID of the committer |
| `Type` | `"GACHA_COMMIT"` |

Commit IDs are derived via iterative chaining (each feeds into the next):
```
id = world.getUniqueEntityId()
for i in 0..amount:
    id = keccak256(id, i)
    commitID[i] = id
```

Every gacha commit also adds its amount to the `GACHA_COMMITS_PENDING`
counter (see [Pending Commits](#pending-commits-and-free-kamis)).

> Source: `LibCommit.sol:44–64`, `LibGacha.sol:31–40`

### Reveal (Step 2)

`KamiGachaRevealSystem.reveal(commitIDs)`:

1. Reject an empty array (`"need commits to reveal"`)
2. **Sort and reject duplicates** — `LibArray.sortAndVerifyNoRepeats` sorts the
   IDs in place and reverts `"LibArray: detected duplicate in array"` if the
   same commit appears twice. Since commit IDs are keccak hashes, the resulting
   order is arbitrary numeric ordering, **not** chronological (the source
   comment claims chronological ordering, but the values sorted are raw hashes)
3. Verify all commits are of type `GACHA_COMMIT` (reverts
   `"LibGacha: invalid commit ID"`)
4. Extract seeds from blockhashes: `seed = keccak256(blockhash(revealBlock), entityID)`
5. Select random Kamis from pool using seeds (no-replacement sampling)
6. Transfer selected Kamis from pool to committers' accounts
7. Increment reroll counter on each withdrawn Kami
8. Release the revealed commits from `GACHA_COMMITS_PENDING` (saturating at 0)

Sorting and duplicate rejection happen in one step
(`LibArray.sortAndVerifyNoRepeats`) — see
[Commit-Reveal → Duplicate-ID Rejection](../utility/commit-reveal.md#duplicate-id-rejection).

The reveal is **owner-agnostic** — anyone can trigger it, and Kamis are sent to
the original committer (stored in `IdHolder`).

**Reveal window:** commits store `revealBlock = block.number` (the commit
block), but reveal is impossible in the commit block itself —
`blockhash(block.number)` returns 0 for the current block, which makes
`LibCommit.hashSeed` revert. Since `blockhash` is also only available for the
most recent 256 blocks, the effective reveal window is
**[commit block + 1, commit block + 256]**.

> Source: `KamiGachaRevealSystem.sol:18–28`, `LibGacha.sol:57–83, 89–100,
> 126–131`, `LibCommit.sol:137–142`

### Force Reveal (Community Manager)

If a player misses the **256-block window** (after which `blockhash()` returns
0), a community manager can call `forceReveal()`. The function is gated by
`onlyCommManager` (requires the caller to hold the `ROLE_COMMUNITY_MANAGER`
flag). It:

1. Checks that the blockhash is no longer available — but this guard never
   fires (the array overload of `LibCommit.isAvailable` always returns
   `false`; see
   [commit-reveal.md → Force Reveal](../utility/commit-reveal.md#force-reveal-community-manager-recovery)),
   so a force reveal also works on commits still inside their window
2. Resets commit blocks to `block.number - 1` (generating new seeds)
3. Proceeds with normal reveal flow

> Source: `KamiGachaRevealSystem.sol:30–49`, `LibCommit.sol:77–82`,
> `AuthRoles.sol:12–14`

## Minting

`KamiGachaMintSystem.execute(amount)`:

1. Verify `amount <= 5` (`"too many mints"`)
2. Resolve the account from the **owner** wallet
3. **Pool floor check** — `getNumFree() + getSupplyHeadroom() ≥ amount`, else
   `"gacha pool exhausted"`; runs before the ticket burn, so a refused mint
   keeps its tickets
4. Deduct `amount` Gacha Tickets (item 10) from player's inventory
5. Create commits with `revealBlock = block.number` (and add `amount` to
   `GACHA_COMMITS_PENDING`)
6. Create up to `amount` new Kamis via `LibKamiCreate.create(amount)` —
   clamped to the remaining 721 headroom, so **none** at the cap
7. Log mint

Kamis created in step 6 go directly into the gacha pool. On reveal, the player
receives random Kamis from the pool (not necessarily the ones just created).

> Source: `KamiGachaMintSystem.sol:20–48`, `LibKamiCreate.sol:95–101`

## Rerolling

`KamiGachaRerollSystem.reroll(kamiIDs)`:

1. **Sort and reject duplicates** in `kamiIDs` — the same
   `LibArray.sortAndVerifyNoRepeats` guard used by reveal, so the same Kami
   cannot be submitted twice in one call
2. Verify all Kamis are owned by caller and in `RESTING` state
3. **Pool floor check** — `getNumFree() > 0`, else `"gacha pool exhausted"`
   (before the ticket burn: on an empty pool a reroll could only hand back
   the deposit)
4. **Force-unequip all items** from each Kami back to the player's inventory
   (`LibEquipment.unequipAll`) — equipment is recovered before the Kami leaves
   the player's ownership
5. Extract previous reroll counts from the Kamis
6. Deduct `kamiIDs.length` Reroll Tokens (item 11) from inventory
7. **Deposit** the player's Kamis into the gacha pool (ownership → `GACHA_ID`)
8. Create commits (same amount as Kamis deposited; added to
   `GACHA_COMMITS_PENDING`)
9. Store previous reroll counts on the commit entities
10. Log reroll

On reveal, the player receives the same number of random Kamis. Each received
Kami's counter is set to the value carried on the commit plus 1 (see Reroll
Counter below).

> Source: `KamiGachaRerollSystem.sol:21–58`, `LibGacha.sol:31–76`

## Reroll Counter

Each Kami's `RerollComponent` counts **withdrawals from the gacha pool**, not
rerolls. The counter:
- Is cleared when a Kami enters the pool (`depositPets` removes it)
- Is transferred from the old Kami to the commit during reroll (mint commits
  carry no value, read as 0)
- Is set to the carried value + 1 on **every** withdrawal from the pool
  (`LibGacha.withdrawPets`)

Because the increment applies to every withdrawal, a freshly minted,
never-rerolled Kami leaves the pool with `Reroll = 1`.

> Source: `LibGacha.sol:46–76` (increment at 62–67)

## Random Selection

`LibGacha.selectPets()` draws Kamis from the pool:

1. Extract seeds from commit blockhashes
2. Use `LibRandom.getRandomBatchNoReplacement(seeds, poolSize)` to compute
   indices into a shrinking pool: `index[i] = seed[i] mod (poolSize − i)`. The
   numeric indices themselves **can** repeat across draws
3. Extract Kamis at those indices, removing each selected Kami from the pool
   (swap-with-last removal pattern) before the next draw

No duplicate draws occur within a single reveal batch — not because the indices
are unique, but because each selected Kami is removed from the pool between
draws, so a repeated index lands on a different Kami.

> Source: `LibGacha.sol:89–121` (pool removal at 112–118), `LibRandom.sol:78–91`

## Buying Gacha Tickets

`GachaBuyTicketSystem` provides two purchase paths:

### Public Mint

`buyPublic(amount)`:

1. Verify global total hasn't exceeded `MINT_MAX_TOTAL`
2. Verify public mint has started: `block.timestamp >= MINT_START_PUBLIC`
3. Verify per-account limit: `MINT_NUM_PUBLIC + amount <= MINT_MAX_PUBLIC`
4. Deduct cost: `amount × MINT_PRICE_PUBLIC` units of item 103
5. Grant Gacha Tickets

### Whitelist Mint

`buyWL()` — always buys exactly 1:

1. Verify global total hasn't exceeded `MINT_MAX_TOTAL`
2. Verify account has `MINT_WHITELISTED` flag
3. Verify WL mint has started: `block.timestamp >= MINT_START_WL`
4. Verify per-account limit: `MINT_NUM_WL + 1 <= MINT_MAX_WL`
5. Deduct cost: `MINT_PRICE_WL` units of item 103
6. Grant 1 Gacha Ticket

Both paths debit **item 103** (`CURRENCY = 103`, commented "ETH token index").
Item 103 is now the **Ether Shard**, the portal-bridged ETH item at scale 5
(1 unit = 0.00001 ETH) — see [token-portal.md](../marketplace/token-portal.md).
The config comments price the mint in milli-ETH (`50 // 0.05 ETH, 50 mETH`),
which no longer matches that scale: at scale 5 the configured `50` and `100`
are 50 and 100 Ether Shards (0.0005 and 0.001 ETH). Both paths are also capped
globally by `MINT_MAX_TOTAL` (`"max mints reached"`). Unlike the auction
ticket sale, neither path checks the gacha pool (`"gacha pool exhausted"`
applies only at mint, reroll and auction purchase).

> ⚠️ UNCERTAIN: the live `MINT_PRICE_*` values and how much of `MINT_MAX_TOTAL`
> has already been consumed (`MINT_NUM_TOTAL` on holder 0) are chain state, so
> whether these paths can still sell tickets, and at what effective ETH price,
> is not determinable from source. Gacha Tickets are also sold by the
> MUSU-priced auction (see [auctions.md](../marketplace/auctions.md)).

> Source: `GachaBuyTicketSystem.sol:17, 42–79, 103–109`, `configs.ts:97–109`,
> `data/portal/tokens.csv`

## Mint Config

| Config Key | Value | Description |
|---|---|---|
| `MINT_MAX_TOTAL` | 3000 | Maximum total tickets purchasable globally |
| `MINT_START_WL` | 1746086400 (2025-05-01 08:00 UTC) | Whitelist mint start time |
| `MINT_PRICE_WL` | 50 (units of item 103) | Whitelist ticket price |
| `MINT_MAX_WL` | 1 | Max WL tickets per account |
| `MINT_START_PUBLIC` | 1746086400 (2025-05-01 08:00 UTC) | Public mint start time (production) |
| `MINT_PRICE_PUBLIC` | 100 (units of item 103) | Public ticket price |
| `MINT_MAX_PUBLIC` | 222 | Max public tickets per account |

The base `initMint` sets `MINT_START_PUBLIC = 0`. Because the gate is
`block.timestamp < MINT_START_PUBLIC`, a value of 0 means **always open**, not
disabled. Production deploys additionally run `initProdConfigs`, which sets
both `MINT_START_WL` and `MINT_START_PUBLIC` to 1746086400 — public mint opens
at the same time as the whitelist mint.

`GACHA_REROLL_PRICE` is a dead config: its only reader,
`LibGacha.getBaseRerollCost` (`LibGacha.sol:136–138`), has no callers, and the
key is never set by any init script. The only reroll cost is 1 Reroll Token
(item 11) per Kami (`KamiGachaRerollSystem.sol:40`).

> Source: `configs.ts:96–110` (initMint), `configs.ts:53–56` (initProdConfigs),
> `deployment/world/state/index.ts:73–75, 97–99`,
> `GachaBuyTicketSystem.sol:17–36, 116–118`

## Logging

| Data Key | Scope | Description |
|---|---|---|
| `KAMI_GACHA_MINT` | Per account | Number of gacha mints |
| `KAMI_GACHA_REROLL` | Per account | Number of gacha rerolls |
| `GACHA_COMMITS_PENDING` | Global (holder 0, index 0) | Unrevealed gacha commits (see [Pending Commits](#pending-commits-and-free-kamis)) |
| `MINT_NUM_WL` | Per account + global | WL tickets purchased |
| `MINT_NUM_PUBLIC` | Per account + global | Public tickets purchased |
| `MINT_NUM_TOTAL` | Per account + global | Total tickets purchased |
| `MINT` | Event | Emitted on ticket purchase (accID, amount, cost) |

> Source: `LibGacha.sol:24, 182–188`, `GachaBuyTicketSystem.sol:90–100`
