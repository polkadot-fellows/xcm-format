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

The proposed change is to introduce a new Instruction called `CreateAsset`. The instruction allows foreign chains to create fully-backed derivatives. Currently these derivatives are created using the `Transact` instruction, or directly on the sovereign chain. 


## Motivation

Currently the `Transact` instruction is used to create derivatives. The `Transact` instruction requires knowledge of the foreign system to determine which call to make. The call can differ per system, and in FRAME-based systems can depend on different instances of a pallet. The CreateAsset instruction abstracts the need to know this call away. Sending consensus systems can send this instruction to any chain without having to know the structure of the destination chain. 

## Specification

The instruction would have the following specification:

```rust
CreateAsset {asset: MultiAsset, owner: MultiLocation}
```

The `CreateAsset` instruction allows for the creation of a single asset. This can be a fungible or non fungible asset.

## Security considerations


## Impact
The impact of this proposal is Low. It introduces a new instruction, so XCVM implementations have to be updated. 

## Alternatives
As previously discussed, it is possible to create assets using the `Transact` instruction. 

## Questions and open Discussions (optional)

- Currently the instruction allows for the creation of a single asset. Should the instruction allow for the creation of multiple assets?

- How should we handle metadata?