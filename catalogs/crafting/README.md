# Crafting Recipes Catalog

> Source: `packages/contracts/deployment/world/data/crafting/recipes.csv`
> Commit: `a99b4b64`

## Files

| File | Entries | Description |
|---|---|---|
| `recipes.csv` | 43 recipes | All crafting recipes with inputs, outputs, tools, and requirements |

## recipes.csv Schema

| Column | Type | Description |
|---|---|---|
| Index | uint32 | Unique recipe ID |
| Name | string | Recipe display name |
| Status | enum | `In Game`, `To Update`, `To Deploy`, `Defunct` (retired, not deployed) |
| Type | enum | `CONSUMABLE`, `REAGENT`, `MATERIAL`, `SPECIAL`, `HIDDEN` |
| Output Index | uint32 | Item produced (references `catalogs/items/items.csv`) |
| Output Name | string | Output item name for reference |
| Output Amount | uint32 | Quantity produced per craft |
| Input Indices | int[] | Comma-separated input item indices |
| Input Amounts | int[] | Comma-separated quantities required |
| Stamina Cost | uint32 | Stamina consumed to craft |
| XP Output | uint32 | Experience gained from crafting |
| Tool Index | uint32 | Required tool item (empty = no tool) |
| Tool Name | string | Tool name for reference |
| Min Level | uint32 | Minimum level listed in the sheet — not written on chain (the deployment's level requirement is commented out) |
| Room Index | uint32 | Room the account must be in to craft (empty = anywhere); written as a `ROOM` requirement since `a99b4b64` |

## Recipe Types

| Type | Count | Description |
|---|---|---|
| CONSUMABLE | 18 | Potions, food, offensive items |
| REAGENT | 13 | Extracted/processed materials (output 250–500 units; Pure Essence outputs 1) |
| MATERIAL | 7 | Bulk construction materials (Timber, Ingot, Ashlar, Empty Cup) |
| SPECIAL | 3 | Quest item assembly (Aetheric Sextant, Dowsing Rod, Igors Amulet) |
| HIDDEN | 2 | Secret recipes: Offer an Obol (1 Obol → Silt-Caked Box, room 11 only); Craft Wonder Egg (5 Obols, `Defunct`) |

## Tools

| Tool | Item Index | Used By | Description |
|---|---|---|---|
| Spice Grinder | 23100 | 13 recipes | Extractions and grinding |
| Portable Burner | 23101 | 20 recipes | Brewing and processing |
| Screwdriver | 23102 | 3 recipes | Assembly and chiseling |
| None | — | 7 recipes | No tool required |

## Status Distribution

| Status | Count |
|---|---|
| In Game | 42 |
| Defunct | 1 (10001 Craft Wonder Egg) |

## Level Requirements

These are the sheet's `Min Level` values. The deployment does not write them
as requirements (the level requirement is commented out in
`deployment/world/state/recipes/requirements.ts`), so no recipe is level-gated
on chain by the data.

Most recipes list level 1. Higher-level recipes:
- **Level 15**: Animistic Poison, Cthonic Blight, Toadstool Liquor, all Timber/Ingot/Ashlar recipes, Uninteresting Paste, Irradiated Root, Pale Potion
- **Level 20**: Flash Talisman, Fortified XP Potion, Pure Essence

## Cross-References

- Outputs → Items: `Output Index` references `catalogs/items/items.csv:Index`
- Inputs → Items: `Input Indices` reference `catalogs/items/items.csv:Index`
- Tools → Items: `Tool Index` references `catalogs/items/items.csv:Index`
- See [mechanics/economy/crafting.md](../../mechanics/economy/crafting.md) for crafting system rules
