# Quest Lines & Chain Map

Cross-reference of quest prerequisites, chains, and storyline groupings.

Source: `packages/contracts/deployment/world/data/quests/quests.csv` —
Requirements column + `requirements.csv` (Type=QUEST entries define quest
completion gates).

---

## How Quest Chains Work

Each quest has a **Requirements** field that can contain:
- **Quest completion gates**: `Complete MSQ001`, `Complete MIN013`, etc.
- **Condition gates**: `In Room: Guardian Skull`, `Own the Dowsing Rod`, etc.
- **Combined**: A quest can require both prior quests AND conditions.

A quest is unlockable only when ALL its requirements are met simultaneously.

---

## Main Story Quest Line (MSQ)

109 quests (indices 1-109). Given by MENU, MINA, or DIMIDIATUS.
The main story splits into parallel branches after MSQ056 (Sanctuary Caves).

### Act I: Tutorial & Surface Exploration (MSQ001-MSQ020)

Linear chain introducing core mechanics, then investigating the world.

```
MSQ001  Welcome to Kamigotchi World
  |
MSQ002  Beginning Your Journey
  |
MSQ003  Your First Kami
  |
MSQ004  What Kami Do: Harvesting
  |--- MSQ005  What Kami Do: Scavenging
  |       |
  |     MSQ006  What Kami Do: Liquidating
  |
  |--- MSQ007  Making $MUSU  (branches from MSQ004, not MSQ006)
          |
        MSQ008  Supporting Local Businesses
          |
        MSQ009  Building a Reputation
          |
        MSQ010  KW and You: Having a Normal One
          |
        MSQ011  KW and You: That Eerie Feeling
          |
        MSQ012  KW and You: Insects in the Miasma
          |
        MSQ013  KW and You: Tossed in the Scrap Heap
          |
        MSQ014  Identifying Materials: Wood
          |
        MSQ015  Identifying Materials: Stone
          |
        MSQ016  Identifying Materials: Metal
          |
        MSQ017  Exploring Other Options
          |
        MSQ018  Harvesting Data I
          |
        MSQ019  Harvesting Data II
          |
        MSQ020  Harvesting Data III
```

**Key branch point**: MSQ004 splits to MSQ005 (Scavenging/Liquidating) AND
MSQ007 (Making $MUSU). MSQ006 is a dead end — the main story continues
through MSQ007.

### Act II: Investigation & Convergence (MSQ021-MSQ036)

MSQ021 requires BOTH MSQ020 AND MIN013 — this is the first major
cross-storyline gate, forcing players to progress Mina's quest line.

```
MSQ020 + MIN013
  |
MSQ021  Squaring the Circle I
  |
MSQ022  Squaring the Circle II
  |
MSQ023  Squaring the Circle III
  |
MSQ024  Squaring the Circle IV
  |
MSQ025  Squaring the Circle V
  |
MSQ026  Squaring the Circle VI
  |
MSQ027  A Forgotten Friend
  |
MSQ028  Deeper Understanding
  |
MSQ029  Computer Blues
  |
MSQ030  Humility
  |
MSQ031  Pyramid Power  (also requires MIN015)
  |
  +--- MIN016  Misogi  (requires MSQ031)
         |
       MSQ032  Safe Hex  (requires MIN016)
         |
       MSQ033  Holier Than Thou
         |
       MSQ034  Taking Great Pains
         |
       MSQ035  Steel Your Heart  (REWARD: Unlock Caves flag)
         |
       MSQ036  The Sanctuary Caves
```

**Key convergence**: MSQ031 requires both MSQ030 AND MIN015. The Mina line
and main story must both be complete to progress. MSQ035 unlocks the Caves
zone.

### Act III: Sanctuary Caves — Main Trunk (MSQ037-MSQ076)

Deep cave exploration. Linear except for branches spawning at MSQ056 and
MSQ070.

