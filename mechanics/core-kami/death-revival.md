# Death & Revival

> Source: `packages/contracts/src/libraries/LibKami.sol` (L76–79),
> `packages/contracts/src/systems/KamiOnyxReviveSystem.sol` (L1–49),
> `packages/contracts/src/systems/HarvestLiquidateSystem.sol` (L84),
> `packages/contracts/src/libraries/LibSacrifice.sol` (L110),
> `packages/contracts/src/systems/KamiUseItemSystem.sol` (L19–52)

## Overview

A Kami dies only when `LibKami.kill()` is executed — by being **liquidated** or
**sacrificed**. Reaching 0 HP does **not** by itself kill a Kami. Dead Kamis
cannot perform Kami actions until revived. Revival costs Onyx Shards (or a
revive item) and restores partial health.

## Death

Death is the `kill()` state transition:

1. **State** → set to `"DEAD"`
2. **Health sync** → set to `0`

```solidity
function kill(components, id):
    setState(id, "DEAD")
    setSyncZero("HEALTH", id)
```

> Source: `LibKami.sol:76–79`

### What triggers death

`LibKami.kill()` has exactly **two call sites** — these are the only ways a Kami
dies:
- **Liquidation** — another player's Kami liquidates it while harvesting
  (`HarvestLiquidateSystem.sol:84`; see combat mechanics)
- **Sacrifice** — voluntary permanent death; the NFT is also sent to the burn
  address (`LibSacrifice.sol:110`; see sacrifice mechanics)

**Reaching 0 HP is not death.** A harvesting Kami whose HP drains to 0 stays in
`HARVESTING` state. It fails `verifyHealthy` checks, so its owner cannot stop or
collect the harvest (`HarvestStopSystem.sol:38`, `HarvestCollectSystem.sol:36`)
— it remains stuck at 0 HP and liquidatable until another player liquidates it
(see [health-healing.md](health-healing.md)).

### What dead Kamis cannot do

Dead Kamis fail all `verifyState(id, "RESTING")` and `verifyHealthy(id)` checks,
which means they cannot:
- Level up
- Harvest
- Equip/unequip items
- Be sent to another account
- Enter the marketplace

Items **can** be used on a dead Kami — `KamiUseItemSystem` performs no state or
health check; only each item's own `USE` requirements apply (revive items
require `STATE == DEAD`, food/potions require `RESTING` or `HARVESTING`)
(`KamiUseItemSystem.sol:26–32`). Quests are account-scoped and are not blocked
by any Kami's state.

> Source: `LibKami.sol:264–270, 284–294`

## Revival

Revival has two paths: the **Onyx revive** (`KamiOnyxReviveSystem`) and **revive
items** used via `KamiUseItemSystem` (see [Revive Items](#revive-items-kamiuseitemsystem)).

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

### Revive Items (KamiUseItemSystem)

Two items carry the `USE` requirement `STATE == DEAD` (item type `Revive` maps
to this requirement at deployment — `requirements.ts:77`) and revive a dead
Kami when used on it. Their effects set state back to `RESTING` and heal a flat
amount (allos `STATE-RESTING` + `HP+n`):

| Item | Index | Effect | USE requirement |
|---|---|---|---|
| Red Ribbon Gummy | 11001 | State → `RESTING`, +10 HP | `STATE == DEAD` |
| "Melkarth's Heroic Awakening" Spell Card | 11002 | State → `RESTING`, +50 HP | `STATE == DEAD` |

Usage goes through the normal `KamiUseItemSystem` flow: the owner's account must
be in the Kami's room and the Kami must be off cooldown; there is no state or
health gate in the system itself (`KamiUseItemSystem.sol:26–32, 40–42`).

Two further items carry the same revival-flavored effects (`STATE-RESTING` +
heal) but **cannot** be used on a dead Kami — their type-derived `USE`
requirements exclude the `DEAD` state:

| Item | Index | Effect | USE requirement |
|---|---|---|---|
| Djed Pillar | 11003 | State → `RESTING`, +5 HP | `STATE == RESTING` (type `Consumable`) |
| Pale Potion | 11004 | State → `RESTING`, +75 HP | `KAMI_CAN_EAT` = `RESTING` or `HARVESTING` (type `Potion`) |

> ⚠️ UNCERTAIN: the flavor text of Djed Pillar and Pale Potion describes
> reviving a liquidated Kami, but their deployed requirements make that
> impossible — as registered, they only function as heals on living Kamis.
> Likely a data-entry mismatch in `items.csv` (Type column vs. intent).

> Source: `deployment/world/data/items/items.csv:68–71`,
> `deployment/world/state/items/requirements.ts:75–79`,
> `deployment/world/data/items/allos.csv:59` (STATE-RESTING),
> `LibGetter.sol:94–100` (STATE / KAMI_CAN_EAT checks)
