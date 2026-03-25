# Crafting

> Source: `packages/contracts/src/libraries/LibRecipe.sol` (L1–238),
> `packages/contracts/src/systems/CraftSystem.sol` (L1–45),
> `packages/contracts/deployment/world/data/crafting/recipes.csv`

## Overview

Crafting converts input items into output items using predefined recipes. Recipes
are registered in a global registry and can have requirements (level, location,
etc.), stamina costs, and XP rewards.

See `catalogs/crafting/recipes.csv` for the full recipe catalog.

## Recipe Registry Shape

Each recipe is a registry entity with:

| Component | Description |
|---|---|
| `EntityType` | `"RECIPE"` |
| `IsRegistry` | Marks as registry entry |
| `IndexRecipe` | Unique recipe index (uint32) |
| `Type` | Recipe type string (behavior category) |
| `Experience` | XP reward per craft |
| `Stamina` | Stamina cost per craft (stored in `Stat.sync` field) |
| **Inputs** | Sub-entity with `Keys` (item indices) + `Values` (quantities) |
| **Outputs** | Sub-entity with `Keys` (item indices) + `Values` (quantities) |

Entity IDs:
- Recipe: `keccak256("registry.recipe", index)`
- Inputs: `keccak256("recipe.input", index)`
- Outputs: `keccak256("recipe.output", index)`

> Source: `LibRecipe.sol:30–84`

## Crafting Process

`CraftSystem.execute(recipeIndex, amount)`:

1. **Verify recipe exists** — reverts if recipe index not found
2. **Verify requirements** — check all conditions via `LibConditional` (e.g.,
   account level, location, specific items owned)
3. **Sync account** — update stamina based on elapsed time
4. **Before craft** — deduct stamina cost: `staminaCost × amount`
5. **Craft** — consume inputs, produce outputs (both scaled by `amount`)
6. **After craft** — grant XP: `recipeXP × amount` to the account
7. **Log** — track crafting stats

> Source: `CraftSystem.sol:16–39`

## Input/Output Scaling

Recipes support **batch crafting** via the `amount` parameter. Both inputs and
outputs are multiplied by the amount:

```
actualInputs[i]  = recipe.inputAmounts[i]  × amount
actualOutputs[i] = recipe.outputAmounts[i] × amount
```

Input consumption implicitly checks balances (reverts on underflow).

> Source: `LibRecipe.sol:136–151, 184–196`

## Requirements

Recipes can have **conditional requirements** checked before crafting:
- Validated via `LibConditional.check()` against the crafter's account
- Requirements are linked to the recipe via an anchor entity
- Common requirement types: minimum level, specific location, item ownership

> Source: `LibRecipe.sol:86–93, 163–166`

## Stamina Cost

Each recipe has a stamina cost stored in the `Stat.sync` field of the
`StaminaComponent`. The cost is deducted from the account's stamina before
crafting:

```
totalCost = staminaCost × amount
LibAccount.depleteStamina(accID, totalCost)
```

> Source: `LibRecipe.sol:125–134`

## XP Reward

If a recipe has an `Experience` component value > 0, that XP is granted to
the **account** (not the Kami) after crafting:

```
totalXP = recipeXP × amount
account.experience += totalXP
```

> Source: `LibRecipe.sol:153–158`