```
MSQ036
  |
MSQ037  Into the Depths
  |
MSQ038  Feeling in the Dark
  |
MSQ039  Where It Stems From
  |
MSQ040  Better Than Chopping Wood?
  |
MSQ041  Throw Me a Bone!
  |
MSQ042  Learning From a Master
  |
MSQ043  Sound of One Hand Clapping
  |
MSQ044  Mystery Machines
  |
MSQ045  Can't Stop With Just One
  |
MSQ046  Sweet As Honey
  |
MSQ047  Ringing Any Bells I
  |
MSQ048  Pipe Dream
  |
MSQ049  Community Service
  |
MSQ050  You Smelt It...
  |
MSQ051  Ringing Any Bells II
  |
MSQ052  Ringing Any Bells III
  |
MSQ053  Flash Back
  |
MSQ054  Out of the Blue
  |
MSQ055  Stop and Smell the Lotus
  |
MSQ056  Feel Like a Mushroom Myself  ** MAJOR BRANCH POINT **
  |
MSQ057  Top 10 Smoke Spots in KW
  |
MSQ058  Flower of the Flock
  |
MSQ059  Ordinary Intermediate Potion Crafting I
  |
MSQ060  Ordinary Intermediate Potion Crafting II
  |
MSQ061  Ordinary Intermediate Potion Crafting III
  |
MSQ062  Ordinary Intermediate Potion Crafting IV
  |
MSQ063  Ordinary Intermediate Potion Crafting V
  |
MSQ064  Full Flood
  |
MSQ065  Old Fiends
  |
MSQ066  First Contact
  |
MSQ067  Soaked to the Bone
  |
MSQ068  On the Fringes
  |
MSQ069  The Doors of Perception
  |
MSQ070  Reach Out And Touch Faith  ** SECOND BRANCH POINT **
  |
MSQ071  A Turn of the Screw
  |
MSQ072  Platforming Game  (also requires In Room: Toadstool Platforms)
  |
MSQ073  What Lies Beneath
  |
MSQ074  Operator Goes To Camp
  |
MSQ075  Digging Up Clues
  |
MSQ076  Get Some Fresh Air
```

### Act III: Sanctuary Caves — Side Branches from MSQ056

Four parallel branches open after completing MSQ056. These require MSQ056
plus being in specific cave rooms.

```
MSQ056 ---|--- MSQ077  Yep, That's Wood  (+ In Room: Reinforced Tunnel)
          |      |
          |    MSQ078  Plasmatics 101
          |      |
          |    MSQ079  Fringe Theories
          |
          |--- MSQ080  The Great Divide  (+ In Room: Canyon Bridge)
          |      |
          |    MSQ081  Edge of the Abyss
          |      |
          |    MSQ096  Into the Canyon  (+ In Room: Giant's Palm)
          |      |
          |    MSQ097  Back of my Hand
          |
          |--- MSQ084  Soot and Ash  (+ In Room: Flower Mural)
                 |
               MSQ085  Sigil Magic
                 |
               MSQ086  Sigil Magic II
```

### Act III: Sanctuary Caves — Side Branches from MSQ070

```
MSQ070 ---|--- MSQ087  The Crystal Set  (+ In Room: Radiant Crystal)
                 |
               MSQ088  The Crystal Set II
                 |
               MSQ089  The Crystal Set III
                 |
               MSQ090  The Crystal Set IV
                 |
               MSQ091  The Crystal Set V
                 |
               MSQ092  Beyond the Pale
                 |
               MSQ093  Tracing Your Roots
                 |
               MSQ094  Rhabdomancy  ** DOWSING ROD BRANCH POINT **
                 |
                 |--- MSQ095  Bitter Fruits  (+ Dowsing Rod + In Room: Marketplace)
                 |--- MSQ098  Heartwood  (+ Dowsing Rod + In Room: Blooming Tree)
                 |--- MSQ099  Hypersensitivity  (+ Dowsing Rod + In Room: Convenience Store)
                 |--- MSQ103  Sealed Fate  (+ Dowsing Rod + In Room: Sacrarium)
```

### Standalone Cave Quests (Room-Gated, No Quest Prereqs)

These quests trigger by visiting specific rooms in the Sanctuary Caves:

```
MSQ082  Blue Light Special  (In Room: Geometric Cliffs)
  |
MSQ083  Vision of Life

MSQ100  Prima Facie  (In Room: Guardian Skull)
  |
MSQ101  Pillars of the Earth

MSQ102  Axis Mundi  (In Room: Sacrarium)
  |
MSQ104  An Ounce of Sense
```

### Act IV: Temple of the Wheel (MSQ105-MSQ109, In Game)

Now live (In Game). Continues from MSQ104.

```
MSQ104
  |
MSQ105  The Turning of the Wheel
  |
MSQ106  Two Faces Under One Hood
  |
MSQ107  Get a Foot In The Door  (Giver: DIMIDIATUS)
  |
MSQ108  Treat Yourself  (Giver: DIMIDIATUS)
  |
MSQ109  Remain Unburdened of Attachments  (Giver: DIMIDIATUS)
```

