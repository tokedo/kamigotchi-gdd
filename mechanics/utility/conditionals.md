# Conditional & Requirement System

> Source: `packages/contracts/src/libraries/LibConditional.sol` (L1–293)

## Overview

The conditional system provides a **generic boolean check framework** used by
quests, auctions, skills, and other systems that gate actions behind
requirements. Each condition is an entity with a type, logic operator, and
comparison value.

## Condition Entity Shape

| Component | Description |
|---|---|
| `Type` | What to check (e.g., `"ITEM"`, `"LEVEL"`, `"FLAG"`, `"QUEST"`) |
| `LogicType` | How to check, format: `"{HANDLER}_{OPERATOR}"` |
| `Index` | Optional item/entity index |
| `Value` | Optional comparison value |
| `IDAnchor` | Parent entity this condition belongs to |
| `For` | Optional target shape override (e.g., `"ACCOUNT"`, `"ROOM"`, `"KAMI"`, `"GLOBAL"`) |

> Source: `LibConditional.sol:41–47, 70–79`

## Logic Format

Logic strings follow the pattern `"{HANDLER}_{OPERATOR}"`:

### Handlers

| Handler | Description |
|---|---|
| `CURRENT` (`CURR_*`) | Check the current value of a property |
| `INCREASE` (`INC_*`) | Check increase from snapshot (not yet implemented for conditionals) |
| `DECREASE` (`DEC_*`) | Check decrease from snapshot (not yet implemented for conditionals) |
| `BOOLEAN` (`BOOL_*`) | Check a boolean flag |

### Operators

| Operator | Comparison | Description |
|---|---|---|
| `MIN` | `value >= threshold` | Minimum required value |
| `MAX` | `value <= threshold` | Maximum allowed value |
| `EQUAL` | `value == threshold` | Exact match |
| `IS` | Boolean true | Flag must be set |
| `NOT` | Boolean false | Flag must not be set |

**Examples:**
- `CURR_MIN` — current value must be at least threshold
- `BOOL_IS` — boolean flag must be true
- `BOOL_NOT` — boolean flag must be false

> Source: `LibConditional.sol:261–285`

## Target Shape Override

The `For` component allows redirecting the check to a related entity:

| For Value | Behavior |
|---|---|
| `""` (empty) | Check the original target (no change) |
| `"ACCOUNT"` | Resolve to the target's account owner |
| `"ROOM"` | Resolve to the target's current room |
| `"KAMI"` | Verify target is a Kami (identity check) |
| `"GLOBAL"` | Check against entity ID 0 (global state) |

This enables conditions like "the Kami's owner must have item X" (For=ACCOUNT)
or "the Kami must be in room Y" (For=ROOM).

> Source: `LibConditional.sol:145–163`

## Check Flow

`LibConditional.check(components, conditionIDs, targetID)`:

1. If no conditions, return true
2. For each condition:
   a. Resolve target shape override (if `For` is set)
   b. Parse logic string into handler + operator
   c. Dispatch to handler:
      - `CURRENT`: look up current balance/value via `LibGetter.getBal`, compare
      - `BOOLEAN`: look up flag via `LibGetter.getBool`, compare
   d. If any condition fails, return false
3. Return true (all conditions passed)

> Source: `LibConditional.sol:116–141`

## Usage

Conditions are attached to parent entities via `IDAnchor`:

| System | Usage |
|---|---|
| Quest requirements | Gating quest acceptance |
| Auction requirements | Gating auction purchases |
| Skill requirements | Gating skill upgrades |
| NPC shop listings | Gating item purchases |

Relationship advancement does **not** use LibConditional — it validates
against its own whitelist/blacklist of prior relationship indices
(`LibRelationship.sol:79–90`).

Query conditions for a parent: `LibConditional.queryFor(components, parentID)`
