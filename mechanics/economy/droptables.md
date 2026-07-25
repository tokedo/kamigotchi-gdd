# Droptables & Loot

> Source: `packages/contracts/src/libraries/LibDroptable.sol` (L1–305),
> `packages/contracts/src/libraries/LibCommit.sol` (L1–172),
> `packages/contracts/src/systems/DroptableRevealSystem.sol` (L1–47)

## Overview

Droptables are weighted random loot tables that resolve items. They are used by
**lootbox items**, **scavenge claims**, and the **Kami sacrifice ritual**.
Resolution uses a **commit-reveal** pattern to prevent front-running.

See `catalogs/` for droptable data (items, rooms, NPCs).

## Droptable Structure

Each droptable is an entity with:

| Component | Description |
|---|---|
| `Keys` | Array of item indices (possible drops) |
| `Weights` | Array of rarity weights (selection probability) |

Weights are stored raw (the CSV `Tiers` value) and converted at selection time
by `LibRandom.calcRarityWeight`: `w → 0` if `w = 0` (never drops), else
`w → 2^(w−1)`. Selection odds are therefore **exponential** in the stored
weight — each +1 doubles an entry's relative odds (a weight-9 entry is 256× as
likely as a weight-1 entry).

> Source: `LibDroptable.sol:139–141`, `LibRandom.sol:32–34`

## Commit-Reveal Pattern

Droptable resolution is a **two-transaction process** to prevent miners/players
from predicting or manipulating loot outcomes:

### Step 1: Commit

```
commitID = LibCommit.commit(world, components, accID, block.number, "ITEM_DROPTABLE_COMMIT")
```

Creates a commit entity storing:
- `BlockReveal` — the block number whose hash will be the randomness source
- `IdHolder` — the account that will receive the loot
- `Type` — `"ITEM_DROPTABLE_COMMIT"`
- `IdSource` — which droptable to resolve
- `Value` — number of rolls (count)

> Source: `LibDroptable.sol:40–50`, `LibCommit.sol:29–40`

### Step 2: Reveal (next block or later)

`DroptableRevealSystem.execute(commitIDs[])`:

1. **Reject empties** — revert `"ItemReveal: no reveals"` on an empty array
2. **Sort & deduplicate** — `LibArray.sortAndVerifyNoRepeats` sorts in place
   and reverts if the same ID appears twice. This must run **before** the
   filter step, which zeroes drained entries that would otherwise look like
   duplicate zeros
3. **Filter invalid** — already-drained or missing commits are replaced with 0
4. **Verify commits** — every non-zero ID must be `ITEM_DROPTABLE_COMMIT` type.
   The check is **non-destructive** (it reads the `Type` rather than extracting
   it) so a partially-revealed commit can be checked again on its next
   transaction
5. **For each commit, while roll budget remains**:
   a. Read (do not clear) the stored block number and derive the seed
   b. Load droptable weights, process rarities
   c. Run weighted random selection for this transaction's share of the rolls
   d. Distribute selected items to the holder's inventory
   e. Emit reveal event
   f. Delete the commit if fully drained, else write back the remainder

> Source: `DroptableRevealSystem.sol:19–27`, `LibDroptable.sol:61–71, 202–209`

### Chunked Reveal (Large Commits)

Reveal cost scales with a commit's roll count, so a large scavenge claim can
mint a single commit too big to reveal inside the block gas limit — stranding
it forever. Reveal is therefore **bounded per transaction**:

| Constant | Value | Meaning |
|---|---|---|
| `MAX_ROLLS_PER_REVEAL` | `5000` | Maximum rolls a single reveal transaction may process |

A transaction starts with a budget of `MAX_ROLLS_PER_REVEAL` and walks the
commit array:

- A commit smaller than the remaining budget is revealed whole and deleted
- A commit larger than the remaining budget consumes what is left; its `Value`
  is decremented by the amount processed and it stays alive for a later
  transaction
- Once the budget hits zero the loop breaks, so commits past that point simply
  wait their turn

**Outcomes do not depend on how the rolls are split.** Each roll is keyed by
its **absolute position within the commit**, counting down from the number of
rolls remaining at commit time:

```
rollSeed(k) = keccak256(seed, remaining − 1 − k)      for k = 0 … chunk−1
```

