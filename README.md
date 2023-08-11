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
- *XCVM* The Cross-Consensus Virtual Machine, for which the definition of XCM messages in large part relies upon.
- *Reserve Location* The *Consensus System* which acts as the reserve for a particular assets on a particular (derivative) *Consensus System*. The reserve *Consensus System* is always known by the derivative. It will have a *Sovereign Account* for the derivative which contains full collateral for the derivative assets.
- *Origin* The *Consensus System* from which a given message has been (directly and immediately) delivered. Specified as a `MultiLocation`.
- *XcmContext* Contextual data pertaining to a specific list of XCM instructions.  It includes the `origin`, specified by a `MultiLocation`, `message_hash`, the hash of the XCM, an arbitrary `topic`, as well as an optional `MultiLocation` of a claimer to the dropped assets.
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
- *Transact Status*
- *Topic*

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

### **3.10** Transact Status

Of type `MaybeErrorCode`, initialized to `MaybeErrorCode::Success`.

Expresses the result of a [`Transact`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L482) instruction after the encoded call within has been dispatched.

### **3.11** Topic

Of type `Option<[u8; 32]>`, initialized to `None`.

Expresses an arbitrary topic of an XCM.  This value can be set to anything, and is used as part of `XcmContext`.


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

- `WithdrawAsset = 0 (MultiAssets)`
- `ReserveAssetDeposited = 1 (MultiAssets)`
- `ReceiveTeleportedAsset = 2 (MultiAssets)`
- `QueryResponse = 3 {query_id: Compact<QueryId>, response: Response, max_weight: Weight, querier: Option<MultiLocation> }`
- `TransferAsset = 4 { assets: MultiAssets, beneficiary: MultiLocation }`
- `TransferReserveAsset = 5 { assets: MultiAssets, dest: MultiLocation, xcm: Xcm<()> }`
- `Transact = 6 { origin_kind: OriginKind, require_weight_at_most: Weight, call: DoubleEncoded<Call> }`
- `HrmpNewChannelOpenRequest = 7 { sender: Compact<u32>, max_message_size: Compact<u32>, max_capacity: Compact<u32> }`
- `HrmpChannelAccepted = 8 { recipient: Compact<u32> }`
- `HrmpChannelClosing = 9 { initiator: Compact<u32>, sender: Compact<u32>, recipient: Compact<u32> }`
- `ClearOrigin = 10`
- `DescendOrigin = 11 (InteriorMultiLocation)`
- `ReportError = 12 (QueryResponseInfo)`
- `DepositAsset = 13 { assets: MultiAssetFilter, beneficiary: MultiLocation }`
- `DepositReserveAsset = 14 { assets: MultiAssetFilter, dest: MultiLocation, xcm: Xcm<()> }`
- `ExchangeAsset = 15 { give: MultiAssetFilter, want: MultiAssets, maximal: bool }`
- `InitiateReserveWithdraw = 16 { assets: MultiAssetFilter, reserve: MultiLocation, xcm: Xcm<()> }`
- `InitiateTeleport = 17 { assets: MultiAssetFilter, dest: MultiLocation, xcm: Xcm<()> }`
- `ReportHolding = 18 { response_info: QueryResponseInfo, assets: MultiAssetFilter }`
- `BuyExecution = 19 { fees: MultiAsset, weight_limit: WeightLimit }`
- `RefundSurplus = 20`
- `SetErrorHandler = 21 (Xcm<Call>)`
- `SetAppendix = 22 (Xcm<Call>)`
- `ClearError = 23`
- `ClaimAsset = 24 { assets: MultiAssets, ticket: MultiLocation }`
- `Trap = 25 (Compact<u64>)`
- `SubscribeVersion = 26 {query_id: Compact<QueryId>, max_response_weight: Weight}`
- `UnsubscribeVersion = 27`
- `BurnAsset = 28 (MultiAssets)`
- `ExpectAsset = 29 (MultiAssets)`
- `ExpectOrigin = 30 (Option<MultiLocation>)`
- `ExpectError = 31 (Option<(u32, Error)>)`
- `ExpectTransactStatus = 32 (MaybeErrorCode)`
- `QueryPallet = 33 { module_name: Vec<u8>, response_info: QueryResponseInfo }`
- `ExpectPallet = 34 { index: Compact<u32>, name: Vec<u8>, module_name: Vec<u8>, crate_major: Compact<u32>, min_crate_minor: Compact<u32> }`
- `ReportTransactStatus = 35 (QueryResponseInfo)`
- `ClearTransactStatus = 36`
- `UniversalOrigin = 37 (Junction)`
- `ExportMessage = 38 { network: NetworkId, destination: InteriorMultiLocation, xcm: Xcm<()> }`
- `LockAsset = 39 { asset: MultiAsset, unlocker: MultiLocation }`
- `UnlockAsset = 40 { asset: MultiAsset, target: MultiLocation }`
- `NoteUnlockable = 41 { asset: MultiAsset, owner: MultiLocation }`
- `RequestUnlock = 42 { asset: MultiAsset, locker: MultiLocation }`
- `SetFeesMode = 43 { jit_withdraw: bool }`
- `SetTopic = 44 ([u8; 32])`
- `ClearTopic = 45`
- `AliasOrigin = 46 (MultiLocation)`
- `UnpaidExecution = 47 { weight_limit: WeightLimit, check_origin: Option<MultiLocation> }`

### Notes on terminology

