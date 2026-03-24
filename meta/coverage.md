# GDD Coverage Tracker

Tracks which game mechanics have been extracted from source code into the GDD.

**Legend**: ⬜ Not started · 🔲 Surveyed (identified in code) · 🟨 In progress · ✅ Extracted

## Core Kami Mechanics
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Kami creation & traits | 🔲 | `LibKamiCreate.sol`, `LibGacha.sol`, `_TraitRegistrySystem.sol` | — |
| Kami stats (health, power, violence, harmony) | 🔲 | `LibStat.sol`, `LibKami.sol`, configs | — |
| Experience & leveling | 🔲 | `LibExperience.sol`, `KamiLevelSystem.sol`, configs | — |
| Health & healing (rest metabolism) | 🔲 | `LibKami.sol`, `LibStat.sol`, configs | — |
| Death & revival | 🔲 | `LibKami.sol`, `KamiOnyxReviveSystem.sol` | — |
| Naming & renaming | 🔲 | `KamiNameSystem.sol`, `KamiOnyxRenameSystem.sol` | — |

## Economy & Items
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Items (creation, types, properties) | 🔲 | `LibItem.sol`, `items.csv`, `allos.csv` | — |
| Inventory management | 🔲 | `LibInventory.sol`, `LibEquipment.sol` | — |
| Equipment & slots | 🔲 | `LibEquipment.sol`, `KamiEquipSystem.sol` | — |
| Crafting & recipes | 🔲 | `LibRecipe.sol`, `CraftSystem.sol`, `recipes.csv` | — |
| Droptables & loot | 🔲 | `LibDroptable.sol`, `droptables.csv` | — |
| Harvesting (farming) | 🔲 | `LibHarvest.sol`, `HarvestStart/Stop/Collect`, configs | — |
| Liquidation (harvest PvP) | 🔲 | `HarvestLiquidateSystem.sol`, `LibHarvest.sol`, configs | — |

## Combat & PvP
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Murder / kill | 🔲 | `LibKill.sol`, `Murder.t.sol` | — |
| Hired hitman | 🔲 | `LibKill.sol`, `HiredHitman.t.sol` | — |
| Sacrifice (commit-reveal) | 🔲 | `LibSacrifice.sol`, `LibCommit.sol` | — |
| Bonus system | 🔲 | `LibBonus.sol` | — |

## World & Movement
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Accounts & stamina | 🔲 | `LibAccount.sol`, configs | — |
| Rooms & exits | 🔲 | `LibRoom.sol`, `rooms.csv` | — |
| Nodes (sub-locations) | 🔲 | `LibNode.sol`, `nodes.csv` | — |
| Scavenging | 🔲 | `LibScavenge.sol`, `droptables.csv` | — |

## Progression & Social
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Skill tree | 🔲 | `LibSkill.sol`, `skills.csv`, `effects.csv` | — |
| Quests | 🔲 | `LibQuest.sol`, `quests.csv`, objectives/requirements/rewards | — |
| Community goals | 🔲 | `LibGoal.sol`, `GoalContribute/ClaimSystem.sol` | — |
| Leaderboard / scoring | 🔲 | `LibScore.sol` | — |
| Factions | 🔲 | `LibFaction.sol`, `factions.csv` | — |
| Relationships (kami-to-kami) | 🔲 | `LibRelationship.sol` | — |
| Friends | 🔲 | `LibFriend.sol` | — |
| Chat & echo | 🔲 | `ChatSystem.sol`, `LibEcho.sol` | — |

## Marketplace & Tokens
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Kami marketplace (list/buy/offer) | 🔲 | `LibKamiMarket.sol` | — |
| Item trading (p2p) | 🔲 | `LibTrade.sol`, `TradeCreate/Execute/Complete` | — |
| NPC shop listings | 🔲 | `LibListing.sol`, `listings.csv` | — |
| Auctions (GDA) | 🔲 | `LibAuction.sol`, `LibGDA.sol` | — |
| Token portal (L1↔L2) | 🔲 | `LibTokenPortal.sol` | — |
| Tax system | 🔲 | `LibTax.sol` | — |
| Newbie vendor (TWAP) | 🔲 | `LibTWAP.sol`, `NewbieVendorBuySystem.sol` | — |
| VIP system | 🔲 | `LibVIP.sol`, `VipScore.sol` | — |

## Gacha & Minting
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Gacha mint/reroll/reveal | 🔲 | `LibGacha.sol`, `KamiGachaMint/Reroll/RevealSystem.sol` | — |
| Mint config & pricing | 🔲 | `configs.ts` (initMint) | — |
| ERC-721 (Kami NFT) | 🔲 | `Kami721.sol`, `LibKami721.sol` | — |

## Math & Utility (cross-cutting)
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Fixed-point math | 🔲 | `FixedPointMathLib.sol` | — |
| Gaussian RNG | 🔲 | `Gaussian.sol`, `LibRandom.sol` | — |
| Cooldown system | 🔲 | `LibCooldown.sol` | — |
| Affinity matching | 🔲 | `LibAffinity.sol` | — |
| Conditional/requirement logic | 🔲 | `LibConditional.sol`, `LibAllo.sol` | — |
| Soulbound items | 🔲 | `LibSoulbound.sol` | — |
