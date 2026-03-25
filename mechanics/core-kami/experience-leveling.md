# Experience & Leveling

> Source: `packages/contracts/src/libraries/LibExperience.sol` (L1–109),
> `packages/contracts/src/systems/KamiLevelSystem.sol` (L1–51),
> `packages/contracts/deployment/world/state/configs/configs.ts` (L84–87)

## Overview

Kamis earn experience (XP) through activities and spend it to level up. Leveling
up consumes XP and grants 1 skill point. The XP cost per level follows an
exponential curve.

> **Note on two XP pools**: The `LibExperience` library is entity-agnostic — it
> operates on any entity ID. The game uses it for **two separate XP pools**:
>
> - **Kami XP** — earned per individual Kami, used to level up (this file)
> - **Account XP** — earned per account (operator), from movement and crafting
>   (see [accounts.md](../world/accounts.md))
>
> Only Kami XP feeds into the level-up system. There is no account-level
> level-up mechanism.

## Kami XP Sources

Kami XP is awarded directly to the Kami entity:

- **Harvesting** — XP equal to harvest output amount, awarded on stop or collect
  (`HarvestStopSystem.sol:97`, `HarvestCollectSystem.sol:95`)
- **Kill salvage** — victim Kami receives XP equal to the salvage amount
  (`LibKill.sol:53`)
- **Item effects on Kami** — items used on a Kami (via `KamiUseItemSystem`) can
  award XP if the item has an XP-type allocation (`LibSetter.sol:48-49`)

Account XP (movement, crafting) is a **separate pool** on the account entity and
does **not** contribute to Kami leveling. See [accounts.md](../world/accounts.md)
for details.

## Level-Up Cost Formula

The XP required to advance from the current level is:

```
cost = base × multiplier^(level - 1)
```

Where:
- `base` = config `KAMI_LVL_REQ_BASE` = **40**
- `multiplier` = config `KAMI_LVL_REQ_MULT_BASE` = **[1259, 3]**
  → value = 1259, precision = 10^3 = 1000
  → multiplier = 1259 / 1000 = **1.259**

The exponentiation uses **WAD math** (18-decimal fixed-point) via `powWad`:

```solidity
multiplierBaseFormatted = (1e18 × 1259) / 1000  // = 1.259 × 10^18
multiplier = powWad(multiplierBaseFormatted, (level - 1) × 1e18)
cost = (base × multiplier) / 1e18
```

> Source: `LibExperience.sol:46–60`, `configs.ts:84–87`

### XP Cost Table (first 10 levels)

| Level | XP to next level | Cumulative XP |
|---|---|---|
| 1 | 40 | 40 |
| 2 | 50 | 90 |
| 3 | 63 | 153 |
| 4 | 80 | 233 |
| 5 | 100 | 333 |
| 6 | 126 | 459 |
| 7 | 159 | 618 |
| 8 | 200 | 818 |
| 9 | 252 | 1,070 |
| 10 | 317 | 1,387 |

> Computed from: `cost(L) = floor(40 × 1.259^(L-1))`

## Level-Up Process

`KamiLevelSystem.execute(kamiID)`:

1. **Verify ownership** — caller must own the Kami via their account
2. **Verify state** — Kami must be in `"RESTING"` state
3. **Check XP** — `currentXP >= calcLevelCost(kamiID)`, else reverts
4. **Sync health** — updates Kami's current HP based on time-based healing
5. **Consume XP** — `experience -= levelCost`
6. **Increment level** — `level += 1`
7. **Grant skill point** — `skillPoints += 1`
8. **Emit NFT metadata update** event
9. **Log** — increments `KAMI_LEVELS_TOTAL` counter on the account

> Source: `KamiLevelSystem.sol:19–46`

## Key Design Notes

- **XP is consumed, not cumulative** — leveling spends XP. A Kami that levels up
  retains only the surplus XP beyond the cost.
- **Level defaults to 1** — any entity without a Level component returns level 1.
- **Experience defaults to 0** — any entity without an Experience component
  returns 0.
- **One level at a time** — there is no batch level-up. Each call advances
  exactly 1 level.
- **Must be resting** — Kamis that are harvesting, dead, or staked externally
  cannot level up.
- **Skill point per level** — every level grants exactly 1 SP for the skill tree
  (see skill system).

> Source: `LibExperience.sol:97–101`, `KamiLevelSystem.sol:29`
