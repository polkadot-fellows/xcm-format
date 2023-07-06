---
Title: Custom Asset Claimer
Number: 0
Status: Draft
Version: 0
Authors:
 - Daniel Shiposha
Created: 2023-06-21
Impact: Low
Requires:
Replaces:
---

## Summary

The proposed change provides a way to set a claimer to potentially dropped assets.
Currently, determining a claimer of dropped assets is implementation-specific, and there is no way to set a custom claimer.
The ability to set a custom claimer origin makes it easier to rescue the dropped assets, especially in the case of a reserve-based transfer.

## Motivation

XCM is an important feature of the Polkadot ecosystem that enables inter-chain communication and value transfer between ecosystem parachains. When assets (fungible or non-fungible) are transferred using XCM, they may be dropped due to miscellaneous errors during the execution. The probability of such an event is low, but it is not negligible, and dropped assets will need to be recovered to restore the user control of them. The process of such recovery is called "claiming", and the account that has the permission to perform it is called "claimer".

Currently, determining a claimer of dropped assets is implementation-specific, and there is no way to set a custom claimer. The implementation could choose a predictable claimer origin such as the same origin as the origin of the XCM message. But also, this choice could be arbitrary if there is no origin, e.g., if `ClearOrigin` was executed before an error happened.

The latter case includes reserve-based transfers. Given that parachains mostly rely on reserve-based transfers, a way to define a custom claimer for the dropped assets makes rescuing more convenient since it provides more ways to do so compared to an arbitrary claimer setting defined by a specific implementation.

 <!-- (though, if no claimer was set, the implementation could define the default one). -->

Having this could become especially important when transferring multiple currencies or NFTs:
 * When transferring multiple currencies, one must choose a currency to pay execution fees.
   If there are insufficient funds in the chosen currency, **all** the assets will be dropped. And the dropped amount could be significant, so we need a convenient way to recover it.
 * When transferring NFTs, we must transfer some fungibles alongside them to pay fees. This is precisely the same situation as with multiple currencies. Also, an NFT representing a unique object could be subjectively very valuable to a user, so its dropping can be compared to dropping a large number of fungibles.

This RFC proposes adding a new instruction `SetAssetClaimer`. The new instruction sets a custom claimer to the dropped assets.

## Specification

The instruction would have the following specification:

```rust
SetAssetClaimer(MultiLocation)
```

The `SetAssetClaimer` instruction allows XCVM to hint to the asset drop implementation that the specified `MultiLocation` is the claimer to all the assets if they are dropped in the case of an error.

The information about the claimer should be a part of the XCM Context so that Asset Transactors could also know to whom to provide a claim in case of an error.

When no claimer is set, the default claimer shall be defined by the implementation of the XCVM.

### Example

```rust
WithdrawAsset(/* <assets-to-transfer> */),
InitiateReserveWithdraw {
    assets: Wild(All),
    reserve: /* <reserve-chain> */,
    xcm: vec![
        // In case of an error on the <reserve-chain>,
        // `<MY_ACCOUNT>` can claim the dropped assets.
        SetAssetClaimer(AccountId32 {
            network: None,
            id: /* <MY_ACCOUNT> */,
        }),
        BuyExecution { /* ...snip... */ },
        DepositReserveAsset {
            assets: Wild(All),
            dest: /* <destination-chain> */,
            xcm: vec![
                // In case of an error on the <destination-chain>,
                // `<MY_ACCOUNT>` can claim the dropped assets.
                SetAssetClaimer(AccountId32 {
                    network: None,
                    id: /* <MY_ACCOUNT> */,
                }),
                BuyExecution { /* ...snip... */ },
                DepositAsset { /* ...snip... */ },
            ].into(),
        }

    ].into(),
}
```

## Security considerations

There are no security risks related to the new instruction from the XCVM perspective.

## Impact

The impact of this proposal is Low. It introduces a new instruction, so XCVM implementations have to be updated.

## Alternatives

Currently, there is no way to set a custom claimer.

