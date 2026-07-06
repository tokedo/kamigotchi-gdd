# Day/Night Cycle (Phases)

> Source: `packages/contracts/src/libraries/utils/LibPhase.sol` (L1–37)

## Overview

Kamigotchi has a **36-hour day cycle** divided into three 12-hour phases. The
phase is derived purely from the block timestamp — there is no state to manage.

## Phases

| Phase | Name | Hours in Cycle |
|---|---|---|
| 1 | DAYLIGHT | 0–11 |
| 2 | EVENFALL | 12–23 |
| 3 | MOONSIDE | 24–35 |

## Calculation

```
hour = (block.timestamp / 3600) % 36
phase = (hour / 12) + 1
```

The cycle repeats every 36 hours (129,600 seconds), offset from Unix epoch.

> Source: `LibPhase.sol:26–29`

## Usage

The phase system is used in two ways:

- **`PHASE` condition type** — any conditional (room gates, quest
  requirements/objectives, node requirements) can check
  `LibPhase.get(block.timestamp) == index`, gating actions to a specific
  phase (`libraries/utils/LibGetter.sol:90–91`).
- **Per-phase data logging** — liquidations and harvests are logged under
  phase-suffixed keys that quest objectives can target:
  - `LIQ_WHEN_{PHASE}` — liquidations per phase (`LibKill.sol:296`)
  - `HARVEST_TIME_{PHASE}` — harvest time per phase (`LibHarvest.sol:380–381`)
  - `HARVEST_WHEN_{PHASE}` — harvest amounts per phase (`LibHarvest.sol:406`)
