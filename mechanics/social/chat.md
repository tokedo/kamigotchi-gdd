# Chat & Echo

> Source: `packages/contracts/src/systems/ChatSystem.sol` (L1–59),
> `packages/contracts/src/libraries/LibEcho.sol` (L1–76),
> `packages/contracts/src/systems/EchoKamisSystem.sol` (L1–29),
> `packages/contracts/src/systems/EchoRoomSystem.sol` (L1–26)

## Chat System

### Overview

The chat system allows players to send messages within the game world. Messages
are emitted as on-chain events scoped to the player's current room. The system
supports configurable access requirements.

### Message Flow

`ChatSystem.execute(message)`:

1. Resolve account from operator address
2. Verify account meets chat requirements (via `LibConditional`)
3. Get the player's current room index
4. Update account's last activity timestamp
5. Log `MESSAGES` counter (incremented by 1)
6. Emit message event via `LibEmitter.emitMessage(world, roomIndex, accID, message)`

Messages are room-scoped — only players in the same room receive the event.

> Source: `ChatSystem.sol:17–33`

### Chat Requirements

Chat access can be gated by configurable requirements (added/removed by the
contract owner). Requirements use `LibConditional`, which can check items,
levels, quest completion, etc.

The requirement anchor is the chat system's own ID (`keccak256("system.chat")`).

```solidity
// Admin functions
ChatSystem.addRequirement(reqType, logicType, index, value, condFor)
ChatSystem.removeRequirement()
```

> Source: `ChatSystem.sol:40–58`

### Logging

| Data Key | Description |
|---|---|
| `MESSAGES` | Total messages sent by this account |

> Source: `ChatSystem.sol:30`

## Echo System

### Overview

The echo system is a utility for **re-emitting component values** as events.
This is used to force the front-end to refresh its state for a specific entity
without any actual data change. It works by reading and re-setting the same
raw bytes, which triggers event emission.

### System Entry Points

`EchoKamisSystem.executeTyped()` — no arguments:

1. Resolve account from operator address
2. Get all Kamis owned by the account
3. For each Kami, call `LibEcho.kami()` to re-emit all components

This broadcasts the full state of all the caller's Kamis to connected clients.

> Source: `EchoKamisSystem.sol:15–24`

`EchoRoomSystem.executeTyped()` — no arguments:

1. Resolve account from operator address
2. Call `LibEcho.room()` with the account ID

This re-emits the account's room assignment to connected clients.

> Source: `EchoRoomSystem.sol:15–21`

### Kami Echo

```solidity
LibEcho.kami(components, kamiID)
```

Re-emits all 21 components of a Kami entity:

| # | Component |
|---|---|
| 1 | EntityType |
| 2 | Health |
| 3 | Harmony |
| 4 | IDOwnsKami |
| 5 | IndexBody |
| 6 | IndexBackground |
| 7 | IndexColor |
| 8 | IndexFace |
| 9 | IndexHand |
| 10 | IndexKami |
| 11 | Experience |
| 12 | Level |
| 13 | MediaURI |
| 14 | Name |
| 15 | Power |
| 16 | Slots |
| 17 | SkillPoint |
| 18 | State |
| 19 | TimeLast |
| 20 | TimeStart |
| 21 | Violence |

Only components that have data for the entity are re-emitted (empty components
are skipped).

> Source: `LibEcho.sol:37–44, 51–75`

### Room Echo

```solidity
LibEcho.room(components, entityID)
```

Re-emits only the `IndexRoom` component for an entity. Used when a player's
room assignment needs to be broadcast to clients.

> Source: `LibEcho.sol:46–49`
