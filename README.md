# Polkadot Cross-Consensus Message (XCM) Format

This page details the message format for Polkadot-based message passing between chains. It describes the formal data format, any environmental data which may be additionally required and the corresponding meaning of the datagrams.

## Background

In Polkadot, three message passing systems use this format: XCMP, HRMP and VMP.

- **XCMP** *Cross-Chain Message Passing* highly scalable message passing between parachains with minimal work on the side of the Relay-chain.
- **VMP** *Vertical Message Passing* message passing between the Relay-chain itself and a parachain.
- **HRMP** *Horizontal Relay-based Message Passing* composite message passing, similar in effect to XCMP (messages between parachains) but implemented using VMP so messages are passed through the Relay-chain using the `RMP` and `PRM` meta-messages.

## Definitions

- *Sovereign Account* An account controlled solely and irrevocably by a particular chain.
- *Origin* The chain, contract or other global, encapsulated, state machine singleton from which a given message has been (directly and immediately) delivered. This is always queryable by the receiving code using the message-passing protocol. Generally specified as a *UDI*
- *Recipient* The chain, contract or other global, encapsulated, state machine singleton to which a given message has been delivered.
- *Teleport* Destroying an asset (or amount of funds/token/currency) in one place and minting a corresponding amount in a second place. Imagine the teleporter from *Star Trek*. The two places need not be equivalent in nature (e.g. could be a UTXO chain that destroys assets and an account-based chain that mints them). Neither place acts as a reserve or derivative for the other. Though the nature of the tokens may be different, neither place is more canonical than the other. This is only possible if there is a bilateral trust relationship between them.
- *Transfer* The movement of funds from one controlling authority to another. This is within the same chain or overall asset-ownership environment and at the same abstraction level.
- *Universal Destination Indicator (UDI)* A chain, contract or global, encapsulated, state machine singleton, or an addressable datastructure that exists therein. Examples include an account on the Relay-chain, an account on a parachain, a parachain, an account managed by a smart-contract in a parachain. It can be any programmatic state-transition system that exists within consensus which can send/receive datagrams.

## Basic Top-level Format

All data is SCALE encoded. We name the top-level XCM datatype `Xcm`.

- `magic: [u8; 2] = 0xff00`: Prefix identifier.
- `version: u16 = 0u16`: Version of XCM; only zero supported currently.
- `type: u16`: Message type.
- `payload`: Message parameters.

For version 0, message `type` must be one of:

- `0u16`: `FFAX`
- `1u16`: `FFAC`
- `2u16`: `FAT`
- `3u16`: `NAT`

## Message Types

### FFAX: Foreign Fungibles Account Transfer

An instructive message commanding the transfer of some amount of funds (measured in the native currency of the *Recipient*) from the (presumed unique or otherwise primary) account owned by the *Origin* to some other destination on the *Recipient*. Typically only used for upward messages. This cannot directly be used for sending funds from one parachain to another since there is no way of giving attribution for the derivate credit.

An *Amount of Funds*, measured in the *Native Currency* of the *Recipient* should be transfered from the *Sovereign Account* of the *Origin* to the account identified by a universal `destination` identifier within the context of the *Recipient*.

Parameter(s):

- `amount: Compact<u256>` The *Amount of Funds* that should be transfered from the *Sovereign Account* of the *Origin* on the *Recipient*.
- `asset: Vec<u8>` The fungible asset type (aka currency code) of the asset to be transfered. Known `asset` types are listed in the Appendix Asset Types. Empty `asset` indicates of the native currency of the recipient chain.
- `destination: DestId` A universal destination identifier which identifies the account/owner/controller on the *Recipient* to be credited. A type 3 (multi-level) ID indicates that an RDC message may be needed.
- `source: DestId` UDI identifing the true source of the transfer, in terms of the `Origin`. Null indicates the `Origin` itself.

### FFAC: Foreign Fungibles Account Credit

A notification message that the *Origin*, acting as a *Reserve*, has received funds into a client account owned by the *Recipient*. It is instructive only in so much as to dictate to the receiving chain the associated destination to which the recipient chain may attribute the credit. The funds are specified in the native currency of the *Origin*. Typically only used for downward messages.

