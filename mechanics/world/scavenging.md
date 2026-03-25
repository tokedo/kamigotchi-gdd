# Scavenging

> Source: `packages/contracts/src/libraries/LibScavenge.sol` (L1–256),
> `packages/contracts/src/libraries/LibAllo.sol` (L1–357),
> `packages/contracts/src/systems/ScavengeClaimSystem.sol` (L1–41)

## Overview

Scavenging is a **secondary reward system** tied to harvest nodes. As Kamis
harvest, they accumulate **scavenge points** proportional to their harvest
output. When enough points accumulate to fill a tier, the player can claim
rewards (droptable rolls, items, bonuses, stats). The scavenge bar resets upon
claiming.

## Scavenge Bar (Registry)

Each node can have a scavenge bar — a registry entry defining how points
convert to rewards.

| Component | Description |
|---|---|
| `EntityType` | `"SCAVENGE"` |
| `IsRegistry` | Marks as registry entry |
| `Type` | Field type (e.g., `"NODE"`) |
| `Index` | Node index this bar belongs to |
| `Affinity` | Affinity type (matches node affinity) |
| `Value` | **Tier cost** — points needed per reward tier |

Registry ID: `keccak256("registry.scavenge", field, index)`

> Source: `LibScavenge.sol:23–38, 52–68`

## Point Accumulation

Points are incremented when a Kami collects or stops harvesting:

```
LibNode.scavenge(nodeIndex, harvestAmount, accountID)
→ LibScavenge.incFor("NODE", nodeIndex, harvestAmount, accountID)
```

Points are stored per-account per-node:

Instance ID: `keccak256("scavenge.instance", field, index, holderID)`

The amount added equals the **harvest output** — more productive harvests
generate more scavenge points.

> Source: `LibScavenge.sol:87–96`, `LibNode.sol:125–133`

## Claiming Rewards

`ScavengeClaimSystem.execute(scavBarID)`:

1. Verify the entity is a valid scavenge bar
2. Calculate claimable tiers:
   ```
   tiers = floor(currentPoints / tierCost)
   remainingPoints = currentPoints % tierCost
   ```
3. If tiers = 0, revert (nothing to claim)
4. Distribute rewards for each tier
5. Log the claim

Points are reset to the remainder after claiming (partial progress toward
the next tier is preserved).

> Source: `ScavengeClaimSystem.sol:17–36`, `LibScavenge.sol:99–114`

## Reward Distribution

Rewards are stored as **allocation entities** (LibAllo) anchored to the
scavenge bar. Each tier claim distributes all configured rewards, multiplied
by the tier count.

Reward anchor: `keccak256("scavenge.reward", registryID)`

### Reward Types

| Allo Type | Description |
|---|---|
| `ITEM_DROPTABLE` | Creates a droptable commit (random loot via commit-reveal) |
| `STAT` | Applies stat modifications to the account/Kami |
| `BONUS` | Assigns temporary bonuses |
| `CLEAR_BONUS` | Clears all bonuses from holder |
| Basic types | Gives items, XP, reputation, etc. via `LibSetter` |
| `DISPLAY_ONLY` | No distribution — for UI display only |

The distribution function returns commit IDs for any droptable rewards
(which must be separately revealed via `DroptableRevealSystem`).

> Source: `LibAllo.sol:188–215, 228–261`, `LibScavenge.sol:116–129`

## Tier Cost Data

From node data, scavenge costs (tier costs) vary by node:

| Cost | Node Examples |
|---|---|
| 100 | Misty Riverside, Tunnel of Trees, starter nodes |
| 200 | Forest paths, cave rooms, mid-tier nodes |
| 300 | Deeper Forest Path, Airplane Crash, advanced nodes |
| 500 | Scrap Confluence, Techno Temple, premium nodes |

Higher scavenge costs mean more harvesting is needed per reward tier, but
these nodes typically have rarer reward droptables.

> Source: `data/rooms/nodes.csv`

## Logging

| Data Key | Scope | Description |
|---|---|---|
| `SCAV_CLAIM_{FIELD}` | Per account (total) | Total scavenge claims for this field |
| `SCAV_CLAIM_{FIELD}` | Per account per node | Claims at a specific node |
| `SCAV_CLAIM_AFFINITY_{AFF}` | Per account per affinity | Claims by node affinity type |

> Source: `LibScavenge.sol:170–180`
