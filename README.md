# Polkadot Cross-Consensus Message (XCM) Format

**Version 3, in-progress.**
**Authors: Gavin Wood.**

This document details the message format for Polkadot-based message passing between chains. It describes the formal data format, any environmental data which may be additionally required and the corresponding meaning of the datagrams.

## **1** Background

There are several kinds of *Consensus Systems* for which it would be advantageous to facilitate communication. This includes messages between smart-contracts and their environment, messages between sovereign blockchains over bridges, and between shards governed by the same consensus. Unfortunately, each tends to have its own message-passing means and standards, or have no standards at all.

XCM aims to abstract the typical message intentions across these systems and provide a basic framework for forward-compatible, extensible and practical communication datagrams facilitating typical interactions between disparate datasystems within the world of global consensus.

Concepts from the IPFS project, particularly the idea of self-describing formats, are used throughout and two new self-describing formats are introduced for specifying assets (based around `MultiAsset`) and consensus-system locations (based around `MultiLocation`).

Polkadot has three main transport systems for passing messages between chains all of which will use this format: XCMP (sometimes known as HRMP) together with the two kinds of VMP: UMP and DMP.

- **XCMP** *Cross-Chain Message Passing* secure message passing between parachains. There are two variants: *Direct* and *Relayed*.
  - With *Direct*, message data goes direct between parachains and is O(1) on the side of the Relay-chain and is very scalable.
  - With *Relayed*, message data is passed via the Relay-chain, and piggy-backs over VMP. It is much less scalable, and parathreads in particular may not receive messages due to excessive queue growth.
- **VMP** *Vertical Message Passing* message passing between the Relay-chain itself and a parachain. Message data in both cases exists on the Relay-chain. This includes:
  - **UMP** *Upward Message Passing* message passing from a parachain to the Relay-chain.
  - **DMP** *Downward Message Passing* message passing from the Relay-chain to a parachain.

### **1.1** XCM Communication Model

XCM is designed around four 'A's:

- *Asynchronous*: XCM messages in no way assume that the sender will be blocking on its completion.
- *Absolute*: XCM messages are guaranteed to be delivered and interpreted accurately, in order and in a timely fashion.
- *Asymmetric*: XCM messages do not have results. Any results must be separately communicated to the sender with an additional message.
- *Agnostic*: XCM makes no assumptions about the nature of the Consensus System between which messages are being passed.

The fact that XCM gives these *Absolute* guarantees allows it to be practically *Asymmetric* whereas other non-Absolute protocols would find this difficult.

Being *Agnostic* means that XCM is not simply for messages between parachain(s) and/or the Relay-chain, but rather that XCM is suitable for messages between disparate chains connected through one or more bridge(s) and even for messages between smart-contracts. Using XCM, all of the above may communicate with, or through, each other.

E.g. It is entirely conceivable that, using XCM, a smart contract, hosted on a Polkadot parachain, may transfer a non-fungible asset it owns through Polkadot to an Ethereum-mainnet bridge located on another parachain, into an account controlled on the Ethereum mainnet by registering the transfer of ownership on a third, specialized Substrate NFA chain hosted on a Kusama parachain via a Polkadot-Kusama bridge.

### **1.2** The XCVM

The XCM format in large part draws upon a highly domain-specific virtual machine, called the _Cross-Consensus Virtual Machine_ or _XCVM_. XCM messages correspond directly to version-aware XCVM programmes, and the XCVM instruction set represents the repertoire of actions from which an XCM message may be composed.

The XCVM is a register-based machine, none of whose registers are general purpose. The XCVM instruction format, set of machine registers and definitions of interaction therefore compromise the bulk of the XCM message format and most of this document's text is taken up with expressing those definitions.

### **1.3** Vocabulary

- *Consensus System* A chain, contract or other global, encapsulated, state machine singleton. It can be any programmatic state-transition system that exists within consensus which can send/receive datagrams. May be specified by a `MultiLocation` value (though not all such values identify a *Consensus System*). Examples include *The Polkadot Relay chain*, *The XDAI Ethereum PoA chain*, *The Ethereum Tether smart contract*.
- *Location* A *Consensus System*, or an addressable account or data structure that exists therein. Examples include the Treasury account on the Polkadot Relay-chain, the primary Web3 Foundation account on the Edgeware parachain, the Edgeware parachain itself, the Web3 Foundation's Ethereum multisig wallet account. Specified by a `MultiLocation`.
- *Sovereign Account* An account controlled by a particular *Consensus System*, within some other *Consensus System*. There may be many such accounts or just one. If many, then this assumes and identifies a unique *primary* account.
- *XCVM* The Cross-consensus Virtual Machine, for which the definition of XCM messages s large part relies upon.
- *Reserve Location* The *Consensus System* which acts as the reserve for a particular assets on a particular (derivative) *Consensus System*. The reserve *Consensus System* is always known by the derivative. It will have a *Sovereign Account* for the derivative which contains full collateral for the derivative assets.
- *Origin* The *Consensus System* from which a given message has been (directly and immediately) delivered. Specified as a `MultiLocation`.
- *Recipient* The *Consensus System* to which a given message has been delivered. Specified as a `MultiLocation`.
- *Teleport* Destroying an asset (or amount of funds/token/currency) in one place and minting a corresponding amount in a second place. Imagine the teleporter from *Star Trek*. The two places need not be equivalent in nature (e.g. could be a UTXO chain that destroys assets and an account-based chain that mints them). Neither place acts as a reserve or derivative for the other. Though the nature of the tokens may be different, neither place is more canonical than the other. This is only possible if there is a bilateral trust relationship both of the STF and the validity/finality/availability between them.
- *Transfer* The movement of funds from one controlling authority to another. This is within the same chain or overall asset-ownership environment and at the same abstraction level.

