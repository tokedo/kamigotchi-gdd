# Kami Creation (Entity + Traits + Stats)

> Source: `packages/contracts/src/libraries/LibKamiCreate.sol` (L1–164),
> `packages/contracts/src/systems/_721BatchMinterSystem.sol` (L1–397)

## Overview

Kami creation is the process of generating a new Kami entity with randomized
traits and derived stats. There are two creation paths:

1. **Runtime creation** (`LibKamiCreate.create`) — used by the gacha mint system
   during normal gameplay
2. **Batch minting** (`_721BatchMinterSystem.batchMint`) — admin-only, used to
   seed the initial gacha pool

Both paths produce the same Kami entity shape, but batch minting uses a
pre-computed seed while runtime creation uses `blockhash(block.number - 1)`.

## Kami Entity Shape

| Component | Initial Value | Description |
|---|---|---|
| `EntityType` | `"KAMI"` | Entity type identifier |
| `IDOwnsKami` | `GACHA_ID` | Owned by gacha pool initially |
| `IndexKami` | Sequential index | ERC-721 token index |
| `Name` | `"{BASE_KAMI_NAME}{index}"` | Default name (e.g., "Kamigotchi 42") |
| `State` | `"RESTING"` | Initial state |
| `Level` | `1` | Starting level |
| `Experience` | `0` | Starting XP |
| `SkillPoints` | `1` | Starting skill points |
| `IndexFace` | Random | Face trait index |
| `IndexHand` | Random | Hand trait index |
| `IndexBody` | Random | Body trait index |
| `IndexBackground` | Random | Background trait index |
| `IndexColor` | Random | Color trait index |
| `Health` | `50 + trait deltas` | Base health stat |
| `Power` | `10 + trait deltas` | Base power stat |
| `Violence` | `10 + trait deltas` | Base violence stat |
| `Harmony` | `10 + trait deltas` | Base harmony stat |
| `Slots` | `0 + trait deltas` | Equipment slot count |
| `MediaURI` | Packed trait indices | Image identifier |

> Source: `LibKamiCreate.sol:43–71, 96–135`

## Entity ID

Kami entity IDs are deterministic based on the token index:

```
kamiID = LibKami.genID(index)
```

The index is derived from the current ERC-721 total supply + 1:
```
nextIndex = uint32(Kami721.totalSupply()) + 1
```

Token indices start at 1 (not 0).

> Source: `LibKamiCreate.sol:154–156`

## Trait Assignment

Each Kami has 5 trait slots, each assigned via **weighted random selection**:

| Slot | Trait Type | Component |
|---|---|---|
| 0 | FACE | `IndexFaceComponent` |
| 1 | HAND | `IndexHandComponent` |
| 2 | BODY | `IndexBodyComponent` |
| 3 | BACKGROUND | `IndexBackgroundComponent` |
| 4 | COLOR | `IndexColorComponent` |

### Randomness

For runtime creation:
```
seed = keccak256(blockhash(block.number - 1), kamiID)
traitSeed[i] = keccak256(seed, i)  // per-slot seed
```

### Weighted Selection

Each trait variant in the registry has a **rarity** value. The weight is:
```
weight = rarity > 0 ? 2^(rarity - 1) : 0
```

Traits with rarity 0 have zero weight (cannot be selected). Higher rarity values
produce exponentially higher weights (more common).

`LibRandom.selectFromWeighted(keys, weights, seed)` performs the selection.

> Source: `LibKamiCreate.sol:106–149`, `_721BatchMinterSystem.sol:200–231`

## Stat Calculation

Stats are computed from a **base** plus **trait deltas**:

```
Base stats:
  health  = 50
  power   = 10
  violence = 10
  harmony  = 10
  slots    = 0

Final stat = base + sum(traitDelta[i] for each trait slot)
```

Each trait variant in the registry can have stat modifiers stored on its entity
(via `HealthComponent`, `PowerComponent`, `ViolenceComponent`,
`HarmonyComponent`, `SlotsComponent`). The delta for each slot is looked up from
the trait registry.

Health is initialized as a Stat with `base = current = finalHealth`:
```
Health: Stat(finalHealth, 0, 0, finalHealth)
Slots:  Stat(finalSlots, 0, 0, finalSlots)
Others: Stat(value, 0, 0, 0)
```

> Source: `LibKamiCreate.sol:116–130`, `_721BatchMinterSystem.sol:145–165`

## Media URI

The media URI is a packed integer encoding of all 5 trait indices:

```
mediaURI = packArr([faceIdx, handIdx, bodyIdx, bgIdx, colorIdx], 8)
```

Each trait index occupies 8 bits. The full image URL is:
```
https://{BASE_URI}/{mediaURI}.gif
```

> Source: `LibKamiCreate.sol:132–135`, `LibKami721.sol:68–73`

## Batch Minter (Admin Seeding)

`_721BatchMinterSystem.batchMint(amount)` — owner-only:

1. **Create** Kami entities in `RESTING` state, owned by `GACHA_ID`
2. **Reveal** traits using a deterministic seed:
   `baseSeed = keccak256(blockhash(deployBlock))` combined with entity IDs
3. **Mint** ERC-721 tokens to the Kami721 contract address (staked in-game)

The batch minter uses a one-time `setTraits()` call to memoize all trait
weights, stats, and offsets from the trait registry for gas efficiency.

> Source: `_721BatchMinterSystem.sol:312–328, 330–333`

## Config

| Key | Value | Description |
|---|---|---|
| `BASE_KAMI_NAME` | `"Kamigotchi "` | Default name prefix for new Kamis |
| `BASE_URI` | `"i.test.kamigotchi.io/kami"` | Base URL for Kami images |

> Source: `configs.ts:62–63`
