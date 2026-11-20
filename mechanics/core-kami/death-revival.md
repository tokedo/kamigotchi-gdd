# Death & Revival

> Source: `packages/contracts/src/libraries/LibKami.sol` (L76–79),
> `packages/contracts/src/systems/KamiOnyxReviveSystem.sol` (L1–49),
> `packages/contracts/src/systems/HarvestLiquidateSystem.sol` (L85),
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
  (`HarvestLiquidateSystem.sol:85`; see combat mechanics)
- **Sacrifice** — voluntary permanent death; the NFT is also sent to the burn
  address (`LibSacrifice.sol:110`; see sacrifice mechanics)

**Reaching 0 HP is not death.** A harvesting Kami whose HP drains to 0 stays in
`HARVESTING` state. It fails `verifyHealthy` checks (`"kami starving.."`), so
its owner cannot stop or collect the harvest (`HarvestStopSystem.sol:38`,
`HarvestCollectSystem.sol:36`) and it is liquidatable. It stays that way until
the owner **feeds it** — Food and Potion items are usable on a `HARVESTING`
Kami and item use has no health gate — after which stop/collect pass again, or
until another player liquidates it. See
[harvesting.md → Starving Kami: Feed First](../economy/harvesting.md#starving-kami-0-hp-feed-first).

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
carry `KAMI_DEAD`, every other Kami consumable `KAMI_CAN_EAT` = `RESTING` or
`HARVESTING`)
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

Four items carry the `USE` requirement `KAMI_DEAD` (`STATE` `IS` index 3 =
`DEAD`, `deployment/world/data/items/requirements.csv:5`) and revive a dead
Kami when used on it. Their effects set state back to `RESTING` and heal a flat
amount (allos `STATE-RESTING` + `HP+n`):

| Item | Index | Effect | USE requirement |
|---|---|---|---|
| Red Ribbon Gummy | 11001 | State → `RESTING`, +10 HP | `KAMI_DEAD` |
| "Melkarth's Heroic Awakening" Spell Card | 11002 | State → `RESTING`, +50 HP | `KAMI_DEAD` |
| Djed Pillar | 11003 | State → `RESTING`, +5 HP | `KAMI_DEAD` |
| Pale Potion | 11004 | State → `RESTING`, +75 HP | `KAMI_DEAD` |

Usage goes through the normal `KamiUseItemSystem` flow: the owner's account must
be in the Kami's room and the Kami must be off cooldown; there is no state or
health gate in the system itself (`KamiUseItemSystem.sol:26–32, 40–42`).

> Before `a99b4b64` the deployment derived each Kami item's state rule from its
> `Type` (`Revive` → `STATE == DEAD`, `Food` / `Potion` → `KAMI_CAN_EAT`,
> `Consumable` → `STATE == RESTING`). Only 11001 and 11002 (type `Revive`) could
> revive; Djed Pillar (type `Consumable`) required a `RESTING` Kami and Pale
> Potion (type `Potion`) a `RESTING` or `HARVESTING` one, so both worked only as
> heals on living Kamis despite their revival text. The rule is now an explicit
> `Requirements` cell per item, and all four carry `KAMI_DEAD`. Whether each
> live item was re-registered with the new rule is chain state.

> Source: `deployment/world/data/items/items.csv:75–78`,
> `deployment/world/data/items/requirements.csv:4–6`,
> `deployment/world/state/items/items.ts:8–20`,
> `deployment/world/state/items/requirements.ts:21–54`,
> `deployment/world/data/items/allos.csv` (STATE-RESTING),
> `LibGetter.sol:94–100` (STATE / KAMI_CAN_EAT checks)
