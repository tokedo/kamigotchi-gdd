# Gacha System (Mint / Reroll / Reveal)

> Source: `packages/contracts/src/libraries/LibGacha.sol` (L1–172),
> `packages/contracts/src/systems/KamiGachaMintSystem.sol` (L1–46),
> `packages/contracts/src/systems/KamiGachaRerollSystem.sol` (L1–61),
> `packages/contracts/src/systems/KamiGachaRevealSystem.sol` (L1–55),
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

> Source: `LibGacha.sol:20, 126–129`

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

> Source: `LibCommit.sol:44–64`, `LibGacha.sol:27–35`

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

> Source: `KamiGachaRevealSystem.sol:18–28`, `LibGacha.sol:75–86, 112–117`,
> `LibCommit.sol:134–138`

### Force Reveal (Community Manager)

If a player misses the **256-block window** (after which `blockhash()` returns
0), a community manager can call `forceReveal()`. The function is gated by
`onlyCommManager` (requires the caller to hold the `ROLE_COMMUNITY_MANAGER`
flag). It:

1. Verifies the blockhash is no longer available
2. Resets commit blocks to `block.number - 1` (generating new seeds)
3. Proceeds with normal reveal flow

> Source: `KamiGachaRevealSystem.sol:30–49`, `AuthRoles.sol:12–14`

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

1. **Sort and reject duplicates** in `kamiIDs` — the same
   `LibArray.sortAndVerifyNoRepeats` guard used by reveal, so the same Kami
   cannot be submitted twice in one call
2. Verify all Kamis are owned by caller and in `RESTING` state
3. **Force-unequip all items** from each Kami back to the player's inventory
   (`LibEquipment.unequipAll`) — equipment is recovered before the Kami leaves
   the player's ownership
4. Extract previous reroll counts from the Kamis
5. Deduct `kamiIDs.length` Reroll Tokens (item 11) from inventory
6. **Deposit** the player's Kamis into the gacha pool (ownership → `GACHA_ID`)
7. Create commits (same amount as Kamis deposited)
8. Store previous reroll counts on the commit entities
9. Log reroll

On reveal, the player receives the same number of random Kamis. Each received
Kami's counter is set to the value carried on the commit plus 1 (see Reroll
Counter below).

> Source: `KamiGachaRerollSystem.sol:20–58`, `LibGacha.sol:41–73`

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

> Source: `LibGacha.sol:41–73` (increment at 61–66)

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

> Source: `LibGacha.sol:75–106` (pool removal at 98–104), `LibRandom.sol:78–91`

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
| `MINT_START_PUBLIC` | 1746086400 (2025-05-01 08:00 UTC) | Public mint start time (production) |
| `MINT_PRICE_PUBLIC` | 100 (0.1 ETH) | Public ticket price |
| `MINT_MAX_PUBLIC` | 222 | Max public tickets per account |

The base `initMint` sets `MINT_START_PUBLIC = 0`. Because the gate is
`block.timestamp < MINT_START_PUBLIC`, a value of 0 means **always open**, not
disabled. Production deploys additionally run `initProdConfigs`, which sets
both `MINT_START_WL` and `MINT_START_PUBLIC` to 1746086400 — public mint opens
at the same time as the whitelist mint.

`GACHA_REROLL_PRICE` is a dead config: its only reader,
`LibGacha.getBaseRerollCost` (`LibGacha.sol:122–124`), has no callers, and the
key is never set by any init script. The only reroll cost is 1 Reroll Token
(item 11) per Kami (`KamiGachaRerollSystem.sol:37`).

> Source: `configs.ts:96–110` (initMint), `configs.ts:53–56` (initProdConfigs),
> `deployment/world/state/index.ts:73–75, 97–99`,
> `GachaBuyTicketSystem.sol:17–36, 116–118`

## Logging

| Data Key | Scope | Description |
|---|---|---|
| `KAMI_GACHA_MINT` | Per account | Number of gacha mints |
| `KAMI_GACHA_REROLL` | Per account | Number of gacha rerolls |
| `MINT_NUM_WL` | Per account + global | WL tickets purchased |
| `MINT_NUM_PUBLIC` | Per account + global | Public tickets purchased |
| `MINT_NUM_TOTAL` | Per account + global | Total tickets purchased |
| `MINT` | Event | Emitted on ticket purchase (accID, amount, cost) |

> Source: `LibGacha.sol:157–163`, `GachaBuyTicketSystem.sol:90–100`
