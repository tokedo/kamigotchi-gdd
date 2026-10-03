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

## What a Cooldown Blocks

A Kami's cooldown is read by exactly these entrypoints, all of which revert
`"kami on cooldown"` while it is active (`LibKami.verifyCooldown`):

| Entrypoint | Whose cooldown | Source |
|---|---|---|
| Harvest start | the Kami starting | `HarvestStartSystem.sol:36` |
| Harvest collect | the harvesting Kami | `HarvestCollectSystem.sol:32` |
| Harvest stop | the harvesting Kami | `HarvestStopSystem.sol:34` |
| Liquidation | the **killer** (not the victim) | `HarvestLiquidateSystem.sol:32` |
| Item use on a Kami (feeding, potions) | the target Kami | `KamiUseItemSystem.sol:28` |

The batched `executeAllowFailure` variants of collect and stop skip a Kami on
cooldown (return `0`) instead of reverting. Conditions can also test it: the
`COOLDOWN` condition type is true while the target's cooldown is active
(`LibGetter.sol:92–93`).

Nothing else reads the cooldown — sending a Kami, marketplace listing and
buying, equip/unequip, gacha reroll and bridging out do not check it.

> Source: `LibKami.sol:236–238, 251–257`; callers as listed

## Transfer / Purchase Cooldown

A Kami that changes hands gets a cooldown on arrival:

| Path | Applied by |
|---|---|
| In-game send (`KamiSendSystem`) | `KamiSendSystem.sol:47–50, 70` |
| Marketplace sale (listing bought, offer accepted, collection offer filled) | `LibKamiMarket._setPurchaseCooldown`, `LibKamiMarket.sol:139, 168, 197, 272–277` |

```
cd = KAMI_MARKET_PURCHASE_COOLDOWN   if the config is set
   = 3600 s (1 hour)                 if it is not set
if cd > 0: LibCooldown.modify(kami, +cd)
```

`modify` **adds** to a still-running cooldown (new end = old end + `cd`) and
otherwise starts one at `now + cd`. No init script sets the key; it is
admin-set through `_KamiMarketRegistrySystem.setPurchaseCooldown(cooldown)`,
where `0` disables it. For the new owner, this cooldown blocks exactly the
actions in [What a Cooldown Blocks](#what-a-cooldown-blocks) — harvesting,
feeding and liquidating with that Kami — and nothing else, so the Kami can be
sent on or re-listed immediately.

> Source: `KamiSendSystem.sol:47–50, 70`, `LibKamiMarket.sol:271–277`,
> `_KamiMarketRegistrySystem.sol:70–73`, `LibCooldown.sol:34–53`

## Config

| Key | Value | Description |
|---|---|---|
| `KAMI_STANDARD_COOLDOWN` | `180` s (`30` in local test configs) | Base cooldown duration in seconds |
| `KAMI_MARKET_PURCHASE_COOLDOWN` | not set by init scripts → `3600` s fallback | Cooldown added to a Kami on send or marketplace sale |

> Source: `configs.ts:31, 124`, `KamiSendSystem.sol:48–50`

## Bonus Integration

The `STND_COOLDOWN_SHIFT` bonus type modifies cooldown duration. A positive
shift increases cooldown; a negative shift decreases it. This is how skills and
equipment can reduce action cooldowns.

> Source: `LibCooldown.sol:105`
