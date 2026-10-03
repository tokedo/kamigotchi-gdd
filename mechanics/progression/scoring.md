# Leaderboard & Scoring

> Source: `packages/contracts/src/libraries/LibScore.sol` (L1–181)

## Overview

The scoring system provides **epoch-based point tracking** with both individual
and aggregate totals. It is a generic building block used by multiple systems:
faction reputation, community goals, the harvest (`COLLECT`) and liquidation
(`LIQUIDATE`) leaderboards, NPC-shop spend, and VIP (see
[Known Score Types](#known-score-types)).

Scores are designed for **reverse-mapping** — the `IdHolder` and `IDType`
components enable front-end leaderboard queries without full table scans.

## Score Entity Shape

| Component | Description |
|---|---|
| `IdHolder` | Entity that owns this score (e.g., account ID) |
| `IDType` | Type discriminator (derived from epoch + index + type string) |
| `Value` | Current score value |

Score ID depends on context — different callers generate IDs differently.

### Epoch-Based Score ID

For standard leaderboard use (per-epoch):

```
scoreID = keccak256("is.score", holderID, epoch, index, type)
typeID  = keccak256("score.type", epoch, index, type)
```

### Context-Specific Score IDs

Other systems generate their own IDs but use `LibScore.incFor/decFor`:

- **Faction reputation**: `keccak256("faction.reputation", holderID, factionIndex)`
- **Goal contributions**: `keccak256("goal.contribution", goalID, accID)`

> Source: `LibScore.sol:42–53, 160–175`

## Total Tracking

Every `incFor` / `decFor` call updates **two** values:

1. **Individual score**: the specific holder's score (`Value` on scoreID)
2. **Global total**: the sum of all holders' scores for that type

```
Total store ID = keccak256("score.total", typeID)
```

This enables percentage calculations (e.g., "this player contributed X% of
the total") without iterating over all scores.

> Source: `LibScore.sol:56–67, 96–108`

## Epoch System

Scores use an **epoch** to segment time periods. The current epoch is stored
as a config value:

```
currentEpoch = LibConfig.get("SCORE_EPOCH")
```

Epochs are manually set by the contract owner (not automatically advanced).
Changing the epoch effectively starts a new leaderboard period — all new score
increments go to the new epoch, while old epoch scores are preserved.

### Auto-Epoch Functions

When called with a `string` type (instead of pre-computed IDs), the score
library automatically reads the current epoch:

```solidity
// e.g. ListingBuySystem.sol:50 — index is the currency item index
LibScore.incFor(components, accID, currencyIndex, "TOTAL_SPENT", spent)
// internally: epoch = getCurrentEpoch(), then generates IDs with epoch
```

### Manual-Epoch Functions

Systems that don't use epochs (e.g., factions, goals) call the raw
`incFor(components, id, holderID, typeID, amt)` directly with pre-computed IDs.

> Source: `LibScore.sol:69–94, 110–135, 144–155`

## Known Score Types

Every `LibScore` writer in the contracts:

| Score Type | Used By | Description |
|---|---|---|
| `COLLECT` (index `MUSU_INDEX`) | Harvest collect and stop (`HarvestCollectSystem`, `HarvestStopSystem`) | Epoch-scoped post-tax MUSU harvested, +output per collect/stop — see [harvesting.md](../economy/harvesting.md#side-effects-on-collection) |
| `LIQUIDATE` (index `0`) | Harvest liquidation (`HarvestLiquidateSystem`) | Epoch-scoped liquidation count for the killer's account, +1 per liquidation |
| `TOTAL_SPENT` (index = currency item) | NPC shops (`ListingBuySystem`) | Epoch-scoped spend at NPC shop listings, indexed per currency item |
| `VIP_SCORE` (index `0`) | VIP (`LibVIP.inc`) | Keyed by the current VIP **stage** in the epoch slot rather than `SCORE_EPOCH` |
| Faction reputation | Faction system (`LibFaction`) | Per-faction reputation points (pre-computed IDs; increment and decrement) |
| Goal contributions | Goal system (`LibGoal`) | Per-goal contribution amounts (pre-computed IDs) |

The only `TOTAL_SPENT` writer is `ListingBuySystem.sol:50` — `LibTrade` makes
no `LibScore` calls (P2P trades are not scored).

> Source: `HarvestCollectSystem.sol:100`, `HarvestStopSystem.sol:110`,
> `HarvestLiquidateSystem.sol:90`, `ListingBuySystem.sol:50`,
> `LibVIP.sol:36`, `LibFaction.sol:79, 85`, `LibGoal.sol:277`

## Operations

### Increment

```solidity
LibScore.incFor(components, holderID, index, type, amt)
```

Creates the score entity if it doesn't exist, then increments both individual
and total values.

### Decrement

```solidity
LibScore.decFor(components, holderID, index, type, amt)
```

Creates the score entity if it doesn't exist, then decrements both individual
and total values. Will revert on underflow.

### Read

```solidity
LibScore.get(components, scoreID) → uint256
```

Returns 0 if the score doesn't exist (safe get).

> Source: `LibScore.sol:56–135, 140–142`

## Config

| Key | Value | Description |
|---|---|---|
| `SCORE_EPOCH` | `1` | Current scoring epoch number |

> Source: `configs.ts:67`
