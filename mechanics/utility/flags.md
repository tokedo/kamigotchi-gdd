# Flag System

> Source: `packages/contracts/src/libraries/LibFlag.sol` (L1–187)

## Overview

Flags are **boolean meta-entities** that mark whether a parent entity has a
specific property. They are used throughout Kamigotchi for gating, state
tracking, and role management.

## Flag Entity Shape

| Component | Description |
|---|---|
| `HasFlag` | Boolean presence marker |
| `IDOwnsFlag` | (optional) Parent entity ID, for reverse lookup |
| `IDType` | (optional) Type anchor hash, for categorized queries |
| `Type` | (optional) Flag type string, for frontend |

Flag ID: `keccak256("has.flag", holderID, flagType)`

> Source: `LibFlag.sol:29–63, 177–179`

## Operations

| Function | Description |
|---|---|
| `set(holderID, flagType, state)` | Set or remove a bare flag |
| `setFull(holderID, parentType, flagType)` | Set flag with reverse mapping |
| `has(holderID, flagType)` | Check if flag exists |
| `checkAll(ids, flag, state)` | Check all entities have/don't have flag |
| `checkAny(ids, flag, state)` | Check any entity has/doesn't have flag |
| `getAndSet(id, flagType, state)` | Atomic read + update |
| `queryFor(holderID)` | Get all reverse-mapped flags for an entity |

> Source: `LibFlag.sol:37–171`

> Note: `queryFor` reads `IDOwnsFlagComponent` (`LibFlag.sol:166–172`), which
> only `setFull` writes (`LibFlag.sol:49–63`). Flags created with bare `set`
> are not returned by `queryFor`.

## Known Flag Types

| Flag | Used By | Description |
|---|---|---|
| `NEWBIE_VENDOR_PURCHASED` | Newbie vendor | One-time purchase tracking (`NewbieVendorBuySystem.sol:48, 61`) |
| `MINT_WHITELISTED` | Gacha ticket system | Whitelist mint eligibility (`GachaBuyTicketSystem.sol:130`) |
| `ITEM_UNBURNABLE` | Item burn system | Items are burnable by default; burning reverts if any input item carries this flag (`LibItem.sol:283`) |
| `NOT_NAMABLE` | — | Defined in comments only (`LibKami.sol:50`, `LibKamiCreate.sol:70`); enforced nowhere — `KamiNameSystem` reads no flag |
| `ROLE_ADMIN`, `ROLE_COMMUNITY_MANAGER` | Auth system | Role gating for admin / community-manager functions (`AuthRoles.sol:14, 22`); set with parentType `"AUTH"` via `_AuthManageRoleSystem` (`_AuthManageRoleSystem.sol:21`) |

Flags are also used by the conditional system (`BOOL_IS` / `BOOL_NOT` logic)
to gate actions behind flag requirements.
