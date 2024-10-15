---
Title: Proposal Template
Number: 0
Status: Draft
Version: 0
Authors:
 - Your Name
Created: 2024-10-01
Impact: Low
Requires:
Replaces:
---

## Summary

This RFC propose a new instruction and modify an existing instruction to support XCQ(Cross-Consensus Query Language). XCQ is designed to provide an universal way to query onchain data across different consensus system.

## Motivation

XCM enables mutable interactions across different consensus systems.
But we still need another subsystem to query information across different consensus systems, which abstract away the concrete implementations in these systems.

Such a subsystem will benefit the tools and UI developers.
For example, for a substrate-based chain, an account can have different value-bearing assets in different pallets (i.e. balances pallet, DEX pallet, liquid staking pallet). Conventionally, wallet developers has no easy way but examine all the pallets to scrape all the information they need. Some operations require reading storage directly, which is also subject to breaking changes.

The proposal of XCQ will serve as a layer between specific chain implementations and tools/UI developers.
Common reusable view functions are supported through an extension-based design in a permission-less manner. Given the result of these view functions, complex computations can be performed through a prebuilt program.

## Specification

The proposed change introduce a new instruction and add a new variant in the `Response` type:

1. A new `ReportQuery` instruction

```rust
ReportQuery {
  query: SizeLimitedXcq,
  weight_limit: Option<Weight>,
  info: QueryResponseInfo,
}
```

Report to a given destination the results of an XCQ query. After query, a `QueryResponse` message of type `XcqResult` will be sent to the described destination.

Operands:

- `query: SizeLimitedXcq` has a size limit(2MB)
  - `program: Vec<u8>`: A pre-built PVM program binary
  - `input: Vec<u8>`: The arguments of the program
- `weight_limit: WeightLimit`: The maximum weight that the query should take. `WeightLimit` is an enum that can specify either `Limit(Weight)` or `Unlimited`.
- `info: QueryResponseInfo`: Information for making the response
  - `destination: Location`: The destination to which the query response message should be sent.
  - `query_id: Compact<QueryId>`: The `query_id` field of the `QueryResponse` message
  - `max_weight: Weight`: The `max_weight` field of the `QueryResponse` message

2. Add a new variant the `Response` type in `QueryResponse`

- `XcqResult = 6 (XcqResult)`
`XcqResult` is a enum
- `Ok = 0 (Vec<u8>)`: XCQ executes successfully with the response as SCALE-encoded.
- `Err = 1 (ErrorCode)`: XCQ fails with the error indicates.
`ErrorCode` is a enum
- `MemoryAllocationError = 0`
- `MemoryAccessError = 1`
- `CallError = 2`
- `OtherPVMError = 3`

### Errors

- `ExceedMaxWeight`

## Security considerations

Since it's a read-only operation, basically it's would be safe.

## Impact

The modification of `Response` type needs to be handled.

## Alternatives

There are other options to support the cross consensus query, including adding raw runtime apis or view functions. However, Defining raw runtime apis and view functions will be a permissioned process and will need a runtime upgrade everytime, which hinders the iterations. Compared to them, XCQ is permission-less, more extendible, more user-friendly.