### **1.4** Document Structure

The format is defined in five main parts. The top-level datagram formats are specified in section 2. The XCVM is defined in sections 3, 4 and 5. The `MultiLocation` and `MultiAsset` formats are defined in sections 6 and 7. Example messages are specified in section 8.

### **1.5** Encoding

All data is SCALE encoded. Elementary types are expressed in Rust-language format, for example:

- A 32-bit signed integer is written as `s32`.
- A 64-bit unsigned integer is written as `u64`.

When a bulleted list of types---possibly named---is given, it implies a simple concatenation of individual typed values. For example, a 64-bit unsigned "index" followed by a list of bytes of "data" could be written as:

- `index: u64`
- `data: Vec<u8>`

## **2** Basic Top-level Format

We name the top-level XCM datatype `VersionedXcm`. This is defined thus:

- `version: u8`: Version of XCM.
- `message`: The message; opaque unless version is known and well-defined.

This document defines only XCM messages whose version identifier is `2`. Messages whose version is lower than this may be defined in earlier versions of this document.

Given that the version of XCM defined at present is `2`, thus we can concretize the message format as:

- `version: u8 = 2`: Version of XCM.
- `message: Vec<Instruction>`

Thus a message is simply the byte `2` to identify the present version, together with a SCALE-encoded series of XCVM instructions.

The effect of any given XCM message is defined as the actions taken by the XCVM when it is initialized properly given the `message` (which is known in XCVM as the *Original Programme*) and the location from which the message originated (which is known in XCVM as the *Original Origin*). The specifics of initialization are defined in the next section.

## **3** The XCVM Registers

The XCVM contains several machine registers, which cannot generally be set at will, but rather begin with specific values and may only be mutated under certain circumstances and/or obeying certain rules.

The registers are named:

- *Programme*
- *Programme Counter*
- *Error*
- *Error Handler*
- *Appendix*
- *Origin*
- *Holding*
- *Surplus Weight*
- *Refunded Weight*

### **3.1** Programme

Of type `Vec<Instruction>`, initialized to the value of the Original Programme.

Expresses the currently executing programme of instructions for the XCVM. This gets potentially changed after either the final instruction is executed or an error occurs.

### **3.2** Programme Counter

Of type `u32`, initialized to `0`.

Expresses the index of the currently executing instruction within the Programme Register. Gets incremented by `1` after each instruction is successfully executed and reset to `0` when the Programme Register is replaced.

### **3.3** Error

Of type: `Option<(u32, Error)>`, initialized to `None`.

Expresses information on the last known error which happened during programme execution. Set only when a programme encounters an error. May be cleared at will. The two internal fields express the value of the Programme Counter when the error occurred and the type of error which happened.

### **3.4** Error Handler

Of type `Vec<Instruction>`, initialized to the empty list.

Expresses any code which should run in case of error. When a programme encounters an error, this register is cleared and its contents used to replace the Programme Register.

### **3.5** Appendix

Of type `Vec<Instruction>`, initialized to the empty list.

Expresses any code which should run following the current programme. When a programme reaches the end successfully, or after an error where the Error Handler is empty, this register is cleared and its contents used to replace the Programme Register.

### **3.6** Origin

Of type `Option<MultiLocation>`, initialized so its inner value is the Original Origin.

Expresses the location with whose authority the current programme is running. May be reset to `None` at will (implying no authority), and may also be set to a strictly interior location at will (implying a strict subset of authority).

### **3.7** Holding Register

Of type `MultiAssets`, initialized to the empty set (i.e. no assets).

Expresses a number of assets that exist under the control of the programme but have no on-chain representation. Can be thought of as a non-persistent register for "unspent" assets.

### **3.8** Surplus Weight

Of type `u64`, initialized to `0`.

Expresses the amount of weight by which an estimation of the Original Programme must have been an overestimation. This includes any weight of instructions which were never dispatched due to an error occurring in an instruction prior, as well as error handlers which did not take effect owing to a successful conclusion and instructions whose weight becomes known after a necessarily conservative estimate.

### **3.9** Refunded Weight

Of type `u64`, initialized to `0`.

Expresses the portion of Surplus Weight which has been refunded. Not used on XCM platforms which do not require payment for execution.

## **4** Basic XCVM Operation

The XCVM operates as a fetch-dispatch loop common in state machines. The steps of the loop are:

- If the value of the Programme Register is empty, then halt.
- Attempt to fetch the instruction found by indexing the Programme Register by the Programme Counter Register:
- If an instruction exists, then execute that instruction (see next section) and check whether it resulted in an error:
  - If it did not result in an error, then:
    - Increment the Programme Counter by `1`.
  - If it did result in an error, then:
    - Set the Error Register accordingly.
    - Reset the Programme Counter to `0`.
    - Set the Programme Register to the value of the Error Handler Register.
    - Reset the Error Handler Register to become empty.
