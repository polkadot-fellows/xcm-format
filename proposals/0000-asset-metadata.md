---
Title: XCM Asset Metadata
Number: 00
Status: Draft
Version: 5
Authors:
 - Daniel Shiposha
Created: 2024-01-17
Impact: Low
Requires:
Replaces:
---

## Summary

The proposed change provides a general way of communicating metadata of assets and asset instances between consensus systems via XCM.

Currently, there is no standard, general, and easy way to do so.

Communicating metadata enables new cross-consensus use cases for NFTs and might simplify the registration of foreign assets (both fungible and nonfungible).

## Motivation

Currently, there is no way to communicate metadata of an asset (or an asset instance) via XCM.

The ability to query and modify the metadata is useful for two kinds of entities:
* **Asset collections** (both fungible and nonfungible).

    Any collection has some metadata. For instance, the name of the collection. The standard way of communicating metadata could help with registering foreign assets within a consensus system. Therefore, this RFC could complement the [RFC #35 for initializing fully-backed derivatives](https://github.com/paritytech/xcm-format/pull/35).

* **NFTs** (i.e., asset instances). 

    The metadata is the crucial aspect of any nonfungible object since metadata assigns meaning to such an object. The metadata for NFTs is just as important as the notion of "amount" for fungibles (there is no sense in fungibles if they have no amount).

    An NFT is always a representation of some object. The metadata describes the object represented by the NFT.

    An on-chain logic can interpret the NFT metadata, so NFTs can represent cross-consensus shared data objects (i.e., the metadata could have not only the media meaning but also a utility function within a consensus system). Currently, such communication of metadata is possible only within one consensus system. This RFC proposes making it possible between different systems via XCM.

## Specification

The Asset Metadata is information bound to an asset class (fungible or NFT collection) or an asset instance (an NFT).
The Asset Metadata could be represented differently on different chains (or in other consensus entities).
However, to communicate metadata between consensus entities via XCM, we need a general format so that *any* consensus entity can make sense of such information.

Let's name this format "XCM Asset Metadata".

This RFC proposes:
1. Using key-value pairs as XCM Asset Metadata since it is a general concept useful for both structured and unstructured data. Both key and value can be raw bytes with interpretation up to the communicating entities.

    The XCM Asset Metadata could be represented as a map.
The total space occupied by keys and values should be bounded by a predefined constant (to be determined, see [questions](#questions-and-open-discussions)).

    Let's call the type of the XCM Asset Metadata map `MetadataMap`.

2. Communicating only the demanded part of the metadata, not the whole metadata.

    * A consensus entity could query the values of interested keys to read the metadata.
        To specify the keys to read, we need a set-like type. Let's call that type `MetadataKeys`.
        The `MetadataKeys` set should also be bounded similarly to `MetadataMap`.

    * A consensus entity could write the values for specified keys.

3. New XCM instructions to communicate the metadata.

### New instructions

#### `ReportMetadata`

The `ReportMetadata` is a new instruction to query metadata information.
It can be used to query metadata key list or to query values of interested keys.

This instruction allows querying the metadata of:
* a collection (fungible or nonfungible)
* an NFT

If an asset (or an asset instance) for which the query is made doesn't exist, an error should be reported via the existing `QueryResponse` instruction.

The `ReportMetadata` can be used without origin (i.e., following the `ClearOrigin` instruction) since it only reads state. In particular, it means one can transfer a currency to pay for the execution of this instruction using a reserve-based transfer.

Safety: The reporter origin should be trusted to hold the true metadata. If the reserve-based model is considered, the asset's reserve location must be viewed as the only source of truth about the metadata.

The use case for this instruction is when the metadata information of a foreign asset (or asset instance) is used in the logic of a consensus entity that requested it.

```rust
/// An instruction to query metadata
/// of an asset or an asset instance.
ReportMetadata {
    /// The ID of an asset (a collection, fungible or nonfungible).
    asset_id: AssetId,

    /// The ID of an asset instance. Applicable only to nonfungible assets.
    ///
    /// It is optional and will have the value `Some`
    /// only if the metadata query is related to an NFT.
    ///
    /// If `None`, the metadata of the collection is reported.
    instance: Option<AssetInstance>,

    /// See `MetadataQueryKind` below.
    query_kind: MetadataQueryKind,

    /// The usual field for Report<something> XCM instructions.
    ///
    /// Information regarding the composition of a query response.
    /// The `QueryResponseInfo` type is already defined in the XCM spec.
    response_info: QueryResponseInfo,
}
```

Where the `MetadataQueryKind` is:

```rust
enum MetadataQueryKind {
    /// Query metadata key list.
    KeyList,

    /// Query values of the specified keys.
    Values(MetadataKeys),
}
```

The `QueryMetadata` works in conjunction with the existing `QueryResponse` instruction. The `Response` type should be modified accordingly: we need to add a new `AssetMetadata` variant to it.

```rust
/// The struct used in the existing `QueryResponse` instruction.
pub enum Response {
    // ... snip, existing variants ...

    /// The metadata info.
    AssetMetadata {
        /// The ID of an asset (a collection, fungible or nonfungible).
        asset_id: AssetId,

        /// The ID of an asset instance. Applicable only to nonfungible assets.
        ///
        /// If `Some`, the reported metadata is related to an NFT.
        /// If `None`, the reported metadata is related to the collection.
        instance: Option<AssetInstance>,

        /// See `MetadataResponseKind` below.
        response_kind: MetadataResponseKind,
    }
}

pub enum MetadataResponseKind {
    /// The metadata key list to be reported
    /// in response to the `KeyList` metadata query kind.
    KeyList(MetadataKeys),

    /// The values of the keys that were specified in the
    /// `Values` variant of the metadata query kind.
    Values(MetadataMap),
}
```

#### `ModifyMetadata`

The `ModifyMetadata` is a new instruction to request a remote chain to modify the values of the specified keys.

This instruction can be used to update the metadata of a collection (fungible or nonfungible) or of an NFT.

The remote chain handles the modification request and may reject it based on its internal rules.
The request can only be executed or rejected in its entirety. It must not be executed partially.

To execute the `ModifyMetadata`, an origin is required so that the handling logic can authorize the metadata modification request from a known source. In particular it means the following:
* The `ModifyMetadata` can be used in conjunction with the `DescendOrigin` instruction. So, interior consensus entities of a requester blockchain can also send modification requests (such as Smart Contracts)
* It is unclear how to pay for the execution of this instruction since we can't send a currency using a reserve-based transfer in the same message with the `ModifyMetadata` due to the `ClearOrigin` instruction embedded into the reserve-based transfer pattern. See [questions](#questions-and-open-discussions).

The example use case of this instruction is to ask the reserve location of the asset to modify the metadata. So that, the original asset's metadata is updated according to the reserve location's rules.

```rust
ModifyMetadata {
    /// The ID of an asset (a collection, fungible or nonfungible).
    asset_id: AssetId,

    /// The ID of an asset instance. Applicable only to nonfungible assets.
    ///
    /// If `Some`, the metadata of an NFT is being modified.
    /// Otherwise, the modification request targets the collection.
    instance: Option<AssetInstance>,

    /// The map contains the keys mapped to the requested new values.
    modification: MetadataMap,
}
```

#### Mofication approvals

This RFC proposes two extra auxiliary instructions related to metadata modification.

For instance, the reserve location containing the true metadata of an asset can have the metadata modification logic that checks if the modification is correct.
Such checks could be anything, so we can't express them all in the XCM format.

However, it seems that a basic authorization check could be typical.
Examples:
1. Only an NFT owner can change the values of specific metadata keys
2. The NFT owner could approve other parties to change the values of a subset of the keys
3. Only the collection owner can change the values of specific metadata keys of NFTs within the collection
4. The NFT collection owner approves a Smart Contract to change the values of certain keys of an NFT. The Smart Contract could serve as an additional check logic, so if a user wants to write a value under a key, the user asks the Smart Contract to write it. The SC will verify that the value is correct and only then will write it to the NFT's metadata.

    **Note** that the mentioned Smart Contract (or other similar thing) may be located on a different chain.

A consensus entity could be notified about the fact that it is approved to modify specific keys and react to that in an arbitrary way. For instance, such a notification may give a consensus entity the confidence that its modification requests will likely be served (rather than immediately rejected).

##### `ApproveMetadataModification`

`ApproveMetadataModification` gives an approval (see the `MetadataModificationApproval` type) to a specified origin. The origin will be notified about that with the `NoteMetadataModificationApproval` message.

Similar to the `ModifyMetadata` instruction, to be executed, the `ApproveMetadataModification` requires an origin. The consequences and questions are the same as for the `ModifyMetadata`.

To withdraw the approval, one could execute the `ApproveMetadataModification` with the empty set of allowed keys.

```rust
/// Give an approval to a specific origin to modify certain keys of an asset (or an asset instance).
/// See the `MetadataModificationApproval` type.
ApproveMetadataModification {
    /// The ID of an asset (a collection, fungible or nonfungible).
    asset_id: AssetId,

    /// The ID of an asset instance. Applicable only to nonfungible assets.
    ///
    /// If `Some`, it is about metadata of an NFT.
    /// Otherwise, is is about metadata of the collection.
    instance: Option<AssetInstance>,

    /// The the `MetadataModificationApproval` type.
    approval: MetadataModificationApproval
}
```

##### `NoteMetadataModificationApproval`

The `NoteMetadataModificationApproval` notifies the receiver consensus entity that it (or its interior) is allowed to modify certain metadata keys of an asset (or an asset instance).

Similar to the `ModifyMetadata` instruction, to be executed, the `NoteMetadataModificationApproval` requires an origin. The consequences and questions are the same as for the `ModifyMetadata`.

Safety: The origin must be trusted in the sense that it may grant such approvals.

```rust
NoteMetadataModificationApproval {
    /// The ID of an asset (a collection, fungible or nonfungible).
    asset_id: AssetId,

    /// The ID of an asset instance. Applicable only to nonfungible assets.
    ///
    /// If `Some`, it is about metadata of an NFT.
    /// Otherwise, is is about metadata of the collection.
    instance: Option<AssetInstance>,

    /// The the `MetadataModificationApproval` type.
    ///
    /// The approved origin is included in this instruction
    /// since it could be an interior consensus entity on the receiver chain.
    approval: MetadataModificationApproval,
}
```

Where the `MetadataModificationApproval` is:

```rust
/// The metadata modification approval description.
struct MetadataModificationApproval {
    /// The origin that is approved to make modifications to the specified keys.
    /// The list of the keys approved to be modified by the origin is specified in the `approve_keys` field.
    approved_origin: MultiLocation,

    /// The keys allowed to be modified by the `approved_origin`.
    approved_keys: MetadataKeys,
}
```

## Security considerations

Some of the security considerations were highlighted in the description of the instruction.
Yet, more could be considered during the discussion of the RFC.

## Impact

The impact of this proposal is Low (however, if we consider the change to the `Response` type as a change of the `QueryResponse` instruction, then the impact is High).

The RFC introduces new instructions, so XCVM implementations have to be updated.

## Alternatives

All the proposed instructions could be replaced with `Transact`.
However, without the proposed instructions, there is no standard communication language about metadata in XCM. Using `Transact` for this purpose could be inconsistent and more error-prone.

## Questions and open Discussions

1. Naming of all the proposed instructions and types
2. Is there a way to simplify or make the proposed instructions more convenient?
3. Are the proposed metadata format and operations general enough?
4. Could the proposed instruction set be reduced without losing both generality and convenience?
5. How one could easily pay for the execution of the `ModifyMetadata`, `ApproveMetadataModification`, and `NoteMetadataModificationApproval`?
6. How to make the `MetadataMap` bounded? The same question for `MetadataKeys`.
7. Can the metadata operations be meaningful for fungible **tokens** (not only for fungible collections)?
8. Are there any additional security considerations?