Since a single-pass reveal covers positions `count−1 … 0` — the same index set
as the unchunked selection helper — the full result is fixed by
`(blockhash, commitID, count)` at commit time and is byte-identical however it
is chunked. A per-chunk nonce would have let a co-bundled commit shift a chunk
boundary and reroll the distribution.

Because every chunk re-reads the **same** reveal blockhash (via
`LibCommit.seedDirect`, which reads without clearing), a drain that stalls past
the 256-block window leaves the remainder unrevealable through the normal path.
`forceReveal` / `resetBlocks` rescues the tail.

A fully-drained commit is deleted by `_consume`, which removes its `IdSource`,
`IdHolder`, `Value`, `BlockReveal` and `Type` components.

> Source: `LibDroptable.sol:26–30` (constant), `:61–71` (budget loop),
> `:74–89` (`_revealSingle`), `:132–157` (`_select`), `:159–165` (`_consume`),
> `LibCommit.sol:98–102` (`seedDirect`)

### Seed Generation

```
seed = keccak256(blockhash(commitBlockNumber), commitEntityID)
```

The `blockhash` is only available for the most recent 256 blocks. If the reveal
happens too late, the blockhash returns 0 and the transaction reverts. An admin
`forceReveal` function can reset the block to `block.number - 1` to rescue
stuck commits.

> Source: `LibCommit.sol:98–102, 134–138`, `DroptableRevealSystem.sol:32–42`

## Weighted Selection

For each roll, the system selects one item from the droptable:

1. Weights are converted in place via `processWeightedRarityInPlace()` — each
   weight `w` becomes `0` if `w = 0`, else `2^(w−1)` (`LibRandom.sol:28–34`)
2. Each roll takes `rollSeed mod totalWeight` and walks the cumulative weight
   array to find the selected index (`_positionFromWeighted`,
   `LibRandom.sol:238–255`)
3. Returns an array of amounts per item index (how many of each item was
   selected)

`LibDroptable._select` mirrors the shared `selectMultipleFromWeighted` helper
but offsets each roll by its absolute position in the commit, so the outcome
is chunk-invariant — see
[Chunked Reveal](#chunked-reveal-large-commits).

> Source: `LibDroptable.sol:132–157`, `LibRandom.sol:28–34, 211–228, 238–255`

## Usage Contexts

### Lootbox Items
- Lootbox items carry an `ITEM_DROPTABLE` allo on their `USE` use case; the
  droptable's `Keys`/`Weights` are stored on the **allo entity** itself
  (`LibAllo.createDT`, `LibAllo.sol:128–142`)
- Using the item → `LibItem.applyAllos` → `LibAllo.distribute` → `giveDT`, which
  calls `LibDroptable.commit(world, comps, alloID, rolls, accID)` with the
  **allo ID** as the droptable source (`LibAllo.sol:209, 252–261`)
- Player calls `DroptableRevealSystem` in a later transaction → items distributed

### Scavenge (Harvest Collect/Stop → Claim)
- Collecting or stopping a harvest only **increments the node's scav bar** by
  the harvest output (`LibNode.sol:132` → `LibScavenge.incFor`) — no rolls
  happen at that point
- `ScavengeClaimSystem` claims rewards: `rolls = ⌊points / tierCost⌋`, and the
  remainder `points mod tierCost` stays on the bar
  (`LibScavenge.extractNumTiers`, `LibScavenge.sol:99–114`)
- Rewards are distributed via `LibAllo.distribute` with the roll count as
  multiplier (`LibScavenge.sol:116–129`); droptable rewards there go through
  the same commit-reveal flow

### Kami Sacrifice (NPC droptable data)
- `data/npc/droptables.csv` contains exactly three tables — **Sacrifice
  Normal**, **Sacrifice Uncommon Pity**, **Sacrifice Rare Pity** — the reward
  tables for the Kami sacrifice ritual
- `LibSacrifice` selects among them by pity counter (`LibSacrifice.sol:33–35,
  229–236`) and stores the chosen table on the sacrifice commit
  (`LibSacrifice.sol:72–79`)
- There is no generic NPC-interaction droptable mechanic

## Data Files

Droptable definitions are in CSV files:
- `data/items/droptables.csv` — item/lootbox droptables
- `data/rooms/droptables.csv` — room/node scavenge droptables
- `data/npc/droptables.csv` — Kami sacrifice reward tables (see above)
