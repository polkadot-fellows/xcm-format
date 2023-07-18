---
Title: Renaming Lock Instructions
Number: 0
Status: Draft
Version: 0
Authors:
 - vstam1
Created: 2023-07-18
Impact: Breaking
Requires:
Replaces:
---

## Summary

This RFC proposes renaming several XCM instructions to align with changes in Substrate naming.
The following instructions and error are affected:

Instructions:
`LockAsset` -> `FreezeAsset`
`UnlockAsset` -> `ThawAsset`
`NoteUnlockable` -> `NoteThawable`
`RequestUnlock` -> `RequestThaw`

Error:
`LockError` -> `FreezeError`


## Motivation

The renaming of these instructions is intended to eliminate potential confusion caused by naming mismatches between XCM and Substrate. By ensuring consistency in naming conventions across both platforms, we can enhance clarity and maintain a unified language.

## Specification

The renamed instructions are defined as follows:
```rust
/// Freeze the locally held asset and prevent further transfer or withdrawal.
///
/// This restriction may be removed by the `ThawAsset` instruction being called with an
/// Origin of `thawer` and a `target` equal to the current `Origin`.
///
/// If the freezing is successful, then a `NoteThawable` instruction is sent to `thawer`.
///
/// - `asset`: The asset(s) which should be frozen.
/// - `unlocker`: The value which the Origin must be for a corresponding `ThawAsset`
///   instruction to work.
FreezeAsset { asset: MultiAsset, thawer: MultiLocation },

/// Remove the freeze over `asset` on this chain and (if nothing else is preventing it) allow the
/// asset to be transferred.
///
/// - `asset`: The asset to be thawed.
/// - `target`: The owner of the asset on the local chain.
ThawAsset { asset: MultiAsset, target: MultiLocation },

/// Asset (`asset`) has been frozen on the `origin` system and may not be transferred. It may
/// only be thawed with the receipt of the `ThawAsset` instruction from this chain.
///
/// - asset: The asset(s) which are now thawable from this origin.
/// - owner: The owner of the asset on the chain where it was frozen. This may be a
/// location specific to the origin network.
///
/// Safety: origin must be trusted to have frozen the corresponding asset
/// prior as a consequence of sending this message.
NoteFreezable { asset: MultiAsset, owner: MultiLocation },

/// Send a ThawAsset instruction to the thawer for the given asset.
///
/// This may fail if the local system is making use of the fact that the asset is frozen or,
/// of course, if there is no record that the asset actually is frozen.
///
/// - asset: The asset(s) to be thawed.
/// - thawer: The location from which a previous NoteThawable was sent and to which
/// a ThawAsset should be sent.
RequestThaw { asset: MultiAsset, thawer: MultiLocation },
```

And Error:
```rust
/// Some other error with Freeze.
FreezeError
```

## Security considerations

The renaming process must ensure that all instances of the old instructions are properly
converted to the new ones to maintain compatibility between the different XCM versions.

## Impact

This change will impact all XCM code that uses the lock instructions, constituting a breaking change. All code should be updated to use these new names. Failure to update the code could result in errors.

## Alternatives

An alternative to this change would be to keep the naming the same. However, this would result in a naming mismatch between XCM and Substrate, which could lead to confusion among developers and users.

## Questions and open Discussions (optional)

- Is `NoteThawable` the most suitable replacement for `NoteUnlockable`, or are there better alternatives?
