# Community Goals

> Source: `packages/contracts/src/libraries/LibGoal.sol` (L1–404),
> `packages/contracts/src/systems/GoalContributeSystem.sol` (L1–50),
> `packages/contracts/src/systems/GoalClaimSystem.sol` (L1–36),
> `packages/contracts/src/systems/_GoalRegistrySystem.sol` (L1–175)

## Overview

Community goals are **multi-player contribution targets** where players pool
resources toward a shared objective. When the goal is completed, each
contributor can claim tiered and/or proportional rewards based on their
individual contribution amount.

Goals are one-time events — once completed, no further contributions are
accepted. Contributions are tracked via `LibScore` for leaderboard
compatibility.

> ⚠️ UNCERTAIN: the goal catalog is not fully derivable from source. The
> deployment seed (`deployment/world/state/goals.ts`) actively creates only
> goals 7 ("Titanic Offering") and 13 ("Secret of the Ooze"); definitions for
> goals 1–6 and 8–11 exist only as commented-out code. Other live goals were
> seeded at runtime by admin calls, so the full set of deployed goal
> instances cannot be reconstructed from this repo.

## Goal Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"GOAL"` |
| `Index` | Unique goal index (uint32) |
| `Name` | Display name |
| `Description` | Goal description |
| `IndexRoom` | (optional) Room the player must be in to contribute/claim |
| `Disabled` | Initially `true` — must be enabled |
| `IsComplete` | Set when goal target is reached |
| `Value` | Current accumulated balance (total contributions so far) |

Goal ID: `keccak256("goal", goalIndex)`

> Source: `LibGoal.sol:68–87`

## Objective

Each goal has exactly one objective, stored as a `LibConditional` condition.
The objective defines:

- **Type**: The resource being contributed (e.g., `ITEM`)
- **Index**: Specific item/resource index
- **Value**: Target amount to complete the goal

Objective ID: `keccak256("goal.objective", goalID)`

The objective's value is the total target. The goal's `Value` component tracks
current progress toward that target.

> Source: `LibGoal.sol:84–86, 157–160`

## Contributing

`GoalContributeSystem.execute(goalIndex, amount)`:

1. Verify goal exists and is enabled
2. Verify contributability:
   - Player meets all requirements (`LibConditional`)
   - Player is in the correct room (if room-gated), or goal is global (no room)
   - Goal is not yet complete
3. Cap contribution if it would exceed the target:
   ```
   if (currentBalance + amount >= targetBalance):
       amount = targetBalance - currentBalance
       mark goal as complete
   ```
4. Deduct the contributed resource from the player's account via `LibSetter.dec`
5. Increment the goal's balance
6. Increment the player's contribution via `LibScore.incFor`
7. If goal just completed, emit `GOAL_COMPLETE` event
8. Log `GOAL_CONTRIBUTION` for the account

> Source: `GoalContributeSystem.sol:17–38`, `LibGoal.sol:151–179`

## Contribution Tracking

Individual contributions are tracked as score entities:

Contribution ID: `keccak256("goal.contribution", goalID, accID)`

The score's `Value` component stores the total amount contributed by this
account. The score's `IDType` points to the goal ID, enabling reverse-mapping
for leaderboards.

Claiming sets `IsComplete` on the contribution entity to prevent double-claims.

> Source: `LibGoal.sol:270–278, 284–286, 296–303`

## Claiming Rewards

`GoalClaimSystem.execute(goalIndex)`:

1. Verify goal exists and is enabled
2. Verify claimability:
   - Goal must be complete (`IsComplete` on goal)
   - Account must have contributed (`Value` exists on contribution entity)
   - Account must not have already claimed (`IsComplete` not set on contribution)
   - Account must be in the correct room (if room-gated)
3. Distribute rewards:
   - **Tiered rewards**: based on contribution amount vs tier cutoffs
   - **Proportional rewards**: multiplied by contribution amount
4. Mark contribution as claimed (`IsComplete` on contribution entity)

> Source: `GoalClaimSystem.sol:15–31`, `LibGoal.sol:181–198, 220–231`

## Reward Tiers

Goals can have multiple reward tiers (e.g., Bronze, Silver, Gold), each with
a **cutoff** — the minimum contribution amount to qualify:

| Cutoff | Behavior |
|---|---|
| `> 0` | Standard tier — player qualifies if `contribution >= cutoff` |
| `= 0` | Special: **proportional** rewards or **display-only** |

**Tier stacking**: Higher tiers receive all lower tier rewards too. A Gold
contributor gets Gold + Silver + Bronze rewards.

Tier entity: created via `LibReference.create("goal.tier", cutoff, tierAnchorID)`
Tier anchor: `keccak256("goal.tier", goalIndex)`

> Source: `LibGoal.sol:89–100, 337–352`

### Proportional Rewards

Rewards anchored to the cutoff=0 tier are distributed proportionally:

```
reward_amount = base_reward × contribution_amount
```

This means larger contributors receive proportionally more. The distribution
uses `LibAllo.distribute(world, comps, rwdIDs, contributionAmt, accID)` with
the contribution amount as a multiplier.

> Source: `LibGoal.sol:196–197, 328–333`

### Display-Only Rewards

Tiers with cutoff=0 can also contain display allocations, created with type
`"DISPLAY_ONLY_{name}"` (`_GoalRegistrySystem.sol:154`) and no `Value`
component (`LibAllo.createEmpty`). They exist solely for client UI display —
but they are **not** caught by the distribution skip guard:
`skipDistribution` matches only the exact string `"DISPLAY_ONLY"`
(`LibAllo.sol:285–287`), which `"DISPLAY_ONLY_{name}"` never equals. The allo
instead falls through to `giveBasic` → `LibSetter.update` with amount 0 (the
missing `Value` reads as 0), landing in the `LibData.inc(..., 0)` fallback —
a no-op. Net effect is unchanged (nothing is distributed), but via a
zero-amount fall-through rather than an explicit skip.

> Source: `_GoalRegistrySystem.sol:142–157`, `LibAllo.sol:118–126, 202–213,
> 229–239, 285–287`, `LibSetter.sol:64–65`

## Room Gating

Goals can optionally require the player to be in a specific room to contribute
or claim. If no room is set (`IndexRoom` not present), the goal is global.

```
checkRoom: if goal has no room → always pass
            else → account's current room must match goal's room
```

> Source: `LibGoal.sol:256–261`

## Requirements

Goals can have additional acceptance requirements (e.g., quest completion,
item ownership) checked via `LibConditional`.

Requirement anchor: `keccak256("goal.requirement", goalIndex)`

> Source: `LibGoal.sol:103–110, 310–315`

## Reward Types

Goal rewards support all standard `LibAllo` types:

| Type | Description |
|---|---|
| Basic (ITEM, etc.) | Items, MUSU, XP via `LibSetter` |
| `ITEM_DROPTABLE` | Random loot via commit-reveal |
| `STAT` | Direct stat modifications |
| `DISPLAY_ONLY_{name}` | Client-only display; distributes nothing (zero-amount fall-through) |

Reward anchor: `keccak256("goal.reward", tierID)`

> Source: `_GoalRegistrySystem.sol:74–157`

## Logging

| Data Key | Description |
|---|---|
| `GOAL_CONTRIBUTION` | Total contribution amount across all goals |

> Source: `LibGoal.sol:358–360`
