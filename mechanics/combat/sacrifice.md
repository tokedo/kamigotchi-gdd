# Sacrifice

> Source: `packages/contracts/src/libraries/LibSacrifice.sol` (L1–437),
> `packages/contracts/src/systems/KamiSacrificeCommitSystem.sol` (L1–70),
> `packages/contracts/src/systems/KamiSacrificeRevealSystem.sol` (L1–67)

## Overview

Sacrifice permanently **burns a Kami** in exchange for a random item reward.
It uses the commit-reveal pattern (see
[droptables.md](../economy/droptables.md#commit-reveal-pattern)) for
manipulation-resistant randomness. A **pity system** guarantees higher-quality
rewards at regular intervals.

## Requirements

- Kami must be owned by the caller's account
- Kami must be in `RESTING` state
- One Kami per transaction

> Source: `KamiSacrificeCommitSystem.sol:36–53`

## Sacrifice Process

### Step 1: Commit

`KamiSacrificeCommitSystem.execute(kamiIndex)`:

1. Verify ownership and RESTING state
2. Sync Kami state (update health, etc.)
3. Increment the account's **pity counter**
4. Select which droptable to use based on pity count
5. Create commit entity (`KAMI_SACRIFICE_COMMIT` type) with droptable ID and
   Kami ID stored
6. **Force-unequip all items** back to the account's inventory
   (`LibEquipment.unequipAll`) — equipment is recovered, not burned with the Kami
7. **Burn the Kami**:
   - Transfer ERC-721 token to burn address (`0x...dEaD`)
   - Set Kami state to `DEAD`, health to 0
   - Clear ownership (Kami no longer appears in party)

> Source: `LibSacrifice.sol:62–91, 98–110`

### Step 2: Reveal

`KamiSacrificeRevealSystem.execute(commitIDs[])`:

1. Validate all commits are `KAMI_SACRIFICE_COMMIT` type
2. Filter out already-revealed or invalid commits
3. For each commit:
   a. Extract droptable ID and holder account
   b. Generate seed from blockhash (same as droptable reveal)
   c. Select **1 item** from the droptable via weighted random
   d. Distribute item to the account's inventory
   e. Emit reveal event

> Source: `LibSacrifice.sol:121–161, KamiSacrificeRevealSystem.sol:31–48`

## Pity System

The pity system guarantees better rewards at fixed intervals. Each account has
a running sacrifice counter.

| Threshold | Every N sacrifices | Droptable |
|---|---|---|
| Rare pity | 100 | `droptable.sacrifice.rare` |
| Uncommon pity | 20 | `droptable.sacrifice.uncommon` |
| Normal | (default) | `droptable.sacrifice.normal` |

**Rare pity takes precedence** when both thresholds align (e.g., sacrifice #100
uses rare, not uncommon).

Pity counter entity: `keccak256("sacrifice.pity", accID)`

> Source: `LibSacrifice.sol:32–38, 205–247`

## Droptables

Three separate droptables with different reward pools:

| Droptable ID | Purpose |
|---|---|
| `keccak256("droptable.sacrifice.normal")` | Standard sacrifice rewards |
| `keccak256("droptable.sacrifice.uncommon")` | Guaranteed uncommon+ items |
| `keccak256("droptable.sacrifice.rare")` | Guaranteed rare+ items |

Each droptable has `Keys` (item indices) and `Weights` (rarity weights),
registered via `_SacrificeRegistrySystem`.

Stored weights are **not** used linearly: before selection, each weight `w` is
transformed to `2^(w−1)` (weight 0 stays 0) via
`LibRandom.processWeightedRarityInPlace` (`LibSacrifice.sol:178`,
`LibRandom.sol:28–34`) — selection probability is **exponential** in the
stored value. See
[droptables.md](../economy/droptables.md#weighted-selection).

### Reward Pools & Odds

All three tables draw from the **36 Pet-Slot equipment items**, indices
30001–30036: 12 themed sets of 3 tiers each. Within a set the tiers run in
index order, so `index mod 3` identifies the tier — `1` → Common, `2` →
Uncommon, `0` → Rare (e.g. 30001 Mask of Avarice / 30002 Veil of Avarice /
30003 Visage of Avarice). Every item of a given tier carries the same stored
weight, so each table is uniform *within* a tier.

| Table | Tiers included | Stored weights | Effective weight per item |
|---|---|---|---|
| Sacrifice Normal | Common, Uncommon, Rare | 9 / 7 / 4 | 256 / 64 / 8 |
| Sacrifice Uncommon Pity | Uncommon, Rare | 7 / 4 | 64 / 8 |
| Sacrifice Rare Pity | Rare only | 4 | 8 |

Resulting tier odds for the single roll each sacrifice grants:

| Table | Common | Uncommon | Rare |
|---|---|---|---|
| Sacrifice Normal | 78.05% (6.50% per item) | 19.51% (1.63% per item) | 2.44% (0.203% per item) |
| Sacrifice Uncommon Pity | — | 88.89% (7.41% per item) | 11.11% (0.926% per item) |
| Sacrifice Rare Pity | — | — | 100% (8.33% per item) |

> Source: `LibSacrifice.sol:32–34, 272–280`,
> `deployment/world/data/npc/droptables.csv`; tier mapping from
> `catalogs/items/items.csv` (30001–30036)

## Burn Mechanics

The burn is **permanent and irreversible**:
- ERC-721 token transferred to `0x000000000000000000000000000000000000dEaD`
- Kami killed via `LibKami.kill()` (state = DEAD, HP = 0)
- Ownership cleared (`IDOwnsKami` set to 0)

Unlike normal death, sacrificed Kamis **cannot be revived**.

> Source: `LibSacrifice.sol:29, 98–110`

## Tracking & Logging

| Data Key | Scope | Description |
|---|---|---|
| `KAMI_SACRIFICE` | Per account | Total sacrifices by this account |
| `KAMI_SACRIFICE_TOTAL` | Global | Total sacrifices across all accounts |
| `SACRIFICE_RARE_PITY` | Per account | Rare pity triggers |
| `SACRIFICE_RARE_PITY_TOTAL` | Global | Total rare pity triggers |
| `SACRIFICE_UNCOMMON_PITY` | Per account | Uncommon pity triggers |
| `SACRIFICE_UNCOMMON_PITY_TOTAL` | Global | Total uncommon pity triggers |
| `SACRIFICE_ITEM_TOTAL` | Per account per item | Items received from sacrifice |

> Source: `LibSacrifice.sol:285–321`
