# Sync Status

Tracks which commit of the Kamigotchi source repo this GDD was last synced against.

| Field | Value |
|---|---|
| **Source repo** | `https://github.com/Asphodel-OS/kamigotchi` |
| **Pinned commit** | `0af5d9f040eaf1160b99d484e0140995c651f61e` |
| **Commit message** | `Fix: ATK_RECOIL_BOOST sign inversion in new recoil formula (#2400)` |
| **Sync date** | 2026-03-31 |
| **Synced by** | Scribe Agent (sync) |

## How to clone at pinned commit

Always clone at the pinned commit to ensure line numbers and logic match the GDD:

```bash
git clone https://github.com/Asphodel-OS/kamigotchi /tmp/kamigotchi
git -C /tmp/kamigotchi checkout 0af5d9f040eaf1160b99d484e0140995c651f61e
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
