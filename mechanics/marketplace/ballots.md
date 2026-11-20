# Ballots (Initia VIP Gauge Voting Rewards)

> Source: `packages/contracts/src/libraries/LibBallot.sol` (L1–355),
> `packages/contracts/src/systems/BallotSettleSystem.sol` (L1–58),
> `packages/contracts/src/systems/BallotClaimSystem.sol` (L1–30),
> `packages/contracts/deployment/world/state/configs/configs.ts` (L242–273),
> `packages/contracts/deployment/world/data/items/items.csv`,
> `packages/contracts/deployment/world/data/listings/listings.csv`,
> `packages/contracts/test/systems/Ballot.t.sol`

## Overview

Accounts whose owner wallet votes for **Yominet** in Initia's VIP gauge earn
**Ballots**, soulbound items spent at a merchant. Votes live on Initia L1,
which the contracts cannot read: a trusted **ballot reporter** wallet reads
each voter's final vote for a closed gauge cycle off L1 and **settles** it on
chain, which records what the account is owed; the player then **claims**,
which mints the items.

Ballots are kept apart from the VIP system ([vip.md](vip.md)): they never touch
VIPP or the VIP score, and nothing on chain ties the gauge cycle to the VIP
stage (each has its own config).

```
L1 gauge vote ──(reporter job)──→ BallotSettleSystem ──→ owed balances (LibData)
                                                              │
                     player (owner or operator) ──→ BallotClaimSystem ──→ items minted
```

> Source: `LibBallot.sol:19–54`, `BallotSettleSystem.sol:13–15`,
> `BallotClaimSystem.sol:12–13`

## Items

| Index | Name | Type | Rarity | How it is earned |
|---|---|---|---|---|
| 13 | Ballot | Misc | Uncommon | Every settled cycle with Yominet power (pool share, below) |
| 14 | Golden Ballot | Misc | Epic | Every 5th cycle of a voting streak |
| 15 | Voter Badge I | Cosmetic | Uncommon | Streak 5, once per account |
| 16 | Voter Badge II | Cosmetic | Rare | Streak 10, once per account |
| 17 | Voter Badge III | Cosmetic | Rare | Streak 26, once per account |
| 18 | Voter Badge IV | Cosmetic | Epic | Streak 52, once per account |

All six carry `NOT_TRADABLE` (an item transfer reverts `"Transfer includes
untradeable item"`). A claim refuses to mint an item that is not registered
(`"Ballot: item not registered"`), since an unregistered item would mint
without its flag. Indices are constants: `BALLOT_INDEX = 13`,
`GOLDEN_BALLOT_INDEX = 14`, `VOTER_BADGE_I_INDEX = 15` (badges 15–18).

> Source: `LibBallot.sol:14–16, 217–221, 271–274`,
> `deployment/world/data/items/items.csv` (rows 13–18),
> `Ballot.t.sol:449–470`

## Cycles

```
cycle = BALLOT_CYCLE[2] + (block.timestamp − BALLOT_CYCLE[0]) / BALLOT_CYCLE[1]
```

`BALLOT_CYCLE = [start, length, base]`: gauge cycle `base` starts at `start`;
`base` is also the first cycle that can settle. Reverts `"Ballot: cycles not
configured"` when start or length is 0 and `"Ballot: cycles not started"`
before `start`.

Deploy data: `[1793260800, 1209600, 40]` — cycle 40 from 2026-10-29 08:00 UTC,
two weeks each, so cycle 41 runs 2026-11-12 → 2026-11-26 08:00 UTC and 42
onward follows L1's two-week grid. The data comment explains the anchor: on L1,
epoch 41 ran three weeks (to 2026-11-26 08:00 UTC), so cycle 40's start is set
a week late here.

> Source: `LibBallot.sol:59–65`, `configs.ts:269–272`

## Settling (reporter only)

`BallotSettleSystem.executeTyped(address owner, uint64 cycle, uint256
yominetPower, uint256 eligibleWeight, bool streakVote)`:

1. `msg.sender` must equal the `BALLOT_REPORTER` config address — else
   `"Ballot: not reporter"`. The deploy step writes the zero address when no
   reporter is given, so settling fails closed
2. The account is the one **owned** by `owner` (`LibAccount.getByOwner`;
   `"Account: no account detected"` otherwise)
3. `yominetPower > 0` — else `"Ballot: no yominet power"` (power is in uinit,
   1 INIT = 10⁶)
4. Cycle window (`verifyCycle`), with `current` = the cycle now:
   - `cycle > 0` and `cycle ≥ base` — else `"Ballot: cycle before launch"`
   - `cycle < current` (closed cycles only) — else `"Ballot: cycle still open"`
   - `cycle + 7 ≥ current` (`SETTLE_WINDOW = 7`) — else `"Ballot: cycle too old"`
   - not yet settled for this account — else `"Ballot: cycle already settled"`
