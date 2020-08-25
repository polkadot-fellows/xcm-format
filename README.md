# Polkadot Cross-Consensus Message (XCM) Format

This page details the message format for Polkadot-based message passing between chains. It describes the formal data format, any environmental data which may be additionally required and the corresponding meaning of the datagrams.

## Background

There are several kinds of *Consensus Systems* for which it would be advantageous to facilitate communication. This includes messages between smart-contracts and their environment, messages between sovereign blockchains over bridges, and between shards governed by the same consensus. Unfortunately, each tends to have its own message-passing means and standards, or have no standards at all.

XCM aims to abstract the typical message intentions across these systems and provide a basic framework for forward-compatible, extensible and practical communication datagrams facilitating typical interactions between disparate datasystems within the world of global consensus.

Concepts from the IPFS project, particularly the idea of self-describing formats, are used throughout and two new self-describing formats are introduced for specifying assets (`MultiAsset`) and consensus-message processing locations (`MultiLocation`).

Polkadot has three main message passing systems all of which will use this format: XCMP and the two kinds VMP (UMP and DMP).

- **XCMP** *Cross-Chain Message Passing* highly scalable secure message passing between parachains. Message data goes direct between parachains and is O(1) on the side of the Relay-chain.
- **VMP** *Vertical Message Passing* message passing between the Relay-chain itself and a parachain. Message data in both cases exists on the Relay-chain. This includes:
  - **UMP** *Upward Message Passing* message passing from a parachain to the Relay-chain.
  - **DMP** *Downward Message Passing* message passing from the Relay-chain to a parachain.

In addition, a third "composite" message passing system is named as **HRelayMessageParachain** *Horizontal Relay-routed Message Passing*. It is implemented through utilising the two routing meta-messages of XMP (`RelayMessageParachain` and `ParachainRelayMessage`) so that parachains may send messages between each other before XCMP is finalised. This relies on the Relay-chain storing and relaying the messages and as such is not scalable. Parathreads may not receive such messages since queues could grow indefinitely.

### XCM Communication Model

XCM is designed around four 'A's:

- *Asynchronous*: XCM messages in no way assume that the sender will be blocking on its completion.
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

All data is SCALE encoded. We name the top-level XCM datatype `XcmPacket`. Generally, this is defined thus:

- `magic: [u8; 2] = 0xff00`: Prefix identifier.
- `version: Compact<u32>`: Version of XCM; only zero supported currently.
- `message: Xcm`: The message; opaque unless version is known and well-defined.

### Xcm Version 0

The first version of Xcm, 0, is defined properly thus:

- `magic: [u8; 2] = 0xff00`: Prefix identifier.
- `version: u8 = 0u8`: Version of XCM; only zero supported currently. Values of 128 and over are explicitly reserved for potential extension of this datatype in the (hopefully distant) future.
- `message: Xcm`

The message, which amounts to the "important part" is simply named `Xcm`. It is defined thus:

- `type: u8`: Message type.
- `payload`: Message parameters.

Where message `type` must be one of:

- `0`: `WithdrawAsset`
- `1`: `ReserveAssetTransfer`
- `2`: `ReserveAssetCredit`
- `3`: `TeleportAsset`

Within XCM version 0, there is a secondary datatype `Ai`, "Asset Instruction", defined as:

- `type: u8`: Instruction type.
- `payload`: Instruction parameters.

It should be enumerated thus:

- `0`: `DepositAsset`
- `1`: `ExchangeAsset`
- `2`: `InitiateReserveTransfer`
- `3`: `InitiateTeleport`

## `Xcm` Message Types

