# Naming & Renaming

> Source: `packages/contracts/src/systems/KamiNameSystem.sol` (L1–55),
> `packages/contracts/src/systems/KamiOnyxRenameSystem.sol` (L1–54),
> `packages/contracts/src/libraries/LibKami.sol` (L107–119)

## Overview

Kamis start with a default sequential name (e.g., "Kamigotchi 42"). Players can
name their Kami once for free (costs Holy Dust) and rename it later for Onyx
Shards. Names are globally unique and max 16 characters.

## First Naming (KamiNameSystem)

The first naming uses the **`NOT_NAMEABLE` flag** — it defaults to `false` (Kamis
are nameable), and is set to `true` after the first naming. This is a one-time
free naming opportunity.

> **Note**: The current code does not explicitly check the flag in KamiNameSystem;
> instead it consumes Holy Dust. The flag mechanism exists in LibKami for
> future use or was used in a prior version.

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

> Source: `LibKami.sol:107–119`

## Tracking

Name changes are logged:
- `KAMI_NAME` counter incremented per account
- For first naming: Holy Dust usage logged via `LibItem.logUse`

> Source: `KamiNameSystem.sol:45–46`, `KamiOnyxRenameSystem.sol:43`