5. Record `BALLOT_SETTLED = 1` and `BALLOT_POWER = yominetPower` for
   (account, cycle); increment the cycle's `BALLOT_VOTERS`
6. Compute the Ballots (below) and add them to the account's owed balance
7. Update the streak and owe milestone items (below)
8. Emit `BALLOT_SETTLE` `(accID, cycle, power, ballots, streak)`; return the
   Ballots

Settling mints nothing — items appear only on claim.

> Source: `BallotSettleSystem.sol:19–41`, `LibBallot.sol:17, 84–115, 319–339`,
> `LibAccount.sol:264–267`, `configs.ts:242–248`

### Ballots per cycle

Voting power is converted to a **weight** in millionths of a Ballot through
four bands (`BALLOT_TIERS = [cap1, cap2, cap3, div1, div2, div3, div4,
floorMin]`, caps in whole INIT): each band earns 1 Ballot per `divN` INIT of
the power that falls inside it.

```
weight = Σ over bands i = 1..4 of  (power inside band i, in uinit) / div_i
         band 1: 0 … cap1·10⁶, band 2: … cap2·10⁶, band 3: … cap3·10⁶, band 4: the rest

atRate = ⌊weight / 10⁶⌋
atPool = ⌊weight × pool / eligibleWeight⌋
base   = min(atRate, atPool)
if power ≥ floorMin·10⁶ and base < floor:  base = floor
ballots = min(base, pool − minted so far this cycle)
```

- `eligibleWeight` is the reporter's sum of every voter's weight for the
  cycle. The cycle's **first** settle locks it (`BALLOT_ELIGIBLE_WEIGHT`);
  every later settle must pass the same value (`"Ballot: weight mismatch"`),
  and the weight settled so far may never exceed it (`"Ballot: weight over
  eligible"`; `"Ballot: no eligible weight"` for 0). This bounds the trusted
  reporter: base payouts stay within the pool.
- When the total weight would pay more than the pool, `atPool` is the smaller
  term and a busy cycle pays out exactly its pool split by weight.
- The floor is paid on top of the split, so the pool cap (`BALLOT_CYCLE_MINTED`)
  cuts floors once the pool is spent: a cycle never mints more than `pool`.
- Settles revert `"Ballot: rates not configured"` (pool 0) or `"Ballot: tiers
  not configured"` (any divisor 0).

Deploy data: `BALLOT_TIERS = [5000, 50000, 150000, 30, 60, 90, 120, 10]` — 1
Ballot per 30 INIT up to 5,000 INIT, per 60 up to 50,000, per 90 up to
150,000, per 120 above; the floor applies from 10 INIT. Example: 1 INIT weighs
33,333 millionths; 5,000 INIT weigh 166,666,666 (166.7 Ballots at rate).

The owner-only `BallotSettleSystem.resetLock(uint64 cycle)` clears a cycle's
locked eligible weight so the reporter can settle it again after the lock
stopped matching (e.g. tiers changed after the first settle); the next settle
locks it again, and the settled weight still bounds it.

> Source: `LibBallot.sol:117–132, 154–190, 245–269`,
> `BallotSettleSystem.sol:53–57`, `configs.ts:259–268`,
> `Ballot.t.sol:167–175`

### Streaks and milestones

A settle builds the streak only when all three hold:

- `streakVote` is true — the reporter judged the vote a streak vote (at least
  `streakShare` basis points of the wallet's vote on Yominet; judged off chain)
- the cycle is **in order**: later than the account's `BALLOT_LAST_CYCLE`
  (which every in-order settle advances) — a late settle of an earlier cycle
  pays Ballots but no streak
- `yominetPower ≥ max(streakPower, floorMin)` INIT

Otherwise the cycle's streak is 0 (a miss). The streak as of a cycle
(`calcStreak`, stored as `BALLOT_STREAK` for that cycle):

```
if streak(cycle − 1) > 0:              streak = streak(cycle − 1) + 1
elif cycle < 3:                         streak = 1
elif streak(cycle − 2) == 0:            streak = 1
elif grace used and cycle < graceCycle + graceRefill:   streak = 1
else:  graceCycle = cycle − 1;          streak = streak(cycle − 2) + 1
```

One missed cycle can be **bridged by grace**, if grace was never used or was
last used at least `graceRefill` cycles ago; the bridged cycle is remembered in
`BALLOT_GRACE_CYCLE`.

Milestones, owed on a streak settle:

- **Golden Ballot** when `streak % goldenEvery == 0`, if the power is at least
  `goldenPower` INIT and the cycle's golden count is under `goldenCap` (both
  levers are off at 0)
- **Voter Badges I–IV** at streaks 5 / 10 / 26 / 52, each **once per account
  ever**: only when the streak passes the account's best streak
  (`BALLOT_STREAK_BEST`), and only for steps above the old best

