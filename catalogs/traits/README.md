# Kami Traits Catalog

> Source: `packages/contracts/deployment/world/data/traits/`
> Commit: `d9b50091`

## Files

| File | Entries | Description |
|---|---|---|
| `bodies.csv` | 30 | Body shapes with affinity and stats |
| `faces.csv` | 36 | Face expressions with affinity and stats |
| `hands.csv` | 27 | Hand types with affinity and stats |
| `backgrounds.csv` | 28 | Background visuals with stats |
| `colors.csv` | 14 | Color palettes with stats |

**Total trait variants: 135**

## How Traits Work

Every Kami has exactly 5 trait slots assigned at creation via weighted random
selection. Traits determine:
- **Base stat modifiers** (Health, Power, Violence, Harmony)
- **Equipment slots** (some traits grant extra slots)
- **Affinity** (body/face/hand — affects combat and harvesting effectiveness)
- **Visual appearance** (the Kami's on-chain image is composed from trait layers)

See [mechanics/gacha/kami-creation.md](../../mechanics/gacha/kami-creation.md)
for the creation system and selection algorithm.

## Common Schema

All trait files share these columns:

| Column | Type | Description |
|---|---|---|
| Index | uint32 | Unique trait ID within its type |
| Name | string | Display name |
| Rarity | enum | Common / Uncommon / Rare / Epic / Legendary |
| Tier | uint32 | Gacha weight (9=common, 4=legendary) — `P = 2^(tier-1)` in selection |
| Tuning | uint32 | Balance weight for stat distribution |
| Health | uint32 | Health stat modifier |
| Power | uint32 | Power stat modifier |
| Violence | uint32 | Violence stat modifier |
| Harmony | uint32 | Harmony stat modifier |
| BPs | uint32 | Blueprint cost (used in trait marketplace) |

Additional columns per type:
- **Bodies/Faces/Hands**: `Affinity` (Normal/Eerie/Insect/Scrap), `Slots` (extra equipment slots)
- **Faces**: `Gated` (Yes/No — access-restricted)
- **Backgrounds**: `Hex` (color code for solid backgrounds)
- **Colors**: `Hex` (color code)

## Rarity Distribution

| Rarity | Tier | Bodies | Faces | Hands | Backgrounds | Colors | Total |
|---|---|---|---|---|---|---|---|
| Common | 9 | 10 | 12 | 8 | 12 | 6 | 48 |
| Uncommon | 8 | 4 | 5 | 4 | 0 | 4 | 17 |
| Rare | 7 | 8 | 7 | 6 | 4 | 0 | 25 |
| Epic | 6 | 5 | 10 | 4 | 6 | 4 | 29 |
| Legendary | 4 | 4 | 2 | 4 | 7 | 0 | 17 |

## Affinity Distribution (Bodies + Faces + Hands only)

| Affinity | Bodies | Faces | Hands | Total |
|---|---|---|---|---|
| Normal | 10 | 21 | 10 | 41 |
| Eerie | 6 | 5 | 5 | 16 |
| Insect | 7 | 4 | 6 | 17 |
| Scrap | 7 | 4 | 6 | 17 |
| (none) | 0 | 2 | 0 | 2 |

## Notable Traits

### Legendary Bodies
- **Snake** (Insect) — balanced 3/3/3 Power/Violence/Harmony
- **Tank** (Scrap) — 30 Health, 6 Harmony
- **Hagoromo** (Eerie) — 9 Power (highest single stat)
- **Suit** (Normal) — 6 Violence, 3 Harmony

### Legendary Faces
- **Lenny 1** — 2/2/2 + 1 Slot, gated
- **Sunglasses** — 40 Health, 4 Power

### Legendary Hands
- **Wings** (Normal) — 90 Health (highest in game)
- **Mudra** (Eerie) — 30 Health, 6 Power
- **Van de Graaf** (Scrap) — 30 Health, 6 Harmony
- **Fairy** (Insect) — 30 Health, 3 Power, 3 Harmony

### Highest Stat Values
- **Health**: Wings (90), No Kami body (70), Third Eye face (60)
- **Power**: Hagoromo body (9), Candles hand (7), Graveyard BG (7)
- **Violence**: Butterfly body (7), Mantis hand (7), Butterfly BG (7)
- **Harmony**: Tank body (6), Van de Graaf hand (6), Sensor face (6)
- **Slots**: Octahedron body (2), Lenny 1 face (1), Cube body (1)
