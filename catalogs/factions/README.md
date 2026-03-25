# Faction Catalog Data

3 factions that players build reputation with through quests, NPC interactions,
and community goals.

> Source: `packages/contracts/deployment/world/data/factions/factions.csv`

## Files

- `factions.csv` — 3 faction definitions

## Column Reference

| Column | Description |
|---|---|
| Image | Asset reference (not populated in CSV) |
| Name | Display name |
| Index | Unique faction index (1-3) |
| Key | Internal key used in code (Agency, Mina, Nursery) |
| Description | Flavor text |

## Factions

| Index | Name | Key | Leader | Description |
|---|---|---|---|---|
| 1 | The Agency | Agency | Menu | Administrative faction running the Kamigotchi world |
| 2 | The Elders | Mina | Mina | Merchant/business faction with mysterious investors |
| 3 | The Nursery | Nursery | (unknown) | Faction seeking to bring mysterious forces into the world |

## Reputation

Reputation is a numeric score tracked per account per faction. It can be
incremented or decremented by quest rewards, community goal contributions,
and other game events.

Faction keys are used in:
- Quest rewards (`rewards.csv` type `"REPUTATION"`)
- NPC faction assignment (`LibFaction.assign`)
- Leaderboard scoring per faction

See `mechanics/social/factions.md` for reputation system, assignment mechanics,
and leaderboard integration.
