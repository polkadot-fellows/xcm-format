---
Title: Publish arbitrary data instruction
Number: 0
Status: Draft
Version: 4
Authors:
 - Francisco Aguirre
Created: 2023-11-16
Impact: Low
Requires:
Replaces:
---

## Summary

This RFC proposes a new instruction, `Publish`, to publish some data to another location.

## Motivation

XCM itself is not a good means of storing and retrieving arbitrary data.
This fixes the storage part, leaving the retrieval to another mechanism.

## Specification

The new instruction looks like this:

```rust
Publish { data: BoundedVec<Key, Value, Bound> }
```

`Key` and `Value` are arrays of bits, they could be anything.

The `Bound` value should be configurable.

### Examples

A program like the following might be sent to another location for publishing some data:

```rust
WithdrawAsset(/* ... */),
BuyExecution { /* ... */ }
Publish { data: [(b"My key", b"My value")] }
RefundSurplus, // <- In case we overpay
DepositAsset { /* ... */ }
```

The other location stores this key value pair in any manner appropriate to it, which might allow different types of retrieval afterwards, or none at all.

## Security considerations

Programs that publish data would need to pay a certain amount of funds proportionate to the size of the data.

## Impact

This proposal adds a new instruction, so the impact is low.

## Alternatives

The other alternative to this is to use `Transact`, since it allows sending an encoded blob of bytes.
This, however, falls short, as it's expected that the blob decodes to a function call on the receiver.
This new instruction makes it clear that it's talking about data and not a function call, while at the same time allowing for multiple key value pairs to be sent.

## Questions and open Discussions (optional)

- Should we account for the published data's size via Weight?
- Should we introduce a new instruction for storage deposits or add another operand to the `Publish` instruction specifying the assets used?
- Should there be a way to delete the published data via XCM?
