# Crafting Recipes Catalog

> Source: `packages/contracts/deployment/world/data/crafting/recipes.csv`
> Commit: `d9b50091`

## Files

| File | Entries | Description |
|---|---|---|
| `recipes.csv` | 41 recipes | All crafting recipes with inputs, outputs, tools, and requirements |

## recipes.csv Schema

| Column | Type | Description |
|---|---|---|
| Index | uint32 | Unique recipe ID |
| Name | string | Recipe display name |
| Status | enum | `In Game`, `To Update`, `To Deploy` |
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
| Min Level | uint32 | Minimum Kami level to craft |

## Recipe Types

| Type | Count | Description |
|---|---|---|
| CONSUMABLE | 21 | Potions, food, offensive items |
| REAGENT | 12 | Extracted/processed materials (output 250–500 units) |
| MATERIAL | 7 | Bulk construction materials (Timber, Ingot, Ashlar) |
| SPECIAL | 3 | Quest item assembly (Aetheric Sextant, Dowsing Rod) |
| HIDDEN | 1 | Secret recipe (Wonder Egg from 5 Obols) |

## Tools

| Tool | Item Index | Used By | Description |
|---|---|---|---|
| Spice Grinder | 23100 | 11 recipes | Extractions and grinding |
| Portable Burner | 23101 | 19 recipes | Brewing and processing |
| Screwdriver | 23102 | 3 recipes | Assembly and chiseling |
| None | — | 4 recipes | No tool required |

## Status Distribution

| Status | Count |
|---|---|
| In Game | 25 |
| To Update | 13 |
| To Deploy | 3 |

## Level Requirements

Most recipes require level 1 (no restriction). Higher-level recipes:
- **Level 15**: Animistic Poison, Cthonic Blight, Toadstool Liquor, all Timber/Ingot/Ashlar recipes, Uninteresting Paste, Irradiated Root, Pale Potion
- **Level 20**: Flash Talisman, Fortified XP Potion, Pure Essence

## Cross-References

- Outputs → Items: `Output Index` references `catalogs/items/items.csv:Index`
- Inputs → Items: `Input Indices` reference `catalogs/items/items.csv:Index`
- Tools → Items: `Tool Index` references `catalogs/items/items.csv:Index`
- See [mechanics/economy/crafting.md](../../mechanics/economy/crafting.md) for crafting system rules
