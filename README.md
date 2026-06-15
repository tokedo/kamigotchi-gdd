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

## Wiki Update Prompt

After this GDD has been synced to a new source commit, open a Claude Code
session in the kami-wiki repo and paste the following prompt:

```
The kamigotchi-gdd repo (local clone, or https://github.com/tokedo/kamigotchi-gdd)
is the single source of truth for game mechanics and data. It is synced to source
commit 91f69796. Update the wiki so it reflects the CURRENT state of the game as
described in the GDD.

How to scope the work efficiently:
1. Read the GDD's meta/sync-status.md (latest Sync Log row) and run
   `git -C <gdd> log` / `git -C <gdd> diff <prev>..<current>` to see which
   mechanic and catalog files moved.
2. For each affected system or catalog, update the matching wiki page(s) so they
   match the GDD's current description and data (catalogs/*.csv are authoritative
   for items, effects, recipes, rooms, quests, etc.).

CRITICAL — the wiki is a present-tense SNAPSHOT, never a changelog:
- Describe how the game works RIGHT NOW. Never describe what changed.
- Do NOT use words like "new", "now", "updated", "added", "removed", "recently",
  "previously", "no longer", "expanded from X to Y", "as of patch", patch/version
  numbers, or dates.
- Overwrite stale content in place. Do not append "what's changed" sections.
- A player must see only the current rules and catalogs, with no hint of history.

Framing examples:
- WRITE  "Listing, selling, sending, or sacrificing a Kami returns its equipped
          items to your inventory."
  NOT    "Equipment is now auto-unequipped when you transfer a Kami."
- WRITE  "There are 192 quests across four main-story acts."
  NOT    "The number of quests was expanded to 192."
- WRITE  "Harvest yield is capped by the Kami's current HP."
  NOT    "A new starve-cutoff now caps harvest yield."

Every fact on the wiki must trace to the current GDD. Where the GDD flags an
UNCERTAIN / discrepancy (see meta/coverage.md and catalog READMEs — e.g. an item
effect referencing an undefined value), verify the live on-chain behavior before
publishing the intended number; if it cannot be verified, document the behavior
that actually occurs, still in present tense.
```

## Layer 2 Projects

This GDD serves as the foundation for downstream outputs:

- **[kamigotchi-wiki](https://github.com/tokedo/kamigotchi-wiki)** — community
  website with interactive guides, databases, and tools — live at
  [kamiwiki.xyz](https://kamiwiki.xyz)
- **[kamigotchi-context](https://github.com/tokedo/kamigotchi-context)** — AI
  agent knowledge base: decision-oriented game context, on-chain integration
  docs, ABIs, and state reading patterns for agents that play Kamigotchi
