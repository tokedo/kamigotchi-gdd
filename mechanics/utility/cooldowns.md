# Cooldown System

> Source: `packages/contracts/src/libraries/utils/LibCooldown.sol` (L1–109)

## Overview

The cooldown system manages **time-locked states** for entities (primarily
Kamis). When a cooldown is set, the entity cannot perform certain actions until
the cooldown expires. Cooldowns are stored as a future timestamp in the
`TimeNextComponent`.

## Setting a Cooldown

`LibCooldown.set(components, entityID)`:

```
baseCooldown = KAMI_STANDARD_COOLDOWN config
bonusShift = LibBonus.getFor("STND_COOLDOWN_SHIFT", entityID)
cooldown = max(0, baseCooldown + bonusShift)
endTime = block.timestamp + cooldown
```

The cooldown duration is the **base value plus any bonus shift**. Bonuses can
reduce the cooldown (negative shift) but never below 0.

> Source: `LibCooldown.sol:25–31, 99–108`

## Modifying a Cooldown

`LibCooldown.modify(components, entityID, delta)`:

- If cooldown is still active (`endTime > now`): new end = `endTime + delta`
- If cooldown has expired: new end = `block.timestamp + delta`

This allows extending or shortening an active cooldown.

> Source: `LibCooldown.sol:34–53`

## Checking Status

```solidity
LibCooldown.isActive(components, entityID) → bool
// true if block.timestamp < endTime

LibCooldown.getEnd(components, entityID) → uint256
// returns the end timestamp (0 if never set)
```

The batch version returns `true` if **any** entity in the array is on cooldown.

> Source: `LibCooldown.sol:58–79`

## Config

| Key | Description |
|---|---|
| `KAMI_STANDARD_COOLDOWN` | Base cooldown duration in seconds |

## Bonus Integration

The `STND_COOLDOWN_SHIFT` bonus type modifies cooldown duration. A positive
shift increases cooldown; a negative shift decreases it. This is how skills and
equipment can reduce action cooldowns.

> Source: `LibCooldown.sol:105`
