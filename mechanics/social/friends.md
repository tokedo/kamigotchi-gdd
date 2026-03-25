# Friends

> Source: `packages/contracts/src/libraries/LibFriend.sol` (L1–186),
> `packages/contracts/src/systems/FriendRequestSystem.sol` (L1–53),
> `packages/contracts/src/systems/FriendAcceptSystem.sol` (L1–48),
> `packages/contracts/src/systems/FriendBlockSystem.sol` (L1–44),
> `packages/contracts/src/systems/FriendCancelSystem.sol` (L1–58)

## Overview

The friend system enables **bidirectional social connections** between player
accounts. Friendships use a **one-way pointer** model: each friendship is
represented by two entities (A→B and B→A). The system supports requesting,
accepting, blocking, and canceling friendships, with configurable limits on
friend count and pending requests.

## Friendship Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"FRIENDSHIP"` |
| `IdSource` | Account that created this direction of the relationship |
| `IdTarget` | Account on the other end |
| `State` | Current state: `"REQUEST"`, `"FRIEND"`, or `"BLOCKED"` |
| `IDAnchor` | Counter pointer for tracking friend/request counts |

Entity ID: `keccak256("friendship", accID, targetID)`

Each friendship has **two** entities with opposite source/target. When A
requests B, one entity exists (A→B, state=REQUEST). When B accepts, a second
entity is created (B→A) and both are set to state=FRIEND.

> Source: `LibFriend.sol:38–52, 178–179`

## States

| State | Description |
|---|---|
| `REQUEST` | Pending request — only the sender's entity exists |
| `FRIEND` | Accepted — both A→B and B→A entities exist with state=FRIEND |
| `BLOCKED` | Blocked — one-way block entity, counter pointer removed |

## Friend Request Flow

### Sending a Request

`FriendRequestSystem.execute(targetAddress)`:

1. Resolve target account from wallet address
2. Verify not requesting self
3. Verify target's pending request count < `FRIENDS_REQUEST_LIMIT` (config: **10**)
4. Verify no existing friendship entity from requester→target
5. Verify no incoming request/friendship/block from target→requester
6. Create friendship entity (state=REQUEST) and update request counter

> Source: `FriendRequestSystem.sol:17–47`

### Accepting a Request

`FriendAcceptSystem.execute(requestID)`:

1. Verify the entity is a friendship in REQUEST state
2. Verify the accepter is the request target
3. Check friend count limits for both parties:
   ```
   limit = FRIENDS_BASE_LIMIT + bonus("FRIENDS_LIMIT", accID)
   ```
   Base limit: **10** (configurable). Bonuses can increase this.
4. Create accepter's friendship entity (B→A, state=FRIEND)
5. Update requester's entity state to FRIEND
6. Update friend counters for both accounts

> Source: `FriendAcceptSystem.sol:18–42`, `LibFriend.sol:55–74`

### Blocking

`FriendBlockSystem.execute(targetAddress)`:

1. Verify not blocking self
2. Remove any existing friendship from target→account (if not already blocked)
3. Create/overwrite account→target friendship with state=BLOCKED
   (the IDAnchor counter pointer is removed for blocked entities)

Blocking automatically unfriends if a friendship existed.

> Source: `FriendBlockSystem.sol:20–38`

### Canceling (Unfriend/Cancel Request/Unblock)

`FriendCancelSystem.execute(friendshipID)`:

| Current State | Who Can Cancel | What Happens |
|---|---|---|
| `REQUEST` | Either party (sender or target) | Delete the request entity |
| `BLOCKED` | Only the blocker | Delete the block entity |
| `FRIEND` | Only the owner of this direction | Delete both A→B and B→A entities |

> Source: `FriendCancelSystem.sol:17–52`

## Counters

Friend counts and request counts are tracked via `IDAnchor` pointers:

| Counter | ID | Description |
|---|---|---|
| Friend count | `keccak256("friendship.ptr", accID, "FRIEND")` | Number of confirmed friends |
| Request count | `keccak256("friendship.ptr", accID, "REQUEST")` | Number of pending incoming requests |

Counts are determined by the `size()` of entities anchored to the counter pointer.

> Source: `LibFriend.sol:78–101, 165–173, 182–185`

## Configuration

| Config Key | Value | Description |
|---|---|---|
| `FRIENDS_BASE_LIMIT` | 10 | Base maximum number of friends per account |
| `FRIENDS_REQUEST_LIMIT` | 10 | Maximum pending incoming requests per account |

The friend limit can be increased by bonuses of type `FRIENDS_LIMIT`.

> Source: `configs.ts:80–81`, `FriendAcceptSystem.sol:30–32`
