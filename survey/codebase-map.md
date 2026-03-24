# Kamigotchi Codebase Map

> Produced 2026-03-24 from source commit `d9b5009`.
> This survey identifies which files contain game mechanics vs. client/infra code.

## Repo Overview

Pnpm monorepo with two packages:

| Package | Purpose | GDD Relevance |
|---|---|---|
| `packages/contracts/` | Solidity smart contracts (ECS on solecs) | **All game logic** |
| `packages/client/` | TypeScript/Vite frontend | Skip (rendering/UI/network) |

---

## Contracts: Game Mechanic Code (EXTRACT)

### Libraries (`src/libraries/`) — ~11,100 LoC total

The core logic. Each `Lib*.sol` is a stateless library called by system contracts.

| Library | Lines | Domain |
|---|---|---|
| `LibItem.sol` | 519 | Item creation, validation, usage |
| `LibKamiMarket.sol` | 503 | P2P Kami marketplace |
| `LibKami.sol` | 477 | Core Kami entity: stats, feeding, health, death |
| `LibTrade.sol` | 467 | Item trading |
| `LibSacrifice.sol` | 433 | Sacrifice (permanent death for rewards) |
| `LibHarvest.sol` | 415 | Resource harvesting — fertility, intensity, strain |
| `LibInventory.sol` | 414 | Inventory management, slots, capacity |
| `LibStat.sol` | 411 | Stat computation (health, power, violence, harmony) |
| `LibQuest.sol` | 407 | Quest accept/complete/drop |
| `LibGoal.sol` | 404 | Community goals |
| `LibTokenPortal.sol` | 394 | L1↔L2 token import/export |
| `LibKill.sol` | 375 | Murder/PvP, hired hitman |
| `LibBonus.sol` | 368 | Bonus system, multipliers |
| `LibAllo.sol` | 357 | Item allocation/allowance |
| `LibAccount.sol` | 299 | Account registration, movement, stamina |
| `LibConditional.sol` | 293 | Conditional logic for prerequisites |
| `LibRoom.sol` | 281 | Room definitions, exits, gates |
| `LibSkill.sol` | 273 | Skill tree, upgrade, respec |
| `LibData.sol` | 267 | Data packing/unpacking |
| `LibEquipment.sol` | 262 | Equipment equip/unequip |
| `LibKami721.sol` | 261 | ERC-721 staking/transfer |
| `LibScavenge.sol` | 256 | Scavenging (loot from rooms) |
| `LibRecipe.sol` | 238 | Crafting recipe validation |
| `LibNode.sol` | 225 | Room nodes (sub-locations) |
| `LibDroptable.sol` | 220 | Loot table resolution |
| `LibListing.sol` | 218 | NPC shop listings |
| `LibFlag.sol` | 187 | Flag system |
| `LibFriend.sol` | 186 | Friend system |
| `LibGacha.sol` | 184 | Gacha minting |
| `LibScore.sol` | 181 | Leaderboard scoring |
| `LibCommit.sol` | 168 | Commit-reveal scheme |
| `LibKamiCreate.sol` | 164 | Kami creation / trait assignment |
| `LibAuction.sol` | 139 | GDA auctions |
| `LibFaction.sol` | 117 | Faction membership |
| `LibRelationship.sol` | 113 | Kami-to-Kami relationships |
| `LibExperience.sol` | 109 | XP and leveling |
| `LibConfig.sol` | 108 | Config read helpers |
| `LibTax.sol` | 103 | Transaction taxes |
| `LibNPC.sol` | 94 | NPC definitions |
| `LibEcho.sol` | 76 | Room/kami broadcasting |
| `LibVIP.sol` | 71 | VIP scoring/stages |
| `LibTWAP.sol` | 60 | Time-weighted average price |
| `LibSoulbound.sol` | 30 | Non-transferable items |

### Utility Libraries (`src/libraries/utils/`) — ~1,700 LoC

