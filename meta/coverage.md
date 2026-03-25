# GDD Coverage Tracker

Tracks which game mechanics have been extracted from source code into the GDD.

**Legend**: ⬜ Not started · 🔲 Surveyed (identified in code) · 🟨 In progress · ✅ Extracted

## Core Kami Mechanics
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Kami creation & traits | ✅ | `LibKamiCreate.sol`, `LibTraitRegistry.sol` | [creation.md](../mechanics/core-kami/creation.md) |
| Kami stats (health, power, violence, harmony) | ✅ | `LibStat.sol`, `Stat.sol`, `LibBonus.sol` | [stats.md](../mechanics/core-kami/stats.md) |
| Experience & leveling | ✅ | `LibExperience.sol`, `KamiLevelSystem.sol`, configs | [experience-leveling.md](../mechanics/core-kami/experience-leveling.md) |
| Health & healing (rest metabolism) | ✅ | `LibKami.sol`, `LibCooldown.sol`, configs | [health-healing.md](../mechanics/core-kami/health-healing.md) |
| Death & revival | ✅ | `LibKami.sol`, `KamiOnyxReviveSystem.sol` | [death-revival.md](../mechanics/core-kami/death-revival.md) |
| Naming & renaming | ✅ | `KamiNameSystem.sol`, `KamiOnyxRenameSystem.sol` | [naming.md](../mechanics/core-kami/naming.md) |
| Kami send (in-game transfer) | ✅ | `KamiSendSystem.sol` | [kami-send.md](../mechanics/core-kami/kami-send.md) |

