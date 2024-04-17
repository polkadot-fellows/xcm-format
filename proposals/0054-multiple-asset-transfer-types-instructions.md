---
Title: Cross-chain multi-type asset transfers
Number: 54
Status: Draft
Version: 1
Authors:
 - Adrian Catangiu
Created: 2024-04-17
Impact: Low
Requires:
Replaces: It _could_ fully replace `DepositReserveAsset`, `InitiateReserveWithdraw` and `InitiateTeleport`.
---

## Summary

This RFC proposes new instructions that provide a way to initiate asset transfers which transfer multiple
types (teleports, local-reserve, destination-reserve) of assets, on remote chains using XCM alone.

The currently existing instructions are too opinionated and force each XCM asset transfer to a single
transfer type (teleport, local-reserve, destination-reserve). This results in inability to combine different
types of transfers in single transfer which results in overall poor UX when trying to move assets across
chains.

## Motivation

XCM is the de-facto cross-chain messaging protocol within the Polkadot ecosystem, and cross-chain
assets transfers is one of its main use-cases. Unfortunately, in its current spec, it does not support
initiating one or more transfers on a remote chain that combine assets with different transfer types.
For example, `ParachainA` cannot instruct `AssetHub` to teleport `ForeignAssetX` to `ParachainX` alongside
`USDT` (which has to be reserve transferred) using current XCM.

There currently exist `DepositReserveAsset`, `InitiateReserveWithdraw` and `InitiateTeleport` instructions
that initiate asset transfers on execution, but they are opinionated in the type of transfer to use.
Combining them is also not possible, because as a result of their individual execution, a message containing
a `ClearOrigin` instruction is sent to the destination chain, making subsequent transfers impossible.

The new instructions proposed by this RFC allow an XCM program to specify or "stage" multiple asset transfer
types, then execute them in one shot with a single `remote_xcm` program sent to the target chain to effect
the transfer and subsequently clear origin.

Multi-hop asset transfers will benefit from this change by allowing single XCM program to handle multiple
types of transfers and reduce complexity.
Bridge asset transfers greatly benefit from this change by allowing building XCM programs to transfer multiple
assets across multiple hops in a single pseudo-atomic action.

I.e. Allows single XCM program execution to transfer multiple assets from `ParaK` on Kusama, through Kusama
Asset Hub, over the bridge through Polkadot Asset Hub with final destination `ParaP` on Polkadot.
With current XCM, we are limited to doing multiple independent transfers for each individual hop in order to
move both "interesting" assets, but also "supporting" assets (used to pay fees).

## Specification

This RFC proposes 4 new XCM instructions:
```rust
/// Move the asset(s) matching given filter from holding to a dedicated assets transfer staging
/// register to be then teleported by a subsequent `ExecuteAssetTransfers` instruction.
///
/// - `assets`: The asset(s) to move from holding to the teleport staging register.
///
/// Kind: *Command*
TeleportTransferAssets(AssetFilter),

/// Move the asset(s) matching given filter from holding to a dedicated assets transfer staging
/// register to be then deposited as reserve by a subsequent `ExecuteAssetTransfers`
/// instruction.
///
/// - `assets`: The asset(s) to move from holding to the reserve deposit staging register.
///
/// Kind: *Command*
LocalReserveDepositAssets(AssetFilter),

/// Move the asset(s) matching given filter from holding to a dedicated assets transfer staging
/// register to be then burned and reserved assets released on a remote chain by a subsequent
/// `ExecuteAssetTransfers` instruction.
///
/// - `assets`: The asset(s) to move from holding to the reserve withdraw staging register.
///
/// Kind: *Command*
DestinationReserveWithdrawAssets(AssetFilter),

/// Cross-chain transfer asset(s) currently in the asset transfer staging registers as follows:
///
/// - teleport staging register: burn local assets and append a `ReceiveTeleportedAsset` XCM
///   instruction to the XCM program to be sent onward to the `dest` location,
///
/// - reserve deposit staging register: place assets under the ownership of `dest` within this
///   consensus system (i.e. its sovereign account), and append a `ReserveAssetDeposited` XCM
///   instruction to the XCM program to be sent onward to the `dest` location,,
///
/// - reserve withdraw staging register: burn local assets and append a `WithdrawAsset` XCM
///   instruction to the XCM program to be sent onward to the `dest` location,
///
/// The onward XCM is then appended a `ClearOrigin` to allow safe execution of any following
/// custom XCM instructions provided in `remote_xcm`.
///
/// The onward XCM also potentially contains a `BuyExecution` instruction based on the presence
/// of the `remote_fees` parameter (see below).
///
/// Parameters:
/// - `dest`: The location of the transfer destination.
/// - `remote_fees`: If set to `Some(asset_filter)`, the asset matching `asset_filter` in
///   either transfer staging register will be transferred first in the `remote_xcm` program,
///   followed by a `BuyExecution(asset)`, then rest of transfers follow. This guarantees
///   `remote_xcm` will successfully pass a `AllowTopLevelPaidExecutionFrom` barrier.
/// - `remote_xcm`: Custom instructions that will be executed on the `dest` chain. Note that
///   these instructions will be executed after a `ClearOrigin` so their origin will be `None`.
///
/// Safety: No concerns.
///
/// Kind: *Command*.
///
/// Errors:
ExecuteAssetTransfers { dest: Location, remote_fees: Option<AssetFilter>, remote_xcm: Xcm<()> },
```