- If no instruction exists:
  - Reset the Programme Counter to `0`.
  - Set the Programme Register to the value of the Appendix Register.
  - Increment the Surplus Weight Register by the estimated weight of the contents of the Error Handler Register.
  - Reset both the Error Handler Register and the Appendix Register to become empty.

The difference from a basic fetch/dispatch loop is the addition of the Error Handler and Appendix Registers. Notably:

- The Error Register is *not* cleared when a programme completes successfully. This allows code in the Appendix register to utilise its value.
- The Error Handler Register *is* cleared when a programme completes regardless of success. This ensures that any error handling logic from a previous programme does not affect any later appended code.

## **5** The XCVM Instruction Set

The XCVM instruction type (`Instruction`) is represented as a tagged union (`enum` in Rust language) of each of the individual instructions including their operands, if any. Since this is SCALE encoded, an instruction is encoded as its 0-indexed place within the instruction list, concatenated with the SCALE encoding of its operands.

The instructions, in order, are:

- `WithdrawAsset`
- `ReserveAssetDeposited`
- `ReceiveTeleportedAsset`
- `QueryResponse`
- `TransferAsset`
- `TransferReserveAsset`
- `Transact`
- `HrmpNewChannelOpenRequest`
- `HrmpChannelAccepted`
- `HrmpChannelClosing`
- `ClearOrigin`
- `DescendOrigin`
- `ReportError`
- `DepositAsset`
- `DepositReserveAsset`
- `ExchangeAsset`
- `InitiateReserveWithdraw`
- `InitiateTeleport`
- `QueryHolding`
- `BuyExecution`
- `RefundSurplus`
- `SetErrorHandler`
- `SetAppendix`
- `ClearError`
- `ClaimAsset`
- `Trap`
- `SubscribeVersion`
- `UnsubscribeVersion`

### Notes on terminology

- When the term _Origin_ is used, it is meant to mean "location whose value is that of the Origin Register". Thus the phrase "controlled by the Origin" is equivalent to "controlled by the location whose value is that of the Origin Register".
- When the term _Holding_ is used, it is meant to mean "value of the Holding Register". Thus the phrase "reduce Holding by the asset" is equivalent to "reduce the value of the Holding Register by the asset".
- Similarly for _Appendix_ (the value of the Appendix Register) and _Error Handler_ (the value of the Error Handler Register).
- The term _on-chain_ should be taken to mean "within the persistent state of the local consensus system", and should not be considered to limit the current consensus system to one which is specifically a blockchain system.
- The type `Compact`, `Weight` and `QueryId` should be intended to mean a compact integer as per the SCALE definition.

### `WithdrawAsset`

Remove the on-chain asset(s) (`assets`) and accrue them into Holding.

Operands:

- `assets: MultiAssets`: The asset(s) to be removed; must be owned by Origin.

Kind: *Instruction*.

Errors: *Fallible*.

### `ReserveAssetDeposited`

Accrue into Holding derivative assets to represent the asset(s) (`assets`) on Origin.

Operands:

- `assets: MultiAssets`: The asset(s) which have been received into the Sovereign account of the local consensus system on Origin.

Kind: *Trusted Indication*.

Trust: Origin must be trusted to act as a reserve for `assets`.

Errors: *Fallible*.

### `ReceiveTeleportedAsset`

Accrue assets into Holding equivalent to the given assets (`assets`) on Origin.

Operands:

- `assets: MultiAssets`: The asset(s) which have been removed from Origin.

Kind: *Trusted Indication*.

Trust: Origin must be trusted to have removed the `assets` as a consequence of sending this message.

Errors: *Fallible*.

### `QueryResponse`

Provide expected information from Origin.

Operands:

- `query_id: QueryId`: The identifier of the query that resulted in this message being sent.
- `response: Response`: The information content.
- `max_weight: Weight`: The maximum weight that handling this response should take. If proper execution requires more weight then an error will be thrown. If it requires less weight, then Surplus Weight Register may increase.

Kind: *Information*.

Errors: *Fallible*.

Weight: Weight estimation may utilise `max_weight` which may lead to an increase in Surplus Weight Register at run-time.

#### `Response`

The `Response` type is used to express information content in the `QueryResponse` XCM instruction. It can represent one of several different data types and it therefore encoded as the SCALE-encoded tagged union:

- `Null = 0`: No information.
- `Assets { assets: MultiAssets } = 1`: Some assets.
- `ExecutionResult { result: Result<(), (u32, Error)> } = 2`: An error (or not), equivalent to the type of value contained in the Error Register.
- `Version { version: Compact } = 3`: An XCM version.

### `TransferAsset`

Withdraw asset(s) (`assets`) from the ownership of Origin and deposit equivalent assets under the ownership of `beneficiary`.

Operands:

- `assets: MultiAssetFilter`: The asset(s) to be withdrawn.
- `beneficiary`: The new owner for the assets.

Kind: *Instruction*.

Errors: *Fallible*.

### `TransferReserveAsset`

Withdraw asset(s) (`assets`) from the ownership of Origin and deposit equivalent assets under the ownership of `destination` (i.e. within its Sovereign account).

Send an onward XCM message to `destination` of `ReserveAssetDeposited` with the given `xcm`.

Operands:

