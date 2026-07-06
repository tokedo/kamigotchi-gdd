# Naming & Renaming

> Source: `packages/contracts/src/systems/KamiNameSystem.sol` (L1–55),
> `packages/contracts/src/systems/KamiOnyxRenameSystem.sol` (L1–54),
> `packages/contracts/src/libraries/LibKami.sol` (L107–119)

## Overview

Kamis start with a default sequential name (e.g., "Kamigotchi 42"). Players can
name their Kami via `KamiNameSystem` — repeatable without limit, costing 1 Holy
Dust each time — or rename it for Onyx Shards (currently disabled). Names are
globally unique and max 16 characters.

## Naming (KamiNameSystem)

Naming is **repeatable without limit** — the system performs no
nameable/already-named check; each naming simply consumes 1 Holy Dust in
room 11. (A `NOT_NAMEABLE` flag mechanism exists in `LibKami` but is unused —
see [Nameable Flag](#nameable-flag).)

### Requirements

| Requirement | Value |
|---|---|
| Location | Room index **11** (specific room) |
| Cost | **1 Holy Dust** (item index 11011) |
| Name length | 1–16 characters |
| Uniqueness | Name must not be taken by any other Kami |

### Process

1. Verify caller owns the Kami
2. Verify Kami is in **room 11**
3. Verify account has at least 1 Holy Dust
4. Validate name: not empty, max 16 chars, not already taken
5. Consume 1 Holy Dust
6. Set the Kami's name

> Source: `KamiNameSystem.sol:22–49`

## Renaming (KamiOnyxRenameSystem)

> **Currently disabled** — the system reverts with "Onyx Features are temporarily
> disabled." as of commit `d9b5009`.

When enabled, renaming allows changing an already-named Kami's name.

### Requirements

| Requirement | Value |
|---|---|
| Location | Room index **11** |
| Cost | **5,000 Onyx Shards** (item index 100) |
| Name length | 1–16 characters |
| Uniqueness | Name must not be taken by any other Kami |

### Process

1. Verify caller owns the Kami
2. Verify Kami is in **room 11**
3. Validate name: not empty, max 16 chars, not already taken
4. Spend 5,000 Onyx Shards
5. Set the new name

> Source: `KamiOnyxRenameSystem.sol:22–47`

## Name Validation Rules

Both systems enforce:
- **Not empty**: `bytes(name).length > 0`
- **Max 16 characters**: `bytes(name).length <= 16`
- **Globally unique**: `LibKami.getByName(name) == 0` (no existing Kami has this name)

> Source: `KamiNameSystem.sol:36–38`, `KamiOnyxRenameSystem.sol:33–35`

## Nameable Flag

`LibKami` maintains a `NOT_NAMEABLE` flag per Kami:
- Default: `false` (Kamis can be named)
- `useNameable(id)` — checks the flag and sets it to `true` (returns `true` if
  the Kami was nameable, consuming the opportunity)
- `setNameable(id, bool)` — directly sets whether a Kami can be named

This is an **inverse flag** (NOT_NAMEABLE) for gas optimization — new Kamis don't
need a flag component set on creation.

**This machinery is dead code**: `useNameable` has zero callers anywhere in
`src/` — neither `KamiNameSystem` nor `KamiOnyxRenameSystem` consults the flag,
so it never gates naming.

> Source: `LibKami.sol:107–119`

## Tracking

Name changes are logged:
- `KAMI_NAME` counter incremented per account
- For naming: Holy Dust usage logged via `LibItem.logUse`
- For Onyx renaming, spend counters are also incremented:
  `TOKEN_SPEND[accID, ONYX_INDEX]`, `TOKEN_SPEND[0, ONYX_INDEX]`, and
  `TOKEN_SPEND_RENAME[0, ONYX_INDEX]`

> Source: `KamiNameSystem.sol:45–46`, `KamiOnyxRenameSystem.sol:42, 44–46`
