# Tax System

> Source: `packages/contracts/src/libraries/LibTax.sol` (L1–103)

## Overview

The tax system provides a generic mechanism for attaching **percentage-based
taxes** to entities. Taxes are defined as entities with a payer (or anchor),
a recipient, and a rate in basis points. The primary use case is harvest taxes,
but the system is designed to support arbitrary tax configurations.

## Tax Entity Shape

| Component | Description |
|---|---|
| `EntityType` | `"TAX"` |
| `IDOwnsTax` | Payer entity ID (e.g., harvest ID) or anchor |
| `IdTarget` | Recipient entity ID |
| `Value` | Tax rate in basis points (max 2000 = 20%) |

Tax ID: `keccak256("tax.instance", payerID, recipientID)`

A single payer can have multiple tax entities (one per recipient). The maximum
tax rate per entity is capped at **20% (2000 basis points)**.

> Source: `LibTax.sol:30–43`

## Bill Calculation

When distributing a taxable amount, the system calculates the tax bill for
all recipients attached to a payer:

```solidity
LibTax.getBillFor(components, originalAmount, payerID)
→ (recipientIDs[], amounts[], amountLeft)
```

For each tax entity:
```
taxAmount = (originalAmount × rate) / 10000
```

Multiple taxes are applied **sequentially** from the original amount (not
compounding). `amountLeft` is the remainder after all taxes are deducted.

> Source: `LibTax.sol:71–95`

## Cap

Maximum tax per entity: **2000 basis points (20%)**. The `create` function
reverts if a rate > 2000 is specified. However, multiple tax entities on the
same payer could theoretically sum to more than 20% total.

> Source: `LibTax.sol:36`

## Usage

The tax system is used by:
- **Harvest system**: Taxes can be attached to harvest instances, diverting a
  portion of harvest output to designated recipients (e.g., node owners,
  factions, community pools)
- **Token portal**: Uses its own tax calculation but follows a similar pattern
  (see [token-portal.md](token-portal.md))

Tax entities can be queried and removed per payer:

```solidity
LibTax.getFor(components, payerID) → taxEntityIDs[]
LibTax.removeFor(components, payerID) // removes all taxes for payer
```

> Source: `LibTax.sol:59–62, 67–69`
