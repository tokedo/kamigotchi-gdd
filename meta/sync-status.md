# Sync Status

Tracks which commit of the Kamigotchi source repo this GDD was last synced against.

| Field | Value |
|---|---|
| **Source repo** | `https://github.com/Asphodel-OS/kamigotchi` |
| **Pinned commit** | `91f69796627d38a678889b2b2bac103a79d3c68a` |
| **Commit message** | `add new spirit ring quests (#2423)` |
| **Sync date** | 2026-06-15 |
| **Synced by** | Scribe Agent (sync) |

## How to clone at pinned commit

Always clone at the pinned commit to ensure line numbers and logic match the GDD:

```bash
git clone https://github.com/Asphodel-OS/kamigotchi /tmp/kamigotchi
git -C /tmp/kamigotchi checkout 91f69796627d38a678889b2b2bac103a79d3c68a
```

## How to handle source repo updates

When the Kamigotchi repo gets new commits:

1. **Finish current extraction pass** at the pinned commit first
2. **Diff the source**: `git -C /tmp/kamigotchi diff 0af5d9f..origin/main -- packages/contracts/src/`
3. **Review which extracted mechanics were affected** by the diff
4. **Update only the affected GDD files** with new logic/line numbers
5. **Update the pinned commit** in this file to the new HEAD
6. **Log the update** in the sync log below

This keeps the GDD internally consistent at all times.

## Sync Log

| Date | Commit | Scope | Notes |
|---|---|---|---|
| 2026-03-24 | `d9b5009` | Full repo clone | Initial survey + began Phase 1 Core Kami extraction |
| 2026-03-31 | `0af5d9f` | `d9b5009..0af5d9f` (3 commits) | Recoil formula rewrite: Karma→Gaussian CDF multiplier, new Recoil Efficacy (affinity nudge), multiplicative recoil formula, DEF_RECOIL_BOOST bonus, KAMI_LIQ_KARMA_EFFICACY config. Co-op re-added to Black Pool dialogue. |
| 2026-07-05 | `91f6979` (unchanged) | Accuracy audit, no sync | Verified `main` HEAD `79b2cf36` differs from pin only by one client-only commit (contracts/ identical). Full 12-agent audit of all mechanics + catalogs; ~35 ERROR-level fixes applied across 49 files; 3 new files (data-tracking.md, admin-operations.md, catalogs/rooms/gates.csv). Findings: `meta/audit-2026-07-05.md`. |
| 2026-06-15 | `91f6979` | `0af5d9f..91f6979` (27 commits) | Harvest starve cutoff (`calcMaxMusu` inverse-strain bounty cap, #2368); `UPON_COOLDOWN_SET` bonus end type for Energy Drink (#2356); force-unequip on all kami ownership-change paths + `unequipAll` + `ON_UNEQUIP_`→`UPON_UNEQUIP_` prefix fix (#2406, #2408); Token Portal enable/disable toggle + claim address-override / onyx migration (#2415, #2417); newbie-vendor proceeds → marketplace fee recipient `KAMI_MARKET_FEE_RECIPIENT` (#2410, #2414); accept-offer custom errors + batch-fee simplification (#2407); Temple of the Wheel account-833 blocker removed, rooms 19/59 live (#2404). Catalogs: quest CSVs re-copied verbatim (155→192 quests; Act IV + Ring-of-Spirits lines live, #2311/#2421/#2422/#2423); items (Ring of Spirits 22802, Cultivation Spell Card XP rebalance, status flips), effects (SP→STAT, CLEARALL→CLEAR_BONUS), recipes & rooms status flips. Flag: undefined `XP+10000` allo on Cultivation III. |
