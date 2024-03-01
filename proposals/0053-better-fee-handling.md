---
Title: Fee handling generalization and simplification
Number: 53
Status: Draft
Version: 5
Authors:
 - Francisco Aguirre
Created: 2024-03-01
Impact: Low
Requires:
Replaces:
---

## Summary

XCM already handles execution fees in an effective and efficient manner using the `BuyExecution` instruction.
However, other types of fees are not handled as effectively -- for example, delivery fees.
Fees exist that can't be measured using `Weight` -- as execution fees can -- so a new method should be thought up for those cases.
This RFC proposes making the fee handling system simpler and more general, by doing two things:
- Adding a `fees` register
- Replacing `BuyExecution` with a new instruction `PayFees` with new semantics

This new instruction only takes the amount of fees that the XCVM can use from the holding register.
The XCVM implementation is free to use these fees to pay for execution fees, transport fees, or any other type of fee that might be necessary.
This moves the specifics of fees further away from the XCM standard, and more into the actual underlying XCVM implementation, which is a good thing.

## Motivation

Execution fees are handled correctly by XCM right now.
However, the addition of extra fees, like for message delivery, result in awkward ways of integrating them into the XCVM implementation.
The standard should have a way to correctly deal with these implementation specific fees, that might not exist in every system that uses XCM.

## Specification

The new instruction that will replace `BuyExecution` is a much simpler and general version: `PayFees`.
This instruction takes some `Assets` and takes them from the holding register, putting them into a new `fees` register.
The XCVM implementation can now use these `Assets` to make sure every necessary fee is paid for, this includes execution fees, delivery fees, or any other fee.

```rust
PayFees { assets: Assets }
```

This new instruction will reserve **the entirety** of `assets` for fee payment.
The assets can't be used for anything else during the entirety of the program.
This is different from the current semantics of `BuyExecution`.

If not all `Assets` in the `fees` register are used when the execution ends, then we trap them.

### Examples

Most XCM programs that pay for execution are written like so:

```rust
// Instruction that loads the holding register
BuyExecution { assets, weight_limit }
// ...rest
```

With this RFC, the structure would be the same, but using the new instruction, that has different semantics:

```rust
// Instruction that loads the holding register
PayFees { assets }
// ...rest
```

## Security considerations

Overpaying for fees might have an increase in trapped assets.
Depending on the method used for trapping assets, this might result in too much storage used.

## Impact

For systems with no fees other than execution, this change would be just a rename of the instruction.
For systems with other types of fees, migration to this new structure would be advised, as they currently can't express the new fees.

## Alternatives

Alternatives include:
1. Simply changing the semantics of `BuyExecution`
2. Create a new instruction for each conceivable type of fees

Alternative number 1, while possible, would result in a lot of existing programs breaking because of changed assumptions.
The change must happen either in a new instruction, or in a new version, to ensure those programs are not broken.
While the instruction could be called the same in a new version of the format, changing the name makes the changes clearer.

Alternative number 2 is not only unfeasible, but also brings a lot of implementation-specific details into the format.

The impact of not doing this is suffering from bugs born from failing to handle the different type of fees.

## Questions and open Discussions (optional)

The name of the instruction is not set in stone.
There might be a better way of handling fee overpayment.
