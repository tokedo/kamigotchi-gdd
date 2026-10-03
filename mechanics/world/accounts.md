# Accounts & Stamina

> Source: `packages/contracts/src/libraries/LibAccount.sol` (L1–299),
> `packages/contracts/src/systems/AccountRegisterSystem.sol` (L1–42),
> `packages/contracts/src/systems/AccountSetOperatorSystem.sol` (L1–34),
> `packages/contracts/src/systems/AccountMoveSystem.sol` (L1–50),
> `packages/contracts/src/systems/AccountUseItemSystem.sol` (L1–41),
> `packages/contracts/deployment/world/state/configs/configs.ts`

## Overview

An account is the top-level player entity. It owns Kamis, holds inventory,
occupies a room, and has a stamina stat that gates movement. Accounts use an
**owner/operator** dual-address model where the owner is the wallet that
created the account and the operator is the address that executes actions.

## Account Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"ACCOUNT"` |
| `IndexAccount` | Sequential account number (global counter) |
| `AddressOwner` | Owner wallet address |
| `AddressOperator` | Operator address (session key) |
| `CacheOperator` | Reverse lookup: operator address → account ID |
| `IndexRoom` | Current room index (starts at 1) |
| `Stamina` | Stat struct (base/shift/boost/sync) |
| `TimeStart` | Account creation timestamp |
| `TimeLastAction` | Last stamina-relevant action (for recovery calc) |
| `TimeLast` | Last general action timestamp |
| `Name` | Account display name |
| `Description` | Account bio |

Entity ID: `addressToEntity(ownerAddress)` — derived from the owner's address.

> Source: `LibAccount.sol:45–71`

## Account Registration

`AccountRegisterSystem.execute(operator, name)`:

1. **World whitelist check** — if `WORLD_PRIVATE` config is true, owner must
   be whitelisted via `WORLD_WHITELIST` flag
2. **Address uniqueness**, in this order:
   - the caller (owner) must not already own an account (`"Account: exists
     for Owner"`)
   - the operator must not already be another account's operator
     (`"Account: exists for Operator"`)
   - the operator must not itself be an account owner (`"Account: Operator is
     an account owner"`)
3. **Name validation** — non-empty, max 16 characters, globally unique
4. Create account entity with:
   - Room set to **1** (starting room)
   - Stamina set to `ACCOUNT_STAMINA[0]` base value
   - Global account counter incremented

> Source: `AccountRegisterSystem.sol:15–37`, `LibAccount.sol:45–71, 164–177`

## Owner / Operator Model

- **Owner**: The wallet address that created the account. Used for high-security
  actions. Account entity ID is derived from this address.
- **Operator**: A separate address that executes day-to-day gameplay actions.
  Can be changed by the owner via `AccountSetOperatorSystem`. Enables session
  keys / delegated wallets.

`AccountSetOperatorSystem.execute(operator)` (owner-signed) applies the same
operator rules as registration: the new operator must not already be in use
(`"Account: Operator already in use"`) and must not be an account owner
(`"Account: Operator is an account owner"`). The previous operator's
reverse-lookup entry is removed and the new one written. Because an account's
ID is its owner address, "is an account owner" is the check
`isAccount(uint256(uint160(operator)))`.