- `assets: MultiAsset`: The asset(s) to be withdrawn.
- `destination: MultiLocation`: The location whose sovereign account will own the assets and thus the effective beneficiary for the assets and the notification target for the reserve asset deposit message.
- `xcm: Xcm`: The instructions that should follow the `ReserveAssetDeposited` instruction, which is sent onwards to `destination`.

Kind: *Instruction*.

Errors: *Fallible*.

### `Transact`

Dispatch the encoded functor `call`, whose dispatch-origin should be Origin as expressed
by the kind of origin `origin_type`.

Operands:

- `origin_type: OriginKind`: The means of expressing the message origin as a dispatch origin.
- `max_weight: Weight`: The maximum amount of weight to expend while dispatching `call`. If dispatch requires more weight then an error will be thrown. If dispatch requires less weight, then Surplus Weight Register may increase.
- `call: ([u8; 32], Vec<u8)`: A well known interface identifier and the encoded body of the transaction to be applied.
- `index: u8`: A disambiguation index in case the call can be dispatched by more than one subsystem.

Kind: *Instruction*.

Errors: *Fallible*.

Weight: Weight estimation may utilise `max_weight` which may lead to an increase in Surplus Weight Register at run-time.

### `HrmpNewChannelOpenRequest`

A message to notify about a new incoming HRMP channel. This message is meant to be sent by the
Relay-chain to a para.

Operands:

- `sender: Compact`: The sender in the to-be opened channel. Also, the initiator of the channel opening.
- `max_message_size: Compact`: The maximum size of a message proposed by the sender.
- `max_capacity: Compact`: The maximum number of messages that can be queued in the channel.

Safety: The message should originate directly from the Relay-chain.

Kind: *System Notification*

### `HrmpChannelAccepted`

A message to notify about that a previously sent open channel request has been accepted by the recipient. That means that the channel will be opened during the next Relay-chain session change. This message is meant to be sent by the Relay-chain to a para.

Operands:

- `recipient: Compact`: The recipient parachain which has accepted the previous open-channel request.

Safety: The message should originate directly from the Relay-chain.

Kind: *System Notification*

Errors: *Fallible*.

### `HrmpChannelClosing`

A message to notify that the other party in an open channel decided to close it. In particular, `initiator` is going to close the channel opened from `sender` to the `recipient`. The close will be enacted at the next Relay-chain session change. This message is meant to be sent by the Relay-chain to a para.

Operands:

- `initiator: Compact`: The parachain index which initiated this close operation.
- `sender: Compact`: The parachain index of the sender side of the channel being closed.
- `recipient: Compact`: The parachain index of the receiver side of the channel being closed.

Safety: The message should originate directly from the Relay-chain.

Kind: *System Notification*

Errors: *Fallible*.

### `ClearOrigin`

Clear the Origin Register.

This may be used by the XCM author to ensure that later instructions cannot command the authority of the Original Origin (e.g. if they are being relayed from an untrusted source, as often the case with `ReserveAssetDeposited`).

Kind: *Instruction*.

Errors: *Infallible*.

### `DescendOrigin`

Mutate the origin to some interior location.

Operands:

- `interior: InteriorMultiLocation`: The location, interpreted from the context of Origin, to place in the Origin Register.

Kind: *Instruction*

Errors: *Fallible*.

### `ReportError`

Immediately report the contents of the Error Register to the given destination via XCM.

A `QueryResponse` message of type `ExecutionOutcome` is sent to `destination` with the given
`query_id` and the outcome of the XCM.

Operands:

- `query_id: QueryId`: The value to be used for the `query_id` field of the `QueryResponse` message.
- `destination: MultiLocation`: The location to where the `QueryResponse` message should be sent.
- `max_response_weight: Weight`: The value to be used for the `max_weight` field of the `QueryResponse` message.

Kind: *Instruction*

Errors: *Fallible*.

### `DepositAsset`

Subtract the asset(s) (`assets`) from Holding and deposit on-chain equivalent assets under the ownership of `beneficiary`.

Operands:

- `assets: MultiAssetFilter`: The asset(s) to remove from the Holding Register.
- `max_assets: Compact`: The maximum number of unique assets/asset instances to remove from the Holding Register. Only the first `max_assets` assets/instances of those matched by `assets` will be removed, prioritized under standard asset ordering. Any others will remain in holding.
- `beneficiary: MultiLocation`: The new owner for the assets.

Kind: *Instruction*

Errors: *Fallible*.

### `DepositReserveAsset`

Remove the asset(s) (`assets`) from the Holding Register and deposit on-chain equivalent assets under the ownership of `destination` (i.e. deposit them into its Sovereign Account).

Send an onward XCM message to `destination` of `ReserveAssetDeposited` with the given `effects`.

Operands:

- `assets: MultiAssetFilter`: The asset(s) to remove from the Holding Register.
- `max_assets: Compact`: The maximum number of unique assets/asset instances to remove from the Holding Register. Only the first `max_assets` assets/instances of those matched by `assets` will be removed, prioritized under standard asset ordering. Any others will remain in holding.
- `destination: MultiLocation`: The location whose sovereign account will own the assets and thus the effective beneficiary for the assets and the notification target for the reserve asset deposit message.
- `xcm: Xcm`: The orders that should follow the `ReserveAssetDeposited` instruction which is sent onwards to `destination`.

Kind: *Instruction*

