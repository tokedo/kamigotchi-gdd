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

## What was completed in this session

1. ~~**Assets catalog**~~ ✅ `catalogs/assets.md` — 700+ asset pointers, CDN patterns
2. ~~**Cross-reference review**~~ ✅ 5 broken catalog paths fixed, all source citations verified
3. ~~**Kami vs Account XP fix**~~ ✅ Corrected XP attribution in experience-leveling.md and accounts.md
4. ~~**Quest catalog**~~ ✅ 155 quests + objectives + requirements + rewards + quest-lines.md + dialogues/
5. ~~**System entry points**~~ ✅ Added to equipment.md, relationships.md, chat.md
6. ~~**Catalog READMEs**~~ ✅ Added skills/README.md, factions/README.md
7. ~~**Config gaps**~~ ✅ VIP_STAGE values, NEWBIE_VENDOR_CYCLE, token addresses, local/prod notes

## All catalogs

| Catalog | Directory | Contents |
|---|---|---|
| Rooms | `catalogs/rooms/` | 70 rooms + 64 nodes + 50 scavenge droptables |
| Items | `catalogs/items/` | 177 items + 94 effects + 6 item droptables |
| NPCs | `catalogs/npcs/` | 2 NPCs + 19 shop listings |
| Recipes | `catalogs/crafting/` | 41 crafting recipes |
| Traits | `catalogs/traits/` | 135 traits (5 categories) |
| Assets | `catalogs/assets.md` | 700+ asset path references + CDN patterns |
| Quests | `catalogs/quests/` | 155 quests + objectives + requirements + rewards + quest chains + dialogues |
| Skills | `catalogs/skills/` | 71 skills + 16 effects |
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