![image](https://user-images.githubusercontent.com/138296/85861827-38ef4f00-b7c1-11ea-8a2d-32cacbcef782.png)

_Basic interaction diagram. Boxes are blockchains, cylinders are accounts, red arrow over an account is a debit, green arrow is a credit. Circled accounts are Holding Accounts. DA is `DepositAsset`, EA is `ExchangeAsset`, RAX is `ReserveAssetTransfer`, RAC is `ReserveAssetCredit`, TA is `TeleportAsset` and WA is `WithdrawAsset`_

### `ReserveAssetTransfer`

An instructive message commanding the transfer of some asset(s) from the (presumed unique or otherwise primary) account owned by the *Origin* to some other destination on the *Recipient*.

`asset`s (with defaults governed by the *Recipient*) should be transfered from the *Sovereign Account* of the *Origin* to the *Sovereign Account* of `destination`. An `ReserveAssetCredit` message will be sent to the `destination` and it must be reachable via messaging.

Parameter(s):

- `asset: MultiAsset` The asset(s) to be transfered.
- `destination: MultiDest` The destination for the assets. This must be reachable for a `ReserveAssetCredit` message.
- `effect: Ai` What should be done with the assets.

### `ReserveAssetCredit`

A notification message that the *Origin*, acting as a *Reserve*, has received funds into a *Sovereign* account owned by the *Recipient*. The `asset` should be minted into the *Holding Account* and some `effect` evaluated.

Parameter(s):

- `asset: MultiAsset` The asset(s) that were transfered.
- `effect: Ai` What should be done with the assets.

### `TeleportAsset`

Some fungible `asset`(s) have been removed from existence (and ownership by `source`) on the *Origin* and should be minted into the holding account on the *Recipient* and some `effect` evaluated.

Parameter(s):

- `asset: MultiAsset` The asset(s) which were debited.
- `effect: Ai` What should be done with the assets on the receiving chain.

### `WithdrawAsset`

An instructive message commanding the removal of some asset(s) from the *Sovereign Account* owned by the *Origin* and their placement into the *Holding Account* and the evaluation of some `effect`.

Parameter(s):

- `asset: MultiAsset` The asset(s) to be debited.
- `effect: Ai` What should be done with the assets.

### `Balances`

Informational message detailing some balances, interpreted based on the context of the destination and the `query_id`.

- `query_id` The identifier of the query which caused this message to be sent.
- `assets` The value for use by the destination.

## `AssetInstruction` Instruction Types

### `Each`: Multiple Instructions

Execute all given `instructions` in order.

- `instructions: Vec<Ai>` Instructions to be executed.

### `DepositAsset`

Deposit all assets in the *Holding Account* into a given `destination` account. It cannot require any futher message passing.

- `asset: MultiAsset` Identifies the asset to be transfered.
- `destination: MultiLocation` Identifies the account/owner/controller to be credited. This should be an account within the *Consensus System* of the context. If this is a *Consensus System*, then it will credit its *Sovereign Account*.

### `ExchangeAsset`

Take `give` assets in the *Holding Account* and attempt to exchange them `for` some other assets.

- `give: MultiAsset` Identifies the maximum assets to be debited from this transaction.
- `for: MultiAsset` Identifies the minimal assets that should be credited from this transaction.

### `InitiateReserveTransfer`

Burn `asset` from the holding account and send an `ReserveAssetTransfer` message to the Reserve Location for the `asset`.

Parameter(s):

- `asset: MultiAsset` The asset to be debited from the *Holding Account* and transfered in the *Reserve Location*.
- `destination: MultiLocation` The destination for the asset from the context of the *Reserve Location*. This is specified from the context of the *Reserve Location* and its value must satisfy the resulting `ReserveAssetTransfer` message.
- `effect: Ai` What should be done with the assets in the final `destination`.

### `InitiateTeleport`

Burn `asset` from the *Holding Account* and send a `TeleportAsset` message to the `destination` to credit them into a *Holding Account* on another *Consensus System* along with some instructions to be `effect`ed.

Parameter(s):

- `asset: MultiAsset` The asset to be teleported.
- `destination: MultiLocation` Identifies the location for sending the `TeleportAsset` message to.
- `effect: Ai` What should be done with the assets in the final `destination`.

### `QueryHolding`

Initiates a `Balances` message to some `destination` with the given `query_id` for each of the `assets` given.

- `query_id: Compact<u64>` Some identifying index.
- `destination: MultiLocation` Recipient of the `Balances` message.
- `assets: Vec<MultiAsset>` Asset(s) to be queried. Use of wildcards (empty/zero/`Undefined` identifiers) should generally be supported.

## `MultiAsset`: Universal Asset Identifiers

Basic format:

- `version: u8`: Version/format code; currently two codes are supported `0x00`, and `0x01`.

### Abstract Fungible assets

- `version: u8 = 0x00` The version; zero for now.
- `id: Vec<u8>` The fungible asset identifier, usually derived from the ticker-tape code (e.g. `*b"BTC"`, `*b"ETH"`, `*b"DOT"`). See *Appendix: Fungible Abstract Asset Types* for a list of known values. The empty value may be used to indicate all assets. In this case, `amount` is ignored but should be set to zero by convention. Contexts may or may not support this.
- `amount: Compact<u128>` The amount of the asset identified. Zero may be used as a wildcard, to indicate all of the available asset or an unknown amount, determined by context.

### Abstract Non-fungible assets

- `version: Compact<u32> = 0x01`
- `class: Vec<u8>` The general non-fungible asset class code. See Appendix: Non-fungible Abstract Asset Types for a list of known values. The empty value may be used to indicate all asset classes. In this case, `instance` is ignored but should be set to `Undefined` by convention. Contexts may or may not support this.
- `instance: AssetInstance` The general non-fungible asset instance within the NFA class. May be identified with a numeric index or a datagram. Most `class`es will support only a single specific kind of `AssetInstance`, however for ease of formatting and to facilitate future compatibility, it is self-describing. `Undefined` may be used to indicate all available assets of this `class`. Contexts may or may not support this.

#### `AssetInstance`

Given by the SCALE `enum` (tagged union) of:

- `Undefined = 0: ()`
- `Index8 = 1: u8`
- `Index16 = 2: Compact<u16>`
- `Index32 = 3: Compact<u32>`
- `Index64 = 4: Compact<u64>`
- `Index128 = 5: Compact<u128>`
- `Array4 = 6: [u32; 4]`
- `Array8 = 7: [u32; 8]`
- `Array16 = 8: [u32; 16]`
- `Array32 = 9: [u32; 32]`
- `Blob = 10: Vec<u8>`

### Concrete Fungible assets

TODO

### Concrete Non-fungible assets

TODO

## `MultiLocation`: Universal Destination Identifiers

Destination identifiers are self-describing identifiers that can specify an owner into whose control some cryptographic asset be placed. It aims to be sufficiently abstract in meaning and general in nature that it works across a variety of chain types, including UTXO, account-based and privacy-preserving.

Basic format:

- `version: u8 = 0x00`: Version code, currently there's only the first version, zero.
- `type: u8`: The type of destination.
- `payload`: The data payload, format determined by the `type`.

### Type 0: `Null`

Indicates that the context under which the value is being evaluated is itself the target.

Written using a shorthand of `.`.

### Type 1: `Parent`

The super-*Consensus System* relative to the context in which the value is being evaluated. Examples would include the Relay-chain if the context was a parachain, or a parachain if the context were a smart-contract hosted by that parachain.

Written using a shorthand of `..`.

### Type 2: `ChildOf`

Many destination types, e.g. smart contracts and parachains, can have secondary destinations nested beneath them and interpreted within their context. This allows for that.

- `primary: MultiLocation` The primary destination.
- `subordinate: MultiLocation` The subordinate destination, interpreted in the context of the primary destination.

A ChildOf, one of whose two `primary`/`subordinate` values is Null is exactly equivalent to the other of the two values.

A ChildOf with a `subordinate` equal to `Parend` is, by definition, exactly equivalent to its `primary`.

Written using a shorthand of `<primary>/<subordinate>`.

### Type 3: `SiblingOf`

Exactly equivalent to a ChildOf whose `primary` is a Parent.

- `sibling: MultiLocation` The destination, interpreted in the context of the current context's super destination.

No specific shorthand, but written by convention as `../<sibling>`.

### Type 7: `OpaqueRemark`

Not a true destination, but a pseudo-destination. Some destinations/contexts support attaching an opaque datagram ("remark") to an operation. In such supported contexts, this attaches a `remark` to the operation. This usually has no meaning to the operation, but rather is a human-readable message to help explain the operation.

Typically used as the `subordinate` field of a `ChildOf`.

- `remark: Vec<u8>` The remark to be recorded with the operation. The actual location remains the current context.

### Type 8: `AccountId32`

This is a generic 32-byte multi-crypto Account ID. If desired, the `network` may be restricted/specified.

- `network: MultiNetwork` The network identifier for the primary network, chain, contract, destination or context on which this account ID functions (if any).
- `id: [u8; 32]` The 32 byte Account ID.

### Type 9: `AccountIndex64`

May be evaluated in the context of any system with a ubiquitous and canonical account indexing system.

An example would be a Substrate Frame chain including the `indices` pallet allowing accounts to be refered to through a short-form index. This refers to the account whose index is provided.

- `network: MultiNetwork` The network identifier for the primary network, chain, contract, destination or context on which this account functions (if any).
- `index: Compact<u64>` The 64-bit account index.

### Type 10: `ParachainPrimaryAccount`

The Polkadot Relay pallet collection includes the idea of a Relay-chain with one or more Parachains/threads, each identified by a `ParaId`, a scalar `u32` value. The Relay-chain has associated sovereign accounts controlled by its parachains. This indicates the primary such account.

- `network: MultiNetwork` The network identifier for the Relay-chain on which this ID functions (if any).
- `index: Compact<u32>` The 32-bit `ParaId`.

### Type 11: `AccountKey20`

- `network: MultiNetwork` The network identifier for the primary network, chain, contract, destination or context on which this account key functions (if any).
- `key: [u8; 20]` The account key, as derived from the 20 bytes at the end of the Keccak-256 hash of the SECP256k1/ECDSA public key, or owed by a smart contract at the address.

## MultiNetwork

Basic format:

- `version: u8`: Version/format code.

### Version 0: Wildcard

Indicates that the identifier is potentially relevant in all syntactically legal positions throughout all *Consensus Systems*.

- `version: u8 = 0u8`

### Version 1: Networks

Indicates that the identifier is relevant only to a specific network, as described.

- `version: u8 = 1u8`
- `network_id: Vec<u8>`

Valid values for `network_id` include:

- `b"dot"`: Polkadot mainnet; `AccountId32` corresponds directly to the public key of either the S/R 25519 or the Ed25519 signature schemes, or the Blake2-256 hash of the 33 byte compact ECDSA public key. Several other account mechanisms (see e.g. the `utility`, `multisig` or `proxy` pallets) are also in place for account authentication. `AccountIndex64` corresponds to an entry in the Index pallet.
- `b"ksm"`: Kusama mainnet; `AccountId32` and `AccountIndex64` correspond as per Polkadot.
- `b"eth"`: Ethereum mainnet; `AccountKey20` corresponds to the trimmed Keccak hash of the ECDSA public key, or a contract address.
- `b"btc"`: Bitcoin mainnet; `AccountKey20` corresponds to a standard Bitcoin address.

## Appendix: Fungible Abstract Asset Types

These generally correspond to the popularly used ticker-tape symbols for each currency.

- `b""` (Reserved as a wildcard.)
- `b"DOT"` Polkadot (mainnet) tokens.
- `b"KSM"` Kusama tokens.
- `b"BTC"` Bitcoin (mainnet) tokens.
- `b"ETH"` Ethereum (foundation, mainnet) tokens.
- `b"ETC"` Ethereum (classic, mainnet) tokens.

## Appendix: Non-Fungible Abstract Asset Classes

- `b""` (Reserved as a wildcard.)

## Examples

### Foreign account exchange

Attempts to exchange 42 DOT for 21 BTC. Balance is queried to allow `H` to check result.

- `H` Home chain.
- `X` Exchange chain on which `H` has holdings.

```
X.WithdrawAsset(
    42 DOT,
    Each(
        ExchangeAsset(*, 21 BTC)
        QueryHolding(../H)
        DepositAsset(../H, *)
    )
)
```

### Transfer via teleport

Two peer chains that trust each other's STF use the teleport functionality to transfer 21 aUSD from Alice on the home chain to Bob on the other.

- `H` Home chain; Alice's account here has been debited by 21 aUSD.
- `D` Destination chain.

```
D.TeleportAsset(
    21 aUSD,
    DepositAsset(Bob, *)
)
```

No fees are paid (we assume they're managed elsewhere).

### Transfer via reserve

Two peer chains that trust a third (reserve) chain's STF use the transfer functionality to transfer 21 DOT from Alice on the home chain to Bob on the other.

- `H` Home chain; Alice's account here has been debited by 21 DOT.
- `D` Destination chain.
- `R` The Relay-chain, acting as a reserve.

```
R.ReserveAssetTransfer(
    21 DOT,
    D,
    DepositAsset(Bob, *)
)
```

This will result in `R.sovereign(H)` being reduced by 21 DOT (together with the fee) and `R.sovereign(D)` being credited with 21 DOT. A message will be sent by `R`:

```
D.ReserveAssetCredit(
    21 DOT,
    DepositAsset(Bob, *)
)
```

### Exchange attempt of two currencies via separate reserves

Alice is attempting to exchange 42 DOT for 21 BTC, but neither the Home chain nor the Exchange chain trust the other's STF. DOT is held in reserve by the Relay chain of which both are parachains. BTC is held in reserve on the BTC bridge chain, which is also a parachain.

- `H` Home chain; Alice's account here has been debited by 42 DOT.
- `X` The exchange parachain.
- `R` The Relay-chain, acting as a reserve for DOT.
- `B` The Bitcoin bridge parachain, acting as a reserve for BTC.

```
R.ReserveAssetTransfer(
    42 DOT,
    X,
    Each(
        ExchangeAsset(*, 21 BTC)
        InitiateReserveTransfer(
            * BTC,
            ../H,
            DepositAsset(Alice)
        )
        InitiateReserveTransfer(
            * DOT,
            H,
            DepositAsset(Alice)
        )
    )
)
```
