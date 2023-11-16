---
Title: Bridge ID for Ethereum Networks
Number: 0
Status: Draft
Version: 0
Authors:
 - Vincent Geddes
Created: 2023-11-16
Impact: Trivial
Requires:
Replaces:
---

## Summary

Upgrade `NetworkId::Ethereum` variant to optionally specify a Bridge ID.

## Motivation

As a convention, assets in foreign consensus systems are generally identified using locations of the form

```
Location {
    parents: 0
    interior: (GlobalConsensus(NetworkId), ...)
}
```

For example, in the relative context of the Polkadot network, a smart contract on Ethereum can be identified using this location:

```
Location {
    parents: 0
    interior: (GlobalConsensus(NetworkId), AccountKey20(0xf1d2d2f924e986ac86fdf7b36c94bcdf32beec15))
}
```

At first glance, this seems reasonable. However there is missing information. Foreign networks at the global level always have their consensus exported to Polkadot through the use of bridging technology.

Assets exported through different bridges are not fungible with one another, for many reasons. So there must be a means to discriminate foreign assets based on the bridge that provided them.

## Specification

Apply the following changes in XCMv4:

1. Add `ExporterId` type:

```rust
pub struct ExporterId([u8; 32])
```

2. Update `NetworkId::Ethereum` variant as to include an optional `exporter_id` field:

```
	/// An Ethereum network specified by its chain ID.
	Ethereum {
		/// The EIP-155 chain ID.
		#[codec(compact)]
		chain_id: u64,
        exporter_id: Some(ExporterId)
    }
```

The `ExporterId` type wraps an uninterpreted 32-byte identifier for a bridge. The XCM format, as an abstract specification, does not bless any concrete bridging implementations. Therefore, an uninterpreted ID is more appropriate.

## Security considerations

* The change affects the ABI of XCM and so careful attention needs to paid to versioning.

* Parachains dealing with synthetic Ethereum assets from multiple bridges need to pay careful attention to the `ExporterId` as these assets are not fungible.

## Impact

This changes the ABI of the `NetworkId` type and all types that rely on it. As such, it is breaking change that will require storage upgrades and so on.

So we should include it in XCMv4, if accepted

## Alternatives

- What other designs have been considered and what is the rationale for not choosing them?

We have considered putting the bridge ID in `Location.interior` as a junction, such as `GeneralKey`, or `GeneralIndex`. However this is a _hack_, since the Bridge is not really a consensus system on Ethereum.

Therefore this approach subverts the meaning and intent of the `Location` type.

Similarly, we have also thought about prepending into the location the address of our bridge's _gateway_ contract on Ethereum. Again, this also a _hack_, since it implies that our gateway contract is the parent consensus system of all other consensus systems on Ethereum, which it certainly is not.

- What is the impact of not doing this?

Snowbridge is currently intending to register wrapped Ethereum tokens in the `ForeignAssets` pallet of AssetHub. We intend to reserve the `NetworkId::Ethereum` junction for our assets, within the context of AssetHub, and for other parachains that wish to reserve-transfer assets from the AssetHub.ForeignAssets pallet.

Specific ethereum bridges that come online in future will need to request a new NetworkId variant for their bridge. These bridges must have the following properties:
1. The bridge wants to use the XCM `ExportMessage` message as its core bridging interface
2. The bridge wants to identify its assets using the `GlobalConsensus(Ethereum)` prefix.

Other existing Ethereum bridges (Wormhole, ChainBridge, etc) are not affected since they don't conform to (1) or (2).