- When the term _Origin_ is used, it is meant to mean "location whose value is that of the Origin Register". Thus the phrase "controlled by the Origin" is equivalent to "controlled by the location whose value is that of the Origin Register".
- When the term _Holding_ is used, it is meant to mean "value of the Holding Register". Thus the phrase "reduce Holding by the asset" is equivalent to "reduce the value of the Holding Register by the asset".
- Similarly for _Appendix_ (the value of the Appendix Register) and _Error Handler_ (the value of the Error Handler Register).
- The term _on-chain_ should be taken to mean "within the persistent state of the local consensus system", and should not be considered to limit the current consensus system to one which is specifically a blockchain system.
- The type `Compact`, `Weight` and `QueryId` should be intended to mean a compact integer as per the SCALE definition.

### [`WithdrawAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L380)

Withdraw asset(s) (`assets`) from the ownership of `origin` and place them into the Holding Register.

Operands:

- `assets: MultiAssets`: The asset(s) to be withdrawn into holding.

Kind: *Instruction*.

Errors:
- `BadOrigin`
- `AssetNotFound`
- `FailedToTransactAsset`
- `HoldingWouldOverflow`

### [`ReserveAssetDeposited `](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L393)

Asset(s) (`assets`) have been received into the ownership of this system on the `origin` system and equivalent derivatives should be placed into the Holding Register.

Operands:

- `assets: MultiAssets`: The asset(s) that are minted into holding.

Kind: *Trusted Indication*.

Safety: `origin` must be trusted to have received and be storing `assets` such that they may later be withdrawn should this system send a corresponding message.

Errors:
- `BadOrigin`
- `UntrustedReserveLocation`
- `HoldingWouldOverflow`

### [`ReceiveTeleportedAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L406)

Asset(s) (`assets`) have been destroyed on the `origin` system and equivalent assets should be created and placed into the Holding Register.

Operands:

- `assets: MultiAssets`: The asset(s) that are minted into the Holding Register.

Kind: *Trusted Indication*.

Safety:`origin` must be trusted to have irrevocably destroyed the corresponding `assets`
prior as a consequence of sending this message.

Errors: 
- `BadOrigin`
- `UntrustedTeleportLocation`
- `AssetNotFound`
- `NotWithdrawable`
- `HoldingWouldOverflow`

### [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426)

Respond with information that the local system is expecting.

Operands:

- `query_id: QueryId`: The identifier of the query that resulted in this message being sent.
- `response: Response`: The message content.
- `max_weight: Weight`: The maximum weight that handling this response should take. 
- `querier`: The location responsible for the initiation of the response, if there is one. In general this will tend to be the same location as the receiver of this message. 
NOTE: As usual, this is interpreted from the perspective of the receiving consensus
system.

Kind: *Information*.

Safety: Since this is information only, there are no immediate concerns. However, it should be remembered that even if the Origin behaves reasonably, it can always be asked to make a response to a third-party chain who may or may not be expecting the response. Therefore the `querier` should be checked to match the expected value.

Errors: 
- `BadOrigin`

#### `Response`

The `Response` type is used to express information content in the [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) XCM instruction. It can represent one of several different data types and it therefore encoded as the SCALE-encoded tagged union:

- `Null = 0`:  No response. Serves as a neutral default.
- `Assets = 1 (MultiAssets)`: Some assets.
- `ExecutionResult = 2 (Option<(u32, Error)>)`: The outcome of an XCM instruction.
- `Version = 3 (u32)`: An XCM version.
- `PalletsInfo = 4 (BoundedVec<PalletInfo, MaxPalletsInfo>)`: The index, instance name, pallet name and version of some pallets.
- `DispatchResult = 5 (MaybeErrorCode)`: The status of a dispatch attempt using [`Transact`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L482).

### [`TransferAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L445)

Withdraw asset(s) (`assets`) from the ownership of `origin` and place equivalent assets under the ownership of `beneficiary`.

Operands:

- `assets: MultiAssets`: The asset(s) to be withdrawn.
- `beneficiary: MultiLocation`: The new owner for the assets.

Kind: *Instruction*.

Errors: 
- `BadOrigin`
- `AssetNotFound`
- `FailedToTransactAsset`

### [`TransferReserveAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L465)

Withdraw asset(s) (`assets`) from the ownership of `origin` and place equivalent assets under the ownership of `dest` within this consensus system (i.e. its sovereign account).

Send an onward XCM message to `destination` of [`ReserveAssetDeposited `](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L393) with the given `xcm`.

Operands:

- `assets: MultiAsset`: The asset(s) to be withdrawn.
- `destination: MultiLocation`: The location whose sovereign account will own the assets and thus the effective beneficiary for the assets and the notification target for the reserve asset deposit message.
- `xcm: Xcm`: The instructions that should follow the [`ReserveAssetDeposited `](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L393) instruction, which is sent onwards to `destination`.

Kind: *Instruction*.

