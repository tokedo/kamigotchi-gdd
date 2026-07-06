# Admin Operations

> Source: `packages/contracts/src/libraries/utils/AuthRoles.sol` (L1–27),
> `packages/contracts/src/systems/_AuthManageRoleSystem.sol` (L1–35),
> individual admin systems cited inline

## Overview

A small set of privileged systems mutates live game state outside normal
player actions: item airdrops, flag grants, force-stopping harvests, batch
order cleanup, and reward distributions. They are gated by role flags, not by
contract ownership (ownership separately gates registry/config systems and
role assignment itself).

## Role Gating

| Modifier | Required flag | Source |
|---|---|---|
| `onlyAdmin` | `ROLE_ADMIN` on the caller's address entity `uint256(uint160(msg.sender))` | `AuthRoles.sol:20–26` |
| `onlyCommManager` | `ROLE_COMMUNITY_MANAGER`, same entity derivation | `AuthRoles.sol:12–18` |

Roles are ordinary flags granted/removed by the contract owner via
`_AuthManageRoleSystem`, written with `LibFlag.setFull` and parentType
`"AUTH"` (`_AuthManageRoleSystem.sol:15–33`). Deployed role assignments live
in `deployment/world/data/auth/roles.csv`: the deployer hot wallet and
`izanami` hold both Admin and Community Manager in production.

## State-Mutating Admin Systems

| System / entrypoint | Gate | Effect |
|---|---|---|
| `_DistributeItemSystem` | `onlyAdmin` | Targeted item airdrop: `(accounts[], itemIndex, amounts[])` → per-account inventory grant; source comment: "compensation, apologies, targeted rewards" (`_DistributeItemSystem.sol:14–39`) |
| `_AdminSetFlagSystem` | `onlyAdmin` | Sets or clears an arbitrary flag on account entities (each target must be an account, `:24`); used for giveaways, airdrops, whitelists (`_AdminSetFlagSystem.sol:14–36`) |
| `_HarvestAdminSystem.stop` / `stopBatched` | `onlyAdmin` | Force-stops a harvest by Kami index: verifies `HARVESTING` state, syncs, stops the harvest, sets the Kami `RESTING`, resets cooldown and harvest bonuses, emits `HARVEST_STOP`; `stopBatched` loops over indices (`_HarvestAdminSystem.sol:21–56`) |
| `_SnapshotT2System.dropKillRewards` | `onlyAdmin` | Batch OBOL distribution to owner addresses (`_SnapshotT2System.sol:19–29`); passport distribution and whitelist helpers exist only as commented-out code (`_SnapshotT2System.sol:31–47, 57–69`) |
| `KamiMarketCancelSystem.executeAdmin` | `onlyAdmin` | Cancels any active Kami market order (listing, offer, or collection offer) on behalf of its owner (`KamiMarketCancelSystem.sol:31–38`) |
| `TradeCancelSystem.executeAdmin` | `onlyAdmin` | Batch-cancels `PENDING` trades; emits the cancel event with acting account 0 (`TradeCancelSystem.sol:37–45`) |
| `TradeCompleteSystem.executeAdmin` | `onlyAdmin` | Batch-completes `EXECUTED` trades on behalf of each maker (`TradeCompleteSystem.sol:37–48`) |
| `DroptableRevealSystem.forceReveal`, `KamiGachaRevealSystem.forceReveal` | `onlyCommManager` | Recovers commits whose 256-block reveal window lapsed — see [commit-reveal.md](commit-reveal.md) (`DroptableRevealSystem.sol:35`, `KamiGachaRevealSystem.sol:33–36`) |

Registry and config systems (`_*RegistrySystem`, `_ConfigSetSystem`) are also
`onlyAdmin`, but they define game content and parameters rather than mutating
player state; they are covered by the respective mechanic docs.
