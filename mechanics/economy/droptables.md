# Droptables & Loot

> Source: `packages/contracts/src/libraries/LibDroptable.sol` (L1–220),
> `packages/contracts/src/libraries/LibCommit.sol` (L1–168),
> `packages/contracts/src/systems/DroptableRevealSystem.sol` (L1–50)

## Overview

Droptables are weighted random loot tables that resolve items. They are used by
**lootbox items**, **scavenge rolls** (on harvest collect/stop), and **NPC drops**.
Resolution uses a **commit-reveal** pattern to prevent front-running.

See `catalogs/` for droptable data (items, rooms, NPCs).

## Droptable Structure

Each droptable is an entity with:

| Component | Description |
|---|---|
| `Keys` | Array of item indices (possible drops) |
| `Weights` | Array of rarity weights (selection probability) |

Higher weight = more likely to be selected. Weights are processed through
`LibRandom.processWeightedRarity()` before selection.

> Source: `LibDroptable.sol:143–151`

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

1. Weights are processed via `processWeightedRarityInPlace()` (converts raw rarity
   weights into cumulative probability distribution)
2. `selectMultipleFromWeighted(weights, seed, count)` performs `count` independent
   weighted random selections
3. Returns an array of amounts per item index (how many of each item was selected)

> Source: `LibDroptable.sol:87–100`

## Usage Contexts

### Lootbox Items
- Items with type `"LOOTBOX"` have an attached droptable
- Opening a lootbox calls `LibItem.droptableCommit()` → creates commit
- Player calls `DroptableRevealSystem` in a later transaction → items distributed

### Scavenge (Harvest Collect/Stop)
- When a Kami collects or stops harvesting, `LibNode.scavenge()` is triggered
- The scavenge roll is amount-weighted (more harvest output = more rolls)
- Uses the node's droptable

### NPC Drops
- NPCs can have droptables that resolve when interacted with

## Data Files

Droptable definitions are in CSV files:
- `data/items/droptables.csv` — item/lootbox droptables
- `data/rooms/droptables.csv` — room/node scavenge droptables
- `data/npc/droptables.csv` — NPC droptables