Errors: *Fallible*.

### `ExchangeAsset`

Reduce Holding by up to some amount of asset(s) (`give`) and accrue Holding with a minimum amount of some alternative assets (`receive`).

Operands:

- `give: MultiAssetFilter`: The asset(s) by which Holding should be reduced.
- `receive: MultiAssets`: The asset(s) by which Holding must be increased. Any fungible assets appearing in `receive` may be increased by an amount greater than expressed, but Holding may not accrue assets not stated in `receive`.

Kind: *Instruction*

Errors: *Fallible*.

### `InitiateReserveWithdraw`

Reduce the value of the Holding Register by the asset(s) (`assets`) and send an XCM message beginning with `WithdrawAsset` to a reserve location.

Operands:

- `assets: MultiAssetFilter`: The asset(s) to remove from the Holding Register.
- `reserve`: A valid location that acts as a reserve for all asset(s) in `assets`. The sovereign account of this consensus system *on the reserve location* will have appropriate assets withdrawn and `effects` will be executed on them. There will typically be only one valid location on any given asset/chain combination.
- `xcm`: The instructions to execute on the assets once withdrawn *on the reserve location*.

Kind: *Instruction*

Errors: *Fallible*.

### `InitiateTeleport`

Remove the asset(s) (`assets`) from the Holding Register and send an XCM message beginning with `ReceiveTeleportedAsset` to a `destination` location.

NOTE: The `destination` location *MUST* respect this origin as a valid teleportation origin for all `assets`. If it does not, then the assets may be lost.

Operands:

- `assets: MultiAssetFilter`: The asset(s) to remove from the Holding Register.
- `destination: MultiLocation`: A valid location that respects teleports coming from this location.
- `xcm`: The instructions to execute *on the destination location* following the `ReceiveTeleportedAsset` instruction.

Kind: *Instruction*

Errors: *Fallible*.

### `ReportHolding`

Send a `QueryResponse` XCM message with the `assets` value equal to the holding contents, or a portion thereof.

Operands:

- `query_id: QueryId`: The value to be used for the `query_id` field of the `QueryResponse` message.
- `destination: MultiLocation`: The location to where the `QueryResponse` message should be sent.
- `assets: MultiAssetFilter`: A filter for the assets that should be reported back.
- `max_response_weight: Weight`: The value to be used for the `max_weight` field of the `QueryResponse` message.

Kind: *Instruction*

Errors: *Fallible*.

### `BuyExecution`

Pay for the execution of the current message from Holding.

Operands:

- `fees: MultiAsset`: The asset(s) by which to reduce Holding to pay execution fees.
- `weight_limit: Option<Weight>`: If provided, then state the amount of weight to be purchased. If this is lower than the estimated weight of this message, then an error will be thrown.

Kind: *Instruction*

Errors: *Fallible*.

### `RefundSurplus`

Increase Refunded Weight Register to the value of Surplus Weight Register. Attempt to accrue fees previously paid via `BuyExecution` into Holding for the amount that Refunded Weight Register is increased.

Kind: *Instruction*

Errors: *Infallible*.

### `SetErrorHandler`

Set the Error Handler Register.

Operands:

- `error_handler: Xcm`: The value to which to set the Error Handler Register.

Kind: *Instruction*

Errors: *Infallible*.

Weight: The estimated weight of this instruction must include the estimated weight of `error_handler`. At run-time, Surplus Weight Register should be increased by the estimated weight of the Error Handler prior to being changed.

### `SetAppendix`

Set the Appendix Register.

Operands:

- `appendix: Xcm`: The value to which to set the Appendix Register.

Kind: *Instruction*

Errors: *Infallible*.

Weight: The estimated weight of this instruction must include the estimated weight of `appendix`. At run-time, Surplus Weight Register should be increased by the estimated weight of the Appendix prior to being changed.

### `ClearError`

Clear the Error Register.

Kind: *Instruction*

Errors: *Infallible*.

### `ClaimAsset`

Create some assets which are being held on behalf of Origin.

Operands:

- `assets: MultiAssets`: The assets which are to be claimed. This must match exactly with the assets claimable by Origin with the given `ticket`.
- `ticket: MultiLocation`: The ticket of the asset; this is an abstract identifier to help locate the asset.

Kind: *Instruction*

Errors: *Fallible*.

### `Trap`

Always throws an error of type `Trap`.

Operands:

- `id: Compact`: The value to be used for the parameter of the thrown error.

Kind: *Instruction*

Errors: *Always*.

### `SubscribeVersion`

Send a `QueryResponse` message to Origin specifying XCM version 2 in the `response` field.

Any upgrades to the local consensus which result in a later version of XCM being supported should  elicit a similar response.

Operands:

- `query_id: QueryId`: The value to be used for the `query_id` field of the `QueryResponse` message.
- `max_response_weight: Weight`: The value to be used for the `max_weight` field of the `QueryResponse` message.

Kind: *Instruction*

### `UnsubscribeVersion`

Cancel the effect of a previous `SubscribeVersion` instruction from Origin.

Kind: *Instruction*

### `BurnAsset(MultiAssets)`

Reduce Holding by up to the given assets.

Holding is reduced by as much as possible up to the assets in the parameter.

Operands:

- `assets: MultiAssets`: The assets by which to reduce Holding.

Kind: *Instruction*

