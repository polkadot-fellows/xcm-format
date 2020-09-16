# Polkadot Cross-Consensus Message (XCM) Format

## Version 0, draft

This page details the message format for Polkadot-based message passing between chains. It describes the formal data format, any environmental data which may be additionally required and the corresponding meaning of the datagrams.

## Background

There are several kinds of *Consensus Systems* for which it would be advantageous to facilitate communication. This includes messages between smart-contracts and their environment, messages between sovereign blockchains over bridges, and between shards governed by the same consensus. Unfortunately, each tends to have its own message-passing means and standards, or have no standards at all.

XCM aims to abstract the typical message intentions across these systems and provide a basic framework for forward-compatible, extensible and practical communication datagrams facilitating typical interactions between disparate datasystems within the world of global consensus.

Concepts from the IPFS project, particularly the idea of self-describing formats, are used throughout and two new self-describing formats are introduced for specifying assets (`MultiAsset`) and consensus-system locations (`MultiLocation`).

Polkadot has three main message passing systems all of which will use this format: XCMP and the two kinds VMP (UMP and DMP).

- **XCMP** *Cross-Chain Message Passing* secure message passing between parachains. There are two variants: *Direct* and *Relayed*.
  - With *Direct*, message data goes direct between parachains and is O(1) on the side of the Relay-chain and is very scalable.
  - With *Relayed*, message data is passed via the Relay-chain, and piggy-backs over VMP. It is much less scalable, and parathreads in particular may not receive messages due to excessive queue growth.
- **VMP** *Vertical Message Passing* message passing between the Relay-chain itself and a parachain. Message data in both cases exists on the Relay-chain. This includes:
  - **UMP** *Upward Message Passing* message passing from a parachain to the Relay-chain.
  - **DMP** *Downward Message Passing* message passing from the Relay-chain to a parachain.

### XCM Communication Model

XCM is designed around four 'A's:

- *Asynchronous*: XCM messages in now way assume that the sender will be blocking on its completion.
- *Absolute*: XCM messages are guaranteed to be delivered and interpreted accurately, in order and in a timely fashion.
- *Asymmetric*: XCM messages do not have results. Any results must be separately communicated to the sender with an additional message.
- *Agnostic*: XCM makes no assumptions about the nature of the Consensus System between which messages are being passed.

The fact that XCM gives these *Absolute* guarantees allows it to be practically *Asymmetric* whereas other non-Absloute protocols would find this difficult.

Being *Agnostic* means that XCM is not simply for messages between parachain(s) and/or the Relay-chain, but rather that XCM is suitable for messages between disparate chains connected through one or more bridge(s) and even for messages between smart-contracts. Using XCM, all of the above may communicate with, or through, each other.

E.g. It is entirely conceivable that, using XCM, a smart contract, hosted on a Polkadot parachain, may transfer a non-fungible asset it owns through Polkadot to an Ethereum-mainnet bridge located on another parachain, into an account controlled on the Ethereum mainnet by registering the transfer of ownership on a third, specialised Substrate NFA chain hosted on a Kusama parachain via a Polkadot-Kusama bridge.

## Definitions

