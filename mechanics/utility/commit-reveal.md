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
fails. The wall-clock duration of the window depends on Yominet's block time,
which nothing in the contracts fixes.

> ⚠️  UNCERTAIN: Yominet's actual block time (and therefore the wall-clock
> length of the 256-block window) is not defined in the source code.

### Force Reveal (Community Manager Recovery)

When the window is missed, a community manager (`ROLE_COMMUNITY_MANAGER`) can
force-reveal — both `forceReveal` entrypoints are gated by `onlyCommManager`
(`DroptableRevealSystem.sol:32`, `KamiGachaRevealSystem.sol:31–33`):
1. Verify the blockhash is no longer available (reverts
   `"no need for force reveal"` otherwise)
2. Reset the commit's block to `block.number - 1`
3. Proceed with normal reveal flow using the new blockhash

> Source: `LibCommit.sol:147–153`

## Availability Check

```solidity
LibCommit.isAvailable(blockNum) → bool
// true if blockhash(blockNum) != 0
```

> Source: `LibCommit.sol:69–71`

## Usage

| System | Commit Type | Purpose |
|---|---|---|
| Gacha mint/reroll | `GACHA_COMMIT` | Random Kami selection from pool (`LibGacha.sol:34, 115`) |
| Droptable rewards | `ITEM_DROPTABLE_COMMIT` | Random loot from weighted tables (`LibDroptable.sol:47, 206`) |
| Sacrifice | `KAMI_SACRIFICE_COMMIT` | Random sacrifice outcome (`LibSacrifice.sol:76, 262`) |
