# Polkadot Cross-Consensus Message (XCM) Format

This page details the message format for Polkadot-based message passing between chains. It describes the formal data format, any environmental data which may be additionally required and the corresponding meaning of the datagrams.

## Background

There are several kinds of consensus systems for which it would be advantageous to facilitate communication. This includes messages between smart-contracts and their environment, messages between sovereign blockchains over bridges, and between shards governed by the same consensus. Unfortunately, each tends to have its own message-passing means and standards, or have no standards at all.

XCM aims to abstract the typical message intentions across these systems and provide a basic framework for forward-compatible, extensible and practical communication datagrams facilitating typical interactions between disparate datasystems within the world of global consensus.

Concepts from the IPFS project, particularly the idea of self-describing formats, are used throughout and two new self-describing formats are introduced for specifying assets (`MultiAsset`) and consensus-message processing locations (`MultiDest`).

Polkadot has three main message passing systems all of which will use this format: XCMP and the two kinds VMP (UMP and DMP).

- **XCMP** *Cross-Chain Message Passing* highly scalable message passing between parachains with minimal work on the side of the Relay-chain.
- **VMP** *Vertical Message Passing* message passing between the Relay-chain itself and a parachain. This includes:
  - **UMP** *Upward Message Passing* message passing from a parachain to the Relay-chain.
  - **DMP** *Downward Message Passing* message passing from the Relay-chain to a parachain.

In addition, a third "composite" message passing system is named as **HRMP** *Horizontal Relay-routed Message Passing*. It is implemented through utilising the two routing meta-messages of XMP (`RMP` and `PRM`) so that parachains may send messages between each other before XCMP is finalised.

In additional to messages between parachain(s) and/or the Relay-chain, XCM is suitable for messages between disparate chains connected through one or more bridge(s) and even for messages between smart-contracts. Using XCM, all of the above may communicate with, or through, each other. e.g. It is entirely conceivable that, using XCM, a smart contract, hosted on a Polkadot parachain, may transfer a non-fungible asset it owns through Polkadot to an Ethereum-mainnet bridge located on another parachain, into an account controlled on the Ethereum mainnet by registering the transfer of ownership on a third, specialised Substrate NFA chain hosted on a Kusama parachain via a Polkadot-Kusama bridge.

## Definitions

- *Sovereign Account* An account controlled solely by a particular UDI.
- *Origin* The chain, contract or other global, encapsulated, state machine singleton from which a given message has been (directly and immediately) delivered. This is always queryable by the receiving code using the message-passing protocol. Generally specified as a *UDI*
- *Recipient* The chain, contract or other global, encapsulated, state machine singleton to which a given message has been delivered.
- *Teleport* Destroying an asset (or amount of funds/token/currency) in one place and minting a corresponding amount in a second place. Imagine the teleporter from *Star Trek*. The two places need not be equivalent in nature (e.g. could be a UTXO chain that destroys assets and an account-based chain that mints them). Neither place acts as a reserve or derivative for the other. Though the nature of the tokens may be different, neither place is more canonical than the other. This is only possible if there is a bilateral trust relationship between them.
- *Transfer* The movement of funds from one controlling authority to another. This is within the same chain or overall asset-ownership environment and at the same abstraction level.
- *Universal Destination Indicator (UDI)* A chain, contract or global, encapsulated, state machine singleton, or an addressable datastructure that exists therein. Examples include an account on the Relay-chain, an account on a parachain, a parachain, an account managed by a smart-contract in a parachain. It can be any programmatic state-transition system that exists within consensus which can send/receive datagrams.

## Basic Top-level Format

All data is SCALE encoded. We name the top-level XCM datatype `Xcm`.

- `magic: [u8; 2] = 0xff00`: Prefix identifier.
- `version: Compact<u32> = 0u32`: Version of XCM; only zero supported currently.
- `type: Compact<u32>`: Message type.
- `payload`: Message parameters.

For version 0, message `type` must be one of:

