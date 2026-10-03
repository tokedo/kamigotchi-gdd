# Commit-Reveal Pattern

> Source: `packages/contracts/src/libraries/LibCommit.sol` (L1–172),
> `packages/contracts/src/libraries/utils/LibArray.sol` (L84–92)

## Overview

The commit-reveal pattern is the **core randomness mechanism** in Kamigotchi.
Since blockchains are deterministic, true randomness is obtained by committing
to a future block number, then using that block's hash as entropy after it is
mined.

## How It Works

### Step 1: Commit

A commit entity is created with:

| Component | Description |
|---|---|
| `BlockReveal` | Target block number to use for randomness |
| `IdHolder` | Entity that owns this commit (e.g., account ID) |
| `Type` | Commit type string (e.g., `"GACHA_COMMIT"`, `"ITEM_DROPTABLE_COMMIT"`) |

Batch commits derive IDs via **iterative chaining** (each ID feeds into the next):
```
id = world.getUniqueEntityId()
for i in 0..amount:
    id = keccak256(id, i)     // each iteration uses the previous id
    commitID[i] = id
```

> Source: `LibCommit.sol:29–64`

### Step 2: Reveal

After the target block is mined, the seed is derived:

```
seed = keccak256(blockhash(revealBlock), entityID)
```

The seed is then used by `LibRandom` for selection.

Two read modes exist:

| Helper | Behaviour |
|---|---|
| `extractSeedDirect` | Reads **and clears** the commit's `BlockReveal` — single-shot reveal |
| `seedDirect` | Reads without clearing, so the same reveal block can be reused across several transactions — used by [chunked droptable reveal](../economy/droptables.md#chunked-reveal-large-commits) |

> Source: `LibCommit.sol:88–102`, `:134–138`

### Duplicate-ID Rejection

Every batch reveal entrypoint first sorts its ID array **in place** and
rejects repeats:

```
LibArray.sortAndVerifyNoRepeats(ids)
// insertion-sort, uniquify, compare lengths
// revert "LibArray: detected duplicate in array" if any were removed
```

Passing the same commit twice in one call is therefore an outright revert, not
a silently-deduplicated no-op. The sort is not incidental — gacha reveal
consumes its commit array in ascending ID order, so sorting and
duplicate-checking are folded into one step. (Commit IDs are keccak hashes, so
that ordering is arbitrary rather than chronological — see
[Gacha → Reveal](../gacha/gacha.md#reveal-step-2).)

Applies to:

| Entrypoint | Array checked |
|---|---|
| `DroptableRevealSystem.execute` | commit IDs |
| `KamiGachaRevealSystem.reveal` | commit IDs |
| `KamiGachaRevealSystem.forceReveal` | commit IDs |
| `KamiGachaRerollSystem.reroll` | Kami IDs |

In `DroptableRevealSystem`, the sort must run **before** `filterInvalid`,
which zeroes already-drained entries — otherwise several zeros would present
as duplicates.

> Source: `LibArray.sol:84–92`, `DroptableRevealSystem.sol:22–25`,
> `KamiGachaRevealSystem.sol:20, 35`, `KamiGachaRerollSystem.sol:24`

## 256-Block Window

`blockhash()` only returns values for the most recent 256 blocks. If a reveal
is not claimed within 256 blocks, the blockhash returns 0 and the reveal
fails with `"Blockhash unavailable. Contact admin"`. The wall-clock duration of
the window depends on Yominet's block time, which nothing in the contracts
fixes.

Every commit in the game stores the **commit transaction's own block** as its
reveal block. `blockhash(block.number)` is 0 inside that same block, so a
reveal cannot land in the commit block either. The effective window is
therefore **blocks `commit + 1` through `commit + 256`**.

### Constants at a Glance

| Constant / rule | Value | Source |
|---|---|---|
| Reveal block stored at commit | `block.number` of the commit tx — gacha mint, gacha reroll, droptable (lootbox / scavenge), sacrifice | `KamiGachaMintSystem.sol:41`, `KamiGachaRerollSystem.sol:44–50`, `LibDroptable.sol:47`, `LibSacrifice.sol:76` |
| Reveal window (expiry) | blocks `commit + 1` … `commit + 256`; outside it the reveal reverts `"Blockhash unavailable. Contact admin"` | `LibCommit.sol:137–142` |
| `MAX_ROLLS_PER_REVEAL` | `5000` rolls per droptable reveal transaction; a larger commit is revealed in chunks across transactions, each chunk re-reading the **same** reveal blockhash, with outcomes independent of the chunking | `LibDroptable.sol:26–30, 55–68` — see [droptables.md → Chunked Reveal](../economy/droptables.md#chunked-reveal-large-commits) |
| Gacha mints per transaction | at most `5` (`"too many mints"`) | `KamiGachaMintSystem.sol:22` |

Because chunks share one reveal block, a chunked drain must also finish within
the 256-block window; a stalled remainder needs a force reveal.

> ⚠️  UNCERTAIN: Yominet's actual block time (and therefore the wall-clock
> length of the 256-block window) is not defined in the source code.

### Force Reveal (Community Manager Recovery)

When the window is missed, a community manager (`ROLE_COMMUNITY_MANAGER`) can
force-reveal — both `forceReveal` entrypoints are gated by `onlyCommManager`
(`DroptableRevealSystem.sol:32`, `KamiGachaRevealSystem.sol:31–33`):
1. Call `if (LibCommit.isAvailable(components, ids)) revert("no need for force
   reveal")` — but see the note below: this guard never fires
2. Reset the commit's block to `block.number - 1`
3. Proceed with normal reveal flow using the new blockhash

> ⚠️ The guard in step 1 is inert. Both entrypoints call the **array**
> overload `isAvailable(components, uint256[])`, whose loop returns `false` on
> the first unavailable blockhash and otherwise falls off the end without a
> `return true` — so it returns `false` for every input. `"no need for force
> reveal"` can never be raised, and a force reveal re-seeds a commit (from the
> previous block's hash) even while its original blockhash is still available.

> Source: `LibCommit.sol:67–82, 147–153`, `DroptableRevealSystem.sol:32–42`,
> `KamiGachaRevealSystem.sol:30–49`

## Availability Check

```solidity
LibCommit.isAvailable(blockNum) → bool
// true if blockhash(blockNum) != 0
```

The per-commit overload `isAvailable(components, id)` reads the commit's
reveal block and applies the same test. The array overload
`isAvailable(components, ids)` always returns `false` (see the force-reveal
note above).

> Source: `LibCommit.sol:67–82`

## Usage

| System | Commit Type | Purpose |
|---|---|---|
| Gacha mint/reroll | `GACHA_COMMIT` | Random Kami selection from pool (`LibGacha.sol:39, 129`) |
| Droptable rewards | `ITEM_DROPTABLE_COMMIT` | Random loot from weighted tables (`LibDroptable.sol:47, 206`) |
| Sacrifice | `KAMI_SACRIFICE_COMMIT` | Random sacrifice outcome (`LibSacrifice.sol:76, 262`) |
