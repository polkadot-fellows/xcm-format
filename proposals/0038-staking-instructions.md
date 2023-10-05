---
Title: New instructions for staking
Number: 46 
Status: Draft
Version: 0
Authors:
 - Gonçalo Pestana (@gpestana)
 - Ankan (@ank4n)
Created: 2023-10-03
Impact: Trivial
Requires: -
Replaces: -
---

## Summary

This RFC proposed a set of new instructions to be used in the context of cross-consensus staking.

## Motivation

> In this section you should describe why this is a valuable problem to solve. What use cases does
> it support and who will benefit from this change? Why is this solution the best in the space of
> possible solutions?

The staking instructions are meant to define a general interface to perform staking that requires
work across multiple consensus systems. The goal is to standarize a minimal interface for staking
through XCM that can be used in the context of relay-chain staking, system parachain staking and
across the whole ecosystem.

## Specification

> Explain the change or feature as you would to another builder in the ecosystem.
> Describe the syntax and semantics of any new feature. Any new terminology or new named concepts should be defined in this section.
> Explain the design in sufficient detail for somebody familiar blockchain to understand. Consider:
> - Its interaction with other features
> - Corner cases are dissected by example
> Provide some examples of how it will be used.

### `StakeAsset(MultiAssets, Multilocation, Multilocation)` 

Stake assets and nominate a specific multilocation.

Operands:

- `assets: MultiAssets`: The assets to stake.
- `staker: MultiLocation`: The staker multilocation account where the stake will be locked.
- `nominated: MultiLocation`: The multilocation where the stake should be allocated to.

Kind: *Instruction*

Errors: *Fallible*:

- `AssetNotFound`
- `NotEnoughFunds`

#### Examples

> TODO

### `UnstakeAsset(MultiAsset, Multilocation, Multilocation)`

Unstake assets from a specific nomination.

Operands:

- `assets: MultiAssets`: The assets to unstake.
- `staker: MultiLocation`: The staker multilocation account where the stake will be unstaked.
- `nominated: MultiLocation`: The multilocation where the stake unstaked from.

Kind: *Instruction*

Errors: *Fallible*:

- `AssetNotFound`
- `NotEnoughFunds`

#### Examples

> TODO

> How to use in staking chain and relay-chain with nomination pools
> How to use in moonbeam

## Security considerations

- The cross-chain systems that rely on the staking instructions assume their conterparties respect 
 that fund locks/unlocks are processed correctly in the destination chain.

## Impact

- No breaking changes; it requires a bump in the XCM versioning.

## Alternatives

- It is not possible to express the proposed staking instructions with the current XCM messages
 without overreylying on the `Transact` instruction.