- *Consensus System* A chain, contract or other global, encapsulated, state machine singleton. It can be any programmatic state-transition system that exists within consensus which can send/receive datagrams. May be specified by a `MultiLocation` value (though not all such values identify a *Consensus System*). Examples include *The Polkadot Relay chain*, *The XDAI Ethereum PoA chain*, *The Ethereum Tether smart contract*.
- *Location* A *Consensus System*, or an addressable account or datastructure that exists therein. Examples include the Treasury account on the Polkadot Relay-chain, the primary Web3 Foundation account on the Edgeware parachain, the Edgeware parachain itself, the Web3 Foundation's Ethereum multisig wallet account. Specified by a `MultiLocation`.
- *Sovereign Account* An account controlled by a particular *Consensus System*, within some other *Consensus System*. There may be many such accounts or just one. If many, then this assumes and identifies a unique *primary* account.
- *Holding Account* A transient notional "account" in which assets inherent in a message are temporarily held.
- *Reserve Location* The *Consensus System* which acts as the reserve for a particular assets on a particular (derivative) *Consensus System*. The reserve *Consensus System* is always known by the derivative. It will have a *Sovereign Account* for the derivative which contains full collateral for the derivative assets.
- *Origin* The *Consensus System* from which a given message has been (directly and immediately) delivered. This is always queryable by the receiving code using the message-passing protocol. Specified as a `MultiLocation`.
- *Recipient* The *Consensus System* to which a given message has been delivered. Specified as a `MultiLocation`.
- *Teleport* Destroying an asset (or amount of funds/token/currency) in one place and minting a corresponding amount in a second place. Imagine the teleporter from *Star Trek*. The two places need not be equivalent in nature (e.g. could be a UTXO chain that destroys assets and an account-based chain that mints them). Neither place acts as a reserve or derivative for the other. Though the nature of the tokens may be different, neither place is more canonical than the other. This is only possible if there is a bilateral trust relationship both of the STF and the validity/finality/availability between them.
- *Transfer* The movement of funds from one controlling authority to another. This is within the same chain or overall asset-ownership environment and at the same abstraction level.

## Basic Top-level Format

All data is SCALE encoded. We name the top-level XCM datatype `VersionedXcm`. Generally, this is defined thus:

- `version: u8`: Version of XCM; only zero supported currently.
- `message: Xcm`: The message; opaque unless version is known and well-defined.

### Xcm Version 0

The first version of XCM, 0, is defined properly thus:

- `version: u8 = 0`: Version of XCM; only zero supported currently.
- `message: Xcm`

The message, which amounts to the "important part" is simply named `Xcm`. It is defined thus:

- `type: u8`: Message type.
- `payload`: Message parameters.

Where message `type` must be one of:

- `0`: `WithdrawAsset`
- `1`: `ReserveAssetDeposit`
- `2`: `TeleportAsset`
- `3`: `Balances`
- `4`: `Transact`
- `5`: `RelayTo`
- `6`: `RelayedFrom`

Within XCM, there is an internal datatype `Order`, which encodes an operation on the holding account. It is defined as:

- `type: u8`: Instruction type.
- `payload`: Instruction parameters.

It should be enumerated thus:

- `0`: `Null`
- `1`: `DepositAsset`
- `2`: `DepositReserveAsset`
- `3`: `ExchangeAsset`
- `4`: `InitiateReserveWithdraw`
- `5`: `InitiateTeleport`
- `6`: `QueryHolding`

## `Xcm` Message Types

