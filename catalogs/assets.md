# Asset Reference Catalog

Maps game entities to their image asset paths in the source repository.
**Pointers only** — no image copies are stored in this GDD.

> Source: `packages/client/src/assets/images/` and on-chain metadata
> Commit: `d9b50091`

---

## Table of Contents

- [Kami Character Images (On-Chain CDN)](#kami-character-images-on-chain-cdn)
- [Room Assets](#room-assets)
- [Item Images](#item-images)
- [Skill Images](#skill-images)
- [NPC Images](#npc-images)
- [Icon Sets](#icon-sets)
- [Map Zone Images](#map-zone-images)
- [Token & UI Assets](#token--ui-assets)
- [Name-to-File Mapping](#name-to-file-mapping)

---

## Kami Character Images (On-Chain CDN)

Kami images are **not bundled in the client**. They are served from a CDN and
referenced on-chain via `MediaURIComponent`.

### URI Construction

| Layer | Source | Pattern |
|---|---|---|
| On-chain | `LibKami721.sol:68-72` | `https://{BASE_URI}/{imageID}.gif` |
| Client | `constants/media.ts:2` | `KAMI_BASE_URI + traits + '.gif'` |
| Config | `LibConfig.getString(comps, "BASE_URI")` | Deployment-configurable |

**Current CDN base URL** (test):
```
https://i.test.kamigotchi.io/kami/
```

### How `imageID` is generated

When a Kami is created, its 5 trait indices (Body, Color, Face, Hand, Background)
are **bit-packed** into a single uint and stored as a string:

```solidity
// LibKamiCreate.sol:132-134
function setURI(IUintComp comps, uint256 id, uint32[] memory traits) internal {
    string memory image = LibString.toString(LibPack.packArr(traits, 8));
    MediaURIComponent(...).set(id, image);
}
```

The resulting `imageID` is a decimal string of the packed value. The CDN maps
this to a pre-rendered GIF of the Kami's trait combination.

### NFT Metadata (ERC-721)

`LibKami721.getJsonUtf()` (lines 92-116) produces:

```json
{
  "external_url": "https://kamigotchi.io",
  "name": "<kami name>",
  "description": "a lil network spirit :3",
  "attributes": [
    {"trait_type": "Body", "value": "<name>"},
    {"trait_type": "Color", "value": "<name>"},
    {"trait_type": "Face", "value": "<name>"},
    {"trait_type": "Hand", "value": "<name>"},
    {"trait_type": "Background", "value": "<name>"},
    // ... affinities, stats
  ],
  "image": "https://<BASE_URI>/<imageID>.gif"
}
```

Trait names are resolved via `LibTraitRegistry.getNameOf()`. See
[catalogs/traits/](traits/) for the full trait catalog.

---

## Room Assets

**Base path**: `packages/client/src/assets/images/rooms/`

### Directory pattern

```
rooms/{ID}_{kebab-case-name}/
  backgrounds/     — Room background images (.png)
  objects/         — Interactive room objects (.png)
  index.ts         — Exports for the room
```

**73 room directories** with client-side assets. Each room has 1-3 background
variants (e.g., `original.png`, `playtest.png`, `pretest.png`) and 0+ objects.

### Room config type

```typescript
// constants/rooms/types.ts
interface Room {
  index: number;
  backgrounds: string[];   // background image paths
  objects: RoomAsset[];     // interactive objects with coordinates
  music?: Music;
}

interface RoomAsset {
  name: string;
  coordinates?: { x1: number; y1: number; x2: number; y2: number };
  dialogue?: number;
}
```

### Example: Room 1 (Misty River)

```
rooms/1_misty-river/
  backgrounds/original.png
  backgrounds/playtest.png
  backgrounds/pretest.png
  objects/butterflies.png
  objects/mooring-post.png
  objects/mushrooms.png
```

### Room directory listing

| ID | Directory Name | ID | Directory Name |
|---|---|---|---|
| 0 | `0_loading` | 50 | `50_ancient_forest_entrance` |
| 1 | `1_misty-river` | 51 | `51_scrap_littered_undergrowth` |
| 2 | `2_tree-tunnel` | 52 | `52_airplane_crash` |
| 3 | `3_gate` | 53 | `53_blooming-tree` |
| 4 | `4_junkyard` | 54 | `54_plane_interior` |
| 5 | `5_restricted` | 55 | `55_shady-path` |
| 6 | `6_office-front` | 56 | `56_butterfly-forest` |
| 9 | `9_forest` | 57 | `57_river-crossing` |
| 10 | `10_forest-insect` | 58 | `58_mouth-of-scrap` |
| 11 | `11_waterfall` | 59 | `59_black-pool` |
| 12 | `12_junkyard-machine` | 60 | `60_scrap-trees` |
| 13 | `13_giftshop` | 61 | `61_decaying-forest-path` |
| 15 | `15_temple-cave` | 62 | `62_centipedes` |
| 16 | `16_techno-temple` | 63 | `63_deeper-forest-paths` |
| 18 | `18_cave-crossroads` | 64 | `64_burning-room` |
| 19 | `19_temple-of-the-wheel` | 65 | `65_forest-hut` |
| 25 | `25_lost-skeleton` | 66 | `66_trading-room` |
| 26 | `26_trash-strewn-graves` | 67 | `67_boulder-tunnel` |
| 27 | `27_guardhouse` | 68 | `68_slippery-pit` |
| 28 | `28_lobby` | 69 | `69_lotus-pool` |
| 29 | `29_road-out-of-woods` | 70 | `70_still-stream` |
| 30 | `30_scrapyard-entrance` | 71 | `71_shabby-deck` |
| 31 | `31_scrapyard-exit` | 72 | `72_hatch-to-nowhere` |
| 32 | `32_road-to-labs` | 73 | `73_broken-tube` |
| 33 | `33_forest-entrance` | 74 | `74_engraved-door` |
| 34 | `34_deeper-into-scrap` | 75 | `75_flood-mural` |
| 35 | `35_forest-road-i` | 76 | `76_fungus-garden` |
| 36 | `36_forest-road-ii` | 77 | `77_thriving-mushrooms` |
| 37 | `37_forest-road-iii` | 78 | `78_toadstool-platforms` |
| 47 | `47_scrap-paths` | 79 | `79_abandoned-campsite` |
| 48 | `48_forest-road-iv` | 80 | `80_radiant-crystal` |
| 49 | `49_clearing` | 81 | `81_flower-mural` |
| | | 82 | `82_geometric-cliffs` |
| | | 83 | `83_canyon-bridge` |
| | | 84 | `84_reinforced-tunnel` |
| | | 85 | `85_giants-palm` |
| | | 86 | `86_guardian-skull` |
| | | 87 | `87_sacrarium` |
| | | 88 | `88_treasure-hoard` |
| | | 89 | `89_trophies-of-the-hunt` |
| | | 90 | `90_scenic-view` |

Master registry: `constants/rooms/index.ts`

See [catalogs/rooms/](rooms/) for the full room catalog with on-chain data.

---

## Item Images

**Base path**: `packages/client/src/assets/images/items/`
**Count**: 180 PNG files
**Index**: `items/index.ts` exports `ItemImages` object

### Pattern

```
items/{snake_case_name}.png
```

Names are derived from the item's on-chain name using the
[cleanName()](#name-to-file-mapping) function.

### Lookup

```typescript
// network/shapes/utils/images.ts:23-27
export const getItemImage = (name: string) => {
  const key = cleanName(name) as keyof typeof ItemImages;
  return ItemImages[key];
};
```

See [catalogs/items/](items/) for the full item catalog.

---

## Skill Images

**Base path**: `packages/client/src/assets/images/skills/`
**Count**: 75 PNG files
**Index**: `skills/index.ts` exports `SkillImages` object

### Pattern

```
skills/{snake_case_name}.png
```

Same [cleanName()](#name-to-file-mapping) lookup as items.

```typescript
// network/shapes/utils/images.ts:29-33
export const getSkillImage = (name: string) => {
  const key = cleanName(name) as keyof typeof SkillImages;
  return SkillImages[key];
};
```

See [catalogs/skills/](skills/) for the full skill catalog.

---

## NPC Images

**Base path**: `packages/client/src/assets/images/npcs/`
**Count**: 39 PNG files
**Index**: `npcs/index.tsx` exports `NpcImages: Record<string, string>`

### Lookup

NPC images are keyed by **filename** (e.g., `"huntress.png"`), not cleanName.

### NPC Image Listing

| NPC | Files |
|---|---|
| Artie | `artie_default_1.png`, `artie_scary_1.png`, `artie_shocked_1.png`, `artie_shrug_1.png` |
| Dimidiatus | `dimidiatus_both.png`, `dimidiatus_frown.png`, `dimitiatus_laugh.png` |
| Huntress | `huntress.png`, `huntress_day.png`, `huntress_day_small.png`, `huntress_night.png`, `huntress_night_small.png`, `huntress_pose_2.png`, `huntress_small.png` |
| Lola | `lola_daydreaming.png`, `lola_default.png`, `lola_smiling.png` |
| Mina | `mina_default.png`, `mina_default_1.png`, `mina_laugh.png`, `mina_laugh_1.png`, `mina_pity.png`, `mina_pity_1.png`, `mina_shocked.png`, `mina_shocked_1.png` |
| Nurse | `nurse_cradling.png`, `nurse_lecturing.png`, `nurse_lecturing_small.png`, `nurse_smiling.png` |
| Rob | `robcat.png`, `robtransparent1.png`, `robtransparent2.png`, `robtransparent3.png` |
| Vend | `vend.png` |
| Menu characters | `menu_chibi_laughing.png`, `menu_chibi_thinking.png`, `menu_chibi_upset.png`, `menu_human_1.png`, `menu_human_2.png` |

See [catalogs/npcs/](npcs/) for the full NPC catalog.

---

## Icon Sets

**Base path**: `packages/client/src/assets/images/icons/`

### Trait type icons

Path: `icons/traits/`

| Trait Type | File |
|---|---|
| Body | `body.png` |
| Color | `color.png` |
| Face | `face.png` |
| Hand | `hand.png` |
| Background | `background.png` |

Exported as `TraitIcons` from `icons/traits/index.ts`.

### Affinity icons

Path: `icons/affinities/`

| Affinity | File |
|---|---|
| Eerie | `eerie.png` |
| Insect | `insect.png` |
| Normal | `normal.png` |
| Scrap | `scrap.png` |

### Stat icons

Path: `icons/stats/`

| Stat | File |
|---|---|
| Health | `health.png` |
| Power | `power.png` |
| Violence | `violence.png` |
| Harmony | `harmony.png` |
| Slots | `slots.png` |
| Stamina | `stamina.png` |
| XP | `xp.png` |

Exported as `StatIcons` from `icons/stats/index.ts`.

### Faction icons

Path: `icons/factions/`

| Faction | File |
|---|---|
| Kamigotchi Nursery | `kamigotchi_nursery.png` |
| Kamigotchi Tourism Agency | `kamigotchi_tourism_agency.png` |
| Mina's Shop | `minas_shop.png` |

### Other icon categories

| Category | Path | Purpose |
|---|---|---|
| Actions | `icons/actions/` | UI action buttons |
| Arrows | `icons/arrows/` | Navigation arrows |
| Battles | `icons/battles/` | Combat UI |
| Clock | `icons/clock/` | Time display |
| Indicators | `icons/indicators/` | Status indicators |
| Menu | `icons/menu/` | Menu UI elements |
| Misc | `icons/misc/` | Miscellaneous |
| Phases | `icons/phases/` | Day/night phase icons |
| Pricing | `icons/pricing/` | Economy UI |
| Statuses | `icons/statuses/` | Kami status icons |
| Triggers | `icons/triggers/` | Event triggers |
| Placeholder | `icons/placeholder.png` | Fallback icon |

---

## Map Zone Images

**Base path**: `packages/client/src/assets/images/map/`

| Zone | File |
|---|---|
| Zone 1 | `z1.webp` |
| Zone 2 | `z2.gif` |
| Zone 3 | `z3.png` |
| Zone 4 | `z4.png` |

---

## Token & UI Assets

### Blockchain token icons

Path: `packages/client/src/assets/images/tokens/`

| Token | File |
|---|---|
| Arbitrum | `arbitrum.png` |
| Base | `base.png` |
| ETH | `eth.png` |
| ETH (old) | `eth_old.png` |
| INIT | `init.png` |
| Onyx | `onyx.png` |

### Other UI assets

| Category | Path | Count | Notes |
|---|---|---|---|
| Loading screens | `images/loading/` | 9 | PNG splash screens |
| Banners | `images/banners/` | 5 | Promotional banners |
| Backgrounds | `images/backgrounds/` | 3 | Tiling patterns (`kami-pattern-*.png`) |
| Help | `images/help/` | 3 | Help screen images |
| Kamis (placeholder) | `images/kamis/` | 3 | `happy.webp`, `lethe.webp`, `placeholderKami.gif` |

---

## Name-to-File Mapping

Items, skills, affinities, factions, and stats share a common name-cleaning
function that converts on-chain names to asset filenames:

```typescript
// network/shapes/utils/images.ts:42-50
const cleanName = (name: string) => {
  if (!name) return '';
  name = name.toLowerCase();
  name = name.replaceAll(/ /g, '_').replaceAll(/-/g, '_');
  name = name.replaceAll('(', '').replaceAll(')', '');
  name = name.replaceAll(`'`, '').replaceAll(`\u2019`, '');
  name = name.replaceAll('"', '').replaceAll('\u201C', '').replaceAll(`"`, '');
  return name;
};
```

**Rules**:
1. Lowercase the name
2. Replace spaces and hyphens with underscores
3. Strip parentheses, single quotes (including curly `\u2019`), and all quote marks

**Example**: `"Black Poppy Extract"` → `black_poppy_extract` → `black_poppy_extract.png`

### Build configuration

All images are bundled at build time via Vite (`vite.config.ts`):
- `assetsInlineLimit: 0` — all assets are separate files (never base64-inlined)
- `assetsInclude`: `['**/*.gif', '**/*.jpg', '**/*.mp3', '**/*.png', '**/*.wav', '**/*.webp']`
- Path alias: `assets:` → `./src/assets`

---

## Asset Count Summary

| Category | Count | Format | Lookup |
|---|---|---|---|
| Kami characters | Dynamic (CDN) | GIF | Packed trait ID |
| Room backgrounds + objects | 281 files across 73 rooms | PNG | Room index → directory |
| Items | 180 | PNG | cleanName() |
| Skills | 75 | PNG | cleanName() |
| NPCs | 39 | PNG | Filename key |
| Icons (all categories) | 92 | PNG | Category-specific exports |
| Map zones | 4 | Mixed | Zone number |
| Tokens | 6 | PNG | Token name |
| UI/Loading/Banners | ~23 | Mixed | Static imports |

**Total bundled assets**: ~700 files
**Total on-chain/CDN assets**: Kami GIFs (one per minted Kami, generated from trait combos)
