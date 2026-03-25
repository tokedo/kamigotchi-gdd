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

The phase system can be used by other mechanics to gate actions or modify
behavior based on time of day (e.g., different scavenging yields, NPC
availability, or enemy behavior during MOONSIDE).
