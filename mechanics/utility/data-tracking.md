# Data Tracking (LibData)

> Source: `packages/contracts/src/libraries/LibData.sol` (L1–267)

## Overview

LibData is the **per-holder statistics/counter store**: a generic key-value
entity pattern used for lifetime statistics, quest objective snapshots, and
miscellaneous bookkeeping. Values are packed into a single `uint256` on
`ValueComponent`, either as a plain number or as a packed `uint32[8]` array.

> Source: `LibData.sol:11–18`

## Entity Shape

Data ID: `keccak256("is.data", holderID, index, type)`

| Key part | Meaning |
|---|---|
| `holderID` | Owning entity — usually an account; `0` for global totals (`LibAccount.sol:286–287`); can be any entity, e.g. a token address cast to uint (`LibTokenPortal.sol:303`) or a fresh log entity (`LibKill.sol:283–287`) |
| `index` | Numeric sub-key — often an item/currency index (`LibListing.sol:181`), a node or account index (`LibKill.sol:292–303`), or `0` when unused |
| `type` | String key naming the statistic (e.g. `"TOKEN_SPEND"`) |

> Source: `LibData.sol:22–28`

## Operations

| Function | Description |
|---|---|
| `getID(holder, index, type)` | Derive the data entity ID (`LibData.sol:22–28`) |
| `inc(...)` | Increment counter; overloads for batched holders/indices/types (`LibData.sol:33–140`) |
| `dec(...)` | Decrement counter, same overload family (`LibData.sol:142–223`) |
| `set(...)` | Set value directly (`LibData.sol:225–238`) |
| `setArray(...)` | Set a packed `uint32[8]` via `LibPack` (`LibData.sol:240–249`) |
| `get(...)` | Read value; returns 0 when unset (`safeGet`) (`LibData.sol:254–266`) |

## Consumers

### Quest INCREASE/DECREASE objectives

When a quest is accepted, every objective whose logic handler is `INCREASE`
or `DECREASE` gets a snapshot entity storing the current
`LibData.get(account, index, type)` value (`LibQuest.sol:106–151`, read at
`:138`). At completion, the delta between the current data value and the
snapshot is compared against the objective threshold (`checkIncrease`,
`LibQuest.sol:232–247`; `checkDecrease`, `LibQuest.sol:248–262`).

### Conditionals and generic setters

- `LibGetter.getBal` falls through to `LibData.get` for any type string it
  does not special-case, so conditional checks (`CURR_*`) can gate on any
  data counter (`LibGetter.sol:45–70`, fallback at `:68`).
- `LibSetter.update` falls through to `LibData.inc` for reward types it does
  not handle, so a basic allocation with an unrecognized type string
  increments a data counter instead of granting anything
  (`LibSetter.sol:38–67`, fallback at `:65`).

### Statistics / analytics logging

Systems log lifetime counters on most player actions. Representative keys:

| Type key | Holder (index) | Written by |
|---|---|---|
| `TOKEN_SPEND`, `TOKEN_SPEND_REVIVE` / `_RESPEC` / `_RENAME` | Account and global 0 (ONYX index) | `KamiOnyxReviveSystem.sol:39–41`, `KamiOnyxRespecSystem.sol:39–41`, `KamiOnyxRenameSystem.sol:44–46` |
| `KAMI_LEVELS_TOTAL` | Account | `LibExperience.sol:105–107`, called with the account ID (`KamiLevelSystem.sol:43`) |
| `LIQUIDATE_TOTAL`, `LIQUIDATE_AT_NODE`, `LIQ_WHEN_{PHASE}` | Account (node index for AT_NODE) | `LibKill.sol:290–298` |
| `LIQUIDATED_VICTIM`, `LIQ_TARGET_ACC` | Victim / attacker account | `LibKill.sol:301–304` |
| `QUEST_COMPLETE`, `QUEST_REPEATABLE_COMPLETE` | Account | `LibQuest.sol:377–382` |
| `TRADE_CREATE` / `_EXECUTE` / `_COMPLETE` / `_CANCEL`, `TRADE_TAX` | Account | `LibTrade.sol:146, 184, 367–405` |
| `ITEM_COUNT` | Global 0 (item index) | `LibInventory.sol:197` |
| `TOTAL_NUM_ACCOUNTS` | Global 0 | `LibAccount.sol:286–287` |

## Relationship to LibScore

LibScore (leaderboard scores) is a **separate, parallel system** — it does
not build on LibData and references no data entities. Its header comment
states when scores are preferred over data: when the value must be
reverse-mappable on the frontend (leaderboards), when a global total per type
is tracked, or when an individual's percentage of the total is needed
(`LibScore.sol:16–35`).