Errors: 
- `BadOrigin`
- `AssetNotFound`
- `FailedToTransactAsset`
- `LocationFull`
- `NotHoldingFees`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`


### [`Transact`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L482)

Apply the encoded transaction `call`, whose dispatch-origin should be `origin` as expressed by the kind of origin `origin_kind`.

The Transact Status Register is set according to the result of dispatching the call.

Operands:

- `origin_kind: OriginKind`: The means of expressing the message origin as a dispatch origin.
- `require_weight_at_most: Weight`: The weight of `call`; this should be at least the chain's calculated weight and will be used in the weight determination arithmetic.
- `call: DoubleEncoded<Call>`: The encoded transaction to be applied.

Kind: *Instruction*.

Errors: 
- `BadOrigin`
- `FailedToDecode`
- `NoPermission`
- `MaxWeightInvalid`


Weight: Weight estimation may utilise `max_weight` which may lead to an increase in Surplus Weight Register at run-time.

### [`HrmpNewChannelOpenRequest`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L494)

A message to notify about a new incoming HRMP channel. This message is meant to be sent by the Relay-chain to a para.

Operands:

- `sender: Compact`: The sender in the to-be opened channel. Also, the initiator of the channel opening.
- `max_message_size: Compact`: The maximum size of a message proposed by the sender.
- `max_capacity: Compact`: The maximum number of messages that can be queued in the channel.

Safety: The message should originate directly from the Relay-chain.

Kind: *System Notification*

### [`HrmpChannelAccepted`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L512)

A message to notify about that a previously sent open channel request has been accepted by the recipient. That means that the channel will be opened during the next Relay-chain session change. This message is meant to be sent by the Relay-chain to a para.

Operands:

- `recipient: Compact`: The recipient parachain which has accepted the previous open-channel request.

Safety: The message should originate directly from the Relay-chain.

Kind: *System Notification*

Errors: 

### [`HrmpChannelClosing`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L529)

A message to notify that the other party in an open channel decided to close it. In particular, `initiator` is going to close the channel opened from `sender` to the `recipient`. The close will be enacted at the next Relay-chain session change. This message is meant to be sent by the Relay-chain to a para.

Operands:

- `initiator: Compact`: The parachain index which initiated this close operation.
- `sender: Compact`: The parachain index of the sender side of the channel being closed.
- `recipient: Compact`: The parachain index of the receiver side of the channel being closed.

Safety: The message should originate directly from the Relay-chain.

Kind: *System Notification*

Errors: 

### [`ClearOrigin`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L549)

Clear the Origin Register.

This may be used by the XCM author to ensure that later instructions cannot command the authority of the Original Origin (e.g. if they are being relayed from an untrusted source, as often the case with [`ReserveAssetDeposited `](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L393)).

Kind: *Instruction*.

Errors: *Infallible*.

### [`DescendOrigin`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L556)

Mutate the origin to some interior location.

Operands:

- `interior: InteriorMultiLocation`: The location, interpreted from the context of Origin, to place in the Origin Register.

Kind: *Instruction*

Errors: 
- `BadOrigin`
- `LocationFull`

### [`ReportError`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L567)

Immediately report the contents of the Error Register to the given destination via XCM.

A [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) message of type `ExecutionOutcome` is sent to `destination` with the given
`query_id` and the outcome of the XCM.

Operands:

- `response_info: QueryResponseInfo`: [Information](#QueryResponseInfo)  for making the response.

Kind: *Instruction*

Errors: 
- `ReanchorFailed`
- `NotHoldingFees`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`

#### `QueryResponseInfo`
Information regarding the composition of a query response.
QueryResponseInfo:
- `destination: MultiLocation`: The destination to which the query response message should be send.
- `query_id: Compact<QueryId>`: The `query_id` field of the [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) message.
- `max_weight: Weight`: The `max_weight` field of the [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) message.

### [`DepositAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L578)

Remove the asset(s) (`assets`) from the Holding Register and place equivalent assets under the ownership of `beneficiary` within this consensus system.

Operands:

- `assets: MultiAssetFilter`: The asset(s) to remove from holding.
- `beneficiary: MultiLocation`: The new owner for the assets.

Kind: *Instruction*

Errors: 
- `AssetNotFound`
- `FailedToTransactAsset`

### [`DepositReserveAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L596)

Remove the asset(s) (`assets`) from the Holding Register and place equivalent assets under the ownership of `dest` within this consensus system (i.e. deposit them into its sovereign account).

Send an onward XCM message to `dest` of [`ReserveAssetDeposited `](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L393) with the given `effects`.

Operands:

- `assets: MultiAssetFilter`: The asset(s) to remove from the Holding Register.
- `dest: MultiLocation`: The location whose sovereign account will own the assets and thus the effective beneficiary for the assets and the notification target for the reserve asset deposit message.
- `xcm: Xcm`: The orders that should follow the [`ReserveAssetDeposited `](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L393) instruction which is sent onwards to `dest`.

Kind: *Instruction*

Errors: 
- `AssetNotFound`
- `FailedToTransactAsset`
- `NotHoldingFees`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`

### [`ExchangeAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L613)

Remove the asset(s) (`want`) from the Holding Register and replace them with alternative assets.

Operands:

- `give: MultiAssetFilter`: The maximum amount of assets to remove from holding.
- `want: MultiAssets`: The minimum amount of assets which `give` should be exchanged for.
- `maximal: bool`: If `true`, then prefer to give as much as possible up to the limit of `give` and receive accordingly more. If `false`, then prefer to give as little as possible in order to receive as little as possible while receiving at least `want`.

Kind: *Instruction*

Errors: 
- `NoDeal`

### [`InitiateReserveWithdraw`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L629)