- `0u32`: `FAX`
- `1u32`: `FAC`
- `2u32`: `TA`
- `3u32`: `RMP`
- `4u32`: `PRM`

## Message Types

### FAX: Foreign/reserve Asset Transfer

An instructive message commanding the transfer of some asset(s) from the (presumed unique or otherwise primary) account owned by the *Origin* to some other destination on the *Recipient*.

`Asset`s (with defaults governed by the *Recipient*) should be transfered from the *Sovereign Account* of the *Origin* to the account identified by a universal `destination` identifier within the context of the *Recipient*.

Parameter(s):

- `asset: MultiAsset` The asset(s) to be transfered.
- `destination: MultiDest` A universal destination identifier which identifies the account/owner/controller on the *Recipient* to be credited. A type 3 (multi-level) ID indicates that an FAC message is needed.
- `source: MultiDest` UDI identifing the true source of the transfer, in terms of the `Origin`. Null indicates the `Origin` itself.

### FAC: Foreign/reserve Asset Credit

A notification message that the *Origin*, acting as a *Reserve*, has received funds into a client account owned by the *Recipient*. It is instructive only in so much as to dictate to the receiving chain the associated destination to which the recipient chain may attribute the credit. The funds are specified in the native currency of the *Origin*.

In other words: `Asset`s (with defaults governed by the *Origin*) have been credited to the *Sovereign Account* of the *Recipient* and a `destination` identifier is provided for the *Recipient* to credit within its own context.

Parameter(s):

- `asset: MultiAsset` The asset(s) to be transfered.
- `destination: MultiDest` A universal destination ID which identifies the sub-account to be credited within the context of the *Recipient*.
- `source: MultiDest` UDI identifing the true source of the transfer, in terms of the `Origin`. Null indicates the `Origin` itself.

### `RMP` Relay Message Parachain

An instructive message to indicate that a given message should be relayed to a further *Destination Chain*. The actual message presented to the *Destination Chain* will be of type `Parachain Relayed Message` and properly present the original sender in it.

Parameter(s):

- `destination: ParaId` The chain index to which this `message` should be relayed within a `PRM`.
- `message: Xcm` The message to be interpreted by the *Recipient*.

### `PRM` Parachain Relay Message

A counterpart message to RMP, this is what is sent by the Relayer to the `destination` parachain mentioned in the RMP message. It is instructive only of the fact that an RMP with `message` and a `destination` equal to the *Receiving Parachain* was sent by the `source` parachain of the *Origin Relay-chain*.

Parameter(s):

- `source: ParaId` The chain index from which this `message` has been relayed from an `RMP`.
- `message: Xcm` The message to be interpreted by the *Recipient*.

### `TA` Teleport Asset

An `amount` of some fungible asset identified by an opaque datagram `asset_id` have been removed from existence on the *Origin* and should be credited on the *Recipient* into the account identified by a universal `destination` identifier.

Parameter(s):

- `asset: MultiAsset` 
- `destination: MultiDest` A universal destination identifier which identifies the account/owner/controller on the *Recipient* to be credited.
- `source: MultiDest` UDI identifing the true source of the transfer, in terms of the `Origin`. Null indicates the `Origin` itself.

## `MultiAsset`: Universal Asset Identifiers

Basic format:

- `version: Compact<u32>`: Version/format code; currently two codes are supported `0x00`, and `0x01`.

### Fungible assets

- `version: Compact<u32> = 0x00`
- `id: Vec<u8>` The fungible asset identifier, usually derived from the ticker-tape code (e.g. b"BTC", b"ETH", b"DOT"). Empty indicates a default value should be used (defined by the evaluation context), if any.
- `amount: Compact<u128>` The amount of the asset identified.

### Non-fungible assets

- `version: Compact<u32> = 0x01`
- `class: Vec<u8>` The general non-fungible asset class code. Empty indicates a default value should be used (defined by the evaluation context), if any.
- `instance: AssetInstance` The general non-fungible asset instance within the NFA class. May be identified with with a numeric index or a datagram. Most `class`es will support only a single specific kind of `AssetInstance`, however for ease of formatting and to facilitate future compatibility, it is self-describing.

