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

**Roles are not grantable through the generic flag system.** Because
`onlyAdmin` is a plain flag lookup at `genID(uint160(addr), "ROLE_*")` and an
account entity ID *is* `uint160(owner)`, a bare flag write would let any admin
mint a functional role — bypassing the owner gate, and with no `IDType` anchor,
invisible to the anchored roles audit. `_AdminSetFlagSystem` therefore rejects
any `flagType` beginning with `ROLE_`, reverting `"roles: use auth registry"`.
Roles can only be set through `_AuthManageRoleSystem.setFull`.

> Source: `_AdminSetFlagSystem.sol:25–33`

## State-Mutating Admin Systems

| System / entrypoint | Gate | Effect |
|---|---|---|
| `_DistributeItemSystem` | `onlyAdmin` | Targeted item airdrop: `(accounts[], itemIndex, amounts[])` → per-account inventory grant; source comment: "compensation, apologies, targeted rewards" (`_DistributeItemSystem.sol:14–39`) |
| `_AdminSetFlagSystem` | `onlyAdmin` | Sets or clears a flag on account entities (each target must be an account); `ROLE_`-prefixed flag types are rejected outright; used for giveaways, airdrops, whitelists (`_AdminSetFlagSystem.sol:16–38`) |
| `_HarvestAdminSystem.stop` / `stopBatched` | `onlyAdmin` | Force-stops a harvest by Kami index **on the owner's behalf, as a player stop does**: verifies `HARVESTING` state, syncs, then claims the balance for the Kami's **owner** (`LibKami.getAccount`, not the caller; tax applies as on any claim), stops the harvest, sets the Kami `RESTING`, grants XP = the output, resets cooldown and `UPON_COOLDOWN_SET` bonuses, adds the output to the node's scavenge bar for the owner, resets harvest-stop bonuses, adds the `COLLECT` score, logs harvest time and amounts, and emits `HARVEST_STOP` with the owner's account and the output; `stopBatched` loops over indices (`_HarvestAdminSystem.sol:23–75`) — see the source-vs-live note below |
| `_SnapshotT2System.dropKillRewards` | `onlyAdmin` | Batch OBOL distribution to owner addresses (`_SnapshotT2System.sol:19–29`); passport distribution and whitelist helpers exist only as commented-out code (`_SnapshotT2System.sol:31–47, 57–69`) |
| `KamiMarketCancelSystem.executeAdmin` | `onlyAdmin` | Cancels any active Kami market order (listing, offer, or collection offer) on behalf of its owner (`KamiMarketCancelSystem.sol:31–38`) |
| `TradeCancelSystem.executeAdmin` | `onlyAdmin` | Batch-cancels `PENDING` trades; emits the cancel event with acting account 0 (`TradeCancelSystem.sol:37–45`) |
| `TradeCompleteSystem.executeAdmin` | `onlyAdmin` | Batch-completes `EXECUTED` trades on behalf of each maker (`TradeCompleteSystem.sol:37–48`) |
| `DroptableRevealSystem.forceReveal`, `KamiGachaRevealSystem.forceReveal` | `onlyCommManager` | Recovers commits whose 256-block reveal window lapsed by re-seeding them from the previous block; the intended "window has lapsed" guard is inert, so it also re-seeds unexpired commits — see [commit-reveal.md](commit-reveal.md#force-reveal-community-manager-recovery) (`DroptableRevealSystem.sol:32–42`, `KamiGachaRevealSystem.sol:31–49`, `LibCommit.sol:77–82`) |

> ⚠️ SOURCE ≠ DEPLOYED WORLD (as of 2026-11-20): the row above is the source
> at the pin. The world still runs the earlier build of `_HarvestAdminSystem` —
> the same system address it has had since January 2026; no newer build is
> registered. On that build an admin stop synced the harvest and then stopped
> it **without claiming**: stopping zeroes the harvest balance, so the owner's
> unclaimed output was lost, with no XP, scavenge points, score or harvest
> logs, and `HARVEST_STOP` was emitted with account 0 and amount 0; it also
> did not reset `UPON_COOLDOWN_SET` bonuses. Until the new build is deployed,
> that is what an admin stop does on chain. The entrypoint signatures
> (`stop(uint32)`, `stopBatched(uint32[])`) are the same in both builds.

Registry and config systems (`_*RegistrySystem`, `_ConfigSetSystem`) are also
`onlyAdmin`, but they define game content and parameters rather than mutating
player state; they are covered by the respective mechanic docs.