---

## Mina Quest Line (MIN001-MIN016)

16 quests. All given by MINA. These run parallel to the main story and feed
into it at critical gates.

```
MSQ007
  |
MIN001  Grand Opening
  |
MIN002  Customer Loyalty Program
  |
MIN003  Early Market Research - I
  |--- MIN004  Early Market Research - II
  |      |
  |    MIN005  Early Market Research - III
  |      |
  |    MIN006  Community Outreach
  |      |
  |    MIN007  Restocking
  |      |--- (feeds into MIN011)
  |
  |--- MIN008  Offering an Apprenticeship  (branches from MIN003)
         |
       MIN009  Basics of Hex - Extracts
         |
       MIN010  Basics of Hex - Potions
         |--- (feeds into MIN011)
```

MIN011 converges both branches:

```
MIN007 + MIN010
  |
MIN011  Hex Education
  |
MIN012  Eye of the White Snake
  |
MIN013  Memorial for the Forgotten
  |
  |--- (feeds into MSQ021 as prerequisite)
  |
MIN014  To Whom It May Concern  (also requires MSQ030)
  |
MIN015  A Matter of Import
  |
  |--- (feeds into MSQ031 as prerequisite)
  |
MIN016  Misogi  (requires MSQ031 — bidirectional dependency)
  |
  |--- (feeds into MSQ032 as prerequisite)
```

### Critical Cross-Storyline Gates

| Gate | Requires | Unlocks |
|------|----------|---------|
| MSQ021 | MSQ020 + **MIN013** | Squaring the Circle series |
| MSQ031 | MSQ030 + **MIN015** | Pyramid Power |
| MIN014 | MIN013 + **MSQ030** | To Whom It May Concern |
| MIN016 | **MSQ031** | Misogi (waterfall test) |
| MSQ032 | **MIN016** | Safe Hex (cave unlock chain) |

---

## Side Quest Lines

24 side quests. Mix of standalone quests and short chains.

### Tutorial Side Quests (from Main Story)

```
MSQ003 --- SQ001  Rejecting Fate (Reroll)

MSQ004 --- SQ002  Health & Safety

MSQ005 --- SQ003  There Are Levels to This
              |
            SQ004  Skill Issue

MSQ008 --- SQ006  What's In a Name?
```

### Exploration Side Quests

```
MSQ017 --- SQ007  Land Survey
              |
            SQ008  Peregrination
```

SQ005 (Spring Rites) is gated by having a Kami liquidated, no quest prereq.

### Mina-Faction Side Quests

```
MIN008 --- SQ009  Tips Help The Most Right Now

MIN013 --- SQ010  Hex Education II
              |--- SQ011  Hex Education III
              |--- SQ014  Teatime

MIN007 --- SQ012  Customer Loyalty Program II
              |
            SQ013  Going Out For a Coffee
```

### Obols Side Chain

```
(Own 1 Obol) --- SQ015  The Wages of Sin
                    |
                  SQ016  Go Suck An Egg
```

### Annfwn (Other World) Side Quests

These quests are accessed through portal rooms in the deep caves.

```
(In Room: Treasure Hoard) --- SQ017  Rearview Mirror
                                 |
                               SQ018  One Man's Trash

(In Room: Trophies of the Hunt) --- SQ019  Happy Hunting Ground
                                       |
                                     SQ020  Better to Light a Candle

(In Room: Scenic View) --- SQ021  Castle in the Air  (Giver: ROB)
                              |
                            SQ022  Sweet Deal  (Giver: ROB)
```

### Adoption, Trading & Resonant Side Quests (SQ028-SQ045)

Rob's trading chain, Zevana's adoption/training chain, and the Dowsing-Rod–
gated "Resonant" cave quests.

```
Rob's trading chain (Giver: ROB):
(In Room: Restricted Area) --- SQ028  The More the Merrier
                                 |--- SQ029  Let’s Have a Look
                                       |--- SQ030  Container Deposit
                                             |--- SQ031  More Than a Fair Exchange
                                                   |--- SQ032  Unfettered
                                                         |--- SQ033  Trading Lunches

Zevana's adoption / training chain (Giver: ZEVANA):
(In Room: Torii Gate) --- SQ034  Not Right Now
                            |--- SQ035  Rookie Training I
                                  |--- SQ036  Rookie Training II
                                        |--- SQ037  Old Acquaintance
                                              |--- SQ038  Push it to the Limit

Resonant / Dowsing-Rod cave quests (require Own the Dowsing Rod):
SQ039  Danger Lies Ahead…  (In Room: Vending Machine, Giver: MENU)
SQ040  A Cautionary Note  (In Room: Shady Path, Giver: MENU)
SQ041  The Rot Creeps  (Complete MSQ094, In Room: Centipedes, Giver: MINA)
SQ042  Some Alien Vastness  (Complete MSQ094, In Room: Lab Entrance)
SQ043  Ringing Other Bells  (Complete MSQ094, In Room: Scrap Confluence)
SQ044  Resonant Drip  (Complete MSQ099, In Room: Wheel Temple, Giver: DIMIDIATUS)
SQ045  Resonant Flow  (Complete MSQ094, In Room: Black Pool, Giver: MINA)
```

