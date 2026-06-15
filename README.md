# Kamigotchi GDD

Technical Game Design Document for [Kamigotchi](https://kamigotchi.io/) — a
pure on-chain MMORPG on Yominet.

Extracted from source code at commit `91f69796`.

## What's Here

- `mechanics/` — 48 mechanic files covering all game systems with formulas,
  configs, and source citations
- `catalogs/` — complete game data: 178 items, 71 skills, 192 quests, 70 rooms,
  64 nodes, 135 traits, 41 recipes, 3 factions, 2 NPCs
- `meta/` — coverage tracker, sync status, next steps

## Sync Prompt

When the Kamigotchi source repo has new commits, open a Claude Code session in
this directory and paste the following prompt:

```
The Kamigotchi source repo is at /tmp/kamigotchi (pull latest first with
git -C /tmp/kamigotchi pull). Our GDD was scribed against commit 91f69796.

1. Run: git -C /tmp/kamigotchi log 91f69796..HEAD --oneline
2. If there are new commits, run: git -C /tmp/kamigotchi diff 91f69796..HEAD --stat
3. For any changed files in src/libraries/, src/systems/, or deployment/world/state/,
   read the diffs and compare against our GDD files in mechanics/ and catalogs/
4. Produce a report:
   - NEW: mechanics/systems not yet in the GDD
   - CHANGED: formulas, configs, or logic that differ from what we documented
   - DATA: catalog changes (new items, quests, skills, rooms, config value changes)
   - NO IMPACT: changes that don't affect game mechanics (tests, tooling, client)
5. Do NOT make any edits yet — just report what needs updating so we can review first
6. After we agree on the changes, update the GDD files and bump the checkpoint
   in meta/sync-status.md to the new commit hash
```

## Layer 2 Projects

This GDD serves as the foundation for downstream outputs:

- **[kamigotchi-wiki](https://github.com/tokedo/kamigotchi-wiki)** — community
  website with interactive guides, databases, and tools — live at
  [kamiwiki.xyz](https://kamiwiki.xyz)
- **[kamigotchi-context](https://github.com/tokedo/kamigotchi-context)** — AI
  agent knowledge base: decision-oriented game context, on-chain integration
  docs, ABIs, and state reading patterns for agents that play Kamigotchi