## Economy & Items
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Items (creation, types, properties) | ✅ | `LibItem.sol`, `LibInventory.sol` | [items.md](../mechanics/economy/items.md) |
| Inventory management | ✅ | `LibInventory.sol` | [items.md](../mechanics/economy/items.md#inventory) |
| Equipment & slots | ✅ | `LibEquipment.sol` | [equipment.md](../mechanics/economy/equipment.md) |
| Crafting & recipes | ✅ | `LibRecipe.sol`, `CraftSystem.sol`, `recipes.csv` | [crafting.md](../mechanics/economy/crafting.md) |
| Droptables & loot | ✅ | `LibDroptable.sol`, `droptables.csv` | [droptables.md](../mechanics/economy/droptables.md) |
| Harvesting (farming) | ✅ | `LibHarvest.sol`, `HarvestStart/Stop/Collect`, configs | [harvesting.md](../mechanics/economy/harvesting.md) |
| Liquidation (harvest PvP) | ✅ | `HarvestLiquidateSystem.sol`, `LibHarvest.sol`, configs | [harvesting.md](../mechanics/economy/harvesting.md#liquidation-pvp) |
| Item usage (use/cast/burn/transfer) | ✅ | `KamiUseItem/CastItem/AccountUseItemSystem.sol`, `ItemBurn/TransferSystem.sol` | [item-usage.md](../mechanics/economy/item-usage.md) |

## Combat & PvP
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Murder / kill | ✅ | `LibKill.sol`, `Murder.t.sol` | [kill.md](../mechanics/combat/kill.md) |
| Hired hitman | ✅ | `LibKill.sol`, `HiredHitman.t.sol` | [kill.md](../mechanics/combat/kill.md#hired-hitman-quest-integration) |
| Sacrifice (commit-reveal) | ✅ | `LibSacrifice.sol`, `LibCommit.sol` | [sacrifice.md](../mechanics/combat/sacrifice.md) |
| Bonus system | ✅ | `LibBonus.sol` | [bonus-system.md](../mechanics/combat/bonus-system.md) |

## World & Movement
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Accounts & stamina | ✅ | `LibAccount.sol`, `AccountRegisterSystem.sol`, configs | [accounts.md](../mechanics/world/accounts.md) |
| Rooms & exits | ✅ | `LibRoom.sol`, `AccountMoveSystem.sol`, `rooms.csv` | [rooms.md](../mechanics/world/rooms.md) |
| Nodes (sub-locations) | ✅ | `LibNode.sol`, `nodes.csv` | [nodes.md](../mechanics/world/nodes.md) |
| Scavenging | ✅ | `LibScavenge.sol`, `LibAllo.sol`, `ScavengeClaimSystem.sol` | [scavenging.md](../mechanics/world/scavenging.md) |
| Day/night cycle | ✅ | `LibPhase.sol` | [day-night-cycle.md](../mechanics/world/day-night-cycle.md) |
| Account customization | ✅ | `AccountSetName/Bio/PFP/OperatorSystem.sol` | [account-customization.md](../mechanics/world/account-customization.md) |

## Progression & Social
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Skill tree | ✅ | `LibSkill.sol`, `LibSkillRegistry.sol`, `skills.csv`, `effects.csv` | [skills.md](../mechanics/progression/skills.md) |
| Quests | ✅ | `LibQuest.sol`, `LibQuestRegistry.sol`, `quests.csv`, objectives/requirements/rewards | [quests.md](../mechanics/progression/quests.md) |
| Community goals | ✅ | `LibGoal.sol`, `GoalContribute/ClaimSystem.sol`, `_GoalRegistrySystem.sol` | [goals.md](../mechanics/progression/goals.md) |
| Leaderboard / scoring | ✅ | `LibScore.sol` | [scoring.md](../mechanics/progression/scoring.md) |
| Factions | ✅ | `LibFaction.sol`, `factions.csv` | [factions.md](../mechanics/social/factions.md) |
| Relationships (NPC) | ✅ | `LibRelationship.sol`, `LibRelationshipRegistry.sol` | [relationships.md](../mechanics/social/relationships.md) |
| Friends | ✅ | `LibFriend.sol`, `FriendRequest/Accept/Block/CancelSystem.sol` | [friends.md](../mechanics/social/friends.md) |
| Chat & echo | ✅ | `ChatSystem.sol`, `LibEcho.sol` | [chat.md](../mechanics/social/chat.md) |

## Marketplace & Tokens
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Kami marketplace (list/buy/offer) | ✅ | `LibKamiMarket.sol`, `KamiMarketList/Buy/Offer/AcceptOffer/CancelSystem.sol`, `KamiMarketVault.sol` | [kami-market.md](../mechanics/marketplace/kami-market.md) |
| Item trading (p2p) | ✅ | `LibTrade.sol`, `TradeCreate/Execute/Complete/Cancel` | [trading.md](../mechanics/economy/trading.md) |
| NPC shop listings | ✅ | `LibListing.sol`, `LibListingRegistry.sol`, `LibGDA.sol`, `listings.csv` | [npc-shops.md](../mechanics/economy/npc-shops.md) |
| Auctions (GDA) | ✅ | `LibAuction.sol`, `LibAuctionRegistry.sol`, `LibGDA.sol`, `AuctionBuySystem.sol` | [auctions.md](../mechanics/marketplace/auctions.md) |
| Token portal (L1↔L2) | ✅ | `LibTokenPortal.sol`, `TokenPortalSystem.sol` | [token-portal.md](../mechanics/marketplace/token-portal.md) |
| Tax system | ✅ | `LibTax.sol` | [tax.md](../mechanics/marketplace/tax.md) |
| Newbie vendor (TWAP) | ✅ | `LibTWAP.sol`, `NewbieVendorBuySystem.sol` | [newbie-vendor.md](../mechanics/marketplace/newbie-vendor.md) |
| VIP system | ✅ | `LibVIP.sol`, `VipScore.sol`, `ProxyVIPScoreComponent.sol` | [vip.md](../mechanics/marketplace/vip.md) |

## Gacha & Minting
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Gacha mint/reroll/reveal | ✅ | `LibGacha.sol`, `KamiGachaMint/Reroll/RevealSystem.sol`, `GachaBuyTicketSystem.sol` | [gacha.md](../mechanics/gacha/gacha.md) |
| Kami creation (entity + traits + stats) | ✅ | `LibKamiCreate.sol`, `_721BatchMinterSystem.sol` | [kami-creation.md](../mechanics/gacha/kami-creation.md) |
| ERC-721 (Kami NFT) | ✅ | `Kami721.sol`, `LibKami721.sol`, `Kami721Stake/Unstake/Transfer/Metadata/IsInWorldSystem.sol` | [erc721.md](../mechanics/gacha/erc721.md) |

## Math & Utility (cross-cutting)
| Mechanic | Status | Source Files | GDD File |
|---|---|---|---|
| Fixed-point math & Gaussian RNG | ✅ | `FixedPointMathLib.sol`, `Gaussian.sol`, `Units.sol`, `LibRandom.sol` | [math-random.md](../mechanics/utility/math-random.md) |
| Cooldown system | ✅ | `LibCooldown.sol` | [cooldowns.md](../mechanics/utility/cooldowns.md) |
| Affinity matching | ✅ | `LibAffinity.sol` | [affinity.md](../mechanics/utility/affinity.md) |
| Conditional/requirement logic | ✅ | `LibConditional.sol` | [conditionals.md](../mechanics/utility/conditionals.md) |
| Allocation/reward system | ✅ | `LibAllo.sol` | [allocations.md](../mechanics/utility/allocations.md) |
| Soulbound locks | ✅ | `LibSoulbound.sol` | [soulbound.md](../mechanics/utility/soulbound.md) |
| Commit-reveal pattern | ✅ | `LibCommit.sol` | [commit-reveal.md](../mechanics/utility/commit-reveal.md) |
| Flag system | ✅ | `LibFlag.sol` | [flags.md](../mechanics/utility/flags.md) |
| NPC system | ✅ | `LibNPC.sol` | [npcs.md](../mechanics/utility/npcs.md) |
