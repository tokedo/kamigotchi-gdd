# Gacha System (Mint / Reroll / Reveal)

> Source: `packages/contracts/src/libraries/LibGacha.sol` (L1–184),
> `packages/contracts/src/systems/KamiGachaMintSystem.sol` (L1–46),
> `packages/contracts/src/systems/KamiGachaRerollSystem.sol` (L1–55),
> `packages/contracts/src/systems/KamiGachaRevealSystem.sol` (L1–57),
> `packages/contracts/src/systems/GachaBuyTicketSystem.sol` (L1–143)

## Overview

The gacha system is the primary mechanism for players to **obtain new Kamis**.
It uses a commit-reveal pattern based on future block hashes for verifiable
randomness. Players spend **Gacha Tickets** (item 10) to mint, or **Reroll
Tokens** (item 11) to exchange existing Kamis for new random ones.

The gacha operates on a **pool** — a set of Kami entities owned by the
`GACHA_ID` sentinel. When a player mints, new Kamis are created and added to
the pool; then on reveal, random Kamis are drawn from the pool and assigned to
the player.

## Gacha Pool

Entity ID: `keccak256("gacha.id")`

The pool contains all Kamis whose `IDOwnsKami` component points to `GACHA_ID`.
Pool size is queried dynamically via `ownerComp.size(abi.encode(GACHA_ID))`.

> Source: `LibGacha.sol:21, 138–141`

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

Commit IDs are deterministically derived:
```
baseID = world.getUniqueEntityId()
commitID[i] = keccak256(baseID, i)
```

> Source: `LibCommit.sol:44–64`, `LibGacha.sol:27–35`

### Reveal (Step 2)

`KamiGachaRevealSystem.reveal(commitIDs)`:

1. Verify all commits are of type `GACHA_COMMIT`
2. Sort commits by entity ID (chronological ordering)
3. Extract seeds from blockhashes: `seed = keccak256(blockhash(revealBlock), entityID)`
4. Select random Kamis from pool using seeds (no-replacement sampling)
5. Transfer selected Kamis from pool to committers' accounts
6. Increment reroll counter on each withdrawn Kami

The reveal is **owner-agnostic** — anyone can trigger it, and Kamis are sent to
the original committer (stored in `IdHolder`).

> Source: `KamiGachaRevealSystem.sol:21–31`, `LibGacha.sol:87–98`

### Force Reveal (Admin)

If a player misses the **256-block window** (after which `blockhash()` returns
0), an admin can call `forceReveal()` which:

1. Verifies the blockhash is no longer available
2. Resets commit blocks to `block.number - 1` (generating new seeds)
3. Proceeds with normal reveal flow

> Source: `KamiGachaRevealSystem.sol:34–51`

## Minting

`KamiGachaMintSystem.execute(amount)`:

1. Verify `amount <= 5` (max mints per transaction)
2. Deduct `amount` Gacha Tickets (item 10) from player's inventory
3. Create commits with `revealBlock = block.number`
4. Create `amount` new Kami entities via `LibKamiCreate.create()` (added to pool)
5. Log mint

The newly created Kamis go directly into the gacha pool. On reveal, the player
receives random Kamis from the pool (not necessarily the ones just created).

> Source: `KamiGachaMintSystem.sol:20–41`

## Rerolling

`KamiGachaRerollSystem.reroll(kamiIDs)`:

1. Verify all Kamis are owned by caller and in `RESTING` state
2. Extract previous reroll counts from the Kamis
3. Deduct `kamiIDs.length` Reroll Tokens (item 11) from inventory
4. **Deposit** the player's Kamis into the gacha pool (ownership → `GACHA_ID`)
5. Create commits (same amount as Kamis deposited)
6. Store previous reroll counts on the commit entities
7. Log reroll

On reveal, the player receives the same number of random Kamis. Each received
Kami's reroll counter is incremented by 1 (tracking how many times it has been
rerolled).

> Source: `KamiGachaRerollSystem.sol:21–49`, `LibGacha.sol:41–69`

## Reroll Counter

Each Kami tracks how many times it has been rerolled via the `RerollComponent`.
This counter:
- Is cleared when a Kami enters the pool (`depositPets` removes it)
- Is transferred from the old Kami to the commit during reroll
- Is incremented (+1) when a Kami is withdrawn from the pool

> Source: `LibGacha.sol:41–69`

## Random Selection

`LibGacha.selectPets()` draws Kamis from the pool:

1. Extract seeds from commit blockhashes
2. Use `LibRandom.getRandomBatchNoReplacement(seeds, poolSize)` to get unique
   indices into the pool
3. Extract Kamis at those indices from the pool (swap-with-last removal pattern)

This ensures no duplicate draws within a single reveal batch.

> Source: `LibGacha.sol:87–118`

## Buying Gacha Tickets

`GachaBuyTicketSystem` provides two purchase paths:

### Public Mint

`buyPublic(amount)`:

1. Verify global total hasn't exceeded `MINT_MAX_TOTAL`
2. Verify public mint has started: `block.timestamp >= MINT_START_PUBLIC`
3. Verify per-account limit: `MINT_NUM_PUBLIC + amount <= MINT_MAX_PUBLIC`
4. Deduct cost: `amount × MINT_PRICE_PUBLIC` in ETH (item 103)
5. Grant Gacha Tickets

### Whitelist Mint

`buyWL()` — always buys exactly 1:

1. Verify global total hasn't exceeded `MINT_MAX_TOTAL`
2. Verify account has `MINT_WHITELISTED` flag
3. Verify WL mint has started: `block.timestamp >= MINT_START_WL`
4. Verify per-account limit: `MINT_NUM_WL + 1 <= MINT_MAX_WL`
5. Deduct cost: `MINT_PRICE_WL` in ETH (item 103)
6. Grant 1 Gacha Ticket

> Source: `GachaBuyTicketSystem.sol:42–79`

## Mint Config

| Config Key | Value | Description |
|---|---|---|
| `MINT_MAX_TOTAL` | 3000 | Maximum total tickets purchasable globally |
| `MINT_START_WL` | 1746086400 (2025-05-01 08:00 UTC) | Whitelist mint start time |
| `MINT_PRICE_WL` | 50 (0.05 ETH) | Whitelist ticket price |
| `MINT_MAX_WL` | 1 | Max WL tickets per account |
| `MINT_START_PUBLIC` | 0 (disabled in prod config) | Public mint start time |
| `MINT_PRICE_PUBLIC` | 100 (0.1 ETH) | Public ticket price |
| `MINT_MAX_PUBLIC` | 222 | Max public tickets per account |
| `GACHA_REROLL_PRICE` | (config) | Reroll cost (checked via `LibConfig`) |

> Source: `configs.ts:90–102`, `GachaBuyTicketSystem.sol:17–36`

## Logging

| Data Key | Scope | Description |
|---|---|---|
| `KAMI_GACHA_MINT` | Per account | Number of gacha mints |
| `KAMI_GACHA_REROLL` | Per account | Number of gacha rerolls |
| `MINT_NUM_WL` | Per account + global | WL tickets purchased |
| `MINT_NUM_PUBLIC` | Per account + global | Public tickets purchased |
| `MINT_NUM_TOTAL` | Per account + global | Total tickets purchased |
| `MINT` | Event | Emitted on ticket purchase (accID, amount, cost) |

> Source: `LibGacha.sol:169–175`, `GachaBuyTicketSystem.sol:90–100`