`TeleportTransferAssets`, `LocalReserveDepositAssets`, `DestinationReserveWithdrawAssets` just move assets
matching the given filters from the holding register to the respective dedicated "asset transfer staging registers".
There would be a dedicated asset transfer staging register for each transfer type (teleport, local-reserve, destination-reserve).

The point of these 3 instructions is to allow "marking" one or more assets from the holding register for a
subsequent asset transfer within same XCM program execution.

Any leftover, not-transferred, assets in these registers at the end of XCM execution are handled the same
way as leftover assets in the holding register - currently they are being trapped.

A subsequent `ExecuteAssetTransfers { dest: Location, remote_fees: Option<AssetFilter>, remote_xcm: Xcm<()> }`
instruction is expected which will actually enact the cross-chain assets transfer.
Its purpose is to handle the local side of the transfer, then forward an onward XCM to `dest` for handling
the remote side of the transfer.

It does so using same mechanisms as existing `DepositReserveAsset`, `InitiateReserveWithdraw`, `InitiateTeleport`
instructions but practically combining all required XCM instructions to be remotely executed into a _single_
XCM program to be sent over to `dest`.

Furthermore, through `remote_fees: Option<AssetFilter>`, it allows specifying a single asset (which should
have already been "staged") to be used for fees on `dest` chain. This single asset is remotely handled/received
by the **first instruction** in the onward XCM and is followed by a `BuyExecution` instruction using it. The
rest of the assets are handled by subsequent instructions, thus allowing
[single asset buy execution](https://github.com/paritytech/polkadot-sdk/issues/2423).

Open Q: for `remote_fees == None`, should we append `UnpaidExecution` or do nothing?

To the onward XCM, following the assets transfers instructions, a `ClearOrigin` is appended to stop acting
on behalf of the source chain, then the caller-provided `remote_xcm` is also appended, allowing the caller
to control what to do with the transferred assets.

### Example usage: transferring 2 different asset types across 3 chains

- Transferring ROCs as the native asset of `RococoAssetHub` and PENs as the native asset of `Penpal`,
- Transfer origin is `Penpal` and the destination is `WestendAssetHub` (across the bridge),
- ROCs are native to `RococoAssetHub` and are registered as trust-backed assets on `Penpal` and `WestendAssetHub`,
- PENs are native to `Penpal` and are registered as teleportable assets on `RococoAssetHub` and as
  trust-backed assets on `WestendAssetHub`,
- Fees on `RococoAssetHub` and `WestendAssetHub` are paid using ROCs.

We can transfer them from `Penpal`, through `RococoAssetHub`, over the bridge to `WestendAssetHub` by executing
a _single_ XCM message:

```rust
PenpalA::execute_with(|| {
    let rocs: Asset = (rocs_id.clone(), rocs_amount).into();
    let pens: Asset = (pens_id, pens_amount).into();
    let assets: Assets = vec![rocs.clone(), pens.clone()].into();

    // XCM to be executed at dest (Westend Asset Hub)
    let xcm_on_dest =
        Xcm(vec![DepositAsset { assets: Wild(All), beneficiary: beneficiary.clone() }]);

    // XCM to be executed at Rococo Asset Hub
    let context = PenpalUniversalLocation::get();
    let reanchored_assets = assets.clone().reanchored(&local_asset_hub, &context).unwrap();
    let reanchored_dest = destination.clone().reanchored(&local_asset_hub, &context).unwrap();
    let reanchored_rocs_id = rocs_id.clone().reanchored(&local_asset_hub, &context).unwrap();
    let fun = WildFungibility::Fungible;
    let xcm_on_ahr = Xcm(vec![
        // both ROCs and PENs are local-reserve transferred to Westend Asset Hub
        LocalReserveDepositAssets(reanchored_assets.clone().into()),
        ExecuteAssetTransfers {
            dest: reanchored_dest,
            remote_fees: Some(AssetFilter::Wild(AllOf { id: reanchored_rocs_id.into(), fun })),
            remote_xcm: xcm_on_dest,
        },
    ]);

    // XCM to be executed locally
    let xcm = Xcm::<penpal_runtime::RuntimeCall>(vec![
        // Withdraw both ROCs and PENs from origin account
        WithdrawAsset(assets.clone().into()),
        // ROCs are reserve-withdrawn on AHR
        DestinationReserveWithdrawAssets(rocs.into()),
        // PENs are teleported to AHR
        TeleportTransferAssets(pens.into()),
        // Execute the transfers while paying remote fees with ROCs
        ExecuteAssetTransfers {
            dest: local_asset_hub,
            remote_fees: Some(AssetFilter::Wild(AllOf { id: rocs_id.into(), fun })),
            remote_xcm: xcm_on_ahr,
        },
    ]);

    <PenpalA as PenpalAPallet>::PolkadotXcm::execute(
        signed_origin,
        bx!(xcm::VersionedXcm::V4(xcm.into())),
        Weight::MAX,
    ).unwrap();
})
```

## Security considerations

There should be no security risks related to the new instruction from the XCVM perspective. It follows the same
pattern as with single-type asset transfers, only now it allows combining multiple types at once.

_Improves_ security by enabling [enforcement of single asset for buying execution](https://github.com/paritytech/polkadot-sdk/issues/2423),
which minimizes the potential free/unpaid work that a receiving chain has to do. It does so, by making the
required execution fee payment part of the instruction logic through the `remote_fees: Option<AssetFilter>`
parameter, which will make sure the remote XCM starts with a single-asset-holding-loading-instruction, immediately
followed by a `BuyExecution` using said asset.

## Impact

No impact to the rest of the spec. These are new, independent instructions, no changes to existing instructions.

## Alternatives

### What other designs have been considered and what is the rationale for not choosing them?

1. Modifying current behavior of `TransferReserveAsset`, `DepositReserveAsset`, `InitiateReserveWithdraw` to also only
"stage" a transfer, then execute it atomically (within same XCM execution) on destination. This is undesirable
because it would be a breaking change. Easier to create new instructions and maybe even deprecate above if desired.
2. Another alternative would be to create a single new instruction:
   ```rust
   InitiateAssetsTransfers {
       teleports: Option<AssetFilter>,
       local_reserve_assets: Option<AssetFilter>,
       dest_reserve_assets: Option<AssetFilter>,
       dest: Location,
       remote_fees: Option<AssetFilter>,
       remote_xcm: Xcm<()>
   }
   ```
   instead of the independent `TeleportTransferAssets(AssetFilter)`, `LocalReserveDepositAssets(AssetFilter)`,
   `DestinationReserveWithdrawAssets(AssetFilter)` "staging" instructions plus the
   `ExecuteAssetTransfers { dest: Location, remote_fees: Option<AssetFilter>, remote_xcm: Xcm<()> }` enactment
   instructions.
3. Create a new `Send { dest: Location, remote_xcm: Xcm<()> }` instruction to allow building of custom asset
   transfers using existing commands: e.g. for a teleport append `WithdrawAsset` and `BurnAsset` to an XCM to
   be locally executed and append `ReceiveTeleportedAsset` to an XCM to be sent to destination chain. 
   One could build whatever "custom" local and remote XCMs, then append `Send { dest, remote_xcm }` to the local
   one and execute it, which will also forward onward `remote_xcm` to be executed with source chain origin.
   However, this looks like a dangerous option as it enables execution of arbitrary instructions on behalf of
   the source chain and would need complex barriers in place to be made safe. Therefore, I deem it more complex
   and error-prone.

### What is the impact of not doing this?

Current multi-chain transfers are forced to happen in multiple programs per asset per "hop", resulting in very poor UX.

## Questions and open Discussions (optional)

- We should make sure it integrates smoothly with the [new fees system](https://github.com/paritytech/xcm-format/pull/53),
especially from an UX perspective.
- Inner `BuyExecution` specifies `WeightLimit::Unlimited`, thus being limited only by the asset "amount". This was
  a deliberate decision for enhancing UX - IMO in practice, people, even programs, care abount limiting fee asset
  amount not used weight.
- When setting `remote_fees == None`, should we append `UnpaidExecution` or do nothing?