In other words: An *Amount of Funds*, measured in the *Native Currency* of the *Origin* have been credited to the *Sovereign Account* of the *Recipient* and a sub-account identifier is provided for the *Recipient* to credit within its own context.

Parameter(s):

- `amount: Compact<u256>` The *Amount of Funds* that have been credited to the *Sovereign Account* of the *Recipient* on the *Origin*.
- `asset: Vec<u8>` The fungible asset type (aka currency code) of the asset to be transfered. Known `asset` types are listed in the Appendix Asset Types. Empty `asset` indicates of the native currency of the recipient chain.
- `destination: DestId` A universal destination ID which identifies the sub-account to be credited within the context of the *Recipient*.
- `source: DestId` UDI identifing the true source of the transfer, in terms of the `Origin`. Null indicates the `Origin` itself.

### RMP: Relay Message Parachain

An instructive message to indicate that a given message should be relayed to a further *Destination Chain*. The actual message presented to the *Destination Chain* will be of type `Parachain Relayed Message` and properly present the original sender in it.

Parameter(s):

- `destination: ParaId` The chain index to which this `message` should be relayed within a `PRM`.
- `message: Xcm` The message to be interpreted by the *Recipient*.

### PRM: Parachain Relay Message

A counterpart message to RMP, this is what is sent by the Relayer to the `destination` parachain mentioned in the RMP message. It is instructive only of the fact that an RMP with `message` and a `destination` equal to the *Receiving Parachain* was sent by the `source` parachain of the *Origin Relay-chain*.

Parameter(s):

- `source: ParaId` The chain index from which this `message` has been relayed from an `RMP`.
- `message: Xcm` The message to be interpreted by the *Recipient*.

### FAT: Fungible Asset Teleport

An `amount` of some fungible asset identified by an opaque datagram `asset_id` have been removed from existence on the *Origin* and should be credited on the *Recipient* into the account identified by a universal `destination` identifier.

Parameter(s):

- `amount: Compact<u256>` The *Amount of Funds* that should be transfered from the *Sovereign Account* of the *Origin* on the *Recipient*.
- `asset: Vec<u8>` The fungible asset type (aka currency code) of the asset to be transfered. Known `asset` types are listed in the Appendix Asset Types. Empty `asset` indicates of the native currency of the recipient chain.
- `destination: DestId` A universal destination identifier which identifies the account/owner/controller on the *Recipient* to be credited.
- `source: DestId` UDI identifing the true source of the transfer, in terms of the `Origin`. Null indicates the `Origin` itself.

### NAT: Non-fungible Asset Teleport

An unique instance of a non-fungible asset identified within its asset `class` by some individual `id` has been removed from existence on the *Origin* and should be created on the *Recipient* and owned the account identified by a universal `destination` identifier.

Parameter(s):

- `class: Vec<u8>` The class of asset; known classes are listed in the Appendix Classes.
- `id: Vec<u8>` The specific instance of the asset, within the asset `class`.
- `destination: DestId` A universal destination identifier which identifies the account/owner/controller on the *Recipient* to be credited.

## `DestId`: Universal Destination Identifiers

Destination identifiers are self-describing identifiers that can specify an owner into whose control some cryptographic asset be placed. It aims to be sufficiently abstract in meaning and general in nature that it works across a variety of chain types, including UTXO, account-based and privacy-preserving.

### Basic format

- `version: u8 = 0x00`: Version code, currently there's only the first version, zero.
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

- `primary: DestId` The primary destination.
- `subordinate: DestId` The subordinate destination, interpreted in the context of the primary destination.

### Type 4: Super-destination

The super-consensus system relative to the context in which the value is being evaluated. Examples would include the Relay-chain if the context was a parachain, or a parachain if the context were a smart-contract hosted by that parachain.

A Type 3 with a `subordinate` equal to this Type 4 is, by definition, exactly equivalent to its `primary`.

### Type 5: Side-destination

Exactly equivalent to a Type 3 *Sub-destination* whose `primary` is a Type 4 *Super-destination*.

- `sibling: DestId` The destination, interpreted in the context of the the current context's super destination.

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
