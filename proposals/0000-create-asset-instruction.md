---
Title: CreateAsset Instruction
Number: 0
Status: Draft
Version: 0
Authors:
 - vstam1
Created: 2023-06-09
Impact: Low
Requires:
Replaces:
---

## Summary

The proposed change is to introduce a new Instruction called `CreateAsset`. The instruction allows foreign chains to create fully-backed derivatives. Currently, these derivatives are created using the `Transact` instruction, or directly on the sovereign chain. 


## Motivation

Currently, the `Transact` instruction is used to create derivatives. The `Transact` instruction requires knowledge of the foreign system to determine which call to make. The call can differ per system, and in FRAME-based systems, it can depend on different instances of a pallet. The CreateAsset instruction abstracts away the need to know this call. Sending consensus systems can send this instruction to any chain without having to know the structure of the destination chain. 

## Specification

The instruction would have the following specification:

```rust
CreateAsset {asset: MultiAsset, owner: MultiLocation}
```

The `CreateAsset` instruction allows for the creation of a single asset. This can be a fungible or non-fungible asset based on the specification of the `asset` field. The MultiAsset can describe a single asset. The `id` field is either an asset identity for a fungible asset or a class for a non-fungible asset. The fun field represents either the amount in the case of a fungible asset or the instance Id for a non-fungible asset. 

The `owner` field is a MultiLocation representation of the owner of the to-be-created asset. MultiLocations can represent all sorts of consensus systems, from a 32-byte representation of a user account using the `AccountId32` Junction to a governance instance using the `Plurality` Junction. 

One of the most common use cases of the `CreateAsset` instruction is the creation of derivate assets. Take the following scenario, for example, where we have two chains (A and B), B is an interior location to A (relay chain <-> parachain relation), and chain A has a non-fungible asset with asset class 1. Now we create a derivative asset on chain B using the `CreateAsset` instruction with as owner Chain A. The instruction will look as follows:
```rust
CreateAsset {
    asset: MultiAsset {
        id: Concrete(MultiLocation {parents: 1, interior: X1(GeneralIndex(1))})
        fun: NonFungible(?) // See open questions
    },
    owner: MultiLocation {parents: 1, interior: Here},
}
```

## Security considerations
The XCVM implementation has to check if the origin is allowed to create this asset. 

## Impact
The impact of this proposal is Low. It introduces a new instruction, so XCVM implementations have to be updated. 

## Alternatives
As previously discussed, it is possible to create assets using the `Transact` instruction. 

## Questions and open Discussions (optional)
- How do we represent the `fun` field in the MultiAsset struct? In most of the FRAME-based systems we currently have, the creation of assets does not require an amount in the case of fungible assets or an instance id in the case of non-fungible assets.

- Currently, the instruction allows for the creation of a single asset. Should the instruction allow for the creation of multiple assets?

- How should we handle metadata?