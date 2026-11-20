# Nodes (Sub-Locations)

> Source: `packages/contracts/src/libraries/LibNode.sol` (L1–256),
> `packages/contracts/deployment/world/data/rooms/nodes.csv`,
> `packages/contracts/deployment/world/state/rooms/nodes.ts` (L1–225)

## Overview

Nodes are sub-locations within rooms where Kamis perform activities. The primary
node type is `HARVEST` — nodes where Kamis can farm resources. Each node has an
affinity type that affects harvest effectiveness, a specific item it yields, and
optionally bonuses, requirements, and a scavenge bar.

See `catalogs/rooms/nodes.csv` for the full node catalog.

## Node Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"NODE"` |
| `IndexNode` | Unique node index (uint32) |
| `Type` | Node type: `"HARVEST"` (currently the only type) |
| `IndexItem` | Item yielded by harvesting on this node |
| `IndexRoom` | Room this node belongs to |
| `Name` | Display name |
| `Description` | Flavor text |
| `Affinity` | (optional) Node affinity: `NORMAL`, `EERIE`, `SCRAP`, `INSECT`, or compound (e.g., `"EERIE-SCRAP"`) |
| `HarvestMultiplier` | Harvest multiplier K, per mille (`1000` = ×1.0), low 16 bits — see [Harvest Multiplier (K)](#harvest-multiplier-k) |

Entity ID: `keccak256("node", nodeIndex)`

> Source: `LibNode.sol:31–76, 245–247`

## Node Types

The code supports multiple node types:

| Type | Description |
|---|---|
| `HARVEST` | Primary type — Kamis farm resources here |
| `HEALING` | Referenced in code comments but not actively used |
| `SACRIFICIAL` | Referenced in code comments but not actively used |
| `TRAINING` | Referenced in code comments but not actively used |

Currently all deployed nodes are `HARVEST` type.

> Source: `LibNode.sol:60, 189–191`

## Affinity System

Node affinities interact with Kami affinities to modify harvest effectiveness
and combat outcomes (see [harvesting.md](../economy/harvesting.md) for the
full efficacy formula).

**Affinity validation rule**: A node cannot combine `NORMAL` with a typed
affinity (e.g., `"NORMAL-EERIE"` is invalid). Compound affinities must be
two typed affinities (e.g., `"EERIE-SCRAP"` is valid). The on-chain format
is hyphen-separated — `isValidAffinity` splits on `"-"` (`LibNode.sol:195`),
and deployment converts the CSV's comma format via `.replace(',', '-')`
(`deployment/world/state/rooms/nodes.ts:16`).

> Source: `LibNode.sol:194–198`, `nodes.ts:16`

## Harvest Multiplier (K)

Every node carries a **harvest multiplier K**, stored per mille in the
`HarvestMultiplier` component (`component.harvest.multiplier`; `1000` = ×1.0,
today's output). A harvest credits its base output × K at each sync, with K
copied from the node when the stretch opens; HP strain stays on the base output
— the credit rule is in
[harvesting.md](../economy/harvesting.md#harvest-multiplier-k-what-a-sync-credits).

| Rule | Source |
|---|---|
| Precision `HARV_MULT_PREC = 1000`; bounds `HARV_MULT_MIN = 100` (×0.1) to `HARV_MULT_MAX = 60000` (×60), so a stored K is never 0 | `LibNode.sol:25–29` |
| Read as the low 16 bits (`uint16`) of the component value | `LibNode.sol:135–137` (`getBaseMultiplier`) |
| Written only by `LibNode.setMultiplierByIndex(index, k)`: reverts `"Node: does not exist"` for an unknown index and `"Node: multiplier out of range"` outside the bounds | `LibNode.sol:140–145` |
| **Set at creation**: `_NodeRegistrySystem.create` decodes a trailing `uint32 k` — `(index, type, item, room, name, description, affinity, k)` — and sets it right after creating the node | `_NodeRegistrySystem.sol:21–40` |
| **Changed on a live node**: `_NodeRegistrySystem.setMultiplier(uint32 index, uint32 k)`, `onlyAdmin`; harvests on the node pick it up at their next sync | `_NodeRegistrySystem.sol:42–45` |
| **Kept on removal**: `LibNode.remove` does not clear K, so a harvest on a node being revised (remove, then create) never reads a missing K; the re-create overwrites it | `LibNode.sol:119–120` |

Deployment reads K from the required `Harvest Multiplier` column of
`nodes.csv` as a decimal factor (`toPerMille`: `0.1` to `60`, at most three
decimals, stored ×1000; anything else throws before any call is queued, and a
revise checks every row before deleting a node). The world-state command
`nodes.setMultipliers <indices>` sets K from the CSV on live nodes without a
revise, so running harvests are untouched.

**At the pin every node has K = 1000**: the column is `1` on all 64 rows of
`nodes.csv` (see [`catalogs/rooms/nodes.csv`](../../catalogs/rooms/nodes.csv)).

> Source: `LibNode.sol:25–29, 119–120, 135–152`,
> `_NodeRegistrySystem.sol:21–45`, `HarvestMultiplierComponent.sol`,
> `deployment/world/state/rooms/nodes.ts:11–30, 46–51, 100–101, 195–212`,
> `deployment/world/state/utils.ts:16–25` (`toPerMille`),
> `deployment/world/api/nodes.ts:11–29, 101–106`,
> `deployment/world/world.ts:183–185`

## Node Bonuses

Nodes can grant **temporary bonuses** to Kamis that harvest there. These
bonuses:
- Are registered with end type `UPON_HARVEST_STOP` — hardcoded, not a parameter
  (`LibNode.addBonus` passes the literal string, `LibNode.sol:91`)
- Are assigned when a Kami starts harvesting on the node
  (`LibNode.assignBonuses` → `LibBonus.assignTemporary`, `LibNode.sol:166–168`,
  called from `HarvestStartSystem.sol:46–47`)
- Are cleared when the Kami stops harvesting (voluntarily or via liquidation)
- Do **not** stack — they are temporary bonuses, so repeat assignment is
  idempotent at level 1 (see
  [bonus-system.md → Stacking](../combat/bonus-system.md#stacking-rules))

Bonuses are anchored to: `keccak256("node.bonus", nodeIndex)`
(`LibNode.genBonusAnchor`, `LibNode.sol:249–251`). Each is a bonus **registry**
entry keyed `keccak256("bonus.registry", nodeBonusAnchor, bonusType)`, so a
node carries at most one entry per bonus type, and any bonus type the bonus
system understands is valid — `addBonus` takes a free-form `bonusType` string
and never validates it (`LibNode.sol:79–95`).

Deleting a node removes its whole bonus set via `regRemoveByAnchor`
(`LibNode.sol:122`).

> Source: `LibNode.sol:79–95, 122, 166–168, 249–251`,
> `HarvestStartSystem.sol:46–47`, `LibBonus.sol:66–92, 177–192`

### Which nodes grant what — runtime world state

> ⚠️ **Not derivable from source at the pin.** No per-node bonus catalog exists
> in this GDD because the source repo contains **no node-bonus data**:
>
> - `deployment/world/data/rooms/nodes.csv` has a trailing
>   **`💥 Node Effects`** column, but it is **empty on all 64 rows**, and the
>   node initialiser never reads it — `initNode` consumes only `Index`,
>   `YieldIndex`, `Name`, `Affinity` and `Harvest Multiplier`, and separately
>   `Level Limit` and `Scav Cost` (`deployment/world/state/rooms/nodes.ts:11–25, 55–58`).
> - The admin helper `api.node.bonus.add` exists
>   (`deployment/world/api/nodes.ts:37–44, 111–113`) but has **zero callers** anywhere
>   in the repo at the pin.
>
> The only writer is **`_NodeRegistrySystem.addBonus`**
> (`_NodeRegistrySystem.sol:55–61`), an `onlyAdmin`-gated entrypoint that takes
> `(uint32 nodeIndex, string bonusType, int256 value)` and forwards to
> `LibNode.addBonus`. Its calldata is generated by `api.node.bonus.add`, which
> the deployment scripts do not invoke, so live node bonuses are **runtime
> world state written by an admin call made outside the deployment pipeline**.
> Reading them requires querying the chain for bonus registry entries anchored
> at `keccak256("node.bonus", nodeIndex)` — they cannot be recovered from this
> repo or from the source repo at the pin.
>
> This applies uniformly, including to the two castle-plane nodes 88
> (Treasure Hoard) and 89 (Trophies of the Hunt), whose `💥 Node Effects`
> cells are empty like every other row.

> Source: `deployment/world/data/rooms/nodes.csv` (header + all 64 rows),
> `deployment/world/state/rooms/nodes.ts:11–25, 53–62`,
> `deployment/world/api/nodes.ts:37–44, 108–124`,
> `_NodeRegistrySystem.sol:55–61`, `LibNode.sol:79–95`

## Node Requirements

Nodes can have **conditional requirements** that a Kami (or its account) must
meet to start harvesting. Checked via `LibConditional.check()` when a harvest
**starts** (`HarvestStartSystem.sol:44`) — a Kami that fails reverts
`"node reqs not met"`; nothing re-checks a harvest already running.

Requirements are anchored to: `keccak256(abi.encodePacked("node.requirement",
nodeIndex))` (`LibNode.genReqAnchor`). They are written only by the
admin-gated `_NodeRegistrySystem.addRequirement`; the only removal path is
deleting the whole node (`LibNode.remove` drops every requirement under the
anchor).

### Level caps from node data

The only requirement the deployment scripts write is a **level cap**: a node's
`Level Limit` cell becomes `LEVEL` / `CURR_MAX` / value = the limit / for
`KAMI` (`addRequirement`, "hardcoded to only allow max levels"). At the pin
13 nodes carry one:

| Max level | Nodes |
|---|---|
| 15 | 1 Misty Riverside, 2 Tunnel of Trees, 3 Torii Gate, 29 Misty Forest Path, 30 Scrapyard Entrance, 55 Shady Path |
| 35 | 31 Scrapyard Exit, 35 Elder Path, 36 Parting Path, 47 Scrap Paths, 50 Ancient Forest Entrance, 57 River Crossing, 59 Black Pool |

The other 51 nodes have no cap. Node 56 Butterfly Forest's cell went from 35
to empty in this sync's range (the room hosts quests SQ119–SQ122 and SQ902).
Emptying a cell removes nothing on chain by itself; only a re-create of the
node (`reviseNodes`: remove, then create) drops the old cap, and the data
marks node 56 for a revise (its row went `To Update`, then back to
`In Game`, within the range). See
[`catalogs/rooms/nodes.csv`](../../catalogs/rooms/nodes.csv).

Caps reach the chain two ways:

- **At node creation** — `initNodes` adds the cap right after `initNode`
  whenever `Level Limit` is not empty.
- **On a live node** — `addNodeRequirements(indices)` (world-state command
  `nodes.addRequirements`) adds the CSV's cap **without re-creating the node**,
  so harvests in progress are untouched (no delete/re-init as in
  `reviseNodes`). For each given index it skips nodes absent from the CSV or
  with an empty `Level Limit`, computes the anchor
  `solidityPackedKeccak256(['string','uint32'], ['node.requirement', index])`
  — mirroring `LibNode.genReqAnchor` — and **skips the node if any entity
  already carries that anchor on chain** (`getEntitiesWithValue` on the anchor
  component). It is **add-only**: there is no admin call that removes a single
  requirement, so an existing cap is never replaced or raised by this helper.

> ⚠️ UNCERTAIN: the helper only adds; whether a given live node already
> carried a cap before it was run is chain state, not source. In particular
> the seven level-35 caps are in the CSV at the pin, but which of those nodes
> carry the cap on chain depends on whether the helper (or a node re-create)
> was run for them.

> Source: `LibNode.sol:98–106, 108–129, 173–187, 238–240, 253–255`,
> `_NodeRegistrySystem.sol:66–87`, `HarvestStartSystem.sol:44`,
> `deployment/world/state/rooms/nodes.ts:36–63, 153–190, 214–225`,
> `deployment/world/world.ts:181–182`

## Scavenging Integration

Each node can have a **scavenge bar** (see [scavenging.md](scavenging.md)).
When a Kami collects or stops harvesting, the node's `scavenge()` function is
called with the harvest output amount, incrementing the scavenge bar progress.

```
LibNode.scavenge(components, nodeIndex, harvestAmount, accountID)
```

If the node has no scavenge bar configured, this is a no-op.

> Source: `LibNode.sol:156–164, 225–227`

## Node-Room Relationship

Each node belongs to exactly one room. The node's `IndexRoom` typically matches
its index (the code notes: "nodeIndex = roomIndex" as a TODO to simplify). In
practice, node indices correspond to room indices for harvest nodes.

> Source: `LibNode.sol:50` (todo comment), `nodes.csv`

## World Data

The current world has **64 in-game harvest nodes** with the following affinity
distribution:

| Affinity | Count | Examples |
|---|---|---|
| Normal | 19 | Tunnel of Trees, Torii Gate, Forest paths |
| Insect | 13 | Forest: Insect Node, Cave Crossroads, Centipedes |
| Eerie | 13 | Misty Riverside, Labs Entrance, Blooming Tree |
| Scrap | 12 | Scrap Confluence, Scrapyard Entrance, Deeper Into Scrap |
| Compound | 7 | Forest: Old Growth 9 (Insect-Eerie), Techno Temple 16 (Eerie-Scrap), Temple of the Wheel 19 (Eerie-Scrap), Trash-Strewn Graves 26 (Eerie-Scrap), Scrap-Littered Undergrowth 51 (Scrap-Insect), Hatch to Nowhere 72 (Insect-Scrap), Guardian Skull 86 (Eerie-Insect) |

Yield: 49 nodes yield MUSU (item 1) and 15 yield VIPP (item 2) — nodes 6, 18,
48, 49, 52, 58, 60, 61, 62, 65, 73, 75, 79, 83 and 88. The same item pays out
the harvest and the liquidation salvage on that node.

Scavenge costs range from 100 to 1,000 points per tier: 100–500 on the
surface, 200–1,000 on the cave and castle planes. Higher costs sit on nodes with rarer
droptables — the six 1,000-point nodes are the essence nodes. Per-tier counts
are in [`catalogs/rooms/README.md`](../../catalogs/rooms/README.md#scavenge-cost-tiers).

> Source: `data/rooms/nodes.csv` (all 64 rows)