`BALLOT_RATES = [pool, graceRefill, floor, goldenEvery, streakShare,
streakPower, goldenPower, goldenCap]`. Deploy data: `[10000, 13, 1, 5, 7500,
100, 0, 0]` — a pool of 10,000 Ballots per cycle, grace refills after 13
cycles, a floor of 1 Ballot, a Golden Ballot every 5 streak cycles, streak
votes need 75 % on Yominet and at least 100 INIT, golden levers off.

> Source: `LibBallot.sol:134–152, 223–243, 276–314`, `configs.ts:249–258`

## Claiming

`BallotClaimSystem.executeTyped()` (no arguments), signed by the account's
**owner or operator** (`LibAccount.getByOwnerOrOperator`):

1. Mint the owed Ballots (`BALLOT_OWED` at index 0) and zero the balance
2. Mint each owed milestone item (badges 15–18, Golden Ballot 14) and zero it
3. Revert `"Ballot: nothing to claim"` if nothing was owed
4. Emit `BALLOT_CLAIM` `(accID, ballots, golden, badges)`, update the
   account's last-action time, return the Ballots minted

Owed balances accumulate across cycles until claimed.

> Source: `BallotClaimSystem.sol:17–29`, `LibBallot.sol:192–221, 341–354`

## Spending: the Ballot Shop

NPC **Alaina** (NPC 10, room 64) sells two listings priced in Ballots, both
`FIXED`:

| Listing | Item | Currency | Price |
|---|---|---|---|
| Ballot Shop Booster Pack | 21004 Booster Pack | 13 Ballot | 5 |
| Ballot Shop Golden Box | 21008 Golden Box | 14 Golden Ballot | 1 |

The Golden Box (Epic lootbox, account item, `NOT_TRADABLE`) gives 5 Booster
Packs (allo `5x Booster Pack`) and one roll on `DT Golden Box`; the Booster
Pack itself became `NOT_TRADABLE` in the same commit. Neither listing has a
sell price: the listing deploy now refuses a sell price on a `NOT_TRADABLE`
item, since selling to a shop skips the tradability check. See
[`catalogs/npcs/listings.csv`](../../catalogs/npcs/listings.csv) and
[`catalogs/items/droptables.csv`](../../catalogs/items/droptables.csv).

> Source: `deployment/world/data/listings/listings.csv` (last two rows),
> `deployment/world/data/npc/npc.csv`, `deployment/world/state/listings.ts:42–47`,
> `deployment/world/data/items/items.csv` (rows 21004, 21008),
> `deployment/world/data/items/allos.csv` (`5x Booster Pack`, `DT Golden Box`)

## Data Keys

All state is `LibData` values; holder = the account unless noted.

| Key (index) | Meaning |
|---|---|
| `BALLOT_SETTLED` (cycle) | 1 once the account's cycle is settled |
| `BALLOT_POWER` (cycle) | Yominet power the cycle was settled with |
| `BALLOT_STREAK` (cycle) | Streak as of that cycle (streak votes only; 0 = none) |
| `BALLOT_STREAK_BEST` (0) | Best streak settled; badges up to it have been owed |
| `BALLOT_GRACE_CYCLE` (0) | Missed cycle last bridged by grace (0 = never) |
| `BALLOT_LAST_CYCLE` (0) | Latest cycle settled in order |
| `BALLOT_OWED` (0) | Ballots settled, not yet claimed |
| `BALLOT_OWED` (item index) | Badges / Golden Ballots settled, not yet claimed |
| `BALLOT_VOTERS` (cycle), holder 0 | Accounts settled in the cycle |
| `BALLOT_ELIGIBLE_WEIGHT` (cycle), holder 0 | Locked eligible weight |
| `BALLOT_CYCLE_WEIGHT` (cycle), holder 0 | Weight settled so far |
| `BALLOT_CYCLE_MINTED` (cycle), holder 0 | Ballots settled so far (≤ pool) |
| `BALLOT_CYCLE_GOLDEN` (cycle), holder 0 | Golden Ballots owed in the cycle |

> Source: `LibBallot.sol:37–53`

## Config

| Key | Deploy value | Description |
|---|---|---|
| `BALLOT_REPORTER` | address from the deploy environment (zero if unset) | The only address that may settle |
| `BALLOT_RATES` | `[10000, 13, 1, 5, 7500, 100, 0, 0]` | Pool, grace refill, floor, golden every, streak share (bps), streak power (INIT), golden power (INIT), golden cap |
| `BALLOT_TIERS` | `[5000, 50000, 150000, 30, 60, 90, 120, 10]` | Band caps (INIT), INIT per Ballot per band, floor minimum (INIT) |
| `BALLOT_CYCLE` | `[1793260800, 1209600, 40, 0, 0, 0, 0, 0]` | Start, length, base cycle |

The ballot configs are written by their own world-state command (`configs
initBallots`), not by the general config init.

> Source: `configs.ts:242–273`, `deployment/world/world.ts:131`
