---
Title: Add SetExecutionHints instruction
Number: 0
Status: Draft
Version: 5
Authors:
 - Francisco Aguirre
Created: 2024-05-23
Impact: Medium
Requires: 37
Replaces:
---

## Summary

RFC#37 introduces a `SetAssetClaimer` instruction.
This idea of instructing the XCVM to change some implementation-specific behavior is useful.
In order to generalize this mechanism, this RFC introduces a new instruction `SetExecutionHints`
and makes the `SetAssetClaimer` be just one of many possible execution hints.

## Motivation

There is a need for specifying how certain implementation-specific things should behave.
Things like who can claim the assets or what can be done instead of trapping assets.

## Specification

A new instruction, `SetExecutionHints`, will be added.
This instruction will take a single parameter of type `ExecutionHint`, an enumeration.
The first variant for this enum is `AssetClaimer`, which allows to specify a location that should be able to claim trapped assets,
as specified in RFC#37.
This means the instruction `SetAssetClaimer` would also be removed, in favor of this.

In Rust, the new definitions would look as follows:

```rust
enum Instruction {
  // ...snip...
  SetExecutionHints(BoundedVec<ExecutionHint, NumVariants>),
  // ...snip...
}

enum ExecutionHint {
  AssetClaimer(Location),
  // more can be added
}

type NumVariants = /* Number of variants of the `ExecutionHint` enum */;
```

## Security considerations

`ExecutionHint`s are specified on a per-message basis, so they have to be specified at the beginning of a message.
If they were to be specified at the end, hints like `AssetClaimer` would be useless if an error occurs beforehand and assets get trapped before ever reaching the hint.

The instruction takes a bounded vector of hints so as to not force barriers to allow an arbitrary number of `SetExecutionHint` instructions.

## Impact

The impact is medium as it implies changing one instruction into another that supersedes it.
However, the conversion is simple.
If `SetAssetClaimer` hasn't been implemented, then the impact is low.

## Alternatives

Call the new system `Config` instead of `ExecutionHint`.
However, it collides with the known `XCM Config`, which is specified once for a runtime.
Instead, `ExecutionHint`s are specified on a per-message basis.

The "hint" terminology is used taking inspiration from GLFW's `WindowHint`s, which change the default
behavior of windows.

## Questions and open discussions

RFC#58 discusses potentially trapping leftover fees as well, this might lead to a lot of assets being trapped,
assets that can be hard to claim.
This RFC sets the groundwork for a potential `LeftoverFeesDestination` hint that allows sending all
leftover fees to a specific location.
It could also be a `LeftoverAssetsDestination` hint to send all leftover assets to one location instead of trapping them.