Errors: *Fallible*

### `ExpectAsset(MultiAssets)`

Throw an error if Holding does not contain at least the given assets.

Operands:

- `assets: MultiAssets`: The minimum assets expected to be in Holding.

Kind: *Instruction*

Errors:

- `ExpectationFalse`: If Holding does not contain the assets in the parameter.

### `ExpectOrigin(MultiLocation)`

Ensure that the Origin Register equals some given value and throw an error if not.

Operands:

- `origin: MultiLocation`: The value expected of the Origin Register.

Kind: *Instruction*

Errors:

- `ExpectationFalse`: If Origin is not some value, or if that value is not equal to the parameter.

### `ExpectError(Option<(u32, Error)>)`

Ensure that the Error Register equals some given value and throw an error if not.

Operands:

- `error: Option<(u32, Error)>`: The value expected of the Error Register.

Kind: *Instruction*

Errors:

- `ExpectationFalse`: If the value of the Error Register is not equal to the parameter.

## **6** Universal Asset Identifiers

*Note on versioning:* This describes the `MultiAsset` (and associates) as used in XCM version of this document, and its version is strictly implied by the XCM it is used within. If it is necessary to form a `MultiAsset` value is used _outside_ of an XCM (where its version cannot be inferred) then the version-aware `VersionedMultiAsset` should be used instead, exactly analogous to how `Xcm` relates to `VersionedXcm`.

### Description

A `MultiAsset` is a general identifier for an asset. It may represent both fungible and non-fungible assets, and in the case of a fungible asset, it represents some defined amount of the asset.

Since a `MultiAsset` value may only be used to represent a single asset, there is a `MultiAssets` type which represents a set of different assets. Sometimes it is needed to express a pattern over the universe of assets; for this purpose there is `WildMultiAsset`, which allows for "wildcard" matching. Finally, there is often occasion to provide a "selector", which might be a general pattern or a set of concrete assets, and for this there is `MultiAssetFilter`.

Fungible assets are identified by a _class_ together with an amount of the asset, the number of units of the asset class that the asset value represents. Non-fungible assets are necessarily unique, so need no _amount_, but this is replaced with an identifier for the asset instance, allowing for multiple unique assets within the same overall class.

Assets classes may be identified in one of two ways: either an abstract identifier or a concrete identifier. A single asset may be referenced from multiple asset identifiers, though will tend to have only a single *canonical* concrete identifier.

#### Abstract identifiers

Abstract identifiers are absolute identifiers that represent a notional asset which can exist within multiple consensus systems. These tend to be simpler to deal with since their broad meaning is unchanged regardless stay of the consensus system in which it is interpreted.

However, in the attempt to provide uniformity across consensus systems, they may conflate different instantiations of some notional asset (e.g. the reserve asset and a local reserve-backed derivative of it) under the same name, leading to confusion. It also implies that one notional asset is accounted for locally in only one way. This may not be the case, e.g. where there are multiple bridge instances each providing a bridged "BTC" token yet none being fungible with the others.

Since they are meant to be absolute and universal, a global registry is needed to ensure that name collisions do not occur.

An abstract identifier is represented as a simple variable-size byte string. As of writing, no global registry exists and no proposals have been put forth for asset labeling.

#### Concrete identifiers

Concrete identifiers are *relative identifiers* that specifically identify a single asset through its location in a consensus system relative to the context interpreting. Use of a `MultiLocation` ensures that similar but non-fungible variants of the same underlying asset can be properly distinguished, and obviates the need for any kind of central registry.

The limitation is that the asset identifier cannot be trivially copied between consensus systems and must instead be "re-anchored" whenever being moved to a new consensus system, using the two systems' relative paths. This is specifically because `MultiLocation` values are fundamentally relative identifiers.

Throughout XCM, messages are authored such that *when interpreted from the receiver's point of view* they will have the desired meaning/effect. This means that relative paths should always by constructed to be read from the point of view of the receiving system, *which may be have a completely different meaning in the authoring system*.

Concrete identifiers are the generally preferred way of identifying an asset since they are entirely unambiguous.

A concrete identifier is represented by a `MultiLocation`. If a system has an unambiguous primary asset (such as Bitcoin with BTC or Ethereum with ETH), then it will conventionally be identified as the chain itself. Alternative and more specific ways of referring to an asset within a system include:

- `<chain>/PalletInstance(<id>)` for a Frame chain with a single-asset pallet instance (such as an instance of the Balances pallet).
- `<chain>/PalletInstance(<id>)/GeneralIndex(<index>)` for a Frame chain with an indexed multi-asset pallet instance (such as an instance of the Assets pallet).
- `<chain>/AccountId32` for an ERC-20-style single-asset smart-contract on a Frame-based contracts chain.
- `<chain>/AccountKey20` for an ERC-20-style single-asset smart-contract on an Ethereum-like chain.

### Format

A `MultiAsset` value is represented by the SCALE-encoded pair of fields:

- `class: AssetId`: The asset class.
- `fun`: The fungibility of the asset, this is a tagged union with two possible variants:
  - `Fungible = 0 { amount: Compact }`: In which case this is a fungible asset and the `amount` is expected to follow the `0` byte to identify this variant.
  - `NonFungible = 1 { instance: AssetInstance }`: In which case this is a non-fungible asset and the `instance` is expected to follow the `1` byte to identify this variant.

