# Death & Revival

> Source: `packages/contracts/src/libraries/LibKami.sol` (L76–79),
> `packages/contracts/src/systems/KamiOnyxReviveSystem.sol` (L1–49)

## Overview

A Kami dies when its health reaches 0 (from harvest strain or combat). Dead Kamis
cannot perform any actions until revived. Revival costs Onyx Shards and restores
partial health.

## Death

When a Kami's health reaches 0, `LibKami.kill()` is called:

1. **State** → set to `"DEAD"`
2. **Health sync** → set to `0`

```solidity
function kill(components, id):
    setState(id, "DEAD")
    setSyncZero("HEALTH", id)
```

> Source: `LibKami.sol:76–79`

### What triggers death

Death can occur from:
- **Harvest strain** — HP drains to 0 while harvesting (see [health-healing.md](health-healing.md))
- **Murder/PvP** — another player kills the Kami (see combat mechanics)
- **Sacrifice** — voluntary permanent death for rewards (see sacrifice mechanics)

### What dead Kamis cannot do

Dead Kamis fail all `verifyState(id, "RESTING")` and `verifyHealthy(id)` checks,
which means they cannot:
- Level up
- Harvest
- Equip/unequip items
- Use items
- Be sent to another account
- Enter the marketplace
- Accept quests

> Source: `LibKami.sol:264–270, 284–294`

## Revival

Revival is performed via `KamiOnyxReviveSystem`.

### Cost

| Resource | Amount |
|---|---|
| Onyx Shards (item index 100) | **33 shards** |

> Source: `KamiOnyxReviveSystem.sol:14`

### Process

1. **Verify ownership** — caller's account must own the Kami
2. **Verify state** — Kami must be `"DEAD"` (reverts otherwise)
3. **Spend Onyx** — deduct 33 Onyx Shards from the account's inventory
4. **Sync** — update any pending state
5. **Set state** → `"RESTING"`
6. **Heal** — restore **33 HP** (not full health)

```
KamiOnyxReviveSystem.execute(kamiIndex):
    verify owner
    require state == "DEAD"
    spend 33 Onyx Shards
    sync(kami)
    setState(kami, "RESTING")
    heal(kami, 33)
```

> Source: `KamiOnyxReviveSystem.sol:21–43`

### Post-Revival State

After revival:
- **State**: `RESTING`
- **Health**: `33 HP` (regardless of max health)
- **Other stats**: unchanged (power, violence, harmony, etc. are not affected)
- The Kami immediately begins passive healing from metabolism (see [health-healing.md](health-healing.md))

### Tracking

Revival spending is logged in three counters:
- Per-account Onyx spend: `TOKEN_SPEND[accID, ONYX_INDEX]`
- Global Onyx spend: `TOKEN_SPEND[0, ONYX_INDEX]`
- Global revive spend: `TOKEN_SPEND_REVIVE[0, ONYX_INDEX]`

> Source: `KamiOnyxReviveSystem.sol:39–41`