The checks run only when an operator is assigned. An address that was already
an account's operator before it registered an account of its own is not
retroactively unassigned; systems that accept either signer resolve such an
address to **its own** account first (e.g. the token portal's operator lane,
see [token-portal.md](../marketplace/token-portal.md#player-entrypoints-and-signers)).

Operator lookup uses a cache component for efficient reverse mapping
(`operatorAddress → accountID`).

> Source: `AccountSetOperatorSystem.sol:15–29`, `LibAccount.sol:134–139,
> 164–177, 253–267`; `TokenPortalSystem.sol:238–244`

## Stamina System

Stamina is stored as a `Stat` struct (see [stats.md](../core-kami/stats.md))
with base/shift/boost/sync fields. The `sync` field tracks current stamina.

### Configuration (`ACCOUNT_STAMINA`)

| Index | Field | Local | Production |
|---|---|---|---|
| `[0]` | Total stamina (base) | 100 | 100 |
| `[1]` | Recovery period (seconds per point) | 1 | 60 |
| `[2]` | Movement cost (stamina per move) | 5 | 5 |
| `[3]` | XP per move | 5 | 5 |

> Source: `configs.ts:24–29, 71–76`
>
> **Local override**: Recovery period is `1` second in local/test environments
> (vs `60` seconds in production) for faster testing. See `configs.ts:24–29`.

### Recovery

Stamina recovers passively over time. Recovery is calculated on-demand when
the account is synced:

```
timePassed = now - lastActionTimestamp
recovery = floor(timePassed / recoveryPeriod)
currentStamina = min(sync + recovery, total)
```

Recovery rounds down — partial periods are lost. The `lastActionTimestamp` is
updated on sync.

> Source: `LibAccount.sol:88–97, 124–128`

### Depletion

Three actions consume stamina:

| Action | Cost |
|---|---|
| Movement | `ACCOUNT_STAMINA[2]` = 5 per move (`LibAccount.sol:80–85`) |
| Crafting | recipe's stamina cost × amount crafted (`LibRecipe.sol:132–133`, `beforeCraft` → `depleteStamina`) |
| Casting an item on an enemy Kami | 10 (hardcoded) (`KamiCastItemSystem.sol:33`) |

```
newStamina = currentStamina - cost
```

Reverts with `"Account: insufficient stamina"` if cost exceeds current stamina.

> Source: `LibAccount.sol:101–110`, `LibRecipe.sol:132–133`, `KamiCastItemSystem.sol:33`

## Account XP

Accounts have their own experience pool, separate from Kami XP. Account XP is
awarded to the **account entity** (not to any individual Kami).

Sources:
- **Movement** — `ACCOUNT_STAMINA[3]` = **5 XP** per room move
  (`LibAccount.sol:84`)
- **Crafting** — XP defined per recipe, awarded as `xp × amount` after craft
  (`LibRecipe.sol:153–158`)
- **Quest rewards** — quests with XP-type allocations award XP to the account
  (`LibQuest.sol:166` passes `accID` to `LibAllo.distribute`)

> **Important**: Account XP is a separate pool from Kami XP. There is no
> account-level level-up mechanism — only Kamis level up. See
> [experience-leveling.md](../core-kami/experience-leveling.md) for Kami XP.

> Source: `LibAccount.sol:80–85`, `LibRecipe.sol:153–158`

## Movement

`AccountMoveSystem.execute(toRoomIndex)`:

1. **Reachability** — destination must be adjacent or a special exit from
   current room (see [rooms.md](rooms.md))
2. **Accessibility** — gate conditions on the destination room must be met
3. **Sync** — recover stamina based on elapsed time
4. **Move** — deduct stamina cost, set new room, grant **account XP**
5. **Log** — increment `MOVE` counter, emit move event

> Source: `AccountMoveSystem.sol:22–45`, `LibAccount.sol:80–85`

## Item Usage (Account-Level)

`AccountUseItemSystem.execute(itemIndex, amount)`:

1. Verify item has `FOR` = `"ACCOUNT"` shape
2. Verify item usage requirements
3. Sync account (recover stamina)
4. Consume item from inventory
5. Apply item allocations (stat effects, bonuses, etc.)

This is for items that affect the **account** directly (e.g., stamina potions),
as opposed to items used on Kamis.

> Source: `AccountUseItemSystem.sol:18–36`

## Account Queries

| Query | Method |
|---|---|
| By owner address | `getByOwner(addr)` — derives entity ID from address |
| By operator address | `getByOperator(addr)` — uses CacheOperator reverse lookup |
| By name | `getByName(name)` — entity type + name component query |
| Owned Kamis | `getKamis(accID)` — queries `IDOwnsKami` component |

> Source: `LibAccount.sol:242–280`
