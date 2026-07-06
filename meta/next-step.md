# Next Step

This file is the single source of truth for what to do next. The Scribe Agent
reads this at the start of every session to know where to pick up.

## Current Phase

**Technical layer complete.** All mechanics extracted, all catalogs populated,
cross-references verified, config gaps filled.

## What to do next

The Layer 1 (technical GDD) is ready to serve as the foundation for Layer 2
outputs:

- **Layer 2a: Community docs** — player-friendly website (quest graph, mechanic
  guides, item database)
- **Layer 2b: AI game context** — compressed decision-relevant facts for AI
  gameplay
- **Layer 2c: Kami lore bot** — narrative context from quest dialogues and story

No further Layer 1 extraction work needed unless the source code changes.

## What was completed in this session (2026-07-05)

1. ~~**Full accuracy audit**~~ ✅ 12-agent audit of every mechanics file and
   catalog against source at pin `91f6979` (≡ `main` for contracts). Findings:
   `meta/audit-2026-07-05.md`.
2. ~~**All ERROR/OMISSION fixes applied**~~ ✅ 49 files corrected (tax basis
   points, exponential droptable weights, 0 HP ≠ death, dead-code item
   pipeline, token-portal unit scale, quest-drop reset, XP table, trait
   counts/affinities, README distribution tables, ~60 stale line citations).
3. ~~**New coverage**~~ ✅ `mechanics/utility/data-tracking.md` (LibData),
   `mechanics/utility/admin-operations.md`, `catalogs/rooms/gates.csv`
   (11 live room gates), Onyx Respec section in skills.md.
4. ~~**Upstream bug flags**~~ ✅ 4 suspected source bugs documented in
   coverage.md open flags (equipment bonuses inert, liquidation
   salvage/spoils bands, account respec revert, room 19 goal 999) — worth
   reporting to the Kamigotchi team.

## All catalogs

| Catalog | Directory | Contents |
|---|---|---|
| Rooms | `catalogs/rooms/` | 70 rooms + 64 nodes + 50 scavenge droptables + 11 room gates |
| Items | `catalogs/items/` | 178 items + 94 effects + 6 item droptables |
| NPCs | `catalogs/npcs/` | 2 NPCs + 19 shop listings |
| Recipes | `catalogs/crafting/` | 41 crafting recipes |
| Traits | `catalogs/traits/` | 135 traits (5 categories) |
| Assets | `catalogs/assets.md` | 700+ asset path references + CDN patterns |
| Quests | `catalogs/quests/` | 192 quests + objectives + requirements + rewards + quest chains + dialogues |
| Skills | `catalogs/skills/` | 72 skills + 16 effects |
| Factions | `catalogs/factions/` | 3 factions |

## Completed phases

- Phase 1: Core Kami (6/6) — creation, stats, experience/leveling, health/healing, death/revival, naming
- Phase 2: Economy (9/9) — Harvesting, Liquidation, Items, Inventory, Equipment, Crafting, Droptables, NPC Shops, Trading
- Phase 3: Combat/PvP (4/4) — Murder/Kill, Hired Hitman, Sacrifice, Bonus System
- Phase 4: World & Movement (4/4) — Accounts/Stamina, Rooms/Exits, Nodes, Scavenging
- Phase 5: Progression & Social (8/8) — Skills, Quests, Community Goals, Scoring/Leaderboard, Factions, Relationships, Friends, Chat/Echo
- Phase 6: Marketplace & Tokens (8/8) — Kami Market, Auctions, Token Portal, Tax, Newbie Vendor, VIP, Item Trading, NPC Shops
- Phase 7: Gacha & Minting (3/3) — Gacha (mint/reroll/reveal/tickets), Kami Creation, ERC-721
- Phase 8: Math & Utility (6/6) — Fixed-point math, Gaussian RNG, Random selection, Cooldowns, Affinity, Conditionals, Allocations, Soulbound
