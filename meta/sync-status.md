# Sync Status

Tracks which commit of the Kamigotchi source repo this GDD was last synced against.

| Field | Value |
|---|---|
| **Source repo** | `https://github.com/Asphodel-OS/kamigotchi` |
| **Pinned commit** | `d9b50091c44f40d708970c102a8410361a85f506` |
| **Commit message** | `UI changes to support fast travel (#2397)` |
| **Sync date** | 2026-03-24 |
| **Synced by** | Scribe Agent (initial survey) |

## How to clone at pinned commit

Always clone at the pinned commit to ensure line numbers and logic match the GDD:

```bash
git clone https://github.com/Asphodel-OS/kamigotchi /tmp/kamigotchi
git -C /tmp/kamigotchi checkout d9b50091c44f40d708970c102a8410361a85f506
```

## How to handle source repo updates

When the Kamigotchi repo gets new commits:

1. **Finish current extraction pass** at the pinned commit first
2. **Diff the source**: `git -C /tmp/kamigotchi diff d9b5009..origin/main -- packages/contracts/src/`
3. **Review which extracted mechanics were affected** by the diff
4. **Update only the affected GDD files** with new logic/line numbers
5. **Update the pinned commit** in this file to the new HEAD
6. **Log the update** in the sync log below

This keeps the GDD internally consistent at all times.

## Sync Log

| Date | Commit | Scope | Notes |
|---|---|---|---|
| 2026-03-24 | `d9b5009` | Full repo clone | Initial survey + began Phase 1 Core Kami extraction |