| Library | Lines | Purpose |
|---|---|---|
| `LibRandom.sol` | 335 | RNG: commit-reveal, Gaussian, seeding |
| `LibGetter.sol` | 181 | Component getter helpers |
| `LibPack.sol` | 114 | Bit-packing for compact storage |
| `LibComp.sol` | 114 | Component access utilities |
| `LibSetter.sol` | 111 | Component setter helpers |
| `LibEntityType.sol` | 111 | Entity type constants/helpers |
| `LibCooldown.sol` | 109 | Cooldown timer mechanics |
| `LibAffinity.sol` | 91 | Affinity matching (body/hand → harvest) |
| `LibReference.sol` | 82 | Entity reference resolution |
| `LibArray.sol` | 79 | Array manipulation |
| `LibERC20.sol` | 70 | ERC-20 interaction |
| `LibFPConverter.sol` | 69 | Fixed-point conversion |
| `LibFor.sol` | 60 | "For" entity resolution |
| `LibGDA.sol` | 51 | Gradual Dutch Auction curve |
| `LibPhase.sol` | 37 | Game phase checks |
| `LibEmitter.sol` | 28 | Event emission |
| `LibDisabled.sol` | 27 | Disabled-entity checks |
| `AuthRoles.sol` | 27 | Role constants |

### Math Utilities (`src/utils/`)

| File | Lines | Purpose |
|---|---|---|
| `FixedPointMathLib.sol` | 378 | WAD/RAY fixed-point math |
| `Gaussian.sol` | 237 | Inverse CDF for Gaussian RNG |
| `VipScore.sol` | 212 | VIP scoring formula |
| `Units.sol` | 41 | Unit conversion constants |

### Systems (`src/systems/`) — Entry Points

70+ system contracts that expose callable functions to players. They validate inputs
then delegate to libraries. Key systems grouped by domain:

**Kami lifecycle**: `KamiGachaMintSystem`, `KamiGachaRevealSystem`, `KamiGachaRerollSystem`, `KamiLevelSystem`, `KamiNameSystem`, `KamiOnyxReviveSystem`, `KamiOnyxRenameSystem`, `KamiOnyxRespecSystem`, `KamiSendSystem`

**Items & crafting**: `KamiUseItemSystem`, `KamiCastItemSystem`, `AccountUseItemSystem`, `ItemBurnSystem`, `ItemTransferSystem`, `CraftSystem`, `DroptableRevealSystem`

**Equipment**: `KamiEquipSystem`, `KamiUnequipSystem`

**Harvesting**: `HarvestStartSystem`, `HarvestStopSystem`, `HarvestCollectSystem`, `HarvestLiquidateSystem`

**Combat**: `KamiSacrificeCommitSystem`, `KamiSacrificeRevealSystem`

**Economy**: `ListingBuySystem`, `ListingSellSystem`, `AuctionBuySystem`, `NewbieVendorBuySystem`, `TradeCreateSystem`, `TradeExecuteSystem`, `TradeCompleteSystem`, `TradeCancelSystem`

**Kami market**: `KamiMarketListSystem`, `KamiMarketBuySystem`, `KamiMarketOfferSystem`, `KamiMarketAcceptOfferSystem`, `KamiMarketCancelSystem`

**Quests & goals**: `QuestAcceptSystem`, `QuestCompleteSystem`, `QuestDropSystem`, `GoalContributeSystem`, `GoalClaimSystem`

**Skills**: `SkillUpgradeSystem`, `SkillRespecSystem`

**Social**: `FriendRequestSystem`, `FriendAcceptSystem`, `FriendCancelSystem`, `FriendBlockSystem`, `ChatSystem`, `EchoKamisSystem`, `EchoRoomSystem`, `RelationshipAdvanceSystem`

**Account**: `AccountRegisterSystem`, `AccountMoveSystem`, `AccountSetNameSystem`, `AccountSetBioSystem`, `AccountSetPFPSystem`, `AccountSetOperatorSystem`

**World/rooms**: `ScavengeClaimSystem`

**Tokens**: `TokenPortalSystem`, `Kami721StakeSystem`, `Kami721UnstakeSystem`, `Kami721TransferSystem`

### Components (`src/components/`) — ECS Data Schema

90+ component definitions. Each represents a data column in the ECS. Key ones by domain:

**Stats**: `HealthComponent`, `PowerComponent`, `ViolenceComponent`, `HarmonyComponent`, `StaminaComponent`, `ExperienceComponent`, `LevelComponent`, `SkillPointComponent`

**Identity**: `NameComponent`, `DescriptionComponent`, `MediaURIComponent`, `EntityTypeComponent`, `TypeComponent`, `SubtypeComponent`, `RarityComponent`

**Ownership**: `IDOwnsKamiComponent`, `IDOwnsInventoryComponent`, `IDOwnsEquipmentComponent`, `IDOwnsQuestComponent`, `IDOwnsSkillComponent`, `IDOwnsTradeComponent`, `IDOwnsRelationshipComponent`, `IDOwnsFlagComponent`

