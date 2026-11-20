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

The deployment writes two kinds of recipe requirement from
`recipes.csv`:

- **Tool** — a non-empty `Tool Index` adds `ITEM` / `CURR_MIN` 1 of that item
  (the tool is required, not consumed)
- **Room** — since `a99b4b64`, a `Room Index` above 0 adds `ROOM` / `BOOL_IS`
  that room, so the recipe can be crafted only while the account is in it. One
  recipe uses it: **10002 Offer an Obol** (1 Obol → 1 Silt-Caked Box), room 11
  — the lake offering, also the objective of the daily quest SQ901 *Wash Your
  Hands*

The `Min Level` column is not written as a requirement (the level requirement
is commented out in the deployment script, "until account levels are in").

> Source: `deployment/world/state/recipes/requirements.ts:3–30`,
> `deployment/world/state/recipes/recipes.ts:70–73`,
> `deployment/world/data/crafting/recipes.csv` (`Tool Index`, `Room Index`),
> `LibGetter.sol` (`ROOM`: the target's room equals the index)

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