#### `AssetInstance`

Given by the SCALE `enum` (tagged union) of:

- `Index8 = 0: u8`
- `Index16 = 1: Compact<u16>`
- `Index32 = 2: Compact<u32>`
- `Index64 = 3: Compact<u64>`
- `Index128 = 4: Compact<u128>`
- `Array4 = 16: [u32; 4]`
- `Array8 = 17: [u32; 8]`
- `Array16 = 18: [u32; 16]`
- `Array32 = 19: [u32; 32]`
- `Blob = 255: Vec<u8>`

## `MultiDest`: Universal Destination Identifiers

Destination identifiers are self-describing identifiers that can specify an owner into whose control some cryptographic asset be placed. It aims to be sufficiently abstract in meaning and general in nature that it works across a variety of chain types, including UTXO, account-based and privacy-preserving.

Basic format:

- `version: Compact<u32> = 0x00`: Version code, currently there's only the first version, zero.
- `type: Compact<u32>`: The type of destination.
- `payload`: The data payload, format determined by the `type`.

### Type 0: 32-byte Standard Substrate Frame Account ID

This is a generic 32-byte multi-crypto Account ID, as used across Polkadot, Kusama, Westend and various other Substrate-based chains. It corresponds directly to the public key of either the S/R 25519 or the Ed25519 signature schemes, or the Blake2-256 hash of the 33 byte compact ECDSA public key. Depending on the exact nature of the chain, several other account mechanisms (see e.g. the `utility`, `multisig` or `proxy` pallets) may be in place for account authentication.

- `id: [u8; 32]` The 32 byte Account ID.

### Type 1: 64-bit Account Index

May be evaluated in the context of any system with a ubiquitous and canonical account indexing system.

An example would be a Substrate Frame chain including the `indices` pallet allowing accounts to be refered to through a short-form index. This refers to the account whose index is provided.

- `index: Compact<u64>` The 64-bit account index.

### Type 2: Parachain Primary Account

The Polkadot Relay pallet collection includes the idea of a Relay-chain with one or more Parachains/threads, each identified by a `ParaId`,  scalar `u32` value. The Relay-chain has associated sovereign accounts controlled by its parachains. This indicates the primary such account.

- `index: Compact<u32>` The 32-bit `ParaId`.

### Type 3: Sub-destination

Many destination types, e.g. smart contracts and parachains, can have secondary destinations nested beneath them and interpreted within their context. This allows for that.

- `primary: MultiDest` The primary destination.
- `subordinate: MultiDest` The subordinate destination, interpreted in the context of the primary destination.

### Type 4: Super-destination

The super-consensus system relative to the context in which the value is being evaluated. Examples would include the Relay-chain if the context was a parachain, or a parachain if the context were a smart-contract hosted by that parachain.

A Type 3 with a `subordinate` equal to this Type 4 is, by definition, exactly equivalent to its `primary`.

### Type 5: Side-destination

Exactly equivalent to a Type 3 *Sub-destination* whose `primary` is a Type 4 *Super-destination*.

- `sibling: MultiDest` The destination, interpreted in the context of the current context's super destination.

### Type 6: 20-byte Ethereum Account Key

- `key: [u8; 20]` The account key, as derived from the 20 bytes at the end of the Keccak-256 hash of the SECP256k1/ECDSA public key, or owed by a smart contract at the address.

## Examples

`0xff0000000200040c4254430004d43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d`

This indicates to the receiving chain that the account controlled by Substrate's testing Alice key (`subkey inspect //Alice`) should be credited with 1 Satoshi worth of its variant/interpretation of the Bitcoin (`BTC`) token.

## Appendix: Fungible Asset Types

- `b"DOT"`
- `b"KSM"`
- `b"BTC"`
- `b"ETH"`

## Appendix: Non-Fungible Asset Classes

(None yet.)