If multiple classes need to be expressed in a value, then the `MultiAssets` type should be used. This is encoded exactly as a `Vec<MultiAsset>`, but with some additional requirements:

- All fungible assets must be placed in the encoding before non-fungible assets.
- Fungible assets must be ordered by class in Standard Ordering (see next).
- Non-fungible assets must be ordered by class and then by instance in Standard Ordering.
- There may be only one fungible asset for each fungible asset class.
- There may be only one non-fungible asset for each non-fungible asset class/instance pair.

(The ordering provides an efficient means of guaranteeing that each asset is indeed unique within the set.)

A `WildMultiAsset` value is represented by the SCALE-encoded tagged union with two variants:

- `All = 0`: Matches for all assets.
- `AllOf = 1 { class: AssetId, fun: WildFungibility }`: Matches for any assets which match the given `class` and fungibility (`fun`).

A `MultiAssetFilter` value is represented by the SCALE-encoded tagged union with two variants:

- `Definite = 0 { assets: MultiAssets }`: Filters for all assets in `assets`.
- `Wild = 1 { wildcard: WildMultiAsset }`: Filters for all assets which match `wildcard`.

#### Standard Ordering

XCM Standard Ordering is based on the Rust-language ordering and is defined:

- For SCALE tagged unions, values are ordered primarily by variant index (ascending) and then ordered by the first field of the variant's item, and then successive fields in case all prior fields are equivalent.
- For SCALE tuples (named or anonymous), values are ordered primarily by the first element/field, and then each successive element/field in the case all prior elements/fields are equivalent.
- For SCALE vectors (`Vec<...>`), value are ordered in standard lexicographic fashion (primarily by the first item and then each successive item in case of a shared prefix, with an item that is a strict prefix of another appearing first).

#### `AssetId`

A general identifier for an asset class. This is a SCALE-encoded tagged union (`enum` in Rust terms) with two variants:

- `Concrete = 0 { location: MultiLocation }`: A concrete asset-class identifier, given by a `MultiLocation` value.
- `Abstract = 1 { name: Vec<u8> }`: A abstract asset-class identifier, given by a `Vec<u8>` value.

#### `AssetInstance`

A general identifier for an instance of a non-fungible asset within its class.

Given by the SCALE tagged union of:

- `Undefined = 0`: Undefined - used if the NFA class has only one instance.
- `Index = 1: { index: Compact }`: A compact `index`. Technically this could be greater than u128, but this implementation supports only values up to `2**128 - 1`.
- `Array4 = 2: { datum: [u8; 4] }`: A 4-byte fixed-length `datum`.
- `Array8 = 3: { datum: [u8; 8] }`: A 8-byte fixed-length `datum`.
- `Array16 = 4: { datum: [u8; 16] }`: A 16-byte fixed-length `datum`.
- `Array32 = 5: { datum: [u8; 32] }`: A 32-byte fixed-length `datum`.
- `Blob = 6: { data: Vec<u8> }`: An arbitrary piece of `data`. Use only when necessary.

#### `WildFungibility`

A general identifier for an asset class. This is a SCALE-encoded tagged union (`enum` in Rust terms) with two variants:

- `Fungible = 0`: Matches all fungible assets.
- `NonFungible = 1`: Matches all non-fungible assets.

## **7** Universal Consensus Location Identifiers

This describes the `MultiLocation` (and associates) as used in XCM version of this document, and its version is strictly implied by the XCM it is used within. If it is necessary to form a `MultiLocation` value is used _outside_ of an XCM (where its version cannot be inferred) then the version-aware `VersionedMultiLocation` should be used instead, exactly analogous to how `Xcm` relates to `VersionedXcm`.

### **7.1** Description

A relative path between state-bearing consensus systems.

`MultiLocation` aims to be sufficiently abstract in meaning and general in nature that it is able to identify arbitrary logical "locations" within the universe of consensus.

A location in a consensus system is defined as an *isolatable state machine* held within global consensus. The location in question need not have a sophisticated consensus algorithm of its own; a single account within Ethereum, for example, could be considered a location.

A very-much non-exhaustive list of types of location include:

- A (normal, layer-1) block chain, e.g. the Bitcoin mainnet or a parachain.
- A layer-0 super-chain, e.g. the Polkadot Relay chain.
- A layer-2 smart contract, e.g. an ERC-20 on Ethereum.
- A logical functional component of a chain, e.g. a single instance of a pallet on a Frame-based Substrate chain.
- An account.

A `MultiLocation` is a *relative identifier*, meaning that it can only be used to define the relative path between two locations, and cannot generally be used to refer to a location universally. Much like a relative file-system path will first begin with any "../" components used to ascend into to the containing directory, followed by the directory names into which to descend, a `MultiLocation` has two main parts to it: the number of times to ascend into the outer consensus from the local and then an interior location within that outer consensus.

A `MultiLocation` is thus encoded as the pair of values:

- `parents: u8`: The levels of consensus to ascend before interpreting the `interior` parameter.
- `interior: InteriorMultiLocation`: A location interior to the outer consensus system found by ascending from the local system `parents` times.

### Interior Locations & Junctions

