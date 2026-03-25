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
| `queryFor(holderID)` | Get all flags for an entity |

> Source: `LibFlag.sol:37–171`

## Known Flag Types

| Flag | Used By | Description |
|---|---|---|
| `NEWBIE_VENDOR_PURCHASED` | Newbie vendor | One-time purchase tracking |
| `MINT_WHITELISTED` | Gacha ticket system | Whitelist mint eligibility |
| `ITEM_BURNABLE` | Item burn system | Whether an item can be burned |
| `NOT_NAMABLE` | Kami naming | Prevents renaming (default: false) |
| Community roles | Auth system | `commManager`, `moderator`, etc. |

Flags are also used by the conditional system (`BOOL_IS` / `BOOL_NOT` logic)
to gate actions behind flag requirements.
