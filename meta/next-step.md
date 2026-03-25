# Next Step

This file is the single source of truth for what to do next. The Scribe Agent
reads this at the start of every session to know where to pick up.

## Current Phase

**All extraction phases + gap analysis complete.**

## What to do next

All mechanics extraction is finished (50+ files). Suggested next steps:

1. **Catalog expansion** — ✅ All catalogs extracted:
   - ~~Rooms~~ ✅ `catalogs/rooms/` — 70 rooms + 64 nodes + 50 scavenge droptables
   - ~~Items~~ ✅ `catalogs/items/` — 177 items + 94 effects + 6 item droptables
   - ~~NPCs~~ ✅ `catalogs/npcs/` — 2 NPCs + 19 shop listings
   - ~~Recipes~~ ✅ `catalogs/crafting/` — 41 crafting recipes
   - ~~Traits~~ ✅ `catalogs/traits/` — 135 traits (30 bodies, 36 faces, 27 hands, 28 backgrounds, 14 colors)
   - ~~Assets~~ ✅ `catalogs/assets.md` — asset path reference (700+ files, CDN patterns)
2. ~~**Cross-reference review**~~ ✅ Completed — 5 broken catalog paths fixed, all 50+ mechanic
   files verified, all source citations valid

### Remaining gaps

- **Quest catalog CSVs** — `catalogs/quests/README.md` references 4 CSV files
  (quests.csv, objectives.csv, requirements.csv, rewards.csv) that were never
  extracted. Source: `packages/contracts/deployment/world/data/quests/`
- **Missing READMEs** — `catalogs/skills/` and `catalogs/factions/` have data
  files but no README.md (minor, all other catalogs have them)

## Completed phases

- Phase 1: Core Kami (6/6) — creation, stats, experience/leveling, health/healing, death/revival, naming
- Phase 2: Economy (9/9) — Harvesting, Liquidation, Items, Inventory, Equipment, Crafting, Droptables, NPC Shops, Trading
- Phase 3: Combat/PvP (4/4) — Murder/Kill, Hired Hitman, Sacrifice, Bonus System
- Phase 4: World & Movement (4/4) — Accounts/Stamina, Rooms/Exits, Nodes, Scavenging
- Phase 5: Progression & Social (8/8) — Skills, Quests, Community Goals, Scoring/Leaderboard, Factions, Relationships, Friends, Chat/Echo
- Phase 6: Marketplace & Tokens (8/8) — Kami Market, Auctions, Token Portal, Tax, Newbie Vendor, VIP, Item Trading, NPC Shops
- Phase 7: Gacha & Minting (3/3) — Gacha (mint/reroll/reveal/tickets), Kami Creation, ERC-721
- Phase 8: Math & Utility (6/6) — Fixed-point math, Gaussian RNG, Random selection, Cooldowns, Affinity, Conditionals, Allocations, Soulbound
