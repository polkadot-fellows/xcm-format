---
Title: InitializeAsset Instruction
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

The proposed change is to introduce a new Instruction called `InitializeAsset`.
The instruction allows foreign chains to initialize fully-backed derivatives. 
Currently, these derivatives are initialized using the `Transact` instruction, or directly on the sovereign chain. 

## Motivation

Currently, the `Transact` instruction is used to initialize derivatives.
The `Transact` instruction requires knowledge of the foreign system to determine which call to make.
The call can differ per system, and in FRAME-based systems, it can depend on different instances of a pallet.
The InitializeAsset instruction abstracts away the need to know this call.
Sending consensus systems can send this instruction to any chain without having to know the structure of the destination chain.

## Specification
The instruction would have the following specification:

```rust
InitializeAsset {
    /// The (concrete) asset ID, relative to the receiver/executor. This is what can
    /// be used as an asset ID when minting or teleporting assets into the chain.
    asset_id: MultiLocation,
    /// The owner location, relative to the receiver/executor. Some implementations
    /// might support all kinds of locations, others might only support
    /// `{ parents: 0, interior: X1(AccountId32 { .. }) }` patterns.
    owner: MultiLocation,
    /// Relevant properties of the asset class/ID to be initialized. The
    /// implementation may require some fields to be `Some` and others `None`.
    properties: AssetProperties,
}

enum AssetProperties {
  Fungible {
    total_supply: Option<u128>,
    min_balance: Option<u128>,
  },
  NonFungible {
    total_supply: Option<u128>
  }
}

enum Error {
    /* snip */
    // The properties parameter is incorrect for the instruction.
    IncorrectProperties,
}
```

The `InitializeAsset` instruction allows for the creation of a single asset. 
The `asset_id` field describes either an asset identity for a fungible asset or a class for a non-fungible asset. 
The fungibility of the asset is based on the `properties` field. 
It can either describe the asset properties of a fungible asset or the properties of a non-fungible asset. 

The `owner` field is a MultiLocation representation of the owner of the to-be-created asset. 
MultiLocations can represent all sorts of consensus systems, from a 32-byte representation of a user account using the `AccountId32` Junction to a governance instance using the `Plurality` Junction. 

The AssetProperties enum describes the asset properties of the to-be-initialized asset.
Depending on the XCVM implementation, different assets can be initialized for the same `InitializeAsset` instruction.
For example, in FRAME-based systems, an asset can be initialized in the `assets` pallet and requires the `min_balance` field to be set, while in EVM-based systems an asset can be initialized as an ERC20 smart contract and requires a total_supply to be set.
Based on the XCVM implementation, some fields are required to be `Some` and others `None`, while some fields might be less strict. 

### Errors
Based on the implementation of the XCVM, the `InitializeAsset` instruction can fail. 
If the XCVM does not support an asset type (fungible or non-fungible) it should throw a `FailedToTransactAsset` error.
When the wrong asset properties are set, an XCVM implementation should throw an  `IncorrectProperties` error.
An XCVM implementation might require a specific origin for the `InitializeAsset` instruction and should throw a `BadOrigin` error otherwise.

### Examples

One of the most common use cases of the `InitializeAsset` instruction is the initialization of derivate assets.
Take the following scenario, for example, where we have two chains (A and B), B is an interior location to A (relay chain <-> parachain relation), and chain A has a fungible asset with asset ID 1 and a minimum balance of 10.
Now we initialize a derivative asset on chain B using the `InitializeAsset` instruction with as owner Chain A.
The instruction will look as follows:
```rust
InitializeAsset {
    asset_id: MultiLocation {parents: 1, interior: X1(GeneralIndex(1))},
    owner: MultiLocation {parents: 1, interior: Here},
    properties: Fungible {
        total_supply: None,
        min_balance: Some(10),
    }
}
```

Now take the previous scenario, and chain A has a non-fungible asset with asset Class 1 and a total supply of 100.
Now we initialize a derivative non-fungible asset on chain B using the `Initialize` instruction with as owner Chain A.
The instruction will look as follows:
```rust
InitializeAsset {
    asset_id: MultiLocation {parents: 1, interior: X1(GeneralIndex(1))},
    owner: MultiLocation {parents: 1, interior: Here},
    properties: NonFungible {
        total_supply: Some(100),
    }
}
```

For a third example, Alice (0x00) initializes a new non-fungible asset with class 3 and a max supply of 5 on the local chain:
```rust
InitializeAsset {
    asset_id: MultiLocation {parents: 0, interior: X1(GeneralIndex(1))},
    owner: MultiLocation {parents: 0, interior: AccountId32 {network: None, id: [0; 32]}}
    properties: NonFungible {
        total_supply: Some(5),
    }
}
```

For a fourth example, a parachain with id 2000 initializes a derivative of their native asset on asset hub parachain 1000:
```rust
InitializeAsset {
    asset_id: MultiLocation {parents: 1, interior: X1(Parachain(2000))},
    owner: MultiLocation {parents: 1, interior: X1(Parachain(2000))},
    properties: Fungible {
        total_supply: None,
        min_balance: Some(10),
    }
}
```

## Security considerations
The XCVM implementation has to check if the origin is allowed to initialize this asset. However, this depends entirely on the implementation.

## Impact
The impact of this proposal is Low. It introduces a new instruction, so XCVM implementations have to be updated. 

## Alternatives
As previously discussed, it is possible to initialize assets using the `Transact` instruction. 

## Questions and open Discussions (optional)

- Currently, the instruction allows for the initialization of a single asset. Should the instruction allow for the initialization of multiple assets?

- What should be the specification of the AssetProperties for both the Fungible and NonFungible variant? 