![image](https://user-images.githubusercontent.com/138296/85861827-38ef4f00-b7c1-11ea-8a2d-32cacbcef782.png)

_Basic interaction diagram. Boxes are blockchains, cylinders are accounts, red arrow over an account is a debit, green arrow is a credit. Circled accounts are Holding Accounts. DA is `DepositAsset`, EA is `ExchangeAsset`, RAX is `ReserveAssetTransfer`, RAC is `ReserveAssetCredit`, TA is `TeleportAsset` and WA is `WithdrawAsset`_

### `WithdrawAsset`

An instructive message commanding the removal of some asset(s) owned by the *Origin* into holding.

Parameter(s):

- `assets: Vec<MultiAsset>` The asset(s) to be withdrawn.
- `effect: Vec<Order>` What should be done with the assets.

### `ReserveAssetDeposit`

A notification message that the *Origin* has received `assets` into a *Sovereign* account controled by the *Recipient*. The `asset` should be minted into the *Holding Account* and some `effect` evaluated on it.

Parameter(s):

- `assets: Vec<MultiAsset>` The asset(s) that were transfered.
- `effect: Vec<Order>` What should be done with the assets.

### `TeleportAsset`

Some `assets` have been removed from existence (and ownership by `source`) on the *Origin* and should be minted into the holding account on the *Recipient* and some `effect` evaluated.

Parameter(s):

- `assets: Vec<MultiAsset>` The asset(s) which were debited.
- `effect: Vec<Order>` What should be done with the assets.

### `Balances`

Informational message detailing some balances, interpreted based on the context of the destination and the `query_id`.

- `query_id` The identifier of the query which caused this message to be sent.
- `assets` The value for use by the destination.

### `Transact`

Apply the encoded transaction `call`, whose dispatch-origin should be `origin` as expressed by the kind of origin `origin_type`.

- `origin_type`: The means of expressing the message origin as a dispatch origin.
- `call`: The encoded transaction to be applied.

Safety: No concerns.

Kind: *Instruction*.

Errors:

### `RelayTo`

Relay an inner message (`inner`) to a locally reachable destination ID `dest`.

The message sent to the destination will be wrapped into a `RelayedFrom` message, with the `superorigin` being this location.

- `dest: MultiLocation`: The location of the to be relayed into. This may never contain `Parent`, and it must be immediately reachable from the interpreting context.
- `inner: VersionedXcm`: The message to be wrapped and relayed.

Safety: No concerns.

Kind: *Instruction*.

Errors:

### `RelayedFrom`

A message (`inner`) was sent to `origin` from `superorigin` with the intention of being relayed.

- `superorigin: MultiLocation`: The location of the `inner` message origin, **relative to `origin`**.
- `inner: VersionedXcm`: The message sent by the super origin.

Safety: `superorigin` must express a sub-consensus only; it may *NEVER* contain a `Parent` junction.

Kind: *Trusted Indication*.

Errors:

## `Order` Types

### `Null`

Do nothing. Exactly equivalent outcome to not executing this instruction.

### `DepositAsset`

Remove the asset(s) (`assets`) from holding and place equivalent assets under the ownership of `dest` within this consensus system.

- `assets: Vec<MultiAsset>`: The asset(s) to remove from holding.
- `dest: MultiLocation`: The new owner for the assets.

Errors:

### `DepositReserveAsset`

Remove the asset(s) (`assets`) from holding and place equivalent assets under the ownership of `dest` within this consensus system.

Send an onward XCM message to `dest` of `ReserveAssetDeposit` with the

- `assets: Vec<MultiAsset>`: The asset(s) to remove from holding.
- `dest: MultiLocation`: The new owner for the assets.
- `effects: Vec<Order>`: The orders that should be contained in the `ReserveAssetDeposit` which is sent onwards to `dest`.

Errors:

### `ExchangeAsset`

Remove the asset(s) (`give`) from holding and replace them with alternative assets.

The minimum amount of assets to be received into holding for the order not to fail may be stated.

- `give: Vec<MultiAsset>`: The asset(s) to remove from holding.
- `receive: Vec<MultiAsset>`: The minimum amount of assets(s) which `give` should be exchanged for. The meaning of wildcards
  is undefined and they should be not be used.

Errors:

### `InitiateReserveWithdraw`

Remove the asset(s) (`assets`) from holding and send a `WithdrawAsset` XCM message to a reserve location.

- `assets: Vec<MultiAsset>`: The asset(s) to remove from holding.
- `reserve: MultiLocation`: A valid location that acts as a reserve for all asset(s) in `assets`. The sovereign account
  of this consensus system *on the reserve location* will have appropriate assets withdrawn and `effects` will
  be executed on them. There will typically be only one valid location on any given asset/chain combination.
- `effects: Vec<Order>`: The orders to execute on the assets once withdrawn *on the reserve location*.

Errors:

### `InitiateTeleport`

Remove the asset(s) (`assets`) from holding and send a `TeleportAsset` XCM message to a destination location.

- `assets: Vec<MultiAsset>`: The asset(s) to remove from holding.
- `dest: MultiLocation`: A valid location that has a bi-lateral teleportation arrangement.
- `effects: Vec<Order>`: The orders to execute on the assets once arrived *on the destination location*.

Errors:

### `QueryHolding`

Send a `Balances` XCM message with the `assets` value equal to the holding contents, or a portion thereof.

- `query_id: u64 (Compact)`: An identifier that will be replicated into the returned XCM message.
- `dest: MultiLocation`: A valid destination for the returned XCM message. This may be limited to the current origin.
- `assets: Vec<MultiAsset>`: A filter for the assets that should be reported back. The assets reported back will be, asset-wise, *the lesser of this value and the holding account*. No wildcards will be used when reporting assets back.

Errors:

### `Remark`

A no-op in XCM terms, but may be used as an "event hook" on the recipient implementation in order to allow chains to receive notifications that an action has completed.

- `note: <Vec<u8> (Compact)`: Arbitrary data to include in the remark.

## `MultiAsset`: Universal Asset Identifiers

*Note on versioning:* This is the `MultiAsset` as used in XCM version 0. If `MultiAsset` is used outside of an XCM message, then it should be placed inside a versioned container `VersionedMultiAsset`, exactly analagous to how `Xcm` is placed inside `VersionedXcm`.

### Description

A `MultiAsset` is a single, general identifier for an asset.

It may represent both fungible and non-fungible assets. A single `MultiAsset` value may only be used to represent a single asset class, so it tends to be used within a `Vec`.

There are `MultiAsset` values which imply some sort of wildcard matching; this may or may not be allowed by the interpreting context.

Assets classes may be identified in one of two ways: either an abstract identifier or a concrete identifier. A single asset may be referenced from multiple asset identifiers, though will tend to have only a single *preferred* identifier.

#### Abstract identifiers

Abstract identifiers are absolute identifiers that represent a notional asset which can exist within multiple consensus systems. These tend to be simpler to deal with since their broad meaning is unchanged regardless stay of the consensus system in which it is interpreted.

However, in the attempt to provide uniformity across consensus systems, they may conflate different instantiations of some notional asset (e.g. the reserve asset and a local reserve-backed derivative of it) under the same name, leading to confusion. It also implies that one notional asset is accounted for locally in only one way. This may not be the case, e.g. where there are multiple bridge instances each providing a bridged "BTC" token yet none being fungible between the others.

Since they are meant to be absolute and universal, a global registry is needed to ensure that name collisions do not occur.

An abstract identifier is represented as a simple variable-size byte string. As of writing, no global registry exists and no proposals have been put forth for asset labeling.

#### Concrete identifiers

Concrete identifiers are *relative identifiers* that specifically identify a single asset through its location in a consensus system relative to the context interpreting. Use of a `MultiLocation` ensures that similar but non-fungible variants of the same underlying asset can be properly distinguished, and obviates the need for any kind of central registry.

The limitation is that the asset identifier cannot be trivially copied between consensus systems and must instead be "re-anchored" whenever being moved to a new consensus system, using the two systems' relative paths.

Throughout XCM, messages are authored such that *when interpreted from the receiver's point of view* they will have the desired meaning/effect. This means that relative paths should always by constructed to be read from the point of view of the receiving system, *which may be have a completely different meaning in the authoring system*.

Concrete identifiers are the generally preferred way of identifying an asset since they are entirely unambiguous.

A concrete identifier is represented by a `MultiLocation`. If a system has an unambiguous primary asset (such as Bitcoin with BTC or Ethereum with ETH), then it will conventionally be identified as the chain itself. Alternative and more specific ways of referring to an asset within a system include:

- `<chain>/PalletInstance(<id>)` for a Frame chain with a single-asset pallet instance (such as an instance of the Balances pallet).
- `<chain>/PalletInstance(<id>)/GeneralIndex(<index>)` for a Frame chain with an indexed multi-asset pallet instance (such as an instance of the Assets pallet).
- `<chain>/AccountId32` for an ERC-20-style single-asset smart-contract on a Frame-based contracts chain.
- `<chain>/AccountKey20` for an ERC-20-style single-asset smart-contract on an Ethereum-like chain.

### Format

Given by the SCALE `enum` (tagged union) of:

- `None = 0`: No assets. Rarely used.
- `All = 1`: All assets. Typically used for the subset of assets to be used for an `Order`, and in that context means "all assets currently in holding". Sometimes written with the shorthand `*`.
- `AllFungible = 2`: All fungible assets. Typically used for the subset of assets to be used for an `Order`, and in that context means "all fungible assets currently in holding".
- `AllNonFungible = 3`: All non-fungible assets. Typically used for the subset of assets to be used for an `Order`, and in that context means "all non-fungible assets currently in holding".
- `AllAbstractFungible = 4: { id: Vec<u8> }`: All fungible assets of a given abstract asset `id`entifier.
- `AllAbstractNonFungible = 5: { class: Vec<u8> }`: All non-fungible assets of a given abstract asset `class`.
- `AllConcreteFungible = 6: { id: MultiLocation }`: All fungible assets of a given concrete asset `id`entifier.
- `AllConcreteNonFungible = 7: { class: MultiLocation }`: All non-fungible assets of a given concrete asset `class`.
- `AbstractFungible = 8: { id: Vec<u8>, amount: Compact }`: Some specific `amount` of the fungible asset identified by an abstract `id`.
- `AbstractNonFungible = 9: { class: Vec<u8>, instance: AssetInstance }`: Some specific `instance` of the non-fungible asset whose `class` is identified abstractly.
- `ConcreteFungible = 10: { id: MultiLocation, amount: Compact }`: Some specific `amount` of the fungible asset identified by an concrete `id`.
- `ConcreteNonFungible = 11: { class: MultiLocation, instance: AssetInstance }`: Some specific `instance` of the non-fungible asset whose `class` is identified concretely.

## `AssetInstance`

A general identifier for an instance of a non-fungible asset class.

Given by the SCALE `enum` (tagged union) of:

- `Undefined = 0`: Undefined - used if the NFA class has only one instance.
- `Index = 1: { index: Compact }`: A compact `index`. Technically this could be greater than u128, but this implementation supports only values up to `2**128 - 1`.
- `Array4 = 2: { datum: [u8; 4] }`: A 4-byte fixed-length `datum`.
- `Array8 = 3: { datum: [u8; 8] }`: A 8-byte fixed-length `datum`.
- `Array16 = 4: { datum: [u8; 16] }`: A 16-byte fixed-length `datum`.
- `Array32 = 5: { datum: [u8; 32] }`: A 32-byte fixed-length `datum`.
- `Blob = 6: { data: Vec<u8> }`: An arbitrary piece of `data`. Use only when necessary.

## `MultiLocation`: Universal Destination Identifiers

*Note on versioning:* This is the `MultiLocation` as used in XCM version 0. If `MultiLocation` is used outside of an XCM message, then it should be placed inside a versioned container `VersionedMultiLocation`, exactly analagous to how `Xcm` is placed inside `VersionedXcm`.

### Description

A relative path between state-bearing consensus systems.

`MultiLocation` aims to be sufficiently abstract in meaning and general in nature that it is able to identify arbitrary logical "locations" within the world of consensus systems.

A location in a consensus system is defined as an *isolatable state machine* held within global consensus. The location in question need not have a sophisticated consensus algorithm of its own; a single account within Ethereum, for example, could be considered a location.

A very-much non-exhaustive list of types of location include:

- A (normal, layer-1) block chain, e.g. the Bitcoin mainnet or a parachain.
- A layer-0 super-chain, e.g. the Polkadot Relay chain.
- A layer-2 smart contract, e.g. an ERC-20 on Ethereum.
- A logical functional component of a chain, e.g. a single instance of a pallet on a Frame-based Substrate chain.
- An account.

A `MultiLocation` is a *relative identifier*, meaning that it can only be used to define the relative path between two locations, and cannot generally be used to refer to a location universally. It is comprised of a number of *junctions*, in order, each morphing the previous location, either diving down into one of its internal locations, called a *sub-consensus*, or going up into its parent location. Correct `MultiLocation` values must have all `Parent` junctions as a prefix to all *sub-consensus* junctions.

A `MultiLocation` value with no junctions simply refers to the "current" interpreting consensus system.

Note: `MultiLocation`s will tend to be written using junction names delimited by slashes, evoking the syntax of other logical path systems such as URIs and filesystems. E.g. a `MultiLocation` value expressed as `../PalletInstance(3)/GeneralIndex(42)` would be a `MultiLocation` of three `Junction`s: `Parent`, `PalletInstance{index: 3}` and `GeneralIndex{index: 42}`.

### Junctions

A `Junction` is encoded as the tagged union of:

- `Parent = 0`: The consensus system of which the context is a member and state-wise super-set. Note: This item is *not* a sub-consensus item: a consensus system may not identify itself trustlessly as a location that includes this junction. Sometimes written with the shorthand `..`

- `Parachain = 1 { index: Compact<u32> }`: An indexed parachain belonging to and operated by the context. Generally used when the context is a Polkadot Relay-chain.

- `AccountId32 = 2 { network: NetworkId, id: [u8; 32] }`: A 32-byte `id`entifier for an account of a specific `network` that is respected as a sovereign endpoint within the context. Generally used when the context is a Substrate-based chain.

- `AccountIndex64 = 3 { network: NetworkId, index: Compact<u64> }`: An 8-byte `index` for an account of a specific `network` that is respected as a sovereign endpoint within the context. May be used when the context is a Frame-based chain and includes e.g. an indices pallet. The `network` id may be used to restrict or qualify the meaning of the index and may not refer specifically to a *blockchain* network. An example might be a smart contract chain which uses different `network` values to distinguish between account indices and smart contracts indices.

- `AccountKey20 = 4 { network: NetworkId, key: [u8; 20] }`: A 20-byte identifier `key` for an account of a specific `network` that is respected as a sovereign endpoint within the context. May be used when the context is an Ethereum or Bitcoin chain or smart-contract.

- `PalletInstance = 5 { index: u8 }`: An instanced, `index`ed pallet that forms a constituent part of the context. Generally used when the context is a Frame-based chain.

- `GeneralIndex = 6 { index: Compact }`: A non-descript `index` within the context location. Usage will vary widely owing to its generality. Note: Try to avoid using this and instead use a more specific item.

- `GeneralKey = 7 { key: Vec<u8> }`: A nondescript datum acting as a `key` within the context location. Usage will vary widely owing to its generality. Note: Try to avoid using this and instead use a more specific item.

- `OnlyChild = 8`: The unambiguous child.

## NetworkId

A global identifier of an account-bearing consensus system.

Encoded as the tagged union of:

- `Any = 0`: Unidentified/any.
- `Named = 1 { name: Vec<u8> }`: Some `name`d network.
- `Polkadot = 2`: The Polkadot Relay chain
- `Kusama = 3`: Kusama.

## Examples

### Foreign account exchange

Attempts to exchange 42 DOT for 21 BTC. `H` may check result by watching for the relevant `Remark` and check the holding account.

- `H` Home chain.
- `X` Exchange chain on which `H` has holdings.

The chains are assumed to have a common parent chain.

```
X.WithdrawAsset {
    assets: [ 42 DOT ],
    effects: [
        ExchangeAsset { give: *, receive: [ 21 BTC ] }
        DepositReserveAsset {
            dest: ../H,
            assets: *,
            effects: Remark()
        }
    ]
)
```

### Transfer via teleport

Two peer chains that trust each other's STF use the teleport functionality to transfer 21 aUSD from Alice on the home chain to Bob on the other.

- `H` Home chain.
- `D` Destination chain.

The chains are assumed to have a common parent chain.

```
Alice.WithdrawAsset {
    assets: [ 21 aUSD ],
    effects: [
        InitiateTeleport {
            assets: *,
            dest: ../D,
            effects: DepositAsset {
                dest: Bob,
                assets: *,
            }
        }
    ]
}
```

No fees are paid (we assume they're managed elsewhere).

### Transfer via reserve

Two peer chains that trust a third (reserve) chain's STF use the transfer functionality to transfer 21 DOT from Alice on the home chain to Bob on the other.

- `H` Home chain.
- `D` Destination chain.
- The Relay-chain; this is their common parant and it acts as a reserve.

```
Alice.WithdrawAsset {
    assets: [ 21 DOT ],
    effects: [ InitiateReserveWithdraw {
        assets: *,
        reserve: ..,
        effects: [ DepositReserveAsset {
            assets: *,
            dest: D,
            effects: [ DepositAsset {
                dest: Bob,
                assets: *
            } ]
        } ]
    } ]
)
```

This will result in the sovereign account of `H` on the Relay being reduced by 21 DOT (together with the fee) and that of `D` being credited with 21 DOT. A message will be sent by the Relay to `D`:

```
ReserveAssetDeposit {
    assets: [ 21 DOT ],
    effects: [ DepositAsset {
        dest: Bob,
        effects: *
    } ]
}
```
