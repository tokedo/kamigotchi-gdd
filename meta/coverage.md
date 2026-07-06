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
| Equipment & slots | ✅ | `LibEquipment.sol`, `KamiEquip/UnequipSystem.sol` | [equipment.md](../mechanics/economy/equipment.md) |
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
| Relationships (NPC) | ✅ | `LibRelationship.sol`, `LibRelationshipRegistry.sol`, `RelationshipAdvanceSystem.sol` | [relationships.md](../mechanics/social/relationships.md) |
| Friends | ✅ | `LibFriend.sol`, `FriendRequest/Accept/Block/CancelSystem.sol` | [friends.md](../mechanics/social/friends.md) |
| Chat & echo | ✅ | `ChatSystem.sol`, `LibEcho.sol`, `EchoKamis/RoomSystem.sol` | [chat.md](../mechanics/social/chat.md) |

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
| Data tracking counters | ✅ | `LibData.sol` | [data-tracking.md](../mechanics/utility/data-tracking.md) |
| Admin operations | ✅ | `AuthRoles.sol`, `_DistributeItemSystem.sol`, `_AdminSetFlagSystem.sol`, `_HarvestAdminSystem.sol`, `_SnapshotT2System.sol` | [admin-operations.md](../mechanics/utility/admin-operations.md) |

## Sync Notes & Open Flags

### 2026-06-15 — sync `0af5d9f..91f69796` (27 commits)

Mechanics touched and re-extracted:

- **Harvesting** — starve cutoff: bounty now capped at `calcMaxMusu` (inverse
  strain). Strain formula + cap documented in `harvesting.md`.
- **Bonuses** — new `UPON_COOLDOWN_SET` end type (Energy Drink), consumed on
  every cooldown reset; `END_TYPE_PREFIX` corrected `ON_UNEQUIP_`→`UPON_UNEQUIP_`.
- **Equipment** — new `unequipAll`; force-unequip on every ownership-change path
  (send, list, market sale, bridge-out, sacrifice, gacha reroll).
- **Token Portal** — global enable/disable toggle (`isEnabled`/`adminToggleEnabled`);
  claim reads token address from Portal registry (overrides receipt → token
  migration semantics).
- **Newbie Vendor** — proceeds routed to `KAMI_MARKET_FEE_RECIPIENT`
  (`0x3d7f…2872`), fallback to vendor address.
- **Marketplace** — accept-offer custom errors + simplified batch fee.
- **Temple of the Wheel** — temporary account-833 blocker removed (rooms 19/59
  now In Game; node 19 open).
- **Catalogs** — quest CSVs re-copied verbatim (192 quests; Act IV MSQ105-109 +
  Ring-of-Spirits SQ100-118 lines now live); items/effects/recipes/rooms updated.

### 2026-07-05 — accuracy audit at pin `91f69796`

Full 12-agent audit of every mechanics file and catalog against source
(`main` HEAD `79b2cf36` differs from the pin only by one client-only commit).
Findings logged in [audit-2026-07-05.md](audit-2026-07-05.md); all
ERROR/OMISSION findings were fixed in the same pass. Highlights:

- ~35 ERROR-level corrections (tax basis points, exponential droptable
  weights, 0 HP ≠ death, dead-code item-use pipeline, token-portal unit
  scale, quest-drop reset, XP table floor, trait counts, invented face
  affinities). Note: the 2026-06-15 sync note above claimed Energy Drink
  uses `UPON_COOLDOWN_SET` — the deployed catalog actually uses
  `UPON_HARVEST_ACTION`; no deployed bonus uses `UPON_COOLDOWN_SET`.
- New docs: [data-tracking.md](../mechanics/utility/data-tracking.md),
  [admin-operations.md](../mechanics/utility/admin-operations.md),
  [gates.csv](../catalogs/rooms/gates.csv) (11 live room gates).
- Zero dangling source references; quest/skill/faction/dialogue CSVs verified
  byte-identical to source.

### Open flags

- ⚠️ **`XP+10000` undefined allo** — Cultivation III Spell Card (11213)
  references effect `XP+10000`, which is **not defined** in `effects.csv`/
  `allos.csv`. Unmatched allos are skipped at deploy, so the card currently
  grants only `HP+100`. See [items README → Known Discrepancies](../catalogs/items/README.md#known-discrepancies).
- 📝 **Quest-line graph** — `catalogs/quests/quest-lines.md` side-quest chains
  for the new SQ028-045 / SQ100-118 / SQ802-803 lines were reconstructed from
  the `Requirements` column (best-effort edges); the source-of-truth CSVs are
  fully synced, but a deeper narrative pass over the new lines is worthwhile.
- ⚠️ **Suspected upstream bug: equipment bonuses inert** — deployed catalog
  registers equipment bonus allos under the `USE` case with bare
  `UPON_UNEQUIP` terminator, while equip reads the `EQUIP` anchor and clears
  `UPON_UNEQUIP_{SLOT}` — catalog equipment bonuses neither attach nor clear.
  See [equipment.md](../mechanics/economy/equipment.md),
  [bonus-system.md](../mechanics/combat/bonus-system.md).
- ⚠️ **Suspected upstream bug: liquidation salvage/spoils bands** — integer
  division defeats the 100% clamp: victim Power 101–199 makes
  `bounty − salvage` underflow (liquidation reverts — victim unliquidatable);
  attacker Power 56–154 takes 101–199% spoils uncapped. See
  [kill.md](../mechanics/combat/kill.md).
- ⚠️ **Suspected upstream bug: account respec always reverts** —
  `SkillRespecSystem` has its RESTING check in the account branch (accounts
  carry no StateComponent); Kami respec conversely skips the RESTING gate.
  See [skills.md](../mechanics/progression/skills.md).
- ⚠️ **Room 19 gate references undefined goal 999** — `gates.ts` gates Temple
  of the Wheel on `getGoalID(999)`, which no goal defines. See
  [gates.csv](../catalogs/rooms/gates.csv) / rooms README.
- ⚠️ **Revival flavor vs requirements** — Djed Pillar (11003) and Pale Potion
  (11004) carry revival flavor text but require RESTING / KAMI_CAN_EAT, so
  they cannot be used on dead Kamis; only 11001/11002 require `STATE == DEAD`.
  See [death-revival.md](../mechanics/core-kami/death-revival.md).
- ⚠️ **Goal catalog not fully derivable from source** — `goals.ts` deploys
  only goals 7 and 13; goals 1–6/8–11 are commented out and were seeded at
  runtime by admin. See [goals.md](../mechanics/progression/goals.md).
- ⚠️ **Token portal enabled-state unverifiable** — `isEnabled` is mutable
  runtime state (defaults `false`); no repo artifact pins the live value. See
  [token-portal.md](../mechanics/marketplace/token-portal.md).
