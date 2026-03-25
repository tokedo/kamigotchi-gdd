# Math & Random Libraries

> Source: `packages/contracts/src/utils/FixedPointMathLib.sol` (L1–378),
> `packages/contracts/src/utils/Gaussian.sol` (L1–237),
> `packages/contracts/src/utils/Units.sol` (L1–42),
> `packages/contracts/src/libraries/utils/LibRandom.sol` (L1–335)

## Overview

These libraries provide the mathematical foundation used throughout Kamigotchi:
fixed-point arithmetic for price calculations, Gaussian distribution functions
for stat generation, and random selection utilities for loot, gacha, and trait
assignment.

## Fixed-Point Math (FixedPointMathLib)

A fork of Solmate's `FixedPointMathLib` providing WAD-precision (18 decimals)
arithmetic. Used by the GDA auction system, Gaussian library, and other
price/value calculations.

### Constants

| Name | Value | Description |
|---|---|---|
| `WAD` | `1e18` | Standard ERC-20 / ETH scalar |

### WAD Operations

| Function | Formula | Description |
|---|---|---|
| `mulWadDown(x, y)` | `(x × y) / WAD` (round down) | Multiply two WAD values |
| `mulWadUp(x, y)` | `(x × y) / WAD` (round up) | Multiply two WAD values, round up |
| `divWadDown(x, y)` | `(x × WAD) / y` (round down) | Divide producing WAD result |
| `divWadUp(x, y)` | `(x × WAD) / y` (round up) | Divide producing WAD result, round up |

### Exponential & Logarithm

| Function | Description | Used By |
|---|---|---|
| `expWad(x)` | `e^x` in WAD precision, range (-42, 136) | GDA pricing, Gaussian |
| `lnWad(x)` | `ln(x)` in WAD precision, x > 0 | GDA pricing, Gaussian |
| `powWad(x, y)` | `x^y = e^(ln(x) × y)` in WAD | GDA pricing |

### Low-Level Operations

| Function | Description |
|---|---|
| `mulDivDown(x, y, d)` | `(x × y) / d` rounded down (assembly, overflow-safe) |
| `mulDivUp(x, y, d)` | `(x × y) / d` rounded up |
| `rpow(x, n, scalar)` | `x^n` with arbitrary scalar (binary exponentiation) |
| `sqrt(x)` | Integer square root (Babylonian method) |
| `log2(x)` | Integer log base 2 |

> Source: `FixedPointMathLib.sol:6–378`

## Gaussian Distribution (Gaussian)

A Solidity implementation of the standard normal distribution (mean=0,
variance=1) using the complementary error function. Used for generating
normally-distributed random values in stat calculations.

### Functions

| Function | Math | Description |
|---|---|---|
| `cdf(x)` | `D(x) = 0.5[1 + erf(x / √2)]` | Cumulative distribution function |
| `pdf(x)` | `Z(x) = (1/√2π)e^(-x²/2)` | Probability density function |
| `ppf(x)` | `D⁻¹(x) = -√2 × ierfc(2x)` | Percent point (inverse CDF) |
| `erfc(x)` | Complementary error function | Identity: `erfc(-x) = 2 - erfc(x)` |
| `ierfc(x)` | Inverse complementary error function | Domain: `0 < x < 2` |

### Precision

- Maximum error vs theoretical: `1.2e-7`
- Maximum error vs Gaussian.js: `1e-15`
- `erfc` domain: approximately `(-6.24, 6.24)` WAD — returns 2 or 0 beyond

### Constants

| Name | Value | Description |
|---|---|---|
| `PI` | `3.141592653589793238` | Pi in WAD |
| `SQRT2` | `1.414213562373095048` | √2 in WAD |
| `SQRT_2PI` | `2.506628274631000502` | √(2π) in WAD |

> Source: `Gaussian.sol:25–237`

## Signed Fixed-Point Helpers (Units.sol)

Free functions for signed integer WAD arithmetic:

| Function | Description |
|---|---|
| `abs(x)` | Absolute value of int256 |
| `muli(x, y, d)` | Signed `(x × y) / d` with overflow check |
| `muliWad(x, y)` | Signed `(x × y) / 1e18` |
| `diviWad(x, y)` | Signed `(x × 1e18) / y` |

> Source: `Units.sol:1–42`

## Random Selection (LibRandom)

Comprehensive randomness library used for loot rolls, gacha selection, trait
assignment, and droptable resolution.

### Rarity Weights

```
weight = rarity == 0 ? 0 : 2^(rarity - 1)
```

Rarity values map to exponentially increasing weights:
- Rarity 0 → weight 0 (cannot be selected)
- Rarity 1 → weight 1
- Rarity 2 → weight 2
- Rarity 3 → weight 4
- Rarity 4 → weight 8
- etc.

> Source: `LibRandom.sol:32–34`

### Unweighted Selection

| Function | Description |
|---|---|
| `getRandom(seed, max)` | Single random index: `seed % max` |
| `getRandomBatch(seed, max, count)` | Batch with replacement |
| `getRandomBatchNoReplacement(seed, max, count)` | Batch without replacement (decrements max) |
| `selectFrom(keys, seed)` | Pick one key from uniform array |
| `selectMultipleFrom(keys, seed, count)` | Pick multiple with replacement |
| `selectMultipleFromNoReplacement(keys, seed, count)` | Pick multiple, swap-and-shrink |

All seed derivation uses: `newSeed = keccak256(abi.encode(seed, i))`

No-replacement selection uses a **virtual swap** pattern: after selecting index
`pos`, the item at `pos` is replaced with the last item, and `max` is
decremented. This ensures unique draws.

> Source: `LibRandom.sol:40–156`

### Weighted Selection

| Function | Description |
|---|---|
| `selectFromWeighted(keys, weights, seed)` | Single weighted pick |
| `selectMultipleFromWeighted(weights, seed, count)` | Multiple weighted picks (returns count array) |
| `pSelectFromWeighted(...)` | Packed bitfield variant |

Algorithm:
1. Sum all weights → `totalWeight`
2. Roll: `roll = seed % totalWeight`
3. Iterate cumulative weights until `roll < cumulativeWeight`
4. Return the key at that position

> Source: `LibRandom.sol:168–284`

### Packed Array Operations

For gas-efficient bitfield-packed arrays (used in trait/item systems):

| Function | Description |
|---|---|
| `pGenerateFromSeed(seed, pMax, numElements, SIZE)` | Generate random packed array from seed |
| `pTotalWeight(packed, SIZE)` | Sum weights from packed array |

> Source: `LibRandom.sol:294–334`

### Randomness Source

All randomness in Kamigotchi derives from **blockhash-based seeds** via the
commit-reveal pattern (`LibCommit`). `LibRandom` itself is deterministic — it
only transforms seeds into selections. The actual entropy comes from:

```
seed = keccak256(blockhash(revealBlock), entityID)
```
