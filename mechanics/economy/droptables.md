# Droptables & Loot

> Source: `packages/contracts/src/libraries/LibDroptable.sol` (L1–220),
> `packages/contracts/src/libraries/LibCommit.sol` (L1–168),
> `packages/contracts/src/systems/DroptableRevealSystem.sol` (L1–50)

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

> Source: `LibDroptable.sol:143–151`, `LibRandom.sol:32–34`

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

> Source: `LibDroptable.sol:31–41`, `LibCommit.sol:29–40`

### Step 2: Reveal (next block or later)

`DroptableRevealSystem.execute(commitIDs[])`:

1. **Verify commits** — all IDs must be `ITEM_DROPTABLE_COMMIT` type
2. **Filter invalid** — skip already-revealed or missing commits (replace with 0)
3. **For each commit**:
   a. Extract the stored block number and generate seed: `seed = keccak256(blockhash(blockNum), commitID)`
   b. Load droptable weights, process rarities
   c. Run weighted random selection `count` times
   d. Distribute selected items to the holder's inventory
   e. Emit reveal event

> Source: `DroptableRevealSystem.sol:22–33`, `LibDroptable.sol:48–115`

### Seed Generation

```
seed = keccak256(blockhash(commitBlockNumber), commitEntityID)
```

The `blockhash` is only available for the most recent 256 blocks. If the reveal
happens too late, the blockhash returns 0 and the transaction reverts. An admin
`forceReveal` function can reset the block to `block.number - 1` to rescue
stuck commits.

> Source: `LibCommit.sol:134–138`, `DroptableRevealSystem.sol:35–45`

## Weighted Selection

For each roll, the system selects one item from the droptable:

1. Weights are converted in place via `processWeightedRarityInPlace()` — each
   weight `w` becomes `0` if `w = 0`, else `2^(w−1)` (`LibRandom.sol:28–34`)
2. `selectMultipleFromWeighted(weights, seed, count)` performs `count`
   independent weighted random selections; each roll takes
   `randN mod totalWeight` and walks the cumulative weight array to find the
   selected index (`_positionFromWeighted`, `LibRandom.sol:238–255`)
3. Returns an array of amounts per item index (how many of each item was selected)

> Source: `LibDroptable.sol:87–100`, `LibRandom.sol:28–34, 238–255`

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