There is a second type `InteriorMultiLocation` which always identifies a consensus system *interior* to the local consensus system. Being strictly interior implies a relationship of subordination: for a consensus system A to be interior to that of B would mean that a state change of A implies a state change in B. As an example, a smart contract location within the Ethereum blockchain would be considered an interior location of the Ethereum blockchain itself.

An `InteriorMultiLocation` is comprised of a number of *junctions*, in order, each specifying a location further internal to the previous. An `InteriorMultiLocation` value with no junctions simply refers to the local consensus system.

An `InteriorMultiLocation` is thus encoded simply as a `Vec<Junction>`. A `Junction` meanwhile is encoded as the tagged union of:

- `Parachain = 0 { index: Compact<u32> }`: An indexed parachain belonging to and operated by the context. Generally used when the context is a Polkadot Relay-chain.

- `AccountId32 = 1 { network: NetworkId, id: [u8; 32] }`: A 32-byte identifier (`id`) for an account of a specific `network` that is respected as a sovereign endpoint within the context. Generally used when the context is a Substrate-based chain.

- `AccountIndex64 = 2 { network: NetworkId, index: Compact<u64> }`: An 8-byte `index` for an account of a specific `network` that is respected as a sovereign endpoint within the context. May be used when the context is a Frame-based chain and includes e.g. an indices pallet. The `network` id may be used to restrict or qualify the meaning of the index and may not refer specifically to a *blockchain* network. An example might be a smart contract chain which uses different `network` values to distinguish between account indices and smart contracts indices.

- `AccountKey20 = 3 { network: NetworkId, key: [u8; 20] }`: A 20-byte identifier `key` for an account of a specific `network` that is respected as a sovereign endpoint within the context. May be used when the context is an Ethereum or Bitcoin chain or smart-contract.

- `PalletInstance = 4 { index: u8 }`: An instanced, `index`ed pallet that forms a constituent part of the context. Generally used when the context is a Frame-based chain.

- `GeneralIndex = 5 { index: Compact }`: A nondescript `index` within the context location. Usage will vary widely owing to its generality. Note: Try to avoid using this and instead use a more specific item.

- `GeneralKey = 6 { key: Vec<u8> }`: A nondescript datum acting as a `key` within the context location. Usage will vary widely owing to its generality. Note: Try to avoid using this and instead use a more specific item.

- `OnlyChild = 7`: The unambiguous child.

#### NetworkId

A global identifier of an account-bearing consensus system.

Encoded as the tagged union of:

- `Any = 0`: Unidentified/any.
- `Named = 1 { name: Vec<u8> }`: Some `name`d network.
- `Polkadot = 2`: The Polkadot Relay chain
- `Kusama = 3`: Kusama.

### Written format

Note: `MultiLocation`s will tend to be written using junction names delimited by slashes, evoking the syntax of other logical path systems such as URIs and file systems. E.g. a `MultiLocation` value expressed as `../PalletInstance(3)/GeneralIndex(42)` would be a `MultiLocation` with one parent and two `Junction`s: `PalletInstance{index: 3}` and `GeneralIndex{index: 42}`.

## **8** The types of error in XCM

Within XCM it is necessary to communicate some problem encountered while executing some message. The type `Error` allows for this to be expressed, and is encoded as the SCALE tagged union of:

- `Overflow = 0`: An arithmetic overflow happened.
- `Unimplemented = 1`: The instruction is intentionally unsupported.
- `UntrustedReserveLocation = 2`: Origin Register does not contain a value value for a reserve transfer notification.
- `UntrustedTeleportLocation = 3`: Origin Register does not contain a value value for a teleport notification.
- `MultiLocationFull = 4`: `MultiLocation` value too large to descend further.
- `MultiLocationNotInvertible = 5`: `MultiLocation` value ascend more parents than known ancestors of local location.
- `BadOrigin = 6`: The Origin Register does not contain a valid value for instruction.
- `InvalidLocation = 7`: The location parameter is not a valid value for the instruction.
- `AssetNotFound = 8`: The given asset is not handled.
- `FailedToTransactAsset = 9`: An asset transaction (like withdraw or deposit) failed (typically due to type conversions).
- `NotWithdrawable = 10`: An asset cannot be withdrawn, potentially due to lack of ownership, availability or rights.
- `LocationCannotHold = 11`: An asset cannot be deposited under the ownership of a particular location.
- `ExceedsMaxMessageSize = 12`: Attempt to send a message greater than the maximum supported by the transport protocol.
- `DestinationUnsupported = 13`: The given message cannot be translated into a format supported by the destination.
- `Transport = 14`: Destination is routable, but there is some issue with the transport mechanism.
- `Unroutable = 15`: Destination is known to be unroutable.
- `UnknownClaim = 16`: Used by `ClaimAsset` when the given claim could not be recognized/found.
- `FailedToDecode = 17`: Used by `Transact` when the functor cannot be decoded.
- `TooMuchWeightRequired = 18`: Used by `Transact` to indicate that the given weight limit could be breached by the functor.
- `NotHoldingFees = 19`: Used by `BuyExecution` when the Holding Register does not contain payable fees.
- `TooExpensive = 20`: Used by `BuyExecution` when the fees declared to purchase weight are insufficient.
- `Trap(u64) = 21`: Used by the `Trap` instruction to force an error intentionally. Its code is included.
- `ExpectationFalse = 22`: Used by `ExpectAsset`, `ExpectError` and `ExpectOrigin` when the expectation was not true.
