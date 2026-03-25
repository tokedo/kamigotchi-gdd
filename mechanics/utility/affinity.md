# Affinity System

> Source: `packages/contracts/src/libraries/utils/LibAffinity.sol` (L1–91)

## Overview

The affinity system implements a **rock-paper-scissors** type effectiveness
model. Each Kami has two affinities (body and hand), which affect combat damage
and harvest efficiency. There are 4 affinity types: **EERIE**, **SCRAP**,
**INSECT**, and **NORMAL**.

## Affinity Types

| Affinity | Strong Against | Weak Against |
|---|---|---|
| EERIE | SCRAP | INSECT |
| SCRAP | INSECT | EERIE |
| INSECT | EERIE | SCRAP |
| NORMAL | — | — |

NORMAL is neutral against all types, except NORMAL vs NORMAL which is a
**Special** interaction.

> Source: `LibAffinity.sol:62–79`

## Effectiveness Levels

```solidity
enum Effectiveness { Strong, Neutral, Weak, Special }
```

Each effectiveness level maps to a shift value loaded from config:

```solidity
struct Shifts { base, up, down, special }  // loaded from config array
```

| Config Index | Field | Description |
|---|---|---|
| 1 | `base` | Neutral effectiveness shift |
| 2 | `up` | Strong effectiveness bonus |
| 3 | `down` | Weak effectiveness penalty (stored positive, negated in code) |
| 4 | `special` | Special interaction shift (NORMAL vs NORMAL) |

The final shift combines config + bonus:
```
shift = configShift[effectiveness] + bonusShift[effectiveness]
```

> Source: `LibAffinity.sol:34–58`

## Attack Effectiveness

Used by the combat/kill system:

```
EERIE → strong vs SCRAP, weak vs INSECT
SCRAP → strong vs INSECT, weak vs EERIE
INSECT → strong vs EERIE, weak vs SCRAP
NORMAL vs NORMAL → Special
All other combinations → Neutral
```

> Source: `LibAffinity.sol:61–79`

## Harvest Effectiveness

Used by the harvesting system (Kami affinity vs node affinity):

```
Same affinity → Strong (bonus yield)
Different non-NORMAL affinities → Weak (penalty)
Either is NORMAL or empty → Neutral
```

> Source: `LibAffinity.sol:82–90`

## Kami Affinities

Each Kami has two affinities derived from its traits:
- **Body affinity** — from the body trait
- **Hand affinity** — from the hand trait

If both match, the Kami's **breed** is `PURE`; otherwise `MIXED`.
