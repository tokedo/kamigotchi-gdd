# CLAUDE.md — Scribe Agent

  ## What This Project Is

  This repo contains a **Game Design Document (GDD)** for Kamigotchi — a pure
  on-chain MMORPG on Yominet. The GDD is a structured, complete description of
  all game mechanics, equations, catalogs, and economy rules, extracted from the
  Kamigotchi source code.

  ## Your Role

  You are the **Scribe Agent**. Your job is to read the Kamigotchi source code
  and produce clean, structured documentation following the schema in this repo.

  ## Source Repos

  - **Code (source of truth)**: https://github.com/Asphodel-OS/kamigotchi
  - **Official docs (secondary)**: https://docs.kamigotchi.io/

  When docs disagree with code, **code wins**. Note the discrepancy.

  ## Extraction Rules

  1. Extract game mechanics, equations, parameters, and object definitions only
  2. Ignore client rendering, network code, deployment, error handling boilerplate
  3. Translate formulas from code to math notation exactly — do not paraphrase
  4. Every fact links back to a source file and line range
  5. Completeness over polish — rough-but-complete beats polished-but-partial
  6. Every mechanic file should be self-contained enough to use in isolation

  ## Rules

  - **Never commit directly to main** — work on a branch, founder merges
  - Always update `meta/sync-status.md` when syncing from the source repo
  - Always update `meta/coverage.md` when adding new content
  - When uncertain about a mechanic's interpretation, add a `> ⚠️  UNCERTAIN:`
    note inline and flag it in coverage.md