Remove the asset(s) (`assets`) from holding and send a [`WithdrawAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L380) XCM message to a reserve location.

Operands:

- `assets: MultiAssetFilter`: The asset(s) to remove from the holding.
- `reserve: MultiLocation`: A valid location that acts as a reserve for all asset(s) in `assets`. The sovereign account of this consensus system *on the reserve location* will have appropriate assets withdrawn and `effects` will be executed on them. There will typically be only one valid location on any given asset/chain combination.
- `xcm: Xcm`: The instructions to execute on the assets once withdrawn *on the reserve location*.

Kind: *Instruction*

Errors: 
- `NotHoldingFees`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`

### [`InitiateTeleport`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L645)

Remove the asset(s) (`assets`) from the Holding Register and send an XCM message beginning with [`ReceiveTeleportedAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L406) to a `destination` location.

NOTE: The `destination` location *MUST* respect this origin as a valid teleportation origin for all `assets`. If it does not, then the assets may be lost.

Operands:

- `assets: MultiAssetFilter`: The asset(s) to remove from the Holding Register.
- `destination: MultiLocation`: A valid location that respects teleports coming from this location.
- `xcm`: The instructions to execute *on the destination location* following the [`ReceiveTeleportedAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L406) instruction.

Kind: *Instruction*

Errors: 
- `AssetNotHandled`
- `NotWithdrawable`
- `NotHoldingFees`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`


### [`ReportHolding`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L659)

Report to a given destination the contents of the Holding Register. A [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) message of type `Assets` is sent to the described destination.

Operands:
- `response_info: QueryResponseInfo`: [Information](#QueryResponseInfo)  for making the response.
 `assets: MultiAssetFilter`: A filter for the assets that should be reported back. The assets reported back will be, asset-wise, *the lesser of this value and the holding register*. No wildcards will be used when reporting assets back.

Kind: *Instruction*

Errors: 
- `ReanchorFailed`
- `NotHoldingFees`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`

### [`BuyExecution`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L672)

Pay for the execution of some XCM `xcm` and `orders` with up to `weight` picoseconds of execution time, paying for this with up to `fees` from the Holding Register.

Operands:

- `fees: MultiAsset`: The asset(s) to remove from the Holding Register to pay for fees.
- `weight_limit: WeightLimit`: The maximum amount of weight to purchase; this must be at least the expected maximum weight of the total XCM to be executed for the `AllowTopLevelPaidExecutionFrom` barrier to allow the XCM be executed.

Kind: *Instruction*

Errors: 
- `NotHoldingFees`
- `Overflow`
- `TooExpensive`
- `HoldingWouldOverflow`


### [`RefundSurplus`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L679)

Refund any surplus weight previously bought with [`BuyExecution`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L672).

Kind: *Instruction*

Errors: *Infallible*.

### [`SetErrorHandler`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L695)

Set the Error Handler Register. This is code that should be called in the case of an error happening.

An error occurring within execution of this code will _NOT_ result in the error register being set, nor will an error handler be called due to it. The error handler and appendix may each still be set.

The apparent weight of this instruction is inclusive of the inner `Xcm`; the executing weight however includes only the difference between the previous handler and the new handler, which can reasonably be negative, which would result in a surplus.

Operands:

- `error_handler: Xcm`: The value to which to set the Error Handler Register.

Kind: *Instruction*

Errors: 
- `WeightNotComputable`

### [`SetAppendix`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L711)

Set the Appendix Register. This is code that should be called after code execution (including the error handler if any) is finished. This will be called regardless of whether an error occurred.

Any error occurring due to execution of this code will result in the error register being set, and the error handler (if set) firing.

The apparent weight of this instruction is inclusive of the inner `Xcm`; the executing weight however includes only the difference between the previous appendix and the new appendix, which can reasonably be negative, which would result in a surplus.

Operands:

- `appendix: Xcm`: The value to which to set the Appendix Register.

Kind: *Instruction*

Errors: 
- `WeightNotComputable`

### [`ClearError`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L718)

Clear the Error Register.

Kind: *Instruction*

Errors: *Infallible*.

### [`ClaimAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L730)

Create some assets which are being held on behalf of the origin.

Operands:

- `assets: MultiAssets`: The assets which are to be claimed. This must match exactly with the assets claimable by the origin of the ticket.
- `ticket: MultiLocation`: The ticket of the asset; this is an abstract identifier to help locate the asset.

Kind: *Instruction*

Errors: 
- `BadOrigin`
- `UnknownClaim`
- `HoldingWouldOverflow`

### [`Trap`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L738)

Always throws an error of type [`Trap`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L738).

Operands:

- `id: Compact`: The value to be used for the parameter of the thrown error.

Kind: *Instruction*

Errors: *Always*.
- [`Trap`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L738): All circumstances, whose inner value is the same as this item's inner value.

### [`SubscribeVersion`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L752)

Ask the destination system to respond with the most recent version of XCM that they support in a [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) instruction. Any changes to this should also elicit similar responses when they happen.

Operands:

- `query_id: Compact<QueryId>`: An identifier that will be replicated into the returned XCM message.
- `max_response_weight: Weight`: The maximum amount of weight that the [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) item which is sent as a reply may take to execute. 
NOTE: If this is unexpectedly large then the response may not execute at all.

Kind: *Instruction*

Errors: 
- `BadOrigin`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`
- `InvalidLocation`

### [`UnsubscribeVersion`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L763)

Cancel the effect of a previous [`SubscribeVersion`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L752) instruction.

Kind: *Instruction*

Errors: 
- `BadOrigin`

### [`BurnAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L774)

Reduce Holding by up to the given assets.

