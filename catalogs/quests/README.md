# Quest Catalog Data

Quest data is too large to inline. Source files:

- `quests.csv` — ~130+ quest definitions (1308 lines): index, title, type, giver, dialogues, requirements, objectives, rewards
- `objectives.csv` — 168 objective definitions: delta type (INC/CURR/BOOL), data type, index, value
- `requirements.csv` — 160 requirement definitions: quest completion chains, item ownership, room checks, time windows
- `rewards.csv` — 56 reward definitions: items, faction reputation, flags

Source: `packages/contracts/deployment/world/data/quests/`

## Quest Type Summary

| Type | Index Range | Count | Giver | Description |
|---|---|---|---|---|
| MAIN | 1–108 | ~108 | MENU | Main story progression |
| FACTION | 2001–2016 | ~16 | MINA | Mina/Elders faction quests |
| SIDE | 3001–3024+ | ~24+ | Various | Optional side content |
| TEST | 10001–10003 | 3 | — | Development test quests |
| REPEATABLE | Various | ~10+ | Various | Daily/time-gated repeatable quests |