### Spirit / Ring of Spirits Side Quests (SQ100-SQ118)

Unlocks after Act IV (MSQ109). Centered on the **Ring of Spirits** key item
(22802), which lets the bearer speak with lost souls scattered across the world.
SQ113-SQ118 are **To Deploy**.

```
(Complete MSQ109) --- SQ100  Get On the Other Side  (Giver: MENU)
                        |--- SQ101  Don’t Look Back  (Giver: MINA)
                              |--- SQ102  Find a Needle in the Haystack

Ring-of-Spirits soul conversations (require Own the Ring of Spirits):
SQ104  Call Your Grandparents  (In Room: Trash-Strewn Graves)
SQ105  Get Blood From a Stone  (In Room: Clearing)
SQ107  Look Who’s Talking  (In Room: Shabby Deck)
  |--- SQ108  My Ears Are Burning
SQ109  Get the Story Straight I  (In Room: Lost Skeleton)
  |--- SQ110  Get the Story Straight II
        |--- SQ111  Get the Story Straight III
SQ112  See the Glass as Half Full  (In Room: Guardian Skull)

To Deploy:
SQ113  Airing it Out  (Complete SQ108)  ──→  SQ114  Conditioned Environment
SQ115  Dry Conversation  (Complete SQ014)
SQ116  Lost and Found  (Complete SQ111)  ──→  SQ117  Trash Pickers  ──→  SQ118  Janitorial Supplies
```

### Diagnostics Side Quests (SQ802-SQ803)

```
SQ802  Quest Diagnostics  (temp quest, Giver: MENU)
  |--- SQ803  Never Brought to Mind  (Giver: ROB)
```

### Special / Conditional Side Quests

```
SQ998  Hidden GEM  (requires Aetheric Sextant + In Room: Slippery Pit)
SQ999  Claim Your Free Gift!  (time-limited, before 25/10/25)
SQ997  Condolences on your Recent Liquidations  (requires Kami liquidated)
```

---

## Test Quests (5 total, Index 1000000-1000004)

Development/testing quests. Not deployed to production.

```
test-0  Daily Test Quest  (daily, no prereqs)
test-1  Crafting Materials  (no prereqs)
  |--- test-3  Crafting Materials II
  |--- test-4  Crafting Toolkit
test-2  Lottery Loving  (no prereqs)
```

---

## Complete Dependency Graph Summary

### Entry Points (No Quest Prerequisites)

| Quest | Condition Gate |
|-------|---------------|
| MSQ001 | None (absolute start) |
| MSQ082 | In Room: Geometric Cliffs |
| MSQ100 | In Room: Guardian Skull |
| MSQ102 | In Room: Sacrarium |
| SQ005 | Kami liquidated |
| SQ015 | Own 1 Obol |
| SQ017 | In Room: Treasure Hoard |
| SQ019 | In Room: Trophies of the Hunt |
| SQ021 | In Room: Scenic View |
| SQ998 | Aetheric Sextant + In Room: Slippery Pit |
| SQ997 | Kami liquidated |
| SQ999 | Time-gated + Kami liquidated |

### Dead Ends (No Quest Depends On Them)

MSQ006, MSQ076, MSQ079, MSQ086, MSQ095, MSQ097, MSQ098, MSQ099, MSQ103,
MSQ109, MIN016 (feeds into MSQ032 but is itself a dead end in the MIN line),
SQ001, SQ002, SQ004, SQ005, SQ006, SQ008, SQ009, SQ011, SQ013, SQ014,
SQ016, SQ018, SQ020, SQ022, SQ997, SQ998, SQ999.

### Longest Chain

MSQ001 through MSQ076: 68 quests in the longest linear path (including the
MSQ021 gate which requires MIN001-MIN013 as a side requirement).