Holding is reduced by as much as possible up to the assets in the parameter. It is not an error if the Holding does not contain the assets (to make this an error, use [`ExpectAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L782) prior).

Operands:

- `assets: MultiAssets`: The assets by which to reduce Holding.

Kind: *Instruction*

Errors: *Infallible*

### [`ExpectAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L782)

Throw an error if Holding does not contain at least the given assets.

Operands:

- `assets: MultiAssets`: The minimum assets expected to be in Holding.

Kind: *Instruction*

Errors:
- `ExpectationFalse`: If Holding does not contain the assets in the parameter.

### [`ExpectOrigin`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L790)

Ensure that the Origin Register equals some given value and throw an error if not.

Operands:

- `origin: Option<MultiLocation>`: The value expected of the Origin Register.

Kind: *Instruction*

Errors:
- `ExpectationFalse`: If Origin is not some value, or if that value is not equal to the parameter.

### [`ExpectError`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L798)

Ensure that the Error Register equals some given value and throw an error if not.

Operands:

- `error: Option<(u32, Error)>`: The value expected of the Error Register.

Kind: *Instruction*

Errors:
- `ExpectationFalse`: If the value of the Error Register is not equal to the parameter.

### [`ExpectTransactStatus`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L807)

Ensure that the Transact Status Register equals some given value and throw an error if not.

Operands:
- `maybe_error_code: MaybeErrorCode`: Expected status of the Transact Status Register.

Kind: *Instruction*

Errors:
- `ExpectationFalse`: If the value of the Transact Status Register is not equal to the parameter.

### [`QueryPallet`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L823)

Queries the existence of a particular pallet type.

Operands:
 - `module_name: Vec<u8>`: The module name of the pallet to query.
 - `response_info: QueryResponseInfo`: [Information](#QueryResponseInfo)  for making the response.

Sends a [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) to Origin whose data field `PalletsInfo` containing the information of all pallets on the local chain whose name is equal to `name`. This is empty in the case that the local chain is not based on Substrate Frame.

Kind: *Instruction*

Errors: 
- `Overflow`
- `ReanchorFailed`
- `NotHoldingFees`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`

### [`ExpectPallet`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L839)

Ensure that a particular pallet with a particular version exists.
Safety
Operands:
 - `index: Compact`: The index which identifies the pallet. An error if no pallet exists at this index.
 - `name: Vec<u8>`: Name which must be equal to the name of the pallet.
 - `module_name: Vec<u8>`: Module name which must be equal to the name of the module in which the pallet exists.
 - `crate_major: Compact`: Version number which must be equal to the major version of the crate which implements the pallet.
 - `min_crate_minor: Compact`: Version number which must be at most the minor version of the crate which implements the pallet.

Kind: *Instruction*

Errors:
- `PalletNotFound`
- `NameMismatch`
- `VersionIncompatible`

### [`ReportTransactStatus`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L861)

Send a [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) message containing the value of the Transact Status Register to some destination.

Operands:
- `query_response_info: QueryResponseInfo`: The [information](#QueryResponseInfo)  needed for constructing and sending the  [`QueryResponse`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L426) message.

Kind: *Instruction*

Errors: 
- `ReanchorFailed`
- `NotHoldingFees`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`

### [`ClearTransactStatus`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L870)

Set the Transact Status Register to its default, cleared, value.

Kind: *Instruction*

Errors: *Infallible*.

### [`UniversalOrigin`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L885)

Set the Origin Register to be some child of the Universal Ancestor.

Safety: Should only be usable if the Origin is trusted to represent the Universal Ancestor child in general. In general, no Origin should be able to represent the Universal Ancestor child which is the root of the local consensus system since it would by extension allow it to act as any location within the local consensus.

Operands:
- `junction: Junction`: The `Junction` parameter should generally be a `GlobalConsensus` variant since it is only these which are children of the Universal Ancestor.

Kind: *Instruction*

Error: 
- `BadOrigin`
- `InvalidLocation`

### [`ExportMessage`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L906)

Send a message on to Non-Local Consensus system.

This will tend to utilize some extra-consensus mechanism, the obvious one being a bridge. A fee may be charged; this may be determined based on the contents of `xcm`. It will be taken from the Holding register.

Operands: 
- `network: NetworkId`: The remote consensus system to which the message should be exported.
- `destination: InteriorMultiLocation`: The location relative to the remote consensus system to which the message should be sent on arrival.
- `xcm: Xcm`: The message to be exported.

As an example, to export a message for execution on Statemine (parachain #1000 in the
Kusama network), you would call with `network: NetworkId::Kusama` and
`destination: X1(Parachain(1000))`. Alternatively, to export a message for execution on
Polkadot, you would call with `network: NetworkId:: Polkadot` and `destination: Here`.

Kind: *Instruction*

Errors: 
- `BadOrigin`
- `Unanchored`
- `Unroutable`
- `NotHoldingFees`
- `AssetNotFound`
- `FailedToTransactAsset`

### [`LockAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L922)

Lock the locally held asset and prevent further transfer or withdrawal.

This restriction may be removed by the [`UnlockAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L935) instruction being called with an Origin of `unlocker` and a `target` equal to the current `Origin`.

If the locking is successful, then a [`NoteUnlockable`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L950) instruction is sent to `unlocker`.

Operands:
- `asset: MultiAsset`: The asset(s) which should be locked.
- `unlocker: MultiLocation`: The value which the Origin must be for a corresponding [`UnlockAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L935) instruction to work.

Kind: *Instruction*

Errors: 
- `BadOrigin`
- `ReanchorFailed`
- `AssetNotFound`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`
- `NotHoldingFees`
- `FailedToTransactAsset`
- `LockError`

### [`UnlockAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L935)

Remove the lock over `asset` on this chain and (if nothing else is preventing it) allow the asset to be transferred.

Operands:
- `asset: MultiAsset`: The asset to be unlocked.
- `target: MultiLocation`: The owner of the asset on the local chain.

Kind: *Instruction*

Errors: 
- `BadOrigin`
- `AssetNotFound`
- `LockError`

### [`NoteUnlockable`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L950)

Asset (`asset`) has been locked on the `origin` system and may not be transferred. It may only be unlocked with the receipt of the [`UnlockAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L935) instruction from this chain.

Operands:

- `asset: MultiAsset`: The asset(s) which are now unlockable from this origin.
- `owner: MultiLocation`: The owner of the asset on the chain in which it was locked. This may be a location specific to the origin network.

Safety: `origin` must be trusted to have locked the corresponding `asset` prior as a consequence of sending this message.

Kind: *Trusted Indication*

Errors: 
- `BadOrigin`
- `AssetNotFound`
- `LockError`

### [`RequestUnlock`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L964)

Send an [`UnlockAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L935) instruction to the `locker` for the given `asset`. This may fail if the local system is making use of the fact that the asset is locked or, of course, if there is no record that the asset actually is locked.

Operands:
- `asset`: The asset(s) to be unlocked.
- `locker`: The location from which a previous [`NoteUnlockable`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L950) was sent and to which an [`UnlockAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L935) should be sent.

Kind: *Instruction*

Errors: 
- `BadOrigin`
- `ReanchorFailed`
- `AssetNotFound`
- `Unroutable`
- `DestinationUnsupported`
- `ExceedsMaxMessageSize`
- `Transport`
- `NotHoldingFees`
- `FailedToTransactAsset`
- `LockError`

### [`SetFeesMode`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L974)

Sets the Fees Mode Register.
	
- `jit_withdraw: bool`: The fees mode item; If true, then the fee assets are taken directly from the origin's on-chain account, otherwise the fee assets are taken from the holding register.

Kind: *Instruction*.

Errors: *Infallible*

### [`SetTopic`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L983)

Set the Topic Register

Kind: *Instruction*

Errors: *Infallible*

### [`ClearTopic`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L990)

Clear the Topic Register

Kind: *Instruction*

Errors: *Infallible*

### [`AliasOrigin`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L997)

Alter the current Origin to another given origin

Operands:
- `origin: MultiLocation`

Errors: 
- `NoPermission`

### [`UnpaidExecution`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L1009)

A directive to indicate that the origin expects free execution of the message.

At execution time, this instruction just does a check on the Origin register. However, at the barrier stage, messages starting with this instruction can be disregarded if the origin is not acceptable for free execution, or the `weight_limit` is `Limited` and insufficient.

Operands:
- `weight_limit: WeightLimit`
- `check_origin: Option<MultiLocation>`

Kind: *Indication*

Errors: 
- `BadOrigin`

### `SetAssetClaimer`

Set a claimer to the dropped assets.

The dropped assets are the assets that are still located in the `Holding Register` after a program finishes its execution.

Operands:
- `origin: MultiLocation`: a `MultiLocation` identifying the claimer of the dropped assets.

Kind: *Instruction*

Errors: *Infallible*

## **6** Universal Asset Identifiers

*Note on versioning:* This describes the `MultiAsset` (and associates) as used in XCM version of this document, and its version is strictly implied by the XCM it is used within. If it is necessary to form a `MultiAsset` value that is used _outside_ of an XCM (where its version cannot be inferred) then the version-aware `VersionedMultiAsset` should be used instead, exactly analogous to how `Xcm` relates to `VersionedXcm`.

### Description

A `MultiAsset` is a general identifier for an asset. It may represent both fungible and non-fungible assets, and in the case of a fungible asset, it represents some defined amount of the asset.

Since a `MultiAsset` value may only be used to represent a single asset, there is a `MultiAssets` type which represents a set of different assets. Sometimes it is needed to express a pattern over the universe of assets; for this purpose there is `WildMultiAsset`, which allows for "wildcard" matching. Finally, there is often occasion to provide a "selector", which might be a general pattern or a set of concrete assets, and for this there is `MultiAssetFilter`.

Fungible assets are identified by a _class_ together with an amount of the asset, the number of units of the asset class that the asset value represents. Non-fungible assets are necessarily unique, so need no _amount_, but this is replaced with an identifier for the asset instance, allowing for multiple unique assets within the same overall class.

Assets classes may be identified in one of two ways: either an abstract identifier or a concrete identifier. A single asset may be referenced from multiple asset identifiers, though will tend to have only a single *canonical* concrete identifier.

#### Abstract identifiers

Abstract identifiers are absolute identifiers that represent a notional asset which can exist within multiple consensus systems. These tend to be simpler to deal with since their broad meaning is unchanged regardless of the consensus system in which it is interpreted.

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

- `id: AssetId`: The asset id.
- `fun`: The fungibility of the asset, this is a tagged union with two possible variants:
  - `Fungible = 0 { amount: Compact<128> }`: In which case this is a fungible asset and the `amount` is expected to follow the `0` byte to identify this variant.
  - `NonFungible = 1 { instance: AssetInstance }`: In which case this is a non-fungible asset and the `instance` is expected to follow the `1` byte to identify this variant.

If multiple ids need to be expressed in a value, then the `MultiAssets` type should be used. This is encoded exactly as a `Vec<MultiAsset>`, but with some additional requirements:

- All fungible assets must be placed in the encoding before non-fungible assets.
- Fungible assets must be ordered by id in Standard Ordering (see next).
- Non-fungible assets must be ordered by id and then by instance in Standard Ordering.
- There may be only one fungible asset for each fungible asset id.
- There may be only one non-fungible asset for each non-fungible asset id/instance pair.

(The ordering provides an efficient means of guaranteeing that each asset is indeed unique within the set.)

A `WildMultiAsset` value is represented by the SCALE-encoded tagged union with four variants:

- `All = 0`: Matches for all assets.
- `AllOf = 1 { id: AssetId, fun: WildFungibility }`: Matches for any assets which match the given `id` and fungibility (`fun`).
- `AllCounted = 2 ( Compact<u32> )`: Matches for all assets up to `u32` number of individual assets. 
- `AllOfCounted = 3 { id: AssetId, fun: WildFungibility, count: Compact<u32> }`: Matches for any assets which match the given `id` and fungibility (`fun`) up to `u32` number of individual assets. 

_Note: different instances of non-fungibles are counted as individual assets_

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
- `Index = 1: { index: Compact<u128> }`: A compact `index`. Technically this could be greater than u128, but this implementation supports only values up to `2**128 - 1`.
- `Array4 = 2: { datum: [u8; 4] }`: A 4-byte fixed-length `datum`.
- `Array8 = 3: { datum: [u8; 8] }`: A 8-byte fixed-length `datum`.
- `Array16 = 4: { datum: [u8; 16] }`: A 16-byte fixed-length `datum`.
- `Array32 = 5: { datum: [u8; 32] }`: A 32-byte fixed-length `datum`.


#### `WildFungibility`

A general identifier for an asset class. This is a SCALE-encoded tagged union (`enum` in Rust terms) with two variants:

- `Fungible = 0`: Matches all fungible assets.
- `NonFungible = 1`: Matches all non-fungible assets.

## **7** Universal Consensus Location Identifiers

This describes the `MultiLocation` (and associates) as used in XCM version of this document, and its version is strictly implied by the XCM it is used within. If it is necessary to form a `MultiLocation` value that is used _outside_ of an XCM (where its version cannot be inferred) then the version-aware `VersionedMultiLocation` should be used instead, exactly analogous to how `Xcm` relates to `VersionedXcm`.

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

- `AccountId32 = 1 { network: Option<NetworkId>, id: [u8; 32] }`: A 32-byte identifier (`id`) for an account of a specific `network` that is respected as a sovereign endpoint within the context. Generally used when the context is a Substrate-based chain.

- `AccountIndex64 = 2 { network: Option<NetworkId>, index: Compact<u64> }`: An 8-byte `index` for an account of a specific `network` that is respected as a sovereign endpoint within the context. May be used when the context is a Frame-based chain and includes e.g. an indices pallet. The `network` id may be used to restrict or qualify the meaning of the index and may not refer specifically to a *blockchain* network. An example might be a smart contract chain which uses different `network` values to distinguish between account indices and smart contracts indices.

- `AccountKey20 = 3 { network: Option<NetworkId>, key: [u8; 20] }`: A 20-byte identifier `key` for an account of a specific `network` that is respected as a sovereign endpoint within the context. May be used when the context is an Ethereum or Bitcoin chain or smart-contract.

- `PalletInstance = 4 (u8)`: An instanced, indexed pallet that forms a constituent part of the context. Generally used when the context is a Frame-based chain.

- `GeneralIndex = 5 (Compact<u128>)`: A nondescript index within the context location. Usage will vary widely owing to its generality. Note: Try to avoid using this and instead use a more specific item.

- `GeneralKey = 6 { length: u8, data: [u8;32] }`: A nondescript array datum, 32 bytes, acting as a key within the context location. Usage will vary widely owing to its generality. Note: Try to avoid using this and instead use a more specific item.

- `OnlyChild = 7`: The unambiguous child.

- `Plurality = 8 { id: BodyId, part: BodyPart }` A pluralistic body existing within consensus. Typical to be used to represent a governance origin of a chain, but can, in principle, also be used to represent things such as multisigs.

- `GlobalConsensus = 9 (NetworkId)` A global network capable of externalizing its own consensus, i.e., a relay chain who wishes to make its existence known to other global consensus systems (such as another relay chain). This is not generally meaningful outside of the universal level.  

#### NetworkId

A global identifier of an account-bearing consensus system.

Encoded as the tagged union of:

- `ByGenesis = 0 ([u8; 32])`: Network specified by the first 32 bytes of its genesis block.
- `ByFork = 1 { block_number: u64, block_hash: [u8; 32] }`: Network defined by the first 32-bytes of the hash and number of some block it contains.
- `Polkadot = 2`: The Polkadot mainnet Relay-chain
- `Kusama = 3`: The Kusama canary-net Relay-chain.
- `Westend = 4`: The Westend testnet Relay-chain.
- `Rococo = 5`: The Rococo testnet Relay-chain.
- `Wococo = 6`: The Wococo testnet Relay-chain.
- `Ethereum = 7 { chain_id: Compact<u64> }`: An Ethereum network specified by its EIP-155 chain ID.
- `BitcoinCore = 8`: The Bitcoin network, including hard-forks supported by Bitcoin Core development team.
- `BitcoinCash = 9`:  The Bitcoin network, including hard-forks supported by Bitcoin Cash developers.

#### BodyId
An identifier of a pluralistic body.

Encoded as the tagged union of:

- `Unit = 0`: The only body in its context. It should be unambiguous to which body `Unit` refers. The `Unit` id should never be used in a context with multiple pluralistic bodies.
- `Moniker = 1 [u8; 4]`: A named body.
- `Index = 2 (Compact<u32>)`: An indexed body.
- `Executive = 3`: The unambiguous executive body (for Polkadot, this would be the Polkadot council).
- `Technical = 4`: The unambiguous technical body (for Polkadot, this would be the Technical Committee).
- `Legislative = 5`: The unambiguous legislative body (for Polkadot, this could be considered the opinion of a majority of lock-voters).
- `Judicial = 6`: The unambiguous judicial body (this doesn't exist on Polkadot, but if it were to get a "grand oracle", it may be considered as that).
- `Defense = 7`: The unambiguous defense body (for Polkadot, an opinion on the topic given via a public referendum on the `staking_admin` track).
- `Administration = 8`: The unambiguous administration body (for Polkadot, an opinion on the topic given via a public referendum on the `general_admin` track).
- `Treasury = 9`: The unambiguous treasury body (for Polkadot, an opinion on the topic given via a public referendum on the `treasurer` track).

#### BodyPart

A part of a pluralistic body.

Encoded as the tagged union of:

- `Voice = 0`: The body's declaration, under whatever means it decides.
- `Members = 1 { count: Compact<u32> }`: A given number of members of the body.
- `Fraction = 2 { nom: Compact<u32>, denom: Compact<u32> }`: A given number of members of the body, out of some larger caucus.
- `AtLeastProportion = 3 { nom: Compact<u32>, denom: Compact<u32> }`:  No less than the given proportion of members of the body.
- `MoreThanProportion = 4 { nom: Compact<u32>, denom: Compact<u32> }`: More than the given proportion of members of the body.



### Written format

Note: `MultiLocation`s will tend to be written using junction names delimited by slashes, evoking the syntax of other logical path systems such as URIs and file systems. E.g. a `MultiLocation` value expressed as `../PalletInstance(3)/GeneralIndex(42)` would be a `MultiLocation` with one parent and two `Junction`s: `PalletInstance{index: 3}` and `GeneralIndex{index: 42}`.

## **8** The types of error in XCM

Within XCM it is necessary to communicate some problem encountered while executing some message. The type `Error` allows for this to be expressed, and is encoded as the SCALE tagged union of:

- `Overflow = 0`: An arithmetic overflow happened.
- `Unimplemented = 1`: The instruction is intentionally unsupported.
- `UntrustedReserveLocation = 2`: Origin Register does not contain a value value for a reserve transfer notification.
- `UntrustedTeleportLocation = 3`: Origin Register does not contain a value value for a teleport notification.
- `LocationFull = 4`: `MultiLocation` value too large to descend further.
- `LocationNotInvertible = 5`: `MultiLocation` value ascend more parents than known ancestors of local location.
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
- `UnknownClaim = 16`: Used by [`ClaimAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L730) when the given claim could not be recognized/found.
- `FailedToDecode = 17`: Used by [`Transact`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L482) when the functor cannot be decoded.
- `MaxWeightInvalid = 18`: Used by [`Transact`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L482) to indicate that the given weight limit could be reached by the functor.
- `NotHoldingFees = 19`: Used by [`BuyExecution`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L672) when the Holding Register does not contain payable fees.
- `TooExpensive = 20`: Used by [`BuyExecution`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L672) when the fees declared to purchase weight are insufficient.
- `Trap(u64) = 21`: Used by the [`Trap`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L738) instruction to force an error intentionally. Its code is included.
- `ExpectationFalse = 22`: Used by [`ExpectAsset`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L782), [`ExpectError`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L798) and [`ExpectOrigin`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L790) when the expectation was not true.
- `PalletNotFound = 23`: Used by ['ExpectPallet'](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L839) when the provided pallet index was not found.
- `NameMismatch = 24`:  Used by ['ExpectPallet'](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L839) when the given pallet's name is different to that expected.
- `VersionIncompatible = 25`:  Used by ['ExpectPallet'](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L839) when the given pallet's version has an incompatible version to that expected.
- `HoldingWouldOverflow = 26`: The given operation would lead to an overflow of the Holding Register.
- `ExportError = 27`: The message was unable to be exported.
- `ReanchorFailed = 28`: `MultiLocation` value failed to be reanchored.
- `NoDeal = 29`: Used by ['ExchangeAsset'](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L613) when no deal is possible under the given constraints.
- `FeesNotMet = 30`: Fees were required which the origin could not pay.
- `LockError = 31`: Some other error with locking.
- `NoPermission = 32`:  Used by [`Transact`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L482) when the state was not in a condition where the operation was valid to make.
- `Unanchored = 33`: Used by [`ExportMessage`](https://github.com/paritytech/polkadot/blob/962bc21352f5f80a580db5a28d05154ede4a9f86/xcm/src/v3/mod.rs#L906) when the universal location of the local consensus is improper.
- `NotDepositable = 34`: An asset cannot be deposited, probably because (too much of) it already exists.
