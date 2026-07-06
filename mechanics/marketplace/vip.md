# VIP System (Initia Integration)

> Source: `packages/contracts/src/libraries/LibVIP.sol` (L1–71),
> `packages/contracts/src/utils/VipScore.sol` (L1–212),
> `packages/contracts/src/components/ProxyVIPScoreComponent.sol` (L1–25)

## Overview

The VIP system integrates with **Initia VIP** to report player engagement scores
on-chain. Scores are tracked in **stages** (2-week epochs) and written to both
the in-game scoring system (`LibScore`) and an external `VipScore` contract.

## Architecture

The system has three layers:

1. **LibVIP** — library called by game systems to increment VIP scores
2. **ProxyVIPScoreComponent** — ECS component proxy that relays calls to the
   external contract (with writer permission enforcement)
3. **VipScore** — standalone contract that stores per-stage, per-address scores
   with allow-list access control

```
Game System ──→ LibVIP.inc() ──→ LibScore (in-game) + ProxyVIPScoreComponent ──→ VipScore contract
```

> Source: `LibVIP.sol:30–47`

## Stages

Stages are 2-week epochs calculated from a genesis timestamp:

```
stage = ((block.timestamp - start) / epochLength) + 1
```

Configuration: `VIP_STAGE` config array = `[start, epochLength]`

Stages are 1-indexed (stage 0 is invalid). Each stage tracks:

| Field | Type | Description |
|---|---|---|
| `stage` | uint64 | Stage number (0 = uninitialized) |
| `totalScore` | uint64 | Sum of all player scores in this stage |
| `isFinalized` | bool | Whether the stage is closed for further scoring |

> Source: `LibVIP.sol:61–66`, `VipScore.sol:18–22`

## Score Increment

`LibVIP.inc(components, accID, amount)`:

1. Look up `VIP_SCORE_ADDRESS` config for the VipScore contract address
2. Calculate current stage from `VIP_STAGE` config
3. Increment in-game score: `LibScore.incFor(accID, stage, 0, "VIP_SCORE", amount)`
4. **Auto-finalize** the previous stage if not already finalized
5. Increment external score via proxy: `ProxyVIPScoreComponent.inc(vipAddr, stage, ownerAddr, amount)`

> Source: `LibVIP.sol:31–47`

## Stage Finalization

When a score is incremented for stage N, the system automatically checks
whether stage N-1 needs finalization:

```solidity
finalizePrevStage(vipComp, vipAddr, currStage):
  if currStage <= 1: return  // nothing to finalize
  if stages[currStage - 1].isFinalized: return  // already done
  vipComp.finalizeStage(vipAddr, currStage - 1)
```

Finalizing a stage:
1. Marks the stage as `isFinalized = true`
2. Emits `FinalizeStage(stage)` event
3. Creates the next stage (stage + 1) if it doesn't exist

> Source: `LibVIP.sol:49–56`, `VipScore.sol:55–66`

## VipScore Contract

The `VipScore` contract is a standalone, non-ECS contract that provides:

### Score Operations

| Function | Description |
|---|---|
| `increaseScore(stage, addr, amount)` | Add to a player's score (reverts if finalized) |
| `decreaseScore(stage, addr, amount)` | Subtract from a player's score (reverts if finalized) |
| `updateScore(stage, addr, amount)` | Set exact score (adjusts totalScore by diff) |
| `updateScores(stage, addrs[], amounts[])` | Batch update scores |
| `finalizeStage(stage)` | Close stage, auto-create next |

All score operations:
- Silently return for `address(0x0)`
- Revert if stage doesn't exist (`StageNotFound`)
- Revert if stage is finalized (`StageFinalized`)
- Maintain an iterable index of scored addresses per stage

> Source: `VipScore.sol:68–166`

### Score Queries

```solidity
getScores(stage, offset, limit) → ScoreResponse[]
```

Returns paginated scores for a stage. Each response includes:
- `addr`: player address
- `amount`: player's score
- `index`: position in the iterable mapping

> Source: `VipScore.sol:185–201`

### Access Control

The VipScore contract uses an **allow-list** (not role-based):

| Function | Description |
|---|---|
| `addAllowList(addr)` | Grant write access |
| `removeAllowList(addr)` | Revoke write access |

The deployer is automatically added to the allow list. The
`ProxyVIPScoreComponent` is granted access so the game world can write scores.

> Source: `VipScore.sol:40–41, 135–141`

## Per-Player Score Storage

Scores are stored in a nested mapping:
```
scores[stage][address] → { isIndexed: bool, amount: uint64 }
```

An iterable index is maintained separately:
```
scoreKeys[stage][index] → address
scoreLength[stage] → count
```

This allows both O(1) lookup by address and paginated enumeration.

> Source: `VipScore.sol:37–39`

## Config

| Key | Value | Description |
|---|---|---|
| `VIP_STAGE` | `[1745481600, 1209600, 0, 0, 0, 0, 0, 0]` | Genesis: 2025-04-24 08:00 UTC, epoch: 1,209,600s (2 weeks) |
| `VIP_SCORE_ADDRESS` | (deployment address) | Address of the deployed VipScore contract |

> Source: `configs.ts:189–193`
