---
Title: Add Paseo NetworkId
Number: 0
Status: Draft
Version: 5
Authors:
 - Francisco Aguirre
Created: 2024-05-23
Impact: Low
Requires:
Replaces:
---

## Summary

This RFC proposes adding the Paseo testnet as a `NetworkId`, for easier identification.

## Motivation

While the Paseo testnet can be addressed using `NetworkId::ByGenesis`, the importance in the Polkadot ecosystem merits
their own `NetworkId`, just as `Rococo`.

## Specification

The `NetworkId` enum is getting a new variant, `Paseo`.

The relevant paragraph in the spec will be changed to reflect this.

## Security considerations

None.

## Impact

The impact of this proposal is low.
XCMs referencing Paseo via `ByGenesis` can just change to use the `Paseo` specific `NetworkId`.

## Alternatives

Not add the specific `NetworkId` and just leave it as `ByGenesis`.
