# Account Customization

> Source: `packages/contracts/src/systems/AccountSetNameSystem.sol` (L1–33),
> `packages/contracts/src/systems/AccountSetBioSystem.sol` (L1–29),
> `packages/contracts/src/systems/AccountSetPFPSystem.sol` (L1–37),
> `packages/contracts/src/systems/AccountSetOperatorSystem.sol` (L1–32)

## Overview

Players can customize their account profile through four systems: name, bio,
profile picture, and operator address.

## Set Name

`AccountSetNameSystem.execute(name)` — must be called by **owner EOA**:

- Name cannot be empty
- Name must be ≤ 16 characters
- Name must be unique (not taken by another account)

> Source: `AccountSetNameSystem.sol:15–28`

## Set Bio

`AccountSetBioSystem.execute(bio)` — called by operator:

- Bio must be ≤ 140 characters

> Source: `AccountSetBioSystem.sol:15–24`

## Set Profile Picture

`AccountSetPFPSystem.execute(kamiID)` — called by operator:

- Copies the `MediaURI` from an owned Kami to the account
- Kami must be owned by the caller's account

> Source: `AccountSetPFPSystem.sol:19–32`

## Set Operator

`AccountSetOperatorSystem.execute(operatorAddress)` — must be called by
**owner EOA**:

- Operator address must not already be in use by another account
- Replaces the previous operator address

The **operator** is a hot wallet that can perform most actions on behalf of the
account, while the **owner** (cold wallet) retains exclusive control over
sensitive operations like naming and operator changes.

> Source: `AccountSetOperatorSystem.sol:15–27`
