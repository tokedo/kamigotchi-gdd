# Commit-Reveal Pattern

> Source: `packages/contracts/src/libraries/LibCommit.sol` (L1–168)

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

After the target block is mined, the seed is extracted:

```
seed = keccak256(blockhash(revealBlock), entityID)
```

The seed is then used by `LibRandom` for selection.

> Source: `LibCommit.sol:90–93, 134–138`

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
(`DroptableRevealSystem.sol:35`, `KamiGachaRevealSystem.sol:33–36`):
1. Verify the blockhash is no longer available (reverts
   `"no need for force reveal"` otherwise)
2. Reset the commit's block to `block.number - 1`
3. Proceed with normal reveal flow using the new blockhash

> Source: `LibCommit.sol:143–149`

## Availability Check

```solidity
LibCommit.isAvailable(blockNum) → bool
// true if blockhash(blockNum) != 0
```

> Source: `LibCommit.sol:69–71`

## Usage

| System | Commit Type | Purpose |
|---|---|---|
| Gacha mint/reroll | `GACHA_COMMIT` | Random Kami selection from pool (`LibGacha.sol:34, 127`) |
| Droptable rewards | `ITEM_DROPTABLE_COMMIT` | Random loot from weighted tables (`LibDroptable.sol:38, 137`) |
| Sacrifice | `KAMI_SACRIFICE_COMMIT` | Random sacrifice outcome (`LibSacrifice.sol:76, 262`) |