**Economy**: `BalanceComponent`, `CostComponent`, `TaxComponent`, `TokenAddressComponent`, `TokenAllowanceComponent`, `TokenHolderComponent`, `ValueComponent`, `WeightsComponent`

**Time**: `TimeComponent`, `TimeStartComponent`, `TimeEndComponent`, `TimeLastComponent`, `TimeNextComponent`, `TimeLastActionComponent`, `TimeResetComponent`, `PeriodComponent`, `DecayComponent`

**World**: `LocationComponent`, `ExitsComponent`, `SlotsComponent`, `IndexRoomComponent`, `IndexNodeComponent`

**State**: `StateComponent`, `IsCompleteComponent`, `IsDisabledComponent`, `HasFlagComponent`, `BlacklistComponent`, `WhitelistComponent`

### Game Data (`deployment/world/data/`) — CSV Catalogs

| Category | Files | Total Rows | Content |
|---|---|---|---|
| Quests | 4 CSVs | ~1,689 | Quest definitions, objectives, requirements, rewards |
| Items | 4 CSVs | ~300 | Item catalog, allocations, requirements, droptables |
| Rooms | 3 CSVs | ~187 | Room map, nodes, scavenge droptables |
| Skills | 2 CSVs | ~88 | Skill tree, effects |
| Traits | 5 CSVs | ~135 | Body, face, hand, color, background cosmetics |
| Crafting | 1 CSV | 41 | Recipes |
| Listings | 3 CSVs | ~60 | NPC shop items and pricing |
| Factions | 1 CSV | 3 | Faction definitions |
| NPCs | 2 CSVs | ~6 | NPC definitions and droptables |
| Auctions | 1 CSV | 2 | Auction definitions |
| Portal | 1 CSV | 2 | Bridgeable tokens |

### Config Parameters (`deployment/world/state/configs/`)

All tunable game constants, organized into:
- `initAccount` — stamina (100 max, 60s/point, 5 cost/move, 5 XP/move)
- `initLeveling` — XP curve (base 40, multiplier 1.259x)
- `initStats` — base stats (HP 50, Power/Violence/Harmony 10), healing metabolism, 180s cooldown
- `initHarvest` — fertility, intensity, bounty, strain (8-param curves each)
- `initLiquidation` — efficacy, animosity, threshold, salvage, spoils, karma, recoil
- `initTrade` — creation fee 3, delivery fee 50, tax 3/10
- `initMint` — 3000 max, WL 0.05 ETH, public 0.1 ETH
- `initSkills` — tree level reqs `[0, 5, 15, 25, 40, 55, 75, 95]`
- `initPortal` — 1-day delay, 1 flat + 100bps tax
- `initFriends` — 10 base, 10 requests
- `initVIP` — stage start + 2-week intervals
- `initNewbieVendor` — 0.005 ETH min, 24h TWAP, 48h cycle

---

## Contracts: Infrastructure Code (SKIP)

| Directory | Purpose | Why Skip |
|---|---|---|
| `src/solecs/` | ECS framework (World, Component, System base classes) | Framework plumbing, not game logic |
| `deployment/contracts/` | Deploy scripts, codegen templates | Deployment infra |
| `deployment/scripts/` | CLI tools (deployer, verifier, codegen) | Tooling |
| `deployment/commands/` | CLI commands | Tooling |
| `deployment/utils/` | Chain/anvil/forge helpers | Infra |
| `test/` | Foundry test suite | Useful as reference but not extraction target |

## Client Package (SKIP)

`packages/client/` contains:
- Vite + TypeScript frontend
- Phaser game engine rendering
- React UI components (modals, menus, HUD)
- Network/RPC layer
- Asset loading

All rendering/UI/network code — skip per extraction rules.

---

## Recommended Extraction Order

1. **Core Kami** — LibKami, LibKamiCreate, LibStat, LibExperience
2. **Economy** — LibHarvest, LibItem, LibInventory, LibEquipment, LibRecipe, LibDroptable, LibListing, LibTrade
3. **Combat/PvP** — LibKill, LibSacrifice, LibBonus
4. **World** — LibRoom, LibNode, LibScavenge, LibAccount
5. **Progression** — LibSkill, LibQuest, LibGoal, LibScore, LibRelationship, LibFaction
6. **Marketplace** — LibKamiMarket, LibAuction, LibGDA, LibTWAP, LibTokenPortal, LibTax
7. **Social** — LibFriend, LibEcho, Chat
8. **Gacha/Mint** — LibGacha, LibCommit, mint configs
9. **Catalogs** — All CSV data
