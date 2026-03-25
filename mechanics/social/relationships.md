# NPC Relationships

> Source: `packages/contracts/src/libraries/LibRelationship.sol` (L1–113),
> `packages/contracts/src/libraries/LibRelationshipRegistry.sol` (L1–170)

## Overview

Relationships are **flags** that track an account's standing with specific
NPCs. Unlike faction reputation (a numeric score), relationships are discrete
states — either the player has a relationship flag or they don't. Relationship
progression is controlled by **blacklists** and **whitelists** that gate
advancement based on what other relationship flags the player already holds.

## Registry Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"RELATIONSHIP"` |
| `IsRegistry` | Marks as registry entry |
| `IndexNPC` | NPC this relationship belongs to |
| `IndexRelationship` | Unique relationship flag index within that NPC |
| `Name` | (optional) Display name for this relationship state |
| `Blacklist` | (optional) Array of relationship indices that block advancement |
| `Whitelist` | (optional) Array of relationship indices required for advancement |

Registry ID: `keccak256("registry.relationship", npcIndex, relIndex)`

Unlike other registries, relationships have a **dual key** (npcIndex +
relIndex) since each NPC can have multiple relationship states.

> Source: `LibRelationshipRegistry.sol:24–37, 167–169`

## Instance Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"RELATIONSHIP"` |
| `IDOwnsRelationship` | Account that holds this relationship |
| `IndexNPC` | NPC index |
| `IndexRelationship` | Relationship flag index |

Instance ID: `keccak256("relationship", accID, npcIndex, relIndex)`

> Source: `LibRelationship.sol:21–33, 110–112`

## Advancement Rules

To advance to a new relationship flag (`canCreate`):

1. **Blacklist check**: If the player already holds ANY relationship in the
   blacklist, advancement is **blocked** (returns false immediately)
2. **Whitelist check**: If the whitelist is **empty**, advancement is valid.
   If the whitelist is non-empty, the player must hold at least ONE
   relationship in the whitelist to advance

```
canCreate(accID, npcIndex, relIndex):
  registry = getRegistry(npcIndex, relIndex)
  if anyBlacklistHeld(accID, registry) → false
  if whitelistEmpty(registry) → true
  if anyWhitelistHeld(accID, registry) → true
  → false
```

This enables branching relationship paths where certain choices block others
(via blacklists) while prerequisites gate advanced states (via whitelists).

> Source: `LibRelationship.sol:42–52, 65–92`

## Queries

### Check Relationship Existence

```solidity
LibRelationship.has(components, accID, npcIndex, relIndex) → bool
```

Returns true if the account has this specific relationship flag.

### Get Relationship Entity

```solidity
LibRelationship.get(components, accID, npcIndex, relIndex) → uint256
```

Returns the entity ID if the relationship exists, 0 otherwise.

> Source: `LibRelationship.sol:55–62, 97–105`

## Design Notes

The relationship system is designed as a **state machine** for NPC
interactions. Different NPCs can have completely different relationship
progressions. For example, an NPC might have:

- Flag 1: "Acquaintance" (no requirements)
- Flag 2: "Trusted" (whitelist: [1], requires Acquaintance)
- Flag 3: "Ally" (whitelist: [2], requires Trusted)
- Flag 4: "Enemy" (whitelist: [1], blacklist: [3], requires Acquaintance but
  blocks if already Ally)

> Source: Inferred from `LibRelationshipRegistry.sol` and blacklist/whitelist pattern
