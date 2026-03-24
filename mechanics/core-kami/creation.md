# Kami Creation

> Source: `packages/contracts/src/libraries/LibKamiCreate.sol` (L72–164),
> `packages/contracts/src/libraries/LibTraitRegistry.sol` (L55–339),
> `packages/contracts/deployment/world/state/configs/configs.ts`

## Overview

A Kami is the core entity in Kamigotchi — a virtual pet that lives on-chain. Kami
creation produces a new ECS entity with traits, stats, an ERC-721 NFT, and places
it in the Gacha pool awaiting claim.

## Creation Flow

`LibKamiCreate.create()` performs these steps in order:

1. **Assign index** — `nextIndex = Kami721.totalSupply() + 1`
2. **Generate entity ID** — deterministic: `id = keccak256("kami", index)`
3. **Verify uniqueness** — reverts if entity type `KAMI` already exists for this ID
4. **Set base properties** (see below)
5. **Roll traits** (5 trait slots, weighted random)
6. **Compute and set stats** (base stats + trait stat deltas)
7. **Set media URI** (packed trait indices)
8. **Mint ERC-721** — `Kami721.mint(nftContractAddress, index)`

> Source: `LibKamiCreate.sol:75–86`

## Base Properties

On creation every Kami receives:

| Property | Initial Value | Source |
|---|---|---|
| Owner | `GACHA_ID` (sits in gacha pool) | `LibKamiCreate.sol:97` |
| Name | `"{BASE_KAMI_NAME}{index}"` (config: `"Kamigotchi "`) | `LibKamiCreate.sol:99`, `configs.ts:62` |
| State | `"RESTING"` | `LibKamiCreate.sol:100` |
| Level | `1` | `LibKamiCreate.sol:101` |
| Skill Points | `1` | `LibKamiCreate.sol:102` |
| Experience | `0` | `LibKamiCreate.sol:103` |

Possible Kami states: `RESTING`, `HARVESTING`, `DEAD`, `721_EXTERNAL`

> Source: `LibKamiCreate.sol:50–51, 96–104`

## Trait System

### Trait Types

Every Kami has exactly **5 traits**, rolled in this order:

| Index | Trait Type | Component |
|---|---|---|
| 0 | `FACE` | `IndexFaceComponent` |
| 1 | `HAND` | `IndexHandComponent` |
| 2 | `BODY` | `IndexBodyComponent` |
| 3 | `BACKGROUND` | `IndexBackgroundComponent` |
| 4 | `COLOR` | `IndexColorComponent` |

> Source: `LibTraitRegistry.sol:320–327`

### Trait Rolling

Each trait is selected via **weighted random selection** from the trait registry:

```
seed = keccak256(blockhash(block.number - 1), kamiEntityID)
for each trait type i in [FACE, HAND, BODY, BACKGROUND, COLOR]:
    traitSeed = keccak256(seed, i)
    trait[i] = weightedRandomSelect(registeredTraits[type], traitSeed)
```

The weight for each registered trait comes from its `RarityComponent` value.
Higher rarity weight = more likely to be selected (weights are processed through
`LibRandom.processWeightedRarity()`).

> Source: `LibKamiCreate.sol:106–114, 137–149`, `LibTraitRegistry.sol:246–253`

### Trait Properties

Each registered trait has:

| Field | Type | Description |
|---|---|---|
| `name` | string | Display name |
| `health` | int32 | Stat delta applied to base health |
| `power` | int32 | Stat delta applied to base power |
| `violence` | int32 | Stat delta applied to base violence |
| `harmony` | int32 | Stat delta applied to base harmony |
| `slots` | int32 | Stat delta applied to base slots |
| `rarity` | uint256 | Weight for random selection (higher = more common) |
| `affinity` | string | Affinity tag (used in harvesting efficacy) |

> Source: `LibTraitRegistry.sol:36–45`

### Trait Catalogs

The trait data is registered from CSVs at deployment:

| CSV | Entries |
|---|---|
| `data/traits/bodies.csv` | 29 body types |
| `data/traits/faces.csv` | 35 face types |
| `data/traits/hands.csv` | 26 hand types |
| `data/traits/colors.csv` | 13 color types |
| `data/traits/backgrounds.csv` | 27 background types |

## Initial Stats

Stats are computed as: **hardcoded base + sum of all trait stat deltas**.

### Base Stats (hardcoded in `LibKamiCreate.setStats`)

| Stat | Base Value |
|---|---|
| Health | 50 |
| Power | 10 |
| Violence | 10 |
| Harmony | 10 |
| Slots | 0 |

> Source: `LibKamiCreate.sol:117`

### Stat Computation

```
for each trait in [FACE, HAND, BODY, BACKGROUND, COLOR]:
    traitStats = TraitRegistry.getStatsByIndex(traitIndex, traitType)
    base.health  += traitStats.health
    base.power   += traitStats.power
    base.violence += traitStats.violence
    base.harmony += traitStats.harmony
    base.slots   += traitStats.slots
```

> Source: `LibKamiCreate.sol:116–130`

### Initial Stat Component Values

Stats are stored as `Stat` structs (see [stats.md](stats.md) for full stat system):

| Stat | base | shift | boost | sync |
|---|---|---|---|---|
| Health | computed | 0 | 0 | **= base** (starts full) |
| Power | computed | 0 | 0 | 0 |
| Violence | computed | 0 | 0 | 0 |
| Harmony | computed | 0 | 0 | 0 |
| Slots | computed | 0 | 0 | **= base** (starts full) |

Health and Slots are "depletable" — their `sync` value starts at max (= base).

> Source: `LibKamiCreate.sol:125–129`

## Media URI

The media URI is a packed integer encoding all 5 trait indices, each packed into
8 bits:

```
mediaURI = toString(pack([faceIdx, handIdx, bodyIdx, bgIdx, colorIdx], 8))
```

> Source: `LibKamiCreate.sol:132–135`

## Batch Creation

`create(components, amount)` creates multiple Kamis in a single transaction by
calling `create()` in a loop.

> Source: `LibKamiCreate.sol:88–91`
