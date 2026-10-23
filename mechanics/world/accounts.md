# Accounts & Stamina

> Source: `packages/contracts/src/libraries/LibAccount.sol` (L1–311),
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
address to **its own** account first (`getByOwnerOrOperator`, and the token
portal's operator lane — see [Who May Sign](#who-may-sign)).

Operator lookup uses a cache component for efficient reverse mapping
(`operatorAddress → accountID`).

> Source: `AccountSetOperatorSystem.sol:15–29`, `LibAccount.sol:134–139,
> 164–177, 253–279`; `TokenPortalSystem.sol:238–244`

## Who May Sign

Every player-facing system turns the calling address (`msg.sender`) into an
account with one of three resolvers (see [Account Queries](#account-queries)).
This is the complete list at the pin; other pages link here rather than
restating it.

**Owner only.** The sender must be the owner wallet itself
(`getByOwner(msg.sender)`); any other address — an operator included —
reverts `"Account: no account detected"`.

| System | Entrypoints | Source |
|---|---|---|
| `AccountSetOperatorSystem` | `execute(operator)` | `AccountSetOperatorSystem.sol:21` |
| `AccountSetNameSystem` | `execute(name)` | `AccountSetNameSystem.sol:17` |
| `TokenPortalSystem` | `deposit`, `withdraw`; `claim` / `cancel` of an owner receipt (sender must equal the receipt account's owner, else `"not receipt owner"`) | `TokenPortalSystem.sol:43, 61, 248–261` |
| `Kami721StakeSystem`, `Kami721UnstakeSystem` | `execute`, `executeBatch` (bridge in / out) | `Kami721StakeSystem.sol:32`, `Kami721UnstakeSystem.sol:36` |
| `KamiMarketBuySystem` | `executeTyped(listingIDs)` (pays ETH) | `KamiMarketBuySystem.sol:29` |
| `NewbieVendorBuySystem` | `executeTyped(kamiIndex)` (pays ETH) | `NewbieVendorBuySystem.sol:51` |
| `GachaBuyTicketSystem` | `buyPublic(amount)`, `buyWL()` | `GachaBuyTicketSystem.sol:43, 63` |
| `KamiOnyxRenameSystem`, `KamiOnyxRespecSystem` | `execute` — disabled: an unconditional revert runs before the owner lookup | `KamiOnyxRenameSystem.sol:23, 26`, `KamiOnyxRespecSystem.sol:21, 24` |

Registration (`AccountRegisterSystem`) is signed by the wallet that becomes
the owner: the account ID is derived from `msg.sender`
(`AccountRegisterSystem.sol:19–32`).

**Owner or operator.** `getByOwnerOrOperator(msg.sender)`: if the sender owns
an account, that account (owner first — an owner address never resolves to an
account that merely named it operator); otherwise the account that named the
sender its operator; an address that is neither reverts
`"Account: no account detected"`.

| System | Entrypoints | Source |
|---|---|---|
| `AuctionBuySystem` | `execute(itemIndex, amt)` | `AuctionBuySystem.sol:23` |
| `ItemTransferSystem` | `execute(indices, amts, targetID)` | `ItemTransferSystem.sol:21` |
| `KamiGachaMintSystem` | `execute(amount)` | `KamiGachaMintSystem.sol:28` |
| `KamiGachaRerollSystem` | `reroll(kamiIDs)` | `KamiGachaRerollSystem.sol:23` |
| `TradeCreateSystem`, `TradeExecuteSystem`, `TradeCompleteSystem`, `TradeCancelSystem` | `execute` (the complete / cancel `executeAdmin` batches are admin-only) | `TradeCreateSystem.sol:24`, `TradeExecuteSystem.sol:18`, `TradeCompleteSystem.sol:19`, `TradeCancelSystem.sol:19` |
| `TokenPortalSystem` | `withdrawToOperator` — the same owner-first order through the portal's own helper, but a sender that is neither reverts `"Account: Operator not found"`; `claim` / `cancel` of an operator-lane receipt — the account's owner or its **current** operator, else `"not receipt owner"` | `TokenPortalSystem.sol:88, 240–244, 248–261` |

**Operator.** Every other player-facing system resolves the sender as an
operator (`getByOperator(msg.sender)` or `verifyOperator()`); an address that
is no account's operator — an owner wallet included — reverts
`"Account: Operator not found"`. (`verifyOperator`'s own message
`"LibAccount: account operator not found"` is unreachable: `getByOperator`
reverts first.) This covers movement, account bio and picture, item use, cast
and burn, equipment, crafting, harvesting and liquidation, scavenge claims,
quests, goals, skills, relationships, friends, chat, Kami level / name / send /
sacrifice / Onyx revive, the NPC shop (`ListingBuySystem`,
`ListingSellSystem`), the item pools (`PoolSystem`), and the Kami
marketplace's list, offer, accept-offer and cancel. The reveal entrypoints
(`KamiGachaRevealSystem.reveal`, `DroptableRevealSystem.execute`) resolve no
account: anyone may call them.

**Operators are never owners.** An address that owns an account cannot be
assigned as an operator (`"Account: Operator is an account
owner"`, `AccountRegisterSystem.sol:24–25`, `AccountSetOperatorSystem.sol:18–19`),
and because the owner-or-operator resolvers check ownership first, an owner's
own account always wins.

Addresses passed as **arguments** follow their own rule: `KamiSendSystem`
resolves the recipient from its **operator** address (`KamiSendSystem.sol:45`,
see [kami-send.md](../core-kami/kami-send.md)); `FriendRequestSystem` and
`FriendBlockSystem` resolve the target from its **owner** address
(`FriendRequestSystem.sol:20`, `FriendBlockSystem.sol:23`).

> Deployed state (chain, not source): the ten systems changed by upstream
> commit `04cb25f8` — the eight owner-or-operator systems above plus
> `KamiSendSystem` and `KamiSacrificeCommitSystem` — were redeployed on chain
> on 2026-10-21 between about 11:00 and 11:10 UTC (blocks
> ≈34,757,700–34,758,400). Before that, the eight accepted the owner only.

> Source: `LibAccount.sol:179–191` (`verifyOperator`, `verifyOwner`),
> `:253–279` (`getByOperator`, `getByOwner`, `getByOwnerOrOperator`); the
> per-system lines above; operator resolution e.g. `AccountMoveSystem.sol:23`,
> `KamiMarketListSystem.sol:25`, `KamiSendSystem.sol:44`;
> `KamiGachaRevealSystem.sol:18`, `DroptableRevealSystem.sol:19`

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

`AccountMoveSystem.execute(toRoomIndex)` — signed by the account's
**operator**:

1. **Reachability** — destination must be adjacent or a special exit from
   current room (see [rooms.md](rooms.md)); else `"AccMove: unreachable room"`
2. **Accessibility** — gate conditions on the destination room must be met;
   else `"AccMove: inaccessible room"`
3. **Sync** — recover stamina based on elapsed time
4. **Move** — deduct stamina cost (`"Account: insufficient stamina"` if
   short), set new room, grant **account XP**
5. **Log** — increment `MOVE` counter, emit move event

Reachability is checked **before** accessibility: a non-adjacent destination
reverts `unreachable` even when it is also gated. See
[rooms.md → Move Check Order](rooms.md#move-check-order-and-revert-strings).

> Source: `AccountMoveSystem.sol:22–45`, `LibAccount.sol:80–85, 101–107`

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
| By owner address | `getByOwner(addr)` — entity ID `uint256(uint160(addr))`; reverts `"Account: no account detected"` unless that entity is an `ACCOUNT` |
| By operator address | `getByOperator(addr)` — `CacheOperator` reverse lookup; reverts `"Account: Operator not found"` on 0 |
| By owner, else operator | `getByOwnerOrOperator(addr)` — `uint256(uint160(addr))` if that entity is an `ACCOUNT`; otherwise the `CacheOperator` lookup; 0 reverts `"Account: no account detected"` |
| By name | `getByName(name)` — entity type + name component query |
| Owned Kamis | `getKamis(accID)` — queries `IDOwnsKami` component |

`verifyOwner()` / `verifyOperator()` wrap `getByOwner(msg.sender)` /
`getByOperator(msg.sender)`. No system calls `verifyOwner` at the pin.

> Source: `LibAccount.sol:179–191, 242–292`
