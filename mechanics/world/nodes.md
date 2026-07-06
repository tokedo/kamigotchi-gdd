# Nodes (Sub-Locations)

> Source: `packages/contracts/src/libraries/LibNode.sol` (L1–225),
> `packages/contracts/deployment/world/data/rooms/nodes.csv`

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

Entity ID: `keccak256("node", nodeIndex)`

> Source: `LibNode.sol:24–69, 214–216`

## Node Types

The code supports multiple node types:

| Type | Description |
|---|---|
| `HARVEST` | Primary type — Kamis farm resources here |
| `HEALING` | Referenced in code comments but not actively used |
| `SACRIFICIAL` | Referenced in code comments but not actively used |
| `TRAINING` | Referenced in code comments but not actively used |

Currently all deployed nodes are `HARVEST` type.

> Source: `LibNode.sol:53, 158–160`

## Affinity System

Node affinities interact with Kami affinities to modify harvest effectiveness
and combat outcomes (see [harvesting.md](../economy/harvesting.md) for the
full efficacy formula).

**Affinity validation rule**: A node cannot combine `NORMAL` with a typed
affinity (e.g., `"NORMAL-EERIE"` is invalid). Compound affinities must be
two typed affinities (e.g., `"EERIE-SCRAP"` is valid). The on-chain format
is hyphen-separated — `isValidAffinity` splits on `"-"` (`LibNode.sol:164`),
and deployment converts the CSV's comma format via `.replace(',', '-')`
(`deployment/world/state/rooms/nodes.ts:11`).

> Source: `LibNode.sol:163–167`, `nodes.ts:11`

## Node Bonuses

Nodes can grant **temporary bonuses** to Kamis that harvest there. These
bonuses:
- Are registered with end type `UPON_HARVEST_STOP`
- Are assigned when a Kami starts harvesting on the node
- Are cleared when the Kami stops harvesting (voluntarily or via liquidation)

Bonuses are anchored to: `keccak256("node.bonus", nodeIndex)`

> Source: `LibNode.sol:72–88, 135–137`

## Node Requirements

Nodes can have **conditional requirements** that a Kami (or its account) must
meet to start harvesting. Checked via `LibConditional.check()`.

Requirements are anchored to: `keccak256("node.requirement", nodeIndex)`

Common requirement types from node data:
- Level limits (e.g., max level 15 for some starter nodes)

> Source: `LibNode.sol:91–99, 142–156`

## Scavenging Integration

Each node can have a **scavenge bar** (see [scavenging.md](scavenging.md)).
When a Kami collects or stops harvesting, the node's `scavenge()` function is
called with the harvest output amount, incrementing the scavenge bar progress.

```
LibNode.scavenge(components, nodeIndex, harvestAmount, accountID)
```

If the node has no scavenge bar configured, this is a no-op.

> Source: `LibNode.sol:125–133, 194–196`

## Node-Room Relationship

Each node belongs to exactly one room. The node's `IndexRoom` typically matches
its index (the code notes: "nodeIndex = roomIndex" as a TODO to simplify). In
practice, node indices correspond to room indices for harvest nodes.

> Source: `LibNode.sol:43` (todo comment), `nodes.csv`

## World Data

The current world has **64 in-game harvest nodes** with the following affinity
distribution:

| Affinity | Count | Examples |
|---|---|---|
| Normal | 19 | Tunnel of Trees, Torii Gate, Forest paths |
| Insect | 15 | Forest: Insect Node, Cave Crossroads, Centipedes |
| Eerie | 14 | Misty Riverside, Labs Entrance, Blooming Tree |
| Scrap | 12 | Scrap Confluence, Scrapyard Entrance, Deeper Into Scrap |
| Compound | 4 | Techno Temple (Eerie-Scrap), Temple of the Wheel (Eerie-Scrap), Hatch to Nowhere (Insect-Scrap), Guardian Skull (Eerie-Insect) |

Scavenge costs range from 100 to 500, with higher costs on more rewarding nodes.

> Source: `data/rooms/nodes.csv`
